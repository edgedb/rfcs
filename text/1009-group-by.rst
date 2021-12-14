::

    Status: Draft
    Type: Feature
    Created: 2021-12-10
    Authors: Michael J. Sullivan <sully@msully.net>
    RFC PR: `edgedb/rfcs#00XX <https://github.com/edgedb/rfcs/pull/XX>`_

==================
RFC 1009: GROUP BY
==================

This RFC proposes adding a `GROUP ... BY` statement to EdgeQL to
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

This query, while it gets the job done is pretty ugly and
distressingly manual. On top of that, it generates a bad query that
needs to do a subquery scan for each element!

Implementing a ``GROUP`` statement in EdgeQL will both allow us to
provide much better developer experience when doing aggregating
queries and give us the structure needed to target Postgres's
aggregation features in compilation.


Specification
=============

The new ``GROUP ... BY`` proposal is an objects-forwards proposal,
which produces a set of free objects containing each produced
group. The encouraged idiom will be to immediately process the results
with a ``SELECT`` statement containing a shape that performs
aggregation, in most cases.

::

   GROUP
       [<alias> := ] <expr>

   BY
       [<alias1> := ] <expr>,  # define parameters to use for
       [<alias2> := ] <expr>,  # grouping
       ...
       [<aliasN> := ] <expr>

   [GROUPINGS (<alias>, ...) ] # specify which parameters will
                               # be used to partition the set;
                               # if unspecified, use all aliases declared in BY



The aliases in the BY clause are optional only if the expression is an
partial path reference such as ``.name``, and ``.name`` is treated as
equivalent to ``name := .name``, except that ``name`` is not made available
to later aliases in the ``BY`` clause.

The ``GROUPINGS`` clause, if omitted, is taken to be the list of all specified
aliases in the ``BY`` clause. This means it is usually superfluous in this
initial pre-grouping sets version, but it does allow aliases in the BY
clause to be defined solely as a helper to later aliases.

The optional alias in the subject will be bound in the ``BY`` clauses.

The *behavior* of ``GROUP BY`` is to evaluate the subject expression and
then, for each element of the set, evaluate each of the ``BY``
expressions (which much be singletons). Then a grouping key is created
based on the ``GROUPINGS`` clause, and the objects are aggregated into groups
with matching grouping keys.

The output is reflected in an ad-hoc free object of the following form::

  {
      key: <another free object containing "alias := value" pointers>,
      elements: <the set of all the objects with the matching key>,
      grouping: <array of keys used in grouping>,
  }



Grouping sets
-------------

To extend this to support multiple grouping sets, we alter the syntax for the
``GROUPINGS`` clause.

To do this, we generalize ``GROUPINGS`` as such::

  [GROUPINGS <grouping-element>, ...]

where ``grouping-element`` is one of::

  (<ident>, ...)
  ROLLUP (<alias-or-list>, ...)
  CUBE (<alias-or-list>, ...)

and ``alias-or-list`` is one of::

  <ident>
  ()
  (<ident>, ...)


Each ``<grouping-element>`` specifies one or more grouping sets.
When grouping by a grouping set, we group by *each* element of the
grouping set.
The ``grouping`` element of the output shape indicates
which grouping set a group is from, by listing all the keys used in
the group, in the order they appeared in the ``BY`` clause.

``ROLLUP`` and ``CUBE`` are basic syntactic sugar for grouping sets.
``ROLLUP`` groups by all prefixes of a list of elements, so
``ROLLUP (a, b, c)`` is equivalent to ``(), (a), (a, b), (a, b, c)``
while ``CUBE`` considers all elements of the power set, so that
``CUBE (a, b)`` is equivalent to ``(), (a), (b), (a, b)``.
These elements can be lists of aliases, also.

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

  [
      {
          key: {element: 'Fire'},
          elements: [{name: 'Imp', cost: 1}, {name: 'Dragon', cost: 5}],
          grouping: ['element']
      },
      {
          key: {element: 'Water'},
          elements: [
              {name: 'Bog monster', cost: 2},
              {name: 'Giant turtle', cost: 3}
          ],
          grouping: ['element']
      },
      {
          key: {element: 'Earth'},
          elements: [
              {name: 'Dwarf', cost: 1},
              {name: 'Golem', cost: 3}
          ],
          grouping: ['element']
      },
      {
          key: {element: 'Air'},
          elements: [
              {name: 'Sprite', cost: 1},
              {name: 'Giant eagle', cost: 2},
              {name: 'Djinn', cost: 4}
          ],
          grouping: ['element']
      }
  ]


Computing the average cost of each "element" that a card can have::

  SELECT (GROUP Card BY .element) {
      element := .key.element,
      avg_cost := sum(.elements.cost) / count(.elements),
  } ORDER BY .element

::

   [
       {el: 'Air', avg_cost: 2.3333333333333335},
       {el: 'Earth', avg_cost: 2.0},
       {el: 'Fire', avg_cost: 3.0},
       {el: 'Water', avg_cost: 2.5}
   ]

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

  [
      {name: 'Imp', cost_ratio: 0.3333333333333333},
      {name: 'Dragon', cost_ratio: 1.6666666666666667},
      {name: 'Bog monster', cost_ratio: 0.8},
      {name: 'Giant turtle', cost_ratio: 1.2},
      {name: 'Dwarf', cost_ratio: 0.5},
      {name: 'Golem', cost_ratio: 1.5},
      {name: 'Sprite', cost_ratio: 0.42857142857142855},
      {name: 'Giant eagle', cost_ratio: 0.8571428571428571},
      {name: 'Djinn', cost_ratio: 1.7142857142857142}
  ]

Counting the number of cards in each possible "element", "number of
owners" combination bucket, as well as those things individually::

  SELECT (
    GROUP Card BY .element, nowners := count(.owners)
    GROUPINGS CUBE (element, nowners)
  ) {
      key: {element, nowners},
      num := count(.elements),
      grouping
  } ORDER BY .grouping THEN .key.element THEN .key.nowners;

::

   [
       {key: {element: [], nowners: []}, num: 9, grouping: []},
       {key: {element: 'Air', nowners: []}, num: 3, grouping: ['element']},
       {key: {element: 'Earth', nowners: []}, num: 2, grouping: ['element']},
       {key: {element: 'Fire', nowners: []}, num: 2, grouping: ['element']},
       {key: {element: 'Water', nowners: []}, num: 2, grouping: ['element']},
       {key: {element: [], nowners: 1}, num: 1, grouping: ['nowners']},
       {key: {element: [], nowners: 2}, num: 5, grouping: ['nowners']},
       {key: {element: [], nowners: 3}, num: 1, grouping: ['nowners']},
       {key: {element: [], nowners: 4}, num: 2, grouping: ['nowners']},
       {key: {element: 'Air', nowners: 2}, num: 3, grouping: ['nowners', 'element']},
       {key: {element: 'Earth', nowners: 2}, num: 1, grouping: ['nowners', 'element']},
       {key: {element: 'Earth', nowners: 3}, num: 1, grouping: ['nowners', 'element']},
       {key: {element: 'Fire', nowners: 1}, num: 1, grouping: ['nowners', 'element']},
       {key: {element: 'Fire', nowners: 2}, num: 1, grouping: ['nowners', 'element']},
       {key: {element: 'Water', nowners: 4}, num: 2, grouping: ['nowners', 'element']}
   ]



Backwards Compatibility
=======================

There are no backwards compatibility concerns. ``GROUP`` is already a
reserved keyword in the implementation.

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

  [GROUPINGS <alias>, ... ]   # specify which parameters will
                              # be used to partition the set;
                              # if unspecified, use all aliases declared in BY

  UNION
      <expr>                  # map every grouped set onto a result set,
                              # merging them all with a UNION ALL
                              # (or UNION for Objects)

The meaning of the first three clauses is essentially identical to
above (except that aliases may not be omitted), but with a totally
different scheme for output.


Instead of producing a free object automatically, the output is
produced by an explicit `UNION` clause. Within the `UNION` clause,
the aliases from the `BY` clause are bound to the grouping columns
and the subject alias is bound to the *set* containing all the
elements in the group.
(The subject alias may be omitted if the subject expression is a
simple set name reference, in which case that name is also used as the
alias.)

This is a solid and reasonable design.

One key difference with the main proposal is that is essentially
orthogonal to other language features. This would make it well suited
when presenting a core calculus for EdgeDB, for example, since it can
be considered in isolation, while the shape based proposal depends on
free objects and arrays. The shape-based version can easily be
implemented in terms of this one.

That said, this is also its disadvantage: the shape-based version fits
in nicer with the language and better leverages the other features it
has.

If the shape-based version winds up as unimplementable, though, we
should strongly consider coming back to this one.

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
      grouping: <array of keys used in grouping>,
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


.. [1] https://github.com/edgedb/edgedb/issues/104#issuecomment-344307260

.. [2] https://github.com/edgedb/edgedb/tree/group-proposal
