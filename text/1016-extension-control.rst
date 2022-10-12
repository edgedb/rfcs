::

    Status: Draft
    Type: Feature
    Created: 2022-10-07
    Authors: Victor Petrovykh <victor@edgedb.com>

===========================================
RFC 1016: Enabling and disabling extensions
===========================================

This RFC proposes a generic mechanism for enabling and disabling extensions.


Motivation
==========

Extensions are part of the schema and affect functionality of the database, so
adding or removing them can have wide-ranging effects on the applications
using the database, not the least of which are the migrations necessary to
make this change. The ability to load an extension, but keep it disabled when
not needed is often desirable in development or testing environments. Loading
the extension would register all the necessary schema changes, thus making
sure that the enabled and disabled extension states are compatible with the
same migration history. The only difference would be extension functionality,
which could allow better flexibility when testing parts of the system that
don't require the extension in question.


Specification
=============

Loaded extensions always export all their definitions, so that any symbol
used in them is valid in EdgeQL or DDL regardless of whether the given
extension is actually enabled or not. Disabled extensions may either produce
errors when used or even silently perform an alternative [fallback] action.

We want to introduce a special command to switch an extension on or off::

  configure extension <ext-name> [enable | disable];

Potentially this might be under the umbrella of ``configure current
database``.

The configuration value could be reflected via a multi link in
``cfg::Config``::

  type cfg::DatabaseConfig extending  cfg::AbstractConfig {
    multi link extensions -> cfg::Extension;
  }

  type cfg::Extension extending cfg::ConfigObject {
    required property name -> str;
    required property is_enabled -> bool {
      default := true;
    }
    required property is_togglable -> bool {
      default := false;
      readonly := true;
    }
  }

The ``is_togglable`` property indicates whether an extension can be toggled on
and off. If it's ``false``, then attemting to toggle the extension state is an
error. This value can only be set at the time of creating an extension and is
immutable after that.

The ``is_enabled`` property indicates whether a given extension is enabled or
not and can potentially be changed by the special ``configure extension``
command as long as the extension is togglable.


Backwards Compatibility
=======================

There should not be any backwards compatibility issues.


Security Implications
=====================

Extensions potentially expose some data to a third-party service. Thus even
after an extension is disabled this data may persist elsewhere even if it's no
longer accessible in EdgeDB.


Rejected Alternative Ideas
==========================

We considered having arbitrary custom config values affecting whether an
extension is enabled or disabled. In particular this was intended to provide
more granular enabling/disabling for a given extension. It was rejected
because the command to enable or disable an extension would then be basically
a ``configure instance set`` or similar. Generally most ``configure`` commands
don't have as far-reaching consequences as enabling an extension could.
Extensions may require things like re-indexing the entire database or even
reflect a large portion of it to some external service. These operations are
significantly more constly than typical ``configure`` tweaks.

We rejected ``alter extension <ext-name> set is_enabled := [true | false];``
as it appears to be a schema changing command, whereas we want to emphasize
that it doesn't touch the schema.

We rejected showing the extension status in the ``schema::Extension`` because
this status is a special configuration rather than part of the schema.

We rejected ideas of using one property ``is_enabled`` to indicate both
whether an extension is enabled or disabled as well as whether it is togglable
in principle. To do so we'd have to user the two ``bool`` values as well as
``{}`` to encode the extension state. This seems a bit more convoluted.