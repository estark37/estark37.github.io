---
layout: post
title:  "Why even a little plaintext matters"
---

This blog post is an expanded version of a Twitter
[thread](https://twitter.com/estark37/status/1417703310258184194) I
posted several years ago about why every website should use
HTTPS. Twitter seems less… readily citable these days, so I thought it
would be good to have a blog post version of it.

The somewhat tongue-in-cheek inspiration for that thread was
[shrug.io](http://shrug.io), a single static page that serves as
a convenient copy-and-paste source for the incredibly useful ASCII art
`¯\_(ツ)_/¯`. Even such a simple and seemingly non-sensitive website
should be accessed over an encrypted and authenticated connection.

## Malware and exploit delivery

Attackers abuse plaintext webpages to deliver malware and browser
exploits. In several well-documented, high-profile, and extremely
disturbing cases, attackers have used unencrypted HTTP redirects to
[deliver](https://www.amnesty.org/en/latest/research/2019/10/morocco-human-rights-defenders-targeted-with-nso-groups-spyware/)
[spyware](https://citizenlab.ca/2023/09/predator-in-the-wires-ahmed-eltantawy-targeted-with-predator-spyware-after-announcing-presidential-ambitions/). Any
visit to any unencrypted website would be sufficient as a vector for
this type of attack. The complexity or sensitivity of the legitimate
destination website is irrelevant to the attack.

There are two arguments that I’ve heard people use to attempt to dismiss this attack vector.

One argument is that the examples I cited are instances of very
well-resourced attackers targeting specific high-value victims, and
that those attacks are going to succeed no matter what we, as
defenders, do – in other words, the
[Mossad/not-Mossad duality](https://www.usenix.org/system/files/1401_08-12_mickens.pdf).
Firstly, this is defeatism. Secondly, the instances I cited are just a
few attacks that have been heavily publicized, and network injection
against plaintext web traffic is cheap and technically trivial. We
should assume that opportunistic attackers distribute malware at scale
via this method unless proven otherwise.

The second argument is more nuanced: there are myriad ways for an
attacker to get a victim to visit a webpage under the attacker’s
control, besides network injection into plaintext traffic. An attacker
can send a link via email or SMS (and even use, for example, an open
redirect to make the link look more harmless), use SEO techniques to
manipulate search results, or find an XSS in a website that the victim
is legitimately visiting. Even if we were to adopt a goal of closing
off every avenue for an attacker to induce a victim to visit a
malicious webpage, that goal would likely be in tension with some of
the fundamental UX and openness promises of the web
platform.

Nevertheless, it’s worthwhile to close off the network injection
vector. The fact that well-resourced attackers use network injection
suggests that other vectors may be costlier, noisier, or less reliable
for the attacker – for example, requiring the victim to fall for
social engineering, or making the attack visible to malware scanning
services.

## Subjectivity of “sensitive”

Some poeple argue that it is okay to encrypt only “sensitive” web
traffic. However, determining what is “sensitive” is subjective,
context-dependent, and technical. The U.S. federal government made
this point in its
[HTTPS-Only Standard](https://https.cio.gov/#:~:text=All%20browsing%20activity%20should%20be%20considered%20private%20and%20sensitive)
in 2015.

In the shrug.io example, consider that copying and pasting text into a
terminal can be
[dangerous](https://thejh.net/misc/website-terminal-copy-paste). In
other examples, consider that network attackers could abuse
capabilities granted to an HTTP website, such as the ability to
communicate with other origins or to
[autoplay video](https://developer.chrome.com/blog/autoplay).
[Web](https://www.chromium.org/Home/chromium-security/deprecating-powerful-features-on-insecure-origins/)
[browsers](https://blog.mozilla.org/security/2018/01/15/secure-contexts-everywhere/)
have long tried to eradicate powerful capabilities from insecure
origins, but as long as HTTP websites exist and are accessible, they
will inevitably have some capability that can be abused. Determining
what these capabilities are and how they might be abused is a
technical, context-dependent task that an average web user should not
be expected to undertake.

## The power of defaults

Finally, simple or allegedly non-sensitive websites should use HTTPS
because it helps the whole web be safer. As HTTPS becomes more
prevalent, even into the long tail, the need to keep plaintext a
first-class citizen diminishes.

For example, as HTTPS usage has grown in the past several years,
Chrome has been able to default to HTTPS for omnibox navigations, and
then for all navigations, providing protection against passive
eavesdroppers. These changes would have been much more contentious in
earlier years when they would have had a negative impact on a larger
number of websites that didn’t support HTTPS.

But there is still work to be done. For more robust default
protections against active attackers, the browser would need to show a
warning before loading an HTTP page. This would be something akin to
[HSTS](https://www.chromium.org/hsts/) by default, which would also,
conveniently, allow browsers to stop maintaining a huge
[hardcoded list](https://scotthelme.co.uk/hsts-preloading/) of
HTTPS-only websites. Browsers can only consider these types of
protections if HTTPS usage is near ubiquitous, which is why website
owners should deploy HTTPS even if they don’t consider it essential
for their individual websites.

<small>_Thanks to my colleague Joe DeBlasio and other teammates on
Chrome who have crystallized many of the points in this post._</small>
