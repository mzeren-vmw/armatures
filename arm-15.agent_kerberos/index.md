ARM-15: Agent Kerberos
======================

Summary
-------

Allow agent authentication via Kerberos using GSSAPI and SPNEGO as an
alternative to Puppet agent certificates.

Motivation
----------

This ARM is motivated by:

1) The **security requirement** to use separate systems for agent authentication
   management and agent configuration management.

2) The **security requirement** to minimize the number of authentication systems
   within an organization.

3) The **operational efficiency** gained by using an existing Kerberos
   infrastructure in lieu of deploying certificate infrastructure for Puppet
   agents.

Organizations that have an established Kerberos infrastructure will benefit from
this enhancement to Puppet. As an example, without this feature setting
up a new Puppet managed host would require two privileged operations: one to
add it to the Kerberos domain, and a second to add it to Puppet. With this
feature adding the host to the domain can automatically add it to Puppet.

> **NOTE**: The "Motivation" section seems best placed here even though
according to the [ARM template][ARM] it should be below Goals and Non-Goals.

[ARM]: ../arm-1.templates/templates/index.md

Goals
-----

This ARM has the following goals:

* Add a minimal set of enhancements to the Puppet master and agent
  infrastructure to allow agent authentication via SPNEGO/Kerberos.

* Support **Linux agent hosts** that have a functioning **GSSAPI infrastructure** and a
  **computer account** in a **Windows Active Directory** domain.

* Allow the majority of agent functionality to operate securely without a client
  certificate -- eliminating the need to deploy and secure certificate authority
  infrastructure for Puppet clients.

* Allow Kerberos based authentication to co-exist with client certificate based
  authentication on the same HTTP server and port.

* Support the apache2 web server.
> **TODO**: we also anticipate support for nginx,
  however no work has been done for that yet.

* Introduce no new hard dependencies. The gssapi, ffi, and ruby-net-ldap gems
  are used, but only required when configured.

* Document the necessary web server and Puppet configuration.

Non-Goals
---------

* This ARM does not include support for the Windows SSPI API (for Windows
  agents), though that would likely be a straightforward extension of this work.

* This ARM supports only the Active Directory Kerberos implementation. This
  limitation is largely due to available testing resources. It is likely that
  other Kerberos implementations would be supported with little to no
  modification.

* This ARM does not support the webrick web server.

* This ARM does not provide a general purpose "pluggable" authentication
  framework.

* This ARM does not include a general purpose Ruby SPNEGO implementation, though
  it should be possible to switch to such a general purpose facility should it
  become available in the future.

* This ARM continues to require that the agent verify the master certificate
  during the SSL handshake to protect against man-in-the-middle (MITM) attacks.

* This ARM does not include support for HTTPS connections from the master to the
  agent, such as kick. Secure master to agent communication may be possible via
  a Kerberos/GSSAPI compatible message bus and MCollective, though this ARM does
  not provide any information on how to configure and use MCollective.

* This ARM does not describe how to correctly configure GSSAPI or otherwise
  setup Kerberos infrastructure.

Description
-----------

This ARM covers three broad areas of work: web server configuration, Puppet
master enhancements, and Puppet agent enhancements.

### Web Server Configuration ###

Configuration of the apache2 web server is relatively straightforward. It
requires mod_auth_kerb and several additions to the virtual host configuration
file. This ARM will update example configuration files and related
documentation.

> **TODO**: list explicit doc pages and example configuration files to update.

Once successfully configured and IFF a request is authenticated, mod_auth_kerb
will place the Kerberos principle name in the REMOTE_USER environment variable
where Puppet can retrieve it.

> **TODO**: there is a known issue with non-local apache hosts that forward
requests to the master. I have been unable to copy the REMOTE_USER value to an
HTTP X header.

> **TODO**: add nginx details.

### Puppet Master Enhancements ###

Two Puppet master enhancements are required: changes to the `Network::Http::Rack`
module to support the `REMOTE_USER` environment variable, and adding a hostname
look up facility.

#### REMOTE_USER Support ####

`Network::Http::Rack#extract_client_info` extracts authentication information
from incoming requests. It looks at the `HTTP_X_CLIENT_DN` and
`HTTP_X_CLIENT_VERIFY` environment variables and uses them to deter min a
"`:node`" name and an "`:authenticated`" boolean value. The existing behavior
for client certificate authentication remains unchanged. When certificate
authentication fails the master newly checks the `REMOTE_USER` environment
variable to authenticate the agent. If it can look up a hostname based on the
value of `REMOTE_USER` then it uses the hostname as `:node` and sets
`'authenticated` to `true`.

The following table summarizes the possible input states and their corresponding
results. (The state names below are for this document only and are not part of
the implementation.)

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
client certificate is ignored and this state is otherwise treated as `KERB`.

> **TODO**: should UNVERIFIED\_W\_KERB generate a warning? Yes.

The `VERIFIED_W_KERB` state occurs when an agent successfully authenticates via
Kerberos and also provides a verifiable client certificate. To maintain backward
compatibility in this case the `REMOTE_USER` environment variable is ignored and
this state is otherwise treated as `VERIFIED`.

> **TODO**: should VERIFIED\_W\_KERB generate a warning? Yes.

#### Hostname Look Up ####

Once the Kerberos principle has been retrieved from `REMOTE_USER` it is
translated to a hostname via an LDAP look up service. It is anticipated that this
will be an external service analogous to the external node classifier. Using an
external service facilitates using credentials other than the puppet account for
the LDAP look up. It also facilitates customization and restart / cache flush of
the look up service.

> **QUESTION**: Should we follow the example of the external node classifier? Or
is there something simpler or more "modern" that we should consider?

In the current prototype implementation the service is "mocked" with a simple
Ruby class that uses ruby-net-ldap to do a Windows Active Directory specific
query. As a convenience while prototyping, configuration settings provide the
LDAP account and password.

### Puppet Agent Enhancements ###

Two agent changes are required: implementation of the SPNEGO protocol in
`Puppet::Network::HTTP::Connection`, and support for certificate-less agents.

#### SPNEGO Support ####

Adding SPNEGO support based on the gssapi gem requires adding an appropriately
formatted `Authorization` header to agent HTTP requests. The master responds
with an `www-authenticate` header which is then passed back to GSSAPI. This may
repeat for several iterations before authentication completes. In the current
prototype a boolean configuration setting controls use of SPNEGO.

**Note**: The agent still requires that the server certificate be verified to
prevent MITM attacks.

#### Certificate-less Agents ####

Puppet agent normally requires the existence of a signed agent certificate. The
`Puppet::SSL::Host` class manages the lifecycle of this certificate. When
configured for Kerberos authentication the agent runs without a certificate. The
`Host` class will be modified to support both modes of operation. In the current
prototype a configuration setting turns several methods of `Host` into no-ops.

Testing and Evaluation
----------------------

> **TODO**: fill out testing and evaluation

* Spec tests.
* Unit tests.
* Command line example of using curl to test master configuration.
* Etc....


Alternatives and Recommendation
-------------------------------

We explored an alternate implementation that used Kerberos principle names
directly as "node" names. A typical Kerberos principle has the form
host$@EXAMPLE.COM. Supporting the mixed case and $ character required changes to
the Puppet AuthStore and was expected to lead to further complications with e.g.
file system support for the $. The current approach of using a hostname look up
service proved much simpler.


Risks and Assumptions
---------------------

* Implementor availability.

* Requirements for broader platform support will significantly slow development.
  The initial target of Linux agents integrated with Active Directory is
  probably the most common configuration that would benefit from this ARM.

> **TODO**: add more Risks and Assumptions ...


Dependencies
------------

This ARM depends on the gssapi gem which in turn depends on the ffi gem. The
hostname look up capability depends on the ruby-net-ldap gem.

These dependencies are only required if the features described above are enabled
via configuration settings.

Impact
------

### kick ###

As described above HTTPS based master -> agent requests are not secured with
certificate-less agents, so this feature prevents the use of kick over HTTP.
MCollective is expected to replace HTTPS for master -> agent communication in
the future.

### MCollective ###

MCollective should work if it uses a Kerberos capable message bus. However, no
research has yet been done on MCollective.

> **QUESTION**: Is there an MCollective expert who could outline the MCollective
road map and how it would interact with Kerberos infrastructure?

### Backward Compatibility ###

This enhancement should be backward compatible with most existing installations.
There are a few corner cases where behavior might change. For example if an
agent's certificate lapses and the web infrastructure is setting the REMOTE_USER
environment variable. When the certificate lapses the REMOTE_USER variable will
newly have an effect. These types of corner cases are really miss-configurations
and are an acceptable risk.

### Security ###

> **TODO**: Flesh out Security

###  Performance and Scalability ###

> **TODO**: Flesh out Performance and Scalability

* Latency for contacting Kerberos domain controller during authentication.

* Authenticate on each request.
    * Mitigated by existing inefficiency of `Connection: close`  behavior.
    * Mitigated by future migration to MCollective.

* Reverse proxy support - Based on current experimentation it does not seem
  possible to place the principle name into an HTTP extension header as we do
  for the client certificate CN. This needs further investigation, but it could
  be a limitation of mod_auth_kerb.

### Portability ###

* ffi includes native code which could limit portability.

### Future work ###

* SSPI API support for Windows agents. Is there a gem for SSPI interop?

* Support and testing with MIT Kerberos domain controllers / LDAP schemas.

* Reverse proxy support (need to figure out how to get REMOTE_USER into an HTTP
  header)

* Describe how Kerberos "channel binding" could be used to prevent MITM without
  a master certificate. Channel binding could also eliminate the need to
  authenticate on each request. OTOH channel binding is a complex topic and not
  well understood by the authors so perhaps it should not be mentioned here....

