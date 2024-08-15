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

The new module will be named ``std::net``.

Functions
---------

The module will provide the following functions:

1. HTTP Request Function:

.. code-block:: edgeql

   scalar type std::net::HttpMethod extending std::enums<`GET`, POST, PUT, `DELETE`, PATCH, HEAD, OPTIONS>;

   function std::net::http_request(
       url: str,
       named only body: optional str,
       named only method: std::net::HttpMethod = std::net::HttpMethod::GET,
       named only headers: optional array<tuple<name: str, value: str>>
   ) -> std::net::HttpResponse;

2. SMTP Send Function:

.. code-block:: edgeql

   function std::net::smtp_send(
       smtp_url: str,
       named only from: multi str,
       named only to: multi str,
       named only subject: str,
       named only text: optional str,
       named only html: optional str,
   ) -> std::net::SmtpResponse

ResponseState enum
------------------

Since network requests are asynchronous, the response state will be
represented by an enum:

.. code-block:: edgeql

   scalar type std::net::ResponseState extending std::enums<Pending, InProgress, Complete, Failed>;

Response Object Types
---------------------

The HTTP response object will be an internal type with the following
structure:

.. code-block:: edgeql

   type std::net::HttpResponse {
       required state: std::net::ResponseState;
       required created_at: datetime;

       status: int16;
       headers: json;
       body: bytes;
   }


The SMTP response object will be an internal type with the following
structure:

.. code-block:: edgeql

   type std::net::SmtpResponse {
       required state: std::net::ResponseState;
       required created_at: datetime;

       reply_code: int16;
       reply_message: str;
   }

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
       response := (
           select std::net::http_request(
               'https://api.example.com/webhook',
               body := payload,
               method := std::net::HttpMethod::POST,
               headers := [("Content-Type", "application/json")],
           )
       )
   select response {
       id,
       state,
       created_at,
   };

SMTP Send
---------

.. code:: edgeql

   with
       html_body := '<html><body><p>Hello, this is a test email.</p></body></html>',
       text_body := 'Hello, this is a test email.',
       response := (
           select std::net::smtp_send(
               'smtp://smtp.example.com:587',
               from := 'sender@example.com',
               to := {'recipient1@example.com', 'recipient2@example.com'},
               subject := 'Test Email',
               html := html_body,
               text := text_body
           )
       )
   select response {
       id,
       state,
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

