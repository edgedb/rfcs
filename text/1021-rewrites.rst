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
to maintain a field that is derived from other object
state. (For example, maintaining a count of friends.)

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

The special variable ``__fields__`` will be bound to all of the
properties of ``<type-name>`` that were explicitly provided in the
``insert`` or ``update``. This allows rewrite rules to distinguish
between an explicitly provided empty set and no value being specified.

(Whether ``<type-name>`` should refer to the object being mutated is
kind of fraught; we leave that undecided for now. Some discussion of
that
`here
<https://github.com/edgedb/edgedb/issues/4142#issuecomment-1386287450>_`.)

Restrictions
------------
Each pointer can have one ``insert`` rewrite rule and one ``update``
rewrite rule. They can be specified together if desired.

Specifying a rewrite rule on a pointer is not allowed if an ancestor
has already specified that kind of rewrite rule for the pointer.
(Alternately: multiple rewrite rules are applied in MRO order?)


Behavior
--------

When we perform ``insert`` or ``update``, after computing the
values for the object, but before performing the operation or
evaluating the write-side access policies, we evaluate each of the
applicable rewrite rules and take the result as the actual value for
the pointer.

The rule for a pointer is applied whether or not that pointer is
explicitly specified in the operation. ``__fields__`` can be used
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
              if 'mtime' not in __fields__
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

XXX: Should access policies be applied inside of mutation rewrites?

Rejected Alternative Ideas
==========================

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
triggers and access policies into one generic mechanims. We decided
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
