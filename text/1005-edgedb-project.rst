::

    Status: Draft
    Type: Feature
    Created: 2021-04-02
    RFC PR: `edgedb/rfcs#30 <https://github.com/edgedb/rfcs/pull/30>`_

=============================================================
RFC 1005: CLI and conventions for local projects using EdgeDB
=============================================================

This RFC describes the design of the ``edgedb project`` CLI commands for
the purposes of initialization of a software project that uses EdgeDB as
well as the associated conventions and changes to the language bindings.


Motivation
==========

`RFC 1001 <1001-edgedb-server-control.rst>`_ introduced the concept of
EdgedDB *instances* and a straightforward way of managing them with
edgedb-cli.  The process of addressing an EdgeDB server has thus been
simplified to passing a ``-I`` argument to the CLI or passing it as the
argument to a ``connect()`` call in bindings.

However, even with ``edgedb server`` a developer needs to take multiple
steps to start a new project using EdgeDB:

1. Install the desired EdgeDB version (``edgedb server install``)
2. Initialize an EdgeDB instance for the project (``edgedb server init myapp``)
3. Create the ``dbschema/`` directory and the initial schema file.

When starting work on a cloned project, the steps are similar:

1. Install the desired EdgeDB version (``edgedb server install``)
2. Initialize an EdgeDB instance for the project (``edgedb server init myapp``)
3. Apply migrations (``edgedb -I myapp migrate``).

Since most projects using EdgeDB are expected to follow the conventions
established by the reasonable defaults of the EdgeDB CLI it makes sense to
automate the above operations to enable the following benefits:

1. Make it so that the project is ready to go after a single command
   (``edgedb project init``).
2. Allow an EdgeDB instance to be linked with a project directory to obviate
   the need to specify an instance name explicitly when running CLI commands
   or connecting with language bindings.
3. Allow specifying certain EdgeDB-specific metadata per project, such as
   the minimum required EdgeDB version.


Overview
========

In this RFC we propose a new group of ``edgedb`` CLI commands under
the ``edgedb project`` prefix:

* ``edgedb project init`` -- populate a new project or initialize an existing
  cloned object;

* ``edgedb project deinit`` -- remove association with and optionally destroy
  the linked EdgeDB instance;

* ``edgedb project status`` -- show EdgeDB-related information about a
  project, such as the associated instance name;

* ``edgedb project list`` -- list all known EdgeDB projects for the current
  user.


Design
======

Project file structure
----------------------

`RFC 1000 <1000-migrations.rst>`_ defined the top-level ``dbschema/`` directory
as the default location of EdgeDB SDL schema, and ``dbschema/migrations`` as
the default expected location of the files containing schema migration scripts.
To mark a directory as an EdgeDB project, an ``edgedb.toml`` file should be
present in the root directory.  The file can be empty or contain the following
TOML::

    [edgedb]
    server-version = <semver-range>

Here, the `server-version` attribute specifies a SemVer range of acceptable
server versions.  Most frequently this would be in the form of a minimum
required version.

Project detection
-----------------

To detect whether an executable or a script is executed in a project context,
it should look for ``edgedb.toml`` in the current working directory
and then the parent directories in sequence.

Nested projects
---------------

Nesting EdgeDB projects is not allowed to avoid confusion.

Project association
-------------------

In order to remove the need to specify an EdgeDB instance explicitly when
running the CLI commands or connecting via language bindings, we propose
to keep a mapping of absolute project paths to instance names in a well-known
location.  The mapping will be updated by the ``edgedb project init`` command
and will be interpreted by language bindings to obtain target instance address
and credentials where no explicit connection configuration is specified.

The project -> instance mapping is expressed as directories under
``~/.edgedb/projects``::

    ~/.edgedb/projects/
        <dir-basename>-<dir-abspath-hash>/
            project-path
            instance-name

Here ``<dir-basename>`` is the trailing component of the path to the project
directory, and ``<dir-abspath-hash>`` is defined as
``lower(strip(base32(abspath(project_dir)), '==='))```.  Hashing is necessary
to avoid bumping into the maximum directory entry name length.
The ``project-path`` file contains the full absolute path to the project
directory and is necessary for reverse lookup (find project by instance name),
and the ``instance-name`` file contains the name of the associated instance.

Language bindings and clients may cache the resolved instance name in memory
to avoid performing a lookup on every ``connect()`` call.  The cache should
be sensitive to the current working directory, although if the directory
changed to a subdirectory the cache should still be valid as nested projects
are not allowed.

Running ``project init`` from ``curl | sh``
-------------------------------------------

The CLI bootstrap script can detect if it's being ran from within a project
directory and ask if the user wants to run ``edgedb project init``.

Effect on ``edgedb server`` commands
------------------------------------

The ``edgedb server destroy`` command should refuse to destroy an instance that
is associated with a project by default, and should recommend to use
``edgedb project deinit`` (or ``edgedb server destroy --force``).

The ``edgedb server upgrade`` command should refuse to continue if the target
server version does not match the ``edgedb.server-version`` range in
``edgedb.toml``.


edgedb project init
===================

Initialize a new project or re-initialize an existing project.

Synopsis
--------

``edgedb project init [options]``

Options
-------

``--project-dir=<dir>``
  Specifies a project root directory explicitly.  If not specified, the project
  directory is detected as described in the "Project detection" section above,
  and if no project directory is detected, a current working directory is used.

``--server-version=<semver-range>``
  Specifies the desired EdgeDB server version as a SemVer range.  Only
  applicable for new projects.  Defaults to ``'*'``, which means latest stable
  version for new projects.  Accepts ``'nightly'`` as a special value denoting
  the latest nightly version.

``--server-instance=<instance>``
  Specifies the EdgeDB server instance to be associated with the project.
  If the specified instance does not exist, it will be created.  If the
  specified instance already exists, it must not be associated with another
  project.  ``edgedb server deinit`` may be used to disassociate an instance
  prior to linking it with another project.

``--server-instance-type=<instance-type>``
  Specifies the desired instance type.  Allowed values for ``<instance-type>``
  are: ``native`` and ``docker``, which correspond to the installation modes
  in ``edgedb server``.

``--non-interactive``
  Run in non-interactive mode.

Implementation
--------------

The ``edgedb project init`` command initializes a brand new project or
re-initializes an existing project.

In a new project:

- an ``edgedb.toml`` file is created in the project directory,
  and ``--server-version``, if specified, is recorded in the
  ``edgedb.server-version`` attribute.

- a ``dbschema`` directory and a ``dbschema/default.esdl`` file are created,
  the latter containing this declaration::

      module default {

      }

- if the specified server version is not installed, ``edgedb server install``
  performs the installation using the first available installation method
  in the order of preference (unless specified explicitly with
  ``--server-instance-type``).

- if the specified or implied server instance does not exist, an attempt to
  create it is made.

- a new record in ``~/.edgedb/projects`` is created for the new project.

In an existing project:

- the ``edgedb.toml`` file is read and validated;

- if the specified server version is not installed, ``edgedb server install``
  performs the installation using the first available installation method
  in the order of preference (unless specified explicitly with
  ``--server-instance-type``).

- if the specified or implied server instance does not exist, an attempt to
  create it is made.

- the record in ``~/.edgedb/projects`` is updated with the new instance name
  if necessary.

- if ``dbschema/migrations`` exists, ``edgedb migrate`` is executed to ensure
  that the configured instance is up-to-date.

Interactive mode
----------------

Here's a simulation of a proposed interactive mode for a new project::

    $ edgedb project init
    `edgedb.toml` was not found in `/home/user/work/myapp` or above.
    Do you want to initialize a new project? [Y/n] Y
    What type of EdgeDB instance would you like to use with this project?
    1. Local (native)
    2. Local (Docker)
    3. Cloud
    Your choice? 1
    Specify the version of EdgeDB to use with this project [latest stable]:
    Specify the name of EdgeDB instance to use with this project [myapp]:

    [shows summary of configuration]

    [asks whether to continue or restart configuration]

    Creating instance `myapp`...

Here's a simulation of a proposed interactive mode for a cloned project::

    $ edgedb project init
    Found `edgedb.toml` in `/home/user/work/myapp`.
    Found no associated EdgeDB instance.
    What type of EdgeDB instance would you like to use with this project?
    1. Local (native)
    2. Local (Docker)
    3. Cloud
    Your choice? 1
    Specify the name of EdgeDB instance to use with this project [myapp]:

    [shows summary of configuration]

    [asks whether to continue or restart configuration]

    Creating instance `myapp` ...
    Running migrations ...


edgedb project deinit
=====================

Remove association with and optionally destroy the linked EdgeDB intstance.

Synopsis
--------

``edgedb project deinit [options]``

Options
-------

``--project-dir=<dir>``
  Specifies a project root directory explicitly.  If not specified, the project
  directory is detected as described in the "Project detection" section above.

``--destroy-server-instance, -D``
  If specified, the associated EdgeDB instance is destroyed by running
  ``edgedb server destroy``.

``--non-interactive``
  Run in non-interactive mode.  Assume affirmative answer for all questions.

Implementation
--------------

The ``edgedb project deinit`` command removes the association with its EdgeDB
instance by removing the corresponding entry from the ``~/.edgedb/projects``
directory.  If ``--destroy-server-instance`` is specified, the associated
instance is destroyed.


edgedb project status
=====================

Shows the information about a project.

Synopsis
--------

``edgedb project status [options]``

Options
-------

``--project-dir=<dir>``
  Specifies a project root directory explicitly.  If not specified, the project
  directory is detected as described in the "Project detection" section above.

``--json``
  Use JSON as output format.


Implementation
--------------

The ``edgedb project status`` shows the following information about the
project:

- associated EdgeDB instance name, or `<none>` if not associated;
- migration status (which deprecates ``edgedb show-status``)


edgedb project list
===================

Lists all known EdgeDB projects for the current user.

Synopsis
--------

``edgedb project list [options]``

Options
-------

``--json``
  Use JSON as output format.


Implementation
--------------

The ``edgedb project list`` outputs a list of projects where each entry
contains a full path to the project directory and the name of an associated
EdgeDB instance.


Rejected Ideas
==============

Store all EdgeDB instance data alongside the project
----------------------------------------------------

We considered placing the credentials for the instance and optionally also
instance data in ``<project-dir>/.edgedb``.

The advantage of this approach is that it does not require explicit linking
of projects with EdgeDB instances and avoids possible instance name conflicts
by generating instance names.

This approach has the following downsides:

- ``<project-dir>/.edgedb`` MUST NOT be committed into VCS, most importantly
  due to possible exposure of secrets and other sensitive data.  Automatic
  modification of ``.gitignore`` (and other VCS ignorefiles) may mitigate this,
  but risk still exists if a user runs ``git add -f``.

- when properly ignored, ``<project-dir>/.edgedb`` is susceptible to accidental
  removal by ``git clean`` which may lead to data loss.

- many users run their project directories on network filesystems or use
  Dropbox for synchronization across their machines.  Placing an EdgeDB
  data directory on any sort of "magical" filesystem may lead to random
  corruption or significant performance issues.

Maintain a copy of instance credentials in the project directory
----------------------------------------------------------------

This is a variant of the approach described above, except data is stored in
the normal location and only the credentials file is copied to the project
directory.

This approach shares the VCS-related concerns of the approach above in that
the credentials file must not be committed, and that ``git clean`` would
disassociate the project with its instance leading to developer puzzlement
and inconvenience.

Use a globally-unique identifier for projects
---------------------------------------------

We considered giving each project a globally-unique identifier recorded in
``edgedb.toml``, and using it as an alias to the name of the associated
EdgeDB instance.  The idea was rejected as it complicates project forks,
because one has to remember to change the project id, and, most importantly,
once the project has been shared, the project id must not be changed to
avoid breaking project clones.
