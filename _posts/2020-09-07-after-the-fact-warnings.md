---
layout: post
title:  'After-the-fact warnings'
---

In my line of work, we talk a lot about security warning design and
effectiveness. Usually, the task of a warning is to prevent a user from doing
something dangerous, such as accepting an invalid certificate or installing
malware. There’s another class of warnings which tell users that something bad
has _already_ happened, with the aim of helping them recover. I’ll call this
category **after-the-fact warnings**.

I don’t know much about after-the-fact warnings, such as best practices for how
to design them or how effective they can be. In fact I know so little that I had
to make up a name for this category of warnings, when there probably already
exists a fancy academic name. So I
[asked](https://twitter.com/estark37/status/1301545990848094208) for some
research pointers on Twitter and collected an interesting reading list. I’ll
summarize some of my observations here. Note that this isn’t a comprehensive
survey -- just some thoughts from the papers that caught my eye the most.

# Real world examples

To set the stage, here are some real world examples of after-the-fact warnings.
Again, this isn’t comprehensive, just some examples that I can think of:

* Antivirus notifications telling people that they have malware on their device
* Password breach notifications, either from a first-party (LinkedIn notifying
  users that their LinkedIn password has been leaked) or third-party service
  provider ([HaveIBeenPwned](https://haveibeenpwned.com/),
  [a web browser](https://security.googleblog.com/2019/12/better-password-protections-in-chrome.html))
* Chrome has a feature
  (“[predictive phishing protection](https://www.blog.google/technology/safety-security/new-security-protections-tailored-you/)”)
  that notices when you type a saved password on a suspected phishing site and
  warns you that you may have been phished and should change your password. This
  is similar to a password breach notification, but has some different
  considerations because the suspected attack is phishing rather than credential
  stuffing.
* Other (non-password) data breach notifications, for example an email from
  Equifax telling people that their personal data has been leaked
* Some browsers have persistent UI telling the user that they’ve bypassed a
  security warning in the past. For example, after a person clicks through a
  certificate, phishing, or malware warning in Chrome, the address bar shows a
  red “Not Secure” chip to let them know that they’re in an unsafe state. It
  might be a bit of a stretch to call this an after-the-fact warning, as it’s
  not clear what action the warning is asking the user to take.

{% include image.html url="/assets/after_cert_error.png" description='Google
Chrome UI after bypassing a certificate error. The page loads normally but the
address bar shows a red “Not Secure” chip to emphasize that the user is in an
unsafe state.' 
short_description='Chrome UI after bypassing a certificate error' %}

Finally, there are some interesting hypothetical examples of after-the-fact
warnings, which is what originally got me interested in this area.
[Certificate Transparency](https://emilymstark.com/2020/07/20/certificate-transparency-a-birds-eye-view.html)
introduces an asynchronous certificate verification step. The browser might
accept a certificate as valid, but then find out that it was actually not valid,
days or weeks later. It’s generally been considered beyond the pale for the
browser to warn the user at this point. How might the browser communicate that a
website the user visited in the past was potentially compromised? What is the
user supposed to do about it? I think the answer to these questions is probably
that the browser shouldn’t try to communicate this message, because it’s too
hard to understand and there’s nothing the user can be expected to do about it.
But I’m not totally sure that’s the right answer, and that’s what got me
interested in how successful after-the-fact warnings can be.

# A sample of academic research

I read 5 papers on this topic:

* [“What was that site doing with my Facebook password?” Designing Password-Reuse Notifications](https://www.mobsec.ruhr-uni-bochum.de/media/mobsec/veroeffentlichungen/2018/09/10/ccsf266-finalv1.pdf) _Golla et al._
    * This paper describes two surveys about notifications that companies send
      when they discover a user’s password in a password dump from another
      service. The first survey studies representative examples of real-world
      notifications, and the second survey uses a model notification designed
      using the lessons from the first survey. Comprehension was pretty poor
      across the board. Most users reported that they would change their
      password upon receiving the model notification, but very few said they
      would change their password to something unrelated. (And of course,
      surveys don’t always give us the best information about what someone would
      do in real life.)
* [(How) Do People Change Their Passwords After a Breach?](https://www.ieee-security.org/TC/SPW2020/ConPro/papers/bhagavatula-conpro20.pdf) _Bhagavatula et al._
    * This paper makes use of the Security Behavior Observatory, which is a
      large group of participants who have agreed to run research software on
      their machines to collect information about their security behaviors. This
      allowed the researchers to observe how many users changed their password
      when their accounts were actually breached in real life. The answer is
      that not so many participants did change their passwords (33%), and of
      those, many participants didn’t do so for quite a long time after the
      breach.
* [“Should I Worry?” A Cross-Cultural Examination of Account Security Incident Response](https://ieeexplore.ieee.org/stamp/stamp.jsp?tp=&arnumber=8835359) _Redmiles_
    * This paper describes a set of interviews with Facebook users from 5
      countries. The interviewers asked participants about how they responded to
      notifications from Facebook that their account had a suspicious login.
      These notifications tell the user that their account may have been
      compromised and then ask them to complete a secondary authentication task,
      such as identifying photos of their friends. One finding that particularly
      caught my eye was that, when asked what alerted them to the suspicious
      login, participants sometimes identified the secondary authentication task
      rather than the initial notification. The paper hypothesizes that this may
      be due to warning fatigue blinding users to the initial notification.
      (Warning fatigue is when users see many warnings and get habituated to
      them, ceasing to take action on them.)
* [I’m too Busy to Reset my LinkedIn Password: On the Effectiveness of Password Reset Emails](http://lersse-dl.ece.ubc.ca/record/316/files/CHI-17_huh_paper.pdf) _Huh et al._
    * This paper describes a Mechanical Turk survey of participants who had
      received password breach notifications from LinkedIn. For a methodological
      contrast, “‘What was that site doing with my Facebook password?’ Designing
      Password-Reuse Notifications” asked people what they would do
      hypothetically, and “(How) Do People Change Their Passwords After a
      Breach?” observed what people actually did (with a slight caveat that
      there were some heuristics used to figure out what people actually did
      from the available data). In this Huh et al. paper, the researchers asked
      people to recall what they did. Each of these methods has its pros and
      cons; asking people to recall what they did can suffer from [recall
      bias](https://en.wikipedia.org/wiki/Recall_bias), though in this case the
      authors took some steps to combat this bias by asking participants to
      provide screenshots to verify that they had actually received the
      notification and changed their password (if applicable). The results
      showed a somewhat higher rate of password change (~50%) than other work,
      but identified some other themes in common with other work: a substantial
      delay before participants took action, and a tendency to choose weak or
      related passwords when they did eventually change their password.
* [You ‘Might’ Be Affected: An Empirical Analysis of Readability and Usability Issues in Data Breach Notifications](https://yixinzou.github.io/publications/chi2019-zou.pdf) _Zou et al._
    * This paper analyzes a dataset of real-world data breach notifications
      downloaded from government databases. Most were mailed notifications.
      Because I’m mostly interested in experimental results from electronic
      notifications, I only skimmed this paper. The paper compiles a list of
      best practices and recommendations for notification design (such as using
      clear and concise language and highlighting key information visually) as
      well as public policy (such as encouraging that notifications are
      delivered across multiple channels). These recommendations seem to broadly
      make sense, but I would be hesitant to apply them too dogmatically to
      computer security warnings without verifying them experimentally.

Finally, there’s a body of adjacent work about vulnerability and
misconfiguration notifications sent to server operators. For example, Durumeric
et al. conducted an experiment notifying 150,000 server operators that their
servers were vulnerable to the Heartbleed vulnerability. I’m more familiar with
this area of research because I’ve done work in it
[myself](https://storage.googleapis.com/pub-tools-public-publication-data/pdf/06a75f932595f27a60092007965934c957b5de21.pdf).
My overall impression is that the effectiveness of this type of notifications
varies quite widely depending on the severity and urgency of the issue and the
effort needed to remedy it. Regardless, I view this area as mostly tangential
because I’m particularly interested in after-the-fact warnings for end users,
which is a substantially different population than server operators.

# What's missing

Here’s a collection of questions that I was hoping to find in these papers but
didn’t:

* There’s a bit of an elephant in the room: **is it possible for users to
  actually recover once they’ve been compromised?** Most of the work I described
  above focuses on getting users to complete a particular action, such as
  changing their password to a new, strong, unique password. But what if that’s
  not enough? Data may have already been leaked or the account’s integrity
  violated. The Redmiles paper (“‘Should I Worry?’”) touches on this issue by
  describing follow-up actions that participants performed to verify their
  account integrity, such as checking whether any unauthorized messages had been
  sent from their account. But I think the research community hasn’t grappled
  with what’s involved in truly recovering from an account compromise, and I
  could spend a long time meditating on whether it makes sense to do
  after-the-fact warnings at all if we don’t know what to tell users to do to
  recover. (Also relevant: my colleague Chris Palmer pondering
  [recoverability](https://noncombatant.org/2019/08/24/recoverability/).)
* **How well do antivirus notifications work?** When AV software pops up a
  warning that the user has some malware installed, how likely are users to
  comprehend the risks and respond effectively? (And, reiterating the elephant
  in the room, what does “effectively” even mean in this case?) I couldn’t find
  any papers on antivirus, though it’s quite possible I missed them.
* I didn’t find any **_in situ_ experiments** that test how users respond to
  different after-the-fact warning variants in real-world settings.
* I didn’t find much work on **habituation for after-the-fact warnings**
  specifically (despite the large body about warning habituation generally and
  for particular types of warnings such as certificate warnings). The Redmiles
  paper hypothesized that one of their findings may have been due to warning
  fatigue, and it would be interesting to dig into that more deeply.
* Finally, I’d be interested to get to a definitive conclusion on **how much the
  source of the notification -- first-party vs. third-party -- matters**. If
  LinkedIn warns you that your LinkedIn password may have been breached, is that
  more effective than receiving the warning from a third party (such as
  [HaveIBeenPwned](https://haveibeenpwned.com/) or a
  [web browser](https://security.googleblog.com/2019/12/better-password-protections-in-chrome.html))?
  I found another paper
  ([“Data Breaches: User Comprehension, Expectations, and Concerns with Handling Exposed Data”](https://research.google/pubs/pub47000/)
  by Karunakaran et al.) that dives into this question a bit. The authors
  surveyed people on whether they are comfortable with various use cases for
  the data exposed in data breaches. For example, are people comfortable with a
  third-party service proactively resetting their password? I didn’t read this
  paper thoroughly yet, though.

For some reason, I had a hard time finding any research at all about
after-the-fact warnings until I asked on Twitter, and it turns out there’s a lot
more than I expected! This seems to be an active but still developing research
area.