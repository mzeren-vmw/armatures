ARM-15: Agent Kerberos
======================

Summary
-------

Allow agent authentication via Kerberos using GSSAPI and SPNEGO as an
alternative to puppet agent certificates.

Goals
-----

This ARM has the following goals:

* Add a minimal set of enhancements to the puppet master and agent
  infrastructure to allow master/agent mutual authentication via
  SPNEGO/Kerberos.

* Allow the majority of agent functionality to operate securely without a client
  certificate -- eliminating the need to deploy and secure certificate authority
  infrastructure for puppet clients

* Allow Kerberos based authentication to co-exist with client certificate based
  authentication on the same apache2 server and port.

* Introduce no new hard dependencies. The gssapi and ffi gems are required on
  both master and agent only when this feature is configured.

* Support Windows/Active Directory "service account" principle names that
  contain '$' characters.

* Document puppet master and agent configuration necessary to enable this
  feature.

* Document apache2 and mod_auth_kerb configuration necessary to enable this
  feature.

Non-Goals
---------

* This ARM does not provide a general purpose "pluggable" authentication
  framework.

* This ARM does not resolve the long standing limitation that node names be
  lower case and merely works around the fact that Kerberos principle names are
  case sensetive. [TODO: add ticket #]

* This ARM will not support the webrick web server. [TODO add nginx here if
  necessary].

* This ARM does not include support for the Windows SSPI API, though that would
  likely be a straightforward extension of this work.

* This ARM does not include a general purpose Ruby SPNEGO implementation, though
  it should be possible to switch to such a general purpose facility should it
  become available in the future.

* This ARM continues to rely on the agent verifying a master certificate during
  the SSL handshake to protect against man-in-the-middle (MITM) attacks. (See
  HTTPS Channel Binding below.)

* This ARM does not include support for protecting HTTPS connections from the
  master to the agent against MITM attacks. (See HTTPS Channel Binding below).
  Secure master to agent communication may in theory be achieved via a
  Kerberos/GSSAPI compatible message bus and MCollective, though this ARM does
  not provide any information on how to configure and use MCollective.

* This ARM does not provide any information on how to correctly configure
  GSSAPI/Kerberos.

Motivation
----------

[TODO: rewrite as persuasive prose instead of bullets.]
[TODO: move Motivation above Goals/Non-Goals?]

This ARM is motivated by:

1) The security requirement to separate agent authentication secret management
   (certificate authority) from agent configuration management (puppet master).

2) The security requirement to minimize the number of authentication systems
   within an organization.

3) The operational efficiency gained by using an existing Kerberos
   infrastructure in lieu of deploying and securing an external certificate
   authority for puppet.

[TODO: provide example of only one privileged operation to add a machine to the
Kerberos domain, instead of two priviledged operations one for Kerberos and
another for puppet/certificate authority.]

This new capability will benefit organizations that have Kerberos infrastructure
available on puppet agent hosts.

Description
-----------

This ARM covers three broad areas of work: web server configuration, puppet
master enhancements, and puppet agent enhancemnets.

Web Server Configuration
------------------------

Configuration of the apache2 web server to support Kerberos authentication is
relatively straighforward. After configuring apache normally the additional
steps involve installing and enabling mod_auth_kerb and adding several settings
in the puppetmaster virtual host configuration file. This ARM
will update example configuration files and related documentation.

Once successfully configured and IFF a request is authenticated_auth_kerb will
place the Kerberos principle name in the REMOTE_USER environment variable where
puppet can retrieve it.

[TODO: list explicit doc pages to update]

[TODO: can nginx provide Kerberos SPNEGO?]

Puppet Master Enhancements
--------------------------

Two puppet master enhancemnts are required: changes to Network::AuthStore to
support Kerberos principle names, and changes to Network::Http::Rack to
support REMOTE_USER environment variable.

Kerberos Principles and Case Sensetivity
----------------------------------------

Puppet has an existing limitation of supporting only lowercase agent
"certificate names". This limitation is enforced in the puppet agent
configuration code. Non-lowercase names will be rejected. Master-side code also
depends on this restriction. For example agent certificate names are used as
file names on case insensetive file systems. Puppet has been successful despite
this limitiation because most agent hosts use case insensetive DNS names.
Additionally since puppet typically acts as a certificate authority agent
configuration files can give hosts alternate, lowercase "certificate names" as
necessary.

Kerberos principle names, howerver, are case sensetive and mod_auth_kerb will
provide the full mixed case name in the REMOTE_USER environment variable.

This ARM intentionally avoids including or taking a dependency on a fix for this
limitation by downcasing all Kerberos principle names at the puppet master. This
is unlikely to be an operational issue as it is unusual to have two principle
names that differ only in case. This ARM requires that the Kerberos domain and
any certificate authorities do not allow lowercased name collisions.

Active Directory Service Accounts and $
---------------------------------------

Windows Active Directory service account names traditionally end in '$'.
Computer accounts are service accounts so in a Active Directory environment all
puppet agent principle names will end in $. For example:

  myagenthost$@EXAMPLE.COM

The Networkd::AuthStore module in puppet currently does not permit $ in "node"
names. The currently proposed solution is to extend the ':opaque' name pattern
to support $.

In addition, agent names are used in file system names. The code for
manipulating these file system names will be tested and enhanced to support $ as
necessary.

REMOTE_USER Support
-------------------

Network::Http::Rack#extract_client_info extracts authentication information from
incoming requests. It looks at the HTTP_X_CLIENT_DN, HTTP_X_CLIENT_VERIFY, and
uses them to determin a ":node" name and ":authenticated" boolean value. The
following table summarizes the possible input states and their corresponding
results:

 STATE             DN            VERIFY  REMOTE_USER    :node          :authenticated
 -----------------|-------------|-------|--------------|--------------|--
 NONE              n/a           n/a     n/a            nil            N
 UNVERIFIED        a.example.com FAILED  n/a            nil            N
 VERIFIED          a.example.com SUCCESS n/a            a.example.com  Y
 KERB              n/a           FAILED  a$@EXAMPLE.COM a$@example.com Y
 UNVERIFIED_W_KERB a.example.com FAILED  a$@EXAMPLE.COM a$@example.com Y
 VERIFIED_W_KERB   a.example.com SUCCESS a$@EXAMPLE.COM a.example.com  Y

The NONE state is not relevant to and will not be changed by this ARM.

The UNVERIFIED and VERIFIED states are part of the traditional client
certificate based agent lifecycle and will not be changed by this ARM.

The KERB state occurs when an agent successfully authenticates via Kerberos and
does not provide a client certificate. This is the typical state for agents
authenticating via Kerberos. The request is considered authenticated and the
node is the downcased value of the REMOTE_USER environment variable.

The UNVERIFIED_W_KERB state occurs when an agent successfully authenticates via
Kerberos and also provides an unverifiable client certificate. In this case the
client certificate is ignorned and this state is otherwise treated as KERB.
[TODO: should we provide a warning in this case?]

The VERFIFIED_W_KERB state occurs when an agent successfully authenticates via
Kerberos and also provides a verfiable client certificate. To maintain backward
compatibility in this case the REMOTE_USER environment variable is ignored and
this state is otherwise treated as VERIFIED. [TODO: should we provide a warning
in this case?]

Agent Changes
-------------

[TODO: fill out agent changes]

* SPNEGO extensions to Net::Http. Lazy loading of gssapi and ffi.

* Allow agent to operate without a certificate.

* ...

Testing and Evaluation
----------------------

[TODO: fill out testing and evaluation]

* Is there a null GSSPI provider that can be installed instead of bringing up a
  real KDC/ADC? Can it be installed in a loopback way?

* Is there any value in mocking out the gssapi gem? Could we combine that with
  mod_rewrite to mock mod_auth_kerb? Or mabye

* Test master using curl to make all of the basic requests: node, catalogc, plugins, etc.



Alternatives and Recommendation
-------------------------------

[TODO: remove optional section?]

It is always possible to use puppet's integrated certificate authority or to use
an external certificate authority instead of using the enhancements provided by
this ARM. In fact all of these mechanisms may be used concurently on the same
puppet master. If however an organization has a requirement for separate agent
authentication management and already has a Kerberos/GSSPI infrastructure in
place then using only Kerberos can improve security and lower operational
overhead. (See also Motivation above.)

Risks and Assumptions
---------------------


* AuthStore bugs and breaking changes

* multiple request transmits due to multiple token exchanges in SPNEGO.

* authentication on each request instead of on each connection

* channel binding

* 

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
