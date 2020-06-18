::

    Status: Final
    Type: Guideline
    Created: 2020-06-17
    RFC PR: `edgedb/rfcs#0001 <https://github.com/edgedb/rfcs/pull/1>`_

=======================================================
RFC 1002: Optionality qualifier in properties and links
=======================================================

This RFC describes the reasons why DDL and SDL declarations for properties
and links assume optionality by default.


Motivation
==========

The data definition language, and more importantly, the schema definition
language, which users use to communicate with EdgeDB, need to avoid
unnecessary verbosity.

This requires some conventions and assumptions about what the user expects
most of the time. Those assumptions shouldn't bar the user from changing
their mind when the need arises.

A default value has far reaching consequences which we cover in this
document.


Implementation
==============

Since the discussion always applies to properties and links, we're referring
to them simply as *pointers* below.

Assume OPTIONAL qualifiers when no explicit qualifier is provided
-----------------------------------------------------------------

This is the currently implemented behavior as of EdgeDB v1.0 alpha 3.

An optional pointer is a form of incomplete data that allows the application
to flexibly decide how to proceed. If the data is there, fine. If it isn't,
the application can include user-level code to deal with that.

Moreover, multi properties and multi links make the case of an empty set
intuitively acceptable: since the cardinality allows for any number of
elements, a state with an empty set is not surprising.

Plenty of examples of existing databases and RPC systems assume optionality
by default. Relational databases assume optionality unless a NOT NULL
constraint is passed. Protocol buffers removed support for required fields
altogether in version 3. JSON-based NoSQL databases like MongoDB assume
optionality.

Promoting an optional field into a required field doesn't create silent
failures in pre-existing queries.

Acknowledged downsides
~~~~~~~~~~~~~~~~~~~~~~

Developers might be failing to fill pointers at INSERT time due
to forgetfulness at query creation time, even if the data could be
provided. They might be harder to fill at a later stage.

Optionality is a special case that the user code has to deal with. Empty sets
in queries might leave to unexpected results, i.e. missing objects. An empty
set is a special case of a default value. A better default might be possible
for a given pointer.

Assuming optionality makes it laborious for the user to change their mind
later, i.e. alter a pointer to become required:

1. there might already be objects in the database with an empty set in
   the given pointer; and
2. client code written to insert data to the database or update data in
   it might become invalid if it depended on the optionality.

The former issue may be addressed with a data migration prior to the
schema migration.  The latter problem is out of scope for EdgeDB but
can be handled by migrating client code *first*.  If the clients aren't
under the database administrator's control, a reasonable choice in this
case is to abort the attempt to make a pointer required.

Additionally, optionality by default in pointers is inconsistent with
non-optionality in function arguments.


Rejected ideas
==============

Assume REQUIRED qualifiers when no explicit qualifier is provided
-----------------------------------------------------------------

An extreme form of optionality everywhere is equivalent to the database
not having a true schema. Unless it's necessary to make a pointer optional,
forcing pointers to be required ensures that whenever data *can* be specified,
it won't be forgotten at INSERT or UPDATE time.

Required pointers simplify type information. There's two particular problems
on the client side with unexpected optionality:

* lack of handling for that special case in user code, and

* surprising effects of EdgeQL queries that omit objects with unexpected
  empty sets in pointers used in the query.

However, since many pointers in real-world schemas will be declared as OPTIONAL
regardless of the default so the downsides of optionality wouldn't be
solved by using REQUIRED as the default qualifier.

Assuming that all pointers are required unless otherwise specified makes it
easier to migrate into optionality later. Client code that was forced to
provide values at INSERT time before will continue working and there are no
incompatible empty sets in the database prior to the migration.  However,
see "Downsides" below.

Downsides
~~~~~~~~~

The main disadvantage at time of writing is that the current behavior as of
EdgeDB v1.0a3 is the opposite: pointers without an explicit qualifier are
assumed optional. This poses a `Migration challenge`_.

Required pointers have to be always specified at INSERT time, making
queries more verbose to write and data more costly to send over the wire.
This can be dealt with by providing default values with SET DEFAULT which
is a form of optionality with an explicit fallback value which doesn't fall
outside of the declared type.

Required pointers by default make it more likely that users would have to
make required-to-optional pointer migrations in the future.  Those hide
a specific gotcha: pre-existing queries that were previously guaranteed
never to encounter empty sets now will.  Most of the time this will be
harmless but some queries that are analytical in nature will now fail to
return objects with empty pointers, which might lead to bugs in user code.

Migration challenge
~~~~~~~~~~~~~~~~~~~

All existing EdgeDB instances assume optionality so a migration would be
in order. The trickiness of that migration is a significant flaw of
the proposal to change the current behavior.

As of Alpha 3, DESCRIBE already includes explicit OPTIONAL qualifiers
everywhere. However, users' ``.esdl`` files and ``.edgeql`` files with
DDL commands don't.

In consequence, while a DUMP and RESTORE from Alpha 3 will likely work
in the face of a change to implicit REQUIRED, user code does not expect
this.

For user safety, EdgeDB would need an intermediate release that forces all
pointer definitions to specify qualifiers explicitly. Since there is little
adoption of the product so far, this might not be a big problem.

Do not provide a default optionality qualifier
----------------------------------------------
We rejected this idea because it made even the simplest DDL and SDL
declarations much wordier. This verbosity was detrimental to readability and
required more typing from the user.

More importantly, despite the usability sacrifice, this approach would solve
the issue only partially. It is still perfectly possible for the user to
choose the wrong qualifier initially, for instance by making the wrong
assumption about a given pointer, or by blindly copying it from somewhere else.


Other observations
==================

A database is not a peer-to-peer RPC platform
---------------------------------------------

Thrift documentation explains that "required is forever". What they mean by
this is that required fields have to be provided by the caller, say a mobile
device. Changing a required field into an optional field requires for the RPC
server to be updated first before any RPC client code can use this
capability.

While this is a valid concern for RPC systems where new clients won't be able
to connect to old servers, in the case of central databases, the most popular
deployment scheme already is to migrate the database first. The central base
is an easier target to control. It's true that there's a risk that this
ordering will not be kept if the schema definition files are shared between
teams responsible for backend and frontend code.

The concerns listed by maintainers of protocol buffers and Thrift don't
seem like they apply to a database which is set up as a central API layer
and the source of truth for the given data.

Protocol buffers removed required fields in version 3, but they also
removed custom default values and rejected the idea of custom validators.
EdgeDB supports both of those features, the latter being constraints.

The REQUIRED qualifier is a special form of a constraint
--------------------------------------------------------

In this sense, specifying a constraint first and removing it later is
easier to deal with than the opposite operation. Not only is the migration
easier but specifying a constraint early usually leads to better data
quality and avoids user-side bugs.
