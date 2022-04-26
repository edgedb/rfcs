::

    Status: Draft
    Type: Feature
    Created: 2022-04-18
    Authors: Elvis Pranskevichus <elvis@edgedb.com>
    RFC PR: `edgedb/rfcs#0054 <https://github.com/edgedb/rfcs/pull/54>`_

===============================
RFC 1011: Object-Level Security
===============================

This RFC proposes adding a mechanism to declare per-type access control
rules in schemas to automatically enforce data access and modification on
per-object level.  This is analogous to row-level security (RLS) in SQL
systems.


Motivation
==========

Object-level security is a powerful feature which allows universal leak-proof
enforcement of data access policies across all uses of a database.  This is
important for security and compliance, but would also be tremendously useful
for backend-less applications that interact with EdgeDB via HTTP (either via
GraphQL or EdgeQL-over-HTTP).  A secondary motivation is that this will allow
implementation of `temporal databases <temporal>`_.


Requirements
============

Although there is no direct dependency, this RFC requires
`RFC 1010 <1001-global-vars.rst>`_ (globals) to be implemented to be practical.
See `Interaction with globals`_ below for details.


Specification
=============

This RFC proposes to add a new schema item type -- *Access Policy* -- that is
defined in the context of an object type and specifies the rules of rewriting
queries that refer to the enclosing object type.

CREATE ACCESS POLICY
--------------------

Define a new access control policy for a given object type.

Required capabilities: DDL.

Synopsis::

    {CREATE|ALTER} OBJECT TYPE <type-name> "{"
        CREATE ACCESS POLICY <name>
            [ WHEN <condition> ]
            { ALLOW | DENY }
            { ALL | READ | WRITE | DELETE } [ ... ]
            [ USING (<expr>) ]
    "}"

If at least one access rule is defined for a type, then all elements become
invisible/immutable by default and must be made visible/mutable by at least
one ``ALLOW`` rule.

An ``ALLOW`` rule *adds* all objects, for which the check expression ``<expr>``
evaluates to true, to the set of visible objects.  ``ALLOW`` rules are
combined using the ``OR`` operator, i.e. they are mutually additive.

A ``DENY`` rule removes objects for which the check expression ``<expr>``
evaluates to true from the set of visible objects.  ``DENY`` rules are combined
using the ``AND`` operator.

Per ``READ`` and ``DELETE`` policies objects that are outside of the visible
set are silently skipped in any ``SELECT``, ``UPDATE`` or ``DELETE`` expression
that scans the relevant type.

A ``WRITE`` policy specifies a *validity* check for new or updated objects and
affects ``INSERT`` and ``UPDATE`` expressions: if the proposed object is
outside of the visible set, an error is raised immediately and the query is
aborted.

The optional ``<condition>`` expression is evaluated for every object
affected by the statement and the policy is applied only if the expression
evaluates to *true*.  It is essentially equivalent to joining ``<condition>``
with ``<expr>`` with an ``AND`` operator.  The reason for a standalone clause
is that it makes it easier to separate *when* a policy is applied from *how* a
policy is applied.  Also, other access policies do not apply to ``when``
expressions, but they *do* apply to ``using`` expressions.

The check expression ``<expr>`` may be omitted, which implies that the policy
matches all objects, e.g. this is equivalent to specifying ``using (true)``.

Example read policy::

    type Movie {
      property rating -> str;
      # Allow all movie objects to be read by default
      access policy default permit read;
      # But deny those that are rated 'R' to users aged under 17.
      access policy age_appropriate
        when ((global current_user).age < 17)
        deny read using (.rating = 'R');
    }

Example read/write policy::

    type Post {
      property author -> User;

      # Only allow reading to the author, but also
      # ensure that a user cannot set the `author` link
      # to anything but themselves.
      access policy author_only
        allow all using (.author = global current_user);
    }

Another example of combination of allow/deny policies::

    abstract type Owned {
      link owner -> User;

      # permit read access to owner
      access policy owner_only
        allow all using (.owner = global current_user);
    }

    abstract type Shared extending Owned {
      # allow read access to friends
      access policy friends_can_read
        allow read using (global current_user in .owner.friends);
    }

    # Post inherits policies from Shared
    # which allow access to either owner
    # or friends initially...
    type Post extending Shared {
      property private -> bool;

      # ... but restrict access to private posts to owner only
      # regardless of what permissions were granted in parent types
      access policy private_owner_only
        when (.private)
        deny all using (.owner != global current_user);
    }



ALTER ACCESS POLICY
-------------------

Alter the definition of an access control policy.

Required capabilities: DDL.

Synopsis::

    ALTER OBJECT TYPE <type-name> "{"
        ALTER ACCESS POLICY <name>
            [ WHEN <condition> ]
            { ALLOW | DENY }
            { ALL | READ | WRITE | DELETE } [ ... ]
            USING (<expr>)
        [ "{" <subcommand>; [...] "}" ];
    "}"

    # where <subcommand> is one of

      CREATE ANNOTATION <annotation-name> := <value>
      ALTER ANNOTATION <annotation-name> := <value>
      DROP ANNOTATION <annotation-name>


DROP ACCESS POLICY
------------------

Remove an access control policy.

Required capabilities: DDL.

Synopsis::

    ALTER OBJECT TYPE <type-name> "{"
        DROP ACCESS POLICY <name>;
    "}"


Interaction with globals
========================

Access policies are especially powerful when combined with RFC 1010
globals, because then data visibility can be globally adjusted with a single
``SET GLOBAL`` statement, which is very useful for authenticated/authorized
data access control.

Example::

    global user_id -> uuid;

    abstract object type Owned {
      required link owner -> User;

      access policy owner_only
        allow all (.owner.id = global user_id)
    }

    object type Purchase extending Owned;

    ...

    set global user_id := <uuid-1>;
    select count(Purchase);
    # 9
    set global user_id := <uuid-2>
    select count(Purchase);
    # 1


Bypassing policies
==================

A superuser can bypass the execution of query rewrite policies by setting
the ``apply_access_policies`` session configuration setting to ``false``.


Mandatory Role-based Access Control (RBAC)
==========================================

Coupled with the role-based permission system (discussed in a future RFC),
object-level security provides reliable mandatory RBAC, where an
``access policy`` is protected by role permissions and cannot be disabled
by unauthorized users.


Introspection
=============

Policies can be introspected via a new ``schema::AccessPolicy`` in the
introspection schema that is linked from ``schema::ObjectType`` via the new
``access_policies`` link.  The ``schema::AccessPolicy`` is exposed as follows::

    type schema::AccessPolicy extending schema::AnnotationSubject {
      multi property access_kind -> schema::AccessKind;
      property condition -> std::str;
      required property action -> schema::AccessPolicyAction;
      required property expr -> std::str;
    };


Implementation considerations
=============================

Access policies primarily affect what IR is generated for a given EdgeQL query.
``READ`` and ``DELETE`` rules wrap set references and transform every ``Foo``
reference into ``(SELECT Foo FILTER <allow-deny-filter>)``.

``WRITE`` actions insert an intermediate shape into ``INSERT`` and ``UPDATE``,
e.g.::

    INSERT Foo { prop := <value> }

is roughly transformed into::

    WITH
      input := { prop := <value> },
      checked := input {
        prop := prop IF (SELECT _ := <check_expr> FILTER _) ELSE raise()
      }
    INSERT Foo { prop := checked.prop }

If specified, the ``WHEN`` conditions must be taken into account, e.g by
combining directly with the ``ALLOW/DENY`` filters and check expressions.


Rejected Alternative Ideas
==========================

Generalized policy based query rewrite
--------------------------------------

A `previous version of this RFC <https://github.com/edgedb/rfcs/pull/50>`_
proposed a generic "query rewrite" mechanism allowing, besides security,
also trigger-like functionality, but such bundling and generality was
deemed to be too complex, and the decision was made to add explicit mechanisms
for object-level security and (in a future RFC) support for trigger actions.

Use database views (a.k.a. contexts) to implement security
-------------------------------------------------------------------------

A proposal was made to implement security on schema-level instead of
type-level, e.g::

    context Authenticate (auth_method -> AuthMethod, token_id -> str) {
      type view User using (
        SELECT User
        FILTER .session.auth_method = global auth_method
               AND .session.token_id = global token_id);
      type view Sessions using (
        SELECT Sessions
        FILTER .auth_method = global auth_method
               AND .token_id = global token_id );
    }

    context User (user_id -> uuid) {
      type view User using (
        SELECT User Filter .user_id = global user_id);
      type view Article using (
        SELECT Article FILTER .owner.id = global user_id);
      type view PublicArticle using (
        SELECT Article FILTER .public);
    }

Context would then need to be activated::

    SET CONTEXT User { user_id: = <uuid>$user_id };

This proposal was rejected because this design poses significant challenges to
composition, i.e. composing several levels of security without the need to
duplicate large chunks of schema, as well as lack of support for mandatory
access control, as contexts are application-centric and are opt-in.


Backwards compatibility
=======================

This RFC does not pose any backwards compatibility issues.
