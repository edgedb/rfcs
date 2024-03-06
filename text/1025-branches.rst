::

    Status: Draft
    Type: Feature
    Created: 2024-01-11
    Authors: Victor Petrovykh <victor@edgedb.com>


===========================
RFC 1025: Database Branches
===========================

Abstract
========

We propose to reframe the current concept of ``database`` in EdgeDB as
``branch``. This avoids some of the confusion in current terminology and
provides a clearer guideline for using EdgeDB in development environment.
Functionally a ``branch`` will be the same as the current ``database``, but
with some additional options and tooling.


Motivation
==========

The current nomenclature of ``database`` and the corresponding commands are
confusing to many users. A running EdgeDB server represents an *instance*
while each instance can have multiple *databases* in it. Since the databases
all run by the same server they all have to use the same underlying EdgeDB
version. These qualities make them impractical for usage across multiple
projects as the projects would have to share resources and perform EdgeDB
upgrades together. We specifically recommend to have a dedicated instance for
each project. This leaves databases as a feature without a clear usage.

Instead we want to enhance the developer experience while using multiple
source control branches. These multiple databases could map well onto
corresponding source branches, representing slightly different version of the
data and schema. Thus there is a good reason to rename the concept of
*database* to *branch* and provide additional branch-handling tools. With this
change it is also now no longer necessary to have a semantic distinction
between the terms *instance* and *database* which may be natural for some
users to think of as interchangeable.

A significant distinction between database branches and VCS branches is that
the database branches are not as "lightweight" so we cannot automatically
create a new branch for every VCS branch corresponding to that branch's schema
and migrations. The mapping between VCS branches and database branches should
be more deliberate and explicitly setup by the user.


Branch DDL Commands
===================

There are several new DDL commands introduced by this proposal:

1) ``create empty branch <newbranch>``

   The most basic command creates a new branch with an empty schema
   (exactly what ``create database`` command currently does).

2) ``create schema branch <newbranch> from <oldbranch>``

   This command creates a new branch and copies the schema of an existing
   branch to it. Only the schema is copied, the data is still empty and needs
   to be populated separately.

3) ``create data branch <newbranch> from <oldbranch>``

   This command creates a new branch and copies both the schema and the data
   of an existing branch to it.

4) ``drop branch <oldbranch>``

   Removes an existing branch from the instance.

5) ``alter branch <oldname> rename to <newname>``

   The command to rename a branch.

The ``create schema/data branch`` command parallels source control branches
that diverge in terms of the schema. It is recommended to setup a new database
branch whenever working on a code branch that alters the schema in some way.
This will help reduce friction when switching between code branches by having
database states that correspond to the schema and avoiding accidental
mismatches in migrations.


Project and CLI Integration
---------------------------

Currently the project configuration file ``edgedb.toml`` does not have to
specify the default *database* it connects to. We  can add a
``edgedb.auto.toml`` config file that can be used to supplement and override
the main ``edgedb.toml`` config. This new file is not intended to be committed
to any VCS repository and kept strictly local. This way the alternative
database branch will not accidentally be committed to the wrong VCS branch. All
the EdgeDB clients will have to be upgraded so that they can look up both
``edgedb.toml`` and ``edgedb.auto.toml`` files.

A new project should ask the user for the branch name defaulting to "main". We
should also suggest adding ``edgedb.auto.toml`` to ``.gitignore``.

The ``database`` CLI commands should be deprecated and new ``branch`` commands
introduced instead.

``edgedb branch create`` command options:

* It must have the ``--from <oldname>`` option. If this option is omitted the
  current branch specified in the ``edgedb.auto.toml`` file is used as the
  "from" branch. After the branch is created, the ``edgedb.auto.toml`` file
  is updated to use the new branch as the connection default.

* It must have the ``--empty`` option. If specified, this is equivalent to
  running ``create empty branch <newname>`` DDL command.

* It should have the ``--copy-data`` option. By default a new branch copies
  schema only.

The old ``drop`` subcommand works pretty much same as before.

The old ``wipe`` subcommand may still be relevant for resetting a particular
branch.

There must be a new ``switch`` sub-command that allows changing the default
branch to connect to by updating the ``edgedb.auto.toml`` file. We can also
offer a post-checkout git hook to update the branch in ``edgedb.auto.toml``
when switching git branches. The mapping between git and EdgeDB branches can
be maintained in the ``edgedb.auto.toml`` file as well. When switching EdgeDB
branches this way we should print a message with the new branch name. By
default, if a git branch is not explicitly mapped to any branch in
``edgedb.auto.toml`` we should use the ``main`` branch. The user can then
call ``edgedb branch switch`` to change the git/EdgeDB branch association.

There must be a new ``edgedb branch rename <oldname> <newname>`` command in
order to be able to rename branches.


Rebasing Branches
-----------------

We need to be able to rebase and merge database branches. This overlaps a lot
with the scope of the ``migration`` commands.

When rebasing one branch on top of another we can use introspection to compare
the respective migration histories and find the point where they diverge.
Afterwards we can try to apply a batch of new migration to the existing branch
and if there are no issues perform a "fast-forward" rebase.

In order to minimize the hassle of rebasing the git branches corresponding to
database branches we need to give migration files names that are distinct in
these branches so that git does not attempt to merge the file contents. We can
reuse the (shortened) migration hash for this purpose. The goal here is to
differentiate migration files so that when parallel VCS branches get merged
the migrations have a high chance to stay in their separate files rather than
being merged into a single file that causes conflicts. We still want to retain
the numeric indexes to make it easier for a human to view the migration
history. In order to update the migrations from the old naming format to this
new one we want to add ``edgedb migration format --upgrade`` command (assuming
that we will have other formatting options later on). If our CLI tools detect
that the migration files are using the old format they should suggest running
``edgedb migration format --upgrade`` in order to proceed with any other
migration or branch commands.

At first rebasing one branch on top of another is the main workflow that we
offer for managing branches and merging them back together. Eventually we may
be able to expand the options to include merges, such as diamond or octopus
merges where the order in which migrations were applied is not as strictly
defined. This can only work with the subset of migration for which we can
prove that the order does not affect semantics.

Here's an example of how the rebase workflow is expected to work using "main"
and "feature" branches:

1) Create a new "feature" VCS branch (a clone of the "main" branch) and a
   corresponding "feature" EdgeDB branch.

2) Work on the "feature" branch, add migrations, etc.

3) When it is time to merge the feature work back into the main branch we want
   to arrange things so that the "feature" branch is in a state that is a
   simple fast-forward w.r.t the "main" branch.

4) In order to achieve the above state we need to make sure "main" code branch
   as well as EdgeDB branch are both up-to-date.

5) Then we want to rebase the "feature" branch code on top of the "main"
   branch code.

6) After that we need to replicate the same rebase operation with the EdgeDB
   branch. Our CLI tools may need to first clone the "main" branch with the
   data into a "temp" branch. Then we can introspect the migration histories
   of "temp" and "feature" branches so that we can establish where they
   diverge. Take all the divergent migrations from the "feature" branch and
   apply them to the "temp" branch. If the operation is successful, drop the
   "feature" branch and rename "temp" to "feature". We now have successfully
   rebased "feature" branch on top of "main".

7) Since the state of "feature" is now a straightforward fast-forward w.r.t.
   the "main" branch we can finally merge "feature" back into main in VCS and
   then merge the EdgeDB branch as well (or rename "feature" EdgeDB branch
   into "main", if the old branch is no longer needed).

In our CLI tools we need a ``edgedb branch rebase`` command to perform step 6)
and also a ``edgedb branch merge`` command to perform a fast-forward merge (by
copying the migration history and applying migrations).


Implementation
--------------

Most of the ``branch`` functionality is either existing ``database``
functionality or can be implemented on top of that by performing migrations.

The option to copy the data can be implemented in Postgres by using the
``CREATE DATABASE newname WITH TEMPLATE = oldname`` which may be preferable
to a dump/restore as it should be faster. The caveat is that unlike with a
dump/restore the template database cannot have any other active connections.
However, this may be reasonable for local development as the benefit is speed.

If we need to use dump/restore approach for creating a new branch ideally we
should try and see if we can pipe the ``pg_dump`` directly into ``pg_restore``
rather than writing files to disk. The dump files may be large (especially if
we implement branch creation *with all data copied*) and if we can avoid
creating them it should avoid various failure scenarios due to disk
space/permission issues.

We need to start bundling ``pg_dump`` and ``pg_resotre`` tools so that we can
run them ourselves when we need them for handling new branches.


Future Considerations
---------------------

We should eventually have a way to simplify working with git branches and
keeping the database branches synchronized. Merging or rebasing of database
branches can also benefit from git integration in order to correctly identify
the migration history and create reasonable migration files. We can also
possibly apply rebasing logic in smaller steps as we could access the
intermediate schema states from the git commit history.


Backwards compatibility
=======================

Database Keyword
----------------

This proposal deprecates the keyword ``database``. We will keep the old
keyword and syntax for backwards compatibility, though. Semantically the old
commands will have an equivalent new command:

* ``create database <name>`` is the same as ``create empty branch <name>``
* ``drop database <name>`` is the same as ``drop branch <name>``

Project Config
--------------

All the EdgeDB clients will have to be upgraded so that they can look up both
``edgedb.toml`` and ``edgedb.auto.toml`` files. An older client will simply
ignore the ``edgedb.auto.toml`` which should be fine for any deployment
environment since we're assuming that these alternative branches are used for
development rather than deployment.

When the ``edgedb.toml`` file does not explicitly specify the branch/database
(and when ``edgedb.auto.toml`` is missing) the default branch name to connect
to depends on the EdgeDB server version:

* EdgeDB 5+ should assume "main" branch as default
* EdgeDB 4.x and prior should assume "edgedb" branch as default


Implementation plan
===================

The proposal can be implemented in stages.


Design Considerations
=====================

DDL vs CLI
----------

The DDL command should be very explicit regarding creation of new branches.
There must not be ambiguity or magic, thus the command is either explicitly
using the ``empty`` keyword or specifying a ``from`` branch. For the CLI,
however, it is acceptable to allow omitting the ``--from`` clause as a
shorthand for branching from whatever the current branch is. This workflow is
similar to how git branching works and thus should be familiar to many
developers.

Local Config
------------

We decided that storing the current branch in ``edgedb.toml`` directly is
problematic since it's way too easy to accidentally commit that into the wrong
VCS branch. So the solution is to use a special ``edgedb.auto.toml`` file
that should not be committed (in fact it's recommended to add it to
``.gitignore``).

This naming pattern was chosen to be similar to Postgres "auto" files. The
alternative ``.local`` filename was rejected as this is not intended to be
edited by the user directly, but rather is an autogenerated config.

This file is easy to find so that the developers don't have to wonder where
the configuration is coming from.

We prefer the structure of ``.toml`` format as more human-friendly. We also
don't want to introduce arcane environment variables.

This also means that if we later decide to introduce a user-oriented
``.local`` configuration file we can still do that and differentiate between
user local config and autogenerated config that doesn't need to be committed
into the VCS repository.

VCS Integration
---------------

The VCS integration can be a nice touch, but it also should be optional. We
can offer suggestions, even offer simple tools, pre-commit or post-checkout
hooks, but we cannot rely on the developers using them. Thus we might offer
some git integration, but it cannot be critical to how EdgeDB branches
operate.

A post-checkout git hook can help us switch EdgeDB branches in sync with
checking out git branches. If we're able to detect when a new branch is
created (so that previous branch SHA is the same as the new one) we can also
add a record to ``edgedb.auto.toml`` associating the new git branch with the
same EdgeDB branch as the old one. Conversely when switching to an existing
git branch that doesn't appear in ``edgedb.auto.toml`` yet it is probably
safer to assume "main" EdgeDB branch as apparently there was no specific need
to create a separate EdgeDB branch for it before, so the changes probably
don't affect the schema and the "main" branch should be fine to use.

Data Copy
---------

We've decided against ``using copy_data := True`` syntax and instead settled
on making the create command having an explicit modifier ``empty``,
``schema``, or ``data``. This seems to make ``create branch`` commands more
explicit and clear.
