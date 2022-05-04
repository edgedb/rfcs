::

    Status: Draft
    Type: Feature
    Created: 2022-05-02
    Authors: Victor Petrovykh <victor@edgedb.com>
    RFC PR: `edgedb/rfcs#0054 <https://github.com/edgedb/rfcs/pull/57>`_

=====================
RFC 1012: Range types
=====================

This RFC proposes adding ``range`` and ``multirange`` scalar types to EdgeQL.
The ``range`` type represents an *interval* of values. While ``multirange``
type represents a set of non-overlapping ranges.


Motivation
==========

Having a scalar type representing intervals would allow defining a consistent
way of working with them, such as determining whether two intervals overlap,
whether one interval is contained in the other, etc. The intervals could also
be used in constraints. A simple usage would be to define a single constraint
using an interval instead of using ``min_value`` and ``max_value``
constraints. More advanced usage could be defining a variant of the
``exclusive`` constraint which ensures non-overlapping interval property for
some objects.

Another usage of intervals is in generating sequences of values to be used
directly or to serve as an iterator set in a ``for`` loop.


Specification
=============

Range types would be similar to ``array`` in that they represent a collection
of values of a particular type. Since intervals in Postgres are not completely
arbitrary, we will only be able to define ``range`` and ``multirange`` over
specific scalar types.

The base scalar types that ``range`` and ``multirange`` can work with:
* ``anypoint`` - any scalar that has "points" which can be used to construct
  a range (like ``int64`` and ``float64``, but not, say, ``json`` and ``str``)
* a helper type expression ``increment<type>``, which is syntax sugar for
  looking at the return type of the operator ``-(type, type)``. It serves to
  identify the appropriate type for "stepping" through values inside an
  interval.
* ``anydiscrete extending anypoint`` - any scalar that conceptually
  represents discrete values (like ``int64`` or ``date``)
* ``anycontiguous extending anypoint`` - any scalar that conceptually
  represents contiguous values (like ``float64`` or ``datetime``)

The interval types themselves would be defined similar to arrays:
* ``range<anypoint>`` represents intervals over some concrete ``anypoint``
  sub-type
* ``multirange<anypoint>`` represents a set of non-overlapping ``range``
  values over the corresponding concrete ``anypoint`` sub-type


Boundaries
----------

A ``range`` value should have an upper and lower bound and an indicator for
each of them whether they are included in the values the ``range`` represents.

It should be possible to omit one or both of the boundaries to represent an
open-ended interval. The ``{}`` will be used to represent the omitted
boundary. It should be an error to omit the boundary, but indicate that it is
inclusive.


Constructors
------------

We will use a function constructor for creating new ``range`` values::

  function range(
      lower: optional anypoint = {},
      upper: optional anypoint = {},
      named only inc_lower: bool = true,
      named only inc_upper: bool = false
  ) -> range<anypoint>

The default values are chosen so as to minimize the amount of keystrokes
required to define ranges for common use-cases such as for iterators.

The ``inc_lower`` and ``inc_upper`` arguments are ``named only`` because when
their default values need to be overridden it is entirely plausible that only
one of them needs to be specified and thus using positional arguments would be
awkward.

A ``multirange`` could be constructed using a function that takes a set of
basic ``range`` values::

  function multirange(
      ranges: set of range<anypoint>
  ) -> multirange<anypoint>

The constructor must normalize the ranges to be non-overlapping and order them
based on boundaries. This will ensure that when a ``multirange`` is unpacked
into its component ranges, the order of the ``range`` values would be based on
their boundaries rather than arbitrary.


Accessing range components
--------------------------

Several functions will be used to access ``range`` components::

  function range_get_upper(range<anypoint>) -> optional anypoint
  function range_get_lower(range<anypoint>) -> optional anypoint
  function range_is_inclusive_upper(range<anypoint>) -> bool
  function range_is_inclusive_lower(range<anypoint>) -> bool


Accessing multirange components
-------------------------------

There will be a function for extracting sub-ranges from a multirange::

  function to_ranges(multirange<anypoint>) -> set of range<anypoint>

By default this function will return the ranges ordered by boundaries from
lowest to highest. This is the order of definition and Postgres by default
returns the sub-intervals in that order.


Helper functions
----------------

In addition to the functions mentioned above there must be a way to generate
an ordered set of values from a ``range`` or ``multirange``::

  function range_unpack(
      val: range<anydiscrete>
  ) -> set of anydiscrete

  function range_unpack(
      val: range<anydiscrete>,
      step: increment<anydiscrete>
  ) -> set of anydiscrete

  function range_unpack(
      val: multirange<anydiscrete>
  ) -> set of anydiscrete

  function range_unpack(
      val: multirange<anydiscrete>,
      step: increment<anydiscrete>
  ) -> set of anydiscrete


  function range_unpack(
      val: range<anycontiguous>,
      step: increment<anycontiguous>
  ) -> set of anycontiguous

  function range_unpack(
      val: multirange<anycontiguous>,
      step: increment<anycontiguous>
  ) -> set of anycontiguous

The ``range_unpack`` function for ``anydiscrete`` intervals **must** have
versions with and without a ``step``. The version without a ``step`` should
use a minimal value for stepping through the iterations. On the other hand
``anycontiguous`` intervals **must** specify the ``step`` explicitly in all
cases as no "natural" step value can be asumed. When invoked on an unbounded
interval ``range_unpack`` will raise an error.

The actual ``std`` funcitons should not use abstract types, but should be
instead overloads with concrete scalar types. The reason is that the ``step``
is not necessarily the same type as the main interval subtype. Specifically,
``cal::local_date``, ``cal::local_time``, ``cal::local_datetime``,
``datetime``, and ``duration`` all have ``duration`` as the step type. This
situation is currently limited to date/time types, but it's conceivable that
in the future there may be some other types that have this interaction.
Overloads accommodate this behavior well and don't require additional complex
mechanisms for figuring out valid range/step type pairs.

EdgeDB can automatically detect all concrete types for which ``range_unpack``
can be defined and generate the appropriate overloaded implementations.

We also need functions for determining overlapping and whether something is
contained in a range::

  function contains(
      haystack: range<anypoint>,
      needle: range<anypoint>
  ) -> bool

  function contains(
      haystack: range<anypoint>,
      needle: anypoint
  ) -> bool

  function contains(
      haystack: multirange<anypoint>,
      needle: multirange<anypoint>
  ) -> bool

  function contains(
      haystack: multirange<anypoint>,
      needle: anypoint
  ) -> bool

  function overlaps(
      haystack: range<anypoint>,
      needle: range<anypoint>
  ) -> bool

  function overlaps(
      haystack: multirange<anypoint>,
      needle: multirange<anypoint>
  ) -> bool

It should be noted that if ``contains(A, B) = true`` then ``overlaps(A, B) =
true``.


Operators
---------

The operators ``+`` and ``-`` should be defined for various combinations of
``range`` and ``multirange``. The semantics of ``+`` is a "union", whereas the
``-`` allows excluding some values from an interval. Both of these operators
produce a ``multirange`` in the general case. Sometimes the resulting
``multirange`` only has one ``range`` component.

We can use ``*`` to perform interval intersection.

In EdgeQL every type is *orderable*, so we must define ``<``, ``<=``, ``>``,
and ``>=`` for interval types as well. We will use the same rule as Postgres:
compare the lower bound first and only if it's the same, compare the upper
bounds.


Casting
-------

``range`` must have an implicit cast to ``multirange``.

``multirange`` must have an assignment cast to ``range``.


Backwards Compatibility
=======================

Much like ``array`` the ``range`` and ``multirange`` keywords can be
unreserved and should not present any backwards compatibility issues.


Security Implications
=====================

There are no security implications.


Rejected Alternative Ideas
==========================

Constructors
------------

One option is to have a special literal to construct a range. A particualrly
neat format can draw inspiration from the way ranges are indicated in math by
using round and square brackets to indicate boundary inclusivity. E.g.
``[5..10)`` or ``(..-10]``. The upside is that this format is familiar from
math. The downside is that this messes with any parentheses matching
processing (auto-complete, highlighting, etc.). This is also a bit too similar
to tuple and array literals, which can add a layer of confusion.

Another option is to have matching parentheses and use an extra symbol to
indicate inclusivity. E.g. ``(=5..10)`` or ``(..=-10)``. It seems that
exclusive boundary is the more natural default for this approach, largely
because it works naturally with omitted boundaries. The upside of this format
is that it plays nice with parentheses matching. The downside is that it's
hard to search for and it's not necessarily intuitive when seeing it.

Generally, having function constructiors is useful in itself and does not
prevent adding literals inthe future if such a need arises.


Accessing range components
--------------------------

We can have functions to access the 4 different components of a range value.
E.g. ``get_range_component(range_val, 'lower')``. This parametrization does
not seem very useful, though.

Alternatively, we can have ``.`` accessors directly on the range values. E.g.
``range_val.upper`` or ``range_val.includes_lower``. This puts a significant
cognitive burden on the developer when trying to determine what ``.`` means in
any given path.

We can have ``[]`` accessors directly on the range values. E.g.
``range_val['upper']`` or ``range_val['includes_lower']``. This format would
be similar to accessing elements of a ``json`` object, which can be a source
of confusion.


Accessing multirange components
-------------------------------

It may be useful to organize individual sub-ranges in a ``multirange`` like an
``array``. This way we can access them by indexing and even use ``len`` to
determine how many sub-range pieces there are in a ``multirange``. This would
require a fair bit of special processing in the compiler. Additionally it
would place extra cognitive burden on the developer to distinguish a
``multirange`` from an ``array``.
