::

    Status: Accepted
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

Triggers are fired off in "stages": first all triggers that were triggered
by the initial query fire, then any triggers that were triggered in the
first stage, and so on.

It is an error if a trigger would need to fire in multiple stages.

The overall state of the database during the trigger is from *after*
the query. That is, if you access data not via ``__new__`` and
``__old__``, the data is from after the query.

More generally, if there are multiple stages of triggers fired, the
database state is from after the previous stage.

Backwards Compatibility
=======================

There should not be any backwards compatibility issues.

Rationale
=========

We support both ``FOR EACH`` and ``FOR ALL`` because for all is more
powerful while for each is simpler and more ergonomic in common
cases. (Especially for ``update``, when otherwise you would often need
to join ``_old__`` and ``__new__``.


Implementation considerations
=============================

``AFTER`` triggers will be implemented through a query rewrite mechanism
in which we inline the triggers bodies into the query and run them on
the modified objects.

``AFTER COMMIT OF`` is the most complex action to implement, because
it requires two parts: 1) the query rewrite part that schedules the
action by writing the affected data and the action expression into the
job table, and 2) the runner task that pops the expression and input
data from the job table and performs the action.

As a result, we will probably skip it for the initial 3.0 implementation.


Security Implications
=====================

This whole feature is at least "security adjacent", since it may be
used to implement things like audit logs and generalized constraints.

The main direct security question is in the interaction with access
policies.  Triggers *can* see the object they are operating on, even
if it would otherwise not be visible due to access policies. (Much
like how a freshly inserted object is still returned from the query
even if it would not be visible.)

Otherwise, access policies *are* applied during trigger evaluation.


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
