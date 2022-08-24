---
layout: post
title:  "Certificate Transparency is really not a replacement for key pinning"
---

I’ve noticed that there’s a common misconception that Certificate
Transparency is a replacement for HTTP public key pinning. If those
words make no sense to you: [HTTP public key
pinning](https://en.wikipedia.org/wiki/HTTP_Public_Key_Pinning) was a
now-mostly-defunct mechanism whereby websites could “pin” themselves
to a set of public keys, so that browsers would not accept a
certificate for that website’s hostname unless one of those pinned
public keys appeared in its certificate chain. Certificate
Transparency is a system whereby certificates must be logged in public
append-only logs, so that domain owners can observe them as they are
issued and detect malicious certificates (see my previous overview
[here](https://emilymstark.com/2020/07/20/certificate-transparency-a-birds-eye-view.html)).

Public key pinning was mostly removed from the web due its
brittleness; it was very easy for websites to accidentally DoS
themselves with PKP, for reasons that deserve their own lengthy
post. CT is often seen as the modern replacement for PKP. It’s true
that the deployment of CT partly motivated the deprecation of PKP, and
it’s also true that both CT and PKP were motivated by the need to
protect against malicious or mistakenly issued certificates, but CT
and PKP actually provide quite different security properties. I
believe that we could theoretically build upon CT to provide similar
security properties to PKP, but the resulting system would be very
hacky and/or quite baroque (a castle of duct tape and toothpicks,
really) and would ultimately be an abuse of CT, which was not designed
to provide these security properties.

## Security properties

Let’s compare and contrast the security models of PKP versus CT.

Typically, when a website enabled public key pinning, it would bless a
set of root or intermediate certificates that could be used to issue
certificates for that website. This reduced the attack surface of the
web PKI: once a browser was aware of a website’s PKP configuration,
until that configuration expired, it would no longer be the case that
any trusted CA could issue a certificate for that website. The
many-single-points-of-failure property is a deep weakness of the web
PKI, and public key pinning mitigated it. A pinned website would be
vulnerable to compromise of only the single-digit number of CAs that
it pinned to, not the several hundred CAs that browsers/OSes typically
trust.

PKP also defined a reporting functionality. The website owner could
configure PKP such that the browser would send a report of any pinning
violation it observed to a reporting endpoint of the website owner’s
choosing. However, this was not a security feature, as an attacker
using a malicious certificate would typically be able to simply block
reports of the violation as they were sent. PKP reporting was designed
to help site owners detect and debug pinning misconfigurations, not
attacks.

In contrast, CT gives a much fuzzier and qualified set of security
properties. As designed, CT ensures that if a browser accepts a
certificate, it will appear in at least one public log within 24 hours
of submission to said log; submission typically (but not always)
happens around certificate issuance time. In CT as designed, ideally,
this guarantee holds even in the event of a compromised or malicious
log, but the basic CT design doesn’t flesh out details fully enough to
say exactly how malicious log behavior will be detected (specifically
for the attack of a split view, where the log presents one view to
e.g. a client and another to e.g. a domain owner monitoring for
malicious certificates).

This ideal security guarantee is already pretty fuzzy, but it gets
even fuzzier when you consider that CT is in different states of
deployment and maturity across different clients, and no client
currently implements CT fully as designed. For example, in Chrome’s
current deployment, the actual security guarantee of CT is something
like this: if a browser accepts a certificate, it will appear in at
least one public log within 24 hours of submission to said log, as
long as the log(s) are not compromised or malicious. In the event of a
compromised or malicious log, a failure to incorporate a certificate
into the log as presented to Google will be detected probabilistically
(in expectation, if a malicious certificate is used on 10,000
connections, log misbehavior will be detected, though this probability
is tweakable in exchange for some privacy and performance
tradeoffs). There is no real defense against a log that maliciously
presents different sets of certificates to different observers.

CT also has an Achilles' heel: what is a domain owner supposed to
actually do if they find a malicious certificate for their domain in a
CT log? Your first thought might be that they should revoke it, which
they should, but revocation in the web PKI is fraught, with some
browsers implementing minimal support for revocation or only
implementing revocation infrastructure for hand-picked high-value
certificates. This is a complex topic and my best summary is that
revocation, or other reactive action, is an unsolved part of the CT
design.

As you can probably surmise just from the number of sentences and
amount of handwaving used to describe their respective security
guarantees, public key pinning offered a much cleaner and *mostly*
stronger security model than CT. It is nearly impossible to even
cleanly state CT’s security model, thus it cannot be seen as a drop-in
replacement for PKP, which offers fairly clean and strong security
guarantees. **PKP directly protects users against attackers with
malicious certificates issued from non-authorized CAs. CT makes it
more likely that domain owners will notice malicious certificates
issued for their domain.**

While PKP generally looks like the stronger security model, there are
two ways in which one could argue that CT offers stronger security
than PKP:

*  First, PKP will not protect against, or even permit detection of,
   the compromise of the CAs to which a website is pinned. If a
   website pins to some CA, and the attacker compromises that CA, the
   attacker wins. In contrast, CT will (ideally, but in reality
   subject to all the caveats above) allow a website owner to detect a
   malicious certificate regardless of which CA issued it.
*  Second, CT has second-order protective effects in the
   ecosystem. Publishing ~all certificates has enabled the detection
   of various CA incidents and bad practices, helping improve CA
   security before larger compromises occur. All websites are safer
   when CAs adopt better security practices.

Therefore, there is not a strict ordering between the security
properties of CT and PKP; neither fully subsumes the other.

## What would it take for CT to subsume PKP?

If CT does not currently match PKP’s security guarantees, it invites
the question: could it be made to? As a warning, this section is
purely a thought exercise, and definitely not a design proposal. The
short answer is that I believe we could, in theory, build upon CT to
get a lot closer to PKP’s security model, but it would be laughably
complicated and would likely end up just as brittle as pinning
was. The point of this thought exercise is to underline that CT is
really not anywhere close to a replacement for PKP. The degree of
hackery needed to make CT into a replacement for PKP illustrates that
it wasn’t designed for this security model.

I’ll introduce two avenues for evolving CT into something that gets
closer to PKP. Both are bad ideas. Also note that this section assumes
some more in-depth knowledge of how CT works; you might want to skip
this section if you only have a high-level understanding of CT, or
read my previously-linked [overview
post](https://emilymstark.com/2020/07/20/certificate-transparency-a-birds-eye-view.html)
first.

### Enforce pinning in the logs

The first bad idea for building upon CT to get something more like
HPKP is simple: CT logs could enforce pins at certificate submission
time, instead of the client enforcing pins at connection time. A
website owner that wishes to pin would configure pins with CT log
operators, instead of configuring pins to be consumed directly by
clients.

If we assume that CT logs are honest, this straightforwardly achieves
the security goals of public key pinning, without nearly as much
brittleness. There are two sources of brittleness with pinning that
are mitigated by pinning in the logs instead of in clients:

*  Traditional HTTP public key pinning is brittle because if a pinning
   configuration needs to be updated, approximately every browser
   instance in the world needs to learn about that updated
   configuration. With pinning in the logs, however, a website owner
   only needs to reach the ~10 CT log operators to update their
   pinning configuration, instead of every browser instance.
*  Another source of brittleness in traditional HTTP public key
  pinning is the difficulty of predicting which certificate chains
  clients might build. If a website owner pins their site to a
  particular root or intermediate CA, they must be sure that affected
  clients are going to verify their certificate through a path that
  includes that CA – which is not easy to predict, due to differences
  in root CA trust stores across clients and cross-signed
  certificates. However, if logs enforce the pins instead of clients,
  website owners only need to be able to predict the paths that CT
  logs build, not the paths that all clients will build. This is
  simpler to reason about, reducing the likelihood of bad pinning
  configurations; plus, as mentioned above, bad pinning configurations
  are easier to deal with when there are fewer points of enforcement.

Despite these appealing properties, enforcing pins at log submission
time is a terrible idea. Here’s why:

*  First of all, it doesn’t really solve the security problem. By
   design, CT logs are not considered trustworthy, thus they should
   not be trusted to enforce pins.
*  Secondly, it’s already difficult and expensive to run a CT log, so
   CT log operators shouldn’t be burdened with additional
   responsibility, especially responsibility that runs counter to a CT
   log’s goal of providing comprehensive transparency. Except as
   needed to mitigate spam and abuse, CT logs should seek to accept
   and publish as much as possible, not to reject
   certificates. Specifically, CT logs should be the way that a domain
   owner or other interested party can find out about a malicious or
   misissued certificate; CT logs shouldn’t be responsible for
   rejecting such a certificate and preventing it from being used.

Despite the terribleness of this idea, I think it’s still an
interesting thought exercise because it reveals some properties that
we might want to preserve in a new system designed to replace HTTP
public key pinning. For example, an off-the-cuff idea for a pinning
replacement might be that if a website owner wishes to pin, they
register their pins with each browser vendor, and the website must
present a countersignature from the browser vendor alongside its
certificate. The browser vendor enforces the pins before issuing this
countersignature. This proposal (which surely has plenty of other
downsides) preserves the idea of having fewer, more predictable
pinning enforcement points, but doesn’t abuse CT logs to achieve that
property.

### "Pinning" to transparency

The second terrible idea is to do away with public key pinning as the
basis for security, and instead provide stronger transparency
guarantees. The goal would be to strengthen CT’s guarantees on an
opt-in basis for website owners who are willing to accept some
inconvenience in exchange.

Recall that CT’s guarantee is (approximately, and depending on
deployment model) that if a browser accepts a certificate as valid, it
will appear in at least one public log within 24 hours of
issuance. What if instead, for an opted-in website, the browser
enforces that the certificate was submitted to more than one log at
least 48 hours ago *and* that the certificate comes with a stapled
OCSP response? This would entail some inconvenience for the site owner
(legitimate certificates would not be usable for 48 hours after
issuance, and the website would need to implement OCSP stapling), but
it would improve security by ensuring that the website owner has a
chance to see malicious certificates in CT logs and revoke them before
they become usable for attacking users.

This idea of “pinning” to increased transparency requirements is not
the same mechanism as PKP, but it achieves similar security: malicious
certificates would not be accepted by clients, as long as the website
owner actively monitors CT logs and revokes unexpected certificates
within the >=24 hour window from when they appear until when they
start being accepted by browsers. In a way this is even an improvement
over PKP’s security guarantees, because it allows website owners to
act on any malicious certificate, even if it is issued by a CA they
consider trustworthy.

However, there are two big asterisks on the security model of this
proposal.

The first is that the attacker could obtain a valid OCSP response for
a malicious certificate and continue to use that for several days
after the website owner notices and revokes the certificate, until the
OCSP response expires. Therefore an attacker may be able to use a
malicious certificate for some period of time, unlike PKP where the
browser will always reject the malicious certificate.

The second asterisk is
that we're assuming that each certificate is logged in at least one
*trustworthy* log. As mentioned previously, this violates a design
goal of CT: logs are not meant to be trustworthy.

The browser could potentially reduce trust in logs by requiring not
just an SCT (the signed statement that a log intends to publish a
certificate) but a stapled inclusion proof (a proof that the log
actually incorporated the certificate). This alone isn’t sufficient,
though; a log might have incorporated the certificate into the view
that the web server and browser are seeing, but not into the view that
is being presented to the website owner as they are monitoring for
malicious certificates. We could go even further and have the browser
vendor distribute STHs to clients so that the browser can verify that
the view being presented to the web server and browser instance is
consistent with the view presented to the browser vendor. To take this
all the way to completion, website owners would be obligated to
interact with the browser vendor to monitor for malicious
certificates, to be sure that the website owner is seeing the same
view of the log as the browser vendor and the browser instance; for
example, the browser vendor could operate a CT monitoring service that
the website owner uses to detect malicious certificates. This would
ensure that the browser, website owner, and browser vendor are all
seeing the same view of the log; if a malicious certificate is
accepted by the browser, even in the face of a malicious log, the
certificate must have been visible to the website owner for at least
24 hours, allowing them to notice and revoke it.

Altogether, this is now a very complex proposal, and still with many
details to be ironed out (such as what happens if a browser instance
can’t obtain up-to-date STHs from the browser vendor). It would surely
involve multiple person-years of work to design and build, and I’m not
at all confident that the resulting system would be any less brittle
than public key pinning. As mentioned above, my point in presenting it
is to illustrate that, while it may theoretically be possible to build
upon CT to achieve something similar to PKP, CT is quite far off from
being a PKP replacement.

## A whole separate misconception: CAA

There is a whole separate misconception that
[CAA](https://en.wikipedia.org/wiki/DNS_Certification_Authority_Authorization)
(Certificate Authority Authorization) is a replacement for PKP. CAA is
a DNS-based policy mechanism that evolved around the same timeframe as
CT. CAA is a DNS record that indicates the CAs that are authorized to
issue certificates for a domain. CAs must check this record before
issuing certificates. However, CAA is [a sign, not a
cop](https://i.imgur.com/mSHi8.jpg). PKP was designed to protect
against malicious or compromised CAs, and CAA does not do this; a
malicious or compromised CA would simply ignore CAA and issue a
malicious certificate. There is essentially no technical security
value to CAA, only protection against CAs’ non-adversarial
mistakes. (CAs make a lot of non-adversarial mistakes, so there is
still a lot of value! Just not direct technical security value in the
way that PKP has security value.)

Both CT and CAA are very valuable, and one could argue that together
these two mechanisms add up to significantly diminish PKP’s marginal
value – but neither is a direct replacement for PKP in terms of
security guarantees.