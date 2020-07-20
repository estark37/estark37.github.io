---
layout: post
title:  "Certificate Transparency: a bird's-eye view"
---

Certificate Transparency (CT) is a still-evolving technology for detecting
incorrectly issued certificates on the web. It’s cool and interesting, but
complicated. I’ve given talks about CT, I’ve worked on Chrome’s CT
implementation, and I’m actively involved in tackling ongoing deployment
challenges -- even so, I still sometimes lose track of how the pieces fit
together. I find it easy to forget how the system defends against particular
attacks, or what the purpose of some particular mechanism is.

The goal of this post is to build up a high-level description of CT from
scratch, explaining why all the pieces are the way they are and how they fit
together. A lot of this material is drawn from a
[guest lecture](https://web.stanford.edu/class/cs253/lectures/Lecture%2012.pdf)
I gave with my colleague [Chris Palmer](http://noncombatant.org/) as part of a
Stanford web security course (CT portion starts at slide 52).

## Why CT?

Certificate authorities (CAs) are the organizations that browsers trust to issue
certificates for domain names after checking that the person receiving the
certificate really does own their domain. CAs can be companies and governments
or the occasional non-profit. It’s fairly common for CAs to issue bad
certificates. Like any organization, sometimes CAs make mistakes, and sometimes
they are compromised or malicious. And sometimes certificates are issued
improperly without the CA doing anything wrong at all: a rogue insider, BGP
hijacker, or hapless vendor might request a certificate for a domain they do not
own.

The security community has invented various mechanisms to prevent browsers from
accepting these “misissued” certificates. For example, server operators could
once use
[public key pinning](https://en.wikipedia.org/wiki/HTTP_Public_Key_Pinning) to
limit the CAs that could issue certificates for their domains, so that not just
any CA compromise would put them at risk. (Unfortunately, pinning turned out to
be a giant footgun, never gained widespread adoption, and is currently dying a
slow death.)

Preventing browsers from accepting misissued certificates is only one part of
the puzzle, though. It’s also important to detect misissued certificates. For
example, in 2011, an attacker hacked the
[DigiNotar](https://en.wikipedia.org/wiki/DigiNotar#Issuance_of_fraudulent_certificates)
CA and improperly issued a wildcard google.com certificate, which they used to
target victims in Iran. While the Google Chrome browser didn’t accept the
certificate due to public key pinning, Google only
[found out](https://slate.com/technology/2016/12/how-the-2011-hack-of-diginotar-changed-the-internets-infrastructure.html)
about the attack by chance because a user posted about the error on a help
forum. This was a close call: would anyone have ever learned about this attack
if that one user hadn’t happened to report it?

Thus was born the desire for a technical means of detecting misissued
certificates. If a domain owner can detect an unauthorized certificate for their
domain, they can revoke[^1] the misissued certificate. If the misissuance happened
because a CA misbehaved, the domain owner can report the incident to a browser
or OS vendor, who might choose to no longer trust the CA to issue certificates.

An unauthorized certificate for a domain isn’t the only type of misissuance. CAs
might create certificates that are bad because they use outdated cryptographic
algorithms, violate rules like the maximum lifetime of a certificate or specific
encodings that are supposed to be used, etc. If browser vendors could detect
these types of misissuances (which are not necessarily malicious, but signs of
bad hygiene), they could ask the CAs to fix these problems or even remove trust
in them if appropriate.

## A strawperson and a slightly-less-strawperson

The very simplest attempt to allow detection of bad certificates would be to
require that CAs publish a list of every certificate they issue. They could
either directly publish their certificates themselves, or forward them to some
other service that publishes them. Domain owners could monitor every CA’s
published certificates for unauthorized certificates, and browser
vendors/researchers/etc. could monitor each CA for bad hygiene.

But the root problem we are trying to solve is that CAs might be compromised,
malicious, or error-prone, so we can’t just rely on CAs to publish their own
issuances. An evil CA issuing an evil certificate would simply not publish that
certificate.

In a slightly more robust system, CAs could submit each issued certificate to a
publisher. The publisher would return a signature on the certificate, and
provide a publicly accessible feed of certificates that it has seen. Browsers
would only accept certificates that come with a signature from a trusted
publisher. This is a little closer to what CT looks like, but there are still a
number of problems to solve.

## Signed Certificate Timestamps

One minor wrinkle with this publisher signature system is that it takes time for
the CA to submit each certificate to these publishers and verify that it got
logged before issuing the certificate. (As will become clear later on, logging a
certificate can require operations on a large data structure and global write
consensus, which can take minutes or hours.) CAs don’t want other services’
potentially slow operations in the critical path of certificate issuance, which
is their core money-making business operation -- and their customers don’t want
this slowdown either, since some web servers rely on fast certificate issuance
to meet business and uptime requirements. So in CT, a signature from one of
these publishers (which is called a CT log) does not actually guarantee that the
log has published the certificate. Instead, the log issues a Signed Certificate
Timestamp (SCT), which is a signed statement that the log has seen the
certificate and promises to publish it within 24 hours of the timestamp it
provides.

## *Trusted* logs?

So far, the system we’ve built up is as follows: CAs submit certificates to logs
as they are issued, and the logs return SCTs promising to publish the
certificate within 24 hours. Browsers don’t accept certificates unless they come
with SCTs from trusted logs. Interested parties, such as domain owners or
researchers, can monitor the data that the logs publish for malicious
certificates.

Now the key question is: why should we trust logs? If a CA could be compromised,
malicious, or error-prone, why couldn’t a log be compromised, malicious, or
error-prone too? One could even imagine a log and CA colluding to issue evil
certificates. The designers of CT wanted logs to be untrusted, and this is how
CT gets complicated.

As a simple way to remove trust from the logs, browsers could require multiple
SCTs from different logs per certificate. With this policy, an attacker would
have to compromise multiple logs to prevent an evil certificate from being
published by any of them. In practice, though, it’s difficult to say what
constitutes distinct logs. If the same organization controls multiple logs and
is colluding with the attacker, a multiple-log policy doesn’t help. Chrome
currently deploys CT with a “One Google, One Non-Google” policy: each
certificate must come with at least one SCT from a Google-operated log and one
from a non-Google-operated log, on the premise that it would be difficult for an
attacker to compromise two such logs. This policy, however, was always meant as
a temporary measure until something more technically sound and organizationally
neutral could be put into place.

To understand how to remove trust from the logs in a more technically sound way,
we need to think about a deep question: what does it actually technically mean
to “publish” a certificate? Each log has an API endpoint that produces a feed of
certificates -- but, intuitively, providing the certificate on that endpoint to
one client at one particular point in time is not sufficient to have “published”
the certificate. To truly publish the certificate, that endpoint must provide
the certificate to anyone querying it at any time after the log signed an SCT
for the certificate. To verify that the log has truly published the certificate,
we need everyone to efficiently check with everyone else that they’ve seen the
certificate in the feed they’ve received from that endpoint. “Everyone” here
includes end-user devices that are validating TLS certificates, as well as
researchers, browser vendors, domain owners, and anyone else who is interested
in monitoring a log’s feed to look for misissued certificates.

Of course, it’s not practical to ask everyone to maintain the list of all
certificates that each log publishes and check with each other to compare that
the entire sequence matches. That could be billions of devices each maintaining
sets of millions or billions of certificates. Furthermore, some of these
entities might be interested in only a very small subset of certificates; for
example, a domain owner is only interested in monitoring for misissued
certificates for their own domain, and has no interest in doing a lot of work to
help other entities verify that unrelated certificates were properly published.

Imagine that the log could produce a short summary of the sequence of
certificates that it has published. The summary is like a cryptographic hash: if
you can provide a sequence of certificates that produces a given summary, then
it’s infeasible to find any other sequence that produces the same summary.
Unlike a vanilla cryptographic hash, these summaries have two special
properties:

1. If the log produces a summary, it can then add more certificates and produce
   an updated summary, and efficiently prove that these two summaries are
   consistent with each other, i.e. the latter summary’s underlying sequence of
   certificates is a supersequence of the former’s.
2. After adding a certificate, when asked for a current summary of the
   certificates it has published, the log can provide an efficient proof that
   the summary contains that certificate.

This magical summary is all the state that anyone observing the log needs to
keep to be able to verify with everyone else that the certificates they care
about have been published. For example, a web browser validating a TLS
certificate can request a summary and a proof that the current certificate is
contained in that summary (property #2). As the browser encounters more
certificates, it can request updated summaries with proofs for each of those
certificates, and it can always verify that the latest summary is consistent
with its previous summary (property #1). All the browser has to do is keep track
of that single latest summary, and it is infeasible for anyone to produce a
sequence of certificates mapping to that summary which does not include the
certificates that the browser has verified. Two different browser users can then
exchange summaries with each other and verify their consistency (property #1) to
verify that the certificates that each browser user has seen have been published
to the other.

This is magical and mind-bending because browser user Alice is able to verify
that all the certificates they saw were “published” to browser user Bob without
browser user Bob ever seeing a single one of those certificates, and vice versa.
That is, the log would be unable to ever produce a feed of certificates that is
consistent with Alice’s summary unless it also contains all the certificates
that Bob saw.

In practice, a browser user is unlikely to ever go download the log’s whole feed
of certificates (why would it?). Instead, other entities keep watch on the
entire feed: browser vendors, researchers, and services called “monitors” that
help domain owners watch for unauthorized certificates for their domains. So
once browser user Alice has verified that their summary is consistent with
browser user Bob’s summary, Alice can then go check consistency with, for
example, a monitor’s summary to verify that all certificates seen by both Alice
and Bob have been seen by the monitor (who is watching all certificates that the
log publishes).

I like to think of these summaries sort of as an accumulator that can eat up
more and more certificates without growing in size[^2]. Once you verify that a
certificate is included in a given summary, you can keep updating that summary
to the latest version, and provided that you check consistency with your
previous summary, you never lose the property that your current summary contains
that certificate. When you compare your summary with someone else’s, your
summary absorbs all of the certificates that they’ve seen, and now you never
lose the property that your current summary contains all your certificates and
all their certificates and all the certificates seen by whomever they’ve checked
their summary with. This is true even though you’ve never seen those
certificates! Because of this accumulation property -- verifying your summary
with someone else’s accumulates all the verification that they’ve done into
yours -- it becomes possible for large numbers of participants to efficiently
check that they’ve each seen the certificates seen by the others. And that is
what we truly mean by “publishing” a certificate.

## Magic isn’t real

Sadly, not all of what I just described is actually deployed in the real world
right now. Currently, Chrome and some other browsers require a number of SCTs to
trust any public certificate issued after particular dates. This was a big
milestone, as all major CAs now log the vast majority of their newly issued
certificates by default. However, there’s still a lot of work to do.

Verifying that a given certificate is included in a summary is called SCT
verification, and no major web browsers actually do it yet. Andrew Ayer has a
great
[post](https://www.agwa.name/blog/post/how_will_certificate_transparency_logs_be_audited_in_practice)
on how it might be deployed and why it’s challenging.

What I called a summary is actually called a Signed Tree Head (STH). I talked
about different parties in the ecosystem exchanging their summaries/STHs and
checking consistency with one another. This prevents what’s known as a
“split-view attack”, where a log presents an STH that includes a malicious
certificate to the client being attacked with the certificate, and a totally
separate STH that doesn’t include the malicious certificate to the client
monitoring for misissusances. There has been an attempt to define
[gossip protocols](https://tools.ietf.org/html/draft-ietf-trans-gossip-05)
for exchanging STHs, but they haven’t been widely deployed yet. Most likely,
centralization will turn out to be key. It’s not particularly likely that every
browser user will gossip STHs with every other browser user, but it is more
feasible for a handful of browser vendors to gossip with one another and then
distribute these gossiped STHs to their users.

The underlying magic of CT is a Merkle tree. This is an old, simple
cryptographic construction: a tree of hashes. CT uses a simple procedure for
updating the tree with new certificates, and this simple construction gives the
two magical properties I described above. The part of CT that seems most magical
is actually quite real, but deploying it at scale in the real world turns out to
be the hard part.

<small><i>Many thanks to the following people for giving feedback on this post:
Zakir Durumeric, Matt Holt, J.C. Jones, Eric Mill, Devon O’Brien, Ryan Sleevi,
and Stephan Somogyi.</i></small>

[^1]: Except they can’t, really. Revoking a certificate is not as simple as
    alerting the CA who then distributes the revocation data via OCSP or CRL.
    [Some](https://blog.mozilla.org/security/2015/03/03/revoking-intermediate-certificates-introducing-onecrl/)
    [browsers](https://dev.chromium.org/Home/chromium-security/crlsets) use
    custom lists; Chrome’s is mostly used for distributing intermediate
    certificates and/or high-value end-entity certificates. Generally domain
    owners have fairly hazy and browser-specific guarantees on if and how
    revocations are distributed. Thus, the community of average web developers
    is still missing a playbook that tells them what to do if they discover an
    unauthorized certificate for one of their domains in the CT logs.

[^2]: While the summaries don’t grow in size, the proofs do (logarithmically
    with the number of certificates the log has published).