---
layout: post
title:  "E2EE on the web: isolating plaintext"
---

With the publication of [Messaging Layer
Security](https://messaginglayersecurity.rocks/) (MLS) as an
[RFC](https://datatracker.ietf.org/doc/rfc9420/), I’ve been pulled
into some recent discussion about bringing end-to-end encryption
(E2EE) to the web. This is a topic that comes up every so often and
has weirdly haunted me throughout my career. (I spent my undergrad and
graduate research years working on [cryptography implementations in
Javascript](https://crypto.stanford.edu/sjcl/acsac.pdf) and how to use
them in [applications](https://css.csail.mit.edu/mylar/mylar.pdf).)

In this post, I’m going to discuss an idea that I’ve seen coming up a
lot lately: the idea of isolating plaintext in an E2EE application so
that it can’t be accessed by application code. But first, some
background on E2EE on the web generally.

## Why isn’t E2EE on the web the same as on any other application platform?

In E2EE, we want to encrypt data locally such that even a malicious
application server can’t see plaintext. On the web, the fundamental
challenge is that application code is downloaded afresh from the
server on approximately each connection, so there are many
opportunities for an attacker to compromise the code that runs locally
on the client and has access to the plaintext.

Deirdre Connolly’s
[classic blog post](https://zfnd.org/so-you-want-to-build-an-end-to-end-encrypted-web-app/)
describes some of these particular challenges of deploying E2EE on the
web. Most notably, the integrity and provenance of web application
code rests entirely on a TLS connection from the browser to an edge
server. In contrast, native desktop or mobile apps (and updates) are
bundled and signed by a developer key mediated by an OS-trusted
service (like an app store). Deirdre argues that E2EE makes sense for
web applications that are one-time use with ephemeral identities for
each use (e.g., video calls where each participant logs in via a
one-time meeting code tailored to that participant), or perhaps for
applications with a trusted identity provider who can dole out
ephemeral keys tied to a long-term identity – but not for applications
like Signal where long-term identity keys are managed locally on a
user’s device. On the web, there is no long-term trustable notion of
what “the application” is, thus there is no sense in storing long-term
secret keys that the application can use – because any one of a
zillion TLS connections, or edge or origin servers, could be
compromised to steal the long-term keys.

I’m not sure I 100% agree with Deirdre’s argument, or perhaps might
frame it differently, but it’s a good summary of why E2EE is different
on the web compared to other application platforms. (Also see: various
Signal community
[commentary](https://community.signalusers.org/t/google-to-retire-chrome-apps-what-will-be-with-signal-desktop/469/6)
on this topic.)

I will also note that, to my knowledge, the web doesn’t really
currently offer APIs for secure long-term identity key generation and
storage that can be used for E2EE. I think the closest you could get
today is to generate a [non-extractable Web Crypto
key](https://developer.mozilla.org/en-US/docs/Web/API/CryptoKey/extractable)
and
[store](https://developer.mozilla.org/en-US/docs/Web/API/SubtleCrypto#key_management_functions)
it in IndexedDB, and request [persistent storage](https://web.dev/persistent-storage/).
However, this method is not guaranteed to work in all circumstances,
does not offer cross-browser compatibility (that is, you can’t
generate and store an identity key for E2EE in one browser and use it
in another browser), and provides no security guarantees with respect
to how the key is stored; the only security guarantee is that it is
not exposed to Javascript code.  That said, the matter of key
generation and storage is mainly “just” missing API surface in the web
platform, which could be added, with care, if the threat model were
deemed coherent. (Except for the cross-browser compatibility bit –
that part seems more challenging to me than a matter of simply adding
APIs.) This might look similar to or overlap with the [WebAuthn
API](https://webauthn.guide/#webauthn-api), which defines an interface
for using public keys for signature-based user authentication on the
web.

## How to mitigate the code integrity and provenance situation

Most often, when people talk about how to make E2EE more palatable on
the web, the general idea is to make the code distribution model look
a bit more like the native app ecosystem. This would not necessarily
entail installing web applications from a store. Rather, it could mean
that E2EE web application code must come bundled and signed (e.g., in
the format of [Signed Exchanges](https://web.dev/signed-exchanges/)),
and has some limitations placed on its execution environment (for
example, a Content Security Policy that prohibits dynamic code
execution). If you squint, you could imagine layering on some type of
code transparency (a la
[Binary Transparency](https://binary.transparency.dev/)) so that user
agents check that they are running the same version of the application
as everyone else… for some definitions of “check”, “same”, and
“everyone else”.

A closely related project is
[Isolated Web Apps](https://github.com/WICG/isolated-web-apps/blob/main/README.md),
though that project is not necessarily targeting an open-web
distribution experience. (For example, there may be an installation
process mediated by a trusted service, according to my understanding.)

There are _many_ tricky details that would need to be worked out, but
“code bundling and signing, with binary transparency” is the general
direction in which many experts’ minds go when posed with the problem
of bringing E2EE to the web.

### A different idea: isolate the plaintext

Sometimes people’s minds go in another direction: perhaps we should
assume that the application code is untrustworthy, and isolate the
sensitive data from it. Plaintext input and output should happen in
some sort of isolated frame embedded in the application. The untrusted
application can send ciphertext in and get ciphertext out, but has no
access to the key – neither the raw key material nor any APIs for
performing cryptographic operations with it.

This idea has been floating around for quite some time, and has
perhaps gained some newfound traction lately because of
[Fenced Frames](https://developer.chrome.com/en/docs/privacy-sandbox/fenced-frame/).
Fenced Frames are a new web platform concept for isolating cross-site data
inside frames with limited communication or exfiltration abilities,
and at first glance this idea looks like a promising primitive for
isolating plaintext in E2EE applications.

Personally, I view this direction as unpromising for E2EE, even given
the existence of Fenced Frames. In order of increasing tractability,
the problems I see are:

* It reduces to an age-old security UX problem: __users won’t notice if
  they are using an untrustworthy spoof of the secure input
  surface__. That is, if plaintext is to be isolated from the web
  application, users have to input their plaintext data into one of
  these isolated frames – and they must be able to notice when they
  are inputting data into an untrustworthy surface provided by the
  malicious application. Perhaps there could be some secure UI
  indicator to tell users when they are inputting data into an
  isolated frame, but we’ve seen time and time again that users
  [won’t notice](https://storage.googleapis.com/pub-tools-public-publication-data/pdf/400599205ab5a1c9efa03e2a7c127eb8200bf288.pdf)
  the absence of such an indicator in the event of an attack.
* We can isolate plaintext, but that __doesn’t help with the key
  distribution problem__. Somehow the user agent needs to learn which
  public key to encrypt data with, and we’re back to square one if the
  public key comes from the (possibly malicious) application code, who
  could simply provide a public key with a known private key to gain
  access to the plaintext. We could try to solve this problem by
  somehow bringing public key verification inside the isolated and
  more trusted context – for example, we could define a
  browser-mediated procedure for checking some kind of key continuity
  or offering some kind of browser-mediated verification UX like
  Signal’s safety number checks. But that seems too inflexible; for
  example, it wouldn’t fit into the model where there is a trusted
  identity provider managing long-term identity keys and doling out
  per-session keys, as described in the video call context
  above. Putting private keys and plaintext inside some kind of
  isolated frame doesn’t really buy us anything as long as the
  potentially malicious application code is specifying the public key
  with which to encrypt, and I don’t see a viable way to solve this
  problem.
* It will be __difficult to truly isolate the frame contents from the
  embedding page__, especially given that the embedding page must be
  able to supply arbitrary code to run inside the isolated frame to
  implement functionality like rich text editors. Side channels might
  turn into a game of whack-a-mole, and it might become difficult to
  trust this approach for many of the highly sensitive applications
  that want E2EE. The Fenced Frames explainer
  [addresses](https://github.com/WICG/fenced-frame/tree/master/explainer#ongoing-technical-constraints)
  the side channel issue, noting that exfiltrating data via side
  channels is a blatant type of abuse that could perhaps be
  mitigatable in the ads ecosystem. But side channels would be a
  perfectly viable attack in the E2EE setting. I also worry about
  social engineering – for example, the embedding application
  convincing users to copy and paste plaintext out of the isolated
  context.
* Finally, I predict this approach will just lead to __weird and bad
  UX__. I think it will be challenging to integrate an isolated frame
  like this seamlessly into an application, and that will hinder E2EE
  adoption and usage on the web. Further, there could be a long tail
  of web app and browser bugs due to the odd nature of these
  frames. It’s one thing to have a slightly buggy weird kind of frame
  for showing ads, and quite another to have that slightly buggy weird
  kind of frame carrying the main functionality of an application.

The ideas of isolating plaintext and improving code
integrity/provenance/auditability are not mutually exclusive, and in
an ideal world perhaps we would get both. But in a world of limited
resources, I would choose to improve code
integrity/provenance/auditability for E2EE web applications rather
than pursue the somewhat quixotic goal of isolating plaintext from
application code.