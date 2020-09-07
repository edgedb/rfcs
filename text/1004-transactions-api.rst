::

    Status: Draft
    Type: Feature
    Created: 2020-09-03
    Authors: Paul Colomiets <paul@colomiets.name>
    RFC PR: `edgedb/rfcs#0004 <https://github.com/edgedb/rfcs/pull/19>`_

==========================
RFC 0004: Transactions API
==========================


Abstract
========

This RFC introduces new API for transactions that:

1. Within transaction requires queries be executed on transaction object
   rather than on connection itself
2. Defines interface for retrying transactions that is easy to use


Motivation
==========

We want to encourage users using connection pools rather than connections
directly. Older API requires connection to acquire a connection first:

.. code-block:: python

   with pool.acquire() as connection:
       with connection.trasaction():
           pass

With the new API we don't need to acquire a connection separately. This
not only simplifies the code, but also allows reconnecting on network
errors.

`More discussion on this topic <https://github.com/edgedb/edgedb/discussions/1708>`_

Second part of the RFC is mainly motiveated by the fact that PostgreSQL
sometimes can't apply concurrent transactions and errors out with::

    Error: could not serialize access due to concurrent update

It's expected that transaction may be repeated and be successful when
repeated. Common case when this happens is using ``INSERT ... UNLESS
CONFLICT ..`` statement in EdgeDB. See `Repeating-EdgeQL<#repeating-edgeql>`_
section for an explanation of why Postgres and EdgeDB can't repeat transactions
automatically.

EdgeDB has only ``REPEATABLE READ`` and ``SERIALIZABLE`` isolation levels.
That means errors like the above may happen more often than with the
default ``READ COMMITTED`` isolation level in Postgres.

As we implement retrying connection on "concurrent update" we also want
to handle:

1. Transaction deadlocks
2. Connection errors anywhere between ``BEGIN TRANSACTION`` and before
   ``COMMIT`` (we can't reliably know whether commit itself worked if we sent
   last ``COMMIT`` and got no response)

`More discussion on retrying <https://github.com/edgedb/edgedb/discussions/1738>`_


Specification
=============

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
function.  User can set either number of retry attempts either globally
(for all errors) or for specific error condition (say only deadlocks).

Current attempt number N is global. Which means if the last error is a
deadlock and N is greater than the number of attempts on a deadlock we
stop retrying and return error (even if previous error was a network
error).

Backoff function by default is ``2^N * 100`` plus random number in range
``0..100`` microseconds. Where first retry (second attempt) has ``N=1``.
Technically:

* In JavaScript: ``n => (2**n) * 100 + Math.random()*100``
* In Python: ``lambda n: (2**n) * 0.1 + randrange(100)*0.1`` (seconds)
* In Rust: ``|n| Duration::from_millis(2u64.pow(n)*100 + thread_rng().gen_range(0,100)``

Backoff is randomized so that if there was a coordinated failure (i.e.
server restart which triggers all current transactions to retry)
transactions don't overwhelm a database by reconnecting simultaneously.
If backoff function is adjusted it's recommeded to keep some
randomization anyway.


TypeScript API
--------------

**Introduce** two methods on a connection pool and one on connection:

.. code-block:: typescript

    type TransactionBlock = (Transaction) => Promise<T>;

    interface Pool {
        retry<T>(block: TransactionBlock, options: RetryOptions): T;
        try_transaction<T>(action: TransactionBlock, options?: TransactionOptions): T;
    }
    class Connection {
        raw_transaction<T>(action: TransactionBlock, options?: TransactionOptions): T;
    }

Introduce interface for making queries:

.. code-block:: typescript

    interface Executor {
        async query(query: string, args: QueryArgs = null): Promise<Set>;
        async queryOne(query: string, args: QueryArgs = null): Promise<any>;
        async queryJSON(query: string, args: QueryArgs = null): Promise<string>;
        async queryOneJSON(query: string, args: QueryArgs = null): Promise<string>;
        async execute(query: string): Promise<void>;
    }

And implement interface by respective classes:

.. code-block:: typescript

   class Transaction implmements Executor {/*...*/}
   class Connection implmements Executor {/*...*/}
   interface Pool extends Executor {/*...*/}

While removing inherent methods with the same name.

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

Note the new API is very similar to older ``transaction`` except the queries are
executed using transaction object, instead of connection itself.

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
            transactionOptions?: TransactionOptions,
        });
        transactionOptions(TransactionOptions): RetryOptions;
        attempts(
            which: AttemptsOption,
            attempts: number,
            backoff_ms?: BackoffFn,
        ): RetryOptions;
    }

The ``PoolOptions`` object receives ``retryOptions`` field:

.. code-block:: typescript

    interface PoolOptions {
      // ...
      retryOptions?: RetryOptions;
    }

Exceptions API
``````````````

Error hierarchy is amended by introducing ``TransientError``, ``NetworkError``
and ``EarlyNetworkError`` with the following relationships:

.. code-block:: typescript

    class TransientError extends TransactionError {}
    class TransactionDeadlockError extends TransientError {}
    class TransactionSerializationError extends TransientError {}
    class NetworkError extends EdgeDBError {}
    class EarlyNetworkError extends NetworkError {}

All network error within connection should be converted into
``EarlyNetworkError`` or ``NetworkError``. Former is used in context
where we catch network error before sending a request.


Python API
----------

For python API plain ``with`` doesn't work any more, so we introduce a loop
and with block. See example below.

Pool methods for creating a transaction:

.. code-block:: python

   class AsyncIOPool:
       def retry(
           options: RetryOptions = None
       ) -> AsyncIterable[AsyncContextManager[AsyncIOTransaction]]: ...
       async def try_transaction(*,
           isolation: str = None,
           readonly: bool = None,
           deferrable: bool = None,
       ) -> AsyncContextManager[AsyncIOTransaction]: ...

   class AsyncIOConnection:
       async def raw_transaction(*,
           isolation: str = None,
           readonly: bool = None,
           deferrable: bool = None,
       ) -> AsyncContextManager[AsyncIOTransaction]: ...

   class Pool:
       def retry(
           options: RetryOptions = None
       ) -> Iterable[ContextManager[Transaction]]: ...
       def raw_transaction(
           isolation: str = None,
           readonly: bool = None,
           deferrable: bool = None,
       ) -> ContextManager[Transaction]: ...

   class Connection:
       def raw_transaction(
           isolation: str = None,
           readonly: bool = None,
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

Example of ``try_transaction``:

.. code-block:: python

      async with db.try_transaction() as tx:
        let val = await tx.fetch("...")
        await tx.execute("...", process_value(val))

Note the new API is very similar to older ``transaction`` except the queries are
executed using transaction object, instead of connection itself.

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

        def transaction_options(self,
           isolation: str = None,
           readonly: bool = None,
           deferrable: bool = None,
        ) -> RetryOptions: ...

        def attempts(
            which: AttemptsOption,
            attempts: int,
            backoff_ms: Callable[[int], [float]],
        ): RetryOptions: ...
    }

Introduce the abstract classes for queries:

.. code-block:: python

    class AsyncExecutor(abc.AbstraceBaseClass):
        async def execute(self, query): ...
        async def query(self, query: str, *args, **kwargs) -> datatypes.Set: ...
        async def query_one(self, query: str, *args, **kwargs) -> typing.Any: ...
        async def query_json(self, query: str, *args, **kwargs) -> str: ...
        async def query_one_json(self, query: str, *args, **kwargs) -> str: ...

    class Executor(abc.AbstractClass):
        def query(self, query: str, *args, **kwargs) -> datatypes.Set: ...
        def query_one(self, query: str, *args, **kwargs) -> typing.Any: ...
        def query_json(self, query: str, *args, **kwargs) -> str: ...
        def query_one_json(self, query: str, *args, **kwargs) -> str: ...
        def execute(self, query: str) -> None: ...

These base classes should be implemented by respective classes:

.. code-block:: python

    class AsyncIOTransaction(AsyncExecutor): ...
    class AsyncIOConnection(AsyncExecutor): ...
    class AsyncIOPool(AsyncExecutor): ...
    class Transaction(Executor): ...
    class Connection(Executor): ...
    class Pool(Executor): ...


Exceptions API
``````````````

Error hierarchy is amended by introducing ``TransientError``, ``NetworkError``
and ``EarlyNetworkError`` with the following relationships:

.. code-block:: python

    class TransientError(TransactionError): ...
    class TransactionDeadlockError(TransientError): ...
    class TransactionSerializationError(TransientError): ...
    class NetworkError(EdgeDBError): ...
    class EarlyNetworkError(NetworkError): ...

All network error within connection should be converted into
``EarlyNetworkError`` or ``NetworkError``. Former is used in context
where we catch network error before sending a request.


Future Work
===========

Transaction on Specific Connection
----------------------------------

Do we want and how ``connection.retry()`` should work?

a. It may only retry on the same connection and fail on disconnect
b. It may reconnect and replace underlying socket in the connection
   object and retry
c. We may only allow transactions on connection pools

This can be postponed to later RFC when we know what are use cases of
it.


Retry Single Queries
--------------------

While it's tempting to retry ``pool.query`` and ``pool.execute`` calls,
it **gives the false sense of security**: no "concurrent update" issues
seen.  But it's better to see such error and turn the whole block of
code into a transaction rather than just a mutation. I.e. retrying a
single mutable request on a "concurrent update" error must be a
deliberate decision.


Retry Single Queries on Connection Error
----------------------------------------

To make changing EdgeDB address or restarting EdgeDB work nicely, we
need to retry simple queries on connection errors too:

.. code-block:: python

    pool.query("SELECT ..")

But there are couple of issues:

1. Repeating read-only queries is always safe, but we don't know which
   ones are readonly. We tackle this below.
2. Repeating non-readonly queries can be dangerous: they may be applied
   twice.

There are couple of ideas to differentiate read-only and mutable
queries:

1. We can expose "read-only" flag in ``Prepare`` (and it's always safe
   to retry before ``Execute`` happens)
2. Enable connection mode that forbids mutating queries outside of
   transaction and retry all (read-only) queries
3. Do (2) and have some method to override it on per-method-call basis
4. Or vice versa add ``pool.read_only_query()`` method
5. Add ``pool.read_only().query()``
6. Add ``pool.with_options({readonly: true}).query()``

Note: methods (2-6) are also helpful for working with primary/replica
installations. But probably only last two would allow full power, as they allow
``pool.read_only(primary=true)`` (i.e. in case you need read-only transaction
that can't go to a replica).

This issue can be solved by a later RFC.


Learning Curve
==============

This complicates learning curve, but:

1. This is already a problem in Postgres, there a lot of people who
   ignore the issue for pet projects and a lot of startups having "normal"
   rate or 500 errors at any point of time, while it's preventable.
2. Repeating transactions would make less incentive to keep transactions
   open for a long time (e.g. while accessing slow network resources like
   external API), which is a problem of itself.
3. Even if we never have failed concurrent updates we would want
   seamless reconnect on connection failures (i.e. server restart,
   primary/replica change, etc.)
4. To make learning curves shorter I think we should intentionally
   inject failures. This is needed so that users quickly find out that side
   effects of their transactions are in effect several times.

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


Keep `transaction` as but Add a Helper for Retrying
---------------------------------------------------

The problem with this approach is that it hard to teach using `retry`
when raw transactions "work on my laptop". However, this is somewhat
alleviated by failure injection.


External References
===================

* `Transaction API Discussion <https://github.com/edgedb/edgedb/discussions/1708>`_
* `Transaction Retry Discussion <https://github.com/edgedb/edgedb/discussions/1738>`_
