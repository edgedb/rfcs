::

    Status: Draft
    Type: Feature
    Created: 2022-12-15
    Authors: Victor Petrovykh <victor@edgedb.com>

========================
RFC 1019: Index fallback
========================

This RFC proposes a mechanism for defining fallback options for indexes in
case of extensions that provide index functionality are disabled.


Motivation
==========

We need a way to provide more than one type of index in EdgeDB. These
alternative ``index`` definitions could be built-in or come from extensions.
Different index-providing extensions may be enabled/disabled depending on
whether EdgeDB is currently in development or production environemnt. We want
to provide a fallback mechanism that allows to keep the same schema for these
different environments.


Specification
=============

We want to provide a way of grouping several abstract indexes into one. The
purpose of this is to provide fallback index implementations when the
preferred one is not available. Such may be the case of full-text search
indexes as the FTS functionality may be provided by a variety of backends. A
particular FTS extension may be disabled in development environment, but
enabled in production. This mechanism for providing fallback for disabled
indexes allows to gracefully handle the differences in indexes without having
to change large portions of the schema. The syntax for this mechanism is
similar to defining a new abstract index and does not directly allow extending
existing indexes. The point is to define a new abstract index in terms of
existing implementations::

  abstract index <name> [ ( named only [<argspec>] [, ...] ) ]
      using <index-name> [ ( [<argname> := <const-expr>] [, ...] ) ]
            [?? ...]
  "{"
      [ <annotation-declarations> ]
      [ ... ]
  "}" ;

  where argspec is:

  <argname>: [ <typequal> ] <argtype> [ = <default-val> ]

  typequal is:

  [ { set of | optional } ]

The ``using`` clause specifies the underlying implementation. It is possible
to supply a ``??``-separated list of index implementations. The ``??``
operator here indicates that it's a kind of fallback mechanism. In that case
the first implementation in the coalesced sequence that is enabled will
actually be used, while other implementations will be ignored. Note that this
``??`` is not really an operator in this case, but a syntactical feature.

The fallback abstract index may declare its own parameters which can them be
referenced in the ``using`` clause by providing them as values for the
underlying indexes. This is necessary because the underlying indexes may not
have the same parameter names or might have parameters of the same name, but
incompatible types and so a unifying layer is needed.

This type of fallback abstract index can be referenced in a concrete index
like any other abstract index.


Examples
--------

Here's an example of defining an FTS fallback index::

  using extension fts;
  using extension es;

  module default {
    # defining an abstract FTS index that relies on
    # elasticsearch by default and falls back to
    # Postgres FTS implementation.
    abstract index my_fts
      using es::fts_index ?? fts::pg_index;

    type Post {
      required property title -> str;
      required property body -> str;

      index my_index on (.title);
      index my_index on (.body);
    }
  }


Backwards Compatibility
=======================

There should not be any backwards compatibility issues.


Security Implications
=====================

There are no security implications.


Rejected Alternative Ideas
==========================

We rejected the idea of using a comma-separated list for fallback options in
favour of using a ``??`` to indicate more clearly the intent. The ``??``
operator already has the semantics of "use the first non-empty value", which
is similar to what we mean when we provide a fallback list. The point is to
have a syntax that signals to the user more clearly the purpose of this
abstract index. This would also be consistent with some possible future where
returning functions is possible and then similar constructs might even be part
of other EdgeQL expressions that pick a function to be used.