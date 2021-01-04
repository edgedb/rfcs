::

    Status: Draft
    Type: Guideline
    Created: 2020-07-02
    RFC PR: `edgedb/rfcs#0014 <https://github.com/edgedb/rfcs/pull/14>`_


===============================
RFC 1003: Consistent CLI Design
===============================

This RFC discusses the naming scheme for EdgeDB command line tools
and their shortcuts in REPL.  The RFC also documents which command line tools
can be exposed in REPL and how, as well as how various REPL options should be
specified.


Motivation
==========

Since the CLI commands are likely to land in third-party documentation
and automation, we want to ensure the design behind them is consistent
and natural to the user.


Guidelines for future CLI commands
==================================

* For commands operating on a single server, especially commands around
  a single database within a single server, prefer ``edgedb <action>``
  and ``edgedb <action>-<object>`` over ``edgedb <group> <command>``;

* For commands allowing maintenance of many servers, use
  ``edgedb server <action>`` and ``edgedb server <action>-<object>``;

* For interactive output, ``--help`` that exceeds the height of the
  current terminal should be shown in a pager like ``less``.

* When using the ``<action>-<object>`` scheme, if the action in question
  would be the literal word "show", it can be omitted, for example:
  ``edgedb migration-log``.

* Sub-command descriptions need to be as informative as possible.
  Repeating the name of the sub-command is not very useful without
  highlighting some details of the action.


Specific Concerns
=================

During discussions about the CLI, a number of specific issues were
raised. They are addressed below for reference.

Ballooning ``--help`` output
----------------------------

**Concern**: With more and more functionality exposed to the CLI, the
``<action>-<category>`` approach has the potential to balloon the output
of ``edgedb -h`` to more than 100 lines, making it harder to read.

**Decision**: a long but flat ``--help`` provides better discoverability
of new functionality compared to ``<group> <command>`` as more possible
actions are listed right away.  If the output exceeds the current height
of the terminal, an effective solution is to use a pager, as done by
``git`` and many other tools.  More initial help output allows for
quicker search of not only command names but also command descriptions.

Command-line design doesn't really follow EdgeQL
------------------------------------------------

**Concern**: The initial design goal of using the ``<action>-<category>``
scheme was to have the CLI naming follow EdgeQL commands, e.g.
``$ edgedb create-superuser-role`` and ``CREATE SUPERUSER ROLE`` look
similar. However this similarity is only superficial as command-line tools
have additional CLI-specific options, such as ``--password-from-stdin``.

**Decision**: This is true.  One-to-one mapping between command-line
functionality and EdgeQL should not be assumed by the user.  There isn't
much we can realistically do to bridge this gap because modes of use
are different between EdgeQL and a command-line tool:

* different escaping rules allowing for more flexibility within EdgeQL
  queries;

* a multi-line editor for EdgeQL queries is provided but user shells
  might not provide multi-line editing of terminal commands; and

* different information passing paradigms, with command-line tools using
  standard I/O pipelines whereas EdgeQL using sub-queries.

Vague verbs or nouns can be misleading
--------------------------------------

**Concern**: the ``edgedb show-status`` command does not indicate what kind
of status it will show. It could be showing status of the current migration,
or of the currently running server, or of how many active connections the
server has at the moment. Renaming it to ``edgedb show-migration-status``
would make it longer to type and overall very verbose. With
``<group> <command>`` scheme the solution is obvious:
``edgedb migration status``.

**Decision**: ``edgedb show-status`` should be renamed to
``edgedb migration-status`` which is consistent with
``edgedb migration-log``.

In general, vague nouns and verbs should be avoided unless
the feature in question is dead obvious (like ``edgedb dump``,
``edgedb query``, or ``edgedb server init``).

This will also future-proof the CLI for new kinds of functionality that
would use the same vague terms (e.g. ``edgedb time-machine-log``).

Category-specific help sections
-------------------------------

**Concern**: The ``<action>-<category>`` scheme does not allow
category-specific help sections. With ``<group> <command>`` scheme, top-level
help sections are natural: ``edgedb migration -h`` command can have a
description of the recommended migrations workflow, while
``edgedb server -h`` would explain to the user how to work with EdgeDB
servers.

**Decision**: the CLI is an unlikely place for a user to learn about new
complex functionality like the migration subsystem. Better suited avenues
for this are the online documentation and/or ``man`` pages, or better yet,
``tldr``.

To address discoverability of related sub-commands, top-level listing of
sub-commands should not be alphabetical but rather grouped by functionality.

``configure`` uses ``<group> <command>`` whereas ``list-`` is flat
------------------------------------------------------------------

**Concern**: ``configure`` is inconsistent with ``list-``.

**Decision**: It should be replaced by ``edgedb insert-config``,
``edgedb set-config``, and``edgedb reset-config``.

Why is ``server`` special?
--------------------------

**Concern**: the ``edgedb server`` CLI design outlined in RFC 1001
directly conflicts with the ``<action>-<category>`` scheme.

**Decision**: this sub-system is different because it is in fact
a tool within a tool. Whereas other commands deal with a single
database server, often a single database within that server, the
``edgedb server`` sub-system deals with management of potentially
many databases and server instances. With that in mind, keeping it
separate makes sense.

In particular, trying to shoehorn the sub-commands of ``edgedb server``
into the ``<action>-<object>`` scheme would essentially move all of
them into flat ``*-instance`` (or ``*-server``) sub-commands.

Are ``list-*`` commands useful?
-------------------------------

**Concern**: there are nine ``list-*`` sub-commands which is over 36%
of the available sub-commands. They mirror REPL functionality for listing
items in the current database, for example ``\la`` or ``\list-aliases``
will list available aliases. Are all of them necessary?

**Decision**: the CLI should keep listings of entities that it can
directly manipulate with other sub-commands. This includes:

* ``list-databases`` (through ``create-database``);

* ``list-ports`` (through ``configure``); and

* ``list-roles`` (through ``alter-role``).

Other listings should be removed as they are redundant with REPL
functionality. Instead, the following form should be enabled::

  $ echo "\list-object-types" | edgedb

This is consistent with behavior of, say, the ``sqlite3`` CLI.

The REPL help for ``\\`` commands is overloaded
-----------------------------------------------

**Concern**: It is convenient to expose the EdgeDB CLI commands directly in
REPL. This can greatly simplify administrative tasks when the DB
administrator has full DB admin rights but yet can't access the server shell.
Sadly, the presence of those commands under the ``\\`` prefix mixes them
up with the REPL-specific escape commands like ``\last-error``.

**Decision**: Expose CLI-specific commands in the REPL via the ``!`` prefix,
similar to how IPython exposes shell commands::

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

Connection options are passed in an unnatural spot
--------------------------------------------------

**Concern**: Current ``edgedb`` command usage is defined as::

  edgedb [FLAGS] [OPTIONS] [SUBCOMMAND]

where ``[OPTIONS]`` is defined as::

      --dsn <dsn>                   DSN for EdgeDB to connect to
  -d, --database <database>         Database name to connect to
  -H, --host <host>                 Host of the EdgeDB instance
  -P, --port <port>                 Port to connect to EdgeDB
  -I, --instance <instance>         Local instance name created with
                                    `edgedb server init` to connect to
                                    (overrides host and port)
  -u, --user <user>                 User name of the EdgeDB user

This sometimes creates unreadable commands through inconvenient syntax,
for instance::

  $ edgedb -d tutorial dump tutorial.edgedb

**Decision**: Passing *connection* options should be allowed anywhere
between ``edgedb`` and sub-commands, enabling the following::

  $ edgedb dump -d tutorial tutorial.edgedb
  $ edgedb dump tutorial.edgedb -d tutorial

This simplifies the overall UX, as for some commands it's logical to
receive the DB name as part of their ``[COMMAND-FLAGS]``.

Allowing connection options to come last also simplifies copy-pasting
them, which is especially useful for full DSNs.


Rejected Ideas
==============

Adopt the ``<group> <command>`` naming scheme for all of CLI
------------------------------------------------------------

An outline of ``edgedb`` subcommands would look like this:

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

This is rejected due to:

* inability to adjust every command naturally in this way;

* disruptive nature of the change;

* less verbose ``help`` output; and

* less natural-sounding commands.

Read "Appendix 2" for research conducted in this area.


Expose REPL-specific commands via the ``\`` prefix
--------------------------------------------------

Examples::

  \d [-v] NAME             describe schema object
  \l, \list-databases      list databases
  \lT [-sI] [PATTERN]      list scalar types
                           (alias: \list-scalar-types)
  <...>

This is rejected due to there being too many backslash commands
(currently 44), some of which duplicate CLI functionality.

(See Concern about ``\`` prefix being overloaded with CLI commands.)


Appendix 1: Current CLI state as of 1.0a7
=========================================

The current naming scheme of EdgeDB command line tools is inconsistent.  Both
``<group> <command>`` as well as ``<action>-<category>`` schemes are used,
and the `<group>` in the first kind can either be a noun or verb.

List of sub-commands::

  $ edgedb -h
  <...>
  SUBCOMMANDS:
    alter-role
    configure
    create-database
    create-migration
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
    migrate
    migration-log
    query
    restore
    self-upgrade
    server
    show-status

Server sub-commands::

  $ edgedb server -h
  <...>
  SUBCOMMANDS:
    destroy
    help
    info
    init
    install
    list-versions
    logs
    reset-password
    restart
    start
    status
    stop
    uninstall
    upgrade

Configuration sub-commands::

  $ edgedb configure -h
  <...>
  SUBCOMMANDS:
    help
    insert
    reset
    set


Appendix 2: CLI design of the 70 most downloaded Homebrew packages
==================================================================

To determine what the other popular tools in the industry are doing,
70 Homebrew packages were investigated in terms of their CLI UX.

The packages were chosen from a list of most downloaded Homebrew
packages.  Packages that were clearly libraries or otherwise lacked
a non-trivial CLI were skipped.

It turns out only 20 of those packages support sub-commands: git, yarn,
imagemagick, awscli, go, maven, heroku, rbenv, gradle, tmux, carthage,
docker, nvm, pyenv, ansible, sbt, terraform, hg, kubectl, hugo,
docker-compose. Almost all do it inconsistently.

Interesting findings:

* Go started with ``go [verb]`` and now with modules had to do
  ``go mod init``.

* Docker deprecated the ``<action>-<object>`` scheme but it looks like
  the community is unaware of it.  New documentation and third-party
  materials are still written using this syntax.  In effect, the tool
  supports both schemes.

* Git and Mercurial mix nouns and verbs as the command ("tag", "branch",
  vs. "pull", "push", "commit").

* Ansible is interesting: "name-of-group" is the sole argument followed
  by command-line options mimicking sub-commands.

* GPG also uses CLI options as sub-commands.

* kubectl is one example that consistently applies "kubectl <verb> <object>".
  To keep this consistency, unnatural sub-commands like "get" are present
  which require another subcommand like "pods". The verbs are called
  "operations" and the operands are called "resources" when they can be
  many things. For most operations, they can only be one thing (like in
  "drain", "convert", "apply", "run"). When they can't, help for the
  polymorphic ones is a challenge to follow.

* The AWS CLI is exactly backwards: it (almost) consistently applies
  ``<service> <verb>`` but that's because it's a multi-tool for dealing
  with logically separate services. As soon as you get to a particular
  service, ``<action>-<object>`` is used.

* A lot of tools use cute but vague commands like "init", "new", and "add".
  It's mostly pretty obvious from context what they are.

Conclusions
-----------

Successful command-line tools mostly use `<verb>` as the sub-command
scheme which is similar to what we are doing now for EdgeDB with the
exception of ``edgedb server`` and the implicit "show".

And when the tools do go for the ``<group> <command>`` scheme, they
don't do it consistently. This shows it's very tricky to accomplish in
a non-trivial application and likely isn't a deciding factor in the
overall user experience.
