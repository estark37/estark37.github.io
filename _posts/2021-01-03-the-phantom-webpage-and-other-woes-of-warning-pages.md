---
layout: post
title:  'The phantom webpage, and other woes of warning pages'
---

2020 had a lot of non-work lowlights, but one highlight is that my team finished
a long-running re-architecture of Chrome’s full-page security warnings. This
post gives a high-level explanation of why this re-architecture was necessary
and some of the lessons I learned about large in-place software redesigns.

In Chrome, full-page security warnings are known as interstitial pages, or
interstitials. Interstitials are in-between pages that sit in front of another
(usually dangerous) page until the user takes some action to load the real page.
Well-known examples include HTTPS certificate warnings and Safe Browsing
phishing/malware warnings, and there are several other lesser known examples
too.

{% include image.html url="/assets/cert_error.png" description='One example of a commonly encountered interstitial page, the HTTPS certificate warning in Chrome.' 
short_description='HTTPS certificate interstitial in Chrome' %}

If you’re interested in understanding the technical details in this post, it
might be helpful to first learn a bit about navigation in Chromium via my
colleague [Nasko Oskov](https://twitter.com/nasko)’s [Lifetime of a
navigation](https://netsekure.org/2017/02/02/chromium-internals-lifetime-of-navigation/)
post or [Chromium university talk](https://youtu.be/mX7jQsGCF6E). A rough
understanding of Chrome’s [multi-process
architecture](https://www.chromium.org/developers/design-documents/multi-process-architecture)
might also be useful.

## Out with the old: overlay interstitials

Since the dawn of Chromium time, interstitial pages were implemented with an
overlay approach. When it was time to show an interstitial, Chrome paused the
problematic request, launched a new renderer process, navigated it to a `data:`
URL that encoded the interstitial page contents, and displayed it atop whatever
page was previously visible.

There’s nothing obviously or predictably wrong with this approach, but over time
the implementation accumulated a
[pile](https://bugs.chromium.org/p/chromium/issues/list?q=hotlist%3Dfixedbycommittedinterstitials&can=1)
of bugs. It eventually became clear that
the design itself was prone to bugginess. There were a few different classes of
bugs that emerged, to which I will now give colorful names:

* **The phantom page**: When an overlay interstitial was shown, the previous
  page was still alive but invisible underneath it. The previous page wasn’t
  unloaded in case the user decided to leave the warning page, in which case
  they should be taken straight back to where they were -- but while the
  interstitial was showing, the user shouldn’t be aware that the previous page
  was still there. But what would happen if the previous page played audio or
  video, ran expensive JavaScript code, popped up a permission prompt or alert
  dialog, or otherwise made its presence known while the interstitial was
  showing? There was a slew of whack-a-mole bugs where behaviors of the previous
  page had to be individually suppressed to prevent users from encountering some
  very strange situations (imagine a permission prompt for “a.com” showing up on
  top of a security warning for “b.com”).
* **The odd renderer out**: Interstitials were a special type of renderer
  process, and various other Chromium features had to be individually taught how
  to deal with them. For example, DevTools, accessibility features, and context
  menus were all eventually found to be broken and needed special-case fixes to
  work with overlay interstitials.
* **Glitches in the navigation matrix**: Overlay interstitials were a special
  case in Chromium’s navigation stack. In some ways they acted like normal
  history entries and in some ways they didn’t, so they were their own special
  species of navigation. This nest of special cases was hard to understand, much
  less test thoroughly, leading to all sorts of strange buggy navigation
  behaviors: for example, various unusual user interactions like refreshing an
  interstitial rapidly would sometimes cause it to disappear.

Because of these problems, over the years, overlay interstitials became somewhat
notorious among Chromium developers. For many of these bugs, fixing them would
have been too costly to be worth it, so they languished open on the bug tracker
for years. I often joke that re-architecting interstitials was the best project
I’ve ever worked on, because everyone was so united in their dislike of the
overlay approach. There were no politics to navigate, no stakeholders to
convince, no outreach to be done. It was a project composed almost entirely of
purely technical problems, and projects like that don’t come along very often!

Lest I come across as too negative towards overlay interstitials, I want to make
two clarifications:
  
First, I bear no ill will or negative judgement towards whomever originally designed
the overlay approach. It would have been hard to predict why the design
wouldn’t work out, especially because many of the problematic interactions
were with features that probably came along after interstitials were
implemented. I’ve poked at other browsers and I think (though I’m not sure)
that at least some of them take an overlay approach too. I have no doubt that
the developers of both Chromium and these other browsers were making
reasonable, intelligent decisions when adopting this design.

Second, there are some nice properties of overlay interstitials. For example, a
separate special-case renderer is prone to bugginess, but is also naturally
good for security. Normal webpages shouldn’t share a renderer process with
security warnings, or else they might be able to maliciously bypass or
suppress the warnings. Another useful property of overlay interstitials is
that they’re not really specific to navigation events. Most of the time, when
the browser shows an interstitial, it’s during a navigation from one page to
another, but this isn’t always the case. Safe Browsing can trigger warnings
when subresources load, and popping up an overlay interstitial works almost
the same in that case as during a navigation. It’s nice to be able to use the
same machinery whether the interstitial is triggered during a navigation event
or at some other point during a page’s lifetime. These properties will
probably make more sense later on, as a contrast to the replacement design
that I’ll explain next.

## In with the new: committed interstitials

To replace overlay interstitials, we designed “committed interstitials,"
so-called because it uses the pattern of committing a warning page to complete a
navigation, rather than overlaying the warning page on top of a previous
navigation. If we are navigating from a.com to b.com and b.com presents an
invalid HTTPS certificate or triggers a Safe Browsing warning, instead of
overlaying a warning interstitial on top of a.com, we instead complete the
navigation to b.com and display the warning page in place of any b.com content.
This is the same as how non-security error pages work: if we can’t resolve the
name b.com, for example, we complete the navigation and display the error page
in place of any b.com content.

By design, this approach fixes all the bugginess I describe above. The previous
page is unloaded just like in a normal navigation, so there is no phantom page.
The interstitial uses a regular renderer just like any other error page, so
there are no special cases needed when, for example, DevTools tries to attach to
it. And, with one exception[^1], interstitials behave like any other navigation
entry, so there are no glitches in the navigation matrix.

This design is, however, by no means simple to implement. The complexity is that
most interstitials are bypassable. In other words, the user can choose to ignore
the warning and proceed to the dangerous page. That’s what fundamentally
separates interstitials from error pages.

The overlay interstitial pattern naturally lends itself to bypassability: when
the user chooses to bypass the warning, Chrome dismisses the overlay and
un-pauses the in-progress request with a callback that was provided when the
overlay was triggered.

But with the committed interstitial pattern, there’s no in-progress navigation
request to un-pause; we already completed the navigation. The general pattern we
follow instead is to modify some state to record the bypass, and then reload the
page; the reload picks up the modified state, causing Chrome to suppress the
interstitial and load the page normally.

This reload approach might sound a little hacky at first, but it’s actually
quite similar to what often happens with overlay interstitials, but just
implemented at a different layer. Overlay interstitials are often triggered by
network-stack events, like a certificate error. In this case, the network stack
opens a connection and then encounters a problem and surfaces it up to the
navigation stack: “Hey, we’ve got a problem, so I’ve paused the request; call
this callback when you’ve decided what to do about it.” The navigation stack and
higher layers display the overlay, record the user decision somewhere if the
user decides to bypass, and then call the callback, telling the navigation stack
to resume the request. Under the hood, though, the navigation stack isn’t really
resuming anything on the wire: it’s just retrying the connection or resending
the request. With committed interstitials, we’ve removed this network-stack
abstraction of a paused request. When the network stack encounters a problem,
the request is represented as done throughout the code base, and once the user
decides to bypass the problem, the navigation stack reloads the page, trying a
new connection. In other words: in most cases, committed interstitials and
overlay interstitials look the same on the wire, even though they are
represented differently in code.

The main challenge with this approach is keeping track of this bypass state and
scoping it appropriately. Sometimes not all requests should share the same
bypass state. In particular, in a few instances where bypasses are one-time-only
and shouldn’t affect any other navigations, we have to be careful to not let the
bypass bleed over anywhere else.

#### An authentication adventure

Keeping track of bypass state is even more complicated when user interactions
aren’t binary (bypass the warning vs. leave the site). Authentication prompts
are one such example. By “authentication prompt,” I mean a modal dialog that
prompts for [HTTP
authentication](https://developer.mozilla.org/en-US/docs/Web/HTTP/Authentication)
credentials, such as for HTTP Basic Auth or proxy authentication. In these
dialogs, users can enter correct credentials, enter incorrect credentials, or
cancel the dialog, leading to more complicated bookkeeping than for an
interstitial that has a binary outcome.

{% include image.html url="/assets/http_auth_prompt.png" description='An HTTP Basic Authentication prompt in Chrome, which, surprisingly, is implemented as an interstitial.' short_description='HTTP authentication prompt in Chrome' %}

“Wait a second,” you might be thinking. “Authentication prompts aren’t security
warnings. What do they have to do with interstitials?”

The answer lies in a subtle aspect of the screenshot above. Notice how the
webpage behind the modal dialog is blank. This is a security measure. If a
navigation from a.com to b.com triggers an authentication prompt, we don’t want
the user to become confused and unknowingly enter credentials for a.com in the
b.com prompt. For this reason, we want to show a blank page underneath the
prompt and we want to show “b.com” in the address bar, to make it as clear as
possible that the user is authenticating to b.com, not a.com. We consider the
URL in the authentication prompt itself too easy to miss on its own.

In other words, we want to show the modal dialog on top of a blank page between
a.com and b.com, which will be dismissed if the user enters correct credentials
and proceeds to load the page. That starts to sound like an interstitial page,
even though it’s not a security warning. Hence, authentication prompts for
cross-origin navigations are implemented as interstitials.

With overlay interstitials, when the network stack encountered an authentication
challenge, the navigation stack would display a blank interstitial page while
the network stack waited to be told what to do. When the user entered
credentials, the navigation stack would pass these credentials to a callback,
causing the network stack to resend the request with the credentials (and cache
them for future requests).

With committed interstitials, when the network stack encounters an
authentication challenge, the navigation stack ends the request and displays a
blank page in place of any content. When the user enters credentials, they are
cached, and the navigation stack reloads the page, which resends the requests
with the credentials from the cache.

This authentication interstitial is tricky to fit into the committed
interstitials pattern, because of the variety of possible user interactions:

* The easy case is that the user **enters correct credentials**. This outcome
  fits nicely into the committed interstitials pattern: modify state to store
  the credentials, and a reload picks up the modified state.

* When the user enters **credentials that turn out to be incorrect**, reasonable
  people can disagree about what the product behavior should be; Chrome re-shows
  the prompt to give the user another chance. This is fairly simple to implement
  because it just repeats the same flow as when the page was loaded the first
  time: the server responds to the incorrect credentials with another
  authentication challenge, the network stack sends the authentication challenge
  up to the navigation stack, and the navigation stack terminates the navigation
  and shows an authentication prompt.

* When the user **cancels the dialog**, we want to show error page content from
  the server (e.g., the body of a
  [401](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/401) response),
  which should hopefully explain to the user why they need to authenticate. This
  is the trickiest case because we need to somehow display the response body
  from the server even though we discarded the server’s response to display the
  blank interstitial page. It turns out that network/navigation infrastructure
  isn’t really set up to do this, so we do a bit of a hack and simply reload
  when the user hits Cancel, suppressing any triggered authentication prompt on
  the reloaded request.

Overall there ends up being a fair amount of fiddly state to keep track of when
we should show an authentication prompt, when we should read and display the
response from the server, etc.

Authentication prompts also, belatedly, revealed a surprising fact about the
committed interstitials re-architecture: it’s actually
[web-exposed](https://github.com/whatwg/fetch/issues/1132). (“Web-exposed” is a
jargon-y way of describing a browser change that web developers might notice or
care about, usually because it could break websites that were depending on the
old behavior or because it violates a standard or documentation.) Previously,
pausing a request mid-navigation meant that Service Workers wouldn’t see the
response unless the user interacted with the interstitial in a way that caused
the request to proceed (e.g., entering correct credentials in an authentication
prompt or bypassing a security warning). But with committed interstitials,
Chrome completes the navigation, which will cause Service Workers to see an
error response. This is far more noticeable in the case of authentication prompt
interstitials, because they happen under normal configurations, whereas security
warnings are exceptional cases. I haven’t quite figured out what to do about
this problem yet.

## The morals of the story

Mostly the point of this post is that I thought some of these implementation
details might be interesting to other Chromium nerds. But, I also think there
are some lessons that might generalize to other architectural changes in large,
mature codebases:

* **Don’t let the perfect be the enemy of the good.** We had to make some
  compromises -- sometimes in code complexity, sometimes in product
  functionality -- when solving technical roadblocks. To continue to make
  progress, our mantra was that a few rough edges wouldn’t be worse than the
  mess that overlay interstitials create. It’s better to have a few known issues
  than dozens.
* **Respect legacy code.** There wasn’t anyone around to defend overlay
  interstitials, so it was easy to assume that they were a well-intentioned
  early design that simply didn’t work out well in practice. Over the course of
  the project, I actually became more sympathetic to the overlay approach and
  discovered more advantages of it. I probably feel more positively towards
  overlay interstitials now than when we started the project, even though I
  still firmly feel that it was the right decision to move away from them.
* **Frontload the edge cases.** This project was definitely a case of the last
  10% being 90% of the work. If I were involved in a redo of this project, I
  would tackle some of the edge cases earlier. The edge cases were hairy because
  they didn’t fit well into the patterns we established in the rest of the
  project: namely, Safe Browsing subresource interstitials and authentication
  prompts. Hidden costs were everywhere for these corners of the project:
  adapting tests, dealing with incompatibilities, complexity that increased
  ramp-up time as people left or joined the team… This project took years longer
  than we initially estimated, which is why it was such a highlight to finally
  wrap it up.

<small><i>Thank you to the many awesome teammates who worked on this project,
especially Carlos Joan Rafael Ibarra Lopez, Livvie Lin, Lucas Garron, and Nasko
Oskov.</i></small>

[^1]: The exception is Safe Browsing subresource interstitials, as I alluded to
    previously -- i.e., interstitials that pop up at some point after page load.
    Overlay interstitials are relatively usable when it comes time to show an
    interstitial that isn’t associated with a navigation event. Committed
    interstitials are much more tied to navigation events, because the whole concept
    is to complete the navigation with the contents of the interstitial page, rather
    than leaving the navigation paused. Thus some of the complexity associated with
    overlay interstitials got inherited by subresource interstitials, though it was
    still significantly simplified as it only needed to accommodate this one
    specific use case.