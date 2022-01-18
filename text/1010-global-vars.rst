::

    Status: Draft
    Type: Feature
    Created: 2022-01-15
    Authors: Elvis Pranskevichus <elvis@edgedb.com>
    RFC PR: `edgedb/rfcs#0049 <https://github.com/edgedb/rfcs/pull/49>`_

==========================
RFC 1010: Global variables
==========================

This RFC proposes adding a mechanism to declare and set global variables
in a schema and set them on per-session basis.


Motivation
==========

The need for non-local data context frequently exists in programming, and the
database is no exception.  EdgeDB schemas offer a large variety of ways to
express the data model with EdgeQL expressions, and the ability to parametrize
these expressions with contextual per-session data will provide even more
power.  Consider the ubiquitous concept of authentication, where there is
a notion of "current user" that is used to filter query results or make other
decisions.  Currently, such data must be passed as an explicit argument to
queries and there is no way to pass such context to schema expressions at all.


Specification
=============

A global variable must be declared before it can be used in any expression.

CREATE GLOBAL
-------------

Define a new global variable.

Synopsis::

    [ WITH <with-item> [, ...] ]
    CREATE [ REQUIRED ] [ MULTI | SINGLE] GLOBAL <name> -> <type> [ "{"
        <subcommand> ; [...]
    "}" ] ;

    # Computed global variable form:

    [ WITH <with-item> [, ...] ]
    CREATE [ REQUIRED ] GLOBAL <name> := <expr> ;

    # where <subcommand> is one of

      SET default := <value>
      RESET default
      RENAME TO <newname>
      USING <expr>
      RESET EXPRESSION
      CREATE ANNOTATION <annotation-name> := <value>
      ALTER ANNOTATION <annotation-name> := <value>
      DROP ANNOTATION <annotation-name>

The ``<type>`` of non-computed globals must be a primitive type, i.e. it
*must not* be an object type or a collection of object types.  This is because
object type variables will require some form of referential integrity checks
like links do (i.e. the ``on target delete`` spec), which complicates the
implementation considerably.  Computed globals can have any type, and so it
is possible to define an object type global in terms of its ``id``, e.g::

    CREATE GLOBAL user_id -> uuid;
    CREATE GLOBAL user := (SELECT User FILTER .id = (GLOBAL user_id));

If a global is declared as ``REQUIRED``, it *must* provide a default value, or,
if the global is computed, the expression must be non-optional.

Explicit cardinality is specified via the ``SINGLE`` or ``MULTI`` keyword,
otherwise the cardinality is assumed to be ``SINGLE`` in non-computed globals,
and is inferred from the expression in computed globals.


ALTER GLOBAL
------------

Alter the definition of a global variable.

Synopsis::

    [ WITH <with-item> [, ...] ]
    ALTER GLOBAL <name>
    "{" <subcommand>; [...] "}" ;

    # where <subcommand> is one of

      SET default := <value>
      RESET default
      RENAME TO <newname>
      USING <expr>
      RESET EXPRESSION
      SET REQUIRED
      DROP REQUIRED
      SET SINGLE
      SET MULTI
      CREATE ANNOTATION <annotation-name> := <value>
      ALTER ANNOTATION <annotation-name> := <value>
      DROP ANNOTATION <annotation-name>


DROP GLOBAL
-----------

Remove a global variable.

Synopsis::

    [ WITH <with-item> [, ...] ]
    DROP GLOBAL <name> ;


SET GLOBAL
----------

Set the value of a non-computed global variable *in the current session*
by evaluating the given expression.

Synopsis::

    SET GLOBAL <name> := <expr> ;


RESET GLOBAL
------------

Reset a non-computed global variable to its default value.

Synopsis::

    RESET GLOBAL <name> ;


Referring to globals in queries
===============================

The new ``GLOBAL <name>`` expression is used to refer to the value of the
given global variable in queries.  For example::

    SELECT User FILTER .id = (GLOBAL user_id)


Implementation
--------------

Non-computed global variables are implemented as implicit query arguments,
i.e global variable references are replaced with a query argument the value
of which is automatically populated from the session state.  Computed globals
are expanded like expression aliases.


Computed globals vs aliases
===========================

While there is a lot of similarity between computed globals and aliases,
they aren't the same and complement each other.  Where an alias usually
stands in place of a material set of objects (and this masquerades as an
object type), a computed global stands in for a settable global.  Additionally,
the alias cardinality is usually ``multi``, whereas most globals would normally
be ``single``.


Backwards Compatibility
=======================

``global`` is already a reserved keyword.  There are no other compatibility
concerns as globals are a new construct.


Security Implications
=====================

There are no security implications.
