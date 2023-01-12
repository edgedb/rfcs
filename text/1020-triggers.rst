::

    Status: Draft
    Type: Feature
    Created: 2022-12-15
    Authors: Michel J. Sullivan <sully@msully.net>

==================
RFC 1020: Triggers
==================

This RFC proposes a mechanism for triggers.

Some text stolen from an earlier RFC by Elvis.

Motivation
==========

Triggers are useful for things such as generating audit logs and
generating external events.

Note
====

This version of the trigger RFC reflects a split between "triggers"
and "mutation rewrites", which will come in another RFC.

Specification
=============

Synopsis::

    type <type-name> "{"
        trigger <name>
	    {after | after commit of}
            { insert | update | delete } [, ... ]
	    for { each | all }
	    do <expr>

    "}"

(N.B: ``AFTER COMMIT OF`` will probably not be implemented in the first
pass.)

``AFTER`` triggers are run after constraint and access policies have
been checked.  For ``insert`` and ``update``, the variable ``__new__``
refers to the new value of the modified object, and for ``update`` and
``delete``, the variable ``__old__`` refers to the old value.

XXX: Whether the overall state of the database (if you query things
apart from ``__new__`` and ``__old__`` is from before the query or
after the query is an open question. Before the query is technically
the most straightforward.

``AFTER COMMIT OF`` triggers will run "shortly" after the containing
transaction is committed. It has ``__new__`` variables, like ``AFTER``
triggers. The state of the database is indeterminate, since an
potentially many queries may have executed after the triggering
even. (This means that the ``__new__`` objects may no longer exist!)

If ``FOR EACH`` is specified the ``<expr>`` is evaluated for
*each* of the modified objects and the variables ``_old__`` and
``__new__`` refer to individual objects. If ``FOR ALL`` is specified,
then the trigger is run once per query where more than one applicable
object was modified, and ``__old__`` and ``__new__`` refer to the
entire set of old and new modified objects.

A failure in an ``AFTER`` trigger causes the query to fail.

Trigger activation for ``AFTER`` triggers is chained but not recursive:
triggers can activate other triggers, but a trigger cycle is an error.
XXX: This could also reasonably disallowed.


Backwards Compatibility
=======================

There should not be any backwards compatibility issues.

Rationale
=========

The split between ``BEFORE`` and ``AFTER`` comes from the desire
to be able to have meaningful ``delete`` triggers and to allow
``update`` triggers to access the old object, while also having the
ability to ``update`` objects that were newly inserted or already
updated (important for ``mtime`` style triggers).

The latter is difficult to do for implementation reasons if the
trigger is inlined inside the query itself (because rows can not be
updated twice by a single query in postgres), while the former can't
be done easily if the triggers are run in a separate query (since the
old date is gone).

We support both ``FOR EACH`` and ``FOR ALL`` because for all is more
powerful while for each is simpler and more ergonomic in common
cases. (Especially for ``update``, when otherwise you would often need
to join ``_old__`` and ``__new__``.


Implementation considerations
=============================

The `original RFC <https://github.com/edgedb/rfcs/pull/50>`_ suggested
implementing ``DO AFTER`` as a simple query rewrite mechanism,
in which the expressions to perform are injected into the query.
We can use approximately that mechansim to implement ``BEFORE`` triggers.

For ``AFTER``, we will instead populate a temporary table with the
id and type of the trigger-containing objects that have been inserted
or updated.  We will then execute the triggers in a separate query
that consumes rows from the temporary table and executes the triggers.

The seperate query will be performed in the same implicit transaction,
and can be pipelined so that no additional roundtrip between EdgeDB
and Postgres is needed.


``DO ON COMMIT`` is the most complex action to implement, because it requires
two parts: 1) the query rewrite part that schedules the action by writing the
affected data and the action expression into the job table, and 2) the runner
task that pops the expression and input data from the job table and performs
the action.

As a result, we will probably skip it for the initial 3.0 implementation.


Security Implications
=====================

Probably?

Rejected Alternative Ideas
==========================

Generalized policy based query rewrite
--------------------------------------
A `previous version of this RFC
<https://github.com/edgedb/rfcs/pull/50>`_ written by Elvis, combined
triggers and access policies into one generic mechanims. We decided
this was likely to be too complex, and that they should be split.

ON SELECT
---------

That RFC also called for being able to do ``DO AFTER`` actions on
``select``, giving as an example::

    abstract object type LoggedAccess {
      policy on select
        do after (for obj in __result__ union (
          insert AccessLog {
            object := obj,
            user_id := global user_id,
            time := datetime_current(),
          }
        ))
    }

I have left this out of the proposal for now because it seems hard
semantically to say what objects get ``select``ed. Presumbably
``select Obj filter .id = ...`` should only fire the policy once,
but how about ``with W := (select Obj), select W filter .id = ...``.


Having a BEFORE/AFTER split
---------------------------

Another `previous version of this RFC
<https://github.com/edgedb/rfcs/pull/70>`_, contained
a distinction between ``BEFORE`` triggers and ``AFTER`` triggers.

``BEFORE`` triggers would be inlined into the query, would have access
to ``__old__``, and could *not* modify objects that had already been
modified.

``AFTER`` triggers would be run in a pipelined query, would not have
access to ``__old__`` (and as such could not be used for ``DELETE``),
and *could* modify objects that had already been modified in the
original query.

(The ``AFTER`` triggers proposed in this RFC are actually mostly the
``BEFORE`` triggers from the old RFC.)

This approach is plausible, and makes it possible to do most of the
things we wanted, but the differing limitations between the two is
complicated and likely confusing.


Implement using postgres triggers
---------------------------------

There is a critical semantic problem in using postgres triggers, which
is that postgres triggers only have access to the old state of the
database and to the new rows. But in edgedb, the state of an object
might be spread across multiple tables (for multi pointers), and so
the full state of a new or updated object may be invisible to a
postgres trigger.
