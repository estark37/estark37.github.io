---
layout: post
title:  "Complaints about program committees"
---

After several years of serving on program committees at computer
security conferences, I recently decided to take a hiatus. The time
commitment became overwhelming, but overall I consider serving on PCs
a worthwhile experience and hope to eventually get back in the game
after taking a break. If you’re considering donating time to review
papers, I’ll mention a few things that I find worthwhile about the
experience:

* It’s a fast way to learn what makes a paper likely to get accepted –
  much faster in my experience than writing papers and submitting them
  for review.
* It’s a forcing function to stay somewhat up-to-date with research in
  your area, which can be easy to treat as low priority if you work in
  industry.
* It’s an opportunity to become familiar with research outside your
  immediate area; I often aimed to review some papers that were just
  outside my comfort zone so that I would learn something new.

Despite these upsides, it’s easy to find many people complaining about
serving on PCs and about other aspects of the computer science peer
review process. I have my gripes too, but I think mine are mostly
fixable.

## The typical program committee experience

Among the conferences on which I’ve served as a PC member, the review
process is pretty similar. It goes like this:

* Upon joining the PC, you select from among a set of topics of
  interest.
* After a submission deadline (which used to be annually but is now
  quarterly, for most top-tier security conferences), PC members bid
  on papers they want to review. You look through submission titles
  and abstracts, pre-sorted by how well they match to the topics of
  interest you selected, and you set a score for each one. Higher
  scores mean you want to review the paper.
    * It used to be easy to overthink the bidding scores, with some
      conferences allowing you to score between -100 and 100, but
      conferences seem to be moving towards a simpler model where you
      have just a few options (e.g., 3 = I would love to review this
      paper, 2 = I want to review this paper, 1 = I wouldn’t mind, 0 =
      no opinion, -1 = definitely not). Bidding can actually take up a
      lot of time if you let it – some top-tier conferences get
      upwards of 500 papers per submission deadline – and I find the
      simplified scoring system to be a big quality-of-life
      improvement.
* A few days later, the PC chairs send review assignments. In my
  experience it’s typical to have 3-5 papers to review per submission
  cycle, with maybe 1-2 months allotted for reviewing.
* Before you submit your review for a paper, you can see who else is
  assigned as a reviewer but can’t read any other reviews.
* Most conferences seem to have at least 2 rounds of reviews. In the
  first round, there are typically just 2 reviewers, and if they are
  both negative, the paper gets rejected at this point. Papers may
  also get “desk-rejected” if they, for example, violate formatting
  guidelines or aren’t properly anonymized. Then, for papers that make
  it to the second round, 2-3 more reviewers are assigned.
* After the second round reviewing deadline, the discussion period
  opens. At this point, reviewers discuss amongst themselves
  asynchronously. Some conferences allow for an author rebuttal, or
  even asynchronous interactivity between the anonymized authors and
  reviewers (e.g., authors can provide a rebuttal, reviewers can
  respond to the rebuttal with a question that the authors can
  answer).
* During the discussion period, reviewers may update their reviews and
  even change their accept/reject decision. In fact, PC chairs will
  often encourage reviewers to do so, so that the final reviews and
  decisions released to authors are consistent.
* Pre-Covid, everyone on the PC would fly to some location for an
  in-person PC meeting to discuss all the papers before final
  decisions were released to authors. I’ve only been to one of
  these. People say that in-person PC meetings were important for
  networking and that especially early career academics are really
  missing out on one of the main benefits of serving on PCs. However,
  in the cold light of 2023, flying 50+ people around the world, for a
  one day meeting to discuss dozens of papers from which most people
  have only read a small handful, just doesn’t have the right
  vibes. Instead, these days PC chairs strongly encourage reviewers to
  come to a decision on each paper via asynchronous online
  discussion. Then a virtual PC meeting is scheduled to discuss only
  the papers for which a decision couldn’t be reached.
* Finally, decisions are released to authors, sometimes with
  shepherding or revision requirements to cement an “Accept” decision.

## My gripes

If I were running a PC, here’s what I would do differently.

### Make reviewer discussions less repetitive

The most eyeroll-inducing aspect of the review process is the amount
of repetition. Typically, once all reviews for a paper have been
submitted, a PC chair will appear in the review portal to say
something like, “Ok, all reviews are in! Please discuss amongst
yourselves and come to a decision!” Following that, most reviewers
summarize their reviews to each other and repeat their
decision. Sometimes there is some substantive debate, often there is
not. Often, there are several rounds of people simply restating their
opinions in response to another reviewer’s disagreement. As the
decision deadline approaches, a PC chair will again appear and say
something like, “We need to come to a decision soon!” This is followed
by reviewers again summarizing their review and restating their
position. If there is disagreement, and no one feels strongly positive
about the paper, gradually everyone who gave an Accept decision says
something like, “I’m okay with letting this one go” and then a Reject
decision is reached.

Overall, the process feels repetitive and inefficient, and somewhat
like a choreographed dance when we could instead jump to the end state
that everyone knows is coming.

Some conferences appoint associate chairs and/or discussion leads for
each paper. These people have the job of facilitating decision-making,
which often takes the form of giving the “Please discuss amongst
yourselves!” and “We need to reach a decision soon!” prompts.

If I were in charge, I would replace discussion leads with decision
leads. The job of the decision lead would be to read the reviews, ask
clarifying questions or probe at reviewers’ positions, and make a
decision (perhaps with some appeal process for reviewers to escalate
to the PC chairs). I think a lot of the inefficiency of the current
process comes from the fact that it’s rare to reach consensus on the
subjective matter of a paper’s merits, and yet no one is empowered to
make a decision in lieu of consensus.

Having decision leads would be less democratic but likely vastly more
efficient. This especially makes sense if you view peer review as a
system for weeding out obviously bad or wrong ideas, rather than as a
system for rigorously ensuring the merits of published papers, but
that’s a philosophical debate for another time.

### Abolish PC meetings

As I hinted above, I don’t think we should go back to in-person PC
meetings. While I recognize some upsides of in-person PC meetings,
they’re just too expensive and inefficient. Nixing travel requirements
makes PC membership more inclusive; enables more frequent submission
deadlines and review cycles; enables a larger reviewer pool, which
makes the job more sustainable; and is vastly more ethical from a
climate perspective.

That said, virtual PC meetings are awful too. I think the transition
to virtual meetings may have been made in haste due to Covid and
wasn’t entirely thought through.

* Reviewer pools are global and the virtual PC meeting is bound to be
  at a ghastly time for some people.
* Virtual PC meetings are scheduled weeks or months in advance, but
  the exact timings and agenda are unknown until shortly before the
  meeting, because attendance is only required for reviewers of papers
  who haven’t reached a decision via online discussion. In practice,
  even though I know the date of the virtual PC meeting ahead of time,
  I don’t reserve the time, because I don’t know exactly which time to
  block off and it seems silly to block off a whole day for a meeting
  that I likely won’t need to attend. So whenever I do need to attend
  a virtual PC meeting, it’s a scramble to make it work. Presumably
  because of similar scheduling problems, oftentimes “mandatory”
  attendees don’t show up.
* Virtual PC meetings often just rehash disagreements that came up in
  the discussion period and sometimes end with a PC chair making a
  fairly arbitrary decision so that everyone can move on.

If I were in charge, I would simply not have PC meetings. Instead,
when reviewers can’t reach a decision, they should just schedule a
call amongst themselves as needed.

### Define evaluation criteria

My biggest gripe is probably the hardest to fix: most conferences have
virtually no criteria for what to accept or reject. Reviewers will
often say things like “This doesn’t meet the bar for $CONFERENCE”, or
“This might be better-suited for a workshop”, or “This contribution
isn’t large enough” – with no further elaboration on what those
statements mean. I’m certainly guilty of statements like these myself,
and I don’t blame anyone else who is guilty too; after all, we aren’t
given any other tools to judge papers’ merits! Besides tribal
knowledge, there’s no definition of what would meet the bar for any
given conference, what differentiates a top-tier conference paper from
a lower-tier conference paper from a workshop paper, or what
constitutes a large enough contribution (or what defines a
“contribution” in the first place).

In contrast, as a manager at a large tech company, I would never dream
of giving a person performance feedback such as “Your work doesn’t
meet the bar” without further elaboration, or at least pointing to the
global job ladder that defines performance at each level. Coming from
industry, where job performance is evaluated on carefully defined and
closely analyzed rubrics, evaluating papers for conferences feels like
the wild west.

With this level of subjectivity, the end result is a lot of bias,
inconsistency, and unpredictability in peer review
decision-making. Though it would be a large undertaking, conferences
need rubrics: well-defined axes on which reviewers should evaluate
papers, with definitions and examples of subpar, acceptable, and
exemplary merit on each axis.

Lest it seem overwhelming to define rubrics upfront, I think paper
rubrics can be introduced incrementally and improved upon over
time. They’ll eventually converge to a state that facilitates more
equitable and consistent decisions.

I don’t aspire to ever serve as a PC chair, which seems like an
extremely time-consuming and thankless job – but I will gladly serve
on any PC whose chair takes my suggestions and puts them into action!
(After my hiatus, that is.)

