::

    Status: Draft
    Type: Feature
    Created: 2022-12-08
    Authors: Michel J. Sullivan <sully@msully.net>

==================
RFC 1017: Triggers
==================

This RFC proposes a mechanism for triggers.

Some text stolen from an earlier RFC by Elvis.

Motivation
==========

Triggers are useful for things such as generating audit logs,
maintaining change metadata (last modification time) and generating external events.

Specification
=============

Synopsis::

    type <type-name> "{"
        trigger <name>
	    {before | after | after commit of}
            { insert | update | delete } [, ... ]
	    for { each | all }
	    do <expr>

    "}"

(N.B: ``AFTER COMMIT OF`` will probably not be implemented in the first
pass.)

``BEFORE`` triggers are run *before* the modifications have taken
effect. For ``insert`` and ``update``, the variable ``__new__`` refers
to the new value of the modified object, and for ``update`` and
``delete``, the variable ``__old__`` refers to the old value.
``BEFORE`` triggers may not make further modifications to the objects.
The state of the database is otherwise that from *before* the query.

``AFTER`` triggers are run *after* the modifications have taken effect.
For ``insert`` and ``update``, the variable ``__new__`` refers
to the new value of the modified object. ``AFTER`` triggers may not be
used for ``delete``. ``AFTER`` triggers *may* make further
modifications to the objects (though be careful: an object may not be
modified twice by two different triggers.)
The state of the database is that from *after* the query.

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

A failure in either a ``BEFORE`` or ``AFTER`` trigger causes the
query to fail.

DML performed in a trigger expression will activate triggers that
operate in a *later* phrase, but not those in the same phase. That is,
DML in a ``BEFORE`` trigger may cause ``AFTER`` triggers to fire.
(This isn't necessary but could be useful.)

``BEFORE`` and ``AFTER`` actions are applied directly onto the
result of an affected statement, whereas ``AFTER COMMIT OF`` is applied
shortly after the transaction containing the affected statement has
been successfully committed.  Thus, ``BEFORE``/``AFTER`` are
suitable for actions that are themselves transactional in nature and
can be rolled back together with the relevant statement, such as
writing into an audit log set.  ``AFTER COMMIT OF``, on the other hand,
is designed for actions that generate non-transactional external side
effects, such as emitting an event on an external message bus.  Unlike
other actions, errors raised in ``AFTER COMMIT OF`` action do not affect
the original statement and will only be logged.
(And, again, ``AFTER COMMIT OF`` will probably be skipped in the first pass.)

Backwards Compatibility
=======================

There should not be any backwards compatibility issues.

Rationale
=========

The split between ``BEFORE`` and ``AFTER`` comes from the desire
to be able to have meaningful ``delete`` triggers and to allow
``update`` triggers to access the old object, while also having the
ability to ``update`` objects that were newly inserted or already
updated (important for maintaining last modification metadata).

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

Implement using postgres triggers
---------------------------------

I haven't thought very hard about htis possibility yet, but I know
that Yury and Elvis hate it. We should potentially still consider it
though.
