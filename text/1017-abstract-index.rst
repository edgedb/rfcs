::

    Status: Draft
    Type: Feature
    Created: 2022-10-07
    Authors: Victor Petrovykh <victor@edgedb.com>

========================
RFC 1017: Abstract index
========================

This RFC proposes formalizing a concept of an abstract index. This would be
the mechanism by which Postgres indexes can be exposed and made available to
the users.


Motivation
==========

We need a way to provide more than one type of index in EdgeDB. These
alternative ``index`` definitions could be built-in or come from extensions.
These ``abstract index`` definitions can then be referenced by name in the
user schema when defining concrete indexes.


Specification
=============

The ``abstract index`` definition can follow a similar pattern to ``abstract
constraint`` definition. It must appear as a top-level module element and
serves as a definition of the index name and parameters. In its current state
it would only be possible to define ``abstract index`` during the compilation
of the standard library. Meanwhile the users can only refer to the existing
abstract indexes when defining their concrete indexes in the user schema.

The new abstract indexes can be defined as follows::

  abstract index <name> [ on ( <func-name>, ... ), ... ]
  "{"
      [ <annotation-declarations> ]
      [ using SQL <index-body> ]
      [ ... ]
  "}" ;

The index can be defined as targetting specific operations by providing a list
of ``func-name`` identifies denoting function names or operators. This is
meant to be similar to the operator classes in Postgres and can be used to
infer the types for which the index is valid. A valid indexable type must be
compatible with all of the provided operations. As a rule the type of the
first parameter of the named function or operator determines the valid types
for an index. Potentially several operator groupings may be provided. Each
grouping then determines their own valid indexable types. Omitting the ``on``
clause implies that the index is generic enough to be valid for ``anytype``.
Note that the underlying Postgres implementation actually takes care of the
operator classes for a given index and typically this EdgeQL clause would not
be reflected into the underlying implementation, but rather it is supposed to
be consistent with it.

Some of the details of the SQL index definition can be provided in the
``index-body``. Essentially this is what the ``CREATE INDEX`` command is going
to be based on. The SQL body here is a template that uses ``__col__``
placeholder where the indexed column/expression would noramlly be, other than
that it is intended to contain the rest of the ``CREATE INDEX`` SQL command.

A concrete index can be defined inside any type in the following manner::

  type <Typename> "{"
    index [<name>] on ( <index-expr> )
      [ except ( <except-expr> ) ]
      [ "{" <annotation-declarations> "}" ] ;
  "}"

We add an optional ``name`` that can be provided here when some non-default
index is used. This ``name`` refers to one of the existing abstract indexes.
The rest of the concrete index definition stays as is.


Exposed postgres indexes
------------------------

Postgres has the following built-in indexes: ``hash``, ``btree``, ``gin``,
``gist``, ``spgist``, and ``brin``. Some of these apply universally to all
types and others are more specialized. We want to expose these indexes as part
of a new ``pg`` module. Here's a breakdown of the exposed indexes:

1) ``pg::hash`` - applicable to ``anytype``, pretty much useful for exact
   matches, but not much else. It's generally smaller that more elaborate
   indexes.
2) ``pg::btree`` - applicable to ``anytype`` and the default index used (when
   no name is provided).
3) ``pg::gin`` - applicable to arrays, JSON, and tsvector, however we don't
   seem to use "contains" or "overlap" operators for anything other than
   arrays. It might later be useful for FTS functionallity (because of
   tsvector) once that gets introduced.
4) ``pg::gist`` - applicable to geometry types, tsquery, tsvector, and ranges.
   So currently we should expose it for ``range<anypoint>``. Eventually we may
   want to expose it for FTS functionality as well targetitng ``str``.
5) ``pg::spgist`` - applicable to geometry types, text, and ranges. So we
   should expose it for ``str`` and ``range<anypoint>``.
6) ``pg::brin`` - applicable to many types, but not all so ``anytype`` is not
   a good idea. Instead it is applicable to ``anyreal``, ``bytes``, ``str``,
   ``uuid``, ``datetime``, ``duration``, ``cal::local_datetime``,
   ``cal::local_date``, ``cal::local_time``, ``cal::date_duration``,
   ``cal::relative_duration``, ``range<anypoint>``.


Examples
--------

Here's an example of defining an default index and a ``spgist`` index::

  module default {
    type Post {
      required property title -> str;
      required property body -> str;

      index on (.title);
      index pg::spgist on (.body);
    }
  }

For the purposes of the standard library the following definitions might
apply::

  # this one is applicable to anytype, so no `on` clause is needed
  CREATE ABSTRACT INDEX pg::hash {
    USING SQL $$
      USING hash ((__col__))
    $$;
  }

  # this is applicable to ranges
  CREATE ABSTRACT INDEX pg::gist on (std::overlaps) {
    USING SQL $$
      USING gist ((__col__))
    $$;
  }


Backwards Compatibility
=======================

There should not be any backwards compatibility issues.


Security Implications
=====================

There are no security implications.


Rejected Alternative Ideas
==========================

We rejected the idea that the abstract indexes should specify the valid types
directly. This seems to contradict both how Postgres defines index
applicability and the notion that indexes should work for any type that
supports certain operations, since those operations are basically relevant for
how the indexes are built.