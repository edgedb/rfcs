::

    Status: Draft
    Type: Feature
    Created: 2024-06-13
    Authors: Michael J. Sullivan <sully@msully.net>


=====================================
RFC 1027: Simplifying path resolution
=====================================

This RFC proposes simplifying EdgeQL's `path resolution semantics
<https://www.edgedb.com/docs/edgeql/path_resolution>`_ by eliminating
the "path factoring" behavior in which multiple "sibling" references
to a path are implicitly "factored out" and iterated over.

Since this change is backwards incompatible, a migration plan is presented.

Recap
=====

Path factoring is the mechanism behind queries like::

    db> select User.first_name ++ ' ' ++ User.last_name;
    {'Peter Parker', 'Tony Stark'}

Here, the two references to ``User`` are identified and "factored out"
into an implicit iteration. In a more explicit form, the above query
could be written as::

    for u in User
    select u.first_name ++ ' ' ++ u.last_name;

Factoring attempts to factor *longest* matching paths, so::

    select User.friends.first_name ++ ' ' ++ User.friends.last_name;

is equivalent to::

    for u in User.friends
    select u.first_name ++ ' ' ++ u.last_name;

Two references to a path are not factored if both of them occur in
different sub-scopes.

A sub-scope is introduced by ``SELECT``, ``UPDATE``, ``DELETE``, and
``GROUP`` in their subject, along with separate nested sub-scopes in
any ``FILTER`` and ``ORDER BY`` clauses.  ``LIMIT`` and ``OFFSET``
clauses also introduce sub-scopes, but those sub-scopes are a sibling
of the sub-scope of the subject of the ``SELECT``, not a child.  They
are also introduced by ``FOR`` in both the ``IN`` clause and the body
and in any bindings defined by ``WITH``.  If ``SELECT`` has a result
alias (``SELECT X := ...``), then the subject is in another nested
sub-scope (one that is a sibling to any clauses) while the alias
(``X``, here) is in the normal ``SELECT`` sub-scope. Finally, a
sub-scope is introduced in each element of a set literal and when
calling a function or using an operator with a ``SET OF`` argument,
such as ``count``, ``sum``, ``assert_exists``, ``array_agg``,
``EXISTS``, ``UNION``, and many more.

Thus::

    db> select (select User.first_name) ++ ' ' ++ (select User.last_name)
    {'Peter Parker', 'Peter Stark', 'Tony Parker', 'Tony Stark'}

    db> select {User.first_name} ++ ' ' ++ {User.last_name}
    {'Peter Parker', 'Peter Stark', 'Tony Parker', 'Tony Stark'}

    db> select (select User.first_name) ++ ' ' ++ {User.last_name}
    {'Peter Parker', 'Peter Stark', 'Tony Parker', 'Tony Stark'}

But::

    db> select (select User.first_name) ++ ' ' ++ User.last_name
    {'Peter Parker', 'Tony Stark'}


This recap is the first time several of these things have been documented.

The most idiomatic way to fetch such data in EdgeQL, however,
remains::

    db> select User { name := .first_name ++ ' ' ++ .last_name }
    {User {name: 'Peter Parker'}, User {name: 'Tony Stark'}}

(And, of course, you probably `shouldn't have first_name and last_name
properties anyway
<https://www.kalzumeus.com/2010/06/17/falsehoods-programmers-believe-about-names/>`_)


Motivation
==========

The desire to remove path factoring comes from two directions: a
desire to simplify and improve the language, and from implementation
concerns.

Language Design
---------------

The path factoring behavior is quite complex, and makes it difficult
to understand the behavior of a query at a glance. Furthermore, it
badly compromises several of the intended design principles of EdgeQL.

EdgeQL aims to support a "top-to-bottom" reading, but path factoring
means that code later in the query can fundamentally alter the
meaning.

Consider::

    db> select (
    ...   obj := {
    ...     matching := (select User filter .name like <str>$pattern),
    ...   },
    ...   nickname := User.nickname,
    ... )

Here, the meaning of the free object being assigned to the ``obj``
tuple field is completely changed by the presence of the reference to
``User.nickname``. Without the reference, ``matching`` would infer as
multi and would contain every matching object. With that reference,
``matching`` infers as single and will contain the object referenced
in the ``nickname`` field when it matches the pattern.

This is weird, and violates the goal of supporting a "top-to-bottom"
reading.

Another (closely related) way to think of this problem is that it is a
failure of compositionality. In a compositional language, the meaning
of an expression ought to be understandable from the body of the
expression and from its enclosing context of variable definitions
alone. This is clearly not the case with path factoring, as path
references in sibling (... and "distant cousin") expressions can
fundamentally alter the meaning.

Worse, there is not a clean way to "opt out" of path factoring in
order to create a "fully compositional" expression. The ``DETACHED``
keyword can be used, but that does too much: it discards the enclosing
variable bindings as well.

Weird error messages
####################

Consider this query, simplified from a genuine user query from a bug
report::

  with name := <str>$0,
  select if name like 'Bot_%' then
    (insert Bot { name := name })
  else
    (insert User { name := name })

It fails with ``InvalidReferenceError: cannot reference correlated set
'name' here``.

The fix is to wrap the ``name`` in the if condition with a ``select``.
I'm not sure how we could possibly make that make sense to a user.

TODO: "cannot reference correlated set" and "changes the
interpretation of ... elsewhere in the query" are bizarre error
messages but I'm not sure if there is any message that would make
users understand them


Implementation Concerns
-----------------------

The current implementation of path factoring is the source of
*substantial* technical complexity in the EdgeDB implementation.
Currently, path factoring is performed "on the fly" during the
first compilation phase, from our AST to our IR.

The output of the EdgeQL->IR compilation phase is not just the main IR
expression, but also a "scope tree" that contains scope nodes for each
sub-scope and binding points for every path used.
When a reference to a path is compiled, we attach it to the scope tree
in the current sub-scope; as part of this process, we search for any
prefix of the path that is "visible" elsewhere in the tree, and if so
we attach the path to the common ancestor in the tree.

Anything consuming IR must understand both the IR expressions
themselves and the scope tree to interpret the meaning correctly.

As mentioned above, the meaning of an expression can not be understood
solely by analyzing the expression and its enclosing context.  This
means cardinality and multiplicity can not be inferred or checked
until the full query is compiled. The parts of materialization that
depend on computing visibility must also be deferred until the fully
query is compiled, which causes many problems.

Maintaining the correct path factoring and scoping during complex
"desugaring" translations in the QL->IR compiler always substantially
complicates things and has been a recurring source of bugs in casts
and other places.

Path factoring introduces substantial complexity in the IR->PG
compiler.  In order to compile factored paths in the correct locations
in the SQL query, we end up needing to "jump around" in the SQL
tree. This makes following and understanding the flow of the IR->PG
quite difficult at times.

Some factoring/scoping related issues
#####################################

* Fix two issues directly reading pointers from a group
  (`#7130 <https://github.com/edgedb/edgedb/pull/7130>`_)
* Fix issues with cached global shapes and global cardinality inference
  (`#7062 <https://github.com/edgedb/edgedb/pull/7062>`_)
* Don't leak objects out of access policies when used in a computed global
  (`#6926 <https://github.com/edgedb/edgedb/pull/6926>`_)
* Fix DML coalesce inside of IF/ELSE
  (`#6917 <https://github.com/edgedb/edgedb/pull/6917>`_)
* Partial rework of how lprop scope tree visibility works
  (`#6775 <https://github.com/edgedb/edgedb/pull/6775>`_)
* Fix issues with empty sets leaking out of optional scopes
  (`#6747 <https://github.com/edgedb/edgedb/pull/6747>`_)
* Fix some bugs involving union and coalescing of optional values
  (`#6590 <https://github.com/edgedb/edgedb/pull/6590>`_)
* Fix inserts silently failing when a json->array handles 'null'
  (`#6544 <https://github.com/edgedb/edgedb/pull/6544>`_)
* Fix coalesced DML in FOR loops over objects
  (`#6526 <https://github.com/edgedb/edgedb/pull/6526>`_)
* Fix scope bugs in SET ... USING statements
  (`#6267 <https://github.com/edgedb/edgedb/pull/6267>`_)
* Fix use of certain empty sets from multiple optional arguments
  (`#5990 <https://github.com/edgedb/edgedb/pull/5990>`_)
* Enable compiling function arguments into subqueries for pgvector opt purposes
  (`#5615 <https://github.com/edgedb/edgedb/pull/5615>`_)
* Fix some obscure optional bugs in the presence of tuple projections
  (`#5610 <https://github.com/edgedb/edgedb/pull/5610>`_)
* Fix an optional scoping bug with important access policy implications
  (`#5575 <https://github.com/edgedb/edgedb/pull/5575>`_)
* Fix a category of confusing scoping related bugs in access policies
  (`#4994 <https://github.com/edgedb/edgedb/pull/4994>`_)
* Fix accessing tuple elements on link properties
  (`#4811 <https://github.com/edgedb/edgedb/pull/4811>`_)
* Fix several issues that manifest when using GROUP BY
  (`#4549 <https://github.com/edgedb/edgedb/pull/4549>`_)
* Fix correlation issue related to factoring_allowlist
  (`#4525 <https://github.com/edgedb/edgedb/pull/4525>`_)
* Fix computed global scoping behavior
  (`#4394 <https://github.com/edgedb/edgedb/pull/4394>`_)
* Fix issue when type injecting some nested DML cases
  (`#4156 <https://github.com/edgedb/edgedb/pull/4156>`_)
* Fix a scope leak that caused miscompiles
  (`#3912 <https://github.com/edgedb/edgedb/pull/3912>`_)
* Fix some scoping issues for singleton set literals
  (`#3883 <https://github.com/edgedb/edgedb/pull/3883>`_)
* Fix IN array_unpack for bigints
  (`#3820 <https://github.com/edgedb/edgedb/pull/3820>`_)
* Fix a collection of nested shape path reference issues
  (`#3700 <https://github.com/edgedb/edgedb/pull/3700>`_)

Specification
=============

Path factoring will be removed::

    db> select User.first_name ++ ' ' ++ User.last_name;
    {'Peter Parker', 'Peter Stark', 'Tony Parker', 'Tony Stark'}


Certain behaviors that can currently be explained using path-factoring
will be retained.

When applying a shape to a path (or to a path that has shapes applied
to it already), the path will be still be bound inside computed
pointers in that shape::

    db> select User {
    ...   name := User.first_name ++ ' ' ++ User.last_name
    ... }
    {User {name: 'Peter Parker'}, User {name: 'Tony Stark'}}


When doing ``SELECT``, ``UPDATE``, or ``DELETE``, if the subject is a
path with zero or more shapes applied to it, the path will still be
bound in ``FILTER`` and ``ORDER BY`` clauses::

    db> select User {
    ...   name := User.first_name ++ ' ' ++ User.last_name
    ... }
    ... filter User.first_name = 'Peter'
    {User {name: 'Peter Parker'}}


Thus, the ``DETACHED`` keyword is sadly still meaningful, though we
should typically recommend using ``WITH`` bindings of new names
instead (if we wanted, we could drop it and require doing that)::

    db> select User { names := detached User.first_name };
    {
      default::User {names: {'Peter', 'Tony'}},
      default::User {names: {'Peter', 'Tony'}},
    }



Remaining problems
------------------

Link properties
###############

The current area where using link properties is probably the most
idiomatic way to do something is when doing operations on link
properties. Consider this query which returns every
``schema::Operator`` with an annotation named ``std::identifier`` with
the value ``'minus'``::

    WITH
        X := schema::Operator
    SELECT
        X { name }
    FILTER
        X.annotations.name = "std::identifier" AND X.annotations@value = 'minus'

This doesn't work anymore if we get rid of path factoring.

Converting it to use a ``FOR`` loop in the ``FILTER`` doesn't work
either, because the variable bound in the ``FOR`` loop won't have
access to the link properties.

This works::

    WITH
        X := schema::Operator
    SELECT
        X {
            name,
            matches := (X.annotations { b := (X.annotations.name = "std::identifier" AND X.annotations@value = 'minus') }).b,
        }
    FILTER
        .matches

but is kind of awful, and lots of sensible variations (like inlining
the definition of matches into the ``FILTER``) are currently buggy.

We need to fix those bugs either way, but I think we should also
upgrade ``FOR`` to work in this case::

    WITH
        X := schema::Operator
    SELECT
        X { name }
    FILTER
        (FOR ann in X.annotations UNION (ann.name = "std::identifier" AND ann@value = 'minus'))


ORDER BY
########

Some queries producing tuples and doing an ORDER BY on something not
in the tuple won't be easily expressable anymore without using free
objects.
For example, the query (on our cards schema)::

    SELECT (User.name, User.deck.name)
    ORDER BY User.name THEN User.deck.cost

returns tuples of user names and names of the cards in their deck,
ordered in part by the cost of the cards. Doing this once path
factoring is removed is made more difficult.

There is a nice seeming approach that does not work::

    FOR u in User
    FOR d in u.deck
    SELECT (u.name, d.name)
    ORDER BY u.name THEN d.cost


This breaks because the ``ORDER BY`` is *inside* the ``FOR`` loops.

One way to do it with the existing implementation is::

    SELECT (SELECT (
      FOR u in User
      FOR d in u.deck
      SELECT { out := (u.name, d.name), cost := d.cost }
    ) ORDER BY .out.0 THEN .cost).out

I'm unsure of how serious of a problem this will be.
This sort of query is not idiomatic anyway.

Two (relatedish) possible solutions include:
 * Explicitly generalize ``FOR`` to allow multiple iterators. Add a
   new construct for allowing ``ORDER BY`` on ``FOR`` to be
   specified. (I have an implementation of this already from way
   back.)
 * Declare that when we have a chain of ``FOR`` statements with a
   ``SELECT`` with an ``ORDER BY`` as the body, the ``ORDER BY`` is
   evaluated "outside" of the ``FOR`` statements.


Backwards compatibility
=======================

This change is explicitly not backwards compatible.  Therefore, a
migration plan is crucial.

Overview
--------
EdgeDB 6.0 allow opting in to the new behavior, while still supporting
the old behavior fully.

We will also provide an opt-in mode that produces an error (or a
warning, if we have time to build a warning system) when we detect
that a query might change its behavior under the new semantics.

EdgeDB 7.0 will drop support for path factoring.  EdgeDB 6.0 will be
an LTS release, so users will have a fair amount of time to get their
migration in order. (I expect it will very simple for most users.)

The RFC author, Sully, will be the 6.0 release manager.


Details
-------

We will introduce a new "future feature" named ``simple_scoping``
along side a configuration setting also named ``simple_scoping``.
The future feature presence will determine which behavior is used
inside expressions within the schema, as well as serve as the default
value if the configuration value is not set. The configuration setting
will allow overriding the presence or absence of the feature.

We will do the same with a ``warn_old_scoping`` flag that will produce
an error when path factoring is depended upon.

The CLI will put ``using feature simple_scoping;`` in new 6.x projects
by default (like we did with ``nonrecursive_access_policies``).

Starting in 7.0, we will produce a warning or an error when
setting these configuration values.

The rationale for this approach is that we need it to be configurable
within the schema in order to control the behavior within the schema
itself. It is also important to be able to configure it on a session
level, so that applications may be gradually migrated.
This leads to using an in-schema "future feature" as the baseline
default configuration value, while allowing it to be overridden
through the configuration system.

TODO: an example.


Implementation
==============

The implementation of ``simple_scoping`` is easy, and consists of
wrapping paths in an extra fence inside the compiler when needed.

The implementation of ``warn_old_scoping`` uses a similar trick, and
some simple analyses.

Eventually we'll want to start tearing out path factoring and taking
advantage of the new simpler rules, which will be more involved but
not on critical paths.

The hardest thing will be recompiling schema expressions when the
future is created or dropped.


Alternatives
============

Frontloaded factoring
---------------------

It may be possible to implement path factoring as a standalone pass,
prior to main compilation. This would not address any of the language
design issues of path factoring, but would retain backwards
compatibility.

It would allow us to remove most of the path-factoring induced
complexity from other phases, and centralize it in one hopefully
simpler phase.

The formalism takes an approach similar to this, though there are a
number of subtleties involved in extending it to the full language. It
may wind up being necessary to also extract type checking as its own
phase, prior to path factoring, which would make it a substantially
more difficult project.

Maintaining both options in the long term
-----------------------------------------

We could maintain the configurable behavior forever.

This would accomplish *some* of the language design benefits... as
long as the user is using the new mode.

In the long run, we want to have a unified EdgeQL language and
ecosystem, without needing to document and explain two versions of
this core piece of semantics.

Eliminate link deduplication also
---------------------------------

Currently, doing ``User.friends`` will deduplicate the result:
returning each object that is linked to by any ``friends`` link,
without duplicates. This behavior does not apply if a link property is
accessed immediately after, so ``User.friends@nickname`` does *not* do
deduplication.

This deduplication behavior means that otherwise totally sensible seeming
queries like ``User.friends { name, @nickname }`` are not allowed.

I do not like this behavior and would like to get rid of it, but it
feels like a more breaking change to me.
