::

    Status: Draft
    Type: Feature
    Created: 2022-12-15
    Authors: Victor Petrovykh <victor@edgedb.com>

===========================
RFC 1018: Extending indexes
===========================

This RFC introduces a parametrized indexes and the possibility of extending
indexes so that users may define their own abstract indexes with customized
settings.


Motivation
==========

Some indexes are complex enought that they require tuning. We want to
introduce index parameters defined in teh abstract indexes for this purpose.
Additionally, it should be possible for users to define their own versions of
existing abstract indexes to customize some of the parameter values. In
particular this would allow slightly different flavours of the same basic
index type to be used for different types or properties of the same type.


Specification
=============

The ``abstract index`` definition can follow a similar pattern to ``abstract
constraint`` definition in regards to having parameters.

One important distinction of the abstract index definitions is that new index
parameters can only be introduced for a base index (i.e. an index that is not
``extending`` another index). Conversely any indexes that use ``extending``
clause cannot introduce any new parameters, instead they can provide specific
values for parameters they inherit. This ensures that new index types
can define any parameters they need, while making it possible for users to
customize these generic indexes for specific purposes.

The new indexes can be defined as follows::

  abstract index <name> [ ( named only [<argspec>] [, ...] ) ]
      [ on ( <func-name>, ... ), ... ]
  "{"
      [ <annotation-declarations> ]
      [ using SQL <index-body> ]
      [ ... ]
  "}" ;

  where <argspec> is:

  <argname>: [ <typequal> ] <argtype> [ = <default-val> ]

  <typequal> is:

  [ { set of | optional } ]

All the index paramters have to be ``named only`` so that they can be
overridden individually when extending an abstract index.

One way to customize indexes is by defining new abstract indexes extending
some generic index and specifying concrete parameter values for them. In
principle it would also be possible to combine several such abstract indexes
defining different parameters as mixins for a single new abstract index. So
extending provides a way of composition for different parameters combinations.

The ``extending`` indexes definition::

  abstract index <name> [ ( [<argname> := <const-expr>] [, ...] ) ]
      extending <index-name> [, ...]
  "{"
      [ annotation-declarations ]
      [ ... ]
  "}" ;

The ``extending`` index can define specific constant values for any
``argname`` parameter that is inherited from the parent abstract indexes. This
mechanism can be used to override the specific parameter values given by the
parent indexes. Each of the parameter values tracks their inherited value
independently of other parameters. This means that any given inheritance layer
may potentially override a single parameter different from other layers and
thus combine all these different parameter values.

A concrete index definition may now reference these additional user-created abstract indexes just like any other abstract index.


Composability
-------------

One way of composing the indexes is by defining intermediate abstract indexes
while setting the defaults for some of the parameters. This works similar to
partial function closures in other languages and allows users to factor out
some common index configuration.

Each parameter should inherit the default value according to the same
inheritance resolution strategy as is applied to types. Each parameter should
track inherited default value independently from other index parameters. This
allows greater flexibility for abstract index composition, while still
preserving fairly intuitive expectations of defaults for the users.

Sometimes the configuration cannot be neatly split between the parameters (in
case we need to partially define an ``array`` or ``json``). For those cases
the solution is to instead define constant valued aliases and compose them to
form the desired index parameters of *concrete* indexes. We already track
whether an expression is constant or not so that we can determine valid
definitions of things like ``default``. This mechanism might need to be
expanded a bit to handle some simple operators (like element access for arrays
and json) in order to fully take advantage of this method of index parameter
composition.


Examples
--------

Here's an example of a possible usage of abstract indexes to compose several
FTS features as needed::

  using extension es;

  module default {
    abstract index stopwords_mixin(
      filter := to_json('["english_stop"]')
    ) extending es::fts_index;

    abstract index lowercase_mixin(
      filter := to_json('["lowercase"]')
    ) extending es::fts_index;

    abstract index htmlfilter_mixin(
      char_filter := to_json('["html_strip"]')
    ) extending es::fts_index;

    # Composing the abstract indexes into what we will use.
    abstract index title_index extending htmlfilter_mixin, lowercase_mixin;
    abstract index body_index extending htmlfilter_mixin, stopwords_mixin;

    type Post {
      required property title -> str;
      required property body -> str;

      index title_index on (.title);
      index body_index on (.body);
    }
  }

Example of using constants to compose configuration for FTS indexes::

  using extension es;

  module default {
    alias eng_stop := to_json('["english_stop"]');
    alias lowercase := to_json('["lowercase"]');

    abstract index my_index(
      char_filter := to_json('["html_strip"]')
    ) extending es::fts_index;

    type Post {
      required property title -> str;
      required property body -> str;

      index my_index(filter := lowercase) on (.title);
      index my_index(filter := eng_stop ++ lowercase) on (.body);
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

We rejected the idea that only parameters that aren't already given explicit
values (via ``extending`` definition) can be specified here. It was intended
to avoid accidentally breaking the general rules provided by the abstract
indexes defining those parameters. In practice it could be far too restrictive
and it would also change how *defaults* work.

We rejected the idea of special syntax that would allow referring to inherited
default values for parameters when defining the parameter expression. This was
deemed to much of a cognitive burden by both introducing a new syntax and by
having index parameter expressions follow slightly different rules than all
other expressions. The idea was dropped in favour of inheriting default
values.

We rejected the idea of inheriting all parameters as a whole based on the
index inheritance. The benefit was that of being a simpler inheritance
mechanism, the drawback was that it made composition much harder.