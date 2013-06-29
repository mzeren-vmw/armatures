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

* Support Windows/Active Directory "service account" principle names that end in
  '$'.

* Allow the majority of agent functionality to operate securely without a client
  certificate -- eliminating the need to deploy and secure client certificate
  authority infrastructure.

* Document puppet master and agent configuration necessary to enable this
  feature.

* Document apache2 and mod_auth_kerb configuration necessary to enable this
  feature.

* Allow Kerberos based authentication to co-exist with client certificate based
  authentication on the same apache2 server and port.

* Introduce no new hard dependencies. The gssapi and ffi gems are required on
  both master and agent only when this feature is configured.

Non-Goals
---------

* This ARM does not provide any information on how to correctly configure
  GSSAPI/Kerberos.

* This ARM does not provide a general purpose "pluggable" authentication
  framework.

* This ARM does not include support for the Windows SSPI API, though that would
  likely be a straightforward extension of this work.

* This ARM will not support the webrick web server. [The ability to support
  nginx is an open question at this time].

* This ARM does not include a general purpose Ruby SPNEGO implementation, though
  it should be possible to switch to such a general purpose facility should it
  become available in the future.

* This ARM continues to rely on the agent verifying a master certificate during
  the SSL handshake to protect against man-in-the-middle (MITM) attacks. (See
  Channel Binding below.)

* This ARM does not include support for protecting connections from the master
  to the agent against MITM attacks. (See Channel Binding below).

Motivation
----------

TODO: move this above Goals/Non-Goals? Also give a more readable, persuasive
overview of the background, and motivation...

This ARM is motivated by:

1) The security requirement to separate agent authentication secret management
   (CA/domain controller) from agent configuration management (puppet).

2) The security requirement to minimize the number of authentication systems.

3) The operational efficiency of using existing Kerberos infrastructure in
   lieu of deploying and securing an external certificate authority for puppet.

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

Once successfully configured and IFF authentication succedes mod_auth_kerb will
place the Kerberos principle name in the REMOTE_USER environment variable where
puppet can retrieve it.

TODO: which doc pages?

TODO: nginx - at least spend a few hours seening what's possible.

Puppet Master Enhancements
--------------------------

Puppet master enhancemnts consist of two key changes: support for the
REMOTE_USER environment variable and changes to AuthStore to support Kerberos
principle names.

REMOTE_USER Support
-------------------

When a request arrives authenication information is available in the
HTTP_X_CLIENT_DN, HTTP_X_CLIENT_VERIFY, and REMOTE_USER
environment variables. The following states are possible:

  DN VERIFY USER STATE
  --|------|----|-----
   N      N  n/a NONE
   Y      N  n/a UNVERIFIED
   Y      Y  n/a VERIFIED
   N      N    Y KERB
   Y      N    Y UNVERIFIED_KERB
   Y      Y    Y VERIFIED_KERB

The NONE state is not relevant to this ARM. The NEW and SSL states are part of
the traditional client certificate based agent lifecycle and will not be changed
by this ARM. The KERB state is a new state. In this case we have only the REMOTE_USER environment variable. 

TODO: investigate nginx support.

REQUIRED -- Describe the enhancement in detail: Both what it is and,
to the extent understood, how you intend to implement it.  Summarize,
at a high level, all of the interfaces you expect to modify or extend,
including APIs, command-line switches, network protocols,
and file formats.  Explain how failures in applications using this
enhancement will be diagnosed, both during development and in
production.  Describe any open design issues.

This section will evolve over time as the work progresses, ultimately
becoming the authoritative high-level description of the end result.
Include hyperlinks to additional documents as required.

Testing and Evaluation
----------------------

What kinds of test development and execution will be required in order
to validate this enhancement, beyond the usual mandatory unit tests?
Be sure to list any special platform or hardware requirements.

What criteria should people use to evaluate the alternatives? If
there are suggestions you have as the author for helping readers
decide between alternatives (or whether to the ARM should be
implemented at all), build a decision tree here.

Alternatives and Recommendation
-------------------------------

Did you consider any alternative approaches or technologies?  If so
then please describe them here and explain why they were not chosen.

Describe which, if any, of the alternatives you recommend and why
you prefer it. This could walk through the Author's use of the
"Evaluation" decision tree explaining the rationale.

* Kerber-ized SSL - find link from Hadoop mailing list where they dropped KSSL.

Risks and Assumptions
---------------------

Describe any risks or assumptions that must be considered along with
this proposal.  Could any plausible events derail this work, or even
render it unnecessary?  If you have mitigation plans for the known
risks then please describe them.

* multiple request transmits due to multiple token exchanges in SPNEGO.

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
