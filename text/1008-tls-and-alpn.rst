::

    Status: Accepted
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
``edgedb instance create``), the user could provide a custom certificate
and its private key to be used on the server::

    --tls-cert-file  Path to a single file in PEM format containing the
                     TLS certificate to run the instance, as well as any
                     number of CA certificates needed to establish the
                     certificate’s authenticity. If not present,
                     a self-signed certificate will be generated and
                     used instead.
    --tls-key-file   Path to a file containing the private key. If not
                     present, the private key will be taken from
                     --tls-cert-file as well.

(This is exactly ``ssl.SSLContext.load_cert_chain()`` from Python [4]_.)

Internally, the specified path will be stored in ``metadata.json`` under
the data directory defined in RFC 1001 [9]_. EdgeDB CLI will copy the
given paths to the system service file to launch the actual EdgeDB
server as command-line parameters of ``edgedb-server``.

    In the future when the EdgeDB CLI supports remote Postgres clusters,
    the data directory will not contain the actual Postgres data files.
    However, ``metadata.json`` is still needed to describe the instance.

If the private key is protected by a password, the user will be prompted
for a password interactively. Alternatively, the user could provide the
password in an environment variable ``EDGEDB_TLS_PRIVATE_KEY_PASSWORD``.
In either way, the password will be stored in the same ``metadata.json``
and be further used to load the private key.

Here is a proposed sample ``metadata.json``::

    {
        "format": 2,
        "version": "1-beta2",
        "current_version": "1.0b2+ga7130d5c7.cv202104290000",
        "slot": "1-beta2",
        "method": "Package",
        "port": 10733,
        "start_conf": "Auto",
        "tls_cert_file": "path/to/cert.pem",
        "tls_key_file": "path/to/key.pem",
        "tls_private_key_password": "password-in-cleartext"
    }

The EdgeDB CLI will correspondingly generate a system service file that
eventually launches the EdgeDB server as follows::

    EDGEDB_SERVER_TLS_PRIVATE_KEY_PASSWORD=password-in-cleartext \
    edgedb-server \
        --port=10733 \
        --tls-cert-file=path/to/cert.pem \
        --tls-key-file=path/to/key.pem

Particularly for Docker installation, there is no ``metadata.json``.
Instead, the metadata is stored in the Docker container spec and the
volume labels. As the user-specified files must be mounted into the
Docker container to be used by the server, the CLI could attach the
password and the path to those files in the Docker as a label on the
corresponding volume.

This RFC does not involve setting up a proper CA-based trust chain for
production usage. The knowledge will be well-documented, and products
like the Aether will have its own way doing so.


Generate Self-signed Certificates
---------------------------------

The EdgeDB CLI will automatically generate a self-signed certificate if
``--tls-cert-file`` is not present in command ``edgedb project init`` or
``edgedb instance create`` under the data directory of that EdgeDB
instance as described in the previous section, in the names of
``edbtlscert.pem`` for the certificate in PEM format, and
``edbprivkey.pem`` for the private key (no passphrase for simplicity).
This certificate is supposed to be used for development purposes only.

Likewise, the ``metadata.json`` file will be updated with the full path
to the generated files for consistency.


Client-side Server Certificate Verification
===========================================

On the client side (both the language bindings and the REPL), TLS server
certificate verification should always be enabled. By default, the
system-wide trusted CA certificates are usually used to verify server
certificates. For server certificates that are signed by untrusted CA,
the users could provide the path to the specific CA certificate file
they trust - for CLI the option is ``--tls-ca-file``, for language
bindings the option is usually ``tls_ca_file`` or ``tlsCaFile``.

In order to accept the self-signed certificate, at the time of
certificate generation, the EdgeDB CLI will also copy the generated
certificate into the so-called credentials JSON - a group of JSON
files named after the EdgeDB instance in a well-known place (e.g.
``~/.config/edgedb/credentials/`` depending on the OS) that are meant to
store credentials for the client to establish connections to the EdgeDB
instance. For example::

    {
        "port": 10732,
        "user": "edgedb",
        "password": "login-password-in-clear-text",
        "database": "edgedb",
        "tls_cert_data": "-----BEGIN CERTIFICATE-----\nMIICvjCCAaagA..."
    }

The language bindings and the REPL should load the certificate from the
value of ``tls_cert_data`` and trust only that certificate for
connecting to the EdgeDB instance if ``tls_cert_data`` is present.

The client allows the user to decide if hostname should be checked. For
CLI, the options are ``--tls-verify-hostname`` to enable the check, and
``--no-tls-verify-hostname`` to disable it. For language bindings, the
option is usually a boolean ``tls_verify_hostname``, where ``true``
means enabling and ``false`` for disabling. By default, the client will
check hostname if ``tls_cert_data`` is not present, and skip hostname
check for self-signed certificate.

In order to connect to remote instances running on a self-signed
certificate (also works for other purposes), a new CLI command is
proposed to create a local credentials JSON file to simplify future
connections::

    edgedb instance link

    Link to a remote EdgeDB instance and assign an instance name to
    simplify future connections.

    USAGE:
        edgedb instance link [FLAGS] [name]

    ARGS:
        <name>
            Specify a new instance name for the remote server. If not
            present, the name will be interactively asked.

    FLAGS:
        --non-interactive
            Run in non-interactive mode (accepting all defaults)

        --quiet
            Reduce command verbosity.

Connection parameters are also accepted. For example::

    $ edgedb instance link --host db.example.org
    Specify the port of the server [default: 5656]:
    > 5656
    Specify the database user [default: edgedb]:
    > john
    Specify the database name [default: edgedb]:
    > edgedb
    Unknown server certificate: SHA1:26725134145cf36c1a18ecd031ee71038b1a1590. Trust? [y/N]
    > y
    Password for 'john': ****
    Specify a new instance name for the remote server [default: db_example_org]:
    > db_example_org
    Successfully linked to remote instance. To connect run:
      edgedb -I db_example_org

The user is responsible for trusting the server certificate, because
trusting unknown certificates in production may lead to MITM attacks.
This command also verifies the user login information with the server
and only create a corresponding credentials JSON file if the login is
successful. In the above example,
``~/.config/edgedb/credentials/db_example_org.json`` is created::

    {
        "host": "db.example.org",
        "port": 5656,
        "user": "john",
        "password": "login-password-in-clear-text",
        "database": "edgedb",
        "tls_cert_data": "-----BEGIN CERTIFICATE-----\nMIICvjCCAaagA..."
    }

In addition, ``edgedb instance unlink`` is also proposed to remove the
link to the remote instances. It simply accepts a name of the remote
instance. However, ``unlink`` will fail if the name points to a local
instance. ``edgedb instance status`` is also updated to include the
remote instances, whose statuses and versions will be probed in parallel
with a short timeout.

    The server may be advertising a chain of certificates. If the chain
    cannot pass the verification, ``edgedb instance link`` will only ask
    the user to trust the last certificate in the chain - which is
    usually an (intermediate) CA certificate. Because EdgeDB CLI will
    always verify the full chain, so only trusting the leaf-most server
    certificate won't allow the CLI to pass the verification.


ALPN and Protocol Changes
=========================

The ALPN support in target programming languages:

* Python [4]_: ``set_alpn_protocols()`` and ``selected_alpn_protocol()``
* Go [5]_: ``SupportedProtos`` and ``NegotiatedProtocol``
* Node.js [6]_: ``ALPNProtocols`` and ``alpnProtocol``
* Deno [10]_: Client-side ALPN support is not ready yet

For now, the EdgeDB server will advertise two protocols in ALPN (however
EdgeDB is not limited to only these two for future possibilities):

* ``edgedb-binary``: The EdgeDB binary protocol
* ``http/1.1``: HTTP-based protocol, including the server system API,
  and extensions like EdgeQL over HTTP, GraphQL over HTTP and Notebook.

The client (including the language bindings and the REPL) should choose
between ``edgedb-binary`` and ``http/1.1`` during TLS handshake based on
the scenario in which the user is using the client. If the client didn't
join the protocol negotiation (e.g. using curl to access the server
stats endpoint), the server will fallback to ``http/1.1`` - then it is
literally just HTTPS.

    Note: the server cannot tell if the client asked for a protocol that
    is not supported by the server, or didn't join the ALPN at all. The
    server will use ``http/1.1`` for both cases. However if the client
    asked for a specific protocol, it must check the ALPN result and
    raise an error if the result is not the expected protocol.

The EdgeDB server will no longer check the magical first-byte to switch
between HTTP protocol and the binary protocol - it is fully replaced by
the ALPN negotiation. Once the protocol is agreed upon, there is
currently no way to switch to another protocol except for reconnecting.


Advanced TLS Settings
=====================

Usually TLS just work out of the box with the default settings. But for
special security reasons, optionally the advanced TLS settings can be
modified in the EdgeDB config system per instance. Specifically:

+-------------------------+--------------------------+--------------------------------------------------------+-------------------+
| EdgeDB Config           | Python SSLContext member | Possible Values                                        | Default Value     |
+=========================+==========================+========================================================+===================+
| ``tls_minimum_version`` | ``minimum_version``      | ``1.2``, ``1.3``, ``MIN_SUPPORTED``, ``MAX_SUPPORTED`` | ``MIN_SUPPORTED`` |
+-------------------------+--------------------------+--------------------------------------------------------+-------------------+
| ``tls_maximum_version`` | ``maximum_version``      | ``1.2``, ``1.3``, ``MIN_SUPPORTED``, ``MAX_SUPPORTED`` | ``MAX_SUPPORTED`` |
+-------------------------+--------------------------+--------------------------------------------------------+-------------------+
| ``tls_ciphers``         | ``set_ciphers()``        | Output of ``openssl ciphers`` in the same format.      |                   |
+-------------------------+--------------------------+--------------------------------------------------------+-------------------+
| ``ecdh_curve``          | ``set_ecdh_curve()``     | A well-known elliptic curve                            |                   |
+-------------------------+--------------------------+--------------------------------------------------------+-------------------+
| ``dh_params``           | ``load_dh_params()``     | DH parameters in PEM format (not path to the file)     |                   |
+-------------------------+--------------------------+--------------------------------------------------------+-------------------+

Specifically for the TLS version, EdgeDB only supports TLS 1.2 and 1.3
for now. ``MIN_SUPPORTED`` is just ``1.2``, but the ``MAX_SUPPORTED`` is
the Python ``ssl.MAXIMUM_SUPPORTED`` magic constant, which is ``1.3`` at
the moment.

The remaining 3 configs will call the set/load methods on ``SSLContext``
only when they are set. EdgeDB doesn't verify the correctness of the
values.


Development of EdgeDB
=====================

The ``edb server`` command (for core development, but works the same as
``edgedb-server`` used by the CLI) will accept similar parameters as the
CLI has, but works slightly differently::

    --tls-cert-file PATH           Specify a path to a single file in PEM format
                                   containing the TLS certificate to run the
                                   server, as well as any number of CA
                                   certificates needed to establish the
                                   certificate’s authenticity. If not present,
                                   the server will try to find `edbtlscert.pem`
                                   in the --data-dir if set.

    --tls-key-file PATH            Specify a path to a file containing the
                                   private key. If not present, the server will
                                   try to find `edbprivkey.pem` in the --data
                                   dir if set. If not found, the private key
                                   will be taken from --tls-cert-file as well.
                                   If the private key is protected by a
                                   password, specify it with the environment
                                   variable:
                                   EDGEDB_SERVER_TLS_PRIVATE_KEY_PASSWORD.

    --generate-self-signed-cert    When set, a new self-signed certificate will
                                   be generated together with its private key if
                                   no cert is found in the data dir. The
                                   generated files will be stored in the data
                                   dir, or a temporary dir (deleted once the
                                   server is stopped) if there is no data dir.
                                   This option conflicts with --tls-cert-file
                                   and --tls-key-file, and defaults to True in
                                   dev mode.

The Python builtin TLS support will be used to handle the certificates
and ALPN, and the TLS transport implementation in uvloop is used for the
network. The ``ssl.SSLContext`` [4]_ will be initialized with the
default ``protocol=ssl.PROTOCOL_TLS``, leaving the control of accepted
TLS protocol versions to ``SSLContext.minimum_version`` and
``SSLContext.maximum_version``, which in turn are managed by the
corresponding EdgeDB configs mentioned in previous chapter, together
with the other minor tunings for ``ssl.SSLContext``.

``--tls-cert-file``, ``--tls-key-file`` are directly the parameters of
``ssl.SSLContext.load_cert_chain()``, while the EdgeDB server would
accept a password for the private key as an environment variable
``EDGEDB_SERVER_TLS_PRIVATE_KEY_PASSWORD``. However, the ``password``
argument of ``load_cert_chain()`` must always be set to a Python
function to avoid triggering OpenSSL to prompt for password. If the env
var is not set, simply return ``b""`` in the function - it will not be
invoked if the private key is not protected by a password.

The ``--generate-self-signed-cert`` will - as explained in the help
message above - automatically generate self-signed certificate using
the Python cryptography [11]_ library. If certificate files pre-exist,
the ``--generate-self-signed-cert`` option will not generate new files
and overwrite.

For core EdgeDB development, the dev REPL ``edb cli`` command is also
enhanced with an additional call to ``edgedb instance link`` to trust
the generated self-signed certificate in local server::

    edgedb instance link _localdev --non-interactive

And ``edb cli`` by default invokes ``edgedb -I _localdev`` for
convenience.

For running tests, the path to the TLS certificate file in use is echoed
to the socket or file specified in ``--emit-server-status`` under JSON
key ``tls_cert_file``, so that the testing client could extract the path
to the certificate and load the TLS context.

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

Then the CLI would generate the client certificates using a local CA. As
the server knows which root CA certificate to trust, it will be able to
verify the authenticity of the client certificates it received through
the wire.

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
We may be able to place the client certificate in the credentials JSON
file so that the user don't have to bother dealing with the certificates
any more. And we could likely skip the passphrase for development client
certificates.


Backwards Compatibility
=======================

While TLS will be enforced by default, compatible mode is still
available for the server before EdgeDB 1.0, but it is only for the
EdgeDB developers (or special use cases like Deno clients) and should
not be enabled by the users.

+------------+----------------+----------------+---------------------------+
|            | Old Server     | New Server     | New Server in Compat Mode |
+============+================+================+===========================+
| Old Client | Accessible     | Friendly Error | Accessible                |
+------------+----------------+----------------+---------------------------+
| New Client | Accessible     | Accessible     | Accessible                |
+------------+----------------+----------------+---------------------------+

The EdgeDB development server (``edb server``) will provide a hidden
option ``--allow-cleartext-connections`` to run the server in compatible
mode for development and testing only. It will fallback to cleartext
transport if the TLS handshake fails. This option is not available in
the EdgeDB CLI (``edgedb instance``).

On the other hand, without ``--allow-cleartext-connections``, the new
server will return a user-friendly error in plain text if the SSL
handshake fails, in binary protocol or HTTP depending on again the
magical first-byte. Similarly, if the new client could not establish a
TLS connection on new servers based on the protocol version, it should
raise a proper error with the reason.


CLI and Server Compatibility
----------------------------

An old version of the CLI won't be able to start a database instance
with the new version of the server, because the new server requires TLS.
A friendly message should be displayed by the server, suggesting to
upgrade the CLI.

New CLI on the other hand could run both old and new servers. The CLI
must check the server version and provide different TLS parameters
accordingly.

The user could use the new CLI to upgrade an existing server instance
running on old server software to the newer version. The CLI will prompt
for options, the user could choose from either letting the CLI create a
self-signed certificate, or specify a certificate and private key
manually.


Security Implications
=====================

Enforcing TLS is supposed to be a full level-up in terms of security. It
provides basic eavesdropping protection, and if configured properly the
MITM protection too.

For both the server-side and client-side (if implemented) certificate
verification, the corresponding private keys and their passphrases are
critical for system security. Malicious parties could use the server
credential to start a fake but valid server, potentially being able to
collect sensitive queries without the user knowing. And a cracker could
use the users' credentials to access their data in the database.

As the server private key passphrase may be stored in the
``metadata.json`` file in clear text, the data directory needs extra
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

6. Don't store user-provided cert passphrase in credentials JSON.

   Storing password in a file is usually risky. The proposed way was
   either using an environment variable, or fetch the passphrase through
   a user-specified command like Postgres. Because EdgeDB server
   instances can be configured to start automatically, using env var is
   just the same as storing in a file, so only the Postgres way is safe.
   For now, we're just assuming credentials JSON is secure, as it is
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

11. Store the private key and passphrase in credentials JSON file.

    This file is not supposed to be used by the server, and the
    passphrase is only needed by the server. Another previous attempt
    was to use a user-specified command for the private key passphrase
    like Postgres, because the the service may auto start and the key
    passphrase has to be provided in some form. However this command
    can be a confusing option for users using Docker, as the command is
    supposed to run on the host machine, which also brings trouble to
    our CLI implementation. So eventually we just store the passphrase
    in ``metadata.json`` and feed it to ``edgedb-server`` as an
    environment variable.

12. Generate self-signed certificate in the CLI.

    We tried this and it works fine for most of the cases. However, we
    would need the certificate generation feature in some cases where
    CLI is not convenient, e.g. edgedb-python testing CI runs server
    directly using ``edgedb-server`` command.

.. [1] https://datatracker.ietf.org/doc/html/rfc5246
.. [2] https://datatracker.ietf.org/doc/html/rfc7301
.. [3] https://github.com/edgedb/edgedb/discussions/1910
.. [4] https://docs.python.org/3/library/ssl.html
.. [5] https://golang.org/pkg/crypto/tls/
.. [6] https://nodejs.org/api/tls.html
.. [7] https://tools.ietf.org/search/rfc2818#section-3.1
.. [8] https://github.com/edgedb/webapp-bench
.. [9] https://github.com/edgedb/rfcs/blob/master/text/1001-edgedb-server-control.rst#instance-names
.. [10] https://github.com/denoland/deno/issues/11479
.. [11] https://cryptography.io/en/latest/
