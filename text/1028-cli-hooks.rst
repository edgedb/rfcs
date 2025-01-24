::

    Status: Draft
    Type: Feature
    Created: 2025-01-14
    Authors: Victor Petrovykh <victor@edgedb.com>

===================
RFC 1028: CLI hooks
===================

This RFC discusses addition of hooks to CLI commands. In order to make it
easier to setup and maintain some of the things dependent on the schema state
we can introduce hooks for some of the schema altering commands.


Motivation
==========

There are a number of tools that generate code based on the current schema
(such as query-building tools, or query reflection tools). They need to be
re-run after every schema change in order to keep the generated code in sync
with the database. In larger projects the number of things that may need
updating after a schema change is likely to grow. Migration hooks could make
it easier to automate these tasks.

There is also a longstanding need to implement some kind of mechanism for
creating data fixtures. Unlike the migration hooks, fixtures are more likely
to be run at project initialization or after a branch wipe command.


Hooks Configuration
===================

All hooks are intended to be used within an active project, not for arbitrary
CLI commands. Thus if a CLI command receives an explicit ``-I <instance>``
option, the command should not trigger any project hooks even if the instance
is linked to a project. Only commands that omit the instance and use the
current project to determine the target instance are going to trigger hooks.

The hooks will be described in the ``gel.toml`` file in the
``[project-hooks]`` table. We define ``migration.apply.after``,
``project.init.after``, ``branch.wipe.after``, and ``branch.switch.after``
hook keys. The values for them are arrays of strings, where each string is
going to be executed as a shell command. The naming scheme is intended to
mimic the command structure to clearly indicate which commands will trigger
the hooks. Current RFC only introduces hooks to be executed after a given
command, but the naming scheme supports future "before" hooks as well.

The scripts are executed in the order they appear in the array, sequentially.
So if you have multiple steps that needs to be executed in a certain order,
you can put the hook commands in the same order you would have executed them
in the shell.

All hooks will use the project root directory as the execution directory.


Configuration Hooks
-------------------

This category of hooks are intended for one-time setup of important fixtures
or application configuration. They are perfect for setting up data fixtures or
exporting connection strings for non-gel tools to use. There are generally two
scenarios when this kind of hook might be needed:

1) After ``gel project init`` the ``project.init.after`` hook will be
   executed. This is good for updating any additional configuration that
   depends on the project database. It is also a good place for data fixture
   scripts.

2) After ``gel branch wipe`` the ``branch.wipe.after`` hook will be executed.
   This may be a good place for restoring the data fixtures. However it is
   probably unnecessary to run any configuration scripts at this time. This is
   why this hook does not simply run all the same scripts as the
   ``project.init.after``.

3) After ``gel branch switch`` command that changes the current branch the
   ``branch.switch.after`` hook will be executed. This is a good place for
   updating any configuration scripts (e.g. updating Postgres connection
   string). This hook can also be used to keep the source code branch in sync
   with the database branch. This is probably not a good hook for running
   data fixtures as the data in branches in unaltered between switches.

If ``project init`` runs migrations, the ``project.init.after`` hook is
triggered *before* the ``migration.apply.after`` hook. The motivation is that
whatever is necessary as a one-time project setup is likely to be a
pre-requisite for migration hooks.

Similarly, ``branch.wipe.after`` and ``branch.switch.after`` hooks are
executed before the migration hook triggers.


Migration Hooks
---------------

This category of hooks is intended for scripts that sync up the codebase with
the schema state. They are defined by the ``migration.apply.after`` setting.
It may be re-compiling the query-builder or reflected individual queries. It
may be generating an ORM compatibility layer. It could also perform any
project-specific custom operation.

The effects produced by these hook scripts should be idempotent, so that
running them several times on the same schema (or nearly same schema with
trivial changes) is safe.

The hooks should run *only once* per ``migration apply`` command, as long as
some pending change was applied. They are not intended to run after applying
every migration file if several migrations are applied by one command.

The ``gel watch`` command will also trigger these hooks (without a need of a
separate TOML config section). This is due to the fact that the ``watch``
command conceptually monitors the schema file changes and automatically
creates and applied migrations. So after the schema file changes are applied
to the database, the ``migration.apply.after`` hook will be executed.

The ``gel branch`` commands may also trigger these scripts since different
branches can have different schemas. The ``switch``, ``rebase``, ``merge``,
and ``wipe`` branch commands all potentially change the current schema. This
means that the hooks associated with applying schema changes must be executed
for these commands as well.


Design Considerations
=====================

It makes sense to follow a convention of filling out the ``[project-hooks]``
table in order of execution priority from highest to lowest::

    [project-hooks]
    project.init.after=[
      "setup_dsn.sh"
    ]
    branch.wipe.after=[]
    branch.switch.after=[
      "setup_dsn.sh"
    ]
    migration.apply.after=[
      "gel-orm sqlalchemy --mod compat --out compat"
    ]

The order in which hook *keys* appear does not impact their priority (we don't
want people getting subtle bugs due to different key order). It would simply
be a convention for any of our examples or auto-generated TOML configs.

These are mostly intended to be development aids so we may need an additional
mechanism for distinguishing cloud *development* and *production* instances to
prevent accidental triggering of the hooks in production.


Rejected Ideas
==============

We don't want any extra environment variables to be setup for the hooks. You
could use whatever you would have used if you ran the scripts by hand from the
shell. For example, to get the instance name ``gel project info
--instance-name``. The idea is that the scripts wouldn't be relying on any
hook-specific magic and you could run them (and thus debug them) by hand with
the identical effects.