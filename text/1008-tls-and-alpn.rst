::

    Status: Draft
    Type: Feature
    Created: 2021-06-15
    Authors: Fantix King <fantix@edgedb.com>
    RFC PR: `edgedb/rfcs#0037 <https://github.com/edgedb/rfcs/pull/37>`_

==============================
RFC 1008: TLS and ALPN Support
==============================

This RFC proposes to change the transport of EdgeDB frontend connections
to TLS, and use ALPN for multiplexing protocol selection.


Motivation
==========

The Transport Layer Security (TLS) Protocol is widely used for providing
privacy and data integrity between two communicating applications [1]_.
EdgeDB as a database should naturally support TLS as the transport layer
protocol.

With the help of EdgeDB tooling, it will be easy to setup TLS
certificates for both development and production environments. Given
also that the benchmark [8]_ result of EdgeDB throughput overhead
comparing TLS to vanilla TCP is about 5% ~ 15%, it is also proposed to
communicate in TLS by default.

.. image:: ./EdgeDB-TLS-Benchmark.png

At last, with TLS enabled by default, it is possible to leverage the TLS
Application-Layer Protocol Negotiation (ALPN) Extension [2]_ for secure
and reliable protocol selection on top of the TLS transport, allowing
EdgeDB to multiplex different frontend protocols like the binary
protocol and the HTTP-based protocol on the same port as recommended in
Pre-RFC 1910 [3]_.


Overview
========

All connections to an EdgeDB server should use a transport with TLS
enabled. An EdgeDB server instance needs at least a TLS certificate and
its corresponding private key to start. A valid certificate and its key
are usually expected to be provided to the server for production
environments. EdgeDB clients (including the bindings and the CLI) must
verify the authenticity and validity of the server certificate, or an
error must be raised and the transport must be closed.

To make the development experience streamlined, the EdgeDB tooling will
automatically generate development-level certificates if none is
provided. This process should be transparent and applies to both
application and core developers.

During the TLS handshake, the client may choose to negotiate a protocol
using ALPN between the EdgeDB binary protocol and the HTTP protocol, and
if succeeded that protocol must be used in the following communication.
Otherwise (or if the client chose not to do ALPN), the HTTP protocol is
supposed to be used as the default protocol.


EdgeDB Server Certificate
=========================

When creating an EdgeDB instance (with either ``edgedb project init`` or
``edgedb server init``), the user could provide a custom certificate and
its private key to be used on the server::

    --tls-certfile  Path to a single file in PEM format containing the
                    certificate as well as any number of CA certificates
                    needed to establish the certificate’s authenticity.
                    If not present, a self-signed certificate will be
                    generated and used instead (for development only!).
    --tls-keyfile   Path to a file containing the private key. If not
                    present, the private key will be taken from
                    --tls-certfile as well.

(This is exactly `ssl.SSLContext.load_cert_chain()` from Python [4]_.)

Internally, the specified path will be stored in ``metadata.json`` under
the data directory defined in RFC 1001 [9]_. EdgeDB CLI will copy the
given paths to the system service file to launch the actual EdgeDB
server as command-line parameters of ``edgedb-server``. With Docker
installation, the given files will be mounted into the server container.

    In the future when the EdgeDB CLI supports remote Postgres clusters,
    the data directory will not contain the actual Postgres data files.
    However, ``metadata.json`` is still needed to describe the instance.

If the private key is protected by a passphrase, a third parameter is
then required::

    --tls-passphrase-cmd    Required if the TLS private key is protected
                            by a passphrase. This command will be ran at
                            the time of startup or reloading, and should
                            contain only the passphrase in its stdout.

If present, the value of this parameter is also stored in the
``metadata.json`` file. For example::

    {
        "format": 2,
        "version": "1-beta2",
        "current_version": "1.0b2+ga7130d5c7.cv202104290000",
        "slot": "1-beta2",
        "method": "Package",
        "port": 10733,
        "start_conf": "Auto",
        "tls_certfile": "path/to/cert.pem",
        "tls_keyfile": "path/to/key.pem",
        "tls_passphrase_cmd": "command-echoing-passphrase-to-stdout"
    }

In this example, the EdgeDB CLI will correspondingly generate a system
service file that eventually launches the EdgeDB server as follows::

    edgedb-server \
        --port=10733 \
        --tls-certfile=path/to/cert.pem \
        --tls-keyfile=path/to/key.pem \
        --tls-passphrase-cmd=command-echoing-passphrase-to-stdout

    This RFC does not involve setting up a proper CA-based trust chain
    for production usage. The knowledge will be well-documented, and
    products like the Aether will have its own way doing so.


Generate Self-signed Certificates
---------------------------------

The EdgeDB CLI will automatically generate a self-signed certificate if
``--tls-certfile`` is not present in command ``edgedb project init`` or
``edgedb server init`` under the data directory of that EdgeDB instance
as described in the previous section, in the names of
``edgedb-cert.pem`` for the certificate in PEM format, and
``edgedb-key.pem`` for the private key (no passphrase for simplicity).
This certificate is supposed to be used for development purposes only.

Likewise, the ``metadata.json`` file will be update with the full path
to the generated files for consistency.


Client-side Server Certificate Verification
===========================================

On the client side (both the language bindings and the REPL), TLS server
certificate verification should always be enabled. In order to accept
the self-signed certificate, at the time of certificate generation, the
EdgeDB CLI will also copy the generated certificate into the so-called
``credentials.json`` - a group of JSON files named after the EdgeDB
instance in a well-known place (e.g. ``~/.edgedb/credentials/``
depending on the OS) that are meant to store credentials for the client
to establish connections to the EdgeDB instance. For example::

    {
        "port": 10732,
        "user": "edgedb",
        "password": "login-password-in-clear-text",
        "database": "edgedb",
        "tls_certdata": "-----BEGIN CERTIFICATE-----\nMIICvjCCAaagAw..."
    }

The language bindings and the REPL should load the certificate from the
value of ``tls_certdata`` and trust only that certificate for connecting
to the EdgeDB instance. However, the client should not enable the check
of the hostname, because 1) the generated self-signed certificate will
not contain the ``subjectAltName`` extension [7]_ as it's not reliable
for the CLI to enumerate all hostnames on some non-local installations,
and 2) hostname check is likely unnecessary for the following scenario.

For remote clients that don't have access to the ``credentials.json``
file on the server-side, a new command is proposed to create a local
``credentials.json`` file for all future connections to the same
instance::

    edgedb authenticate

    Authenticate to a remote EdgeDB instance and assign an instance name
    to simplify future connections.

    USAGE:
        edgedb authenticate [OPTIONS] <host:port>

    ARGS:
        <host:port> IP/DNS name and the port of the target instance.

    OPTIONS:
        --name <name>
            Specify a new instance name for the remote server. If not
            present, the name will be interactively asked.

        --user <user>
            The database user to log into the remote server. If not
            present, the username will be interactively asked.

        --password <password>
            The password for the database user to log into the remote
            server. If not present, the username will be interactively
            asked. This is also available as an environment variable
            `EDGEDB_PASSWORD`.

        --database <database>
            The name of the default database to connect to.

For example::

    $ edgedb authenticate db.example.org:5656
    User: john
    Password: ******
    Default database: edgedb
    Here is the server certificate:
      Hostname: db.example.org
      Org: Company Inc.
      Fingerprints:
        SHA-256: 63:2B:11:99:44:40:17:DF:37:FC:C3:DF:0F:3D:15
    Confirm? [Y/n] Y
    Login successful.
    Please specify a name for this instance: [db_example_org_5656]
    Credential file created, you can now connect to the database with:
        edgedb -I db_example_org_5656

The user is responsible for trusting the server certificate, because
trusting unknown certificates in production may lead to MITM attacks.
This command also verifies the user login information with the server
and only create a corresponding ``credentials.json`` file if the login
is successful. In the above example,
``~/.edgedb/credentials/db_example_org_5656.json`` is created::

    {
        "host": "db.example.org",
        "port": 5656,
        "user": "john",
        "password": "login-password-in-clear-text",
        "database": "edgedb",
        "tls_certdata": "-----BEGIN CERTIFICATE-----\nMIICvjCCAaagAw..."
    }

And then the client logic for server certificate verification is just
the same as for local development as explained earlier in this section.


ALPN and Protocol Changes
=========================

The ALPN support in target programming languages:

* Python [4]_: ``set_alpn_protocols()`` and ``selected_alpn_protocol()``
* Go [5]_: ``SupportedProtos`` and ``NegotiatedProtocol``
* Node.js [6]_: ``ALPNProtocols`` and ``alpnProtocol``

For now, the EdgeDB server will advertise two protocols in ALPN (however
EdgeDB is not limited to only these two for future possibilities):

* ``edgedb-binary``: The EdgeDB binary protocol
* ``http``: HTTP-based protocol, including the server system API, and
  extensions like EdgeQL over HTTP, GraphQL over HTTP and the Notebook.

The client (including the language bindings and the REPL) should choose
between ``edgedb-binary`` and ``http`` during TLS handshake based on the
scenario in which the user is using the client. If the client didn't
join the protocol negotiation (e.g. using curl to access the server
stats endpoint), the server will fallback to ``http`` - then it is
literally just HTTPS.

    Note: the server cannot tell if the client asked for a protocol that
    is not supported by the server, or didn't join the ALPN at all. The
    server will use ``http`` for both cases. However if the client asked
    for a specific protocol, it must check the ALPN result and raise an
    error if the result is not the expected protocol.

The EdgeDB server will no longer check the magical first-byte to switch
between HTTP protocol and the binary protocol - it is fully replaced by
the ALPN negotiation. Once the protocol is agreed upon, there is
currently no way to switch to another protocol except for reconnecting.


Advanced TLS Settings
=====================

Usually TLS just work out of the box with the default settings. But for
special security reasons, optionally the advanced TLS settings can be
modified in the EdgeDB config system per instance. Specifically:

+-------------------------+--------------------------+--------------------------------------------------------------------------+
| EdgeDB Config           | Python SSLContext member | Possible Values                                                          |
+=========================+==========================+==========================================================================+
| ``tls_minimum_version`` | ``minimum_version``      | ``1.0``, ``1.1``, ``1.2``, ``1.3``, ``MIN_SUPPORTED``, ``MAX_SUPPORTED`` |
+-------------------------+--------------------------+--------------------------------------------------------------------------+
| ``tls_maximum_version`` | ``maximum_version``      | ``1.0``, ``1.1``, ``1.2``, ``1.3``, ``MIN_SUPPORTED``, ``MAX_SUPPORTED`` |
+-------------------------+--------------------------+--------------------------------------------------------------------------+
| ``tls_ciphers``         | ``set_ciphers()``        | Output of ``openssl ciphers`` in the same format.                        |
+-------------------------+--------------------------+--------------------------------------------------------------------------+
| ``tls_ecdh_curve``      | ``set_ecdh_curve()``     | A well-known elliptic curve                                              |
+-------------------------+--------------------------+--------------------------------------------------------------------------+
| ``tls_dh_params``       | ``load_dh_params()``     | DH parameters in PEM format (not path to the file)                       |
+-------------------------+--------------------------+--------------------------------------------------------------------------+

The corresponding Python method or property will not be touched if the
config is not set. The TLS protocol versions and the magic constants
``MIN_SUPPORTED`` and ``MAX_SUPPORTED`` are mapped to corresponding TLS
constants. Other than that, EdgeDB doesn't verify the correctness of the
config values.


Development of EdgeDB
=====================

The ``edb server`` command (for core development, but works the same as
``edgedb-server`` used by the CLI) will accept similar parameters as the
CLI has::

    --tls-certfile  Path to a single file in PEM format containing the
                    certificate as well as any number of CA certificates
                    needed to establish the certificate’s authenticity.
                    If not present, a self-signed certificate will be
                    generated and used instead (for development only!).
    --tls-keyfile   Path to a file containing the private key. If not
                    present, the private key will be taken from
                    --tls-certfile as well.

The Python builtin TLS support will be used to handle the certificates
and ALPN, and the TLS transport implementation in uvloop is used for the
network. The ``ssl.SSLContext`` [4]_ will be initialized with the
default ``protocol=ssl.PROTOCOL_TLS``, leaving the control of accepted
TLS protocol versions to ``SSLContext.minimum_version`` and
``SSLContext.maximum_version``, which in turn are managed by the
corresponding EdgeDB configs mentioned in previous chapter, together
with the other minor tunings for ``ssl.SSLContext``.

``--tls-certfile``, ``--tls-keyfile`` are directly the parameters of
``ssl.SSLContext.load_cert_chain()``, while the EdgeDB server would
accept a password for the private key as an environment variable
``EDGEDB_TLS_PRIVATE_KEY_PASSWORD``. However, the ``password`` argument
of ``load_cert_chain()`` must always be set to a Python function to
avoid triggering OpenSSL to prompt for password. If the env var is not
set, simply return ``b""`` in the function - it will not be invoked if
the private key is not protected by a password.

When ``--tls-certfile`` is not present and the server is in ``devmode``
or ``testmode``, the server will use the CLI to generate a self-signed
certificate and use it to run the TLS server for development and
testing. The generated certificate (with its private key) should be
written to a ``credentials.json`` file just like the regular CLI
behavior, together with the connection parameters. For example, the
server would run this "hidden" CLI command on start::

    edgedb _dev --port 5656 --user edgedb \
        --password login-password-in-clear-text --database edgedb

And it overwrites ``~/.edgedb/credentials/local-dev.json`` with::

    {
        "port": 5656,
        "user": "edgedb",
        "password": "login-password-in-clear-text",
        "database": "edgedb",
        "tls_certdata": "-----BEGIN RSA PRIVATE KEY-----\nMIIEowIBAA..."
    }

And echo back the generated certificate in standard output so that the
server could simply read and use. So that the EdgeDB developer could
always use ``edgedb -Ilocal-dev`` to access the dev server. The test
suite could also take advantage from this mimic of real-life EdgeDB
scenario to cover some real cases.

Another server-side topic that was discussed in this RFC is the UNIX
domain socket. It is proposed that the non-admin UNIX socket support
should be removed, while the admin UNIX socket remains in clear-text
binary protocol.


Client Certificate
==================

Supporting client certificate authentication is a nice-to-have feature
in this RFC, as implementing a proper client certificate authentication
system can be complicated - if we also issue the client certificates,
we'd probably reconsider the CA idea below. In this section, we're only
discussing the feasibility.

First of all, we'd want to add a new Auth method ``Certificate`` beyond
the other two methods ``Trust`` and ``SCRAM``. The ``Certificate``
``Auth`` entry tells the EdgeDB server which users are allowed to
authenticate themselves using a client certificate.

Then we would need a new config type to store the trusted client
certificate fingerprints. The idea is to let the CLI generate the client
certificates using a local CA, and the CLI tells the server to trust the
generated certificates.

The certificate should contain the authorized database role in CN or an
X.509 extension, and that role must match the requested login user
during authentication. As the server may support several different Auth
methods at the same time with a customizable priority, a client
certificate is not mandatory in TLS. But if provided and if the server
is configured with ``Certificate`` Auth, then the client certificate
will be used as one authentication attempt.

On the client side, user may use the CLI to generate a client
certificate (and its corresponding private) for a particular database
role in a certain EdgeDB instance, and use the two files to establish a
connection to that EdgeDB server. The private key passphrase - if set -
must be securely provided through either environment variables, or API
parameters (following Python ``SSLContext.load_cert_chain()`` style).
We may be able to place the client certificate in the
``credentials.json`` file so that the user don't have to bother dealing
with the certificates any more. And we could likely skip the passphrase
for development client certificates.


Backwards Compatibility
=======================

While TLS will be enforced by default, compatible mode is still
available for the server before EdgeDB 1.0, but it is only for the
EdgeDB developers and should not be enabled by the users.

+------------+----------------+----------------+---------------------------+
|            | Old Server     | New Server     | New Server in Compat Mode |
+============+================+================+===========================+
| Old Client | Accessible     | Friendly Error | Accessible                |
+------------+----------------+----------------+---------------------------+
| New Client | Friendly Error | Accessible     | Friendly Error            |
+------------+----------------+----------------+---------------------------+

The EdgeDB development server (``edb server``) will provide a hidden
option ``--no-tls`` to run the server in non-TLS compatible mode (just
like the old server) for development and testing only. This option is
not available in the EdgeDB CLI (``edgedb server``).

At the same time, the new server will return a user-friendly error in
plain text if the SSL handshake fails, in binary protocol or HTTP
depending on again the magical first-byte. Similarly, if the new client
could not establish a TLS connection, it should raise a proper error
with the reason.


Security Implications
=====================

Enforcing TLS is supposed to be a full level-up in terms of security. It
provides basic eavesdropping protection, and if configured properly the
MITM protection too. This needs to be carefully documented as this RFC
does not involve setting up a proper CA-based trust chain for MITM
attacks.

For both the server-side and client-side (if implemented) certificate
verification, the corresponding private keys and their passphrases are
critical for system security. Malicious parties could use the server
credential to start a fake but valid server, potentially being able to
collect sensitive queries without the user knowing. And a cracker could
use the users' credentials to access their data in the database.

As the server private key passphrase is stored in the
``credentials.json`` file in clear text, this directory needs extra
attention for security purposes in production environments.


Rejected Alternative Ideas
==========================

1. Maintain a local CA per EdgeDB installation for all instances.

   Having a shared Certificate Authority (CA) makes the client easier to
   trust all the certificates issued by the CA - only the root CA
   certificate needs to be trusted. However, the path to the root CA
   certificate still needs to be stored somewhere. It's just cleaner to
   have separate self-signed certificates per development instance.

2. Import (copy) and manage user-specified certificates.

   Managing certificates in a consistent well-known place sounded like
   an idea. However, "if user specified the path to a file on the
   command-line they assume that file is used, not copied somewhere".
   And we still want to reload the certificate on e.g. each startup, so
   copying would not work.

3. Managing trusted certificates (letsencrypt).

   The common way certbot verifies the ownership of the hostname -
   namely exporting some files over HTTP and modifying DNS entries, they
   likely won't work in the EdgeDB scenario.

4. Advanced TLS settings in command parameters.

   This is simply unnecessary when we have the EdgeDB config system,
   which could also survive a backup and restore.

5. Adding passphrase to self-signed certificates.

   As the self-signed certificates are meant for development only, we
   didn't find a scenario where a passphrase is useful.

6. Don't store user-provided cert passphrase in ``credentials.json``.

   Storing password in a file is usually risky. The proposed way was
   either using an environment variable, or fetch the passphrase through
   a user-specified command like Postgres. Because EdgeDB server
   instances can be configured to start automatically, using env var is
   just the same as storing in a file, so only the Postgres way is safe.
   For now, we're just assuming ``credentials.json`` is secure, as it is
   designed to store passwords. Further comments are welcome.

7. Add a client-side switch to manually trust self-signed certificates.

   Good documentation would be sufficient. We proposed the SSH way for
   remote client connecting to a server running on a self-signed cert.

8. Python server generates the self-signed certificate.

   The EdgeDB server is a user of the certificate - the CLI is the one
   actually organizes the certificates. The server should just use
   whatever certificate is provided. Even for the special case of the
   development of the EdgeDB server itself, the CLI is still available.

9. Use separate ALPN protocol for EdgeQL, GraphQL, etc.

   On protocol level, they are all HTTP-based protocol. And there is no
   reason to redo the path-based extension system again with ALPN.

10. Automatically detect certificate and private key from data directory.

    The idea was to allow the server look into its data directory for
    the TLS key pair and use it automatically, so that the CLI could
    just store the generated self-signed key pairs into the data
    directories. But this is not possible for future instances with
    remote Postgres clusters - the server won't use a persistent data
    directory. So we decided to just pass in the paths to the key pair.

.. [1] https://datatracker.ietf.org/doc/html/rfc5246
.. [2] https://datatracker.ietf.org/doc/html/rfc7301
.. [3] https://github.com/edgedb/edgedb/discussions/1910
.. [4] https://docs.python.org/3/library/ssl.html
.. [5] https://golang.org/pkg/crypto/tls/
.. [6] https://nodejs.org/api/tls.html
.. [7] https://tools.ietf.org/search/rfc2818#section-3.1
.. [8] https://github.com/edgedb/webapp-bench
.. [9] https://github.com/edgedb/rfcs/blob/master/text/1001-edgedb-server-control.rst#instance-names
