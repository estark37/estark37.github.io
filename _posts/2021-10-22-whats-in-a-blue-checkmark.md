---
layout: post
title:  "What's in a blue checkmark?"
---

Blue checkmarks are Twitter’s way of indicating
[verified user accounts](https://help.twitter.com/en/managing-your-account/about-twitter-verified-accounts).
Other social media platforms like Instagram have similar features. I
have absolutely zero inside knowledge about how the blue checkmark
program works behind the scenes, but I’ve always found it to be an
interesting feature from a security and anti-abuse perspective. What
does the blue checkmark actually mean, and what do people think it
means? What would happen (if anything) if Twitter had a bug where all
the blue checkmarks disappeared for a day?

Twitter has gotten a lot more transparent recently about what the blue
checkmark means and is meant to achieve. Their documentation says that
it’s used to mark authentic accounts of public interest. But there is
still a lot to ponder about what those words mean (what’s “public
interest”? what’s “authentic”?) and why this is a useful feature. The
verification program could be motivated by mis-/disinformation,
harassment and abuse (e.g. preventing impersonation), scams and
phishing, or some combination of the above. It’d be fascinating to
know what Twitter’s internal success metrics (if any) are for the blue
checkmark feature.

## Examples

Here are some interesting recent examples of why account verification is a complex feature:

* [FBI](https://twitter.com/fbicbs): a verified Twitter account for a
  TV show called “FBI”. My friend [Eric
  Mill](https://twitter.com/konklone) pointed me to this example and
  questioned why an account that doesn’t represent the actual [Federal
  Bureau of Investigation](https://twitter.com/FBI) should get to use
  the display name “FBI” with a blue checkmark next to it. Apparently,
  verified accounts are allowed to have display name collisions, or
  perhaps aren’t proactively identified.
* [A mistakenly verified fake account of author Cormac McCarthy](https://www.theguardian.com/books/2021/aug/03/twitter-admits-it-verified-fake-account-of-author-cormac-mccarthy).
  Earlier this year, Twitter mistakenly verified a parody Cormac
  McCarthy account -- and they did so proactively after a viral tweet
  from the account. Oops.
* [Many examples of verified accounts spreading mis- or disinformation](https://www.motherjones.com/politics/2019/04/twitter-verified-accounts-misinformation/).
  If the goal of the blue checkmark is to help users identify
  high-quality sources of information, it clearly isn’t a perfect
  solution.

None of these examples serve to illustrate that the blue checkmark is
a bad feature, but rather that it has a complex value
proposition. There are at least two major sources of complexity. One
is that the identity verification process itself is likely subject to
human error and judgement, and quite possibly exploitable too. The
other is that it’s not clear (to me, at least) how users interpret and
act on the blue checkmark.

## Does the blue checkmark have a threat model?

If you, like me, pay attention to the world of web browser security,
the blue checkmark feature probably reminds you of a browser security
feature: [Extended Validation (EV)
certificates](https://en.wikipedia.org/wiki/Extended_Validation_Certificate). An
EV certificate is a type of HTTPS certificate that entails an identity
verification process beyond the normal domain name control validation
that is more typically involved in obtaining an HTTPS
certificate. Browsers used to show a special green indicator in the
address bar for EV certificates, though recently there has been a
[trend](https://www.troyhunt.com/extended-validation-certificates-are-really-really-dead/)
towards making this identity indicator much more subtle.

EV indicators and blue checkmarks are both indications of some kind of
identity verification process. They both have some ambiguity around
their threat models and value. And they both have dual sources of
complexity: the identity verification process itself and the question
of how people comprehend the identity verification indicator.

For EV, most security experts define the threat model around phishing:
in theory, users would not enter credentials into a phishing page
because they would be tipped off to the attack by the absence of the
EV indicator in the browser UI. Because EV has at least the semblance
of a threat model, it’s possible to design experiments that measure
whether the mechanism actually protects against the intended
threats. On the identity verification process itself,
[researchers](https://arstechnica.com/information-technology/2017/12/nope-this-isnt-the-https-validated-stripe-website-you-think-it-is/)
have obtained proof-of-concept certificates with misleading names and
cross-jurisdiction name collisions. On comprehension of the EV
indicator, early on in EV’s life, several lab studies
[failed](https://www.adambarth.com/papers/2007/jackson-simon-tan-barth.pdf)
to show that it achieved its anti-phishing goals. My team on Chrome
ran a [field
study](https://storage.googleapis.com/pub-tools-public-publication-data/pdf/400599205ab5a1c9efa03e2a7c127eb8200bf288.pdf)
showing that experimentally removing the EV indicator didn’t change
users’ interactions with websites -- an unexpected result if you
believe that the EV indicator helps users identify phishing attacks.

On the other hand, I’m not aware of much research (especially computer
science research; send me links!) about the blue checkmark identity
verification process or how users comprehend the blue checkmark. I
suspect that if you asked 3 security experts about the blue
checkmark’s threat model, you’d get 3 different flavors of answers:
(1) impersonation and abuse, (2) mis-/disinformation, and (3) it’s not
a security feature and doesn’t have a threat model. Without a
well-defined threat model, it’s difficult to rigorously study the
success of the feature. Nevertheless, if I had infinite time,
resources, and power to indulge my blue checkmark fascination, there
are a few experiments I would run.

## My blue checkmark research agenda

If I were an academic researcher working in this area, I would work on
understanding the identity verification process in a black-box
way. The challenge is that this must be done in an ethical way, which
probably means that you can’t, for example, attempt to actually get
fake accounts verified. An empirical study (with proper disclosure
practices) is likely to be both ethical and fruitful, e.g. gathering
datasets of verified accounts on different social media platforms and
identifying name collisions, categorizing their trustworthiness, or
looking for patterns of misbehavior (such as spreading
misinformation). This might give some insight into how robust account
verification processes are.

The more interesting hypothetical research agenda is what I would do
if I were a user researcher working at Twitter, with access to
internal data and the ability to run in-product experiments. From this
vantage point, I would study how users comprehend the blue checkmark.

I might start with a lab study where people view disinformation with
or without a blue check, and survey them to understand if they are
more skeptical of the information when the blue checkmark is
absent. My guess is that Twitter has already done this study
internally, and I would be very interested to see the results!

With access to internal data, I think it would be possible to
understand whether the blue checkmark program helps with
impersonation, harassment, and abuse. For example, one could look at
whether people become less likely to report impersonation or
harassment after they get verified. However, these results would need
to be interpreted with a grain of salt, because the blue checkmark
itself could make people more likely to become targets of this kind of
abuse, among other possible conflating factors.

I’m not sure what the equivalent of Chrome’s EV removal field study
would be for the blue checkmark. Twitter could experimentally remove
the blue checkmark for a subset of verified accounts, but it’s not
clear what to measure then -- partially because the threat model isn’t
clear, and partially because the intended effect of the blue checkmark
may not be directly measurable. I suppose they could measure whether
impersonation/harassment reports go up for that subset of accounts
over time. Or they could compile a dataset of known misinformation
websites (as in [this
paper](https://homes.cs.washington.edu/~ericzeng/papers/Zeng-ConPro2020-BadNews.pdf))
and identify verified accounts that spread that misinformation. They
could then experimentally un-verify those accounts and determine if
engagement with misinformation on those accounts drops.

Another flavor of experiment would be to remove the blue checkmark
from all verified accounts for a subset of users, and measure how a
variety of user metrics might change. But this is not a particularly
scientific approach without a clear hypothesis about how we expect
user behavior to change with the ablation of the blue checkmark
feature. This lack of hypothesis differs from our EV study where we
could clearly articulate and measure what we expected to
change. (Namely, we expected that metrics such as form submissions and
time spent on page would decrease on EV websites, since we
hypothesized that users would think they were phishing pages due to
the lack of EV indicator.) Perhaps in the blue checkmark experiment,
we might hypothesize that users are more likely to share information
from verified accounts than unverified. Thus we would expect users in
the experiment group to retweet and share less from verified accounts
and more from unverified accounts, compared to users in the control
group.

I’ll close with the slightly cynical observation that I may be
entirely overthinking the blue checkmark; it could be that blue
checkmarks aren’t really about security, anti-abuse, or
mis-/disinformation at all, but rather that they’re just a way to
drive more engagement from particular users. Nevertheless, the
identity verification problem on social media platforms is real, so I
find it interesting to think about how that problem could be defined
and solved, regardless of whether social media platforms are tackling
it in earnest today.