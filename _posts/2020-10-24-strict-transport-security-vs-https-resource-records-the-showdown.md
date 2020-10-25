---
layout: post
title:  'Strict Transport Security vs. HTTPS Resource Records: the showdown'
---

[HTTPS resource records](https://datatracker.ietf.org/doc/draft-ietf-dnsop-svcb-https/00/?include_text=1)
(HTTPS RRs) are a new type of DNS record. The standard is still in progress and
covers various intended use cases, mostly around delivering configuration
information and parameters for how to access a service. In this post, I’m going
to ignore all that and focus on one very simple use case: an HTTPS RR, on its
own, can signal that a domain supports HTTPS, allowing the browser to upgrade
connections to that domain from http:// to https://. In theory, browsers could
look up HTTPS RRs alongside the A/AAAA lookup for a hostname, and if an HTTPS RR
is present, then the browser would send the request over https:// and not
http://. The idea is to prevent
[sslstrip](https://www.venafi.com/blog/what-are-ssl-stripping-attacks)-style
attacks where a request to a site over http:// gets rewritten by the attacker,
before the server even has a chance to redirect the http:// request to https://.

This aspect of HTTPS RRs looks similar to another technology:
[HTTP Strict Transport Security](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Strict-Transport-Security)
(STS). STS allows a hostname to tell a web browser that it
only wants to be accessed over HTTPS. Any http:// URLs using that hostname will
be automatically rewritten to https://, and if the https:// connection fails for
any reason, there won’t be any fallback to http://. Servers have two options if
they want to enable STS:

* Servers can serve an HTTP header to enable and configure STS. When the browser
  sees this header, it will enforce STS for the hostname for a specified period
  of time.
* Some browsers ship a baked-in list of hostnames that have configured STS,
  known as the HSTS *preload list*.

The advantage of the preload list is that hostnames on it will be protected even
if the user has never visited them before. With the HTTP header, at least the
first visit to the site is unprotected until the browser sees and processes the
header. The disadvantage of the preload list is that it’s unscalable, and that
it’s harder to undo. Once a hostname is on the preload list, if it stops
supporting HTTPS, it won’t be accessible in most major browsers until all
browsers ship an update.

With respect to HTTPS upgrading, HTTPS RRs and STS look so similar that you
might wonder if HTTPS RRs obsolete STS. That turns out to be a subtle question.
The two technologies have subtle differences in their threat models, but also in
their deployment tradeoffs, such that neither is clearly better than the other.

## Comparing threat models

First of all, for HTTPS RRs to have any security benefits at all, we must assume
*secure* DNS traffic. Typically DNS lookups and responses are sent in the clear,
in which case an attacker on the network could simply rewrite the response to
the HTTPS query to indicate that the domain does not support HTTPS. So, as I
talk about HTTPS RRs as a replacement for STS, I’m assuming the ubiquitous
deployment of DNS-over-HTTPS (DoH), an in-progress protocol for sending, well,
DNS traffic over HTTPS. When a client and resolver are using DoH, an attacker on
the network between them can’t read or modify DNS queries and responses -- at
most, the attacker can block the traffic outright. The challenges of DoH
deployment could make up a whole other blog post, but I talk a bit about the
current status [below](#deployment-status). In the rest of this section, I’ll
just wave a magic wand and assume that all DNS traffic is secure. With that
assumption, I’ll walk through the attacks that work in one of HTTPS RRs or STS
but not the other.

### Malicious resolvers

*Attack: a malicious or compromised DNS resolver colludes with an attacker
on-path to the web server to prevent traffic to the web server from being
upgraded to https://.*

The most obvious and crucial difference between HTTPS RRs and STS is that, with
HTTPS RRs, we are placing trust in at least the first-hop resolver to deliver
the correct HTTPS RRs. A malicious resolver could claim that a domain has no
HTTPS RR configured, causing the browser to access the site over http:// instead
of https://.

What about resolvers further upstream? It depends on the extent to which
[DNSSEC](https://en.wikipedia.org/wiki/Domain_Name_System_Security_Extensions)
is deployed for the domain in question and whether the client is using a
resolver that validates DNSSEC. DNSSEC is a system for authenticating the data
served via DNS, but it’s spottily deployed, 20 years after it came into
existence. I can’t seem to find a canonical, up-to-date data source about how
widely it’s deployed (please send me pointers if you know of any)[^1]. In theory, if
all the relevant zone administrators used DNSSEC to sign their records, and the
client’s resolver validated those signatures, then only the client’s first-hop
resolver could prevent the https:// upgrade. Any other malicious resolver along
the path would not be able to forge an HTTPS RR (or lack thereof) that passes
DNSSEC validation. I’m not sure how common this level of DNSSEC deployment is in
practice these days, though it seems that a few years ago it was quite
[uncommon](https://www.usenix.org/system/files/conference/usenixsecurity13/sec13-paper_lian.pdf).

### Attacking the first visit

*Attack: a network attacker intercepts the user’s first visit to a site and
prevents it from being upgraded to https://.*

I mentioned that the comparison between HTTPS RRs and STS is subtle, and one of
the reasons is that preloaded STS vs HTTP-header STS have different security
properties. So the comparison is really 3-way, not 2-way. In particular, when
configured via HTTP header, STS does not protect a client’s first visit to a
hostname. It might also not protect some subsequent visits, depending on how
often the client visits the site and how quickly the header configures STS to
expire. In contrast, both preloaded STS and HTTPS RRs apply to all of a client's
connections to the site.

### Attacking the upgraded connection

*Attack: a network attacker can’t prevent a connection from being upgraded to
https://, but intercepts the https:// connection with a malicious invalid
certificate.*

The “strict” in Strict Transport Security doesn’t just refer to the fact that
the browser strictly rewrites http:// URLs to https:// when STS is in use. It
also refers to how the browser handles HTTPS certificate errors: namely,
strictly. That is, when a user encounters an HTTPS certificate error on an STS
site, the browser is forbidden from giving them the option to bypass the
certificate error, as it usually would.

As far as I know, there’s no plan for HTTPS RRs to have the same effect; the
presence of an HTTPS RR would signal that the domain should be contacted over
https:// only, but doesn’t affect certificate error UI.

This is a topic for another blog post, but my opinion is that it was a mistake
to make certificate errors non-bypassable for STS. Possibly STS was originally
envisioned as a technology for highly sensitive sites, not as a standard part of
HTTPS configuration as it is today. I could see a possible reasonable future in
which HTTP-header STS goes away, and preloaded STS and HTTPS RRs stick around as
two tiers of HTTPS upgrading. Preloaded STS, with its inherent scaling
limitations, would be the tier for a limited number of highly sensitive sites,
where even the first-hop resolver shouldn’t be trusted and even certificate
errors shouldn’t be bypassable. HTTPS RR would be the mass-market STS, offering
a lower level of security but less deployment risk.

### Dropping traffic

*Attack: a network attacker selectively drops client traffic to prevent
connections from being upgraded to https://.*

In Chrome, and probably in other major browsers too, the preloaded STS list
fails open when the browser hasn’t been updated in a long time. If a domain
owner wants to stop supporting HTTPS after adding their domain to the preload
list, they can remove their domain from the preload list -- but then they have
to wait for that to propagate to all users they want to support, and some
clients might never receive the update. With the fail-open expiration in place,
there’s a finite amount of time that domain owners have to wait for preload list
removals to propagate. After that amount of time, they know that any client
which hasn’t yet received the update with the removal isn’t enforcing preloaded
STS at all anymore.

This client behavior means that if an attacker blocks a client from receiving
updates over a long period of time, that attacker can prevent the browser from
upgrading preloaded domains to https://. However, we don’t usually talk about
this attack much because if an attacker can block the browser from getting
updates for a long period of time, the attacker can likely wait until a security
vulnerability is disclosed and exploit that to do much more harm than preventing
https:// upgrades.

In the HTTPS RR scenario, one might wonder what happens if an attacker blocks
HTTPS RR queries. Even when all traffic between the client and DNS resolver is
encrypted, the HTTPS RR query might have distinctive properties (such as
response size) that allow attackers to block it selectively while allowing other
queries to succeed. As I’ll discuss [below](#deployment-status), we don’t really
know the security consequences of this attack yet. In an ideal world, when a
client doesn’t receive a response to an HTTPS RR query, it would treat it as a
network error and not allow the connection to proceed at all, such that this
attack would be a denial-of-service rather than a downgrade to http://. But it’s
not known yet if that ideal behavior is going to be deployable in light of
real-world, non-attack failure conditions.

## Deployment status

STS is mature and widely deployed, and often even recommended as a standard step
when setting up HTTPS. It’s implemented in all major browsers. HTTPS RRs are, on
the other hand, brand new and not fully specified or implemented yet. It’s not
yet clear if HTTPS RRs will be deployable at all, so for now the STS vs HTTPS
RRs question is an academic exercise: any HTTPS website owner looking to protect
their users from sslstrip-style attacks today must use STS.

There are at least two major hurdles to deploying https:// upgrades via HTTPS
RRs:

* **DoH deployment**. Several
  [major](https://support.mozilla.org/en-US/kb/firefox-dns-over-https)
  [browsers](https://www.chromium.org/developers/dns-over-https)
  now implement DoH, with varying approaches that have different tradeoffs.
  Still, while I don’t have exact numbers on hand, it’s safe to say that
  relatively few web users are using DoH today. DoH is not exactly a dependency
  for HTTPS RRs, but with respect to https:// upgrading, HTTPS RRs provide
  fairly little value if the DNS traffic isn’t secure[^2]. With HTTPS RRs over
  plaintext, an attacker who wishes to prevent an https:// upgrade must be on
  the path between the client and the resolver in addition to the path between
  the client and the web server; thus HTTPS RRs over plaintext provide some
  security value compared to nothing at all, but very little compared to STS.
* **Reliability of a new record type**. What happens if a domain hasn’t
  configured HTTPS RRs? The client should receive a NXDOMAIN response -- and the
  client should, ideally, assume that if it receives no response, something
  funny is afoot and the client should refuse to connect to the site. With this
  client behavior, as I mentioned above, even with DoH in use, an attacker can
  selectively block HTTPS RR responses but achieve at most a denial-of-service
  by doing so. However, it’s not yet known if this behavior is going to be
  deployable, because it might turn out that in real-world environments, HTTPS
  RRs get blocked for any number of non-attack reasons. If this is the case and
  can’t be fixed, then clients might have to allow plain http:// connections
  when they don’t receive HTTPS RRs responses, which significantly weakens the
  security properties of https:// upgrades. Chrome is currently planning an
  [experiment](https://groups.google.com/a/chromium.org/g/blink-dev/c/brZTXr6-2PU/m/g0g8wWwCAwAJ)
  to investigate how the DNS ecosystem handles a new record type.

## Just to make things a little more complicated...

As you can see, comparing HTTPS RRs to STS is already quite complicated, with no
one tool coming out cleanly ahead in my mind. But just to make things a little
more complicated, there are more HTTPS upgrading proposals that one can compare
to as well! One idea that comes up often is that browsers should use https:// as
the default scheme instead of http://. In other words, if a person types just
“example.com” in the browser address bar, with no specific scheme, today
browsers will send the request to http://example.com. But with HTTPS adoption so
widespread now, perhaps the browser should send the request to
https://example.com instead, unless the person specifically types “http://”.

Would this change of default scheme obviate STS and/or HTTPS RRs? Or conversely,
do STS and/or HTTPS RRs make it pointless to change the default scheme? The
answer to both questions is “no”:

* Changing the default scheme doesn’t obviate STS and/or HTTPS RRs for several
  reasons. One very simple reason is that STS and HTTPS RRs apply to all
  requests, not just typed navigation requests. Another more complex reason is
  that changing the default scheme is not likely to be deployable in the
  near-term without unacceptable breakage. Browsers will likely have to fall
  back to http:// if the default scheme https:// fails -- meaning that an
  attacker can block the https:// request to cause the browser to fall back to
  http://. Thus STS and HTTPS RRs (over DoH) provide better security than
  simply reling on a default https:// scheme with a fallback to http://.
* Changing the default scheme is still useful, however, because STS and HTTPS
  RRs are unlikely to be universally deployed anytime soon. Defaulted typed
  navigations to https:// will protect sites that support https:// but haven’t
  configured any https:// redirect or upgrading -- though this protection is
  against passive attackers only as long as the browser will fall back to
  http://. Even in a world where all HTTPS sites use STS or HTTPS RRs, using
  https:// as a default scheme provides a small amount of defense in depth
  against the various weaknesses I’ve discussed above.

Clearly, there’s a lot going on in this space. My prediction is that we’ll end
up with a patchwork of different HTTPS upgrading mechanisms that each play a
slightly different role. I also predict that we’ll see some slight weakening of
security in the name of deployability and scalability; for example, perhaps
fewer sites will be preloaded in favor of slightly less secure but more scalable
HTTPS upgrading mechanisms like HTTPS RRs.

<small><i>Thanks to [Kaustubha Govind](https://twitter.com/cos_theta) and Dan
McArdle for explaining many of the things in this post to me!</i></small>

[^1]: [This dashboard](https://stats.dnssec-tools.org) gives a per-TLD breakdown
      but doesn’t have stats on what percentage of clients are using resolvers
      that validate DNSSEC. Thanks to Jacob Hoffman-Andrews for the pointer!

[^2]: There are use cases other than HTTPS upgrading where HTTPS RRs
      provide more notable value even if retrieved over plaintext -- in
      particular, use cases that are aimed at foiling passive attackers.