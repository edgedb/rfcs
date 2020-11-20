::

    Status: Draft
    Type: Feature
    Created: 2020-09-03
    Authors: Paul Colomiets <paul@colomiets.name>
    RFC PR: `edgedb/rfcs#0004 <https://github.com/edgedb/rfcs/pull/19>`_

==============================
RFC 0004: Robust Client API
==============================


Abstract
========

This RFC describes client API that is:

1. Resilient against network errors and server failover (i.e. when server
   changes IP address) both within a single transactions and for
   non-transactional queries.
2. Retries transactions in a number of error conditions such as
   deadlock and concurrent update error.
3. Provides uniform access to the underlying querying API from pool,
   connection and transaction objects.


Motivation
==========

The major motivation for the API change is having robust language
bindings that:

1. Tolerate server restart and IP address migration
2. Robust towards intermittent network errors or dropping via idle
   timeout
3. Ready for replica support
4. And (mostly as a consequence of (1)) don't crash when application and
server are started at the same time

Additional details on the transaction API enhancements follow.


Improvements to Transaction API
-------------------------------

We want to encourage users using connection pools rather than
connections directly. Older API requires connection to acquire a
connection first:

.. code-block:: python

   with pool.acquire() as connection:
       with connection.trasaction():
           pass

With the new API we don't need to acquire a connection separately. This
not only simplifies the code, but also allows reconnecting on network
errors.

`More discussion on this topic <https://github.com/edgedb/edgedb/discussions/1708>`_

This part of API change is also motivated by the fact that PostgreSQL
sometimes can't apply concurrent transactions and errors out with::

    Error: could not serialize access due to concurrent update

It's expected that transaction may be repeated and be successful when
repeated. Common case when this happens is using ``INSERT ... UNLESS
CONFLICT ..`` statement in EdgeDB. See `Repeating-EdgeQL<#repeating-edgeql>`_
section for an explanation of why Postgres and EdgeDB can't repeat transactions
automatically.

EdgeDB supports only ``REPEATABLE READ`` and ``SERIALIZABLE`` isolation
levels.  That means errors like the above may happen more often than
with the default ``READ COMMITTED`` isolation level in Postgres.

As we implement retrying connection on "concurrent update" we also want
to handle:

1. Transaction deadlocks
2. Connection errors anywhere between ``BEGIN TRANSACTION`` and before
   ``COMMIT`` (we can't reliably know whether commit itself worked if we sent
   last ``COMMIT`` and got no response)

`More discussion on retrying <https://github.com/edgedb/edgedb/discussions/1738>`_


Specification
=============


Connections and Pools
---------------------

Instead of exposing a TCP connection to the user we use a set of wrapper
classes:

.. code-block:: python

    class Connection:
        _impl: _SingletonPool
        _desired_config: SessionConfig

    class ReadOnlyConnection:
        _impl: _SingletonPool
        _desired_config: SessionConfig

    class _SingletonPool
        _impl: RawConnection

    class RawConnection:
        _protocol: BlockingIOProtocol
        _current_config: SessionConfig

    def connect() -> Connection: ...

Similarly we update connection pool classes API:

.. code-block:: python

    class Pool(ReadOnlyPool):
        _impl: _PoolImpl
        _desired_config: SessionConfig

    class ReadOnlyPool:
        _impl: _SingletonPool
        _desired_config: SessionConfig

    class _PoolImpl:
        _connections: deque[RawConnection]  # Simplified

Both ``Pool`` and ``Connection`` have:

1. ``execute``, ``query`, ``query_one`` set of query functions
   (implement ``Executor`` abstract class defined below). All the
   methods reconnect if called on broken connection. Read-only queries
   broken in the middle of the query are retried.
2. ``retry`` method (described below)
3. ``try_transaction`` method for more fine-grained control over
   transactions (this is older ``transaction`` method, see below)

``ReadOnly*`` counterparts differ from non-read-only ones in two
important ways:

1. They send read-only flag to the database server, which can reject queries
   that modify data. And *may* use replica connection.
2. Works as type-checker hint that function that received a connection
   does no modifications to the data.

``RawConnection`` class is exposed for users who want to do their own
session config management stuff, connection pooling or control of
reconnection.

Note: this section contains only sync Python example, async Python and
JavaScript bindings undergo changes similar enough that we don't think
it makes sense to put them here explicitly.


Connection Configuration Methods
--------------------------------

The following things are modified by a method that returns a distinct
object of the same type and same underlying network connections but
distinct config (i.e. ``Connection`` retains the same
``_SingletonPool`` and ``Pool`` retains the same ``_PoolImpl``):

* Asking for a read-only access
* Setting session configuration
* Transaction options

Here are method signatures:

.. code-block:: python

    class Connection:
        ...
        def read_only(self,
            primary: bool = false,
            allow_upgrade: bool = false,  # allows with_modifications
        ) -> ReadOnlyConnection: ...
        def with_modifications(self) -> Connection: ...
        def with_session_config(self, **config) -> Connection: ...
        def with_transaction_options(self, isolation: ...) -> Connection: ...
        def with_retry_options(self, attempts: int = 3, ...) -> Connection: ...

    class Pool:
        ...
        def read_only(self,
            primary: bool = false,
            allow_upgrade: bool = false,  # allows with_modifications
        ) -> ReadOnlyConnection: ...
        def with_modifications(self) -> Connection: ...
        def with_session_config(self, **config) -> Pool: ...
        def with_transaction_options(self, isolation: ...) -> Pool: ...
        def with_retry_options(self, attempts: int = 3, ...) -> Pool: ...

The ``ReadOnlyPool`` and ``ReadOnlyConnection`` get the same methods
(including ``read_only`` method itself, which is no-op).

After modification, connection/pool object can be used interchangeably:

.. code-block:: python

    conn = edgedb.connect()
    read_only_conn = conn.read_only()
    conn.execute("INSERT User { ... }", ...)
    print(read_only_conn.query("SELECT User"))
    read_only_conn.execute("INSERT User { .. }", ...)  # throws an error
    conn.read_only(allow_upgrade=True) \
        .with_modifications().execute("INSERT User { ... }", ...)  # good

Or session config example:

.. code-block:: python

    conn = edgedb.connect()
    conn2 = conn.with_session_config(param="value")
    conn.execute("INSERT User { ... }", ...)  # without config
    print(conn2.query("SELECT User"))  # with config
    conn.execute("INSERT User { .. }", ...)  # without config again

There are different ways of these options are actually applied:

1. Read-only access is marked as a header to query compilation message
   (i.e. metadata for compiling specific query)
2. Session configuration is currently applied by sending extra queries
   prior to the query itself and persist in that connection until
   reset or the connection is closed
3. Transactions options are client-side and don't require any change in
   either connection or network protocol

But all of that are implementation details. We provide uniform interface
for all these options and future protocols or connection pools may use
a different underlying implementation. E.g. in the future:

1. Multiple session configuration options may be gathered and applied in
   one batch
2. Connection pool may pick a separate connection to a read-only replica
   for a read-only query.
3. Session configuration is cached in the connection state and is not
   touched if the same configuration is used.
4. Connection pool may cache connections with the same session state so
   we don't need to apply session state on each request.
5. Eventually we may allow some session configuration to be applied on
   a per-request basis or vice versa make the read-only flag the session
   configuration.

All of these optimizations would be seamless for the API users,
who should trust the language bindings to do the optimal thing.


Transactions API
----------------

General idea of the feature:

1. Introduce a block of user's code that may be repeated
2. Pass a transaction object to the block
3. Repeat the block within a new transaction until succeeds or attempt
   number is exhausted

The block is introduced by ``connection_pool.retry``, the exact code
is different for different languages because of language limitations
and convetions.

Normal transactions that aren't retried are executed with
``try_transaction`` method.

The ``retry`` function configured by the number of attempts and backoff
function. This is configured by calling ``with_retry_options`` prior
to the ``retry`` function. Former can be called prior to setting global
connection pool variable to achieve global setting, or can be called at
any time to achieve needed granularity for this setting.

Current attempt number N is global. Which means if the last error is a
deadlock and N is greater than the number of attempts on a deadlock we
stop retrying and return error (even if previous error was a network
error).

Backoff function by default is ``2^N * 100`` plus random number in range
``0..100`` milliseconds. Where first retry (second attempt) has ``N=1``.
Technically:

* In JavaScript: ``n => (2**n) * 100 + Math.random()*100``
* In Python: ``lambda n: (2**n) * 0.1 + randrange(100)*0.1`` (seconds)
* In Rust: ``|n| Duration::from_millis(2u64.pow(n)*100 + thread_rng().gen_range(0,100)``

Backoff is randomized so that if there was a coordinated failure (i.e.
server restart which triggers all current transactions to retry)
transactions don't overwhelm a database by reconnecting simultaneously.
If backoff function is adjusted it's recommeded to keep some
randomization anyway.


TypeScript Transactions API
```````````````````````````

**Introduce** two methods on a connection pool and connection:

.. code-block:: typescript

    type TransactionBlock = (Transaction) => Promise<T>;

    interface Pool {
        retry<T>(block: TransactionBlock): T;
        try_transaction<T>(action: TransactionBlock): T;
    }
    class Connection {
        retry<T>(block: TransactionBlock): T;
        try_transaction<T>(action: TransactionBlock): T;
    }

Raw connection has only ``try_transaction`` method:

    class Connection {
        raw_transaction<T>(action: TransactionBlock, options?: TransactionOptions): T;
    }

Note: transaction options are passed directly to ``try_transaction`` as
it doesn't have ``with_transaction_options`` method.

Introduce interface for making queries:

.. code-block:: typescript

    interface ReadOnlyExecutor {
        async query(query: string, args: QueryArgs = null): Promise<Set>;
        async queryOne(query: string, args: QueryArgs = null): Promise<any>;
        async queryJSON(query: string, args: QueryArgs = null): Promise<string>;
        async queryOneJSON(query: string, args: QueryArgs = null): Promise<string>;
        async execute(query: string): Promise<void>;
    }
    interface Executor extends ReadOnlyExecutor {}

And implement interface by respective classes:

.. code-block:: typescript

   class Transaction implements Executor {/*...*/}
   class Connection implements Executor {/*...*/}
   interface Pool extends Executor {/*...*/}
   class ReadOnlyTransaction implements ReadOnlyExecutor {/*...*/}
   class ReadOnlyConnection implements ReadOnlyExecutor {/*...*/}
   interface ReadOnlyPool extends ReadOnlyExecutor {/*...*/}

While removing inherent methods with the same name.

Note: while ``Connection.try_transaction`` block is active,
``Executor`` methods are disabled on the connection object itself
(i.e. they throw ``TransactionActiveError``).

Example of the recommended transaction API:

.. code-block:: typescript

    await pool.retry(tx => {
        let val = await tx.fetch("...")
        await tx.execute("...", process_value(val))
    })

Example using ``try_transaction``:

    await pool.try_transaction(tx => {
        let val = await tx.fetch("...")
        await tx.execute("...", process_value(val))
    })

Note the new API is very similar to older ``transaction`` except the
queries are executed using transaction object, instead of connection
itself.

**Deprecate** ``transaction()`` method::

    `connection.transaction(f)` is deprecated. Use `pool.retry(f)`
    (preferred) or `connection.try_transaction(f)`.

The ``RetryOptions`` signature:

.. code-block:: typescript

    type BackoffFn = (attempt: number) => number;
    enum AttempsOption {
        All,
        NetworkError,
        ConcurrentUpdate,
        Deadlock,
    }
    class RetryOptions {
        constructor({
            attempts: number = 3,
        });
        attempts(
            which: AttemptsOption,
            attempts: number,
            backoff_ms?: BackoffFn,
        ): RetryOptions;
    }


Exceptions API
''''''''''''''

Error hierarchy is amended by introducing ``TransientError``, ``NetworkError``
and ``EarlyNetworkError`` with the following relationships:

.. code-block:: typescript

    class TransientError extends TransactionError {}
    class TransactionDeadlockError extends TransientError {}
    class TransactionSerializationError extends TransientError {}
    class NetworkError extends ClientError {}
    class EarlyNetworkError extends NetworkError {}

All network error within connection should be converted into
``EarlyNetworkError`` or ``NetworkError``. Former is used in context
where we catch network error before sending a request.

``TransactionActiveError`` is introduced to signal that
queries can't be executed on the connection object:

.. code-block:: typescript

    class TransactionActiveError extends InterfaceError {}


Python Transactions API
```````````````````````

For python API plain ``with`` doesn't work any more, so we introduce a
loop and with block. See example below.

Pool methods for creating a transaction:

.. code-block:: python

   class AsyncIOPool:
       def retry() -> AsyncIterable[AsyncContextManager[AsyncIOTransaction]]: ...
       async def try_transaction(*,
           isolation: str = None,
           read_only: bool = None,
           deferrable: bool = None,
       ) -> AsyncContextManager[AsyncIOTransaction]: ...

   class AsyncIOConnection:
       def retry() -> AsyncIterable[AsyncContextManager[AsyncIOTransaction]]: ...
       async def try_transaction(*,
           isolation: str = None,
           read_only: bool = None,
           deferrable: bool = None,
       ) -> AsyncContextManager[AsyncIOTransaction]: ...

   class Pool:
       def retry() -> Iterable[ContextManager[Transaction]]: ...
       def try_transaction(
           isolation: str = None,
           read_only: bool = None,
           deferrable: bool = None,
       ) -> ContextManager[Transaction]: ...

   class Connection:
       def retry() -> Iterable[ContextManager[Transaction]]: ...
       def raw_transaction(
           isolation: str = None,
           read_only: bool = None,
           deferrable: bool = None,
       ) -> ContextManager[Transaction]: ...


Example usage of ``retry`` on async pool:

.. code-block:: python

    async for tx in db.retry():
      async with tx:
        let val = await tx.fetch("...")
        await tx.execute("...", process_value(val))

Example usage of ``retry`` on sync pool:

.. code-block:: python

    for tx in db.retry():
      with tx:
        let val = tx.fetch("...")
        tx.execute("...", process_value(val))

This works roughly as follows:

1. ``retry()`` returns an (async-)iterator which has no methods.
2. Every yielded element is a transaction object, strongly referencing
   the iterator that created it internally.
3. If the code in the ``async with`` / ```with`` block succeeds,
   the transaction object messages its iterator to stop iteration.


Example of ``try_transaction``:

.. code-block:: python

      async with db.try_transaction() as tx:
        let val = await tx.fetch("...")
        await tx.execute("...", process_value(val))

Note the new API is very similar to older ``transaction`` except the
queries are executed using transaction object, instead of connection
itself.

**Deprecate** old transaction API::

    DeprecationWarning: `connection.transaction()` is deprecated. Use
    `pool.retry()` (preferred) or `connection.try_transaction()`.

Add ``RetryOptions`` class:

.. code-block:: Python

    type DelayFn = (attempt: number) => number;

    class AttempsOption(enum.Enum):
        ALL = "ALL"
        NETWORK_ERROR = "NETWORK_ERROR"
        CONCURRENT_UPDATE = "CONCURRENT_UPDATE"
        DEADLOCK = "DEADLOCK"

    class RetryOptions:
        def __init__(self, attempts=3): ...

        def attempts(
            which: AttemptsOption,
            attempts: int,
            backoff_ms: Callable[[int], [float]],
        ): RetryOptions: ...
    }

Introduce the abstract classes for queries:

.. code-block:: python

    class AsyncReadOnlyExecutor(abc.AbstraceBaseClass):
        async def execute(self, query): ...
        async def query(self, query: str, *args, **kwargs) -> datatypes.Set: ...
        async def query_one(self, query: str, *args, **kwargs) -> typing.Any: ...
        async def query_json(self, query: str, *args, **kwargs) -> str: ...
        async def query_one_json(self, query: str, *args, **kwargs) -> str: ...

    class AsyncExecutor(AsyncReadOnlyExecutor):
        pass

    class ReadOnlyExecutor(abc.AbstractClass):
        def query(self, query: str, *args, **kwargs) -> datatypes.Set: ...
        def query_one(self, query: str, *args, **kwargs) -> typing.Any: ...
        def query_json(self, query: str, *args, **kwargs) -> str: ...
        def query_one_json(self, query: str, *args, **kwargs) -> str: ...
        def execute(self, query: str) -> None: ...

    class ReadOnlyExecutor(ReadOnlyExecutor):
        pass

Note: while ``Connection.try_transaction`` block is active,
``Executor`` methods are disabled on the connection object itself
(i.e. they throw ``TransactionActiveError``).

These base classes should be implemented by respective classes:

.. code-block:: python

    class AsyncIOTransaction(AsyncExecutor): ...
    class AsyncIOConnection(AsyncExecutor): ...
    class AsyncIOPool(AsyncExecutor): ...
    class Transaction(Executor): ...
    class Connection(Executor): ...
    class Pool(Executor): ...
    class AsyncIOReadOnlyTransaction(AsyncReadOnlyExecutor): ...
    class AsyncIOReadOnlyConnection(AsyncReadOnlyExecutor): ...
    class AsyncIOReadOnlyPool(AsyncReadOnlyExecutor): ...
    class ReadOnlyTransaction(ReadOnlyExecutor): ...
    class ReadOnlyConnection(ReadOnlyExecutor): ...
    class ReadOnlyPool(ReadOnlyExecutor): ...


Exceptions API
''''''''''''''

Error hierarchy is amended by introducing ``TransientError``,
``NetworkError`` and ``EarlyNetworkError`` with the following
relationships:

.. code-block:: python

    class TransientError(TransactionError): ...
    class TransactionDeadlockError(TransientError): ...
    class TransactionSerializationError(TransientError): ...
    class NetworkError(ClientError): ...
    class EarlyNetworkError(NetworkError): ...

All network error within connection should be converted into
``EarlyNetworkError`` or ``NetworkError``. Former is used in context
where we catch network error before sending a request.

Additionally ``TransactionActiveError`` is introduced to signal that
queries can't be executed on the connection object (i.e. when
connection object is "borrowed" for the duration of the transaction):

.. code-block:: python

    class TransactionActiveError(InterfaceError):
        _code = 0x_FF_02_01_04


Server Availability Timeout
---------------------------

Previously, when TCP connect gets "connection refused" error or when
timeout happens on handshake both connection and pool API would crash
the application. This is weird when initially starting an application
cluster simultaneously with the database, or just when starting a
projece locally using ``docker-compose up``.

This RFC introduces a connection parameter:

.. code-block:: python

    edgedb.connect('inst1', wait_until_available_sec=30)
    await edgedb.async_connect('inst1', wait_until_available_sec=30)
    await edgedb.async_connection_pool('inst1', wait_until_available_sec=30)

.. code-block:: typescript

    await edgedb.connect('inst1', {wait_until_available_sec: 30})
    await edgedb.createPool('inst1', {
        connectOptions: {
            wait_until_available_sec: 30,
        }
    })

The semantics are the following:

1. On initial connect or pool creation, function blocks for up to the
   specified number of seconds until connection is established (at least
   single one for connection pool).
2. If first connection could not be established for this timeout, error
   is thrown that holds the reason of the last failure.
3. On the subsequent reconnects this timeout is also obeyed (i.e.
   connection might try to reconnect multiple times) and error is thrown
   after the specified timeout as a result of the operation that
   requires connection (i.e. on query or transaction start).
4. For ``retry`` we bail out if this timeout is reached during any
   single reconnect. We don't retry immediately after database is marked
   as unavailable.
5. Subsequent queries or transactions after failure will retry for the
   specified timeout each time.

Only the following conditions are treated as eligible for reconnect:

1. Name resolution failed
2. File not found (Unix socket not bound yet)
3. Connection reset, connection aborted, connection refused (server is
   restarting or not ready yet)
4. Timeout happened during connect or authentication

All other errors are propagated immediately.

This timeout is different from ``connect_timeout``. Connect timeout
determines how long individual connect attempt may take. And
``wait_until_available`` determines how many such attempts could be
made (i.e. how many ones fits the time frame).


EdgeDB Changes
--------------

To support features above we add two headers to EdgeDB queries:

1. For PrepareComplete_, CommandComplete_ server-side messages:
   ``SERVER_OPT_HAS_FEATURES``
2. For Prepare_, OptimisticExecute_, Execute_, ExecuteScript_
   client-side messages: ``QUERY_OPT_ALLOW_FEATURES``

Both contain the set of strings joined by comma:

1. `modification` -- query is not read-only
2. `session` -- query contains session config change
3. `config` -- server or database config change
4. `ddl` -- query contains DDL
5. `transaction` -- query contains start/commit/rollback of transaction
    or savepoint manipulation

In case of ``SERVER_OPT_HAS_FEATURES`` it describes what is actually
contained in the query. And in case of ``QUERY_OPT_ALLOW_FEATURES``
client can specify what of these things are allowed in this query.

Read-only queries are always allowed. When ``QUERY_OPT_ALLOW_FEATURES``
is omitted any query is allowed (default). With the empty string only
read-only queries are allowed.

``SERVER_OPT_HAS_FEATURES`` is an empty string for read-only queries (the
field is present) as it indicates that query has been analyzed.

The ``SERVER_OPT_HAS_FEATURES`` is needed for the following tasks:

1. Retry standalone (non-transactional) read-only queries
2. Warn when features are used in inapropriate context (warn when
   session modification queries are sent on non-raw connection, which
   means they can be lost at any point due to reconnect)

By default:

1. Pool supports all except ``transaction,session``
2. Read-only pool and read-only connection support none
3. Connection warns on ``session`` and ``transaction``
4. Connection got from pool errors on ``session`` and ``transaction``
5. Everything is allowed in ``RawConnection``

Note: session settings and transactions should be activated using
special methods ``with_session_config``, ``try_transaction`` and
``retry``, rather than by using ``execute(...)`` or ``query(...)`` in
the former case they are allowed internally. And this should be
indicated in the respective error messages.


.. _PrepareComplete: https://www.edgedb.com/docs/internals/protocol/messages#preparecomplete
.. _CommandComplete: https://www.edgedb.com/docs/internals/protocol/messages#commandcomplete
.. _Prepare: https://www.edgedb.com/docs/internals/protocol/messages#ref-protocol-msg-prepare
.. _OptimisticExecute: https://www.edgedb.com/docs/internals/protocol/messages#ref-protocol-msg-optimistic-execute
.. _Execute: https://www.edgedb.com/docs/internals/protocol/messages#ref-protocol-msg-execute
.. _ExecuteScript: https://www.edgedb.com/docs/internals/protocol/messages#ref-protocol-msg-execute-script


Future Work
===========


More Configuration
------------------

Setting tracing/debugging metadata for queries may be implemented
in the same way:

.. code-block:: python

    def handle(request, db):
        db = db.with_metadata(
            uri=request.uri,
            username=request.session.get('username'),
        )
        handle_user_request(request, db)

And this can be done on transaction level too:

.. code-block:: python

    async for tx in db.retry():
      async with tx:
        tx = tx.with_metadata(
            uri=request.uri,
            username=request.session.get('username'),
        )
        tx.query("...")


More Read-Only Options
----------------------

Replica config may be specified when configuring a read-only connection:

.. code-block:: python

   def read_only(self, primary=False, max_replica_lag=10): ...

Perhaps this should be encapsulated into replica options:

.. code-block:: python

   def read_only(self, primary=False, replicas: ReplicaOptions): ...


Learning Curve
==============

The ``retry`` method complicates learning curve curve, but:

1. This is already a problem in Postgres, there a lot of people who
   ignore the issue for pet projects and a lot of startups having
   "normal" rate or 500 errors at any point of time, while it's
   preventable.
2. Repeating transactions would make less incentive to keep transactions
   open for a long time (e.g. while accessing slow network resources
   like external API), which is a problem of itself.
3. Even if we never have failed concurrent updates we would want
   seamless reconnect on connection failures (i.e. server restart,
   primary/replica change, etc.)
4. To make learning curves shorter I think we should intentionally
   inject failures. This is needed so that users quickly find out that
   side effects of their transactions are in effect several times.

So while increasing learning curve, we fix heisenbugs and simplify
operations.

Failure Injection
-----------------

The following is proposed to be done by default:

Collect statitics of how many queries are executed in the previous
second and on each new request trigger a failure with the probability of
``1/n`` where ``n`` is the number of requests in the previous second. We
still need to figure out whether ``n`` counts queries, transactions, or
mutable queries/transactions (and have a list of exceptions, perhaps:
dump+restore+migrations).

The idea is that there will be ~1 failure per second. So on local
instance when testing manually it would hit almost every request (which
is fine as repeating them shouldn't be prohibitively costly). But under
a huge load of thousands requests per second, one retry per second
doesn't influence anything so even for production and/or benchmarks this
is fine.

I think it should be disabled by an explicit command-line argument like
``--disable-failure-injection`` but might be tweaked with configuration
settings?


Alternatives
============

Alternative Names
-----------------

For ``retry`` method:

1. ``db.atomic(t => t.execute(..))``
2. ``db.mutate(transaction => transaction.execute(..))``
3. ``db.apply(transaction => transaction.execute(..))``
4. ``db.unit_of_work(t => t.execute(..))``
5. ``db.block(t => t.execute(..))``
6. ``db.try(transaction => transaction.execute(..))``
7. ``db.retry_transaction(t => t.execute(..))``

For ``try_transaction`` method:

1. ``with db.raw_transaction() as t:``
2. ``with db.plain_transaction() as t:``
3. ``with db.unreliable_transaction() as t:``
4. ``with db.single_transaction() as t:``

``read_only`` could be ``with_read_only`` to support convention. But it
looks like it's clear enough.

``with_modifications`` could be ``read_write`` or ``with_read_write`` or
``clear_read_only``. Or it could be ``read_only(false)``.


Alternative Python API
----------------------

For python API we could support funcional API:

.. code-block:: python

    def handler(req, db: edgedb.Pool):
        await db.retry(my_tx, req)

    async def my_tx(transaction, req):
        let val = await transaction.fetch("...")
        await transaction.execute("...", process_value(val))

And/or decorator API:

.. code-block:: python

    def handler(req, db: edgedb.Pool):
        do_something(req)

        @db.retry()
        def my_tx(transaction):
            let val = transaction.fetch("...")
            transaction.execute("...", process_value(val))

        return render_page(val)

Function call API has the issue of variables are propagated either
are parameters to ``retry`` function itself or force users using
``partial``.

While decorator API doesn't work for async code or at least requires
extra ``await ...`` line.


Alternative Exceptions API
--------------------------

Instead of classes we might have `is_transient`, `is_network_failure`,
`is_early_network_error` method on `EdgeDBError` class. This would allow
adding more errors later without changing class hierarchy.


Repeating EdgeQL
----------------

One may think that why we can collect all the queries in the client (or
even at the server) and retry.

The problem is that sometimes writes depend on previous reads:

.. code-block:: python

    user = await db.query_row("SELECT User {balance}")
    prod = await db.query_row("SELECT Product {price}")
    if user.balance > prod.price:
        await db.query(
            "SELECT User { balance := .balance - <decimal>$price }",
            price=prod.price)
    else:
        return "not enough money"

If it happens that two transactions updating money will happen
concurrently, it's possible that user have negative balance, even while
code suggests it can't (when retrying transaction we don't check ``if``
again). But if we retry the whole block of code it will work correctly.


Enabling Retries in Connection Options
--------------------------------------

At least for JavaScript we could keep old API, and then use connection
configuration to introduce retries:

.. code-block:: javascript

    let conn = connect('mydb', {transactionRetries: 5});
    await conn.transaction(t => {
      // ...
    })

There are few problems of this approach:

1. This is **not composable**: some sub-application might want to rely
   on repeating transaction, but no way to ensure that. Another
   sub-application might repeat manually an extra repeating automatically
   might make transaction slower and introduce unexpected repeatable side
   effects.
2. This doesn't help in case of pythonic `with db.transaction()` as we
   allow now.
3. If we're advising `transaction` on connection object, reconnecting on
   network failures would be an issue


Keep `transaction` as is but Add a Helper for Retrying
------------------------------------------------------

The problem with this approach is that it hard to teach using ``retry``
when raw transactions "work on my laptop". However, this is somewhat
alleviated by failure injection.


Retry All Single Queries
------------------------

This specification describes that read-only non-transactional queries
should be retried automatically.

While it's tempting to retry all ``pool.query`` and ``pool.execute``
calls (even modifications), it **gives the false sense of security**: no
"concurrent update" issues are seen.  But it's better to see such error
and turn the whole block of code into a transaction rather than just a
mutation. I.e. retrying a single mutable request on a "concurrent
update" error must be a deliberate decision.


Is ``with_modifications`` Needed?
---------------------------------

It's intuitive that databases are mutable by default. But there is a
large class of applications that are mostly read-only and must have
limited and easy to find places having mutations. For those apps it's
better to have read-only connection by default and use a pattern like
this for writes:

.. code-block:: python

    def save(conn):
        async for tx in conn.with_mutations().retry():
          async with tx:
            ...

While retry is also easy to find, nothing stops user from writing:

.. code-block:: python

    conn.fetch("INSERT User { .. }")

Except the read_only configuration.

The metadata is will probably be implemented later and works the same:

.. code-block:: python

    def http_handler(req, conn):
        req['conn'] = conn.with_metadata(url=req.url)
        process_request(req)

Use Context Manager
-------------------

While it's tempting to use context manager for configuring connection,
in particular for the session config:

.. code-block:: python

    def handler(conn):
        with conn.session_config(name="value"):
            conn.query("...")

But behavior implies that session config is reset on each
``with/__exit__``, which has consequences:

* When ``handler()`` is interrupted, e.g. a timeout occurred, connection
  could be left in the inconsistent state (i.e. should be dropped)
* It prevents caching the session config, e.g. in the case above:
  ``handler(c); handler(c)`` will not set session config second
  time, but use the knowledge that these settings are already active.

Also it improves clarity and composition. E.g. (here we demonstrate
``with_metadata`` method with is a part of `Future Work`_:

.. code-block:: python

    def money_transfer(tx):
        user1 = tx.with_metadata(user='user1')
        user2 = tx.with_metadata(user='user2')
        withdraw(user1)
        deposit(user2)
        log_withdraw(user1)
        log_deposit(user2)

(This implies ``Transaction`` also contains ``with_metadata`` method)

Additional thing considered is that with context manager, if connection
is failed in the middle of the block:

.. code-block:: python

    with conn.session_config(name="value"):
        conn.query("...")
        # << -- here
        conn.query("...")

The implementation should reconnect and reapply the session
configuration anyway. So it deviates from the usual behavior of context
manager anyway.


Disabling Features
------------------

We may introduce a pool and/or connection configuration to disable
``ddl`` and ``config`` features on the requests. And/or disable
``session`` and ``transaction`` features on connection. This should be
default for many applications. But since scripting DDL and database
configuration is also a valid use case and since the risk of misusing
that is quite small we don't include it into the specification. Also
disabling ``ddl`` and ``config`` features should be covered by
permissions.


External References
===================

* `Transaction API Discussion <https://github.com/edgedb/edgedb/discussions/1708>`_
* `Transaction Retry Discussion <https://github.com/edgedb/edgedb/discussions/1738>`_
* `Stateful Connection Configuration Discussion <https://github.com/edgedb/edgedb/discussions/1896>`_
* `Failover and Replication Discussion <https://github.com/edgedb/edgedb/discussions/1859>`_
