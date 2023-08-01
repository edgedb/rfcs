::

    Status: Draft
    Type: Feature
    Created: 2022-08-12
    Authors: Victor Petrovykh <victor@edgedb.com>

==========================
RFC 1015: Full text search
==========================

This RFC outlines a generic approach to full-text search (also referred to as
FTS in this document) functionality, keeping the core features the same across
various specific implementations.


Motivation
==========

Advanced text search functionality is important when dealing with text fields
containing natural human language. When filtering such fields simpler tools
such as ``LIKE`` or regular expressions are not flexible enough. Full-text
search tools can be built-in (based on Postgres FTS capability) or external
(e.g. Elasticsearch). External tools would likely be implemented in the form
of extensions. However, we want the EdgeQL queries that use FTS to be the
same, regardless of how the search is implemented. This means that this
functionality should use the same built-in functions and even the query
sub-language should be the same. Additionally, we want to make it easier to
look for entire *objects* matching a particular search query based on all/some
of the ``str`` fields of that object type.

How Postgres does it?
---------------------

It is useful to consider how Postgres handles full-text search (generally
other FTS implementations also support similar features, so this is useful
baseline). Postgres introduces two types for this purpose: ``tsvector`` and
``tsquery``.

The ``tsvector`` type stores a processed version of the text. It basically
contains information about the tokens making up the text as well as their
positions. This is the version of the text that is actually targeted by the
FTS. Postgres also provides a few different ways to convert plain text into a
``tsvector``. There's also a way to create an index that converts a text
column into a ``tsvector`` and enables fast searches. Functions that convert
text into ``tsvector`` can also take an argument that determines which parser
will be used for the conversion.

The ``tsquery`` type encapsulates the search conditions, such as which lexemes
should or must appear and their relative order. Postgres also has several ways
of converting a string into a ``tsquery``.

Postgres provides a way to assign relative weights to different lexemes in the
``tsvector`` which then can be used to rank FTS matches.


Specification
=============

There's a lot of functionality involved in supporting FTS, but it's a fairly
niche thing, so it makes sense to create a ``module std::fts`` for all of the
FTS tools.


Indexing
--------

Indexes can be used to declare which properties should be subject to FTS
functionality. They can also be used to annotate properties with search
groups, language information, etc. Search groups can then be used to fine tune
results by assigning different weights to each group during searches. Language
information would determine what data is targeted based on the language of the
search query. The proposed index definitions would thus have a semantic effect
determining how exactly FTS should be carried out for a given object type.

In EdgeDB we don't need an explicit "document" scalar that users interact
with, because we don't allow arbitrary strings to be processed by FTS
functions, only indexed properties. However, we still need a type similar to
``tsvector`` to record things like weight category and language that
correspond to a particular part of the indexed string. This type should be
opaque and not valid as a result of a user query. Instead it can be used in
index expressions and returned by functions that can only be used in index
expressions. The type being opaque makes it possible to add more FTS functions
that affect how the string is interpreted without having to change all the
signatures.

This processed document type and a couple of functions that use it are::

  type fts::DocVector {
    # This is hidden implementation, so it's more of a concept
    # than an actual object type. Under the hood it might be a
    # record or not even have instances at all (see Implementation).
    required document: str;
    language: str;
    weight_category: str;
  }

  function fts::language(doc: str, lang: str) -> fts::DocVector
  function fts::weight(doc: str, category: str) -> fts::DocVector

  # Both functions can take a str or DocVector so that they can be
  # applied in arbitrary order
  function fts::language(doc: fts::DocVector, lang: str) -> fts::DocVector
  function fts::weight(doc: fts::DocVector, category: str) -> fts::DocVector

Since ``DocVector`` is opaque and is not valid in regular queries, we should
really make it some sort of "internal" or "system" type. It doesn't need to be
part of schema introspection and likely will only appear as part of signatures
of the special index functions. Incidentally, these special functions also
don't need to be exposed to introspection.

*Language* is an important part of FTS and we generally assume it will
be part of the ``index`` declaration as well as of the search functions.

Postgres passes the language as part of the index configuration: ``CREATE
INDEX pgweb_idx ON pgweb USING GIN (to_tsvector('english', body));``.

Elasticsearch has language-specific analyzers which can be used in the mapping
corresponding to a given index.

Algolia has an ``indexLanguages`` setting for its indexes.

Typesense does not appear to have any language-specific index settings.

In order to simplify FTS query language, we will use a ``str`` to represent
queries rather than introducing a new scalar type. This string is expected to
contain FTS search queries in a format similar to how they might be typed into
a search field.


Abstract Index
--------------

We assume that each FTS implementation will supply its own ``index`` with the
particular parameters that are configurable for that implementation (which
could be some backend-specific JSON config, etc.). The general index
definition, however has no parameters and will be of the following form::

  abstract index fts::textsearch()

Additionally there's sometimes a need to index prefixes to facilitate
prefix-based searches (such as for autocomplete). This can be accomplished by
adding a parameter to the ``abstract index`` definition::

  abstract index fts::textsearch(
      named only index_prefixes: bool = false
  )

Setting up a prefix-based index, wouldn't really do much for Postgres.

In Elasticsearch that would translate into ``index_prefixes`` with default
minimum and maximum prefix values.

In Algolia prefix searches are by default enabled for all, but can be disabled
via ``disablePrefixOnAttributes``, which would correspond to not turning on
``index_prefixes``.

In Typesense prefix searching is entirely controlled by the query and no
special index settings are necessary one way or another, much like in
Postgres.

Therefore, if we introduce a generic prefix search FTS functionality we can
add ``index_prefixes`` parameter to the generic abstract index definition.


Concrete Index
--------------

As mentioned earlier concrete indexes on types will specify which properties
the FTS functions should be targeting. When targeting multiple properties they
can be specified as separate index ``on`` parameters or just concatenated into
a single expression. Keeping the parameters separate makes it possible to
assign different weights to them.

For example::

  type Document {
      title: str;
      body: str;
      internal: str;

      index fts::textsearch on (
        fts::language(.title, 'english'),
        fts::language(.body, 'english'),
      );
  }

The above schema declares that only ``title`` and ``body`` properties are
subject to FTS functions, while the ``internal`` property will be ignored by
searches. These properties are wrapped in a ``language`` special function that
specifies the language of the corresponding parts (which affects the
indexing).

In order to add weights, we use the other special function which can only
appear in index specification::

  type Document {
      title: str;
      body: str;
      internal: str;

      index fts::textsearch on (
        fts::weight(fts::language(.title, 'english'), 'A'),
        fts::weight(fts::language(.body, 'english'), 'B'),
      );
  }

The ``fts::weight`` function cannot be used in regular queries (because its
return type is not supposed to be exposed to the user), but in index
specification it associates a weight group with a particular indexed
expression (typically just a property). Using strings to denote weight groups
makes it less likely that the users confuse the *group name* with the actual
associated weight *value*.

The most basic naming scheme for weight groups could be similar to Postgres:
just using the alphabet starting with 'A'. We can have more than 4 groups,
though for the general case. If the backend supports fewer groups than
indicated by index specification, we simply use as many groups as we can
starting with 'A' and merge the rest into the last valid group ('D' for
Postgres).

In principle the group names can be generalized to be any ``str`` and the
lexicographical order can then be used to determine the relative group order
(for purpose of picking group weights). This would allow using Postgres-style
groups 'A', 'B', 'C', 'D' or give groups more descriptive names like '1 - most
relevant', '2 - regular', '3 - marginal'.


Inheritance
-----------

Since ``fts::textsearch`` index is semantic and has effect on how FTS
functions work, we cannot have multiple versions of this index defined on a
single object type. So at most one FTS index can be defined on any type
(subject to ``??`` once the extension enabling/disabling is implemented). In
particular this means that when an FTS index is inherited by a type and a new
index definition is provided in the extending type, the new definition
completely overrides the inherited one.

For example::

  type Document {
      body: str;
      index fts::textsearch on (fts::language(.body, 'english'));
  }

  type InternalDoc extending Document {
      internal: str;
  }

  type TitledDoc extending Document {
      title: str;
      index fts::textsearch on (fts::language(.title, 'english'));
  }

The ``InternalDoc`` type has no FTS index of its own, so it inherits the index
from ``Document``. Thus only the ``.body`` is indexed. Conversely,
``TitledDoc`` defines a new ``fts::textsearch`` index on ``.title``, but the
``body`` property would no longer be indexed for ``TitledDoc`` objects.

The reason for this design is that we're limited to having no more than one
FTS index per type as it covers all search functionality. This means that we
must have a way of overriding an existing index with type-specific one. The
least surprising way of doing that is for the explicitly specified index to
override the inherited one.

In case of multiple inheritance indexes may conflict with one another, that is
considered an error and the user is required to explicitly provide an FTS
index to resolve this (to minimize ambiguity).

All of these extra rules apply only to FTS indexes and not indexes in general.


Search Functions
----------------

An important feature of the search function is that it actually performs the
job typically done by the ``FILTER`` clause. The function is supposed to
filter a bunch of objects based on the FTS query and indexed fields. In
addition to filtering, the results may be annotated with relevance (ranked)
and potentially with highlighted matches::

  function std::fts::search(
      doc: anyobject,
      query: str,
      named only language: str,
      named only weights: optional array<float64> = {},
      named only rank_opts: optional str = 'default',
      named only highlight_opts: optional str = {}
  ) -> optional tuple<
    object: anyobject,
    rank: float64,
    highlights: array<str>
  >

The ``doc`` input provides the object that should be searched. The
details of which properties should be searched and other FTS parameters will
be provided by the FTS index on the specified type. Searching an unindexed
type should simply produce no matches (resulting in an ``{}``).

The ``language`` parameter indicated which language the query is using and
therefore allows to target only the relevant documents if there are multiple
languages.

If ranking was requested (which is the default behavior), then the value will
be written in ``rank``, otherwise the value ``0`` is used.

The optional ``weights`` array provides relevance multipliers corresponding to
each of the weight groups indicated by the FTS index. The weights have to be
in the range from 0 to 1. If the weights are outside of the valid range an
error is produced. There should be a default weights array in case no weights
are provided. By default weights should start with ``1`` and be halved for
every next group, indicating diminishing relevance. The ``weights`` array of
values provided explicitly overrides the corresponding defaults. This means
that first 4 groups would by default have the following weights: ``1``,
``0.5``, ``0.25``, ``0.125``. On the other hand if the ``match`` was called
with ``weights := [1, 0.7]`` as an argument, then the first 4 groups would
have these weights: ``1``, ``0.7``, ``0.25``, ``0.125``. In order to only
search the first 2 weight groups explicit weights of ``0`` need to be
specified for the other groups: ``weights := [1, 0.7, 0, 0]``.

If highlighting the matched text was requested, then the appropriate value
will be written in ``highlights`` array. The array is needed in case the
matches are highlighted separately (e.g. only the matched words are returned,
omitting the surrounding text). The array will be empty if highlighting was
not requested.

The ``rank_opts`` and ``highlight_opts`` can potentially be backend-specific,
however either one of them can take the value of ``{}`` with the meaning that
ranking/highlighting of the results is not necessary. Both of them should also
always accept ``default`` as a valid argument indicating that
ranking/highlighting of the results should be done in some basic manner
(typically using some faster method).

It is worth considering implementing a search function that returns a free
object instead of the named tuple (with the same structure). A free object
would interact more naturally and smoothly with shapes used to get the right
data for the output.

Another type of search functionality involves "autocomplete"-style search for
things matching a certain prefix. This is similar to regular matching, but
typically the last word is treated as a prefix, while the start of the query
matches exactly. Although it's possible to use one of the arguments to switch
to this mode, it's probably better to have a dedicated function instead, since
the actual FTS implementation may need a different call to be compiled to
access the backend::

  function std::fts::autocomplete(
      doc: set of anyobject,
      query: str,
      named only language: str,
      named only weights: optional array<float64>,
      named only rank_opts: optional str = 'default',
      named only highlight_opts: optional str = {}
  ) -> optional tuple<
    object: anyobject,
    rank: float64,
    highlights: array<str>
  >


Implementation
^^^^^^^^^^^^^^

One of the implementation concerns is how inheritance interacts with FTS
indexes and search functionality. The natural interpretation is that all the
different fields should be targeted polymorphically as per individual index
specification. This means that searching the root type of some documents would
be a good way to target all different document varieties without having to
explicitly specify them one by one.

A couple of different approaches can be used to implement this:

1) A hidden ``_document`` column can be added for purposes of Postgres-backed
   FTS functionality. This makes it possible to resolve all the nuances in the
   index specification and provide a uniform way of accessing the searchable
   parts of the documents for FTS functions. The downside is that this is
   duplicating the data that's already present as actual properties and
   indexes.
2) Expand a single search call into several calls, each of them targeting only
   one table. This avoid data duplication, but is much harder to implement.

The ``DocVector`` type is needed to declare the signatures of special
functions like ``weight`` and ``language``, but is never meant to be used
directly in queries. In fact, in all likelihood, it may never have any
instances of actual values, because these values are used exclusively by the
compiler and generally decomposed into their individual components and then
the components are compiled (often as literals) into the specific underlying
indexes.


Query DSL
---------

There's no way to encompass all the features of many different FTS
implementations in a single clear search language spec. Instead we want to
provide a simple search language to cover many common use-cases and to cover
searches that are driven by user input. One of the limitations is that the
search query must be expressible as a ``str``, ideally similar to actual
user-input.

We will follow the format similar to Google search queries:

- Search terms can appear as plain words. These are *acceptable* terms.
- Terms quoted by ``"..."`` must be treated as a phrase (preserving word
  order). These are *highly desirable* terms.
- Terms prefixed by ``-`` are *excluded* terms and they must not appear in the
  matching document at all.
- ``OR`` may appear between any terms. It doesn't affect any *acceptable*
  terms, but it downgrades any adjacent *highly desirable* terms to be now
  *acceptable*. When appearing next to an *excluded* term, it makes that
  exclusion optional (which usually negates its usefulness).
- ``AND`` may appear between any terms. It doesn't affect any *highly
  desirable* or *excluded* terms, but it upgrades an *acceptable* term to be
  *highly desirable*.
- Ideally all *highly desirable* terms must appear in the matching document.
- At least some of the *acceptable* terms must appear in the matching
  document. The more the better, but there's no strict preference for which
  ones get matched.

For example:

- The search string ``quick brown fox jumps`` indicates that as long as the
  document contains any of the three search terms. So ``brown sugar`` or
  ``running foxes``  are valid matches, but ``the quick brown dog jumps over
  the fox`` is a much better match (more matched terms), which should be
  reflected in the rankings.
- The search string ``"quick brown" fox jumps`` indicates that the document
  must contain the phrase "quick brown" and possibly some of the words "fox"
  and "jump" (or their variants). So ``the quick brown dog jumps over the
  fox`` is a valid match, but not ``the fox is quick and brown`` (phrase not
  matched).
- The search string ``quick AND brown fox jumps`` indicates that the document
  must contain the words "quick" and "brown" and and possibly some of the
  words "fox" and "jump". Thus ``the fox is quick and brown`` is a valid match
  and so is ``jump to the quick recipe for brown sugar``.

To map this kind of search query to Postgres backend ``websearch_to_tsquery``
can be used. The main caveat might be that each term is linked with ``&`` in
Postgres, so we may need to inject explicit ``or`` to avoid this.

Elasticsearch has ``operator`` values ``and`` and ``or`` available to specify
whether all terms in a query should be matched or just any of them. There's a
``must_not``, and ``match_phrase`` operation in addition to ``match``. The
query can be nested and have complex structure. Generally the ``str`` query
will need to be parsed and split into this nested JSON structure.

Algolia with ``advancedSyntax`` turned on uses a very similar syntax for
search queries (quoted phrases and ``-`` for negative matches). By default it
looks for all the query terms, but we can use ``optionalWords`` instead to
make searches that don't have to match everything.

Typesense, much like Algolia has a very similar query syntax to the one
proposed above. By default all terms are optional, but the more of them are
matched the better the ranking of the result.

As a general rule FTS implementation will adhere to the above rules on
best-effort basis. The specifics of each backend may affect how strict are the
matches and phrases.


Backwards Compatibility
=======================

There are no backwards compatibility issues because we now have nested
modules.


Security Implications
=====================

There are no security implications.


Rejected Alternative Ideas
==========================

We change the return of ``match`` function from FTSResult to a named tuple.
This simplifies the spec and implementation without loss of functionality.

We rejected the idea of using annotations for specifying the FTS language
settings. This is due to the fact that it's difficult for functions to
properly interface with that and it may result in unnecessary compiler-level
magic.

In general using annotations for fine-tuning FTS functionality impacts the
complexity of the implementation potentially requiring recompiling any user
functions that actually use FTS functionality inside of them. So, at least for
the moment, this is undesirable.

Rename result "boost" into "weight" and make it part of the function signature
rather than an annotation that is magically picked up by the compiler.
Additionally, make the valid range of "weight" to be [0, 1] rather than being
arbitrary and relying on automatic scaling of these values relative to each
other. The benefit of the fixed valid range is that it makes it more clear
whether a particular "weight" value is large or small without needing a larger
context.

The search functions only take objects, not arbitrary ``str`` expressions.
This means that we sacrifice flexibility of arbitrary searches for not having
to worry whether the search expression actually matches the index expression
and therefore whether the FTS index is going to be actually used.

We rejected the ``test`` and ``match`` function duality because they are
low-level and hard to use with filters and pagination, while still using the
index efficiently. In addition boolean ``test`` seems to be a very limited in
terms of usefulness as compared to ``match`` which allows ranking results. A
single ``search`` function can definitely perform the duties of both of them.
Given that we no longer allow searching arbitrary strings and completely rely
on indexes to mark the parts of documents that are subject to FTS, the niche
usefulness of ``test`` is further reduced.

We rejected element-wise ``search`` in favor of a function operating on a set
of objects. This approach is in line with the general operation flow in
EdgeDB, where a query is a pipeline that processes a set gradually
transforming it into the result. Having this as a set function makes it less
awkward for the user to associate ranking and the actual objects and then
filter based on that, by removing the necessity for the user to manually
inject the ranking into the object shape, in order to filter and paginate
based on that. Instead the user now applies the search function to the desired
set of objects and has an annotated result set ready for filtering and
pagination.

The ``language`` parameter should not be omitted from search functions because
there might be multiple indexed languages and the backend needs to know which
one the query targets (and therefore which stemmer to use, etc.).

We rejected ``set of`` documents parameter for the search function, largely
because it's harder to implement and we don't rely of operating on a set of
objects. We considered making the ``search`` function return an *ordered* set
or results, but in the end opted out of it and thus we no longer need this
function to operate on sets.