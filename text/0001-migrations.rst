..
    Status: Draft
    Type: Feature
    Created: 2020-01-28
    RFC PR: `edgedb/rfcs#0001 <https://github.com/edgedb/rfcs/pull/1>`_
    EdgeDB Issue: `edgedb/edgedb#0000 <https://github.com/edgedb/edgedb/issues/0000>`_

====================
RFC 0001: Migrations
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
In the database, the migrations are recorded as ``sys::Migration`` objects.

The recommended way define the EdgeDB schema is via SDL files that are
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
user to provide input in schenarios where the migration is either ambiguous,
or impossible without a data migration.

The ``edgedb migrate`` command is used to bring the database schema to the
specified migration state.  By default, it applies all unapplied migrations
from ``dbschema/migrations/``, but can also be used to migrate to a specified
point in the migration history, possibly reversing migrations that have already
been applied.

Implementation
==============

All migration operations are implemented as EdgeQL statements, no protocol
modifications are necessary.

There are three ways to create and apply a migration:

1. The ``CREATE MIGRATION <name> { ... }`` statement that is used to
   record and apply a previously generated migration.  This is the statement
   used by ``edgedb migrate`` to record and apply new migrations.

2. The ``START MIGRATION <name> TO <schema>; ... COMMIT MIGRATION`` block
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

    CREATE MIGRATION <name> "{"
        <subcommand> ;
        [...]
    "}" ;

    # where <subcommand> is any valid DDL, DML or query,
    # except CONFIGURE, MIGRATION and TRANSACTION statements.

``CREATE MIGRATION`` executes its body as a normal EdgeQL script and creates
a corresponding ``sys::Migration`` object.  The statement is transactional,
i.e it either succeedes fully or not at all.

START MIGRATION
---------------

Synopsis::

    START MIGRATION <name> TO "{"
        <sdl-declaration> ;
    "}" ;

The ``START MIGRATION`` statement starts a *migration block*, where the
``<sdl-declaration>`` is the desired target state of the database schema as
an SDL declaration.  A transaction is started if none is running already,
otherwise the statement creates a transaction savepoint.  In either case
the migration block is either committed successfully, or not at all.

While the migration block is active:

* Any statement issued to the database, *instead of being executed*, is
  recorded to be part of the migration. Like with ``CREATE MIGRATION``,
  configuration, migration and transaction control statements are not
  allowed, with the exception of ``DECLARE SAVEPOINT`` and ``ROLLBACK TO
  SAVEPOINT``.

* The ``DESCRIBE CURRENT MIGRATION AS JSON`` statement returns a complete
  description of the current migration: statements that have already been
  recorded to be part of the migration script as well as automatically
  generated list of statements required to complete the migration.  See
  the "DESCRIBE MIGRATION" section below for details.

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
migration, as well as the generated list of statements required to complete
the migration. The latter may possibly contain *alternatives* for statements
where there is no certainty, i.e. an ``ALTER`` vs a ``DROP + CREATE``.
Additionally, each DDL statement may be accompanied by other metadata, such
as the indication to provide a data migration expression for alterations
that require it.

The returned JSON conforms to the following pseudo-schema::

    {
      // List of confirmed migration statements
      "confirmed": [
        "<stmt text>",
        ...
      ],

      // List of proposed migration steps
      "proposed": [{
        // List of proposed variants for the migration step
        "variants": [{
          "statements": [{
            "text": "<stmt text template>",
            "required-user-input": [{
              "name": "<placeholder variable>",
              "prompt": "<statement prompt>",
            }]
          }],
          "confidence": (0..1) // confidence coefficient
          "prompt": "<variant prompt>"
        }, ... ]
      }, ... ]
    }

    Here:

      <stmt text>:
        Regular statement text.
      <stmt text template>:
        Statement text template with interpolation points using the \(name)
        syntax.
      <placeholder variable>:
        The name of an interpolation variable in the statement text template
        for which the user prompt is given.
      <statement prompt>:
        The text of a user prompt for an interpolation variable.
      <variant prompt>:
        Prompt for the proposed migration step variant.

Example::

    {
      "confirmed": [
        "CREATE TYPE User { CREATE PROPERTY name -> str }"
      ],

      "proposed": [{
        "variants": [{
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
          "confidence": 0.6
          "prompt": "Did you alter the Address.number property?"
        }, {
          "statements": [{
            "text": "ALTER TYPE Address { " +
                    "DROP PROPERTY number; " +
                    "CREATE PROPERTY number -> int64; }"
          }],
          "confidence": 0.4
        }]
      }]
    }

Discussion
==========

Downsides of the selected approach
----------------------------------

The apprach described in this RFC requires a server connection to generate
migrations.
