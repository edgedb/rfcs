::

    Status: Draft
    Type: Feature
    Created: 2020-01-28
    RFC PR: `edgedb/rfcs#0001 <https://github.com/edgedb/rfcs/pull/1>`_

====================
RFC 1000: Migrations
====================

This RFC describes the design and implementation of schema migrations in
EdgeDB.  This includes related EdgeQL commands, command-line tooling and
some considerations for framework integration.

Motivation
==========

EdgeDB maintains a strict database schema.  This provides strong data integrity
guarantees, code validation and makes the schema a single source of truth.
However, changing the schema requires applying a certain amount of effort and
having a process, both in development and for deployments.

EdgeDB needs to have complete support for schema migrations as a built-in
feature because:

* the database server is the best place to generate, validate and record
  schema migrations, since all mechanisms to ensure schema and data integrity
  are already there;

* built-in support and tools are language- and platform-agnostic, and the
  database users are not tied to a particular framework or library that
  implements migrations as a purely client-side concern.

Overview
========

EdgeDB maintains changes to a database schema as a linear history of
migrations.  A migration is essentially an EdgeQL script that performs
the necessary transformations using DDL and, possibly, regular queries.
In the database, the migrations are recorded as ``schema::Migration`` objects::

    type schema::Migration {
        # Migration name derived from the hash of contents and parent name.
        required property name -> str {
            constraint exclusive;
        };
        # Migration description.
        property message -> str;
        # Parent migrations.
        multi link parents -> schema::Migration;
        # Migration script.
        property script -> array<str>;
    }

The recommended way to define the EdgeDB schema is via SDL files that are
committed into the repository of a project that uses EdgeDB.  The canonical
location for the schema files is the ``dbschema/`` directory in the project's
root folder.

The ``edgedb create-migration`` command reads the files in the schema folder
and creates the migration script with the help of the database server.
Migration scripts are stored to the ``dbschema/migrations/`` directory and
can be committed into the project's repository.  Migration files contain
valid EdgeQL text and can be modified by the user directly, for example, to
insert data migration queries.

The ``create-migration`` command is normally interactive and relies on the
user to provide input in scenarios where the migration is either ambiguous,
or impossible without a data migration.

The ``edgedb migrate`` command is used to bring the database schema to the
specified migration state.  By default, it applies all unapplied migrations
from ``dbschema/migrations/``, but can also be used to migrate to a specified
point in the migration history, possibly reversing migrations that have already
been applied.

The ``edgedb show-status`` command is used to show the current migration
state: whether all recorded migrations have been applied on the server, and
whether there are untracked changes in the local schema that require a
new migration.


Migration History
=================

The migration history is defined as a linear sequence of migrations (although
this might change in the future to support merging migration histories).
Each migration has a unique name that is a hash derived from the token
stream of the migration script and the migration parent name.  To
make migration names valid identifiers the hash is prefixed with letter
``'m'``.

The names of the migrations are computed by the ``edgedb migrate`` command
starting from the latest committed migration (if any), using the following
algorithm:

1. Get the latest migration id from the database.
2. Scans all files in the migrations dir and finds a file that references
   that parent.
3. If there's no such file --- terminate: there's no in-progress migration.
4. if there's one such file and it has the largest sequential file name --
   then there's only one in-progress migration file, so it is committed.
5. If there's a file with that parent and other files after it it means
   that we have a few in-progress (not yet committed) migration files.
   The parent names in them are fluid -- generated and present in the file,
   but aren't yet in the database.

   Scenarios:

   a) A user has two in-progress migration files: 0004.edgeql and 0005.edgeql.
      If the user modifies 0004.edgeql and runs edgedb migration commit, we
      compute the name for 0004 and commit it; we then update the parent name
      in 0005 and commit it too.

   b) A user has two in-progress migration files: 0004.edgeql and 0005.edgeql.
      If the user modifies 0005.edgeql and runs edgedb migration commit,
      we first commit the 0004, validate that 0005 has the correct parent
      name, and if needed we fix it. We also compute it's new name, and then
      commit 00005.

If the tool detects that any of the already-committed migrations have been
altered, it should issue an error and abort.  This RFC makes no special
provision as to the recovery from this situation.  The user will either
have to revert the migration file alteration, or reset the migration history
with ``RESET SCHEMA``.


Migration File Format
=====================

The file format and layout used by the suite of CLI commands described in
this RFC is defined as follows.

Each migration is stored in a separate file.  Files are named using a
simple monotonic decimal counter ('00001.edgeql', '00002.edgeql', etc).
The contents of the file is the body of the migration and must be valid
EdgeQL containing exactly one ``CREATE MIGRATION`` statement that completely
describes a given migration::

    CREATE MIGRATION <name> ONTO <parent-name> {
        SET message := <migration message>;
        <Migration EdgeQL text>
    }


Migration Identifiers
=====================

Each migration has a unique identifier that is derived from its contents.  It
is spiritually analogous to commit ids in Git.  The format of the migration
name is as follows::

    "m" <version> <hash>

    # where
    #   <version>
    #     the single-digit version number of migration name derivation
    #     algorithm, currently only "1" is a valid value;
    #   <hash>
    #     the hash of the migration contents; in version 1 specified as:
    #
    #       lowercase ( base32 ( sha256 ( <tokenstream> ) ) ),
    #
    #     where <tokenstream> is a concatenation of tokens produced
    #     by lexing a CREATE MIGRATION command representing
    #     the migration composed as follows:
    #
    #       CREATE MIGRATION [ ONTO <parent-name> ] "{"
    #          [ SET message := <message> ; ]
    #          <EdgeQL text>
    #       "}"
    #
    #     when concatenating the token stream, each token is separated
    #     by a null character '\x00'.

First revision must have a parent version of `initial`.


Migration operation classification
==================================

The migrations system classifies the migrations operations into two categories:
safe and unsafe, based on whether the operation is automatically reversible
without losing any data present at the beginning of the migration.
For example, all ``CREATE`` operations are considered safe by definition, but
also alterations to schema that doesn't involve data mutation, such as
annotations, indexes, etc. All other operations are classified as unsafe.

Unsafe operations require confirmation in the interactive flows, and raise
an error in non-interactive flows (unless ``--allow-unsafe`` is specified).


Implementation
==============

All migration operations are implemented as EdgeQL statements, no protocol
modifications are necessary.

There are three ways to create and apply a migration:

1. The ``CREATE MIGRATION { ... }`` statement that is used to
   record and apply a previously generated migration.  This is the statement
   used by ``edgedb migrate`` to record and apply new migrations.

2. The ``START MIGRATION TO <schema>; ... COMMIT MIGRATION`` block
   that allows generating migrations using the target SDL specification and
   a set of special commands.  This statement is used by the
   ``edgedb create-migration`` command to generate a migration to a given
   schema state.

3. Any DDL statement executed outside of an explicit migration command creates
   an anonymous migration by wrapping itself with an implicit
   ``CREATE MIGRATION``.

CREATE MIGRATION
----------------

Synopsis::

    CREATE MIGRATION [ <name> ONTO <parent-name> ] "{"
        [ SET message := <message> ; ]
        <subcommand> ; [...]
    "}" ;

    # where
    #   <name>
    #      the name of the migration, autogenerated if not specified;
    #   <parent-name>
    #      optional name of a parent migration, it is an error
    #      to specify any parent other than the last applied
    #      migration;
    #   <message>
    #      optional migration message
    #   <subcommand>
    #      any valid DDL, DML or query, except CONFIGURE,
    #      MIGRATION and TRANSACTION statements.

``CREATE MIGRATION`` executes its body as a normal EdgeQL script and creates
a corresponding ``schema::Migration`` object.  The statement is transactional,
i.e it either succeeds fully or not at all.

START MIGRATION
---------------

Synopsis::

    START MIGRATION TO "{"
        <sdl-declaration> ;
    "}" ;

The ``START MIGRATION`` statement starts a *migration block*, where the
``<sdl-declaration>`` is the desired target state of the database schema as
an SDL declaration.  A transaction is started if none is running already,
otherwise the statement creates a transaction savepoint.  In either case
the migration block is either committed successfully, or not at all.
``START MIGRATION`` records the name of the latest committed migration
as ``parent``, which is verified again when ``COMMIT MIGRATION`` is ran
to ensure that the migration is still valid.

While the migration block is active:

* DDL, DML and query statements are *not executed immediately*, and
  are instead recorded to be part of the final migration text.  To clarify:
  the DDL commands do affect the session schema state, so subsequent statements
  are interpreted as if the preceding DDL commands were applied.
  Like with ``CREATE MIGRATION``, configuration, migration and transaction
  control statements are not allowed, with the exception of
  ``DECLARE SAVEPOINT`` and ``ROLLBACK TO SAVEPOINT``.

* The ``DESCRIBE CURRENT MIGRATION AS JSON`` statement returns a complete
  description of the current migration: statements that have already been
  recorded to be part of the migration script as well as automatically
  generated proposal for the next DDL statement to include to advance
  the migration.  See the "DESCRIBE MIGRATION" section below for details.

* The ``ALTER CURRENT MIGRATION REJECT PROPOSED`` statement is used to
  inform the server that the currently proposed statement returned by
  a prior call to ``DESCRIBE CURRENT MIGRATION AS JSON`` is not acceptable
  and that the next execution of ``DESCRIBE`` should return an alternative
  proposal.  If the client rejected all proposed statements, the next
  ``DESCRIBE`` will return an empty "proposed" value, and it will be
  the responsibility of the client to explicitly issue the necessary
  DDL command to complete the migration.

* The ``POPULATE MIGRATION`` statement uses the statements suggested by
  the database server to complete the migration.

* If an error occurs when the migration block is active, the client can either
  abort the migration with ``ABORT MIGRATION``, or rollback to a known
  savepoint with ``ROLLBACK TO SAVEPOINT``.

* Once the migration script is complete, ``COMMIT MIGRATION`` runs it and
  records the migration.

DESCRIBE MIGRATION
------------------

Synopsis::

    DESCRIBE CURRENT MIGRATION AS JSON;

This is a special form of the ``DESCRIBE MIGRATION`` statement that is valid
only inside a migration block. It returns a full description of the current
migration block: statements that have already been recorded to be part of the
migration, as well as the next statement that is proposed to advance the
migration.  The proposed section also contains the confidence coefficient,
which is an estimate of how likely the database system sees a particular
change to be.

The returned JSON conforms to the following pseudo-schema::

    {
      // Name of the parent migration
      "parent": "m1...",

      // Whether the confirmed DDL makes the migration complete,
      // i.e. there are no more statements to issue.
      "complete": {true|false},

      // List of confirmed migration statements
      "confirmed": [
        "<stmt text>",
        ...
      ],

      // The variants of the next statement
      // suggested by the system to advance
      // the migration script.
      "proposed": {
        "statements": [{
          "text": "<stmt text template>",
          "required-user-input": [{
            "name": "<placeholder variable>",
            "prompt": "<statement prompt>",
          }]
        }],
        "confidence": (0..1), // confidence coefficient
        "prompt": "<variant prompt>",
        "safe": {true|false}
      }
    }

    Where:

      <stmt text>:
        Regular statement text.
      <stmt text template>:
        Statement text template with interpolation points using the \(name)
        syntax.  The client should treat templates and variables as strings
        and perform string interpolation.
      <placeholder variable>:
        The name of an interpolation variable in the statement text template
        for which the user prompt is given.
      <statement prompt>:
        The text of a user prompt for an interpolation variable.
      <variant prompt>:
        Prompt for the proposed migration step variant.

Example::

    {
      "parent": "m1vrzjotjgjxhdratq7jz5vdxmhvg2yun2xobiddag4aqr3y4gavgq",

      "complete": false,

      "confirmed": [
        "CREATE TYPE User { CREATE PROPERTY name -> str }"
      ],

      "proposed": {
        "statements": [{
          "text": "ALTER TYPE Address " +
                  "ALTER PROPERTY number " +
                  "SET TYPE int64 USING \(expr)",
          "required-user-input": [{
            "name": "expr",
            "prompt": "Altering Address.number to type " +
                      "int64 requires an explicit conversion expression",
          }]
        }],
        "confidence": 0.6,
        "prompt": "Did you alter the Address.number property?",
        "safe": false
      }
    }

The algorithm for obtaining the "proposed" data is as follows:

1. Compute schema "A" from the state immediately preceding the
   ``START MIGRATION`` command plus all statements in the "confirmed" list.

2. Compute schema "B" from the SDL declaration passed to ``START MIGRATION``.

3. Calculate the similarity matrix for each pair of objects of the same class
   in schema "A" and schema "B" (e.g. compare every object type in schema "A"
   with schema "B").  Only top-level objects are considered (i.e. object types,
   abstract link definitions etc). The rows of the matrix represent objects
   from schema "A", columns -- objects from schema "B", and the cells contain
   *similarity index*, which is a floating point value from ``0.0`` to ``1.0``,
   where ``0`` means "completely dissimilar" and ``1.0`` means "identical".
   The similarity score is set to ``0`` if the command generated by this pair
   in a previous run was rejected by ``ALTER CURRENT MIGRATION REJECT
   PROPOSED``.  For example::

        +-------+-----+-----+-----+
        | A \ B | Foo | Bar | Baz |
        +-------+-----+-----+-----+
        | Foo   | 1.0 | 0.2 | 0.1 |
        +-------+-----+-----+-----+
        | Bar   | 0.2 | 0.9 | 0.3 |
        +-------+-----+-----+-----+

4. Exclude all objects that are considered to be the same in both schemas from
   the similarity matrix, i.e. exclude rows and columns that contain a ``1.0``
   value.

5. For each object in schema "B" in the similarity matrix, find the cell with
   the greatest similarity score.  If the similarity score is greater than
   ``0.6`` assume the object has been ``ALTER``-ed and generate an appropriate
   DDL command.  Exclude the row and the column from further consideration.

6. For each remaining row in the matrix, assume object deletion and generate
   the appropariate ``DROP`` DDL.  For each remaining column in the matrix,
   assume new object and generate the appropriate ``CREATE`` DDL.

7. Sort the generated DDL commands according to the dependency topology.

8. Pick the first DDL statement from the sorted list.  This is the new
   "proposed" statement.


ALTER CURRENT MIGRATION REJECT PROPOSED
---------------------------------------

This statement is only valid while inside the ``START MIGRATION`` block.
When executed it marks the statement found in the "proposed" branch of
``DESCRIBE CURRENT MIGRATION AS JSON`` as "unacceptable" and forces the
server to recompute the diff with this command excluded.  This, effectively
allows the client to implement the (y) / (n) flow for each proposed statement
in the migration.  For (y) the client would send back the proposed statement
to be recorded in the "confirmed" list, and for (n) an ``ALTER CURRENT
MIGRATION REJECT PROPOSED`` is sent instead.


REVERT SCHEMA
-------------

The ``REVERT SCHEMA`` statement is used to revert the schema to a state
at a given migration.  Migrations committed after the specified migrations
are reverted in reverse order, and reverts are recorded as migrations
themselves.  Each affected migration must be reversible.

Synopsis::

    REVERT SCHEMA TO <migration> ;

RESET SCHEMA
------------

The ``RESET SCHEMA`` statement is used to *reset* the schema to a state
at a given migration.  The difference from ``REVERT SCHEMA`` is that affected
migrations are *not* recorded as reverts, and the resulting state looks like
they never have been applied at all.  Each affected migration must be
reversible.

Synopsis::

    RESET SCHEMA TO <migration> ;

Bare DDL
--------

Each individual DDL command executed outside a migration block gets wrapped
into an implicit ``CREATE MIGRATION`` regardless of whether it is a part of a
transaction or not.  This is necessary to correctly track the state of the
schema.

Use of bare DDL for the purpose of schema migrations is discouraged.
To enforce a "no-bare-DDL" policy, the ``allow_bare_ddl`` configuration option
may be set to ``false``, which will prohibit all DDL operations outside of
migration blocks.

Discussion
==========

Downsides of the selected approach
----------------------------------

The approach described in this RFC requires a server connection to generate
migrations.

Design considerations
---------------------

``ABORT MIGRATION`` is chosen instead of ``ROLLBACK MIGRATION``, or even
``ROLLBACK`` because the first can be confused with a migration revert command,
and ``ROLLBACK`` would terminate the entire transaction, whereas the
migration block might only be a subset of it.

The identifiers of the schema objects are not preserved in the migration,
because those are internal to the database instance and should generally not
be relied upon by the clients.  This is similar to OID values in Postgres.

The current design does not allow multiple migration parents (i.e. migration
history merges), but neither does it prohibit the concept as a future
feature.

A variant of ``START TRANSACTION`` without an explicit schema target was
considered to create a "free form" migration using DDL statements, but it's
unclear if such a feature is useful at this moment.


Multiple choice proposals in DESCRIBE MIGRATION
-----------------------------------------------

An earlier version of this RFC proposed to include mutltiple scenarios in the
"proposed" section of ``DESCRIBE CURRENT MIGRATION AS JSON`` to be presented
as multiple choice by the client.  That approach has major downsides:

1) Different scenarios may create significantly different DDL dependency paths,
   for example::

       type Foo;

       type Bar {
           link spam -> Foo;
       };

   migrated to::

       type Ham;

       type Bar {
           link spam -> Ham;
       };

   If the change from ``Foo`` to ``Ham`` is interpreted as an ``ALTER``,
   then only the following DDL will be necessary::

       ALTER TYPE Foo RENAME TO Ham;

   Alternatively, if the change is interpreted as a ``DROP``/``CREATE``,
   the operation is as follows::

       CREATE TYPE Ham;
       // This is a destructive operation, because
       // all current "spam" links would be dropped.
       ALTER TYPE Bar ALTER LINK spam SET TYPE Ham;
       DROP TYPE Foo;

   In a complex migration it will be very difficult to split, attribute
   and sort the schema diff to make sure that both the ``RENAME`` variant,
   and the ``DROP`` variant get sorted consistently at the top of the
   proposed DDL script, considering that there may be other dependencies
   on the involved types.

2) Listing all potential variants right away might be overwhelming in
   certain scenarios.  Simplified example::

      type Foo {
          property a -> str;
      };

   migrated to::

      type Foo {
          property b -> str;
          property c -> str;
          property d -> str;
          property e -> str;
      };

   There are 5 possible scenarios: 1) `a` renamed to `b`,
   2) `a` renamed to `c`, 3) `a` renamed to `d`, 4) `a` renamed to `e`,
   5) `a` dropped.  In a complex migration the potential space
   for the list of variants may be enormous.
