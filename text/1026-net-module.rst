::

    Status: Draft
    Type: Feature
    Created: 2024-08-14
    Authors: Scott Trinh <scott@edgedb.com>

=============================
RFC: EdgeDB Networking Module
=============================

This RFC outlines the addition of a networking module to EdgeDB’s
standard library, providing functionality for making HTTP requests and
sending SMTP messages.

Motivation
==========

The EdgeDB Auth extension (ext::auth) currently sends emails directly to
configured SMTP services at certain times in various authentication
ceremonies and workflows:

-  Sign up with email (sends email verification email)
-  Password reset (sends password reset email)
-  Magic link sign up (sends magic link email)
-  Magic link sign in (sends magic link email)

To allow full customization of emails and provide hooks into
authentication events generally, we need to support sending webhook
requests to the application directly. This would enable applications to
trigger their own logic for sending fully customized and
internationalized emails, SMS, log analytics, update usage billing
records, etc.

Specification
=============

Module Name
-----------

The new module will be named ``net``.

Utility types
-------------

Since requests are sent asynchronously, we need a way to track the status of the
request. This is done by using a ``net::Task`` type to represent a request and
its current state. Each protocol will have its own ``net::Task`` concrete type.

.. code-block:: edgeql

  scalar type net::TaskState extending std::enums<Pending, InProgress, Complete, Failed>;

  scalar type net::TaskFailure extending std::enums<NetworkError, Timeout>;

  abstract type net::Task {
    required state: net::TaskState;
    required created_at: datetime;
    failure: tuple<failure: net::TaskFailure, message: str>;
  }

HTTP
----

.. code-block:: edgeql

  abstract type net::http::Method extending std::enums<`GET`, POST, PUT, `DELETE`, PATCH, HEAD, OPTIONS>;

  type net::http::Request {
    required url: str;
    required method: net::http::Method;
    required headers: array<tuple<name: str, value: str>>;
    required body: bytes;
  };

  type net::http::Response {
    required status: int16;
    required headers: array<tuple<name: str, value: str>>;
    body: bytes;
  };

  type net::http::Task extending net::Task {
    required request: net::http::Request;
    response: net::http::Response;
  };

  function net::http::request(
    url: str,
    named only body: optional bytes,
    named only method: net::HttpMethod = net::HttpMethod::GET,
    named only headers: optional array<tuple<name: str, value: str>>
  ) -> net::http::Task;

SMTP
----

.. code-block:: edgeql

  type net::smtp::Request {
    required url: str;
    required from: multi str;
    required to: multi str;
    required subject: str;
    required text: optional str;
    required html: optional str;
  };

  type net::smtp::Response {
    required reply_code: int16;
    reply_message: str;
  };

  type net::smtp::Task extending net::Task {
    required request: net::smtp::Request;
    response: net::smtp::Response;
  };

  function net::smtp::send(
    url: str,
    named only from: multi str,
    named only to: multi str,
    named only subject: str,
    named only text: optional str,
    named only html: optional str,
  ) -> net::smtp::Task;

Implementation Details
----------------------

1. Requests will be stored in a queue table in the database.
2. A Rust process will handle sending the requests.
3. Each protocol (HTTP, SMTP) will have its own queue and pool of worker
   processes.
4. Simple retry logic will be implemented for failed requests.
5. URLs will initially be represented as plain strings, with the
   possibility of adding type-checked URL support in the future.

Examples
========

HTTP Request
------------

.. code:: edgeql

   with
       payload := '{"key": "value"}',
       task := (
           select net::http::request(
               'https://api.example.com/webhook',
               body := payload,
               method := net::HttpMethod::POST,
               headers := [("Content-Type", "application/json")],
           )
       )
   select task {
       id,
       state,
       request,
       created_at,
   };

SMTP Send
---------

.. code:: edgeql

   with
       html_body := '<html><body><p>Hello, this is a test email.</p></body></html>',
       text_body := 'Hello, this is a test email.',
       task := (
           select net::smtp::send(
               'smtp://smtp.example.com:587',
               from := 'sender@example.com',
               to := {'recipient1@example.com', 'recipient2@example.com'},
               subject := 'Test Email',
               html := html_body,
               text := text_body
           )
       )
   select task {
       id,
       state,
       request,
       created_at,
   };

Backwards Compatibility
=======================

This RFC introduces new functionality and does not affect existing
features. There are no backwards compatibility issues.

Rejected Alternative Ideas
==========================

1. Using pg_net: While pg_net provides similar functionality, it was
   decided to implement our own solution for better control and
   integration with EdgeDB. This allows end users to more easily scale
   sending by scaling the EdgeDB server rather than scaling PostgreSQL.
2. Fully configurable queuing mechanism: For the initial implementation,
   a simple, built-in policy will be used instead of a fully
   configurable one to reduce complexity.

Future Related Work
===================

1. Add support for more protocols (e.g., AMQP, ZeroMQ, SQS, FTP).
2. Implement fully type-checked URLs and standard library functions to
   assist in constructing correct URLs, and with quoting and
   concatenation.
3. Integration with a future EdgeDB queuing module to gain a more
   sophisticated retry mechanism with backoff strategies.

