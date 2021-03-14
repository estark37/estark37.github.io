---
layout: post
title:  'Tips for reading web standards'
---

People have written [a](https://ietf.org/about/participate/)
[fair](https://www.w3.org/wiki/How_To_Contribute)
[amount](https://infrequently.org/2018/06/effective-standards-work-part-1-the-lay-of-the-land/)
about how to contribute to web and web-adjacent standards, but before you can
contribute to them, there’s an essential first step: you have to be able to read
them. Before I worked on a web browser for a living, standards intimidated me,
and I would steer clear of them in favor of friendlier documentation like
[MDN](https://developer.mozilla.org/en-US/). But as I began to work on Chrome, I
found that I needed to be able to read standards, and eventually write and edit
them as well. Nowadays, I’ll typically go to a standard as my preferred form of
documentation; sometimes it can be more efficient than reading other
documentation that can be more out of date or imprecise. (Just as sometimes it
can be more efficient to read the code than the documentation!)

Here are some tips that I find helpful when navigating the web standards world.

## A little bit of context helps

Most standards development that is relevant to web development happens in either
the W3C, the WHATWG, or the IETF. (I’m ignoring the ECMAScript standard that
underlies JavaScript because I’ve never had to deal with it and have nothing
helpful to say about it.) These venues have different scopes and conventions,
and I think this context can help make their documents more comprehensible:

  * The IETF deals with the internet as a whole, whereas W3C and WHATWG are
    particularly focused on the web. There is some overlap between these
    areas, of course. The IETF would typically standardize something like a
    transport protocol, whereas the W3C would typically standardize something
    like a CSS feature. An HTTP header, for example, could show up in either
    group, depending on its meaning and usage. The WHATWG tends to focus on
    core browser technologies like [HTML](https://html.spec.whatwg.org/) and
    the algorithms by which browsers [fetch](https://fetch.spec.whatwg.org/)
    HTTP resources.
      * Corollary: When a W3C/WHATWG document talks about a user agent, it’s
        usually referring to a web browser like Google Chrome or Mozilla
        Firefox. When an IETF document talks about a user agent, it is usually
        referring to any HTTP client, which might be a web browser or might be
        something quite different from a web browser like `curl`.
  * You might also encounter the [WICG](https://www.w3.org/community/wicg/), a
    W3C group where standards ideas are documented and iterated upon before
    they graduate to being real grown-up specs. For the most part, you can
    read WICG documents just like any other W3C spec, though they might be
    less polished, less precise, or changing more rapidly.
    
## Always read the editor’s/latest draft

Many standards are living documents, updated frequently as browsers evolve. It’s
usually best to read the latest version, which is sometimes called the editor’s
draft. For example, Googling for “Service Workers spec” might get you to
[https://www.w3.org/TR/service-workers/](https://www.w3.org/TR/service-workers/),
but you’ll want to click on the Editor’s Draft link on that page to get to the
[latest version](https://w3c.github.io/ServiceWorker/v1/).

## [W3C] Read the explainer cover-to-cover.

W3C standards often come with a less formal document called an explainer. The
explainer details the motivations, what problems it’s trying to solve, example
use cases, design decisions, and privacy/security considerations, all at a
higher level than how that standard itself is written. For example, the
[`paymentRequest`
explainer](https://github.com/zkoch/paymentrequest/blob/gh-pages/docs/explainer.md)
includes example client code and a number of design considerations. The
explainer often contains useful context that makes the spec itself more
approachable.

## Rarely read the spec cover-to-cover.

I like to think of specs more like code than prose. When I need to figure out
how a part of a system works, I don’t typically sit down and read the relevant
source file from beginning to end. Instead, I’ll find an entry point -- maybe a
particular function that looks relevant to what I’m trying to understand -- and
follow the control flow from there as needed. When I finish reading the
function, I usually won’t keep reading to the next function below it; instead, I
might need to read functions that this function calls, or see what code calls
this function.

This is how I read specs, too. Indeed, many specs are written as a collection of
algorithms written in a very high-level pseudocode with some expository text
around them. Reading the algorithms top to bottom rarely makes any sense.
Instead, you should look for the algorithm that looks most relevant and start
there; “step out” to its entrypoints or “step in” to other algorithms that it
calls as needed. Note that stepping out/in may involve jumping in to a different
spec.

For example, suppose that I want to know whether Subresource Integrity applies
to 404 responses. [Subresource
Integrity](https://w3c.github.io/webappsec-subresource-integrity/) is a web
browser feature that allows developers to include external subresources while
guaranteeing that they are the expected resources. The spec adds an `integrity`
attribute to a few HTML elements that fetch resources. The attribute’s value is
a hash of the expected resource, and the browser verifies that the resource
matches the hash as it is loaded.

So, my question is whether browsers are supposed to check the `integrity`
attribute when the resource request results in a 404 response. I start with the
[Subresource Integrity editor’s
draft](https://w3c.github.io/webappsec-subresource-integrity/); a quick search
reveals that this spec doesn’t say anything about 404s specifically (or 200s,
for that matter). Scanning the Table of Contents, I scan through the names of
algorithms in the [Response verification
algorithms](https://w3c.github.io/webappsec-subresource-integrity/#verification-algorithms)
section, and see that [3.3.5 Does response match
metadataList?](https://w3c.github.io/webappsec-subresource-integrity/#does-response-match-metadatalist)
looks like the main algorithm that checks the `integrity` attribute. It doesn’t
say anything about 404s, so I check an algorithm that it calls into which looks
possibly relevant: [3.3.2. Is response eligible for integrity
validation?](https://w3c.github.io/webappsec-subresource-integrity/#is-response-eligible)
Nothing about 404s there, either. From there I hop over to the [Fetch
spec](https://fetch.spec.whatwg.org/) and search for SRI to see where Fetch
calls into SRI. (There’s a little bit of institutional knowledge here -- I
already know that Fetch is often the right place to look when I’m looking for
something that happens while a resource is being loaded from the network.) In
Step 15 of [Main Fetch](https://fetch.spec.whatwg.org/#main-fetch), I see that
network errors are exempted from SRI checks, but 404s probably wouldn’t be
considered a network error, so I conclude that SRI is indeed applied to 404
responses.

The general point is that I answered this question by following references
within and among standards, not by reading any one particular standard straight
through.

## Make a mental glossary

Like many types of technical writing, standards often use fancy language for
simple concepts. I gave an example above: you can often substitute a simple
phrase like “web browser” for “user agent”. Another example is “browsing
context” in
[HTML](https://html.spec.whatwg.org/multipage/browsers.html#browsing-context),
which (as the spec notes) you can often think of as a tab or window in a
browser. Usually, I find that making these substitutions as I read improves my
comprehension.

One final tip: spec editors usually don’t bite, and their documents aren’t set
in stone. If you find a problem with a spec, whether it’s a typo or a major
error or a passage that’s difficult to understand, there’s often a link to a
GitHub repo and/or email list where you can report it -- and the editors will
usually be happy and appreciative to hear from you.