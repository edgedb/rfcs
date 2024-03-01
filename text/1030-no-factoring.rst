::

    Status: Draft
    Type: Feature
    Created: 2024-03-01
    Authors: Michael J. Sullivan <sully@msully.net>


=====================================
RFC 1030: Simplifying path resolution
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

(And, of course, you `shouldn't have first_name and last_name
properties anyway
<https://www.kalzumeus.com/2010/06/17/falsehoods-programmers-believe-about-names/>`_)


Motivation
==========

The desire to remove path factoring comes from two directions: a
desire to simplify and improve the language, and from implementation
concerns.

Language Design
---------------


Implementation Concerns
-----------------------

The current implementation of path factoring is the source of
*substantial* technical complexity in the EdgeDB implementation.
Currently, path factoring is performed "on the fly" during the
first compilation phase, from our AST to our IR


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



Backwards compatibility
=======================

This change is explicitly not backwards compatible.  Therefore, a
migration plan is crucial.


Alternatives
============


Frontloaded factoring
---------------------

Maintaining both options in the long term
-----------------------------------------

In the long run, we want to have a unified EdgeQL language and
ecosystem, without needing to document and explain two versions of
this core piece of semantics.
