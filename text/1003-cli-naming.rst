::

    Status: Draft
    Type: Feature
    Created: 2020-07-02
    RFC PR: `edgedb/rfcs#0014 <https://github.com/edgedb/rfcs/pull/14>`_


===============================
RFC 1003: Consistent CLI Design
===============================

This RFC defines the naming scheme for EdgeDB command line tools
and their shortcuts in REPL.  The RFC also documents which command line tools
can be exposed in REPL and how, as well as how various REPL options should be
specified.


Motivation
==========

The current naming scheme of EdgeDB command line tools is inconsistent.  Both
``<group> <command>`` and ``<action>-<category>`` schemes are used, e.g.::

  $ edgedb -h
  <...>
  SUBCOMMANDS:
    alter-role
    configure
    create-database
    create-superuser-role
    describe
    drop-role
    dump
    help
    list-aliases
    list-casts
    list-databases
    list-indexes
    list-modules
    list-object-types
    list-ports
    list-roles
    list-scalar-types
    query
    restore
    server

and::

  $ edgedb server -h
  <...>
  SUBCOMMANDS:
    help       Prints this message or the help of the given subcommand(s)
    init       Initialize a new server instance
    install    Install edgedb-server
    restart    Restart an instance
    start      Start an instance
    status     Status of an instance
    stop       Stop an instance

Furthermore, RFC 1000 adds more top-level subcommands, such as:

* ``edgedb migrate``,
* ``edgedb create-migration``,
* ``edgedb show-status``.

The initial design goal of using the ``<action>-<category>`` scheme was to
have the CLI naming follow EdgeQL commands, e.g. ``$ edgedb create-role`` and
``CREATE ROLE`` are looking similar.  However this similarity is only
superficial, as command line tools have additional CLI-specific options,
such as ``--password-from-stdin``.

Summarizing the above:

1. With more and more functionality exposed to the CLI, the
   ``<action>-<category>`` approach has the potential to balloon the output
   of ``edgedb -h`` to more than 100 lines, making it effectively unreadable.

   The ``<group> <command>`` scheme would only display the few top-level
   categories making it easier for users to build a mental model of what's
   possible with the CLI.

2. The ``edgedb show-status`` command does not indicate what kind of status
   it will show. It could be showing status of the current migration, or of
   the currently running server, or of how many active connections the server
   has at the moment.  Renaming it to ``edgedb show-migration-status`` would
   make it longer to type and overall very verbose.

   With ``<group> <command>`` scheme the solution is obvious:
   ``edgedb migration status``.

3. The ``edgedb show-status`` command prompts another question: what
   if another kind of "display status" command is needed?

   With ``<group> <command>`` scheme, there would be no conflict between
   ``edgedb server status`` and ``edgedb migration status``.

4. The ``<action>-<category>`` scheme does not allow category-specific
   help sections.

   With ``<group> <command>`` scheme, top-level help sections are natural:
   ``edgedb migration -h`` command can have a description of the recommended
   migrations workflow, while ``edgedb server -h`` would explain to the user
   how to work with EdgeDB servers.

5. ``edgedb configure`` currently implements the ``<group> <command>`` scheme;
   it should have been ``edgedb insert-config``, ``edgedb set-config``,
   ``edgedb reset-config``.

6. The ``edgedb server`` CLI design outlined in RFC 1001 directly conflicts
   with the ``<action>-<category>`` scheme.

The last observation is what prompted the creation of this RFC.  We either
need to update RFC 1001 to use the ``<action>-<category>`` scheme or
adopt the ``<group> <command>`` for all other commands.  Combining two
drastically different CLI designs in one tool is not an option.


Overview
========

The RFC proposes to adopt the ``<group> <command>`` naming scheme for
the ``edgedb`` CLI.  Here's an outline of current and future ``edgedb``
subcommands:

* ``server`` (see also RFC 1001.)
  - ``init``
  - ``install``
  - ``restart``
  - ``start``
  - ``status``
  - ``stop``

* ``dump``
  - ``db`` -- backup a database.
  - ``all`` -- backup all databases, as well as roles, configs, etc into
    a directory.
  - ``restore-db``
  - ``restore-all``
  - ``config`` -- backup system configuration.

* ``migration``
  - ``status``
  - ``create``
  - ``apply``

* ``role``
  - ``create [--superuser]``
  - ``alter``
  - ``drop``
  - ``list``

* ``db``
  - ``create``
  - ``rename``
  - ``drop``
  - ``list``

* ``config [--system]`` (used to be ``edgedb configure``)
  - ``set``
  - ``reset``
  - ``add`` (used to be ``insert``)
  - ``show``

* ``run [--stdin | -c]`` -- run an EdgeQL script or command from stdin
  or passed via the command line with ``-c``.


Design Considerations
=====================

List Commands
-------------

Currently, EdgeDB REPL has shortcuts to list aliases, roles, etc. for the
current database, with ``\la`` or ``\list-aliases`` kind of syntax.  The
list commands are also exposed to the CLI.

While these commands are quite handy to have inside REPL, their usefulness
as standalone CLI tools is questionable, especially when an arbitrary EdgeQL
introspection query (or even shortcuts like ``\la``) can be easily piped into
the ``edgedb`` command.

This RFC proposes to limit the number of actual CLI commands to the practical
minimum.


CLI via REPL
------------

REPL-specific commands should be exposed via the ``\`` prefix, e.g.::

  \d [-v] NAME             describe schema object
  \l, \list-databases      list databases
  \lT [-sI] [PATTERN]      list scalar types
                           (alias: \list-scalar-types)
  <...>

It is convenient to expose the EdgeDB CLI commands directly in REPL.  This
can greatly simplify administrative tasks when the DB administrator has full
DB admin rights but yet can't access the server shell.  For that purpose, all
CLI commands are exposed with the ``!`` prefix, similar to how IPython
exposes shell commands::

  >>> \list-databases
  tutorial

  >>> SELECT 1;
  {1}

  >>> !dump db tutorial
  done

This way the internal REPL help system is not overloaded with rarely
needed help on CLI commands and would only show the list of convenient ``\``
commands with a hint that ``!help`` or ``!h`` can be used to list
all CLI options.

Name-spacing ``\help`` and ``!help`` is good for the usability, because
the latter set of commands is not going to be used as frequently as
the former.


Connection Options
------------------

Current ``edgedb`` command usage is defined as::

    edgedb [FLAGS] [OPTIONS] [SUBCOMMAND]

where ``[OPTIONS]`` is defined as::

    -d, --database <database>         Database name to connect to
    -H, --host <host>                 Host of the EdgeDB instance
    -P, --port <port>                 Port to connect to EdgeDB

We propose to allow passing *connection* options anywhere between
``edgedb``, ``<group>``, and ``<command>``::

	$ edgedb [OPTIONS] <group> <action> [COMMAND-FLAGS]
	$ edgedb <group> [OPTIONS] <action> [COMMAND-FLAGS]
	$ edgedb [OPTIONS] <group> <action> [OPTIONS] [COMMAND-FLAGS]

This simplifies the overall UX, as for some commands it's logical to
receive the DB name as part of their ``[COMMAND-FLAGS]``, e.g.::

	$ edgedb -d tutorial backup dump

vs.::

	$ edgedb backup dump -d tutorial

The latter specified flags shadow the former ones, e.g.::

	$ edgedb -d foo backup dump -d tutorial

would backup the ``tutorial`` database.
