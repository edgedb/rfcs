::

    Status: Active
    Type: Process
    Created: 2020-10-08
    Authors: Łukasz Langa <lukasz@edgedb.com>
    RFC PR: `edgedb/rfcs#0022 <https://github.com/edgedb/rfcs/pull/22/>`_

====================================
RFC 0002: The EdgeDB Release Process
====================================


Overview
========

This document describes how we're releasing a new version of EdgeDB.
This includes both the CLI tool and the server, as well as announcements.


Before we begin
===============

In general, the order of releasing various components should be the
following: bindings, CLI, server.

This means that we want to make sure that the nightly bindings work
with the old and the new server. Similarly, we want to make sure that
the nightly CLI works with the old and new server.

1. Check with the team on the ``#engineering`` channel in Slack
   whether there are any release blockers.

2. Check if the latest nightly build of the CLI and of the server was
   green.

3. Perform the pre-release testing to ensure that the users have a
   smooth upgrading experience.


Pre-release testing
===================

The "bindings" in the checklist below refer to all of our currently
supported bindings:

- `Python <https://github.com/edgedb/edgedb-python>`_ bindings.
- `JavaScript and TypeScript
  <https://github.com/edgedb/edgedb-js>`_ bindings.
- `Go <https://github.com/edgedb/edgedb-go>`_ bindings.
- `Deno <https://github.com/edgedb/edgedb-deno>`_ bindings.

All of the binding tests need to be performed for each of our
supported bindings.

1. Using an old version (stable release) of CLI and server, either
   initialize a new project or restore an existing one.

2. Update the bindings to the nightly version.

3. Try to connect and run some queries using the new bindings and an
   old (stable release) version of the server.

4. Update the CLI to the nightly version.

5. Try connecting to the old (stable release) server instances. Run
   queries.

6. Create a new project using the old server to run instances. Make
   sure the following sub-tasks are possible:

   - successfully initialize a new project
   - connect to it
   - run queries
   - perform at least one migration

7. Upgrade the server to the nightly build.

8. Try to connect to an existing (freshly upgraded) project via the
   CLI and run queries.

9. Try to connect and run some queries using the new bindings and the
   freshly upgraded project.

10. Try initializing a new project using the nightly server. Make sure
    the following sub-tasks are possible:

    - successfully initialize a new project
    - connect to it
    - run queries
    - perform at least one migration

11. Try to connect and run some queries using the new bindings and the
    freshly created project.


Publishing the binaries
=======================

CLI
---

1. Create a release branch:

   - ``git switch -c releases/1.0a6``

   - update version in ``Cargo.toml`` and run ``cargo check``

   - commit (ideally with a name like "edgedb-cli 1.0a6"

2. Tag the commit:

   - ``git tag -s v1.0a6``

   - the message can be something like::

        v1.0a6 "Wolf 359"

        See changelog at
        https://github.com/edgedb/edgedb/blob/master/docs/changelog/1_0_a6.rst

   - ``git push --follow-tags``

3. Start the release flow by going to:

   - https://github.com/edgedb/edgedb-cli/actions/workflows/release.yml?query=workflow%3A%22Build%2C+Test%2C+and+Publish+a+Release%22
   - select the newly pushed release branch from the dropdown and press "Run Workflow"

Server
------

.. note::

    Only build the server after the CLI is published.

1. Create a release branch:

   - ``git switch -c releases/1.0a6``

2. Tag the last commit:

   - ``git tag -s v1.0a6``

   - the message can be something like::

        v1.0a6 "Wolf 359"

        See changelog at
        https://github.com/edgedb/edgedb/blob/master/docs/changelog/1_0_a6.rst

   - ``git push --follow-tags``

4. Start the release flow by going to:

   - https://github.com/edgedb/edgedb/actions/workflows/release.yml?query=workflow%3A%22Build+Test+and+Publish+a+Release%22
   - select the newly pushed release branch from the dropdown and press "Run Workflow"

5. When all is good, you can check out the tag locally, build EdgeDB
   and run ``edb gen-test-dumps`` to generate test dumps for the version
   you're releasing now.  Commit them to **master**, not to the release
   branch, they're not needed there.


External places to bump binaries at
-----------------------------------

1. Update tutorial.edgedb.com to run on the latest release. The package
   to update is edgedb-cloud/docker/embedded/, use the README there for
   update instructions. After uploading a new package to ECR, kick the
   Fargate job by running ``edbcloud fargate tutorial/t1 --force``.

2. Update Docker Hub. This should happen automatically during the server
   GitHub Action release build (debian-buster).

3. Update the DigitalOcean image. Instructions can be found in
   edgedb-deploy/digitalocean-1-click/README.md.

4. Update the Homebrew tap.

The tap is auto-updating nightly. If you need to bump it faster,
use the HTTP repository dispatch documented in the README of the
tap.

Alternatively, on an installed Homebrew the repository lives in
``/usr/local/Homebrew/Library/Taps/edgedb/homebrew-tap``.  Go there
and run ``./autoupdate.py``, commit and push the changes.




Updating the Website
====================

The Downloads page
------------------

A number of places on the `Downloads <downloads_>`_ page refer to
a particular version. In particular you want to update:

* src/pages/download.jsx
* src/pages/index.jsx
* content/download/linux.centos.md
* content/download/linux.ubuntu.*
* content/download/linux.debian.*

Announcement Blog Post
----------------------

Looking for a theme in the changelog is a good way to phrase the
announcement blog post.  Remember to give the post a fresh UUID.


.. _downloads: https://edgedb.com/download
