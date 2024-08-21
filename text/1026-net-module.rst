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
request.

.. code-block:: edgeql

  scalar type net::RequestState extending std::enums<Pending, InProgress, Complete, Failed>;

  scalar type net::RequestFailure extending std::enums<NetworkError, Timeout>;

``net::RequestState`` represents the current asynchronous state of a request.
The worker process will update the state of the request as it progresses through
the various states.

``net::RequestFailure`` represents the types of failure that can occur when
trying to send the request, not a failure response from the server.

HTTP
----

.. code-block:: edgeql

  abstract type net::http::Method extending std::enums<`GET`, POST, PUT, `DELETE`, PATCH, HEAD, OPTIONS>;

  type net::http::ScheduledRequest {
    required state: net::RequestState;
    required created_at: datetime;
    failure: tuple<failure: net::RequestFailure, message: str>;

    required url: str;
    required method: net::http::Method;
    required headers: array<tuple<name: str, value: str>>;
    required body: bytes;

    response: net::http::Response {
      constraint exclusive;
    };
  };

  type net::http::Response {
    required status: int16;
    required headers: array<tuple<name: str, value: str>>;
    body: bytes;
  };

  function net::http::request(
    url: str,
    named only body: optional bytes,
    named only method: net::HttpMethod = net::HttpMethod::GET,
    named only headers: optional array<tuple<name: str, value: str>>
  ) -> net::http::ScheduledRequest;

SMTP
----

.. code-block:: edgeql

  type net::smtp::ScheduledRequest {
    required state: net::RequestState;
    required created_at: datetime;
    failure: tuple<failure: net::RequestFailure, message: str>;

    required url: str;
    required from: multi str;
    required to: multi str;
    required subject: str;
    required text: optional str;
    required html: optional str;

    response: net::smtp::Response {
      constraint exclusive;
    };
  };

  type net::smtp::Response {
    required reply_code: int16;
    reply_message: str;
  };

  function net::smtp::send(
    url: str,
    named only from: multi str,
    named only to: multi str,
    named only subject: str,
    named only text: optional str,
    named only html: optional str,
  ) -> net::smtp::ScheduledRequest;

Implementation Details
----------------------

- Pending ``ScheduledRequest`` objects will be represent the queue of requests to be sent.
- A Rust process will handle sending the requests.
- Each protocol (HTTP, SMTP) will have its own pool of worker processes.
- URLs will initially be represented as plain strings, with the possibility of adding type-checked URL support in the future.

Examples
========

HTTP Request
------------

.. code:: edgeql

   with
       payload := '{"key": "value"}',
       request := (
           select net::http::request(
               'https://api.example.com/webhook',
               body := payload,
               method := net::HttpMethod::POST,
               headers := [("Content-Type", "application/json")],
           )
       )
   select request {
       id,
       state,
       created_at,
       url,
   };

SMTP Send
---------

.. code:: edgeql

   with
       html_body := '<html><body><p>Hello, this is a test email.</p></body></html>',
       text_body := 'Hello, this is a test email.',
       request := (
           select net::smtp::send(
               'smtp://smtp.example.com:587',
               from := 'sender@example.com',
               to := {'recipient1@example.com', 'recipient2@example.com'},
               subject := 'Test Email',
               html := html_body,
               text := text_body
           )
       )
   select request {
       id,
       state,
       created_at,
       url,
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
3. Allow retrying through configuration at request creation time.
4. Integration with a future EdgeDB queuing module to gain a more
   sophisticated retry strategies, durability, and reliability.
