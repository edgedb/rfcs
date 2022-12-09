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

Requirements
============

Specification
=============

Synopsis::

    type <type-name> "{"
        trigger <name>
            on { insert | update | delete } [, ... ]
            <action>
    "}"

    # where <action> can be one of:

    DO WHEN <element-expr>

    DO AFTER <set-expr>

    DO ON COMMIT <set-expr>

(N.B: ``DO ON COMMIT`` will probably not be implemented in the first
pass.)

For ``DO WHEN`` triggers, the ``<element-expr>`` is evaluated for
*each* of the modified objects. ``__result__`` is bound to the object
being considered. For ``update`` triggers, ``__old__`` is bound to the
*previous* value of the object. The state of the database is otherwise
that from *before* the query.

For ``DO AFTER`` triggers, the ``<set-expr>`` is evaluated for the
entire set of objects affected by the statement, bound in
``__result__``. The state of the database is that from *after* the query.

A failure in either a ``DO WHEN`` or ``DO AFTER`` trigger causes the
query to fail.

``delete`` triggers cannot be ``DO AFTER``, since the object is
already too gone to do anything useful with it.

DML performed in a trigger expression will activate triggers that
operate in a *later* phrase, but not those in the same phase. That is,
DML in a ``DO WHEN`` trigger may cause ``DO AFTER`` triggers to fire.
(This isn't necessary but could be useful.)

``DO WHEN`` and ``DO AFTER`` actions are applied directly onto the
result of an affected statement, whereas ``DO ON COMMIT`` is applied
shortly after the transaction containing the affected statement has
been successfully committed.  Thus, ``DO WHEN``/``DO AFTER`` are
suitable for actions that are themselves transactional in nature and
can be rolled back together with the relevant statement, such as
writing into an audit log set.  ``DO ON COMMIT``, on the other hand,
is designed for actions that generate non-transactional external side
effects, such as emitting an event on an external message bus.  Unlike
other actions, errors raised in ``DO ON COMMIT`` action do not affect
the original statement and will only be logged.
(And, again, ``DO ON COMMIT`` will probably be skipped in the first pass.)

Backwards Compatibility
=======================

There should not be any backwards compatibility issues.

Rationale
=========

The split between ``DO WHEN`` and ``DO AFTER`` comes from the desire
to be able to have meaningful ``delete`` triggers and to allow
``update`` triggers to access the old object, while also having the
ability to ``update`` objects that were newly inserted or already
updated (important for ``mtime`` style triggers).

The latter is difficult to do for implementation reasons if the
trigger is inlined inside the query itself (because rows can not be
updated twice by a single query in postgres), while the former can't
be done easily if the triggers are run in a separate query (since the
old date is gone).

``DO WHEN`` triggers are evaluated for each object individually so
that ``update`` triggers can be given access to the old object.

``DO AFTER`` policies are evaluated for the whole set at once because
that's what Elvis's old RFC did and it seems sometimes useful. I don't
feel strongly about it.

Implementation considerations
=============================

The `original RFC <https://github.com/edgedb/rfcs/pull/50>`_ suggested
implementing ``DO AFTER`` as a simple query rewrite mechanism,
in which the expressions to perform are injected into the query.
We can use approximately that mechansim to implement ``DO WHEN`` triggers.

For ``DO AFTER``, we will instead populate a temporary table with the
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
