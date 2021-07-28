::

    Status: Accepted
    Type: Feature
    Created: 2021-05-19
    Supersedes: RFC 1003: Consistent CLI Design


===============================
RFC 1006: Simplified CLI Design
===============================

This RFC proposes to change the EdgeDB CLI tools to adapt one single
naming scheme: ``edgedb <group> <action>``.

This RFC supersedes "RFC 1003: Consistent CLI Design".


Motivation
==========

The EdgeDB CLI (as of EdgeDB 1.0 Beta 2) sports two different naming schemes:

* ``edgedb <group> <action>`` used by ``edgedb server`` and ``edgedb project``;

* ``edgedb <action>`` used by all other commands, e.g.
  ``edgedb create-migration`` or ``edgedb create-role``.

The explanation for treating ``edgedb server`` and ``edgedb project``
differently from all other commands is that they are essentially "tools within
a tool". Technically this is a valid argument: both command groups do not work
with a single database or even take database connection arguments, contrary
to most other commands. But in reality, we cannot expect users to understand
the "tool within a tool" concept without reading RFCs or the documentation.
Most likely the CLI would be just perceived as inconsistent at best, or
inconvenient in the worst case.

With adding the ``edgedb project`` subcommand, it became clear that maintaining
two entirely different naming schemes in one tool will be confusing to the
users.  Here's what a portion of ``--help`` output would look like if we keep
the current approach::

    <...>

    Migrations:
        create-migration         Create a migration script
        apply-migrations         Bring current database to the latest or
                                 a specified revision
        show-migrations          Show all migration versions
        check-migrations         Show current migration state

    Server Management:
        server status            Status of an instance
        server init              Initialize a new server instance
        server start             Start an instance
        server stop              Stop an instance
        server restart           Restart an instance
        server info              Show server information

    <...>

The actual ``--help`` output will be considerably longer than that. It will
likely take a few days for the users to not be confused whether to type
``edgedb project init`` or ``edgedb init-project``, because they have just
run ``edgedb create-migration``.

Lastly, we will inevitably continue evolving the EdgeDB CLI by adding new
subcommands. For example, we might add the ``edgedb cloud`` command group which
would classify as a "tool within a tool". Or we might add subcommands to manage
authentication and other aspects of database management, which are unlikely
to be classified as "tools within a tool", in which case we will have more
commands in the form of ``edgedb create-X`` or ``edgedb show-Y``. Ultimately,
the number of commands with entirely different naming will only continue to
grow.


Overview
========

The RFC proposes to have the following commands structure for the
the EdgeDB RC1 release::

  SUBCOMMANDS:

    dump                       Create a database backup
    restore                    Restore a database backup from file
    configure                  Configure a DB or a server instance

    migration apply            Migrate the database to the latest revision
    migration create           Create a new migration
    migration log              Show the migrations log
    migration status           Show the current migration state
    migrate                    An alias for `edgedb migration apply`

    project init               Initialize a new EdgeDB project
    project status             Show the status of the current project
    project list               List all projects
    project unlink             Clean-up the project configuration

    instance create            Initialize a new server instance
    instance status            Show statuses of all or of a matching instance
    instance start             Start an instance
    instance stop              Stop an instance
    instance restart           Restart an instance
    instance destroy           Destroy an instance and remove its data
    instance logs              Show logs of an instance
    instance upgrade           Upgrade an instance to a newer server version
    instance reset-password    Reset instance admin password
    instance link              Link a remote instance
    instance unlink            Unlink a remote instance

    database create            Create a new DB
    database drop              Drop the DB

    describe object            Describe a database object
    describe schema            Describe the database schema

    list                       List matching database objects by name and type
                               (run `edgedb list --help` for more info)

    server                     Manage local installations of EdgeDB server

    cli upgrade                Upgrade the 'edgedb' command to the latest version
    cli uninstall              Uninstall the 'edgedb' command

The output of ``edgedb list --help``::

  edgedb list [SUBCOMMAND]

  SUBCOMMANDS:

    list aliases               List type aliases
    list casts                 List casts
    list databases             List databases
    list indexes               List indexes
    list modules               List modules
    list roles                 List roles
    list types                 List object types
    list scalars               List scalar types

The output of ``edgedb server --help``::

    server info                Show locally installed EdgeDB servers
    server install             Install an EdgeDB server locally
    server uninstall           Uninstall an EdgeDB server locally
    server list-versions       List available and installed versions of the server


Interactivity
=============

We assume that most of the time the CLI will be used by humans than in scripts.
Therefore, commands might prompt the user about any missing information
interactively.

``edgedb`` command should accept a ``--non-interactive`` flag to disable
interactivity for any subcommand, e.g.::

    $ edgedb --non-interactive
    Error: cannot run EdgeDB REPL in non-interactive mode.

    $ edgedb --non-interactive instance link foo
    Error: instance name "foo" is already taken.
    Pass '--override' to override it.


Design Considerations
=====================

Why there is no ``edgedb role``
-------------------------------

We will likely introduce role management commands when we begin working on
streamlining auth management and implementing the access control layer.


Why there is no ``edgedb query``
--------------------------------

We already have ``edgedb -c``.  We can add ``edgedb query`` if it is requested
by users.


Occasional Duplication of Commands
----------------------------------

Some of the commands will have aliases:

* ``edgedb migrate`` is an alias for ``edgedb migration apply``. The reason
  for having the alias: this will be a very popular and frequently typed
  command.

* ``edgedb list X`` might become an alias for ``edgedb X list``
  for some types of entities in the future.  While this does not seem like
  a "pure" solution, there is no harm in having aliases like this.

In general, we believe that having aliases for some commands cannot
harm the overall developer experience of using the CLI.


RFC 1001: edgedb server commands restructuring
----------------------------------------------

`Beta 2 feedback <https://github.com/edgedb/edgedb/issues/2647>`_ revealed
some user confusion related to the CLI and EdgeQL terminology:

1. EdgeQL had ``CONFIGURE SYSTEM`` to configure instances.
2. The CLI had ``edgedb server install`` to essentially provision
   installed versions of EdgeDB Server software locally.
3. The CLI had ``edgedb server start`` to start an instance of
   a locally installed server.

(1) has been `fixed <https://github.com/edgedb/edgedb/pull/2712>`_ by
deprecating ``CONFIGURE SYSTEM`` and introducing ``CONFIGURE INSTANCE``
command.

The confusion between (2) and (3) can be solved by splitting ``edgedb server``
command group into two:

* ``edgedb server`` to control what versions of EdgeDB Server are installed
  on the local machine, and

* ``edgedb instance`` to control locally (and remotely, in the future)
  running instances of specific EdgeDB Server versions.

All of the ``instance`` subcommands should accept an instance name or a name
filter as their first argument.


RFC 1003 -- Rejected Ideas
--------------------------

The superseded RFC 1003 explicitly rejected the ``<group> <command>`` naming
scheme, quote::

    * inability to adjust every command naturally in this way;

    * disruptive nature of the change;

    * less verbose ``help`` output; and

    * less natural-sounding commands.

The verbosity of the updated ``edgedb --help`` output can be and will be
tweaked until it hits the perfect balance of being readable and informative.

The less natural-sounding commands argument is valid, as
``edgedb create-migration`` certainly sounds more natural than
``edgedb migration create``. But given that we will likely have between more
than 30 subcommands, it is clear that giving users a way to organize
subcommands mentally in categories to memorize the overall structure is more
important than "making commands sound like plain English".

The proposed change is indeed very disruptive but we believe it is still worth
implementing it before 1.0. It is important to understand that RFC 1003 was
written when the CLI had only one "tool within a tool" — ``edgedb server``.
Since then we have added ``edgedb project`` and it became apparent that we
will likely continue to add more tools like that.


REPL Introspection
==================

The ``describe`` and ``list`` CLI subcommands will be exposed in REPL
via the backslash syntax. The below table outlines the new mapping:

================================= =============================================
          CLI command                              REPL Command
================================= =============================================
``edgedb describe object``        ``\d``
``edgedb describe schema``        ``\ds``
``edgedb list databases``         ``\l``
``edgedb list aliases``           ``\la``
``edgedb list casts``             ``\lc``
``edgedb list indexes``           ``\li``
``edgedb list modules``           ``\lm``
``edgedb list types``             ``\lt``
``edgedb list scalars``           ``\ls``
``edgedb list roles``             ``\lr``
================================= =============================================

The ``\l`` and ``\d`` REPL shortcuts are used especially frequently so we
are shortening them to one letter (instead of calling them ``\do`` and
``\ld``.)


Changes Summary
===============

Changes in the CLI:

================================= ===============================================
        Old CLI command                                Comments
================================= ===============================================
``edgedb configure``              Keep as is
``edgedb alter-role``             Remove
``edgedb create-superuser-role``  Remove
``edgedb create-database``        Rename to ``edgedb database create``
``edgedb create-migration``       Rename to ``edgedb migration create``
``edgedb describe``               Rename to ``edgedb describe object``
``edgedb drop-role``              Remove
``edgedb dump``                   Keep as is
``edgedb help``                   Remove (we can later implement long help)
``edgedb list-aliases``           Rename to ``edgedb list aliases``
``edgedb list-casts``             Rename to ``edgedb list casts``
``edgedb list-databases``         Rename to ``edgedb list databases``
``edgedb list-indexes``           Rename to ``edgedb list indexes``
``edgedb list-modules``           Rename to ``edgedb list modules``
``edgedb list-object-types``      Rename to ``edgedb list types``
``edgedb list-scalar-types``      Rename to ``edgedb list scalars``
``edgedb list-roles``             Rename to ``edgedb list roles``
``edgedb migrate``                Keep as is; also add ``edgedb migration apply``
``edgedb migration-log``          Rename to ``edgedb migration log``
``edgedb project``                Keep as is
``edgedb query``                  Remove
``edgedb restore``                Keep as is
``edgedb self-upgrade``           Rename to ``edgedb cli upgrade``
``edgedb server info``            Keep as is
``edgedb server install``         Keep as is
``edgedb server uninstall``       Keep as is
``edgedb server list-versions``   Keep as is
``edgedb server upgrade``         Rename to ``edgedb instance upgrade``
``edgedb server init``            Rename to ``edgedb instance create``
``edgedb server status``          Rename to ``edgedb instance status``
``edgedb server start``           Rename to ``edgedb instance start``
``edgedb server stop``            Rename to ``edgedb instance stop``
``edgedb server restart``         Rename to ``edgedb instance restart``
``edgedb server destroy``         Rename to ``edgedb instance destroy``
``edgedb server logs``            Rename to ``edgedb instance logs``
``edgedb server reset-password``  Rename to ``edgedb instance reset-password``
``edgedb show-status``            Rename to ``edgedb migration status``
================================= ===============================================

Changes in REPL:

================================= ===============================================
        Old REPL command                             Comments
================================= ===============================================
``\d``                            Keep as is
``\l``                            Keep as is
``\la``                           Keep as is
``\lc``                           Keep as is
``\li``                           Keep as is
``\lm``                           Keep as is
``\lt``                           Keep as is
``\lT``                           Rename to ``\ls``
``\lr``                           Keep as is
``\list-databases``               Rename to ``\list databases``
``\list-scalar-types``            Rename to ``\list scalars``
``\list-object-types``            Rename to ``\list types``
``\list-aliases``                 Rename to ``\list aliases``
``\list-indexes``                 Rename to ``\list indexes``
``\list-modules``                 Rename to ``\list modules``
``\list-roles``                   Rename to ``\list roles``
``\list-casts``                   Rename to ``\list casts``
================================= ===============================================

Changes to other command line flags:

================================= ===============================================
        Old Flag                                     Comments
================================= ===============================================
``--no-version-check``            Rename to ``--no-cli-update-check``
================================= ===============================================

Backwards Compatibility
=======================

We will supporting all existing commands (e.g. ``edgedb create-migration``)
until the 1.0 release.

The old commands will be hidden from the ``--help`` output. When run, old
commands will render a deprecation warning, e.g.::

    $ edgedb create-migration
    The `create-migration` command has been deprecated.
    Use `edgedb migration create` instead.

    <... edgedb migration create output ...>

Same transition strategy applies to REPL::

    db> \lT
    The `\lT` shortcut has been deprecated, use `\ls` instead.

    <... list of scalar types ...>
