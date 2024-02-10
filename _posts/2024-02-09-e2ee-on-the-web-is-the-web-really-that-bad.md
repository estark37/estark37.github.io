---
layout: post
title:  "E2EE on the web: is the web really that bad?"
---

In my
[last blog post](https://emilymstark.com/2023/09/09/e2ee-on-the-web-isolating-plaintext.html),
I discussed why people often view the web as a uniquely unsuited platform
for implementing end-to-end encryption (E2EE).
[This view](https://emilymstark.com/2023/09/09/e2ee-on-the-web-isolating-plaintext.html#:~:text=On%20the%20web%2C%20there%20is%20no%20long%2Dterm%20trustable%20notion%20of%20what%20%E2%80%9Cthe%20application%E2%80%9D%20is%2C%20thus%20there%20is%20no%20sense%20in%20storing%20long%2Dterm%20secret%20keys%20that%20the%20application%20can%20use%20%E2%80%93%20because%20any%20one%20of%20a%20zillion%20TLS%20connections%2C%20or%20edge%20or%20origin%20servers%2C%20could%20be%20compromised%20to%20steal%20the%20long%2Dterm%20keys.)
is that the web doesn’t offer a long-term trustable notion of what the
application is. In that earlier post, I explored the idea of treating
the application as untrustworthy and isolating sensitive data from
it. In this post, I’m going to pontificate on whether web applications
are truly less trustworthy than native applications, especially in an
E2EE setting, and if so, how we should bridge the gap. The gap is
narrower than it appears at first glance, especially with desktop
applications. To close it, though, the devil is in the (UX- and
deployment-related) details.

## Is the web really worse?

Often, security experts argue that the web isn’t suited for E2EE
applications because of the vast attack surface for code injection –
abusable by the developer or by an external attacker. For many web
applications, a web browser receives and runs code from a zillion
servers, retrieved over a zillion TLS connections. Compromising just
one of those servers or connections is enough to inject untrustworthy
code into the application, likely without detection. This is different
from any other mainstream platform, all of which offer stronger
statements about the integrity and/or provenance of the application
code.

I think this argument is basically factually correct, but less
compelling than it appears at first glance.

### Web versus mobile

On mobile platforms, the allegedly stronger security model comes from
the policies, review processes, and distribution models of app stores,
as well as the operating system’s design of security principles and
use of sandboxing. Apps are signed with an offline developer key, and
usually distributed through a store, with human review against
anti-abuse policies and a centralized distribution point operated by
the OS vendor. Individually, each of these properties might not
provide much in the way of hard guarantees, but together they form a
nice defense-in-depth model.

If we compare mobile to web, mobile is convincingly a more secure
platform for developing high-security applications. This is especially
true if the threat model includes the application developer (e.g.,
E2EE messaging), because there is another party (the OS vendor)
besides the application developer involved in distributing the
application.

### Web versus desktop

On desktop platforms, the situation is much messier:

*  Many desktop applications silently auto-update directly from the
   application vendor’s servers, which looks a lot more like the web’s
   code distribution model.
*  Desktop applications these days are often built identically to web
   applications and then packaged up for native distribution, e.g.,
   through [Electron](https://www.electronjs.org/).
*  Desktop applications might run different code depending on dynamic
   data (such as experiment configurations) that is loaded and
   reloaded often, possibly relying on TLS alone to secure these
   configurations from injection.
*  Unlike web apps, desktop applications are often written in
   vulnerable memory-unsafe languages like C++.
*  Desktop applications are often installed at the same privilege level
   and not isolated from each other as well as mobile apps or even web
   apps.

Compared to the web, I would say that desktop’s main advantage in
application security is the common use of code signing with an offline developer key -- and, as a corollary, the notion of a concrete application that can be signed, as opposed to the continuous river of code that forms a web application. Does this delta matter? I'm not sure:

*  The security value of code signing boils down to the security of the
   code signing PKI as well as the UX of verification failures, and I
   don’t have a lot of confidence in either of those aspects of the
   system. Further, the legitimate owner of the code signing key is an
   attacker in the E2EE threat model; thus code signing is only partially
   relevant to the E2EE threat model.
*  The idea of having a discrete application is appealing, but I find
   it hard to say exactly why. Perhaps there are some advantages in
   terms of auditing and forensics, or in tightly controlled
   environments like enterprise deployments, where a knowledgeable
   administrator might qualify updates manually. It also provides a
   basis on which to build other security mechanisms, such as
   comparing the version of the code that one user is running to what
   everyone else is running – but in practice, this isn’t widely
   deployed on today’s desktop operating systems.

That said, desktop application security is getting better over
time, with new technologies like
[Mac OS notarization](https://developer.apple.com/documentation/security/notarizing_macos_software_before_distribution),
improvements in code signing UX, more store-like distribution models,
and stronger isolation primitives. If there historically hasn’t been a
huge gap between desktop application security and web app security, it
would be a shame for that gap to widen and leave the web behind. But,
today, I don't believe the gap is very wide, especially if you
consider the application developer an attacker, and that is why I
looked askance at anyone who claims the web isn’t suited for E2EE
while happily distributing desktop versions of their E2EE apps.

## Bringing the web closer to parity: a UX problem

If we do care about the delta in security model between the web and
other platforms, then we could build some kind of code bundling and
signing mechanism for web applications, perhaps with some kind of
transparency layer on top to make the code publicly auditable and make
it harder to target specific users with malicious code. A
bundling/signing/transparency solution for the web could probably be
built out of some of a collection of mechanisms that already exist or
have at least been explored. Related ideas include
[Subresource Integrity](https://developer.mozilla.org/en-US/docs/Web/Security/Subresource_Integrity),
[Isolated Web Apps](https://github.com/WICG/isolated-web-apps/blob/main/README.md),
[Signed Exchanges](https://developer.chrome.com/blog/signed-exchanges)
and [Web Packaging](https://wicg.github.io/webpackage/draft-yasskin-dispatch-web-packaging.html),
Meta’s [Code Verify extension](https://engineering.fb.com/2022/03/10/security/code-verify/),
and [source code](https://github.com/twiss/source-code-transparency)
and [supply chain](https://datatracker.ietf.org/wg/scitt/about/)
transparency proposals.

That is, many of the pieces of the solution already exist. It’s not
hard to imagine a mechanism for bundling up a web application, signing
it with an offline key, and distributing that to the browser to run in
isolation, without allowing any dynamic code loading. The daunting
challenges that remain are user experience questions and fiddly
deployment details:

*  **Website opt-in/opt-out.** Any kind of code
   bundling/signing/transparency mechanism has to co-exist with the
   open web as it exists today. But then how does the browser know
   whether example.com is a normal web application, with all the
   normal mechanisms of dynamic code loading, rather than a special
   bundled/signed web application? If example.com is normally a
   special bundled/signed web application, but the attacker
   compromises it to start serving a regular legacy web application,
   how does the browser know anything is wrong? Or if the browser
   simply punts the information to the user, how should the user
   know that anything is wrong? I don’t think there is a good answer
   for this yet.
     *  _Separate origin_: The browser could use separate web origins
        for the signed/bundled version of an origin and the regular
        one. This would isolate secret data like long-term identity
        keys. However, separating one “physical” origin into different
        “logical” origins has historically proven to be a complex
        endeavor, and it opens up UX questions like whether the
        regular version of the origin could phish the signed/bundled
        version to break security guarantees.
     *  _Signal in Certificate Transparency_: One proposal is to stick
        a
        [signal](https://github.com/twiss/source-code-transparency/issues/7)
        into Certificate Transparency logs, but this is somewhat
        clunky, abusing the web PKI and Certificate Transparency for a
        purpose for which they weren’t designed. Furthermore, it
        yields a weak security model. A downgrade could only be
        detected after the fact, within Certificate Transparency’s
        somewhat nebulous
	[security guarantees](https://emilymstark.com/2022/08/23/certificate-transparency-is-really-not-a-replacement-for-key-pinning.html).
     *  _Sticky header_: One could imagine using an
        [HSTS](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Strict-Transport-Security)-like
        header, by which bundled/signed web applications instruct the
        browser to only accept bundled/signed code for the domain. The
        browser remembers this instruction for some configurable
        period of time. This would probably be okay, but it’s not
        ideal. HSTS’s security guarantees only hold if the user visits
        the site to refresh the browser’s cached directives at some
        predictable cadence, and it doesn’t protect the first visit
        from downgrade. Any kind of “sticky” domain-based mechanism
        becomes problematic when a domain owner transfers the domain,
        lets it expire, or changes their mind about how they want to
        configure their website. And HSTS has inspired some
        unmaintainable
        [hacks](https://scotthelme.co.uk/hsts-preloading/) for sites
        that need stronger security. These problems are all somewhat
        acceptable for HSTS because, in the very long run, we might
        hope to deprecate HSTS as HTTPS gradually becomes more
        strongly preferred in browsers. But for code bundling/signing,
        there isn’t any long-term story in which bundled/signed
        applications become the default or only supported mechanism,
        so these drawbacks are perhaps less tolerable.
*  **Failure UX.** Whatever the solution for website opt-in/opt-out,
   there will have to be some UX in place for when the browser
   expects a valid bundled/signed web application but doesn’t get
   one: for example, the signature is invalid, the browser expected
   a bundled/signed application but got a legacy resource, or there
   was some failure in the code transparency verification (if such a
   thing comes to pass). What happens then? Can the user bypass the
   error? If so, then, as with other security warnings, the warning
   must be designed just so to allow some users to make reasonable,
   informed judgment calls without putting less informed users at
   risk, and without contributing to a broader warning fatigue or
   habituation effect in E2EE apps. If the browser does not let the
   user bypass the error, then the entire system must be designed
   carefully such that false positive errors are virtually
   nonexistent.
*  **Development and deployment experience.** Finally, whatever the
   mechanism for app bundling/signing/transparency, it has to be
   usable by developers. I find that it’s tempting to think of E2EE
   as a niche feature, thus app bundling/signing as something that
   only a small number of highly security-sensitive web applications
   would use. It’s tempting to make the problems more tractable by
   assuming that the web applications would be small and static, or
   amenable to custom heavy-handed development workflows, or built
   by highly sophisticated developers. In reality, some of the
   largest and most complex web applications in the world – such as
   facebook.com – need E2EE. The target market includes web
   applications with vast amounts of legacy code, complex bespoke
   development workflows, highly performance-sensitive userbases,
   you name it. Further, if we designed E2EE on the web as a niche
   feature, it would be a self-fulfilling prophecy. By designing it
   as a mass-market feature, it increases the chances that E2EE
   becomes table-stakes for modern applications on any
   platform. Usable code integrity and provenance mechanisms for the
   web would also benefit many other types of security-sensitive
   applications besides E2EE apps.

In the effort to bring E2EE to the web, I hope these challenges in user experience and developer ergonomics don’t get forgotten or underestimated.

<small><i>Thanks to David Adrian, Richard Barnes, Nick Doty, Wendy
Knox Everette, Ryan Hurst, April King, Jon Millican, and Chris Palmer
for feedback on this post.</i></small>