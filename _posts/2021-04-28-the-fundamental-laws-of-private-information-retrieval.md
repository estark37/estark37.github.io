---
layout: post
title:  'The fundamental laws of private information retrieval'
---

I was recently writing an entirely different blog post and went down a rabbit
hole about private information retrieval (PIR), discovering all the bad
intuitions that I have about this problem. So now you get a blog post about my
misconceptions about PIR, rather than the blog post I was originally intending
to write (though maybe that one will eventually get written too). I’ll first
give a brief introduction to PIR, and then walk through some of the fundamental
laws of nature regarding this problem that were completely mismatched with my
intuitions.

## Private information retrieval

Suppose there is a server with a database of _n_ bits, indexed from 0 to
_n_-1. The client has an index of interest and wants to retrieve the value of
the bit in the database at that index. The catch is that the client wants to do
this retrieval without the server learning which bit the client is interested
in. In fact, the client doesn’t want the server to learn anything at all about
its query. For example, if the server can even deduce that there is some index
that the client is _not_ querying, that knowledge would violate this privacy
requirement.

A simple way to accomplish this privacy goal is for the server to send the whole
database to the client, and the client can simply read the bit at its index of
interest. Of course this scheme is quite inefficient, so PIR researchers try to
develop more efficient schemes that maintain the same privacy for the client’s
query. (You might notice that this simple scheme is not just inefficient but
also possibly problematic in another sense of privacy: the server might not want
to leak its entire database to the client. PIR doesn’t generally address privacy
for the server, but another type of protocol called [oblivious
transfer](https://en.wikipedia.org/wiki/Oblivious_transfer) does.)

There are many different types of efficiency that might matter for any
particular PIR application, but in this post I’ll be mostly talking about
communication costs and server computation costs. In the simple scheme I just
described, both communication and server computation are linear in the size of
the database. In general, the communication cost is a lower bound on the server
computation costs, since the server has to at least read or write every bit that
is sent.

There are various different flavors of PIR. Most relevantly to this post, a PIR
scheme can be:

  * **Information-theoretically secure vs. computationally secure**. As in other
    types of cryptography, information theoretically secure means that what the
    attacker sees is uniformly randomly distributed across the set of possible
    messages: there is literally zero information about what element is being
    queried in the messages that the attacker sees. In other words, the client’s
    queries are distributed identically regardless of which database element the
    client is querying for. Computationally secure, on the other hand, means
    that the distribution that the attacker sees may not be actually uniformly
    random, but it’s computationally indistinguishable from random. In the case
    of computationally secure PIR, no efficient (polynomially-bounded) attacker
    can distinguish a query for one element from a query for another element.
  * **Single-server vs. multi-server**. The single-server setting is the
    straightforward setting I described initially: a server holds a database of
    bits, and a client exchanges messages with the server to retrieve the bit at
    an index of interest. In the multi-server setting, there are multiple
    servers that each hold a copy of the database, and the client exchanges
    messages with these multiple servers to query the index of
    interest. Importantly, the servers are assumed to be non-colluding; that is,
    the security of the scheme only holds if the servers don’t exchange
    information with each other.
  * **Entirely online vs. offline/online**. So far I’ve described entirely
    online PIR, meaning that the entire protocol happens at query time. There
    are also schemes that introduce a preprocessing phase (where the server does
    some preprocessing work ahead of time) and/or an offline phase (an
    interactive protocol between the client and server that happens ahead of
    time, before the client knows which index it will be querying later on in
    the online phase).

What started me down this PIR rabbit hole was a
[paper](https://eprint.iacr.org/2021/345.pdf) that introduces an
information-theoretically secure, two-server, online/offline protocol. While the
paper is very well-written, I don’t have a lot of background in the area, and I
found myself mystified by how each of these design choices contributed to the
final result. Why two servers and not three or one? What, concretely, does the
offline phase actually buy you? I then went and read a bunch of other papers and
gradually pieced together the answers to these questions, many of which involve
several fundamental laws of PIR, and I think I better understand now what
various design choices can or cannot buy you in a PIR scheme.

### Law of PIR #1: Information theoretically secure PIR cannot be more efficient than sending the whole database.

The first law of PIR is that if you want information theoretical security and
you want efficiency, then you must have more than one server. And there seem to
be general ways to turn information theoretically secure multi-server schemes
into computationally secure single-server schemes that have similar costs,
though I didn’t dive deeply into that particular corner of the rabbit hole. So
if you’re designing a PIR scheme, you might as well focus on an information
theoretically secure multi-server scheme to get the best bang for your buck.

Here I suffered a lack of intuition; it wasn’t immediately clear to me why
multi-server schemes can achieve efficiency that single-server schemes
inherently cannot, so I went and read the
[proof](https://madhu.seas.harvard.edu/papers/1995/pir-journ.pdf) of this law of
nature. (As an aside, one of the fun things about PIR, especially information
theoretic, is that it doesn’t involve much fancy math at all; the proof of this
law of nature is one paragraph long, and the fanciest math I’ll be talking about
in this post is an exclusive-or.)

Here’s the intuition behind the proof. In a single-server scheme, suppose that
_T_ is a transcript from a run of the protocol in which a client queries a
server for element _i_ from database _D1_. Consider some other database _D2_,
where element _j_ has a different value in _D1_ and _D2_. By the information
theoretic security requirement, _T_ must also be a valid protocol transcript for
querying element _j_ from database _D1_ -- this is exactly what information
theoretic security means, that the distribution of possible transcripts for
querying one element has to be identical to the distribution of possible
transcripts for any other element. But this means that _T_ can’t be a valid
protocol transcript for querying element _j_ from _D2_, because item _j_ has
different values in _D1_ and _D2_, but _T_ can’t lead to the client getting
different results. This means, again by the information theoretic security
requirement, that _T_ can’t be a valid transcript for querying element _i_ from
_D2_. You can repeat this argument for every one of the _2^n_ possible
databases: that is, every possible database must have at least one unique
transcript for querying element _i_, therefore there must be at least _2^n_
possible protocol transcripts, requiring _n_ bits to communicate.

Introducing a second server effectively relaxes the privacy requirement, which
allows more efficient communication. In a multi-server protocol, it’s not
required that any transcript _T_ for fetching some index also be a valid
transcript for fetching every other index of the same database. It’s only
required that the part of the transcript that _each server_ sees is valid for
fetching every other index. It’s therefore no longer necessary that every
possible database has at least one of its own unique transcripts for fetching
_i_. Different databases can “share” the same transcript for fetching _i_
without violating privacy because it’s not a requirement that the same full
transcript also be valid for an element where the databases differ. Since
different databases can share transcripts in this sense, the set of possible
transcripts need not be as large as in the single-server setting.

### Law of PIR #2: Adding servers cannot reduce server computation costs, though they may be able to reduce communication costs.

This law breaks down into two parts: why more servers can't reduce computation,
and why more servers can reduce communication.

#### Why can't more servers reduce computation?

My (completely wrong) initial intuition was that schemes with multiple servers
would be able to reduce server computation costs. It seemed to me that one
server would have to touch fewer bits if there was another server that was
touching other bits that the first server couldn’t see. But it turns out that
server computation cannot be sublinear even in a multi-server scheme.

I didn’t go read the formal proof of this law, but it does start to feel
intuitive after a while. If the client is querying for element _i_, at least one
of the servers has to read element _i_ from the database (otherwise, there’s no
way that the value of element _i_ could be communicated to the client). Suppose
there are _k_ servers, and each server reads at most _l_ elements, and each
server guesses that _i_ is a randomly selected one of the elements that it
reads. Then each server’s probability of a successful guess is at least
(1/_k_)·(1/_l_) -- the probability that this is the server that reads element
_i_ times the probability that its random guess is correct. For
information-theoretic security to hold, the server’s probability of a successful
guess must be no more than _1/n_ which implies that _l_ has to be at least
_n/k_. In other words, each server must read a number of elements that is linear
in _n_. Sadness.

#### Why can more servers reduce communication?

We’ve established that adding more servers to a PIR scheme can’t reduce server
computation, but multi-server schemes aren’t subject to the same lower bounds on
communication. I talked a little bit above, in Law of PIR #1, about why having
multiple servers can theoretically reduce communication: with multiple servers,
different possible databases can “share” the same protocol transcripts for
querying the same element, so the scheme doesn’t need as many possible protocol
transcripts and thus fewer bits are needed to express those
transcripts. However, this insight still left me with questions, namely: how
exactly do multi-server schemes achieve better communication? And, does adding
_more_ servers inherently _further_ reduce communication costs?

Let’s look at the basic form of multi-server schemes. Let’s just consider a two
server scheme. The basic (inefficient, non-optimized) skeleton of a two server
PIR scheme is:

  * The client chooses a random set of elements, where each element is in the
    set with probability ½ (a set of size _n_/2 on average).
  * The client chooses one of the two servers and sends this set of
    elements. The server sends back the exclusive-or of all the values of the
    elements indexed by the set.
  * The client removes _i_ from the set if it’s present, or adds it if it’s
    not. The client sends the resulting set to the second server. That server
    sends back the exclusive-or of all the values of elements indexed by that
    set.
  * The client exclusive-ors the two responses together, and ends up with the
    value of the *i*th element. Voila!

In this scheme, each of the server sees a completely random set of elements (the
“exclusive-or” of an element with a random set produces a random set, just as
flipping a bit in a random bit-string produces a random bit-string). Thus the
servers learn zero information about the client’s query, and yet the client ends
up with the value of interest.

The recipe for a communication-efficient two server PIR scheme is now to take
this basic scheme and find a clever way to compress the sets in the client’s
queries.

For example, one modern approach is to express the sets with a new primitive
called [puncturable pseudorandom sets](https://eprint.iacr.org/2019/1075.pdf),
which allows a set of elements to be expressed in terms of a relatively short
key that can be efficiently transformed into a key for the same set but with an
element added or removed. (Puncturable pseudrorandom sets are based on an
existing primitive called puncturable PRFs, which I don’t know too much about
but are used for other interesting things like anonymous credentials. Maybe a
topic to research for another post.)

The earliest (as far as I can tell) multi-server
[approach](https://www.tau.ac.il/~bchor/PIR.pdf) doesn’t use any fancy
primitives to compress the sets in the client’s queries. Instead, it re-imagines
the database from a bit-string to a matrix of bits. Instead of selecting each
element to be in the random set with probability ½, the client selects a set of
rows and columns, where each row and column is included with probability ½. The
server exclusive-ors all the elements in the cross-product of those rows and
columns. The query is now of size _sqrt(n)_ instead of _n_/2.

But, there’s a catch with this multidimensional scheme: you need more servers to
get the correct result. This is because, by compressing the query in this way,
the client has lost precision. With elements expressed as a cross-product of
rows and columns, the client can’t simply add or remove the single element of
interest to the query for the other server. If the client were to add the row
and column corresponding to the element of interest, the server would also add
in a bunch of other elements from adding that row and column into the
cross-product of elements that it reads. We therefore need four servers to
cancel out these extra elements: one server gets the original randomly chosen
set of rows and columns in its query, one server gets the element of interest’s
row added to its set, another server gets the element of interest’s column added
to its set, and another server gets the element of interest’s row and column
added. (I’ve said “added” for simplicity here, but it’s really “added or
removed” depending on whether the row and column of the element of interest were
already present in the original randomly selected set.) The responses to all
these queries cancel out and leave only the element of interest.

This scheme is especially interesting because it can be generalized to more
servers. Instead of a matrix where queries are compressed to _sqrt(n)_ size by
expressing them as a cross-product of rows and columns, you can imagine a 3D,
4D, etc. cube that further compresses the queries. As the query gets more and
more compressed, it loses more and more precision: adding or removing an element
of interest adds more and more elements that aren’t of interest, requiring more
and more servers to cancel out those uninteresting elements.

Thus, in this multidimensional scheme, the more servers you have, the lower the
communication costs. But it doesn’t seem to be a general fact that adding more
servers reduces communication -- the more recent scheme that I described above
based on puncturable pseudorandom sets doesn't generalize beyond two servers. In
summary: you need at least two servers to get efficient communication, but
whether or not it makes communication more efficient to add even more servers
depends on the particular scheme.

### Law of PIR #3: If you want to reduce server computation, you need a preprocessing or offline phase.

Having probed a bit at why and how adding servers can reduce communication, but
not server computation, the final question is how to reduce server
computation. Preprocessing is the answer, but I didn’t immediately form an
intuition for why preprocessing or an offline phase would help with reducing
server computation. The whole intuition behind the lower bound on server
computation is that, at query time, the server has to touch a lot of the
database to avoid learning anything interesting from its own access pattern, so
it seemed to me that reducing the access pattern footprint at query time through
any means would fundamentally harm privacy guarantees.

In contrast to my initial intuition, here’s a simple way to see how
preprocessing can reduce query-time server computation without harming
privacy. Imagine that, in the aforementioned basic two server scheme, the
servers precompute the answers to all possible queries they could get, and just
look up the answers at query time. This is obviously a prohibitively inefficient
preprocessing phase, but it illustrates how work can be moved from online query
time to an offline phase without losing privacy.

What would a more interesting (i.e., efficient) preprocessing step look like?
Let’s start by seeing what would happen if, in the basic two server scheme, we
just started shrinking the number of elements in the queries (reducing both
communication and server computation costs). Instead of selecting each element
with probability ½, suppose we selected each element with probability ⅓ or ¼,
ending up with a set of size _n/3_ or _n/4_ on average -- or we could even make
those numbers smaller. As we shrink the size of the set, the query that contains
_i_ starts to get pretty shady. If each server guesses that the element of
interest is a random element in its set, the server that received _i_ in its
query is going to guess correctly with increasing probability as we shrink the
number of elements in the query.

The first [crucial observation](https://eprint.iacr.org/2019/1075.pdf) is that
we can avoid that loss in privacy by moving the query that contains _i_ into an
offline phase, so that the client doesn't need to send a privacy-harming query
that contains _i_. One of the servers computes a bunch of sets, computes the
answer for each of these sets by exclusive-oring the bits indexed by each set,
and sends these answers to the client ahead of time (compressing the sets using
one of the methods discussed above, such as puncturable pseudorandom sets). This
all happens in an offline phase, before the client begins querying. For example,
one way to construct these sets is to randomly divide the database into
_sqrt(n)_ sets of size _sqrt(n)_. Now the client can compute the answer to the
query containing _i_ by itself, without needing to send this problematic,
privacy-leaking query to the server: the client simply looks for a set
containing _i_ with the corresponding answer that it already got from the server
in the offline phase.

But we’re not quite done: there’s a second crucial observation to make. To
complement the answer that the client gets from the offline phase, the client
sends the set not containing _i_ to the other server at query time (I'll call
this the non-*i* query). This query also violates privacy, but not nearly as
much as sending a query that contains _i_. As we shrink the number of elements
in the query, the server learns less and less from the query that doesn’t
contain _i_: all the server knows is that the client is _not_ interested in the
elements in the query, and that knowledge gets less interesting as the number of
elements in the query shrinks. That is, the client telling the server a set of
_n_/3 elements that they’re not interested in is a lot more informational than a
smaller set of _n_/8 or _n_/32 elements that they’re not interested in.

With a set of size _sqrt(n)_, it turns out, this non-*i* query contains little
enough information that some small tricks can push it out of the realm of
usefulness for the adversarial server. One such trick is to, with small
probability, just do the wrong query, i.e., the client leaves _i_ in and removes
another element at random. This introduces some small correctness error: with
this small probability, the client won’t be able to compute the actual answer it
is interested in. This can be somewhat clunkily accounted for by repeating the
whole scheme in parallel a bunch of times to drive down the correctness error to
an acceptable amount. (Interestingly, we get information-theoretic security but
not perfect correctness!) Another more recent
[trick](https://eprint.iacr.org/2021/345.pdf) is to, at query time, decide with
small probability to send a fresh _sqrt(n)_-sized query containing _i_ to one
server and the same query not containing _i_ to the other server. While I didn’t
work through the actual math, I think the idea is that this case happens rarely
enough that the servers don’t get any advantage in guessing _i_, but often
enough that it erases the servers’ certainty about what isn’t _i_.

In summary, the recipe for these schemes is to precompute answers to the most
privacy-leaky parts of the online protocol, leaving the online phase with a
small enough amount of leakage that it can be wiped out with some small amount
of additional computation.

## Summary of the laws of PIR

After all that, I’m left with the following understanding: a two server
online/offline scheme is the way it is because (a) multiple servers are
necessary to achieve information-theoretic security efficiently, (b)
single-server computational security can be constructed in a black-box way from
a two server information-theoretic scheme, (c) multiple servers are needed to
reduce communication, but adding more than two servers doesn’t necessarily
reduce communication further, and (d) an offline phase is needed to reduce
server computation. Wphew! Now that I’ve gotten nerd-sniped by all that, maybe
someday I will write the blog post I intended to write, of which PIR was just a
small part.

<small><i>Thanks to Lea Kissner and Joe Hall for giving me feedback on this post!</i></small>