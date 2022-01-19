::

    Status: Draft
    Type: Feature
    Created: 2022-01-15
    Authors: Elvis Pranskevichus <elvis@edgedb.com>
    RFC PR: `edgedb/rfcs#0050 <https://github.com/edgedb/rfcs/pull/50>`_

====================================
RFC 1011: Policy based query rewrite
====================================

This RFC proposes adding a mechanism to declare per-type policies in schemas
to modify or augment the behavior of statements that contain references to a
type on which the policy is defined.


Motivation
==========

The main motivation for this mechanism is to provide a way to define
*object level security* (equivalent of row level security in SQL) by declaring
policies that automatically inject the necessary filters based on the session
context.

The secondary motivation is the ability to trigger additional processing
whenever an object in question is read or modified.  This is commonly referred
to as "triggers" in SQL databases and is very useful to implement patterns
such as audit logs or to automatically generate external change events.


Requirements
============

Although there is no direct dependency, this RFC requires RFC 1010 (globals)
to be implemented to be practical.  See `Interaction with globals`_ below
for details.


Specification
=============

This RFC proposes to add a new schema item type -- *Policy* -- that is
defined in the context of an object type and specifies the rules of rewriting
queries that refer to the enclosing object type.

CREATE POLICY
-------------

Define a new query rewrite policy for a given object type.

Required capabilities: DDL.

Synopsis::

    {CREATE|ALTER} OBJECT TYPE <type-name> "{"
        CREATE POLICY
            ON { SELECT | INSERT | UPDATE | DELETE } [, ... ]
            [ WHEN <condition> ]
            <action>
    "}"

    # where <action> can be one of:

    PERMIT <element-expr>

    RESTRICT TO <element-expr>

    CHECK <element-expr>

    DO AFTER <set-expr>

    DO ON COMMIT <set-expr>

Policies apply to the given list of statement types.  Not all actions are
valid for all statement types.  See table below for legal combinations.

The optional ``<condition>`` expression is evaluated for every object
affected by the statement and the policy is applied only if the expression
evaluates to a set containing at least one ``true`` value.

``<element-expr>`` is evaluated for every object affected by the statement
the same way as a ``FILTER`` clause is evaluated, and can use abbreviated
paths, for example::

    POLICY ON SELECT RESTRICT TO .rating != 'R'

``<set-expr>`` is evaluated for the entire set of objects affected by the
statement and can refer to the set as ``__result__``, for example::

    POLICY ON UPDATE DO AFTER (
        FOR obj IN __result__ UNION (
        INSERT AuditLog {
            text := 'updated ' ++ obj.name
        })
    )

    # or

    POLICY ON INSERT DO AFTER (
        INSERT EventLog {
            text := 'inserted ' ++ count(__result__) ++ ' things'
        }
    )

The ``PERMIT`` and ``RESTRICT TO`` actions are *filtering* actions and define
which elements of the object set are visible in a query.  If at least one
filtering policy is defined for a type, then all elements become invisible by
default and must be made visible by at least one ``PERMIT`` policy.

The filtering actions are defined as follows:

- ``PERMIT`` -- add all objects, for which the expression evaluates to a
  set containing at least one ``true`` value, to the set of visible objects.
  ``PERMIT`` policies are combined using the ``OR`` operator.

- ``RESTRICT TO`` -- restrict the set of visible objects to only objects for
  which the expression evaluates to a set containing at least one ``true``
  value.  ``RESTRICT`` policies are combined using the ``AND`` operator and
  are applied *after* all ``PERMIT`` policies.

The ``CHECK`` policy action is a *validation* action and is applied to
``INSERT`` and ``UPDATE`` statements.  An object modification will be
allowed only if the ``<element-expr>`` returns a set containing at least one
``true`` value, otherwise an error will be raised and the statement execution
aborted.  Note that in ``UPDATE`` the expression is applied to the object will
all updates applied, not the original object.

The ``DO AFTER`` and ``DO ON COMMIT`` are *trigger* actions and are applied
to the result of a statement *as a whole*.  The result set is available in
the ``<set-expr>`` expression as ``__result__``.  ``DO AFTER`` action
is applied directly onto the result of an affected statement, whereas
``DO ON COMMIT`` is applied shortly after the transaction containing the
affected statement has been successfully committed.  Thus, ``DO AFTER`` is
suitable for actions that are themselves transactional in nature and can be
rolled back together with the relevant statement, such as writing into an
audit log set.  ``DO ON COMMIT``, on the other hand, is designed for actions
that generate non-transactional external side effects, such as emitting an
event on an external message bus.  Unlike other actions, errors raised in
``DO ON COMMIT`` action do not affect the original statement and will only
be logged.

Allowed statement/action combination are as follows:

+--------------+--------+--------+--------+--------+
|              | SELECT | INSERT | UPDATE | DELETE |
+--------------+--------+--------+--------+--------+
| PERMIT       |  yes   |   n/a  |  yes   |  yes   |
+--------------+--------+--------+--------+--------+
| RESTRICT     |  yes   |   n/a  |  yes   |  yes   |
+--------------+--------+--------+--------+--------+
| CHECK        |  n/a   |   yes  |  yes   |  n/a   |
+--------------+--------+--------+--------+--------+
| DO AFTER     |  yes   |   yes  |  yes   |  yes   |
+--------------+--------+--------+--------+--------+
| DO ON COMMIT |  n/a   |   yes  |  yes   |  yes   |
+--------------+--------+--------+--------+--------+

It is an error to specify a policy action for a statement that does not
support it.


ALTER POLICY
------------

Alter the definition of a query rewrite policy.

Required capabilities: DDL.

Synopsis::

    ALTER OBJECT TYPE <type-name> "{"
        ALTER POLICY
            ON { SELECT | INSERT | UPDATE | DELETE } [, ... ]
            [ WHEN <condition> ]
            <action>
        "{" <subcommand>; [...] "}" ;
    "}"

    # where <subcommand> is one of

      CREATE ANNOTATION <annotation-name> := <value>
      ALTER ANNOTATION <annotation-name> := <value>
      DROP ANNOTATION <annotation-name>


DROP POLICY
-----------

Remove a query rewrite policy.

Required capabilities: DDL.

Synopsis::

    ALTER OBJECT TYPE <type-name> "{"
        DROP POLICY
            ON { SELECT | INSERT | UPDATE | DELETE } [, ... ]
            [ WHEN <condition> ]
            <action>
    "}"


Interaction with globals
========================

Query rewrite policies are especially powerful when combined with RFC 1010
globals, because then data visibility can be globally adjusted with a single
``SET GLOBAL`` statement, which is very useful for authenticated/authorized
data access control.

Example::

    global user_id -> uuid;

    abstract object type Owned {
      required link owner -> User;

      policy on select, update, delete
        permit (.owner.id = global user_id)

      policy on insert, update
        check (.owner.id = global user_id)
    }

    object type Purchase extending Owned;

    ...

    set global user_id := <uuid-1>;
    select count(Purchase);
    # 9
    set global user_id := <uuid-2>
    select count(Purchase);
    # 1


Another example::

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


Bypassing policies
==================

A superuser can bypass the execution of query rewrite policies by setting
the ``apply_rewrite_policies`` session configuration setting to ``false``.


Introspection
=============

Policies can be introspected via a new ``schema::RewritePolicy`` in the
introspection schema that is linked from ``schema::ObjectType`` via the new
``policies`` link.  The ``schema::RewritePolicy`` is exposed as follows::

    type schema::RewritePolicy extending schema::AnnotationSubject {
      multi property statements -> schema::StatementKind;
      property condition -> std::str;
      required property action_kind -> schema::PolicyActionKind;
      required property action -> std::str;
    };


Implementation considerations
=============================

Like the title of the RFC suggests, policies are a query rewrite mechanism,
which means that they primarily affect what IR is generated for a given
EdgeQL query.  All actions except ``DO ON COMMIT`` are pure EdgeQL transforms.

``PERMIT`` and ``RESTRICT TO`` wrap set references and transform every ``Foo``
reference into ``(SELECT Foo FILTER <permit-restrict-filter>)``.

``CHECK`` actions insert an intermediate shape into ``INSERT`` and ``UPDATE``,
e.g.::

    INSERT Foo { prop := <value> }

is roughly transformed into::

    WITH
      input := { prop := <value> },
      checked := input { prop := <check_expr> }
    INSERT Foo { prop := checked.prop }

``DO AFTER`` actions transform affected statements roughly as::

    WITH
      __result__ := <orig-stmt>,
      do_after := <do-after-expr>
    SELECT
      __result__

When specified, the ``WHEN`` conditions must be taken into account, e.g by
combining directly with the ``PERMIT/RESTRICT`` filters, and ``CHECK``
expressions.  Trigger actions can be made conditional by wrapping the action
in a ``FOR`` statement referencing the policy condition, e.g.::

    FOR _ IN (SELECT true FILTER <policy-condition>)
    UNION (<action>)

``DO ON COMMIT`` is the most complex action to implement, because it requires
two parts: 1) the query rewrite part that schedules the action by writing the
affected data and the action expression into the job table, and 2) the runner
task that pops the expression and input data from the job table and performs
the action.


Backwards compatibility
=======================

This RFC does not pose any backwards compatibility issues.
