ARM-15: Agent Kerberos
======================

Summary
-------

Allow agent authentication via Kerberos using GSSAPI and SPNEGO as an
alternative to puppet agent certificates.

Motivation
----------

This ARM is motivated by:

1) The **security requirement** to use separate systems for agent authentication
   management and agent configuration management.

2) The **security requirement** to minimize the number of authentication systems
   within an organization.

3) The **operational efficiency** gained by using an existing Kerberos
   infrastructure in lieu of deploying and securing an additional certificate
   authority.

This new capability will benefit organizations that have an established Kerberos
infrastructure that they wish to leverage for Puppet agent authentication.

As a concrete example, without this feature setting up a new Puppet managed host
would require two priviledged operations: one to add it to the Kerberos domain,
and a second to add it to Puppet. With this feature adding the host to the
domain can automatically add it to Puppet.

> **NOTE**: acording to the ARM spec "Motivation" should be below Goals and
Non-Goals, but it seems best placed here...

Goals
-----

This ARM has the following goals:

* Add a minimal set of enhancements to the puppet master and agent
  infrastructure to allow agent authentication via SPNEGO/Kerberos.

* Support Linux agents that have a pre-existing, functioning, GSSAPI
  infrastructure and a computer account in a Windows Active Directory domain.

* Support the apache2 web server.
> **TODO**: we also anticipate support for nginx,
  however no work has been done for that yet.

* Allow the majority of agent functionality to operate securely without a client
  certificate -- eliminating the need to deploy and secure certificate authority
  infrastructure for puppet clients.

* Allow Kerberos based authentication to co-exist with client certificate based
  authentication on the same http server and port.

* Introduce no new hard dependencies. The gssapi and ffi gems are required on
  both master and agent, but only when this feature is configured.

* Document the necessary web server and puppet configuration.

Non-Goals
---------

* This ARM does not provide a general purpose "pluggable" authentication
  framework.

* This ARM does not support the webrick web server.

* This ARM does not include support for the Windows SSPI API (for Windows
  agents), though that would likely be a straightforward extension of this work.

* This ARM does not include a general purpose Ruby SPNEGO implementation, though
  it should be possible to switch to such a general purpose facility should it
  become available in the future.

* This ARM continues to rely on the agent verifying a master certificate during
  the SSL handshake to protect against man-in-the-middle (MITM) attacks. (See
  HTTPS Channel Binding below.)

* This ARM does not include support for connections from the master to the
  agent. (See HTTPS Channel Binding below). Secure master to agent communication
  may, in theory, be achieved via a Kerberos/GSSAPI compatible message bus and
  MCollective, though this ARM does not provide any information on how to
  configure and use MCollective.

* This ARM does not provide any information on how to correctly configure
  GSSAPI/Kerberos.

Description
-----------

This ARM covers three broad areas of work: web server configuration, puppet
master enhancements, and puppet agent enhancemnets.

### Web Server Configuration ###

Configuration of the apache2 web server is relatively straighforward. After
configuring apache normally the additional steps involve installing and enabling
mod_auth_kerb and adding several settings in the puppetmaster virtual host
configuration file. This ARM will update example configuration files and related
documentation.

> **TODO**: list explicit doc pages and example configuration files to update.

Once successfully configured and IFF a request is authenticated, mod_auth_kerb
will place the Kerberos principle name in the REMOTE_USER environment variable
where puppet can retrieve it.

> **TODO**: there is a known issue with non-local apache hosts that forward
requests to the master. I have been unable to copy the REMOTE_USER value to an
HTTP X header.

> **TODO**: add nginx details.

### Puppet Master Enhancements ###

Two puppet master enhancemnts are required: changes to the `Network::Http::Rack`
module to support the `REMOTE_USER` environment variable, and adding a hostname
lookup facility.

#### REMOTE_USER Support ####

`Network::Http::Rack#extract_client_info` extracts authentication information
from incoming requests. It looks at the `HTTP_X_CLIENT_DN`,
`HTTP_X_CLIENT_VERIFY`, and uses them to determin a "`:node`" name and an
"`:authenticated`" boolean value. The existing behavior for SSL based
authentication remains unchanged. When SSL cannot authenticate the client the
master checks the `REMOTE_USER` to see if it can be used to authenticate the
client and provide a hostname. If successful it considers the client as
authenticated and uses the hostname as the `:node`.

The following table summarizes the possible input states and their corresponding
results. (State names are for this document only and are not part of the implementation.)

STATE             | DN             | VERIFY  | REMOTE_USER     | :node          | :authenticated
------------------|----------------|---------|-----------------|----------------|---------------
NONE              | n/a            | n/a     | n/a             | nil            | false
UNVERIFIED        | dn.example.com | FAILED  | n/a             | nil            | false
VERIFIED          | dn.example.com | SUCCESS | n/a             | dn.example.com | true
KERB              | n/a            | FAILED  | rn$@EXAMPLE.COM | rn.example.com | true
UNVERIFIED_W_KERB | dn.example.com | FAILED  | rn$@EXAMPLE.COM | rn.example.com | true
VERIFIED_W_KERB   | dn.example.com | SUCCESS | rn$@EXAMPLE.COM | dn.example.com | true

The `NONE` state is not relevant to and will not be changed by this ARM.

The `UNVERIFIED` and `VERIFIED` states are part of the traditional client
certificate based agent lifecycle and will not be changed by this ARM.

The `KERB` state occurs when an agent successfully authenticates via Kerberos
and does not provide a client certificate. This is the typical state for agents
authenticating via Kerberos. The value from `REMOTE_USER` is passed to a new
facility that looks up a corresponding hostname.

The `UNVERIFIED_W_KERB` state occurs when an agent successfully authenticates via
Kerberos and also provides an unverifiable client certificate. In this case the
client certificate is ignorned and this state is otherwise treated as `KERB`.

> **TODO**: should UNVERIFIED\_W\_KERB generate a warning?

The `VERFIFIED_W_KERB` state occurs when an agent successfully authenticates via
Kerberos and also provides a verfiable client certificate. To maintain backward
compatibility in this case the `REMOTE_USER` environment variable is ignored and
this state is otherwise treated as `VERIFIED`.

> **TODO**: should VERIFIED\_W\_KERB generate a warning?

#### Hostname Lookup ####

### Puppet Agent Enhancemnts ###

Two agent changes are required: implementation of the SPNEGO protocol in
`Puppet::Network::HTTP::Connection`, and support for certficate-less agents.

#### SPNEGO Support ####

Adding SPNEGO support based on the gssapi gem requires adding an appropriately
formatted `Authorization` header to agent HTTP requests. The master responds
with an `www-authenticate` header which is then passed back to GSSAPI. This may
repeat for several iterations before authentication completes.

> **TODO**: The current prototype implementation assumes the negotiation
completes in a single roundtrip.

**Note**: The agent still requires that the server certificate be verified to
prevent MITM attacks.

In the current prototype use of SPNEGO is controlled by a boolean configuration
setting.

#### Certificate-less Agents ####

Puppet agent normally requires the existence of a signed agent certificate. The
`Puppet::SSL::Host` class manages the lifecycle of this certificate. When
configured for Kerberos authentication the agent runs without a certificate. The
`Host` class will be modified to support both modes of operation.

In the current prototype a configuration setting turns makes several methods of
`Host` into no-ops.

#### Hostname Lookup ####



Testing and Evaluation
----------------------

[TODO: fill out testing and evaluation]

* Spec, unit, tests.

* Provide curl example for how to test a master from the command line.

* Etc....


Alternatives and Recommendation
-------------------------------

In terms of implementation alternatives we explored using the Kerberos principle
name directly as the "node" name. A typical Kerberos principle would be
host$@EXAMPLE.COM. Supporting the mixed case and $ character required changes to
the AuthStore and was expected to lead to further complications with e.g.
filesystem support for the $. The current approach of using a principle to
hostname lookup service proved much simpler.

In terms of alternatives to the feature itself, customers still have the
opportunity to use Puppet's built-in CA capabilities or to integrate with
external CA's. These are all valid approaches and they can in fact be combinded.
We recommend Kerberos only for customers who already have a functioning Kerberos
infrastructure.

Risks and Assumptions
---------------------

* multiple request transmits due to multiple token exchanges in SPNEGO.

* authentication on each request instead of on each connection

* channel binding


Dependencies
------------

Describe all dependencies that this ARM has on other ARMs, components,
products, or anything else.  Dependences upon other ARMs should also
be listed in the "depends:" field in the metadata.json.

Describe any ARMs that depend upon this ARM before they can be implemented.

Impact
------

How will this work impact other parts of the platform, the product,
and the contributors working on them?  Omit any irrelevant items.

- Other Puppet components: ...

* kick

* MCollective

- Compatibility: ...
- Security: ...

* Channel Binding and master->agent MITM protection.

- Performance/scalability: ...

* Latency for contacting KDC

* Multiple re-sends for multiple token exchange.

* Reverse proxy support. Based on current experimentation it does not seem
possible to place the principle name into an HTTP extension header as we do for
the client certificate CN.

- User experience: ...
- I18n/L10n: ...
- Accessibility: ...
- Portability: ...
- Packaging/installation: ...
- Documentation: ...
- Spin-offs/Future work: ...

* SSPI support

* Channel Binding

* Reverse proxy support (need to figure out how to get REMOTE_USER into an HTTP header)

- Other: ...
