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
contains information about the tokens making up the text well as their
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
niche thing, so it makes sense to create a ``module fts`` for all of the FTS
tools.

In EdgeDB we should avoid having an explicit "document" scalar like
``tsvector``. Instead we can rely on various parameters, annotations and
indexes to specify how the plain text ``str`` should be parsed into tokens.
This will allow us to keep the queries involving FTS mostly implementation
agnostic. Some FTS backends may have functions specific to them, but the goal
is to provide good coverage of common FTS functionality.

In order to simplify FTS query language, we will use a ``str`` to represent
queries rather than introducing a new scalar type. This string is expected to
contain FTS search queries in a format similar to how they might be typed into
a search field.

The searching functionality can be split along two axis:
1) what FTS operates on (``str`` vs ``Object``)
2) what FTS returns (``bool`` vs some ranked results)

The task of testing whether a document matches an FTS query would be performed
by the following built-in functions::

  function fts::test(doc: str, query: str) -> bool
  function fts::test(doc: Object, query: str) -> bool

The input can be either a ``str`` or an entire ``Object``. The object type
must have some ``str`` properties, however. Trying to perform FTS on a type
without ``str`` properties is an error. This function is generally intended to
be used in ``filter`` clauses in cases where there's no need for hinting what
matched, but the results are simple enough to be self-explanatory.

The ranked results need to be rich objects with several relevant properties,
akin to free objects used for returning ``group`` results. The difference is
that we cannot use a ``FreeObject`` as a return type of a function, we need a
specialized type for that: ``FTSResult``. This type is defined as follows::

  type fts::FTSResult {
    property rank -> float64 {
      default := 0;
    }
    property highlights -> array<str> {
      default := [];
    }
  }

If ranking was requested, then the value will be written in ``rank``.

If highlighting the matched text was requested, then the appropriate value
will be written in ``highlights`` array. The array is needed in case the
matches are highlighted separately (e.g. only the matched words are returned,
omitting the surrounding text).

These enhanced FTS matches would be performed by the following built-in
functions::

  function fts::match(
    doc: str,
    query: str,
    rank_opts: optional str = 'default',
    highlight_opts: optional str = {}
  ) -> FTSResult

  function fts::match(
    doc: Object,
    query: str,
    rank_opts: str = 'default',
    highlight_opts: optional str = {}
  ) -> FTSResult

The ``rank_opts`` and ``highlight_opts`` can potentially be backend-specific,
however either one of them can take the value of ``{}`` with the meaning that
ranking/highlighting of the results is not necessary. Both of them should also
always accept ``default`` as a valid argument indicating that
ranking/highlighting of the results should be done in some basic manner
(typically using some faster method).

Another type of search functionality involves "autocomplete"-style search for
things matching a certain prefix. This is similar to regular matching, but
typically the last word is treated as a prefix, while the start of the query
matches exactly::

  function fts::autocomplete(
    doc: str,
    query: str,
    rank_opts: optional str = 'default',
    highlight_opts: optional str = {}
  ) -> FTSResult

  function fts::autocomplete(
    doc: Object,
    query: str,
    rank_opts: str = 'default',
    highlight_opts: optional str = {}
  ) -> FTSResult


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
  terms, but if it downgrades any adjacent *highly desirable* terms to be now
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
  must contain the phrase "quick brown" and at least one of the words "fox"
  and "jump" (or their variants). So ``the quick brown dog jumps over the
  fox`` is a valid match, but not ``the fox is quick and brown`` (phrase not
  matched).
- The search string ``quick AND brown fox jumps`` indicates that the document
  must contain the words "quick" and "brown" and at least one of "fox" or
  "jump". Thus ``the fox is quick and brown`` is a valid match and so is
  ``jump to the quick recipe for brown sugar``.

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

Indexing
--------

A big part of efficient FTS is having a good index on the fields that need to
be searched. It is not strictly speaking necessary, but is often desirable.

We assume that fine-tuning of the FTS implementations will be done by some
separate configuration that also allows to select which FTS implementation to
use. All we need to specify in the schema is that the particular properties
need to be indexed for FTS::

  index on ( <index-expr> )
  [ except ( <except-expr> ) ]
  [ using fts ]
  [ "{" annotation-declarations "}" ] ;

The ``<index-type>`` specifies that this is an FTS index as opposed to the
default EdgeDB index.

One important parameter of FTS is *language*, so we need a way to specify that
when creating an index. We can do that with a special annotation::

  abstract annotation fts::language;

It seems that requiring to repeat this annotation for each index would be
burdensome, so ideally it should be possible to set it at the type level as
well as at the index level.

Postgres passes the language as part of the index configuration: ``CREATE
INDEX pgweb_idx ON pgweb USING GIN (to_tsvector('english', body));``.

Elasticsearch has language-specific analyzers which can be used in the mapping corresponding to a given index.

Algolia has an ``indexLanguages`` setting for its indexes.

Typesense does not appear to have any language-specific index settings.

Additionally there's a common need to index prefixes to facilitate
prefix-based searches (such as for autocomplete). This can be accomplished by
adding an annotation to the index::

  type Post {
    annotation fts::language := 'English';

    required property title -> str;
    index on (.title) using fts {
      annotation fts::index_prefixes := 'true';
    }

    required property content -> str;
    index on (.content) using fts;
  }

Setting up a prefix-based index, wouldn't really do much for Postgres.

In Elasticsearch that would translate into ``index_prefixes`` with default
minimum and maximum prefix values.

In Algolia prefix searches are by default enabled for all, but can be disabled
via ``disablePrefixOnAttributes``, which would correspond to not turning on
``fts::index_prefixes``.

In Typesense prefix searching is entirely controlled by the query and no
special index settings are necessary one way or another, much like in
Postgres.

In general, index annotations may be used by some of the specific FTS
implementations to further tweak how those indexes.


Boosting results
----------------

Often in FTS matches in different fields have different relevance. A system
that boosts some results ranking is therefore needed to account for this in
EdgeDB. However, the system of weights/boosts is implemented very differently
across the various FTS engines and these differences are not necessarily easy
to reconcile. In particular, some FTS systems may have limited number of boost
levels. We therefore would need to map the boosts as specified in EdgeDB
schema as best we can to the specific backend.

The boost value is basically a hint of how much search matches in a certain
field should be prioritized compared to other fields. This value can be
specified by adding an annotation to the relevant properties::

  type Post {
    annotation fts::language := 'English';

    required property title -> str;
    index on (.title) using fts {
      annotation fts::index_prefixes := 'true';
    }
    annotation fts::boost := '2';

    required property content -> str;
    index on (.content) using fts;
  }

This feature is generally only relevant to whole-object searches. The details
of how exactly the results get prioritized will depend on specific backends.


Boost implementations
^^^^^^^^^^^^^^^^^^^^^

In Postgres there are only four weight categories (``A``, ``B``, ``C`` and
``D``), but they can be assigned arbitrary numerical weight values for
individual queries.

In Elasticsearch ``boost`` is part of the query parameters and can be
specified on a per-field basis when performing a ``match``::

  GET /_search
  {
    "query": {
      "bool": {
        "should": [
          { "match": {
              "title":  {
                "query": "War and Peace",
                "boost": 2
          }}},
          { "match": {
              "author":  {
                "query": "Leo Tolstoy",
                "boost": 2
          }}},
          { "bool":  {
              "should": [
                { "match": { "translator": "Constance Garnett" }},
                { "match": { "translator": "Louise Maude"      }}
              ]
          }}
        ]
      }
    }
  }

In Algolia custom ranking can be based on just arbitrary attributes, but
rather than serving as a rank multiplier it instead serves as a sorting
criterion when matches are found.

Typesense has a ``query_by_weights`` option where weights are in the 0-127
range, which can be used together with ``query_by`` option that specifies
fields.

What this seems to suggest is that we map boost values to the specific
underlying implementation on a best effort basis. The idea is that boosts are
approximate hints in general and so being treated in a fuzzy manner should be
acceptable in practice for most applications.

Consider this example with just 2 boosted properties::

  type Post {
    required property title -> str {
      annotation fts::boost := '7';
    }

    required property content -> str {
      annotation fts::boost := '2.3';
    }
  }

The Postgres FTS implementation can then assign categories ``A`` and ``B`` to
the 2 properties and specify the following weight array for searches: ``{0, 0,
2.3, 7}``. This should produce the desired effect of ranking matches in the
``title`` proportionately higher than matches in ``content``.

Elasticsearch backend might instead normalize these weights to fit the 0-10
integer range and internally use ``10`` and ``3`` (that's 10 * 2.3 / 7 rounded
to closest integer) instead. It's not an exact weight proportion, but very
close in practice.

Algolia can use the values directly for custom ranking.

Typesense can add ``query_by: title, content`` and ``query_by_weights: 7,
2.3`` to the search queries on this object type.

In a situation when there are more properties with distinct weights in the
schema than the particular FTS engine supports we can then normalize the
schema weights to the closest available weight that the engine supports.

In case of Elasticsearch this would still be the same process as demonstrated
above for 2 weights. In the case of Postgres, the process would involve
identifying 4 clusters based on how far the given weights are from each other
and then using an average in each group.


Backwards Compatibility
=======================

There's some small chance of a name clash between an annotation used for FTS
purposes and an existing annotation. This is mostly mitigated if all the FTS
stuff resides in its own module.


Security Implications
=====================

There are no security implications.


Rejected Alternative Ideas
==========================

. . .