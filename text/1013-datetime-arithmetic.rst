::

    Status: Draft
    Type: Feature
    Created: 2022-06-06
    Authors: Victor Petrovykh <victor@edgedb.com>
    RFC PR: `edgedb/rfcs#0059 <https://github.com/edgedb/rfcs/pull/59>`_

==============================
RFC 1013: Date/time arithmetic
==============================

This RFC proposes defining subtraction (``-`` operator) for  all of the
"local" variants of date/time scalars. It also proposes adding a
``date_duration`` to represent intervals with whole number of days and without
any smaller components. This is a conceptual change in how ``local_date`` is
treated w.r.t. arithmetic, being now more analogous to integers. Finally, we
need to add functions for normalizing, truncating and producing various
flavours of duration as well as extracting its various components.


Motivation
==========

At the time of writing all local date/time scalars lack the operator for
finding the difference (``-``) between values of the same type. It is not
possible to simply subtract a ``local_datetime`` value from another
``local_datetime``, ``local_date`` from ``local_date``, and ``local_time``
from ``local_time``.

We have 2 different scalar type representing a time delta: ``duration`` and
``relative_duration``. The ``duration`` scalar is meant to represent an
absolute value of some period of time in the real world (e.g. a specific
number of seconds or some other unambiguous equivalent units). The
``relative_duration`` can represent time intervals in unambiguous units
(seconds, minutes, hours) as well as in ambiguous, but useful calendar units
such as *days*, *weeks*, *months*, *years*, etc. A *month* may not have a
well-defined number of seconds, yet it is a well-defined time period w.r.t. a
specific calendar system and can be used in that context.

It seems that the result of subtracting one ``local_datetime`` value from
another should result in a ``relative_duration``. It is analogous to the same
operation on ``datetime`` resulting in ``duration``. Since ``datetime``
represents specific points in actual reality we can define an absolute value
of ``duration`` between them. In contrast, ``local_datetime`` does not contain
enough information to pin it to any specific point in reality and so at best
only a ``relative_duration`` can be given as the result of a subtraction
operation. It's an operation that ignores the potential nuance of DST
affecting the time difference.

In the same vein as ``local_datetime``, the subtraction of one ``local_time``
value from another should also produce ``relative_duration``. The
``local_time`` represents a certain value on a clock. For the purpose of this
RFC it may be easier to imagine the clock as sort of "broken", not able to
tick on its own, but still able to be adjusted by hand. The questions of
arithmetic on such a clock are not about real passage of time, but rather
about the amount of adjustment the clock requires to change the value from one
to another. For example, imagine two snapshots of a clock: one at midnight,
another at 4 AM. The time difference between these clock snapshots is 4 hours.
However we have no way of knowing whether it would be 4, 28 or even 3 or 5
hours (if a DST adjustment happened) between these two states of an accurate
clock in the real world. So despite the fact that ``local_time`` difference
seems to operate entirely in the range of hours/minutes/seconds it is
inherently ambiguous and relative, rather than absolute like ``duration``.

The ``local_date`` type represents a calendar and by analogy with the other
two local scalar types it may seem that ``relative_duration`` is the natural
unit for measuring the difference between two dates. However, for practical
reasons, it seems that it may be more appropriate to treat ``local_date`` as
being more akin to a special kind of integer (counting the days) whereas
``local_datetime`` is akin to a float. So the difference between two
``local_date`` values would be some interger-like duration representing the
number of days. It is unambiguous an often what is practicaly useful. We wll
call this new scalar type ``date_duration`` and it will be like
``relative_duration``, but without the elements smaller than a day.

Working with ``relative_duration`` sometimes requires converting between hours
and days or between days and months. We will need to expose the ``justify``
and ``truncate`` functionality to achieve this.

Finally, even though relative duration measured in days is arguably more
accurate for many smaller time periods, it can be better to produce a relative
duration measured accurately in larger units for longer time periods and so we
need to expose ``age`` from Postgres for this purpose, possibly under a
different name.


Specification
=============

The ``local_datetime`` will introduce the following subtraction operator::

  CREATE INFIX OPERATOR
  std::`-` (
    l: cal::local_datetime,
    r: cal::local_datetime
  ) -> cal::relative_duration {
      USING SQL $$
          SELECT ("l" - "r")::edgedb.relative_duration_t
      $$;
  };

The result is expected to be a ``relative_duration`` in days and
hours/minutes/seconds. This is the most precise format as ``local_datetime``
cannot account for DST and so 1 day and 24 hours are always equivalent
durations in this context.

The ``local_time`` will introduce the following subtraction operator::

  CREATE INFIX OPERATOR
  std::`-` (
    l: cal::local_time,
    r: cal::local_time
  ) -> cal::relative_duration {
      USING SQL $$
          SELECT ("l" - "r")::edgedb.relative_duration_t
      $$;
  };

The result is expected to be a ``relative_duration`` in the range [-24 hrs, 24
hrs].

Introduce ``date_duration`` scalar that is "whole days" version of
``relative_duration``. Much like integers are not related to floats,
``date_duration`` is not directly a subtype of ``relative_duration``. Instead
they have the following casts between them::

  CREATE CAST FROM cal::date_duration TO cal::relative_duration {
      USING SQL CAST;
      ALLOW IMPLICIT;
  };

  CREATE CAST FROM cal::relative_duration TO cal::date_duration {
      USING SQL CAST;
  };

Similarly, we need an implicit cast from ``local_date`` to
``local_datetime``::

  CREATE CAST FROM cal::local_date TO cal::local_datetime {
      USING SQL CAST;
      ALLOW IMPLICIT;
  };

The above is analogous to integers implicitly casting into floats.

The ``local_date`` will introduce the following subtraction operator::

  CREATE INFIX OPERATOR
  std::`-` (
    l: cal::local_date,
    r: cal::local_date
  ) -> cal::date_duration {
      USING SQL $$
          SELECT ("l" - "r")::edgedb.date_duration_t
      $$;
  };

The result is expected to be the number of days.

The ``+`` operators for ``local_date`` should produce ``local_date`` results
only if the other operand is ``date_duration``, otherwise ``local_datetime``
should be produced, analogous to ``int64 + int64 = int64``, but ``int64 +
float64 = float64``. So we defined these operators as follows::

  CREATE INFIX OPERATOR
  std::`+` (l: cal::local_date, r: cal::date_duration) -> cal::local_date
  {
      USING SQL $$
          SELECT ("l" + "r")::edgedb.date_t
      $$;
  };

  CREATE INFIX OPERATOR
  std::`+` (l: cal::local_date, r: std::duration) -> cal::local_datetime
  {
      USING SQL $$
          SELECT ("l" + "r")::edgedb.timestamp_t
      $$;
  };

  CREATE INFIX OPERATOR
  std::`+` (
    l: cal::local_date, r: cal::relative_duration
  ) -> cal::local_datetime
  {
      USING SQL $$
          SELECT ("l" + "r")::edgedb.timestamp_t
      $$;
  };


Duration functions
------------------

We also will introduce normalization functions for ``relative_duration``::

  CREATE FUNCTION
  cal::duration_normalize_hours(dur: cal::relative_duration)
    -> cal::relative_duration
  {
      USING SQL FUNCTION 'justify_hours';
  };

  CREATE FUNCTION
  cal::duration_normalize_days(dur: cal::relative_duration)
    -> cal::relative_duration
  {
      USING SQL FUNCTION 'justify_days';
  };

  CREATE FUNCTION
  cal::duration_normalize_days(dur: cal::date_duration)
    -> cal::date_duration
  {
      USING SQL FUNCTION 'justify_days';
  };

The ``duration_normalize_hours`` converts 24 hrs chunks into days.
The ``duration_normalize_days`` converts days into months assuming that 1 month = 30 days.

Notice that for ``relative_duration`` the assumtion that 1 day is always 24
hours holds true because it represents the time adjustment necessary on some
clock rather than real time (like ``duration``). Therefore the
``duration_normalize_hours`` transformation is technically lossless and could
safely be reversed. However, not every month is 30 days and so the
``duration_normalize_days`` and ``duration_normalize`` are
potentially lossy transformations aiming to produce an approximately
equivalent ``relative_duration`` using months, years, etc. so care must be
taken when using these conversions.

Only ``duration_normalize_days`` has an overloaded version to accept
``date_duration`` and return the same type. There are no hours to normalize
for ``date_duration`` and thus the other two normalization functions are
unnecessary for ``date_duration``.

We also need a function for extracting various ``duration`` and
``relative_duration`` components::

  CREATE FUNCTION
  std::duration_get(dur: cal::relative_duration, el: std::str) -> std::float64
  {
    ...
  };

  CREATE FUNCTION
  std::duration_get(dur: std::duration, el: std::str) -> std::float64
  {
    ...
  };

The components avaialable for extraction from ``relative_duration`` are:
*millenium*, *century*, *decade*, *year*, *quarter*, *month*, *day*, *hour*,
*minutes*, *seconds*, *milliseconds*, *microseconds*, and *totalseconds*. The
*totalseconds* converts a ``relative_duration`` to seconds represented as a
``float64`` value. Components greater than *hour* are not available for
``duration`` and will produce an error if there's an attempt to extract them.

In addition to extraction function we also introduce a truncation function for
``relative_duration``. We will overload already existing ``duration_truncate``
to also accept ``relative_duration`` input and extend the list of truncated
precision to include components greate than *hour*::

  CREATE FUNCTION
  std::duration_truncate(
    dt: cal::relative_duration,
    unit: std::str
  ) -> cal::relative_duration
  {
    ...
  };

We expose the ``age`` functionality as ``relative_delta``. Basically this is a
counterpart to ``-`` operator, but performed symbolically and producing an
accurate result for the specific inputs::

  CREATE FUNCTION
  cal::relative_delta(
    l: cal::local_date,
    r: cal::local_date
  ) -> cal::date_duration
  {
    SET force_return_cast := true;
    USING SQL FUNCTION 'age';
  };

  CREATE FUNCTION
  cal::relative_delta(
    l: cal::local_datetime,
    r: cal::local_datetime
  ) -> cal::relative_duration
  {
    SET force_return_cast := true;
    USING SQL FUNCTION 'age';
  };



Backwards Compatibility
=======================

There's a change in ``+`` arithmetic. The addition of ``local_date`` and
``duration`` or ``relative_duration`` now results in ``local_datetime``
instead of ``local_date``. This is a backwards-incompatible change as
``local_datetime`` cannot be used instead of ``local_date`` without an
explicit cast.

It is also desirable to later deprecate arithmetic operators that allow
interactions between the "local" date/time types and absolute ``duration`` as
that is not well-defined.


Security Implications
=====================

There are no security implications.


Rejected Alternative Ideas
==========================

We specifically rejected the option of ``-`` operating on two ``local_date``
values producing a ``relative_duration`` by default. In practice, this
operation would produce ``relative_duration`` expressed using exclusively days
anyways. The main difference then would be that in order to use the result of
this operation one would need to use ``<int64>relative_duration_get(dur,
'days')``.

We also rejected the notion that subtracting one ``local_date`` from another
should produce an integer, like it does in Postgres. Basically, this seems to
break the symmetry in ``-`` and ``+`` arithemtic for date/time types, while
not addressing future use of ``date_duration`` as a "step" in
``generate_series`` type of scenario for ``local_date``, with the step being a
"month" or some other larger unit::

  CREATE INFIX OPERATOR
  std::`-` (
    l: cal::local_date,
    r: cal::local_date
  ) -> int64 {
      USING SQL $$
          SELECT ("l" - "r")::int8
      $$;
  };

By analogy with ``datetime_get`` being able to work with either ``datetime``
and ``local_datetime`` it makes sense to use ``duration_get`` (instead of
``relative_duration_get``) to operate on both ``duration`` and
``relative_duration``.

We decided against having a convenience function ``duration_normalize``,
because the 2 funcitons it combines have different effect of prcision of the
result::

  CREATE FUNCTION
  cal::duration_normalize(dur: cal::relative_duration)
    -> cal::relative_duration
  {
      USING SQL FUNCTION 'justify_interval';
  };