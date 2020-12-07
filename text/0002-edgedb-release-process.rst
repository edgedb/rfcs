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

1. Check with the team on the #engineering channel in Slack whether there
   are any release blockers.

2. Check if the latest nightly build of the CLI and of the server was
   green.


Publishing the binaries
=======================

CLI
---

1. Create a release branch:

    - git switch -c releases/1.0a6

    - update version in ``Cargo.toml`` and run ``cargo check``

    - commit (ideally with a name like "edgedb-cli 1.0a6"

2. Tag the commit:

    - git tag -s v1.0a6

    - the message can be something like:

        > v1.0a6 "Wolf 359"
        >
        > See changelog at
        > https://github.com/edgedb/edgedb/blob/master/docs/changelog/1_0_a6.rst

3. Start the release flow by going to:

    - https://github.com/edgedb/edgedb-cli/actions?query=workflow%3A%22Build%2C+Test%2C+and+Publish+a+Release%22

Server
------

.. note::

    Only build the server after the CLI is published.

1. Create a release branch:

    - git switch -c releases/1.0a6

2. Tag the last commit:

    - git tag -s v1.0a6

    - the message can be something like:

        > v1.0a6 "Wolf 359"
        >
        > See changelog at
        > https://github.com/edgedb/edgedb/blob/master/docs/changelog/1_0_a6.rst

4. Start the release flow by going to:

    - https://github.com/edgedb/edgedb/actions?query=workflow%3A%22Build+Test+and+Publish+a+Release%22

5. When all is good, you can check out the tag locally, build EdgeDB
   and run `edb gen-test-dumps` to generate test dumps for the version
   you're releasing now.  Commit them to **master**, not to the release
   branch, they're not needed there.


External places to bump binaries at
-----------------------------------

1. Update tutorial.edgedb.com to run on the latest release. The package
   to update is edgedb-cloud/docker/embedded/, use the README there for
   update instructions. After uploading a new package to ECR, kick the
   Fargate job by running `edbcloud fargate tutorial/t1 --force`.

2. Update Docker Hub. This should happen automatically during the server
   GitHub Action release build (debian-buster).

3. Update the Homebrew tap.

  The tap is auto-updating nightly. If you need to bump it faster,
  use the HTTP repository dispatch documented in the README of the
  tap.

  Alternatively, on an installed Homebrew the repository lives in
  /usr/local/Homebrew/Library/Taps/edgedb/homebrew-tap.  Go there
  and run `./autoupdate.py`, commit and push the changes.




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
