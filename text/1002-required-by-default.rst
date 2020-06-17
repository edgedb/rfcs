..
    Status: Draft
    Type: Guideline
    Created: 2020-06-17
    RFC PR: `edgedb/rfcs#0001 <https://github.com/edgedb/rfcs/pull/1>`_

===============================================================
RFC 1002: REQUIRED qualifier by default in properties and links
===============================================================

This RFC describes the reasons why DDL and SDL declarations for properties
and links assume non-optionality by default.


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
to them simply as *fields* below.

Assume REQUIRED qualifiers when no explicit qualifier is provided
-----------------------------------------------------------------

The main advantage of assuming that all fields are required unless otherwise
specified is its resilience to migration into optionality later. Client code
that was forced to provide values before will continue working and there are
no incompatible empty sets in the database prior to the migration.

An extreme form of optionality everywhere is equivalent to the database
not having a true schema. Unless it's necessary to make a field optional,
forcing fields to be required ensures that whenever data *can* be specified,
it won't be forgotten at INSERT or UPDATE time.

Required fields simplify type information. There's two particular problems
on the client side with unexpected optionality:

* lack of handling for that special case in user code, and

* surprising effects of EdgeQL queries that omit objects with unexpected
  empty sets in fields used in the query.

Downsides
~~~~~~~~~

* The main disadvantage at time of writing is that the current behavior as of
  EdgeDB v1.0a3 is the opposite: fields without an explicit qualifier are
  assumed optional. This poses a migration challenge.

* Required fields have to be always specified at INSERT time and often
  specified at UPDATE time, making queries more verbose to write and data
  more costly to send over the wire. This can be dealt with by providing
  default values with SET DEFAULT which is a form of optionality with an
  explicit fallback value which doesn't fall outside of the declared type.

* Thrift documentation explains that "required is forever". What they mean
  by this is that required fields have to be provided by the caller, say a
  mobile device. Changing a required field into an optional field requires
  for the database to be updated first before any application code can use
  this capability. While this is a valid concern for RPC systems where new
  clients won't be able to connect to old servers, in the case of central
  databases, the most popular deployment scheme already is to migrate the
  database first. The central base is an easier target to control. It's true
  that there's a risk that this ordering will not be kept if the schema
  definition files are shared between teams responsible for backend and
  frontend code.


Rejected ideas
==============

Assume OPTIONAL qualifiers when no explicit qualifier is provided
-----------------------------------------------------------------

This is the current behavior.

An optional field is a form of incomplete data that allows the application
to flexibly decide how to proceed. If the data is there, fine. If it isn't,
the application can include user-level code to deal with that.

Plenty of examples of existing databases and RPC systems assume optionality
by default. Protocol buffers removed support for required fields altogether
in version 3. JSON-based NoSQL databases like MongoDB assume optionality.
HTML form fields are optional by default.

Downsides
~~~~~~~~~

Developers might be failing to fill fields at INSERT time due
to forgetfulness at query creation time, even if the data could be
provided. They might be harder to fill at a later stage.

Optionality is a special case that the user code has to deal with. Empty sets
in queries might leave to unexpected results, i.e. missing objects. An empty
set is a special case of a default value. A better default might be possible
for a given field.

Assuming optionality makes it harder for the user to change their mind later,
i.e. alter a field to become required:

1. there might already be objects in the database with an empty set in
   the given field; and
2. client code written to insert data to the database or update data in
   it might become invalid if it depended on the optionality.

The former issue may be addressed with a data migration prior to the
schema migration.  The latter problem is out of scope for EdgeDB but
can be handled by migrating client code *first*.  If the clients aren't
under the database administrator's control, a reasonable choice in this
case is to abort the attempt to make a field required.

Do not provide a default optionality qualifier
----------------------------------------------
We rejected this idea because it made even the simplest DDL and SDL
declarations much wordier. This verbosity was detrimental to readability and
required more typing from the user.

More importantly, despite the usability sacrifice, this approach would solve
the issue only partially. It is still perfectly possible for the user to
choose the wrong qualifier initially, for instance by making the wrong
assumption about a given field, or by blindly copying it from somewhere else.


Migration
=========

All existing EdgeDB instances assume optionality so a migration is in
order. The trickiness of that migration might be the biggest flaw of
the proposal to change the current behavior.

As of Alpha 3, DESCRIBE already includes explicit OPTIONAL qualifiers
everywhere. However, users' ``.esdl`` files and ``.edgeql`` files with
DDL commands don't.

In consequence, while a DUMP and RESTORE from Alpha 3 will likely work
in the face of a change to implicit REQUIRED, user code does not expect
this.

For user safety, EdgeDB would need an intermediate release that forces all
field definitions to specify qualifiers explicitly. Since there is little
adoption of the product so far, this might not be a big problem.


Other observations
==================

A database is not a peer-to-peer RPC platform
---------------------------------------------

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
