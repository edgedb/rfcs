::

    Status: Draft
    Type: Feature
    Created: 2023-01-23
    Authors: Yury Selivanov <yury@edgedb.com>


======================================================
RFC 1022: Typing free objects & simplifying SDL syntax
======================================================

Abstract
========

This RFC proposes enhancements to the EdgeDB type system allowing users to
explicitly type free objects. Free object types can then be used to type
function parameters or used to indicate types of query arguments.

While discussing the necessary new syntax for typing free objects, this
RFC proposes a set of adjustments to the current EdgeDB Schema Declaration
Language syntax.


Typing free objects
===================

The proposed grammar is structured with the following requirements in mind:

* The syntax must be "light" as it will appear frequently in EdgeQL
  queries inside cast (e.g. ``<type_expr>$foo``) constructs.

* We want to automatically infer as many things as possible.

* We want the syntax to play nice with the existing syntax for typing
  arrays and tuples.

The grammar looks as follows::

    type_name:
        # name of the type, such as "int64" or "default::User"

    type_expr:
        # type expressions valid in SDL, such as "A | B" or
        # "array<int64>"

    property_or_link:
        ["multi" | "single"] ["required" | "optional"] ["property" | "link"] \
            name ":" type_name | type_expr | free_object_type

    free_object_type:
        "{"
            property_or_link
            [ "," property_or_link ] *
        "}"


Key takeaways:

* EdgeDB will infer whether a field is a property or a link, making
  "property" and "link" keywords optional.

* Similar to SDL, all properties and links are *optional* and
  *single* by default.

(Note that the syntax deviates from SDL: we use ``:`` instead of ``->`` and
``property`` / ``link`` modifier keywords are optional. To make the new free
object types syntax compatible with SDL we also propose to update the SDL
syntax in the below `Updates to SDL`_ section.)

Examples:

A free object as a query argument::

    with
        data := <{a: int64, b: str}>$arg
    select
        data.a + <str>data.b;

A more complex free object with nested data and JSON casts::

    with
        data := <{
            required pokemon: str,
            multi required locations: {
                required point: tuple<float64, float64>,
                street: str
            }
        }>json_array_unpack(<json>$arg)

    for item in data union (
        insert Pokemon {
            name := item.pokemon,
            # ...
        }
    )

In user-defined functions::

    function compute(pokemon: {
        required pokemon: str,
        multi required locations: {
            required point: tuple<float64, float64>,
            street: str
        }
    }) -> int64 using (
        select # ...
    )


Updates to SDL
==============

Summary:

* Make the ``property`` and ``link`` keyword optional for all non-computed
  properties and links. The kind of a pointer can be inferred automatically
  in all situations. Users will be encouraged to explicitly specify the
  kind of a pointer for readability or enforcement purposes where necessary.

* Replace the ``->`` symbol with ``:`` for type annotations of pointers.
  ``->`` will still be valid syntax, but we will update our documentation
  and output of introspection commands to use ``:``.

* Keep the ``->`` symbol for function return type annotation (in other words
  we will not add support for using ``:`` in return type annotation).

Motivation:

* The SDL syntax of declaring object types and ad-hoc free object types
  has to look similar. But having ``->`` symbols inside EdgeQL's casts will
  negatively impact the readability of the language, e.g. compare
  ``select <{x: int64, y: int64}>`` vs ``select <{x -> int64, y -> int64}>``.

* The history of the ``->`` syntax in SDL has its roots back in the time when
  SDL was a whitespace sensitive language. We could not use ``:`` because it
  used to denote a block, e.g.::

    property foo -> int64:
      annotation title := 'the foo-est of foos'

  However, the modern SDL isn't whitespace sensitive and we can now use ``:``.

* There's no strict requirement for forcing users to use ``property`` and
  ``link`` modifiers. We can always infer the kind of the pointer by its
  specified or inferred type. The intent mainly was to improve the readability
  of the schema with explicit modifiers. However, now it is clear that (a) we
  need a shorter syntax for inline object typing, and (b) our users wish
  that our SDL would have a more concise syntax. By making ``property`` and
  ``link`` optional our SDL syntax becomes closer to popular languages
  like TypeScript, ultimately improving the DX and the first impression of our
  SDL.

Example:

+--------------------------------------+--------------------------------------+
| Current syntax:                      | Proposed syntax:                     |
+======================================+======================================+
|::                                    | ::                                   |
|                                      |                                      |
|  abstract type Content {             |   abstract type Content {            |
|    required property title -> str;   |     required title: str;             |
|    multi link actors -> Person {     |     multi actors: Person {           |
|      property character_name -> str; |       character_name: str;           |
|    };                                |     };                               |
|  }                                   |   }                                  |
|                                      |                                      |
|  type Movie extending Content {      |   type Movie extending Content {     |
|    property release_year -> int32;   |     release_year: int32;             |
|  }                                   |   }                                  |
|                                      |                                      |
|  type Show extending Content {       |   type Show extending Content {      |
|    property num_seasons :=           |     property num_seasons :=          |
|      count(.<show[is Season]);       |       count(.<show[is Season]);      |
|  }                                   |   }                                  |
+--------------------------------------+--------------------------------------+


Implementation timeline
=======================

Typing free objects sounds like an EdgeDB 4.0 feature, especially given that
we enable tuples to be used as query arguments in 3.0.

However, the proposed SDL changes (making ``property`` and ``link`` optional
and replacing ``->`` with ``:``) are better to implement as early as possible,
in other words in 3.0. Happily the proposed change should be backwards
compatible.


Future enhancements
===================

Simplifying tuple/array type syntax
-----------------------------------

We can potentially simplify tuple and array types syntax as follows:

===================================== =========================================
Current                               Future
===================================== =========================================
``select <tuple<int64, str>>$0``      ``select <(int64, str)>$0``
``select <array<tuple<int64>>>$0``    ``select <[(int64,)]>$0``
``property foo -> tuple<int64, str>`` ``foo: (int64, str)``
===================================== =========================================


Rejected ideas
==============

Make "required" the default pointer type in free objects
--------------------------------------------------------

This would draw nice parallels between tuples and free objects in EdgeQL
queries, but would make it exceptionally hard to comprehend types in SDL files.


Make property/link modifiers optional for computed fields
---------------------------------------------------------

Computed pointers and fields like ``default`` both use the ``:=`` syntax.
Making ``property`` and ``link`` modifiers optional for computeds would make
them look like fields.
