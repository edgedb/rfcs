::

    Status: Draft
    Type: Feature
    Created: 2023-01-25
    Authors: Yury Selivanov <yury@edgedb.com>


==============================
RFC 1023: Adding Splats Syntax
==============================

Abstract
========

We propose to add a splats syntax to EdgeQL. The closest equivalents to this
proposal are fragments in GraphQL and `SELECT *` in SQL.

Splats in ``select`` commands and in free object types will allow for better
REPL user experience and will improve readability of complex queries and type
expressions.


Splats in select shapes
=======================

EdgeQL *shapes* is a syntax construct used to specify what exact properties,
links, and ad-hoc computed data should be fetched by the query.

This proposal assumes the following schema for all of its example code::

    abstract type Person {
      required property name -> str { constraint exclusive };
    }

    type Hero extending Person {
      property secret_identity -> str;
      multi link villains := .<nemesis[is Villain];
    }

    type Villain extending Person {
      link nemesis -> Hero;
    }

With this schema in mind, here's an example of the EdgeQL's shape syntax::

    select Person {
      name,  # propety name

      name_upper := str_upper(.name),  # computed property

      villains: {  # fetch data traversing the "villains" link
        name
      },

      [is Hero].secret_identity,  # fetch `secret_identity` property
                                  # for all `Person` objects that are also
                                  # instances of `Hero`
    }

We propose to enhance the shape construct with the following syntax:

* ``*``: extend the shape with all properties (not links!) defined on the type,
  including the inherited ones.

  A simple example::

    select Person {
      *  # will expand into "id, name"
    };

  An example of ``*`` including inherited pointers::

    select Hero {
      *  # will expand into "id, name, secret_identity"
    };

  An example of ``*`` including pointers that were already specified::

    select Hero {
      name := 'try me!',
      *,  # will expand into "id, name, secret_identity" leading to
          # a compile-time error (for the same reason why
          # ``select Hero {name, name := 'blah'};`` would not compile.)
          # No shadowing is allowed.
    };

  An complex example illustrating using ``*`` in nested shapes::

    select Hero {
      *,  # will expand into "id, name, secret_identity"

      villains: {
        *  # will expand into "id, name"
           # (the `Villain` type doesn't have the `secret_identity` property)

        nemesis: {
          *  # will expand into "id, name, secret_identity"
        }
      }
    };

* ``<type expression>.*``: extend the shape with all properties reachable from
  the computed type of ``type expression``.

  A trivial example when the type expression is a reference to the base type::

    select Hero {
      Person.*  # will expand into "id, name"
    };

  A more complicated type expression::

    select Hero {
      (Hero | Villain).*  # would expand to "id, name"
    }

* ``[is ...].*``: a polymorphic variant.

  Example::

    select Person {
      [is Hero].*  # expand into
                   #   {
                   #      [is Hero].id,
                   #      [is Hero].name,
                   #      [is Hero].secret_identity,
                   #   }
    }


Splats in free object types
===========================

This section builds on the concepts introduced in
`RFC 1022 - Typing free objects & simplifying SDL syntax <./1022-freetypes.rst>`_.

Allowing splats to be used in the EdgeQL's type sub-language (particularly,
allowing them to be used in free object type declarations) will
lead to more concise function declarations and type casts.

We propose to extend the free shape type syntax with the following constructs:

* `<type expression>.*`: include all properties from the computed type of
  ``type expression`` to the final free object's type. Example::

     function validate(data: {
       Person.*
     }) -> bool using (...)

     # `data` parameter will accept free objects that have all properties
     # declared in the Person type (retaining their cardinality bounds & types)

   An example of a more complicated type expression::

     function validate(data: {
       (Hero | Villain).*,  # will expand into:
                            #   { required id: uuid, required name: str }

       foo: str,            # add a "foo" property to this free object type
     }) -> bool using (...)

* `<modifier> <type expression>.*`: include all properties from the computed
  type expression overriding cardinality.

   An example of including all properties from another type but making
   them all optional:

     function validate(data: {
       optional Person.*
     }) -> bool using (...)

     # `data` parameter will accept free objects that have all properties
     # declared in the Person type (making them all optional)

  An example of making all expanded fields required:

     function validate(data: {
       required Hero.*  # will expand into:
                        #   {
                        #     required id: uuid,
                        #     required name: str,
                        #     required secret_identity: str
                        #   }
     }) -> bool using (...)

When we add a ``never`` type eventually (to pave the path to implementing
the ``raise`` expression) we will allow ``never`` types to shadow properties
expanded from splats::

     function validate(data: {
       required Hero.*,
       id: never,

       # will expand into:
       #   {
       #     required name: str,
       #     required secret_identity: str
       #   }
     }) -> bool using (...)


Rejected ideas
==============

Use ``...`` for splats
----------------------

The prefix ``...`` operator, available in JavaScript (the spread operator)
and in GraphQL (fragments), seemed like a viable alternative to ``*``.

We decided against using it in EdgeQL for the following reasons:

* With the existing EdgeQL grammar in mind, ``...[is Hero]`` splat would
  look to the reader as if ``[is Hero]`` is applied to the result of the splat.
  E.g.::

    select Person {
      ...[is Hero]
    }

  Would be interpreted as::

    select {
      id[is Hero],
      name[is Hero]
    }

  which is nonsense.

* ``...`` as a prefix operator would make type expressions syntax look
  inconsistent when a splat is used next to a direct field reference.

  Compare:

  +----------------------------------+-----------------------------------+
  |::                                | ::                                |
  |                                  |                                   |
  |  {                               |   {                               |
  |    Foo.prop,                     |     Foo.prop,                     |
  |    Bar.*                         |     ...Bar                        |
  |  }                               |   }                               |
  +----------------------------------+-----------------------------------+


Backwards compatibility
=======================

The proposal is fully backwards compatible.


Implementation plan
===================

The proposal can be implemented in stages. E.g. EdgeDB version N can ship
with the basic ``*`` supported in shapes, and EdgeDB N+1 can add the rest
of the proposed syntax and capabilities.
