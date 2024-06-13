::

    Status: Accepted
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
      *  # Will expand into "id, name".
    };

  An example of ``*`` including inherited pointers::

    select Hero {
      *  # Will expand into "id, name, secret_identity".
    };

  An example of ``*`` including pointers that were already specified::

    select Hero {
      name := 'try me!',
      *,  # Will expand into "id, name, secret_identity",
          # where `name` will be coming from the `name := 'try me!'`
          # computed.
    };

  An complex example illustrating using ``*`` in nested shapes::

    select Hero {
      *,  # Will expand into "id, name, secret_identity".

      villains: {
        *  # Will expand into "id, name"
           # (the `Villain` type doesn't have the `secret_identity` property).

        nemesis: {
          *  # Will expand into "id, name, secret_identity".
        }
      }
    };

  Another example to demonstrate that splats work at a "symbolic"
  level, expanding the actual shape to be selected, yet ignoring how
  and where things are actually defined::

    with
      CapHero := Hero { name := str_upper(.name) }
    select
      (
        CapHero { * },      # will select a shape with upper-cased names
        CapHero { Hero.* }  # will *also* select a share with upper-cased names
      )

* ``**``: extend the shape with all properties and *links* defined on the type,
  including the inherited ones. Linked shapes will be defined with the
  ``*`` splat.

  A simple example::

    select Person {
      **  # Will expand into "id, name".
          # The `Person` type doesn't have any links defined on it.
    };

  An example of ``**`` including inherited pointers and links::

    select Hero {
      **  # Will expand into:
          #   {
          #     id,
          #     name,
          #     secret_identity,
          #     villains: { * }
          #   }
          #
          # which will in turn expand into:
          #   {
          #     id,
          #     name,
          #     secret_identity,
          #     villains: { id, name }
          #   }
    };

  It's possible use ``**`` and redefine the pointers it expands into::

    select Hero {
      **,
      villains: {     # Use `**` to auto-include all linked types
        name,         # into the shapes, but define the `villains`
        level := 80   # link to include just the `name` property
      }               # and the `level` computed.
    };

  Note that ``**`` does not expand the ``__type__`` link.

* ``<type expression>.*`` and ``<type expression>.**``: extend the shape with
  all properties/links reachable from the computed type of ``type expression``.

  A trivial example when the type expression is a reference to the base type::

    select Hero {
      Person.*  # Will expand into "id, name".
    };

  A more complicated type expression using ``*``::

    select Hero {
      (Hero | Villain).*  # Would expand to "id, name".
    }

  A more complicated type expression using ``**`` (the query wouldn't
  compile but we use it nevertheless to illustrate the proposed behavior
  of ``**``)::

    select Hero {
      (Hero & Villain).**  # Would expand into
                           #   {
                           #     id,
                           #     name,
                           #     secret_identity,
                           #     villains: { * },
                           #     nemesis: { * }
                           #   }
    }

* ``[is ...].*`` and ``[is ...].**``: polymorphic variants for the above
  splat syntaxes.

  An example of ``*``::

    select Person {
      [is Hero].*  # Expands into
                   #   {
                   #      [is Hero].id,
                   #      [is Hero].name,
                   #      [is Hero].secret_identity,
                   #   }
    }

  An example of ``**``::

    select Person {
      [is Hero].**  # Expands into
                    #   {
                    #      [is Hero].id,
                    #      [is Hero].name,
                    #      [is Hero].secret_identity,
                    #      [is Hero].villains: { * },
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


Rejected ideas
==============

Use prefix/postfix ``...`` for splats
-------------------------------------

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

* With ``...`` as a postfix operator implementing the proposed ``*`` syntax it
  is unclear how we would design its ``**`` variant. Using postfix ``......``
  operator is obviously not a viable option.


Make ``*`` expand to both links and properties
----------------------------------------------

* Users will inevitably use splats in their application code (i.e. not just in
  REPL) and selecting all links can make queries slower. Besides, selecting all
  properties is typically a more common need than selecting all properties
  and all linked data.

* We already have splats in our TypeScript query builder API and the current
  implementation only expands ``*`` into list of properties.


Field exclusion syntax
----------------------

Field exclusion can be useful to splat every property from a type except a
few specific ones. For example, an earlier revision of this RFC was proposing
to use the ``never`` type for this purpose::

     function validate(data: {
       required Hero.*,
       id: never,

       # will expand into:
       #   {
       #     required name: str,
       #     required secret_identity: str
       #   }
     }) -> bool using (...)

However, it was pointed out that in the context of EdgeQL using ``never`` like
this can be problematic, as it would propagate through the query typing
converting everything to ``never``. Another alternative would be to use an
unary ``-`` operator, as in::

     function validate(data: {
       required Hero.*,
       -id,

       # will expand into:
       #   {
       #     required name: str,
       #     required secret_identity: str
       #   }
     }) -> bool using (...)

the downside of that approach is that the semantics of ``-`` in this context
is not entirely clear. Ultimately it was decided that designing the field
exclusion syntax, while possible, is out of scope of this proposal.

Allow splats to be used on values
---------------------------------

The following query would not compile::

    with
      h := (select Hero { computed := 42 })
    select
      Hero {
        h.*  # compile-time error!
      }

the query would fail with a compile-time error suggesting that ``.*`` can
only be used on a *type*. A simple way to fix the query would be to
use the ``typeof`` operator::

    with
      h := (select Hero { computed := 42 })
    select
      Hero {
        (typeof h).*  # The shape will expand to
                      #   {
                      #     id,
                      #     name,
                      #     secret_identity,
                      #     computed,
                      #   }
      }


Backwards compatibility
=======================

The proposal is fully backwards compatible.


Implementation plan
===================

The proposal can be implemented in stages. E.g. EdgeDB version 3.0 will have
the basic ``*`` and ``**`` operators supported in shapes, while EdgeDB 4.0
or later can have the proposed type language extensions implemented.
