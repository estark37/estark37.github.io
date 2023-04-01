---
layout: post
title:  "EV as a defense against malicious DV"
---

Today I’m going to continue a series of blog posts called “Deep dives
into topics recently written about by people named Eric.”
[Last time](https://emilymstark.com/2022/12/18/death-to-the-line-of-death.html)
it was [Eric Lawrence](https://textslashplain.com/), and today our
jumping off point will be
[Eric Rescorla](https://educatedguesswork.org/)’s blog.

Eric Rescorla (aka ekr) recently wrote a fantastic and comprehensive
blog post on EU regulation regarding server authentication
certificates (called QWACs) and their relationship to Extended
Validation (EV) certificates. There’s one
[part](https://educatedguesswork.org/posts/eidas-article45/#dv-misissuance)
of that post in particular that I know I’m going to end up referencing
a lot, because I had been meaning to write it up myself: why EV
certificates (or QWACs) do not defend a site against a malicious
domain validation (DV) certificate for the same domain.

I won’t rehash Eric’s post and in fact I’m not going to discuss QWACs
at all in this post; I’ll scope this to EV certificates. Eric’s post
is a great reference for background on QWACs and EV, and if you want
more background on EV certificates, I recommend Section 2.2 of
[this paper](https://www.usenix.org/system/files/sec19-thompson.pdf).
Note also that, if you’re not familiar with the term “DV certificate”,
that just refers to a regular TLS certificate – the kind that you can
get for free from a CA Let’s Encrypt or generally what you’d get in
the cheapest tier from any commercial CA. It’s called a “DV
certificate” because all you need to do to get one is validate that
you control the domain name that appears in the certificate.

In this post, I want to dig into the common misconception that EV
certificates protect against an attacker who can hijack a domain
validation attempt to obtain a malicious DV certificate from a CA. An
attacker might be able to obtain a malicious DV certificate via a BGP
hijack or manipulating a DNS response, among other ways. Some people
view EV certificates as a remedy to the problem that DV certificates
can be obtained too easily by a malicious attacker.

The reasoning behind this misconception goes as follows. Obtaining an
EV certificate involves more stringent identity verification than DV,
and thus an attacker who can obtain a malicious DV certificate for a
victim domain cannot necessarily obtain a malicious EV certificate for
that domain. If all the attacker can obtain is a malicious DV
certificate, and if users expect to see the browser EV UI when loading
the victim domain, then the attack is foiled. The user will realize
that something is amiss due to the missing EV UI, and they will not
proceed to interact with the page.

Eric explains the flaw in this reasoning: the web’s security boundary
is the origin (the scheme, host, and port), which does not take the
certificate into account. Per the
[same-origin policy](https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy),
there is no separation between resources loaded over connections
authenticated with different certificates, as long as those resources
share the same origin. Eric’s post gives one example that shows this
property: a website’s main resource is loaded with an EV certificate,
but a Javascript library from the same website is loaded over a
different connection that uses a potentially malicious DV certificate,
thus compromising the website despite the fact that it uses an EV
certificate. This example is but one of many ways that an EV
certificate fails to protect a website from an attacker with a
malicious DV certificate:

* The attacker could use a malicious DV certificate to intercept the
  initial page load. Cookies or other sensitive data could be sent to
  the attacker in this initial request before the user even has a
  chance to see that there is no EV UI in the browser. It’s game over
  before the page even loads.
* The victim website may load resources such as Javascript libraries
  from other domains. That is, there may very well be multiple domains
  for which the attacker could obtain a malicious DV certificate to
  attack the victim website.
* Pages from the same origin loaded in different tabs or windows can
  control each other. For example, a page loaded with an EV
  certificate may open a new window to a page loaded with a malicious
  DV certificate. The latter page can manipulate the former
  arbitrarily via Javascript.

And there are many more attack vectors too. (See the classic, though
perhaps a bit dated, paper
[Beware of Finer-Grained Origins](https://seclab.stanford.edu/websec/origins/fgo.pdf).)
Clearly, using an EV certificate on your website does not protect you
from an attacker who can obtain a malicious DV certificate for your
domain.

## DV is pretty good, actually

Before I go further, I want to note that DV certificates are actually
pretty good, these days – and rapidly getting more secure. I would bet
that most industry security teams are not losing sleep over the threat
of a malicious DV certificate compared to all the other threats to
worry about. The recent advances in DV security are really a whole
other topic to write about, but, in a nutshell,
[Certificate Transparency](/2020/07/20/certificate-transparency-a-birds-eye-view.html)
and
[CAA](https://en.wikipedia.org/wiki/DNS_Certification_Authority_Authorization)
have made a big difference, with more security improvements on the
horizon like
[multi-perspective domain validation](https://letsencrypt.org/2020/02/19/multi-perspective-validation.html)
and [RPKI](https://blog.cloudflare.com/rpki/).

## But what if…

The web platform is mutable, and for someone who is worried about
malicious DV certificates, it’s reasonable to ask whether we could
introduce some changes or new features in the web platform that could
insulate websites using EV certificates from the threats of a
malicious DV certificate.

_Warning_: bad ideas ahead! As in many of my other blog posts, the
rest of this post describes a thought experiment that I do not endorse
as a concrete proposal. The point of exploring the thought experiment
is to show that while perhaps theoretically possible, it would be a
formidable project to undertake.

The first step in this thought experiment is that EV certificates and
DV certificates would need to be separated by origin. Perhaps we would
need to expand the notion of origin from (scheme, host, port) to
(scheme, host, port, EV/DV), where the last component is a bit
indicating whether the certificate used to load resources in the
origin was EV or DV. This assumes that all EV certificates are equally
trustworthy, which may or may not be a reasonable assumption, but
let’s go with it for simplicity. This new origin concept would
theoretically close off some of the attack vectors discussed above:
for example, if a webpage loaded from the EV version of the origin was
loaded in one tab and a webpage loaded from the DV version in another,
the two would not be able to directly script each other because the
browser would now consider them cross-origin.

To securely isolate resources with EV certificates from DV
certificates, we’d need to place some restrictions on EV
origins. Perhaps EV origins would not be allowed to load cross-origin
subresources, or would only be allowed to do so over connections that
also used EV certificates. We cannot allow the EV origin to be
contaminated by, e.g., importing a Javascript library from a DV
origin. This goal is somewhat uncharted territory: while web browsers
have lots of practice isolating origins from each other, there is no
existing concept of more or less dangerous origins, and no
well-trodden path for providing additional isolation between origins
on top of typical cross-origin isolation. This would also represent a
significantly different development model for web developers who want
to use EV origins; including and interacting with resources from other
origins is a pretty core part of web development.

Speaking of interactions with other origins, we’d need to define a way
to serialize these special EV origins. For example, suppose a page
with an EV origin received a message from another page via
`postMessage`. To use the `postMessage` API securely, message
recipients must validate the origin from which they received the
message, typically by string comparison on the `message.origin`
field. What string should appear in the `message.origin` field if the
message comes from an EV versus DV origin? That is, how can an EV
origin receiving a message validate that the message did not come from
a DV origin? The authors of the now-defunct [Suborigins
project](https://w3c.github.io/webappsec-suborigins/) faced this same
issue and decided to serialize a suborigin name into the host
component of the origin. I didn’t work on this project myself, but as
I recall, the ramifications of changing origin representation in this
way were never fully worked out. It’s an especially complex
proposition for the idea of EV/DV origins. Presumably DV origins would
be serialized as just scheme, host, and port for backwards
compatibility, with a special serialization just for EV origins – but
this means that any developer validating an origin would need to
remember to use the special EV origin serialization in their
validation check. If they forget to do so, the origin check in
question would be vulnerable to abuse by DV origins. This fail-open
property (if you forget to do the special thing anywhere, you are
vulnerable) is generally a harbinger of bad security outcomes.

A final consideration is that one very important security-critical
thing in the web platform is actually not
([yet?](https://chromestatus.com/feature/4945698250293248)) isolated
by origin, but rather by domain: cookies. If you set a cookie on
http://example.com, it will be sent to https://example.com, and vice
versa. This means that cookies would not “just work” in this thought
experiment of separate EV and DV origins. Cookies are keyed by domain,
so a cookie set on https://example.com would be shared between the EV
and DV versions of that origin. Since cookies are often used to store
sensitive data like authentication tokens, this destroys the security
properties we are looking to achieve. Further work on isolating
cookies by origin would be necessary for the EV/DV origin proposal to
get useful security properties.

There are probably many additional complexities to this origin
separation idea that I’m not considering. Overall, it may be
theoretically possible, but it is a hugely complex proposition both
for browser developers and for web developers who would want to use an
EV origin.

## Separating identity from TLS

Even if we could offer strong isolation between EV certificates and DV
certificates on the web, EV certificates would still retain many
suboptimal security implications. Much has been written elsewhere on
the downsides of EV. Perhaps most importantly, tying certificate
issuance to manual verification processes discourages automated
certificate issuance and renewal, which is critical for so many
security improvements. (I discussed this at length in my
[Enigma talk](https://www.usenix.org/conference/enigma2023/presentation/stark)
in January.) Much of the discourse around QWACs has focused on
separating legal identity from TLS: a TLS certificate can be used to
authenticate a domain name and a separate mechanism can be used to
establish the legal identity of the domain name. This separation of
responsibility is likely to lead to a much healthier future for web
security.