---
layout: post
title:  'Splitting up trust'
---

When we think about browsing the web, we usually think about a web browser and a
web server communicating. But today’s (and yesterday’s) web browsers are often
far more complex than a single piece of client software. That single piece of
client software is supported by many services run by different entities. An
example of such a supporting service is a DNS resolver, or a telemetry service
operated by the browser vendor.

These browser-supporting services can introduce privacy challenges, which often
take a similar shape: the browser needs to talk to a service to look up or
report some information, but the end user doesn’t want the service to learn too
much from the information it receives. Sometimes each piece of reported
information is itself sensitive (e.g., a [capability URL](https://www.w3.org/TR/capability-urls/)
that a person visited). Sometimes each individual piece of reported information
is innocuous on its own, but we don’t want a service to be able to link requests
coming from the same person and build up a profile of that person over time.

Recently the browser world has seen a trend of systems that solve these privacy
challenges by splitting up trust among two or more servers, with each ideally
run by a separate organization. Instead of the browser trusting a single service
with sensitive data, the browser splits up the request among multiple servers,
and as long as one of the servers is behaving honestly, the user’s privacy is
preserved.

## Examples

Here are some examples of systems that split up trust among two or more servers,
where at least one is assumed to be honest, and how they apply to web browsers:

### Oblivious DNS

As web browsers have adopted DNS over HTTPS (DoH), which hides DNS queries from
the network, there has been increased interest in protecting user privacy
against recursive resolvers as well as the network. Within the realm of
practicality, the recursive resolver has to be able to see the query to service
it, but the resolver doesn’t need to know which query comes from a particular
user, nor that multiple queries come from the same user.

An initial proposal for so-called Oblivious DNS came from
[academia](https://odns.cs.princeton.edu/pdf/pets.pdf). These researchers
cleverly designed a system that didn’t require any changes to existing DNS
infrastructure. In this proposal, clients encrypt queries as domains under a new
TLD (`<encrypted query>.odns`, for example). The recursive resolver treats this
as a normal query, forwarding it to the authoritative `.odns` nameserver, which
decrypts it and forwards it on to the resolver for the real query. Thus the
recursive resolver is transformed into a proxy for DNS resolution without even
knowing. The recursive resolver sees only encrypted queries, which isn’t very
interesting information. The `.odns` resolver sees plaintext queries but can’t
link them to a particular user because they all come from recursive resolvers.

Why not just stick a SOCKS or CONNECT proxy in front of the recursive resolver,
you ask? At first glance this would seem to be a much simpler way to achieve
similar privacy properties: the proxy would see only encrypted data, and the
recursive resolver wouldn’t be able to distinguish which query is coming from
which user (within the anonymity set of the users using the same proxy). The
paper doesn’t address this explicitly, as far as I can tell, but first of all, I
suspect it was written before widespread DoH deployment, so setting up an
encrypted tunnel to the recursive resolver would have required changes to the
resolver. Secondly, even if DoH were in the picture, the client would need to
open a fresh connection for each query through the proxy to prevent the
recursive resolver from linking queries together, which would be a significant
performance hit.

A little bit later, Cloudflare, Apple, and Fastly designed and deployed a
variant of Oblivious DNS, specifically designed to work with DoH, and thus named
[Oblivious DoH](https://blog.cloudflare.com/oblivious-dns/) (ODoH). This ODoH
variant looks a bit similar to the SOCKS/CONNECT proxy strawperson I just
mentioned. That is, ODoH puts a proxy in front of the recursive resolver,
instead of transforming the recursive resolver into a proxy. However, if it were
actually a SOCKS or CONNECT proxy, then, as previously mentioned, the client
would need to open a fresh connection per query, or else the recursive resolver
would be able to link multiple queries coming in over the same connection. ODoH
instead encrypts queries at the application layer, allowing the proxy to pool
multiple requests from different users onto the same connection to the recursive
resolver. The proxy is therefore essentially an HTTP proxy where the requests
and responses just happen to contain encrypted data. Since the request/response
bodies are encrypted as they pass through the proxy, the proxy doesn’t see any
interesting user data, and the recursive resolver doesn’t see which user the
queries are coming from. Encrypting at the application layer means that the
recursive resolver must know how to decrypt queries and encrypt the responses,
but fortunately Cloudflare, one of the main drivers of ODoH, operates a popular
DNS resolver and can modify it however they see fit.

(I'll resist going on a tangent about academic researchers often making
interesting mispredictions about deployment feasibility: the original ODNS
researchers went to great lengths to avoid modifying existing DNS infrastructure
in the name of deployability, whereas it turned out that modifying a popular DNS
resolver was no hurdle at all. A topic for another post...)

### Oblivious HTTP

Oblivious HTTP (OHTTP) is a more recent
[proposal](https://www.ietf.org/archive/id/draft-thomson-http-oblivious-01.html)
that is basically a generalization of ODoH. It separates the protocol
(encrypting HTTP requests and responses and passing them through a proxy) from
the particular application (DNS queries) so that it can be used for other
applications. The example given in the proposal is browser telemetry
queries. Browser telemetry is how vendors measure how their product is
performing: for example, the browser might report how often a user clicks on a
particular button or how long it takes each webpage to load. While these
datapoints are not particularly sensitive in isolation, one could imagine
building up some interesting information about a user if one could see their
telemetry reports over time. Oblivious HTTP protects this information: as with
ODoH, the proxy sees only encrypted requests, and the origin server (the browser
telemetry server, in this example) doesn’t know which data came from which
user. As with ODoH, the advantage of encrypting HTTP requests and responses
instead of using connection-layer encryption is to allow multiple requests to be
pooled over the same connection.

### Prio

ODoH and OHTTP are, in some sense, cheap hacks (though I don’t mean that in a
derogatory sense). They are fairly easy to deploy, at least for the players
driving them, but they don’t give strong privacy guarantees. For example, in
ODoH, the recursive resolver still sees how often particular domain names are
resolved, even if it can’t tell who is resolving them. And for users, the
privacy is only as strong as the number of users who are sharing the same proxy.

Another approach is to use cryptographic protocols to achieve much stronger
privacy. One example of this is
[Prio](https://people.csail.mit.edu/henrycg/pubs/nsdi17prio/), a two-server
system for the browser telemetry use case. Prio allows browser vendors to
compute aggregate statistics over user telemetry reports without seeing the
actual reports -- as long as the two servers involved aren’t colluding.

There is probably a whole other long blog post involved in explaining how Prio
works, so here is an attempt at a very succinct and highly simplified
explanation. First imagine that there are a bunch of browsers who each hold some
secret value, and the browser vendor wants to know the sum of those values --
with the help of two non-colluding servers. Each client splits up its secret
value into two halves where neither half on its own contains any information
about the secret value (this is called a
[secret-sharing scheme](https://en.wikipedia.org/wiki/Secret_sharing)).
Conveniently, there exists ways to do this splitting such that if each server
receives a half from each client and sums those shares up, and then we sum the
the two sums from the servers, the result is the sum of all the client values --
exactly what the browser vendor wants to know, without the browser learning
anything about the individual client values.

This is the seed of how Prio works, but Prio adds quite a bit of additional
complexity to achieve two additional properties efficiently:

* Robustness against malicious clients sending invalid inputs that throw off the
  protocol. In Prio, clients send a type of
  [zero-knowledge proof](https://en.wikipedia.org/wiki/Zero-knowledge_proof)
  showing that their inputs are valid without revealing any information about the
  input itself.

* The ability to compute complex functions, not just the linear combinations of
  values that are permitted directly by the secret-sharing scheme. To accomplish
  this, the Prio clients pre-encode their input values using some math tricks
  such that the final value of interest can be constructed as a linear
  combination of the encoded values. These encoded values are used as the inputs
  to the secret-sharing scheme instead of the raw values, and the aforementioned
  proof technique can be used to show that the values are properly encoded.

Interestingly, Prio is, I think, the most complex of the systems I’m describing
in the post and yet also the most widely
deployed. [Firefox](https://blog.mozilla.org/security/2019/06/06/next-steps-in-privacy-preserving-telemetry-with-prio/)
has used it for browser telemetry, and
[various](https://covid19-static.cdn-apple.com/applications/covid19/current/static/contact-tracing/pdf/ENPA_White_Paper.pdf)
[companies](https://github.com/google/exposure-notifications-android/blob/master/doc/enexpress-analytics-faq.md)
have used it for COVID exposure notifications.

### Checklist

Phishing blocklists are the final example I’ll mention, and this is an
interesting case study because they’ve been deployed for a relatively long time,
and because there are a variety of deployment models currently in use by
different browsers. Phishing blocklists are lists of malicious webpages, often
compiled by browser vendors, and served to browsers to block when users try to
access them. Some browsers query plaintext URLs to the browser vendor to check
against the blocklists in real-time when loading a webpage, and some use a
[hashing protocol](https://developers.google.com/safe-browsing/v4) to obtain
some measure of privacy. Some
[browsers](https://www.zdnet.com/article/apple-will-proxy-safe-browsing-traffic-on-ios-14-5-to-hide-user-ips-from-google/)
use blocklists from third-party vendors, and the browser vendor operates a proxy
between the browser client and the third-party vendor.

None of these deployed approaches provide strong cryptographic privacy: in all
of them, a malicious server learns something from observing the blocklist
queries. A recent academic paper called
[Checklist](https://people.csail.mit.edu/henrycg/pubs/checklist/) proposes a new
two-server private information retrieval (PIR) protocol and tailors it
specifically for phishing blocklists. I wrote a lot of about PIR in a
[previous post](https://emilymstark.com/2021/04/28/the-fundamental-laws-of-private-information-retrieval.html),
so I won’t go into much detail here. In short, the browser sends queries to two
servers and learns whether a particular URL is on the phishing blocklist. As
long as the servers aren’t colluding, neither learns any information at all
about what URL the browser is querying.

## What's the threat model, anyway?

A common thread among these case studies is that most of these proposed systems
aren’t widely deployed in web browsers yet. There are many possible reasons that
they haven’t been deployed yet, but one to ponder is whether the costs are worth
the benefits -- even for the relatively cheap proposals like ODoH and OHTTP. And
this invites the question: What is the threat model for these systems? What
exactly are we protecting against?

One reason that the cost/benefit calculation is not straightforward is that, in
many (but not all) applications, the browser vendor is running the service that
receives the sensitive information. Generally, web browsers have a fairly
complex threat model with respect to their vendors. While many people would say,
for example, that they don’t want their browser vendor snooping on their
browsing history, those same people do inherently trust their browser vendor to
not be overtly malicious. In a world of autoupdating (that is, the world of most
modern browser users today), an overtly malicious browser vendor could push
malicious code, potentially even stealthily to a subset of targeted users. We
can [dream](https://wiki.mozilla.org/Security/Binary_Transparency) about a world
in which such a malicious update might be detectable, but that’s not the world
we live in today, by a long shot.

This state of affairs doesn’t mean that users must trust their browser vendors
fully, but they certainly must trust them to some extent, and that “to some
extent” is hard to pin down. Is it reasonable to assume, for example, that a
rogue insider (or attacker exploiting a bug in the vendor’s server-side
infrastructure) can access sensitive data but can’t push malicious code? If the
vendor/attacker can’t push malicious code, could they instead serve data from an
endpoint that gets parsed
[vulnerably](https://chromium.googlesource.com/chromium/src/+/refs/heads/main/docs/security/rule-of-2.md#verifying-the-trustworthiness-of-a-source)
in a memory-unsafe language, or could they serve an experiment or telemetry
configuration that changes the browser’s behavior in an exploitable way? Could
an attacker withhold an autoupdate that fixes an exploitable security bug,
leaving users in a vulnerable state?

This is a complex threat model to reason about. The attacker is somewhere short
of fully malicious, but potentially more powerful than an honest-but-curious
attacker who follows protocols honestly but seeks to passively learn information
from the messages it receives during the protocol.

The threat model gets even more nuanced when you introduce multiple servers with
a non-collusion assumption. Usually, but not always, the idea is that the
multiple servers would be run by separate organizations. Both organizations must
be malicious or compromised for the privacy guarantees to be broken. Is this a
meaningful improvement, given the not-complete-but-probably-significant trust
that users must already place in their browser vendor? That depends on the
specific application (how malicious must each server be to compromise the
systems’ privacy guarantees? how could a somewhat-malicious browser vendor
access the same information on its own?). And of course it also depends on
factors such as the technical and social reputability of the organizations,
their legal jurisdictions, and what threats you are concerned about (rogue
insiders versus law enforcement, for example).

In some cases, the servers could actually be run by the same organization with
non-collusion assured via attestation, audits, or some other means -- each of
which has their own complex assumptions.

In short, threat modelling for a web browser against its vendor is really hard,
and it gets even harder when you introduce a non-collusion assumption among
multiple servers. Threat modelling is a lot simpler when the browser vendor is
not the attacker we’re concerned about (for example, querying a phishing
blocklist provided by someone other than the browser vendor, or ODoH). But even
when the browser vendor isn't in the picture, whether you trust two
organizations to not collude is still a complicated social, legal, and
technical question.

## Pitfalls

Related to this nuanced threat model, there are a few pitfalls I’ve noticed
people tend to fall into when proposing two-non-colluding-server approaches for
browser features:

### Sharing implementations and/or infrastructure

It may be less realistic to assume non-collusion if the two servers share the
same implementation or infrastructure. Breaking the system entirely may only be
a matter of finding and exploiting one bug, or compromising one organization
(such as a common cloud provider). This is a commonly cited problem in the
[Certificate Transparency](https://emilymstark.com/2020/07/20/certificate-transparency-a-birds-eye-view.html)
ecosystem, where many otherwise independent log servers use the same
[implementation](https://github.com/google/trillian). Certificate Transparency
is not exactly comparable to the systems I’ve been discussing in this post -- in
particular, log servers are not exactly assumed to be non-colluding, at least
theoretically -- but in practice, exploiting multiple log servers with the same
bug would be quite damaging for Certificate Transparency clients.

But on the other hand, multiple independent server implementations is not
obviously better. It may cost more to find one exploitable bug in a hardened,
mature, well-resourced implementation than one bug in each of two subpar
implementations.

### Unlinkability is weak

Unlinkability is a fairly weak privacy guarantee, and I think it is probably set
as the goal more often because it is relatively cheap and feasible than because
it is the ideal goal (in fact the OHTTP spec is pretty
[explicit](https://www.ietf.org/archive/id/draft-thomson-http-oblivious-01.html#name-applicability)
about this). In most applications that I can think of, individual queries can be
sensitive in particular circumstances, even if they aren’t linked to particular
users. For example, anonymous DNS queries usually aren’t sensitive, but
occasionally they are -- an uptick of traffic to a particular hostname, even if
not linked to specific users, might indicate an impending product launch. I’ve
also often seen developers assume that URLs are not sensitive if they are not
linked to a particular user, but as noted above, capability URLs are a
counter-example to this belief.

In addition, in many browsing applications, side channels may be a threat to
unlinkability. For example, the attacker may be able to distinguish which
queries are coming from the same user via timing information or request size.

Truly private systems like Prio and Checklist guarantee that the servers learn
zero information from each query, which is a much stronger guarantee at the cost
of more complexity.

### Failing to consider availability

In some applications, distributing trust among multiple servers trades off
exploitability for availability requirements. These applications are a nice
example of why availability is often cited as a security property. For example,
in ODoH, if the oblivious proxy server goes down or becomes unavailable, the
client may have to fall back to regular DoH, losing its privacy guarantees. Or
in Checkpoint, an unavailable helper server leaves the client vulnerable to
phishing. The servers must be highly available in order for clients to rely on
them and be able to fail closed in the event that they are unavailable. In
practice, I suspect that clients will fail open in many applications, meaning
that an attack on the servers’ availability is an attack on users’ privacy.

***

Splitting up trust among multiple servers is an appealing concept for browser
features, but it’s a lot more nuanced than it looks at first glance. Because the
trust relationship between browser users and browser vendors is already so hard
to nail down, I think multi-server approaches are most likely to be fruitful and
coherent in applications where the browser vendor isn’t the party from which we
are trying to protect users’ privacy.