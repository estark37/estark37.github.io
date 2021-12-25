---
layout: post
title:  "When a web PKI certificate won't cut it"
---

In recent years, setting up a public HTTPS website has gotten easier
and easier, thanks to widespread automated certificate management,
free certificates, inexpensive CDN support, and other
developments. However, for the most part, these advancements – and the
web PKI in general – are designed for publicly accessible
websites. That is, a website with a publicly resolvable domain name
can undergo domain name validation to get an HTTPS certificate. You
can also get an HTTPS certificate for a public IP address, but this
type of certificate is much more rare and less widely supported than
certificates for public domain names. What you cannot do is get a
publicly trusted HTTPS certificate for a non-public domain name (such
as an intranet hostname) or a reserved private network or localhost IP
address (such as 127.0.0.1). That is, a certificate authority like
Let’s Encrypt or DigiCert will not be able to provide you with an
HTTPS certificate for `foo.test` or `192.168.0.1` that works with an
out-of-the-box client like a major web browser. This is because
there’s no way for the certificate authority to validate that you are
the true owner of such a name; by definition, there is no such concept
of the true owner of such a name.

This deficiency affects a variety of use cases, from intranet hosts in
an enterprise to media servers, IoT devices, routers, and other
devices commonly found on home networks. This is problematic for
several reasons:

* First, there are many use cases where non-public hosts run web
  services that need authenticated encryption just as much as a public
  host does. Private networks are not generally secure: consider the
  possibility of an unsecured wifi network (e.g., a coffee shop), a
  weakly secured wifi network (e.g., home network with weak password
  or outdated encryption settings), or a wiretap. Web services
  accessible on such networks benefit from HTTPS as much as a public
  web service does.
* Second, even in the case of a truly secure private network, there is
  no way for a web browser or other client to know that the network is
  secure. TLS/HTTPS is the security language of the web. If a browser
  connects to a server with TLS, then the browser considers the
  connection secure, and otherwise it doesn’t. In the case of a
  non-public domain name or IP address, the browser must assume it is
  using an unencrypted connection. And any unencrypted web service,
  regardless of the sensitivity of the data traversing that
  connection, places constraints on how web browsers treat
  encryption. This is similar to arguments for why it’s
  [important](https://twitter.com/estark37/status/1417703310258184194)
  for even “unimportant” or “non-sensitive” public websites to support
  HTTPS.

Currently, there’s no canonical way for non-public web services to
support HTTPS, but there are a variety of approaches that have been
discussed and/or implemented, with various pros and cons. In this
post, I’ll survey some of these, especially focusing on a few recent
developments that are in vogue. In this post, I’m focusing mostly on
secure connection establishment, not on naming or discovery
mechanisms. There’s a separate post to be written on naming and
discovery, covering e.g. mDNS and other ways to discover and name
services that don’t rely on the global DNS or public IP addresses. And
I’ll also note that this list of approaches for authenticating
non-public services isn’t comprehensive; this post is mostly meant to
cover approaches that I see coming up most often in discussions on
this topic recently.

## Locally installed CAs

Everything I said above about HTTPS certificates applies to publicly
trusted CAs, the CAs that are trusted out-of-the-box by major web
clients. But for most web clients, users can customize the set of
trusted CAs, including installing their own CAs that are mostly free
to issue whatever certificates they’d like, including to non-public
domain names or private IPs.

This approach is commonly used in enterprise environments, allowing
managed devices to make secure connections to e.g. intranet hostnames
with certificates issued by an enterprise CA that is installed on all
end-user devices in the enterprise. It’s also a common approach for
[development use
cases](https://github.com/FiloSottile/mkcert). However, it’s generally
viewed as too fiddly, technical, and error-prone for consumer use
cases like a home media server. This is becoming even more true as
some clients are starting to add more
[friction](https://httptoolkit.tech/blog/android-11-trust-ca-certificates/)
for installing locally trusted CAs (and for good reason – which is a
topic for another post).

## The Plex method

Plex is a media platform that, among other things, provides a web app
that lets users stream media from servers on their home
networks. Filippo Valsorda has a classic
[writeup](https://blog.filippo.io/how-plex-is-doing-https-for-all-its-users/)
of how Plex establishes HTTPS connections to these local media
servers. I won’t rehash the whole writeup here, but the gist of it is
that Plex provisions each user’s local media servers with a regular
HTTPS wildcard certificate for `*.<user id>.plex.direct`. Browsers
access the local media servers via hostnames like `1-2-3-4.<user
id>.plex.direct`, which dynamically resolves to `1.2.3.4` (where
`1.2.3.4` would in reality would be the IP address of the media server
on the user’s local network).

The Plex approach is generally seen as an elegant and secure way to
establish encrypted connections to devices on a private
network. However, it suffers from one major
[drawback](https://github.com/WICG/private-network-access/issues/23):
it doesn’t work in the presence of overzealous home routers that block
DNS resolutions to private IPs. This is a common security feature in
home routers, meant to protect against [DNS rebinding
attacks](https://en.wikipedia.org/wiki/DNS_rebinding), and there’s no
workaround; it causes Plex to fall back to unencrypted http://.

## WebRTC and WebTransport

There are some lesser-known corners of the web platform that allow
clients to connect to servers with a given certificate hash, treating
the connection as secure even if the certificate doesn’t chain to a
trusted certificate authority. This mechanism has been around for a
while in the form of WebRTC, a suite of protocols designed for
peer-to-peer communication on the web. In WebRTC, connections are
[authenticated](https://www.rfc-editor.org/rfc/rfc8826#section-4.3.2.3)
with self-signed certificates that applications can verify themselves
outside the web PKI. Depending on your perspective, this is either a
cool feature for talking to peers that can’t obtain publicly trusted
HTTPS certificates, or a major loophole in the web browser security
model.

A more recent specification called
[WebTransport](https://w3c.github.io/webtransport/) (not yet widely
supported) allows web applications to establish [HTTP/3-based
connections](https://datatracker.ietf.org/doc/html/draft-ietf-webtrans-http3/)
to participating servers. WebTransport connections can use normal web
PKI certificates, or for talking to devices that don’t have normal
certificates, a certificate hash that doesn’t need to chain to a
certificate authority and can be verified by the application, as in
WebRTC.

The major downside of WebRTC and WebTransport are that they are
designed for specific use cases, such as streaming peer-to-peer media,
and they can be quite cumbersome to adapt for other use cases. A
developer can’t simply set up a regular HTTP server, install a
certificate, and build a website that accesses that server via WebRTC
or WebTransport. Instead, the server has to support particular
protocols, and the web client has to access data and resources over
these protocols. For example, suppose a website wants to load an image
from a web server on a private network. The website can’t simply do so
via `<img src=”https://local-web-server-name/image.jpg”/>`; instead,
the local web server has to, for example, support HTTP/3 and
WebTransport, and the developer has to write JavaScript code to make a
WebTransport connection, and then define and implement a protocol on
top of the WebTransport connection to fetch the image resource and
dynamically insert it into the page. This might get easier over time
as better web server and library support matures, but at the moment
it’s quite far off the well-lit path.

Even with better ecosystem support, any use case that shoehorns
something HTTP-like into WebRTC or WebTransport is going to end up
with some weird properties. For example, resources fetched in this way
won’t benefit from caching or other HTTP features built into the
browser. Even the security properties are a bit weird; in fact, the
WebTransport spec editors are currently in the process of
incorporating a nuanced
[discussion](https://github.com/w3c/webtransport/pull/375) about how
certificate hashes compare to typical public web PKI certificates in
security.

If you squint, WebRTC and WebTransport can be seen as representatives
from a class of approaches that use a public web service to bootstrap
a secure connection to a non-public web service. The Plex approach
also uses a public website for bootstrapping, though there’s a subtle
difference. In Plex, the public website is used for discovery: it
tells users what name to use to access their non-public web services,
and thereafter those hostnames are authenticated in the normal web PKI
way. In contrast, when using something like WebRTC or WebTransport,
the public website is used to boostrap authentication: it tells the
browser what certificate should be used to authenticate a non-public
web server. While pretty technical in nature, this distinction between
discovery and authentication can be rather important for usability,
depending on the use case. For example, one could imagine a user
bookmarking a Plex server on their local network and later accessing
it directly while offline, without being able to access the public
website for bootstrapping. This kind of offline access model doesn’t
apply very cleanly to WebRTC or WebTransport, where the user needs an
online publicly accessible website to make the WebRTC or WebTransport
connection to the server on the private network.

## Trust-on-first-use

Most of the above methods can be implemented today; they don’t rely on
browsers implementing any new technology. (The exception is
WebTransport, which is still in development and not widely supported
by major web browsers yet.)

There are some different models that either browsers or device vendors
or both could adopt but which haven’t been fully explored or
implemented yet. A
[document](https://docs.google.com/document/d/170rFC91jqvpFrKIqG4K8Vox8AL4LeQXzfikBQXYPmzU/edit#)
has been circulating for a while (I don’t know who wrote it,
unfortunately) that proposes a trust-on-first-use (TOFU) approach for
this problem. In this model, the browser shows a special UI instead of
a typical certificate error warning when it encounters a self-signed
certificate for a non-public IP address or domain name. After the user
accepts this special warning, the browser remembers the certificate or
key and doesn’t show the warning again. As a corollary, the browser
must expand its origin concept to include the remembered key or
certificate as part of the origin. Otherwise, two different services
could claim the same non-unique name and attack each other or pollute
each others’ application state.

Personally I find this approach a bit unsatisfying. It dumps a lot of
nuanced security reasoning onto the user. It discourages key rotation;
the UX degrades with shorter-lived keys. And it’s not clear how it
would apply to loading subresources from the local network (as opposed
to loading a top-level page from the local network). However, it has
one major upside: browsers can adopt it without requiring much work
from device vendors. It’ll work out of the box with any device that is
currently using a self-signed certificate.

## Shared secrets

The final approach I’ll discuss is rather aspirational. Various people
have [proposed](https://httpslocal.github.io/proposals/#approach-2)
that browsers support a special mode by which a user can enter a
secret that is engraved or displayed on the device to which they want
to connect. This secret can be used to bootstrap a secure connection
via a
[PAKE](https://en.wikipedia.org/wiki/Password-authenticated_key_agreement),
which can be integrated into TLS. As with a TOFU model, the browser
would need to remember the secret for a particular origin and
incorporate something about the secure connection into the origin
concept to avoid polluting origins cross-application when a non-unique
name is reused. Overall, I think this approach is a better UX than the
TOFU model, but it’s still a bit awkward for cross-origin requests
(e.g., a public website loading a resource from a device on a private
network). In recent years, browsers have moved away from trying to
communicate these cross-origin trust decisions to users because
they’re so complex and don’t match users’ mental models of how
websites work.

Perhaps the most interesting thing about the shared secret/PAKE
approach is that it would require device vendors to do some pretty
significant implementation work. It remains unclear if device vendors
have the right incentives and resources to do this work – for a shared
secret approach or for any other method that would be non-trivial for
them to adopt. Some device vendors have even claimed that their
devices are too underpowered to do any form of TLS at all. If we
accept that claim, all of these approaches are dead in the water for
such devices. It’s also unclear what the plan would be for the
gazillions of non-updateable devices already in the field. Ultimately
device vendor incentives are quite possibly the biggest hurdle for
tackling the problem of how to securely connect to devices with
non-public names, rather than the technical challenges.