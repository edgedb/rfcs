::

    Status: Final
    Type: Feature
    Created: 2021-12-10
    Authors: Michael J. Sullivan <sully@msully.net>
    RFC PR: `edgedb/rfcs#0039 <https://github.com/edgedb/rfcs/pull/39>`_

===============
RFC 1009: GROUP
===============

This RFC proposes adding a ``GROUP`` statement to EdgeQL to
perform aggregation.


Motivation
==========

Aggregation of data by keys is a key operation in analytics-like
queries, and EdgeQL currently serves poorly.

Consider trying to, in our "cards" test basebase representing a
collectible card game, finding the average cost of a card for each of
the different "elements" are card could have. In EdgeQL now, this can
be expressed as::

  FOR element in (DISTINCT Card.element) UNION (
      WITH g := (SELECT Card FILTER .element = element)
      SELECT {
          element := element,
          avg_cost := sum(g.cost) / count(g)
      } ORDER BY .element
  )

This query, while it gets the job done is fairly complex and
distressingly manual. On top of that, it generates a bad query that
needs to do a subquery scan for each element!

Implementing a ``GROUP`` statement in EdgeQL will both allow us to
provide much better developer experience when doing aggregating
queries and give us the structure needed to target Postgres's
aggregation features in compilation.


Specification
=============

The new ``GROUP`` proposal is an objects-forwards proposal,
which produces a set of free objects containing each produced
group. The encouraged idiom will be to immediately process the results
with a ``SELECT`` statement containing a shape that performs
aggregation, in most cases.

::

   GROUP
       [<alias> := ] <expr>

   [USING
       <alias1> := <expr>,     # define parameters to use for
       <alias2> := <expr>,     # grouping
       ...
       <aliasN> := <expr>]

   BY <grouping-ref>, ...      # specify which parameters will
                               # be used to partition the set;

where a ``<grouping-ref>`` is one of::

  <identifier>
  .<path-name>


The ``USING`` clause defines a set of aliases. Each alias is available in
later alias definitions and in the ``BY`` clause.

The ``BY`` clause may refer to aliases as well as to immediate partial
paths of the subject expression.


The *behavior* of ``GROUP`` is to evaluate the subject expression and
then, for each element of the set, evaluate each of the ``USING``
expressions (which much be singletons). Then a grouping key is created
based on the ``BY`` clause, and the objects are aggregated into groups
with matching grouping keys.

The output is reflected in an ad-hoc free object of the following form::

  {
    key: <another free object containing "alias := value" pointers>,
    elements: <the set of all the objects with the matching key>,
    grouping: <set of keys used in grouping>,
  }


Grouping sets
-------------

To extend this to support multiple grouping sets, we alter the syntax for the
``BY`` clause.

To do this, we generalize ``BY`` as such::

  BY <grouping-element>, ...

where ``grouping-element`` is one of::

  <ref-or-list>
  {<grouping-element>, ...}
  ROLLUP(<ref-or-list>, ...)
  CUBE(<ref-or-list>, ...)

and ``ref-or-list`` is one of::

  <grouping-ref>
  ()
  (<grouping-ref>, ...)


Braces specify a grouping set (and can be thought of as declaring
a set of keys, using set syntax.)
When grouping by a grouping set, we group by *each* element of the
grouping set.
The ``grouping`` element of the output shape indicates
which grouping set a group is from, by listing all the keys used in
the group, in the order they appeared in the ``BY`` clause.

When there are multiple top-level ``grouping-elements``, and some are
grouping sets, then the cartesian product of them is taken to determine
the final grouping sets. That is ``a, {b, c}`` is equivalent
to ``{(a, b), (a, c)}``. The best way to think about this is as analogue
of EdgeQL's normal evaluation rules, which computes cross products
whenever lifting operators over sets.


``ROLLUP`` and ``CUBE`` are basic syntactic sugar for grouping sets.
``ROLLUP`` groups by all prefixes of a list of elements, so
``ROLLUP (a, b, c)`` is equivalent to
``{(), (a), (a, b), (a, b, c)}`` while ``CUBE``
considers all elements of the power set, so that
``CUBE (a, b)`` is equivalent to ``{(), (a), (b), (a, b)}``.
These elements can be lists of aliases, also.

If a grouping set occurs multiple times, the group appears in the output
once for each time it was specified.

This all basically exactly follows SQL, except that we write braces
instead of ``GROUPING SETS``.


Reference implementation
========================

A reference implementation of ``GROUP BY`` has been implemented in the
toy evaluator model [2]_.

Examples
========

The example outputs in this section come from the toy evaluator model,
and so don't exactly match what the CLI output will be.

Doing a very basic GROUP BY without any aggregation of the data::

  GROUP Card {name} BY .element

This produces::

  {
    {
      key: {element: 'Air'},
      grouping: {'element'},
      elements: {
	default::Card {name: 'Sprite'},
	default::SpecialCard {name: 'Djinn'},
	default::Card {name: 'Giant eagle'},
      },
    },
    {
      key: {element: 'Earth'},
      grouping: {'element'},
      elements: {
	default::Card {name: 'Dwarf'},
	default::Card {name: 'Golem'},
      },
    },
    {
      key: {element: 'Fire'},
      grouping: {'element'},
      elements: {
	default::Card {name: 'Dragon'},
	default::Card {name: 'Imp'},
      },
    },
    {
      key: {element: 'Water'},
      grouping: {'element'},
      elements: {
	default::Card {name: 'Giant turtle'},
	default::Card {name: 'Bog monster'},
      },
    },
  }


Computing the average cost of each "element" that a card can have::

  SELECT (GROUP Card BY .element) {
      element := .key.element,
      avg_cost := sum(.elements.cost) / count(.elements),
  } ORDER BY .element

::

  {
    {element: 'Air', avg_cost: 2.3333333333333335},
    {element: 'Earth', avg_cost: 2.0},
    {element: 'Fire', avg_cost: 3.0},
    {element: 'Water', avg_cost: 2.5},
  }

Computing the ratio of each card's cost to the average of its element::

  SELECT (
    FOR g in (GROUP Card BY .element) UNION (
      WITH U := g.elements,
      SELECT U {
          name,
          cost_ratio := .cost / math::mean(g.elements.cost)
      })
  ) ORDER BY .name;

::

  {
    default::Card {name: 'Bog monster', cost_ratio: 0.8},
    default::SpecialCard {name: 'Djinn', cost_ratio: 1.7142857142857142},
    default::Card {name: 'Dragon', cost_ratio: 1.6666666666666667},
    default::Card {name: 'Dwarf', cost_ratio: 0.5},
    default::Card {name: 'Giant eagle', cost_ratio: 0.8571428571428571},
    default::Card {name: 'Giant turtle', cost_ratio: 1.2},
    default::Card {name: 'Golem', cost_ratio: 1.5},
    default::Card {name: 'Imp', cost_ratio: 0.3333333333333333},
    default::Card {name: 'Sprite', cost_ratio: 0.42857142857142855},
  }

Counting the number of cards in each possible "element", "number of
owners" combination bucket, as well as those things individually::

  SELECT (
    GROUP Card
	USING nowners := count(.owners)
    BY CUBE (.element, nowners)
  ) {
      key: {element, nowners},
      num := count(.elements),
      grouping
  }
  ORDER BY array_agg((SELECT _ := .grouping ORDER BY _))
  THEN .key.element THEN .key.nowners;

::

  {
    {key: {element: {}, nowners: {}}, num: 9, grouping: {}},
    {key: {element: 'Air', nowners: {}}, num: 3, grouping: {'element'}},
    {key: {element: 'Earth', nowners: {}}, num: 2, grouping: {'element'}},
    {key: {element: 'Fire', nowners: {}}, num: 2, grouping: {'element'}},
    {key: {element: 'Water', nowners: {}}, num: 2, grouping: {'element'}},
    {
      key: {element: 'Air', nowners: 2},
      num: 3,
      grouping: {'element', 'nowners'},
    },
    {
      key: {element: 'Earth', nowners: 2},
      num: 1,
      grouping: {'element', 'nowners'},
    },
    {
      key: {element: 'Earth', nowners: 3},
      num: 1,
      grouping: {'element', 'nowners'},
    },
    {
      key: {element: 'Fire', nowners: 1},
      num: 1,
      grouping: {'element', 'nowners'},
    },
    {
      key: {element: 'Fire', nowners: 2},
      num: 1,
      grouping: {'element', 'nowners'},
    },
    {
      key: {element: 'Water', nowners: 4},
      num: 2,
      grouping: {'element', 'nowners'},
    },
    {key: {element: {}, nowners: 1}, num: 1, grouping: {'nowners'}},
    {key: {element: {}, nowners: 2}, num: 5, grouping: {'nowners'}},
    {key: {element: {}, nowners: 3}, num: 1, grouping: {'nowners'}},
    {key: {element: {}, nowners: 4}, num: 2, grouping: {'nowners'}},
  }

Comparison with SQL
===================

In SQL, ``GROUP BY`` is a clause that may be applied to ``SELECT``,
not a standalone statement. SQL ``GROUP BY`` changes the meaning
of the statement such that aggregate functions are computed across
all rows in a *group*, rather than across all rows. Additionally,
it requires all columns other than the grouped keys to be referenced
only as arguments to aggregate functions.

We want our ``GROUP`` to be more flexible than SQL's. Since we support
sets as a first class object, we directly expose the groups as a set,
which is output as an element of a free shape. This allows directly
outputting the full groups, as well as more complex queries such as
the "ratio of each card's cost" example above.

The big advantage of our ``GROUP`` is that all of the results are
exposed in ways which are idiomatic to the language and easily
composable.


Implementation notes
--------------------

The increased flexibility comes with a downside, however, which is that
mapping our ``GROUP`` to SQL's may be difficult.

In basic, common cases, where the result of the ``GROUP`` is
immediately consumed by a shape that uses ``.elements`` only in the
argument to aggregates, we should be able to directly take advantage
of SQL ``GROUP BY`` with little drama.

In the case where a ``GROUP`` is directly presented to the output, we
should also be able to use SQL ``GROUP BY`` without much trouble,
since ``array_agg``, used to produce our serialized output, is an
aggregate function.

That is also the core of an implementation strategy for the "general
case" of ``GROUP`` that I am fairly confident is reasonably
implementable: use ``array_agg`` in the SQL ``GROUP BY`` and treat it
as a materialized computed set.

We've discussed using window functions to implement the general
case. We'll need to dicuss this more, but after looking at the docs,
it's not obvious to me how that would work in general. It might be
doable for certain cases, like the "ratio of each card's cost"
example?

I think using window functions in the "fully general" case can't work,
since they don't seem to support grouping sets?


Backwards Compatibility
=======================

``BY`` needs to be made into a reserved keyword in order for the
``USING`` clause to permit a trailing comma. (Otherwise we can't
distinguish between the start of the ``BY`` clause and the definition
of an alias named ``BY`` at one token of lookahead.)

``GROUP`` is already a reserved keyword in the implementation.

Security Implications
=====================

There are no security implications.

Rejected Alternative Ideas
==========================

Non-shape based GROUP BY
------------------------

The initial recent proposal, heavily inspired by the original deleted
EdgeQL ``GROUP BY`` [1]_, was (approximately)::

  GROUP
      [<alias> := ] <expr>

  BY
      <alias1> := <expr>,     # define parameters to use for
      <alias2> := <expr>,     # grouping
      ...
      <aliasN> := <expr>

  [USING <alias>, ... ]       # specify which parameters will
                              # be used to partition the set;
                              # if unspecified, use all aliases declared in BY

  UNION
      <expr>                  # map every grouped set onto a result set,
                              # merging them all with a UNION ALL
                              # (or UNION for Objects)

In this proposal, the ``BY`` clause acts like the ``USING`` clause
above, and ``USING`` acts like ``BY``, except only can refer to
aliases, and can be omitted, in which case it is considered to group
by every alias.

Instead of producing a free object automatically, the output is
produced by an explicit ``UNION`` clause. Within the ``UNION`` clause,
the aliases from the ``BY`` clause are bound to the grouping columns
and the subject alias is bound to the *set* containing all the
elements in the group.
(The subject alias may be omitted if the subject expression is a
simple set name reference, in which case that name is also used as the
alias.)

This is a solid and reasonable design, though I'm sure we could argue
about the syntax for a while.

One key difference with the main proposal is that is essentially
orthogonal to other language features. This would make it well suited
when presenting a core calculus for EdgeDB, for example, since it can
be considered in isolation, while the shape based proposal depends on
free objects and arrays. The shape-based version can easily be
implemented in terms of this one.

That said, this is also its disadvantage: the shape-based version fits
in nicer with the language and better leverages the other features it
has.

The implementation of GROUP actually operates by implementing this
version and then desugaring the shape-based version into it.

(One other annoying issue with this approach is because of the
presence of UNION in the expression grammar, either we need to use a
different keyword for UNION (like INTO), or the BY clauses need to be
FOR iterator style restricted expressions. Indeed it was while
considering this that we came upon that solution for FOR.)


Doubly nested shape output
--------------------------

One proposal considered was to produce output in a *doubly* nested
form, with the outer grouping done by grouping set, like so::

  {
      grouping: <set of keys used in grouping>,
      group: {
          key: <another free object containing "alias := value" pointers>,
          elements: <the set of all the objects with the matching key>,
      },
  }


This has the advantage that it fully leans into shapes and
conveniently organizes the data when using grouping sets.

Unfortunately it significantly complicates the common case of having
only a single grouping set, and I believe would require a second outer
GROUP BY to implement.

It is pretty straightforward to get this behavior when desired, though
by performing another outer GROUP BY.


Shape based BY CLAUSE
---------------------

Another proposal is to have a shape-based BY clause, something like::

  group User { first := .name[0] }
  by { age, first }


Colin can write some more here, if he'd like.

I think this was intended to be separate grouping sets for ``age`` and
``first``, though you could also imagine it being grouping on both.

This was decided against because it felt like a needless abuse of
shape-like syntax and because it prevents grouping on scalar types
without wrapping them in a free object.


Other formats for grouping
--------------------------

Initial versions of this proposal had the ``grouping`` field as an
array instead of a set.

The advantage of approach is that it is easy to compare arrays for
equality, to order by arrays, and group by an array.

The disadvantage is that we are a set-based language, and it is
somewhat incongrous to directly produce an array
instead. Additionally, an array needs to have some sort of fixed
ordering, which users would need to occasionally reason about
(probably ordered by first appearance in the ``BY`` clause).

To support this decision, we need to add a set equality
operator (for comparing against ``grouping``) and an easy way
to sort (for producing something we can group or order by
in a less cumbersome way than
``array_agg((SELECT _ := .grouping ORDER BY _))``.

Another released option would be to have it be a set, but to specify
an order for the set, so that it could be used in an ``array_agg``
without needing to sort it explicitly. This was felt to be a departure
from our normal semantics, so we aren't specifying that. (The
implementation does still work that way, though.)


.. [1] https://github.com/edgedb/edgedb/issues/104#issuecomment-344307260

.. [2] https://github.com/edgedb/edgedb/tree/group-proposal
