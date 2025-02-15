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

Also, there is a need to watch for file changes (schema, queries) and
potentially respond to these by running some scripts.


Hooks Configuration
===================

All hooks are intended to be used within an active project, not for arbitrary
CLI commands. Thus if a CLI command receives an explicit ``-I <instance>``
option, the command should not trigger any project hooks even if the instance
is linked to a project. Only commands that omit the instance and use the
current project to determine the target instance are going to trigger hooks.

The hooks will be described in the ``gel.toml`` project manifest file in the
``[hooks]`` table. We define the following hook keys:

* ``project.init.before``
* ``project.init.after``
* ``branch.switch.before``
* ``branch.switch.after``
* ``branch.wipe.before``
* ``branch.wipe.after``
* ``migration.apply.before``
* ``migration.apply.after``
* ``schema.update.before``
* ``schema.update.after``

Their values are strings that are going to be executed as shell commands.

The naming scheme is intended to mimic the command structure to clearly
indicate which commands will trigger the hooks. The exception to that is the
``schema.update`` hooks that are intended to trigger in response to any kind
of schema change, regardless of which command causes it.

All hooks will use the project root directory as the execution directory.
Hooks are executed using ``/bin/sh`` on all platforms. On Windows, the hooks
are always executed in WSL.

If the shell exits with a non-zero status code, the CLI will exit immediately,
without executing any subsequent hooks or CLI actions.

The hooks have two versions: ``before`` and ``after``. The ``before`` hooks
are intended to be triggered prior to the command making any changes. That
way, any error while executing a ``before`` hook would cause the command to
fail before it applies any changes. The ``after`` hooks are intended to
trigger after all the direct effects of the triggering command have been
resolved. The ``after`` hooks are intended to work with the new and updated
state of Gel.


Command Hooks
-------------

This category of hooks are intended for one-time setup of important fixtures
or application configuration. They are perfect for setting exporting
connection strings for non-gel tools to use. They are primarily triggered by
the corresponding CLI commands, but sometimes commands may trigger more than
one hook:

* ``gel project init`` command triggers the ``project.init.before``
  and ``project.init.after`` hook. If the migrations are applied at the end of
  the initialization, then the ``migration.apply.before``,
  ``schema.update.before``, ``migration.apply.after``, and
  ``schema.update.after`` hooks are also triggered.
* ``gel branch switch`` command triggers ``branch.switch.before``,
  ``schema.update.before``, ``branch.switch.after``, and ``schema.update.after``
  hooks in that relative order.
* ``gel branch wipe`` command triggers the ``branch.wipe.before``,
  ``schema.update.before``, ``branch.wipe.after``, and ``schema.update.after``
  hooks in that relative order.
* ``gel branch rebase`` and ``gel branch merge`` commands trigger
  ``migration.apply.before``, ``schema.update.before``,
  ``migration.apply.after``, and ``schema.update.after`` hooks in that
  relative order. Notice that although these are branch commands, but they do
  not change the current branch, instead they modify and apply migrations.
  That's why they trigger the ``migration.apply`` hooks.
* ``gel migrateion apply`` (or ``gel migrate``) command triggers
  ``migration.apply.before``, ``schema.update.before``,
  ``migration.apply.after``, and ``schema.update.after`` hooks in that
  relative order.

Overall the order in which the hooks are triggered is this:

#. ``project.init.before``
#. ``project.init.after`` (signifying that the project is initialized
   and thus ready for the rest of the tools/hooks).
#. ``branch.switch.before``
#. ``branch.wipe.before``
#. ``migration.apply.before``
#. ``schema.update.before``
#. ``branch.switch.after``
#. ``branch.wipe.after``
#. ``migration.apply.after``
#. ``schema.update.after``


Schema Hooks
------------

These hooks are a bit more abstract than the ones corresponding to specific
CLI commands. The ``schema.update.before`` and ``schema.update.after`` hooks
are intended for scripts that sync up the codebase with the schema state
(regardless of what caused the schema state to change). It may be re-compiling
the query-builder or reflected individual queries. It may be generating an ORM
compatibility layer. It could also perform any project-specific custom
operation, like integration tests, type-checkers, or linters.

The effects produced by these hook scripts should be idempotent, so that
running them several times on the same schema (or nearly same schema with
trivial changes) is safe.

The ``gel watch`` command will also trigger these hooks (without a need of a
separate TOML config section). This is due to the fact that the ``watch``
command conceptually monitors the schema file changes and automatically
creates and applied migrations. So the schema file changes would trigger
``migration.apply.before``, ``schema.update.before``,
``migration.apply.after``, and ``schema.update.after`` hooks in that relative
order.


Watch Configuration
===================

Sometimes in a Gel project there's a need to respond to some file changes: run
a migration due to schema change or run code generators due to query file
change. The CLI already supports watching the schema for changes and applying
them, but we can generalize this mechanism and make it more flexible, much
like the hooks that respond to CLI commands.

The idea is to have a mechanism for specifying a file system path you want to
watch for modifications and a script that gets triggered by those changes.
Unlike hooks, there are no "before" and "after" triggers here. Only one kind
of response is possible: trigger when a change to the watched entity is
detected.

The watch configuration will be described in the ``gel.toml`` project manifest
file in the ``[watch]`` table. We will introduce a ``[watch.files]`` sub-table
for watching file system changes and triggering scripts.

The general structure of watch tables is going to use the key (potentially
quoted) to specify *what is being watched* and the corresponding value to
specify the script that will be triggered by changes.

The ``gel watch`` command (without needing further options) would then be used
to start the watch process and monitor whatever is specified in the
``gel.toml``. By default ``gel watch`` will only watch the files specified in
the ``gel.toml`` config and execute the trigger scripts. Running ``gel watch
--migrate`` will additionally monitor schema changes and perform real-time
migrations.

This is a backwards incompatible change compared to how ``edgedb watch`` operates
now.

The output of the scripts will then appear in the same TTY as the ``gel
watch`` command.

We may want to setup a debouncer so that we can delay before triggering the
scripts on a sequence of changes. This is mostly to reduce unnecessary
multiple triggers for the same watched entity.

Another consequence of executing triggered scripts in the background is that
sometimes the changes are invalid in some way and the script will fail. This
is considered part of normal operation (e.g. syntax error in a saved query
file) and the scripts should be such that failure does not create some
non-recoverable state.

Due to the nature of monitoring changes, we cannot guarantee any particular
order in which the watch scripts will be triggered when multiple changes occur
at once. This is independent of the source of multiple triggers: whether it is
due to multiple files being updated or multiple watch rules matching the same
file. The scripts are effectively triggered asynchronously and the exact order
is an implementation detail that should not be relied upon.


Files
-----

The ``[watch.files]`` table should contain keys that are interpreted as file
system paths. They can also contain common glob patterns (such as provided by
`this library <https://docs.rs/globset/latest/globset/#syntax>`_).

The paths should be valid in the underlying file system (therefore using ``/``
for Linux and ``\`` for Windows, etc.). The relative file paths are assumed to
start at the project root.

The values corresponding to the keys are strings that are going to be executed
as shell commands, much like for the hooks.

All watch scripts will use the project root directory as the execution
directory. They are executed using ``/bin/sh`` on all platforms. On Windows,
the scripts are always executed in WSL.

An example of this configuration::

    [watch.files]
    "queries/*.edgeql"="npx @edgedb/generate queries"

Only files in the project directories can be watched. If the ``gel.toml``
config specifies files outside of the project to be watched it should cause an
error for the ``gel watch`` command. The invalid spec should not be ignored
(silently or with a warning).


CLI Interactions
================

All of the above settings are intended as *project* settings. Which means that
the hooks can only be triggered by *project-specific* commands. If a command
overrides the default project settings (custom instance, branch, etc.) we can
no longer assume that the project hook is valid for that command and no hooks
will be triggered.

Similarly, the watch settings are invalid if they attempt to monitor files
outside of the project directory structure.


Design Considerations
=====================

It makes sense to follow a convention of filling out the ``[hooks]``
table in order of execution priority from highest to lowest::

    [hooks]
    project.init.after="setup_dsn.sh"
    branch.wipe.after=""
    branch.switch.after="setup_dsn.sh"
    schema.update.after="gel-orm sqlalchemy --mod compat --out compat"

The order in which hook *keys* appear does not impact their priority (we don't
want people getting subtle bugs due to different key order). It would simply
be a convention for any of our examples or auto-generated TOML configs.

These are mostly intended to be development aids so we may need an additional
mechanism for distinguishing cloud *development* and *production* instances to
prevent accidental triggering of the hooks in production.


Future Possibilities
====================

The debouncer delay can be configurable in the ``[watch]`` section as
``deboucer-delay=<integer>`` with the value being the delay in milliseconds.

We can introduce a ``[watch.gel-config]`` sub-table for monitoring changes to
the various database config values and responding to them.

We can set up a logfile for watch scripts (because that creates a record which
survives closing of terminals or reboots) as an additional convenience
feature. This can be specified in the general ``[watch]`` section as
``logfile="<path-to-logfile>"``.

The process launched by a script could have additional environmental variables
such as `GEL_HOOK_NAME` with value, for example, `branch.wipe.before`.

Watch could have a `--retry-sec` flag, which would retry running a script
if it had exited with a non-zero status code.

Rejected Ideas
==============

We don't want any extra environment variables to be setup for the hooks. You
could use whatever you would have used if you ran the scripts by hand from the
shell. For example, to get the instance name ``gel project info
--instance-name``. The idea is that the scripts wouldn't be relying on any
hook-specific magic and you could run them (and thus debug them) by hand with
the identical effects.

We don't want to provide a list of scripts to run for hooks and watch
triggers. This is because with a list of scripts we need to specify what
happens when some scripts fail. Should the rest of the list be executed or
aborted. Under different circumstances different approaches would make sense
and we would need to implement all these interaction variants. Instead any
such complexity can be handled inside a singe script that the trigger
references.

We no longer try to generalize the monitoring the schema file changes and
auto-migrating them as part of regular ``gel.toml`` watch spec. There is a bit
of special handling and safe-guards involved in that monitoring that make it a
little too special. Instead we offer ``--migrate`` flag to enable
auto-migrations.

The hooks order should not be "wrapped". Imagine the following "wrapped" order:

* ``branch.switch.before``
* ``schema.update.before``
* ``gel branch switch foo``
* ``schema.update.after``
* ``branch.switch.after``

If the ``brach.switch`` hooks are managing some branch-dependent ``.env``
changes such as the currently used Postgres connection string for a SQL tool,
then ``schema.update.after`` hook executes in the new branch, but with the old
connection string. So it could result in an inconsistent state for processing
schema changes if that requires to coordinate SQL and Gel schemas. A similar
argument can work even for ``before`` hooks, although it would be a little
less intuitive. The idea is that if an effect is a result of a cascade (i.e.
schema is about to change *because* of a branch change), it still makes sense
for the hooks to run in the order in which they logically trigger each other.
This gives the opportunity for the previous steps in this cascade to setup the
correct context for the next triggered step. For example, the
``branch.switch.before`` hook can setup a flag that tells
``schema.update.before`` to skip all validation checks since it's about to be
replaced entirely with whatever is in the new branch.


Backwards Compatibility
=======================

The ``gel watch`` command will operate in a way that is the opposite to what
it ``edgedb watch`` used to do. The functionality of ``gel watch`` to
specifically monitor schema changes and attempt to auto-apply them in real
time involves some special handling on our end and would be enabled via an
opt-in flag ``--migrate`` (or ``-m``). Thus running the ``gel watch`` command
will no longer monitor and auto-apply schema changes. To do this going forward
``gel watch --migrate`` needs to be used.
