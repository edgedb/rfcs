::

    Status: Accepted
    Type: Feature
    Created: 2020-04-29
    RFC PR: `edgedb/rfcs#0007 <https://github.com/edgedb/rfcs/pull/7>`_

=================================================================
RFC 1001: CLI for installation and control of local EdgeDB server
=================================================================

This RFC describes the design of the ``edgedb`` CLI subcommand for the
purposes of installation, update and control of a local EdgeDB server.


Motivation
==========

Currently, the tasks of installing, updating and running an EdgeDB server
instances are entirely manual and vary a lot across the supported platforms.
From the standpoint of local development this creates unnecessary friction
and opens lots of possibilities for user error.  The current state also
necessitates a non-trivial amount of documentation that new users must read.

By implementing the installation, update, and control logic into the ``edgedb``
CLI we can significantly simplify the process of getting started with EdgeDB
without leaving the familiar development process:

1. Download the ``edgedb`` binary via the most convenient channel
   (`npm`, `pip`, `curl`).
2. Use the downloaded ``edgedb`` binary to either install and initialize
   a local server, or configure a remote server instance for development.


Overview
========

The RFC proposes a new group of ``edgedb`` CLI commands under ``edgedb server``
prefix:

* ``edgedb server list-versions`` -- list EdgeDB server versions available
  for installation;

* ``edgedb server install`` -- install or update a specific version of the
  EdgeDB server on the local machine;

* ``edgedb server uninstall`` -- uninstall a specific version of the
  EdgeDB server or all versions of EdgeDB from the local machine;

* ``edgedb server init`` -- initialize a new EdgeDB server instance;

* ``edgedb server start`` -- starts an EdgeDB server instance;

* ``edgedb server status`` -- show the status of the local EdgeDB server;

* ``edgedb server logs`` -- show the logs of the specified EdgeDB server
  instance;

* ``edgedb server stop`` -- stop the given EdgeDB server instance;

* ``edgedb server restart`` -- restart the given EdgeDB server instance;

* ``edgedb server upgrade`` -- upgrade the specified EdgeDB server instance
  to the new major version.

* ``edgedb server prune`` -- removes upgrade backups and other unused data.


Design Considerations
=====================

Instance names
--------------

Most commands described below refer to EdgeDB server instances by name.
The simplest interpretation is that the instance name is just the name
of a data directory folder in a well-known location.  For system instances
(created with ``edgedb server start --system``) this would be
directories under ``/var/lib/edgedb/data/``.  For user instances, this
would be directories under ``$XDG_DATA_HOME/edgedb/data/``.  In both
situations the base directory location for data should be configurable.

The set of instance names is unique, and in situations where a system
instance is created with the same name as an existing user instance,
the user instance "masks" the system instance.  ``edgedb server status``
should tell if an instance is system-wide or user-local.

Interactive mode
----------------

Most commands described below offer an interactive wizard mode that can
be selected by passing the ``-i`` or ``--interactive`` option to the command.
Whenever a command, running in non-interactive mode, encounters a
lack-of-input situation it should hint at the availability of the interactive
mode.

No ``--docker`` by default
--------------------------

Using Docker by default introduces a non-trivial dependency on software that
might not be available to the user, so this is an opt-in feature.


edgedb server list-versions
===========================

List EdgeDB server versions available for installation.

Synopsis
--------

``edgedb server list-versions [options]``

Options
-------

``--installed-only``
  only list the installed versions of the EdgeDB server.



edgedb server install
=====================

Downloads and installs a given EdgeDB server version
(latest stable by default).

Arguments
---------

``--version=<ver>``
  specifies the major version of the server to install.

``--nightly``
  if passed, the latest nightly build from the specified version channel
  is installed.

``--update``
  if specified, ``edgedb install`` will only attempt to update the existing
  installations.

``--method={package|docker}``
  Use specified installation method. ``package`` installs a native package on
  supported operating systems.  While ``docker`` uses Docker instead of
  downloading and installing packages directly onto the user's system.  The
  Docker daemon must be present and accessible by the user.


Implementation
--------------

By default, ``edgedb install`` will use system's package manager. If platform
is unsupported error message will show other options, like installing it
in Docker container if the latter is available on the system.


edgedb server uninstall
=======================

Uninstalls the specified version of EdgeDB.

Synopsis
--------

``edgedb server uninstall [options]``

If there are multiple versions installed, either ``--all`` or
``--version`` or ``--unused`` is required.

Options
-------

``--version=<ver>``
  Specifies the version to uninstall.  The specified server version must
  not be currently running.

``--all``
  Uninstalls all versions of EdgeDB.

``--unused``
  Uninstalls all versions of EdgeDB that are not used in any instance.


edgedb server init
==================

Initialize a new EdgeDB server instance with the specified name.

Synopsis
--------

``edgedb server init [options] <name>``

Options
-------

``<name>``
  The name of the EdgeDB instance.  Must be unique.

``--version=<ver>``
  Optionally specifies the server version to use.  If not specified,
  the latest installed server version is used.

``--start-conf=auto|manual``
  If set to ``auto`` (the default), the server will be started automatically
  on system boot.

``--port=<port-number>``
  Optionally specifies the port number on which the server should listen.

``--system``
  By default, ``edgedb server start`` runs the server in the user scope,
  if ``--system`` is specified, it is started as a system-wide service
  instead.

``--server-options -- <options>``
  Specifies the ``edgedb-server`` command line options verbatim.
  Must be the last argument.



edgedb server start
===================

Starts an EdgeDB server instance with the specified name.

Synopsis
--------

``edgedb server start [options] <name>``

Options
-------

``<name>``
  The name of the EdgeDB instance.  Must be unique.

``--server-options -- <options>``
  Passes ``edgedb-server`` options verbatim.  Must be the last argument.

``--foreground``
  Run server in the foreground instead of running as a system service.


edgedb server status
====================

Shows the status of the specified server instance or all instances.

Synopsis
--------

``edgedb server status [options] [<name>]``

Options
-------

``<name>``
  The name of the EdgeDB instance.  If not specified status of the all
  instances is printed.


Implementation
--------------

The command outputs the state of the server instance
(``running`` or ``stopped``), the port number it is configured to run on,
the scope of the instance (system-wide or user-local), and the runtime under
which the server is running (docker or native).


edgedb server logs
==================

Show the logs of the specified EdgeDB server instance.

Synopsis
--------

``edgedb server logs [options] <name>``

Options
-------

``<name>``
  The name of the EdgeDB instance.

``--tail <number>``
  Show the last ``number`` of log entries.

``--follow``
  Show the recent log entries and then continuously output new log entries
  as they are added to the log.


edgedb server stop
==================

Stops the specified EdgeDB server instance.

Synopsis
--------

``edgedb server stop [options] <name>``

Options
-------

``<name>``
  The name of the EdgeDB instance.

``--mode=<fast|graceful>``
  The server restart mode. The ``fast`` mode (the default) does not wait
  for the clients to disconnect and forcibly terminates connections, all
  in-progress transactions are rolled back. The ``graceful`` mode waits
  for the clients to disconnect gracefully.


edgedb server upgrade
=====================

Upgrades the specified EdgeDB server instance to a given EdgeDB version.

Synopsis
--------

There are few modes of operation of this command:

``edgedb server upgrade``
  Without arguments this command upgrades all instances which aren't running
  nightly EdgeDB to a latest minor version of the server.

``edgedb server upgrade <name> [--to-version=<ver>|--to-nightly]``
  Upgrades specified instance to the specified major version of the server or
  to the latest nightly, by default upgrades to the latest stable. This only
  works for instances that initially aren't running nightly.

``edgedb server upgrade --nightly``
  Upgrades all existing nightly instances to the latest EdgeDB nightly.

Options
-------

``<name>``
  The name of the EdgeDB instance. If omitted all stable instances will
  be upgraded to the latest minor version. (Or all nightly instances
  will be upgraded with ``--nightly``)

``--to-version``
  Specifies the version of EdgeDB to upgrade to.  If not specified,
  the latest available installed version is used.

``--to-nightly``
  Specifies that the instance should be upgraded to the nightly version.

``--nightly``
  Upgrade all instances currently running nightly to the latest nightly
  version (includes upgrades across major versions).

``--allow-downgrade``
  Allow downgrading to an older version.  Downgrades are prohibited by
  default.

``--revert``
  Revert the upgrade if the original data directory has not been removed.

Implementation
--------------

For minor upgrade this command:

* stops all running instances
* upgrades the package
* starts all instances

For any other upgrade:

* dumps everything to a directory ``<instance-name>.dump``
  using ``edgedb dump --all``
* upgrades needed packages
* renames old data directory to ``<instance-name>.backup``
* inits new server and restores data via ``edgedb restore --all```

This keeps the original data directory in case ``--revert`` is requested.


edgedb server restart
=====================

Restart the specified EdgeDB server instance.

Synopsis
--------

``edgedb server restart [options] <name>``

Options
-------

``<name>``
  The name of the EdgeDB instance.

``--mode=<fast|graceful>``
  The server restart mode. The ``fast`` mode (the default) does not wait
  for the clients to disconnect and forcibly terminates connections, all
  in-progress transactions are rolled back. The ``graceful`` mode waits
  for the clients to disconnect gracefully.


edgedb server prune
===================

Removes upgrade backups.

Synopsis
--------

``edgedb server prune [options]``

Options
-------

``--upgrade-backups``
  Prune upgrade backups.  After this ``edgedb server upgrade --revert``
  will be impossible.
