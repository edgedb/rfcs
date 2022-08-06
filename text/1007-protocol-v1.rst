::

    Status: Draft
    Type: Feature
    Created: 2022-07-21
    Authors: Fantix King <fantix@edgedb.com>
    RFC PR: `edgedb/rfcs#0062 <https://github.com/edgedb/rfcs/pull/62>`_

=====================
RFC 1007: Protocol v1
=====================

This RFC proposes to change the EdgeDB binary protocol, and bumps the protocol
version to 1.0, including simplified messages, script handling and a stateless
design.

Pre-RFCs: edgedb/edgedb#3772, edgedb/edgedb#3933, edgedb/edgedb#4009


Motivation
==========

EdgeDB binary protocol v0 has been serving EdgeDB 1.x pretty well, but we could
see multiple places to optimize, especially after RFC 1010 [1]_:

1. Some of the concepts or designs are preserved from the PostgreSQL binary
   protocol, but they are now proved to be redundant or useless in EdgeDB, like
   the concept of prepared statements.

2. Message headers are complicating things, quoting Yury in edgedb/edgedb#3772:

     The motivation for adding headers in the first place was to simplify the
     process of extending the protocol with new functionality. The idea was
     that drivers won't need to change the parsing logic too much.
     Unfortunately, in practice, this resulted in all headers having arbitrary
     values encoding and even more complicated messages parsing code.

3. ``ExecuteScript`` in v0 is broken. It doesn't guarantee atomicity like its
   equivalent does in Postgres (``SimpleQuery``); transaction control can be a
   real mess if used in an improper way and EdgeDB has no limit for that; and
   it doesn't take arguments or return any value, which would be nice to have.

4. Protocol v0 maintains a physical "session" (concept is similar to Postgres)
   on the server-side per TCP connection, even when there is no active
   transaction. This stops EdgeDB from having lightweight stateless client
   connections that could come and go with minimal cost, especially when we
   want to tunnel the binary protocol over HTTP. Also, a stateless design would
   give the client bindings a lot of flexibility to maintain virtual sessions
   in their preferred ways, when it comes to access controls particularly.


Overview
========

Per proposal, EdgeDB 2.0 will be delivered with binary protocol v1.0, while
still supporting the legacy protocol back to version 0.13 for the time being.
Pre-v0.13 protocol handshakes will be rejected, however database dumps from the
early EdgeDB 1.0 alpha versions are still restore-able in EdgeDB 2.0.

EdgeDB official client bindings - at the time of EdgeDB 2.0 - will support any
servers running binary protocol 0.13 or above. This may change as protocol v0
is phased out as time goes.

A summary of all protocol changes here, and we'll explain later in details.

+----------------------------+----------------------------+---------+-------------------------+--------+----------------------------+-----------------------------------+
| V0 Name                    | V1 Name                    | Code    | Change                  | Header | Replacement                | Link                              |
+============================+============================+=========+=========================+========+============================+===================================+
|                            | ``Annotation``             |         | :white_check_mark:Added |        |                            | `diff <#annotation>`_             |
+----------------------------+----------------------------+---------+-------------------------+--------+----------------------------+-----------------------------------+
| ``Header``                 | ``KeyValue``               |         | :pencil2:Modified       |        | Also ``Annotation``        | `diff <#keyvalue>`_               |
+----------------------------+----------------------------+---------+-------------------------+--------+----------------------------+-----------------------------------+
| ``Prepare``                | ``Parse``                  | ``'P'`` | :pencil2:Modified       | Yes    |                            | `diff <#parse>`_                  |
+----------------------------+----------------------------+---------+-------------------------+--------+----------------------------+-----------------------------------+
|                            | ``Capability``             |         | :white_check_mark:Added |        |                            | `diff <#capability>`_             |
+----------------------------+----------------------------+---------+-------------------------+--------+----------------------------+-----------------------------------+
|                            | ``CompilationFlag``        |         | :white_check_mark:Added |        |                            | `diff <#compilationflag>`_        |
+----------------------------+----------------------------+---------+-------------------------+--------+----------------------------+-----------------------------------+
| ``IOFormat``               | ``OutputFormat``           |         | :pencil2:Modified       |        |                            | `diff <#outputformat>`_           |
+----------------------------+----------------------------+---------+-------------------------+--------+----------------------------+-----------------------------------+
| ``PrepareComplete``        |                            | ``'1'`` | :x:Dropped              |        | ``CommandDataDescription`` | `diff <#preparecomplete>`_        |
+----------------------------+----------------------------+---------+-------------------------+--------+----------------------------+-----------------------------------+
| ``DescribeStatement``      |                            | ``'D'`` | :x:Dropped              |        | ``Parse``                  | `diff <#describestatement>`_      |
+----------------------------+----------------------------+---------+-------------------------+--------+----------------------------+-----------------------------------+
| ``DescribeAspect``         |                            |         | :x:Dropped              |        |                            | `diff <#describeaspect>`_         |
+----------------------------+----------------------------+---------+-------------------------+--------+----------------------------+-----------------------------------+
|                            | ``StateDataDescription``   | ``'s'`` | :white_check_mark:Added |        |                            | `diff <#statedatadescription>`_   |
+----------------------------+----------------------------+---------+-------------------------+--------+----------------------------+-----------------------------------+
| ``CommandDataDescription`` | ``CommandDataDescription`` | ``'T'`` | :pencil2:Modified       | Yes    |                            | `diff <#commanddatadescription>`_ |
+----------------------------+----------------------------+---------+-------------------------+--------+----------------------------+-----------------------------------+
| ``Execute``                |                            | ``'E'`` | :x:Dropped              |        | ``Execute``                | `diff <#execute-v0>`_             |
+----------------------------+----------------------------+---------+-------------------------+--------+----------------------------+-----------------------------------+
| ``OptimisticExecute``      | ``Execute``                | ``'O'`` | :pencil2:Modified       | Yes    |                            | `diff <#execute-v1>`_             |
+----------------------------+----------------------------+---------+-------------------------+--------+----------------------------+-----------------------------------+
| ``ExecuteScript``          |                            | ``'Q'`` | :x:Dropped              |        | ``Execute``                | `diff <#executescript>`_          |
+----------------------------+----------------------------+---------+-------------------------+--------+----------------------------+-----------------------------------+
| ``Flush``                  |                            | ``'H'`` | :x:Dropped              |        | ``Sync``                   | `diff <#flush>`_                  |
+----------------------------+----------------------------+---------+-------------------------+--------+----------------------------+-----------------------------------+
| ``CommandComplete``        | ``CommandComplete``        | ``'C'`` | :pencil2:Modified       | Yes    |                            | `diff <#commandcomplete>`_        |
+----------------------------+----------------------------+---------+-------------------------+--------+----------------------------+-----------------------------------+

"Header" means if the message had headers in v0 and changed in v1. Besides,
these messages and structs below have only the headers changes:

+-------------------+---------+--------------------------++----------------+---------+-----------------------++-----------------------+---------+------------------------------+
| Name              | Code    | Link                     || Name           | Code    | Link                  || Name                  | Code    | Link                         |
+===================+=========+==========================++================+=========+=======================++=======================+=========+==============================+
| ``ErrorResponse`` | ``'E'`` | `diff <#errorresponse>`_ || ``LogMessage`` | ``'L'`` | `diff <#logmessage>`_ || ``ReadyForCommand``   | ``'Z'`` | `diff <#readyforcommand>`_   |
+-------------------+---------+--------------------------++----------------+---------+-----------------------++-----------------------+---------+------------------------------+
| ``RestoreReady``  | ``'+'`` | `diff <#restoreready>`_  || ``Dump``       | ``'>'`` | `diff <#dump>`_       || ``Restore``           | ``'<'`` | `diff <#restore>`_           |
+-------------------+---------+--------------------------++----------------+---------+-----------------------++-----------------------+---------+------------------------------+
| ``DumpHeader``    | ``'@'`` | `diff <#dumpheader>`_    || ``DumpBlock``  | ``'='`` | `diff <#dumpblock>`_  || ``ProtocolExtension`` |         | `diff <#protocolextension>`_ |
+-------------------+---------+--------------------------++----------------+---------+-----------------------++-----------------------+---------+------------------------------+


Headers Change
==============

The v0 ``Header`` field is proposed to be replaced by actual specific fields
in the message for those functional headers, like ``allowed_capabilities``,
``compilation_flags`` and ``implicit_limit`` in the ``Parse`` (`diff
<#parse>`_) message; while for the future-flexible informational headers, they
will be fulfilled by one of the following structs in v1:

* ``KeyValue`` (`diff <#keyvalue>`_)
* ``Annotation`` (`diff <#annotation>`_)

``KeyValue`` is basically identical as ``Header`` but with a different name,
used for specific messages that still requires arbitrary attributes, like the
``ErrorResponse`` (`diff <#errorresponse>`_) message. For the remaining
majority of messages with headers, a textual ``Annotation`` is in place for any
future text information, like the ``LogMessage`` (`diff <#logmessage>`_).

``Annotation`` must only be used for auxiliary information not essential for
the given protocol message's functionality, e.g. tracing and debug data. Both
the server and the client implementations should work with ignoring annotations
completely.


Command Phase Message Flow
==========================

.. raw:: html

  <img src="./EdgeDB-Protocol-v0.png" height="371px" align="right">

In protocol v0, the command-phase granular flow is like, there are 3 sub-flows:

1. ``Prepare`` -> ``DescribeStatement`` -> ``Execute``

   This is the basic flow for all new queries without cached descriptors.

2. ``OptimisticExecute``

   Only when descriptors are cached and they matches the server knowledge, can
   the client complete the query with one single message.

3. ``OptimisticExecute`` -> ``Execute``

   Same as (2), but the relevant server schema is updated since last execute,
   that means the cached descriptors are outdated. In this case,
   ``OptimisticExecute`` behaves just like ``Prepare`` + ``DescribeStatement``,
   and the client should then complete the query with an ``Execute``.

The original reason for such design was 1) to support planned named/prepared
statements, and 2) to minimize the round-trips based on (1). However prepared
statement was never implemented, and will not be implemented as we are moving
towards a stateless protocol design, this flow is now becoming suboptimal
because of too many messages and unclear behavior, like ``Prepare`` is always
followed by ``DescribeStatement`` and the client never had to use one of them
separately; ``Execute`` cannot work alone - it must follow either ``Prepare``
or ``OptimisticExecute``; on the other hand, ``OptimisticExecute`` and
``Execute`` both execute queries, but ``OptimisticExecute`` sometimes doesn't.

.. raw:: html

  <img src="./EdgeDB-Protocol-v1.png" height="371px" align="right">

So the idea in v1 here is, drop ``DescribeStatement`` (`diff
<#deescribestatement>`_) and ``Execute`` (`diff <#execute-v0>`_), while
renaming ``Prepare`` to ``Parse`` (`diff <#parse>`_), and renaming
``OptimisticExecute`` to ``Execute`` (`diff <#execute-v1>`_), so that
``Execute`` always do execute, and ``Parse`` is only needed when the client
wants to actively cache descriptors.

In protocol v1, a successful ``Execute`` **always** mean the query is executed.
If the client provides an invalid descriptor for input arguments, the server
will return a ``CommandDataDescription`` message followed by an immediate
``ParameterTypeMismatchError``, indicating that the query was never executed.
However if it is **only** the **output** descriptor that mismatches, the server
will still execute the query, but return a ``CommandDataDescription`` message
right before the query result, so that the client could rebuild output codecs
and decode result in a single round-trip, see also `Query with State
<#query-with-state>`_.

For queries that take no arguments, the client could use the special "NULL type
ID" (``00000000-0000-0000-0000-000000000000``) as input type ID, and it is safe
to assume that the server won't return a ``ParameterTypeMismatchError`` under
protocol v1, so that simple queries can also run in a single round-trip even
without caching input descriptors.

Protocol v0 has an ``IOFormat`` enumeration for the client to choose data
serialization format, but this was never applied on input arguments. So in
protocol v1, we simply rename it to ``OutputFormat`` (`diff <#outputformat>`_),
and add a new value: ``NONE``. When set to ``NONE`` in ``Parse`` or ``Execute``
messages, the server will guarantee the output type ID is the special NULL, and
there will be no data returned, even if the given command text yields data.
In v1, ``NONE`` is the proper implementation of the recommended ``execute()``
client-bindings API, comparing to the ``query()`` API that uses ``BINARY``, or
``query_json()`` API that uses ``JSON``.

``Parse`` is no longer a must-to-have, but still provided in protocol v1 as a
dedicated way to do "parse only" without actually executing a query. ``Parse``
always return a ``CommandDataDescription`` message. Also, without the concept
of named/prepared statement, the ``statement_name`` field is no longer needed.


Script Handling
===============

As mentioned in `Motivation <#motivation>`_, ``ExecuteScript`` in protocol v0
is pretty much broken mainly due to the lack of atomicity. Protocol v1 proposes
to drop ``ExecuteScript`` (`diff <#executescript>`_), and have the new
``Parse`` and ``Execute`` handle scripts properly, as well as input/output of
scripts. This is rather a server-side change than a protocol change, but as it
changes the meaning of ``Parse`` and ``Execute``, so let's still look into it.

EdgeQL commands with more than one statements separated by top-level colons are
considered as scripts. Under protocol v1, scripts are no different than
single-statement commands - or rather, all commands can be treated as scripts.
Specifically:

1. Both client-bindings API ``execute()`` and ``query()`` accept scripts;
2. A script is always executed atomically, meaning it will be executed either
   in an implicit transaction, or as a part of the outer explicit transaction;
3. All statements in a single script share the same input arguments;
4. The output of a script is always the output of the last statement, the same
   applies on result cardinality.

Scripts must not contain transaction-control commands like
``start transaction``, ``commit`` or ``rollback``, regardless of the allowed
capabilities set. Because transaction-control commands in a script make it hard
to reason about atomicity, see also the `rejected alternative ideas
<#rejected-alternative-ideas>`_.

One exception is migration blocks - they are not transaction-control commands
when they are showing up within transactions, including the implicit
transactions wrapping scripts. Therefore, migration blocks are allowed in
scripts, but with one condition: the migration block must be complete. In other
words, you cannot leave a migration block undone in a script with only the
``start migration`` command without a matching ``commit migration`` or ``abort
migration`` command. However, you can have multiple migration blocks in one
script, even with other commands in between - all of them will be executed in a
single implicit transaction, of course when there is no outer explicit one.


Stateless Design
================

The main purpose of this RFC is to introduce a stateless design with the EdgeDB
protocol v1 - the server will no longer store any state attached to client
connections; instead it's more like the server will react in a request-response
pattern, while the client shall be responsible for maintaining the states and
tell the server in each request. In order to do that, a new "state" concept is
proposed.


The Concept of State
--------------------

State is defined as a payload of data that provides a context for EdgeQL
commands to be compiled and ran with. Currently, state is consist of module
aliases, session config and global values, for example:

.. code-block::

    {
        module := 'default',
        aliases := [ ( 'alias', 'module::target'), ... ],
        config := cfg::Config {
            session_idle_transaction_timeout: <duration>'0:05:00',
            query_execution_timeout: <duration>'0:00:00',
            allow_dml_in_functions: false,
            allow_bare_ddl: AlwaysAllow,
            apply_access_policies: true,
        },
        globals := { 'mod::key' := value, ... },
    }

State is created and sent by the client with ``Parse`` and ``Execute``
messages. The server compiles and executes the given command(s) in the context
of the given state. The command(s) in ``Execute`` may modify the state, the
server will then include an updated state in the ``CommandComplete`` message.


Describing the State
--------------------

State is serialized (and also utilized) in the same way as input arguments.
A new type descriptor "input shape descriptor" is proposed to describe state
data as "sparse objects". This is similar to the object shape descriptor [2]_,
only that sparse objects will skip serializing missing properties. For example,
the state itself is a sparse object, it will skip serializing ``aliases`` if no
module aliases are set, same for other properties. Also, the values of
``config`` and ``globals`` are also sparse objects.

The state descriptor depends on the database schema, especially the session
config and user-defined globals. In order for the clients to be able to encode
states, the server will send an extra ``StateDataDescription`` (`diff
<#statedatadescription>`_) message after the initial successful authentication
following the ``AuthenticationOK`` [3]_ message. The client should build codecs
accordingly and encode states on demand.

The database schema may change, by either the current client or other
concurrent clients, and that may affect the state schema. In order for all
clients to have the latest state descriptor, the server will send additional
``StateDataDescription`` messages:

1. If the current executed command modified the state schema, an additional
   ``StateDataDescription`` will be sent right before the ``CommandComplete``
   message.
2. If the state schema is modified concurrently, the client will receive a
   ``StateDataDescription`` message followed by an immediate
   ``StateMismatchError`` when trying to ``Parse`` or ``Execute`` with a state
   of outdated descriptor ID.

This is further explained in the following detailed message flow.


Query with State
----------------

.. raw:: html

  <img src="./EdgeDB-Execute.png" height="861px" align="right">

Two new fields are added in the ``Parse`` (`diff <#parse>`_), ``Execute``
(`diff <#execute-v1>`_) and ``CommandComplete`` (`diff <#commandcomplete>`_)
messages:

* ``state_typedesc_id``
* ``state_data``

Clients should set ``state_typedesc_id`` in ``Parse`` and ``Execute`` to the
``typedesc_id`` of the most-recently received ``StateDataDescription``, and set
``state_data`` with the current state encoded with the corresponding codecs. If
the given ``state_typedesc_id`` doesn't match the current schema, the server
will return a ``StateDataDescription`` message built with the latest schema,
immediately followed by a ``StateMismatchError`` in an ``ErrorResponse``.

``StateMismatchError`` is retryable, the client could simply retry encoding the
state with updated codes and send the same request again if encoding succeeds,
or simply raise an encoding error, or even try something smarter to convert
the state values into types compatible with the new codecs, depending on the
decision of the client implementation.

If the client chooses not to send a state (use default session config and
global values, ``default`` as current module, and no module aliases),
``state_typedesc_id`` should be set to NULL, and the server will ignore
``state_data`` and use default state directly. NULL ``state_typedesc_id`` will
never cause a ``StateMismatchError``.

On successful execution of EdgeQL commands, the server will return:

1. An optional ``CommandDataDescription`` if the output type ID mismatches;
2. Zero or more ``Data`` messages;
3. An optional ``StateDataDescription`` if the given command(s) modified the
   state **schema** (e.g. creating a new global);
4. A ``CommandComplete`` message.

The ``CommandComplete`` message may carry a valid state if the executed command
modified the state (e.g. setting a global value), or a NULL type ID if not.
When the ``state_typedesc_id`` in ``CommandComplete`` is NULL, its
``state_data`` must be ignored; or else, the client could choose to either
update the locally-maintained state, or simply ignore the state from server and
stick to user-specified state (this usually comes with disabling the
``SESSION_CONFIG`` capability [4]_, like the current official drivers do).

If not NULL, the ``state_typedesc_id`` in ``CommandComplete`` is guaranteed to
be the same as it is in ``StateDataDescription`` if any, or the same as it is
in ``Execute`` otherwise.


State and Transactions
----------------------

The state schema is transactional, because it is derived from the user schema.
This means, the state schema may be temporarily different in a transaction than
the current state schema of the database. The client should maintain a separate
state descriptor and corresponding codecs in transactions, and use them for
encoding states within transactions, until the end of the transaction.

If the state schema is modified within a transaction, the DDL ``Execute`` will
receive ``StateDataDescription`` normally, while the closing ``commit`` command
will not receive any ``StateDataDescription``, because ``commit`` doesn't
change the state schema in the current connection - it only made the state
schema globally visible. However, if the closing command is a ``rollback``,
there will be a ``StateDataDescription`` received. Savepoints apply too.

The state itself is also transactional. This is only visible when doing a roll
back: the ``CommandComplete`` message of a ``rollback`` will carry a state seen
in the ``Execute`` message of the corresponding ``start transaction`` command,
if the state in the ``Execute`` message of the ``rollback`` is not the same.
``rollback to savepoint`` applies too. Though, this behavior is only used in
the REPL.


Backwards Compatibility
=======================


Security Implications
=====================


Rejected Alternative Ideas
==========================


Appendix: Difference in Protocol
================================

This is a complete change set of protocol v1 comparing to v0.

Annotation
----------

.. code-block:: diff

    +struct Annotation {
    +  // Name of the annotation
    +  string          name;
    +
    +  // Value of the annotation (in JSON
    +  // format).
    +  string          value;
    +};

KeyValue
--------

.. code-block:: diff

    -struct Header {
    +struct KeyValue {
    -  // Header code (specific to the type of the
    +  // Key code (specific to the type of the
       // Message).
       uint16          code;

    -  // Header data.
    +  // Value data.
       bytes           value;
     };

Parse
-----

.. code-block:: diff

    -struct Prepare {
    +struct Parse {
       // Message type ('P').
       uint8           mtype = 0x50;

       // Length of message contents in bytes,
       // including self.
       uint32          message_length;

    -  // A set of message headers.
    -  uint16          num_headers;
    -  Header          headers[num_headers];
    +  // A set of annotations.
    +  uint16          num_annotations;
    +  Annotation      annotations[num_annotations];
    +
    +  // A bit mask of allowed capabilities.
    +  uint64<Capability> allowed_capabilities;
    +
    +  // A bit mask of query options.
    +  uint64<CompilationFlag> compilation_flags;
    +
    +  // Implicit LIMIT clause on returned sets.
    +  uint64          implicit_limit;

    -  // Data I/O format.
    -  uint8<IOFormat> io_format;
    +  // Data output format.
    +  uint8<OutputFormat> output_format;

       // Expected result cardinality.
       uint8<Cardinality> expected_cardinality;

    -  // Prepared statement name. Currently must
    -  // be empty.
    -  bytes           statement_name;
    -
       // Command text.
       string          command_text;
    +
    +  // State data descriptor ID.
    +  uuid            state_typedesc_id;
    +
    +  // Encoded state data.
    +  bytes           state_data;
     };

Capability
----------

.. code-block:: diff

    +enum Capability {
    +  MODIFICATIONS   = 0x1;
    +  SESSION_CONFIG  = 0x2;
    +  TRANSACTION     = 0x4;
    +  DDL             = 0x8;
    +  PERSISTENT_CONFIG = 0x10;
    +  ALL             = 0xffffffffffffffff;
    +};

CompilationFlag
---------------

.. code-block:: diff

    +enum CompilationFlag {
    +  INJECT_OUTPUT_TYPE_IDS = 0x1;
    +  INJECT_OUTPUT_TYPE_NAMES = 0x2;
    +  INJECT_OUTPUT_OBJECT_IDS = 0x4;
    +};

OutputFormat
------------

.. code-block:: diff

    -enum IOFormat {
    +enum OutputFormat {
       BINARY          = 0x62;
       JSON            = 0x6a;
       JSON_ELEMENTS   = 0x4a;
    +  NONE            = 0x6e;
     };

PrepareComplete
---------------

.. code-block:: diff

    -struct PrepareComplete {
    -  // Message type ('1').
    -  uint8           mtype = 0x31;
    -
    -  // Length of message contents in bytes,
    -  // including self.
    -  uint32          message_length;
    -
    -  // A set of message headers.
    -  uint16          num_headers;
    -  Header          headers[num_headers];
    -
    -  // Result cardinality.
    -  uint8<Cardinality> cardinality;
    -
    -  // Argument data descriptor ID.
    -  uuid            input_typedesc_id;
    -
    -  // Result data descriptor ID.
    -  uuid            output_typedesc_id;
    -};

DescribeStatement
-----------------

.. code-block:: diff

    -struct DescribeStatement {
    -  // Message type ('D').
    -  uint8           mtype = 0x44;
    -
    -  // Length of message contents in bytes,
    -  // including self.
    -  uint32          message_length;
    -
    -  // A set of message headers.
    -  uint16          num_headers;
    -  Header          headers[num_headers];
    -
    -  // Aspect to describe.
    -  uint8<DescribeAspect> aspect;
    -
    -  // The name of the statement.
    -  bytes           statement_name;
    -};

DescribeAspect
--------------

.. code-block:: diff

    -enum DescribeAspect {
    -  DATA_DESCRIPTION = 0x54;
    -};

StateDataDescription
--------------------

.. code-block:: diff

    +struct StateDataDescription {
    +  // Message type ('s').
    +  uint8           mtype = 0x73;
    +
    +  // Length of message contents in bytes,
    +  // including self.
    +  uint32          message_length;
    +
    +  // Updated state data descriptor ID.
    +  uuid            typedesc_id;
    +
    +  // State data descriptor.
    +  bytes           typedesc;
    +};

CommandDataDescription
----------------------

.. code-block:: diff

     struct CommandDataDescription {
       // Message type ('T').
       uint8           mtype = 0x54;

       // Length of message contents in bytes,
       // including self.
       uint32          message_length;

    -  // A set of message headers.
    -  uint16          num_headers;
    -  Header          headers[num_headers];
    +  // A set of annotations.
    +  uint16          num_annotations;
    +  Annotation      annotations[num_annotations];
    +
    +  // A bit mask of allowed capabilities.
    +  uint64<Capability> capabilities;

       // Actual result cardinality.
       uint8<Cardinality> result_cardinality;

       // Argument data descriptor ID.
       uuid            input_typedesc_id;

       // Argument data descriptor.
       bytes           input_typedesc;

       // Output data descriptor ID.
       uuid            output_typedesc_id;

       // Output data descriptor.
       bytes           output_typedesc;
     };

Execute (v0)
------------

.. code-block:: diff

    -struct Execute {
    -  // Message type ('E').
    -  uint8           mtype = 0x45;
    -
    -  // Length of message contents in bytes,
    -  // including self.
    -  uint32          message_length;
    -
    -  // A set of message headers.
    -  uint16          num_headers;
    -  Header          headers[num_headers];
    -
    -  // Prepared statement name.
    -  bytes           statement_name;
    -
    -  // Encoded argument data.
    -  bytes           arguments;
    -};

Execute (v1)
------------

.. code-block:: diff

    -struct OptimisticExecute {
    +struct Execute {
       // Message type ('O').
       uint8           mtype = 0x4f;

       // Length of message contents in bytes,
       // including self.
       uint32          message_length;

    -  // A set of message headers.
    -  uint16          num_headers;
    -  Header          headers[num_headers];
    +  // A set of annotations.
    +  uint16          num_annotations;
    +  Annotation      annotations[num_annotations];
    +
    +  // A bit mask of allowed capabilities.
    +  uint64<Capability> allowed_capabilities;
    +
    +  // A bit mask of query options.
    +  uint64<CompilationFlag> compilation_flags;
    +
    +  // Implicit LIMIT clause on returned sets.
    +  uint64          implicit_limit;

    -  // Data I/O format.
    -  uint8<IOFormat> io_format;
    +  // Data output format.
    +  uint8<OutputFormat> output_format;

       // Expected result cardinality.
       uint8<Cardinality> expected_cardinality;

       // Command text.
       string          command_text;

    +  // State data descriptor ID.
    +  uuid            state_typedesc_id;
    +
    +  // Encoded state data.
    +  bytes           state_data;
    +
       // Argument data descriptor ID.
       uuid            input_typedesc_id;

       // Output data descriptor ID.
       uuid            output_typedesc_id;

       // Encoded argument data.
       bytes           arguments;
     };

ExecuteScript
-------------

.. code-block:: diff

    -struct ExecuteScript {
    -  // Message type ('Q').
    -  uint8           mtype = 0x51;
    -
    -  // Length of message contents in bytes,
    -  // including self.
    -  uint32          message_length;
    -
    -  // A set of message headers.
    -  uint16          num_headers;
    -  Header          headers[num_headers];
    -
    -  // EdgeQL script text to execute.
    -  string          script;
    -};

Flush
-----

.. code-block:: diff

    -struct Flush {
    -  // Message type ('H').
    -  uint8           mtype = 0x48;
    -
    -  // Length of message contents in bytes,
    -  // including self.
    -  uint32          message_length;
    -};

CommandComplete
---------------

.. code-block:: diff

     struct CommandComplete {
       // Message type ('C').
       uint8           mtype = 0x43;

       // Length of message contents in bytes,
       // including self.
       uint32          message_length;

    -  // A set of message headers.
    -  uint16          num_headers;
    -  Header          headers[num_headers];
    +  // A set of annotations.
    +  uint16          num_annotations;
    +  Annotation      annotations[num_annotations];
    +
    +  // A bit mask of allowed capabilities.
    +  uint64<Capability> capabilities;

       // Command status.
       string          status;
    +
    +  // State data descriptor ID.
    +  uuid            state_typedesc_id;
    +
    +  // Encoded state data.
    +  bytes           state_data;
     };

ErrorResponse
-------------

.. code-block:: diff

     struct ErrorResponse {
       // Message type ('E').
       uint8           mtype = 0x45;

       // Length of message contents in bytes,
       // including self.
       uint32          message_length;

       // Message severity.
       uint8<ErrorSeverity> severity;

       // Message code.
       uint32          error_code;

       // Error message.
       string          message;

       // Error attributes.
       uint16          num_attributes;
    -  Header          attributes[num_attributes];
    +  KeyValue        attributes[num_attributes];
     };

LogMessage
----------

.. code-block:: diff

     struct LogMessage {
       // Message type ('L').
       uint8           mtype = 0x4c;

       // Length of message contents in bytes,
       // including self.
       uint32          message_length;

       // Message severity.
       uint8<MessageSeverity> severity;

       // Message code.
       uint32          code;

       // Message text.
       string          text;

    -  // Message attributes.
    -  uint16          num_attributes;
    -  Header          attributes[num_attributes];
    +  // Message annotations.
    +  uint16          num_annotations;
    +  Annotation      annotations[num_annotations];
     };

ReadyForCommand
---------------

.. code-block:: diff

     struct ReadyForCommand {
       // Message type ('Z').
       uint8           mtype = 0x5a;

       // Length of message contents in bytes,
       // including self.
       uint32          message_length;

    -  // A set of message headers.
    -  uint16          num_headers;
    -  Header          headers[num_headers];
    +  // A set of annotations.
    +  uint16          num_annotations;
    +  Annotation      annotations[num_annotations];

       // Transaction state.
       uint8<TransactionState> transaction_state;
     };

RestoreReady
------------

.. code-block:: diff

     struct RestoreReady {
       // Message type ('+').
       uint8           mtype = 0x2b;

       // Length of message contents in bytes,
       // including self.
       uint32          message_length;

    -  // A set of message headers.
    -  uint16          num_headers;
    -  Header          headers[num_headers];
    +  // A set of annotations.
    +  uint16          num_annotations;
    +  Annotation      annotations[num_annotations];

       // Number of parallel jobs for restore,
       // currently always "1"
       uint16          jobs;
     };

Dump
----

.. code-block:: diff

     struct Dump {
       // Message type ('>').
       uint8           mtype = 0x3e;

       // Length of message contents in bytes,
       // including self.
       uint32          message_length;

    -  // A set of message headers.
    -  uint16          num_headers;
    -  Header          headers[num_headers];
    +  // A set of annotations.
    +  uint16          num_annotations;
    +  Annotation      annotations[num_annotations];
     };

Restore
-------

.. code-block:: diff

     struct Restore {
       // Message type ('<').
       uint8           mtype = 0x3c;

       // Length of message contents in bytes,
       // including self.
       uint32          message_length;

    -  // A set of message headers.
    -  uint16          num_headers;
    -  Header          headers[num_headers];
    +  // A set of key-value pairs.
    +  uint16          num_attributes;
    +  KeyValue        attributes[num_attributes];

       // Number of parallel jobs for restore
       // (only "1" is supported)
       uint16          jobs;

       // Original DumpHeader packet data
       // excluding mtype and message_length
       bytes           header_data;
     };

DumpHeader
----------

.. code-block:: diff

     struct DumpHeader {
       // Message type ('@').
       uint8           mtype = 0x40;

       // Length of message contents in bytes,
       // including self.
       uint32          message_length;

    -  // A set of message headers.
    -  uint16          num_headers;
    -  Header          headers[num_headers];
    +  // A set of key-value pairs.
    +  uint16          num_attributes;
    +  KeyValue        attributes[num_attributes];

       // Major version of EdgeDB.
       uint16          major_ver;

       // Minor version of EdgeDB.
       uint16          minor_ver;

       // Schema.
       string          schema_ddl;

       // Type identifiers.
       uint32          num_types;
       DumpTypeInfo    types[num_types];

       // Object descriptors.
       uint32          num_descriptors;
       DumpObjectDesc  descriptors[num_descriptors];
     };

DumpBlock
---------

.. code-block:: diff

     struct DumpBlock {
       // Message type ('=').
       uint8           mtype = 0x3d;

       // Length of message contents in bytes,
       // including self.
       uint32          message_length;

    -  // A set of message headers.
    -  uint16          num_headers;
    -  Header          headers[num_headers];
    +  // A set of key-value pairs.
    +  uint16          num_attributes;
    +  KeyValue        attributes[num_attributes];
     };

ProtocolExtension
-----------------

.. code-block:: diff

     struct ProtocolExtension {
       // Extension name.
       string          name;

    -  // A set of extension headers.
    -  uint16          num_headers;
    -  Header          headers[num_headers];
    +  // A set of extension annotaions.
    +  uint16          num_annotations;
    +  Annotation      annotations[num_annotations];
     };


.. [1] https://github.com/edgedb/rfcs/blob/master/text/1010-global-vars.rst
.. [2] https://www.edgedb.com/docs/reference/protocol/typedesc#object-shape-descriptor
.. [3] https://www.edgedb.com/docs/reference/protocol/messages#ref-protocol-msg-auth-ok
.. [4] https://github.com/edgedb/rfcs/blob/master/text/1004-transactions-api.rst#connection-configuration-methods
