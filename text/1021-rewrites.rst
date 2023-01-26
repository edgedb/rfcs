::

    Status: Draft
    Type: Feature
    Created: 2023-01-12
    Authors: Michel J. Sullivan <sully@msully.net>

===========================
RFC 1021: Mutation rewrites
===========================

This RFC proposes a mechanism for specifying rewrite rules that are
applied to properties and links on INSERT and UPDATE of objects.

They can be thought of as a generalization of ``default``.


Motivation
==========

It is sometimes useful to have a pointer in an object automatically updated
whenever the object has an ``UPDATE`` performed on it.

The most straightforward example of this is having a
``modification_time`` field that tracks when an object was last
modified.

It is also valuable when (probably for performance reasons) you wish
to maintain a field that is derived from other object state (for
example, maintaining a count of friends) or if you want to rewrite
incoming data on the database side, perhaps for backward compatability
reasons.


Note
====

This is a companion to
`RFC 1020 <https://github.com/edgedb/rfcs/blob/master/text/1020-triggers.rst>`_,
in that they work together to cover most of the desired use cases that motivated
a previous `previous version triggers RFC <https://github.com/edgedb/rfcs/pull/70>`_


Specification
=============

Synopsis::

    type <type-name> "{"
        { property | link } <name> -> <type> "{"
            rewrite { insert | update } [, ... ] using (<expr>)
        "}"
    "}"

We allow pointer definitions to be augmented with a ``rewrite`` rule.

Expression interpretation
-------------------------
In ``<expr>``, ``__subject__`` (and the implicit path prefix) will be
bound to the new value object being inserted or updated.

In purely ``update`` rewrite rules, ``__old__`` will refer to the old
version of the object.

The special variable ``__specified__`` will be bound to a named tuple
containing a ``bool`` field for each pointer of ``<type-name>`` that
is true if that pointer was explicitly provided in the ``insert`` or
``update``.
This allows rewrite rules to distinguish between an
explicitly provided empty set and no value being specified.

The names are sorted alphabetically in the tuple.
Note that which pointers appear in the tuple is determined by the
type where the rewrite rule was *defined*, not the type at which
it is being *evaluated* (which could be a subtype). This ensures
that the type is always consistent.

(Whether ``<type-name>`` should refer to the object being mutated is
kind of fraught; we leave that undecided for now. Some discussion of
that
`here
<https://github.com/edgedb/edgedb/issues/4142#issuecomment-1386287450>_`.)

Restrictions
------------
Each pointer can have one ``insert`` rewrite rule and one ``update``
rewrite rule. They can be specified together if desired.

Behavior
--------

For any pointer, the rewrite rule that applies is the one that appears
*first* in the MRO (like we do with defaults).

When we perform ``insert`` or ``update``, after computing the
values for the object, but before performing the operation or
evaluating the write-side access policies, we evaluate each of the
applicable rewrite rules and take the result as the actual value for
the pointer.

The rewrites are evaluated "simultaneously"; each rewrite sees the
original object.

Access policies *are* evaluated inside the expression.

The rule for a pointer is applied whether or not that pointer is
explicitly specified in the operation. ``__specified__`` can be used
to determine whether it was and take action based on that.

Note that for ``insert``, rewrite rules are *extremely* similar to
``default`` (which can now refer to other pointers), but are applied
even when a value was specified.


Examples
========

Update a modification time field every time an object is modified::

  type Entry {
      # ...
      required property mtime -> datetime {
          rewrite insert, update using (datetime_of_statement())
      }
  };


Update a modification time field every time an object is modified, but
allow a manual override of it also::

  type Entry {
      # ...
      required property mtime -> datetime {
          rewrite insert, update using (
              datetime_of_statement()
              if not __specified__.mtime
              else .mtime
          )
      }
  };

Maintain a cached count of the cardinality of a multi link::

  type Post {
      # ...
      multi link likes -> Like;
      required property cached_like_count -> int64 {
          rewrite insert, update using (count(.likes))
      }
  };

Rewrite a incoming pointer (maybe for backward compatibility after a
format change)::

  type Item {
      # ...
      required property product_code -> str {
          rewrite insert, update using (str_upper(.product_code))
      }
  };


Backwards Compatibility
=======================

There should not be any backwards compatibility issues.


Implementation considerations
=============================

Most of the infrastructure for computing a "contents" row for the
main object table is already there, and it shouldn't be too hard to
wrap that and replace some fields in it.

Dealing with ``multi`` pointers might be pretty nasty, though. We
don't currently generate "contents" CTEs for them in all the general
cases (such as doing ``-=``), so there might be a lot of subtle
engineering work needed to get everything positioned for this.

We can probably skip supporting ``multi`` pointers in the first take
of this, if necessary.


Security Implications
=====================

Access policies *are* evaluated inside the expression.


Rejected Alternative Ideas
==========================

Different ways of representing which pointers are specified
-----------------------------------------------------------

The original proposal had a ``__fields__`` field that contained a
set of strings of names of specified pointers. This worked but was
ugly and would have required some special work to implement
efficiently in the common case. If you actually want such a set,
it can be obtained in the current proposal with::

  (select json_object_unpack(<json>__specified__) filter <bool>.1).0

Another proposal was to have a magic "function" (or operator) that
returned whether a field was set, such as ``specified(.friends)``
would be true if ``friends`` was specified in the DML statement.
This was rejected because it had to either be purely magic syntax
or required introducing a new notion of "unspecified" into the
semantics that could only be distinguished from ``{}`` by the
new ``specified`` function, and because the named tuple proposal
reads just as well but without any worrying implications.

Calling ``__specified__`` something else
----------------------------------------

Originally I proposed ``__fields__``, which was bad. ``__specified__``
is kind of long, so something shorter would be nice, but our time
spent looking at a thesaurus did not help us.

The best option we had was ``__set__``, which Yury hated. That would
look something like::

  type Entry {
      # ...
      required property mtime -> datetime {
          rewrite insert, update using (
              datetime_of_statement() if not __set__.mtime else .mtime
          )
      }
  };


Making this explicitly an extension of default
----------------------------------------------

Another proposal was to treat this exactly as default generalized to
``update`` (to handle the mtime cases) and to add a notion of
``cached property`` for things like the ``cached_like_count`` case.

This was rejected because while we do eventually want some kind of
cached/materialized values, there is a lot of complexity in the design
space there and we don't want to ship a super limited version of it
that might mislead users and limit our options in the future.

It also doesn't support genuine "rewrite" style operations.


Making mutation rewrites per-object instead of per-pointer
----------------------------------------------------------

Doing it per-object makes it unclear how it should compose in the
presence of inheritance. We would need to be much more innovative
in terms of syntax and semantics. (Probably: return a free object,
which then gets composed in some way.)


Generalized policy based query rewrite
--------------------------------------
A `previous RFC
<https://github.com/edgedb/rfcs/pull/50>`_ written by Elvis, combined
triggers and access policies into one generic mechanism. We decided
this was likely to be too complex, and that they should be split.

I also think there would have been severe implementation difficulties.


Using triggers and having a BEFORE/AFTER split
----------------------------------------------

Another `previous version of the trigger RFC
<https://github.com/edgedb/rfcs/pull/70>`_, contained
a distinction between ``BEFORE`` triggers and ``AFTER`` triggers.

``AFTER`` triggers would be run in a pipelined query, would not have
access to ``__old__`` (and as such could not be used for ``DELETE``),
and *could* modify objects that had already been modified in the
original query.

That handled this case, and was probably workable, but was generally
complex and the distinctions between ``BEFORE`` and ``AFTER`` triggers
were weird and heavily implementation driven.


Implement using postgres triggers
---------------------------------

There is a critical semantic problem in using postgres triggers, which
is that postgres triggers only have access to the old state of the
database and to the new rows. But in edgedb, the state of an object
might be spread across multiple tables (for multi pointers), and so
the full state of a new or updated object may be invisible to a
postgres trigger.
