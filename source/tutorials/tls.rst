Encrypting Syslog Traffic with TLS (SSL) [short version]
========================================================

*Written by* `Rainer Gerhards  <http://www.gerhards.net/rainer>`_
*(2008-05-06)*

Abstract
--------

**In this paper, I describe how to encrypt**
`syslog <http://www.monitorware.com/en/topics/syslog/>`_ 
**messages on the network.**
Encryption is vital to keep the confidiental content of
syslog messages secure. I describe the overall approach and provide an
HOWTO do it with `rsyslog's <http://www.rsyslog.com>`_ TLS features. 

Please note that TLS is the more secure successor of SSL. While people
often talk about "SSL encryption" they actually mean "TLS encryption".
So don't look any further if you look for how to SSL-encrypt syslog. You
have found the right spot.

This is a quick guide. There is a more elaborate guide currently under
construction which provides a much more secure environment. It is highly
recommended to `at least have a look at it <rsyslog_secure_tls.html>`_.

Background
----------

**Traditional syslog is a clear-text protocol. That means anyone with a
sniffer can have a peek at your data.** In some environments, this is no
problem at all. In others, it is a huge setback, probably even
preventing deployment of syslog solutions. Thankfully, there are easy
ways to encrypt syslog communication. 

The traditional approach involves `running a wrapper like stunnel around
the syslog session <rsyslog_stunnel.html>`_. This works quite well and
is in widespread use. However, it is not thightly coupled with the main
syslogd and some, even severe, problems can result from this (follow a
mailing list thread that describes `total loss of syslog messages due to
stunnel
mode <http://lists.adiscon.net/pipermail/rsyslog/2008-March/000580.html>`_
and the `unreliability of TCP
syslog <http://rgerhards.blogspot.com/2008/04/on-unreliability-of-plain-tcp-syslog.html>`_).

`Rsyslog supports syslog via GSSAP <gssapi.html>`_\ I since long to
overcome these limitatinos. However, syslog via GSSAPI is a
rsyslog-exclusive transfer mode and it requires a proper Kerberos
environment. As such, it isn't a really universal solution. The
`IETF <http://www.ietf.org/>`_ has begun standardizing syslog over plain
tcp over TLS for a while now. While I am not fully satisfied with the
results so far, this obviously has the  potential to become the
long-term solution. The Internet Draft in question, syslog-transport-tls
has been dormant for some time but is now (May of 2008) again being
worked on. I expect it to turn into a RFC within the next 12 month (but
don't take this for granted ;)). I didn't want to wait for it, because
there obviously is need for TLS syslog right now (and, honestly, I have
waited long enough...). Consequently, I have implemented the current
draft, with some interpretations I made (there will be a compliance doc
soon). So in essence, a TLS-protected syslog transfer mode is available
right now. As a side-note, Rsyslog is the world's first implementation
of syslog-transport-tls.

Please note that in theory it should be compatible with other, non IETF
syslog-transport-tls implementations. If you would like to run it with
something else, please let us know so that we can create a compatibility
list (and implement compatbility where it doesn't yet exist). 

Overall System Setup
--------------------

Encryption requires a reliable stream. So It will not work over UDP
syslog. In rsyslog, network transports utilize a so-called "network
stream layer" (netstream for short). This layer provides a unified view
of the transport to the application layer. The plain TCP syslog sender
and receiver are the upper layer. The driver layer currently consists of
the "ptcp" and "gtls" library plugins. "ptcp" stands for "plain tcp" and
is used for unencrypted message transfer. It is also used internally by
the gtls driver, so it must always be present on a system. The "gtls"
driver is for GnutTLS, a TLS library. It is used for encrypted message
transfer. In the future, additional drivers will become available (most
importantly, we would like to include a driver for NSS).

What you need to do to build an encrypted syslog channel is to simply
use the proper netstream drivers on both the client and the server.
Client, in the sense of this document, is the rsyslog system that is
sending syslog messages to a remote (central) loghost, which is called
the server. In short, the setup is as follows:

**Client**

-  forwards messages via plain tcp syslog using gtls netstream driver to
   central server on port 10514

**Server**

-  accept incoming messages via plain tcp syslog using gtls netstream
   driver on port 10514

Setting up the system
---------------------

Server Setup
~~~~~~~~~~~~

At the server, you need to have a digital certificate. That certificate
enables SSL operation, as it provides the necessary crypto keys being
used to secure the connection. There is a set of default certificates in
./contrib/gnutls. These are key.pem and cert.pem. These are good for
testing. If you use it in production, it is very easy to break into your
secure channel as everybody is able to get hold of your private key. So
it is a good idea to generate the key and certificate yourself.

You also need a root CA certificate. Again, there is a sample CA
certificate in ./contrib/gnutls, named ca.cert. It is suggested to
generate your own.

To configure the server, you need to tell it where are its certificate
files, to use the gtls driver and start up a listener. This is done as
follows:

    ::

        # make gtls driver the default
        $DefaultNetstreamDriver gtls

        # certificate files
        $DefaultNetstreamDriverCAFile /path/to/contrib/gnutls/ca.pem
        $DefaultNetstreamDriverCertFile /path/to/contrib/gnutls/cert.pem
        $DefaultNetstreamDriverKeyFile /path/to/contrib/gnutls/key.pem

        $ModLoad imtcp # load TCP listener

        $InputTCPServerStreamDriverMode 1 # run driver in TLS-only mode
        $InputTCPServerStreamDriverAuthMode anon # client is NOT authenticated
        $InputTCPServerRun 10514 # start up listener at port 10514

This is all you need to do. You can use the rest of your rsyslog.conf
together with this configuration. The way messages are received does not
interfer with any other option, so you are able to do anything else you
like without any restrictions.

Restart rsyslogd. The server should now be fully operational.

Client Setup
~~~~~~~~~~~~

The client setup is equally simple. You need less certificates, just the
CA cert. 

    ::

        # certificate files - just CA for a client
        $DefaultNetstreamDriverCAFile /path/to/contrib/gnutls/ca.pem

        # set up the action
        $DefaultNetstreamDriver gtls # use gtls netstream driver
        $ActionSendStreamDriverMode 1 # require TLS for the connection
        $ActionSendStreamDriverAuthMode anon # server is NOT authenticated
        *.* @@(o)server.example.net:10514 # send (all) messages

Note that we use the regular TCP forwarding syntax (@@) here. There is
nothing special, because the encryption is handled by the netstream
driver. So I have just forwarded every message (\*.\*) for simplicity -
you can use any of rsyslog's filtering capabilities (like
epxression-based filters or regular expressions). Note that the "(o)"
part is not strictly necessary. It selects octet-based framing, which
provides compatiblity to IETF's syslog-transport-tls draft. Besides
compatibility, this is also a more reliable transfer mode, so I suggest
to always use it.

Done
~~~~

After following these steps, you should have a working secure syslog
forwarding system. To verify, you can type "logger test" or a similar
"smart" command on the client. It should show up in the respective
server log file. If you dig out your sniffer, you should see that the
traffic on the wire is actually protected.

Limitations
~~~~~~~~~~~

The RELP transport can currently not be protected by TLS. A work-around
is to use stunnel. TLS support for RELP will be added once plain TCP
syslog has sufficiently matured and there either is some time left to do
this or we find a sponsor ;).

Certificates
------------

In order to be really secure, certificates are needed. This is a short
summary on how to generate the necessary certificates with GnuTLS'
certtool. You can also generate certificates via other tools, but as we
currently support GnuTLS as the only TLS library, we thought it is a
good idea to use their tools.

Note that this section aims at people who are not involved with PKI at
all. The main goal is to get them going in a reasonable secure way. 

CA Certificate
~~~~~~~~~~~~~~

This is used to sign all of your other certificates. The CA cert must be
trusted by all clients and servers. The private key must be
well-protected and not given to any third parties. The certificate
itself can (and must) be distributed. To generate it, do the following:

#. generate the private key:

   ::

       certtool --generate-privkey --outfile ca-key.pem

   This takes a short while. Be sure to do some work on your
   workstation, it waits for radom input. Switching between windows is
   sufficient ;)

#. now create the (self-signed) CA certificate itself:

   ::

       certtool --generate-self-signed --load-privkey ca-key.pem --outfile ca.pem

   This generates the CA certificate. This command queries you for a
   number of things. Use appropriate responses. When it comes to
   certificate validity, keep in mind that you need to recreate all
   certificates when this one expires. So it may be a good idea to use a
   long period, eg. 3650 days (roughly 10 years). You need to specify
   that the certificates belongs to an authrity. The certificate is used
   to sign other certificates.

#. You need to distribute this certificate to all peers and you need to
   point to it via the $DefaultNetstreamDriverCAFile config directive.
   All other certificates will be issued by this CA.
   Important: do only distribute the ca.pem, NOT ca-key.pem (the
   private key). Distributing the CA private key would totally breach
   security as everybody could issue new certificates on the behalf of
   this CA.

Individual Peer Certificate
~~~~~~~~~~~~~~~~~~~~~~~~~~~

Each peer (be it client, server or both), needs a certificate that
conveys its identity. Access control is based on these certificates. You
can, for example, configure a server to accept connections only from
configured clients. The client ID is taken from the client instances
certificate. So as a general rule of thumb, you need to create a
certificate for each instance of rsyslogd that you run. That instance
also needs the private key, so that it can properly decrypt the traffic.
Safeguard the peer's private key file. If somebody gets hold of it, it
can malicously pretend to be the compromised host. If such happens,
regenerate the certificate and make sure you use a different name
instead of the compromised one (if you use name-based authentication). 

These are the steps to generate the indivudual certificates (repeat: you
need to do this for every instance, do NOT share the certificates
created in this step):

#. generate a private key (do NOT mistake this with the CA's private key
   - this one is different):

   ::

       certtool --generate-privkey --outfile key.pem

   Again, this takes a short while.

#. generate a certificate request:

   ::

       certtool --generate-request --load-privkey key.pem --outfile request.pem

   If you do not have the CA's private key (because you are not
   authorized for this), you can send the certificate request to the
   responsible person. If you do this, you can skip the remaining steps,
   as the CA will provide you with the final certificate. If you submit
   the request to the CA, you need to tell the CA the answers that you
   would normally provide in step 3 below.

#. Sign (validate, authorize) the certificate request and generate the
   instances certificate. You need to have the CA's certificate and
   private key for this:

   ::

       certtool --generate-certificate --load-request request.pem --outfile cert.pem \ --load-ca-certificate ca.pem --load-ca-privkey ca-key.pem

   Answer questions as follows: Cert does not belogn to an authority; it
   is a TLS web server and client certificate; the dnsName MUST be the
   name of the peer in question (e.g. centralserver.example.net) - this
   is the name used for authenticating the peers. Please note that you
   may use an IP address in dnsName. This is a good idea if you would
   like to use default server authentication and you use selector lines
   with IP addresses (e.g. "\*.\* @@192.168.0.1") - in that case you
   need to select a dnsName of 192.168.0.1. But, of course, changing the
   server IP then requires generating a new certificate.

After you have generated the certificate, you need to place it onto the
local machine running rsyslogd. Specify the certificate and key via the
$DefaultNetstreamDriverCertFile /path/to/cert.pem and
$DefaultNetstreamDriverKeyFile /path/to/key.pem configuration
directives. Make sure that nobody has access to key.pem, as that would
breach security. And, once again: do NOT use these files on more than
one instance. Doing so would prevent you from distinguising between the
instances and thus would disable useful authentication.

Troubleshooting Certificates
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

If you experience trouble with your certificate setup, it may be useful
to get some information on what is contained in a specific certificate
(file). To obtain that information, do 

::

    $ certtool --certificate-info --infile cert.pem

where "cert.pem" can be replaced by the various certificate pem files
(but it does not work with the key files).

Conclusion
----------

With minumal effort, you can set up a secure logging infrastructure
employing TLS encrypted syslog message transmission.

Feedback requested
~~~~~~~~~~~~~~~~~~

I would appreciate feedback on this tutorial. If you have additional
ideas, comments or find bugs (I \*do\* bugs - no way... ;)), please `let
me know <mailto:rgerhards@adiscon.com>`_.

Revision History
----------------

-  2008-05-06 \* `Rainer Gerhards`_ \*
   Initial Version created
-  2008-05-26 \* `Rainer Gerhards`_ \*
   added information about certificates

Copyright
---------

Copyright (c) 2008-2014 `Rainer Gerhards`_ and
`Adiscon <http://www.adiscon.com/en/>`_.

Permission is granted to copy, distribute and/or modify this document
under the terms of the GNU Free Documentation License, Version 1.2 or
any later version published by the Free Software Foundation; with no
Invariant Sections, no Front-Cover Texts, and no Back-Cover Texts. A
copy of the license can be viewed at
`http://www.gnu.org/copyleft/fdl.html <http://www.gnu.org/copyleft/fdl.html>`_.

[`rsyslog site <http://www.rsyslog.com/>`_\ ]
