---
layout: post
title:  "What's the right UX for an expired certificate?"
---

Every once in a while, I encounter some variation of the following
question: how can a TLS certificate go from perfectly acceptable one
day to completely insecure the next? In other words, why does the
browser show a scary full-page warning for a certificate that expired
one day, or even one hour, ago – the same as a certificate that is
self-signed, chains to an unknown root, or presents the wrong name?
The premise behind these questions is that an expired certificate
(especially one that is recently expired) is not as bad as a
certificate with some other type of validation error, and thus the
warning UX shouldn’t be as severe.

My answer to this question is three-fold: there are historical and
security reasons to use the same UX for all different types of
certificate validation errors, and there is a set of warning design
problems to consider too.

Looking at these reasons in more detail:

## Historical reasons

Historically, browser designers just didn’t put that much thought into
certificate errors, and I suspect that using a customized UX for
expired certificates was just not on their radar. Thus it became
typical for all types of certificate errors to share the same warning
UX. In other words, it’s the status quo, and maybe no one really
thought that much about changing it or cared enough to do so.

I will note that the warning UX for expired certificates has evolved a
bit over time. Some browsers now try to detect when a certificate
appears to be expired due to an incorrect client clock and prompt the
user to fix their clock. However, I’m not sure if there’s ever been
any appetite to go beyond that.

## Security reasons

Depending on the certificate verifier implementation in use, it may be
incorrect to treat an expired certificate as _only_ expired. That is,
a verifier may short-circuit and reject an expired certificate without
checking that it is otherwise valid. Thus, making the warning UX less
severe for expired certificates may be downright incorrect and
insecure; it may be that there is some other, more severe security
problem with the certificate that the verifier didn’t discover.

Still, suppose we write this off as a technical limitation and assume
that the certificate verifier only tells us that the certificate is
expired if expiration is truly the only error. Even then, there are
security problems with treating expiration as a less severe error than
other validation problems.

Here’s one example of this type of security problem. Expired
certificates cannot generally be revoked. For example, a certificate
may fall off, or not be added to, a
[CRL](https://en.wikipedia.org/wiki/Certificate_revocation_list) if it
is expired. If an attacker compromises the key corresponding to an
expired certificate, or compromises a key but then sits on it until
the certificate expires, the legitimate certificate owner’s only line
of defense is the browser warning UX for an expired certificate. That,
in my opinion, is a compelling reason that the browser warning UX for
an expired certificate should be as severe as for a known key
compromise.

More generally, a browser’s idea of a secure certificate evolves over
time, and there’s no guarantee that an expired certificate conforms to
the requirements that the browser has for certificates today. For
example, suppose the browser introduces new audit requirements for
CAs. After the point at which all certificates issued under the old
audit requirements have expired, it no longer makes sense to treat
expired certificates as if they are as secure as currently valid
certificates, even if we didn’t care about the actual date validity of
certificates. These older certificates were issued under weaker audit
requirements and don’t conform to the browser’s current idea of
security. Certificate expiration is part of how old ideas of security
are phased out and new ones are phased in, but that evolution only
works if expired certificates are actually treated as insecure.

## Warning design reasons

Even if you don’t buy any of my arguments above, and still think that
expired certificates should have a different warning UX than other
certificate errors, my question is: what is that correct UX? What
would it be trying to achieve?

One answer might be that the warning UX should try to alert the server
administrator that the certificate needs to be renewed, without
disrupting users. At the risk of a blatant appeal to authority, I can
tell you from years of experience designing browser security UI that
this is a very hard balance to strike. The most effective way to get a
server administrator’s attention is to get their users’ attention. And
the most effective way to get users’ attention is disruptive full-page
warnings. While unfortunately we didn’t publish it publicly, my team
actually did some research on a similar question of how to design
browser security warnings to most effectively phase out an old TLS
version. We found some marginal statistical significance when using a
softer warning UX to get server administrators to upgrade their TLS
configurations, but the real impact to server configurations came from
full-page warnings telling users that their security was at risk. (h/t
to [Chris Thompson](https://research.google/people/ChrisThompson/) for
designing and running this excellent study)

Another answer might be that the warning UX should try to communicate
to the user that there is some security risk, but not as much risk as
for other certificate validation errors. Again, this is a very hard
balance to strike; there are published
[negative results](https://research.google/pubs/pub43265/) relating to user
comprehension of certificate warnings, and I imagine that trying to
get users to understand and reason about specific types of certificate
warnings is even harder.

At best, I think a less severe full-page warning for expired
certificates – perhaps one that is easier to bypass than the status
quo certificate warning – _could_ effectively incentivize server
administrators to renew their certificates, while being slightly less
disruptive or scary to users and inducing less warning fatigue. But,
in light of the security reasons outlined above, I don’t think it’s
the correct warning design.

Finally, and more philosophically, I think there is some version of
the
[Zero one infinity rule](https://en.wikipedia.org/wiki/Zero_one_infinity_rule)
buried in here somewhere. If you think that it is incorrect to show a
severe full-page warning for a certificate that expired one day ago,
but you think it is correct to show a severe full-page warning for a
certificate that expired three years ago, where is the cutoff? At what
point does an expired certificate become truly insecure? If you don’t
have a principled answer to that question, then I think the answer
should be either zero or infinity: either we treat certificates as
fully invalid the moment they expire, or we treat even
infinitely-expired certificates as (at least somewhat) valid, and I
don’t think many people would argue for the latter approach.

For all of these reasons, customizing the certificate warning UX for
expired certificates (beyond obviously good ideas like prompting users
to fix incorrect client clocks) feels fraught. We’re better off trying
to improve deployment of automated certificate renewals so that
accidentally expired certificates become far less commonplace.