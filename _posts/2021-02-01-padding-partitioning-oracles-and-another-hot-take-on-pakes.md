---
layout: post
title:  'Pa(dding|rtitioning) oracles, and another hot take on PAKEs'
---

I’ve been trying to read more applied crypto papers here and there, and this
past week, [Partitioning Oracle Attacks](https://eprint.iacr.org/2020/1491.pdf)
(Len, Grubbs, Ristenpart) came up on my reading list. This paper presents a new
class of attack against certain cryptographic protocols that are based on
low-entropy secrets (e.g., passwords). It’s a way to dramatically speed up
guessing attacks -- for example, in some cases the attack allows the attacker to
binary-search over a dictionary of passwords instead of trying them one-by-one.
This type of attack seems likely to become a classic because it applies to some
well-known encryption schemes and many different applications.

In this post, I’ll give a high-level overview of another classic oracle attack
(padding oracles), and compare and contrast to the new concept of partitioning
oracle attacks as I understand them; my goal here is to make these attacks
accessible to people with minimal cryptography background. Then I’ll give one of
my hot takes about PAKEs, which is probably mostly of interest to people who
have read the partitioning oracles paper or similar.

## Oracles by example

In cryptography, “oracle” means a black-box function or system that takes an
input and returns an output. In the context of an attack, an oracle is used to
formally model an implementation error that gives the attacker some extra
information. Oracles attacks are interesting and educational because they
illustrate why it’s so important that an encryption of a message reveals _no
(efficiently computable) information_ about the message itself: even seemingly
innocuous extraneous information can be used, surprisingly, to break the
encryption scheme entirely.

Typically, when cryptographers describe an attacker for a cryptosystem, they
model the attacker as having certain capabilities, such as the capability to
request encryptions for a reasonable number of messages of their choice. Then
the attacker has a challenge that they have to solve, such as distinguishing an
encryption of another message of their choice from an encryption of a random
message.

But in a real implementation of that cryptosystem, the attacker might learn a
lot more than is modeled in these formal enumerations of their capabilities. The
attacker might be able to observe the time that it takes to encrypt a message,
or might even learn something about the contents of a message from elsewhere in
the system. Oracles are a way to formally describe this type of leakage. For
example, suppose a server decrypts a message and then inserts it into a
database, and the database returns an error in a short-circuit if the message’s
first byte is `0x00`, and goes on to do some other processing otherwise. An
attacker can learn from observing the timing of this database operation whether
the message starts with `0x00`. We can model this as an oracle, by saying that
the attacker has the capability to query an oracle function whose output is 1 if
and only the message starts with `0x00`.

The classic example of an oracle attack is a padding oracle, where the attacker
learns a single extra bit about any arbitrary ciphertext: whether it is padded
correctly. “Padding” here refers to the fact that many encryption schemes don’t
work on messages of arbitrary lengths. For example, it’s common for encryption
algorithms to process messages in blocks of 16 bytes, so messages have to be
padded to a multiple of 16 bytes, with some well-defined and reversible padding
scheme. One example of a padding scheme is to fill in the padding bytes with the
length of the padding. In this scheme, a message of length 60 bytes will be
padded with 4 bytes set to the value `0x04`.

When implementing cryptographic algorithms, it’s easy to accidentally provide a
padding oracle to attackers. The oracle often comes from a timing side-channel,
because it’s tempting to short-circuit if padding is invalid and not go on to do
other processing that is done for correctly padded messages.

And it’s not too hard to implement a padding oracle attack for many encryption
schemes. I’ll illustrate this with a hypothetical example that demonstrates the
basic concept.

Suppose you have an encryption scheme where flipping a bit in a ciphertext flips
the corresponding bit in the underlying plaintext. So if you encrypt some
message _m_, flip the last bit in the ciphertext, and then decrypt it, you’ll
get a message which is the same as _m_ but with the last bit flipped. (It might
seem, intuitively, that a secure encryption scheme shouldn’t have this property
that you can transform a ciphertext to produce a predictable transformation in
the plaintext, but this property on its own isn’t actually enough to break the
encryption algorithm. The property is called _malleability_, and some common
encryption algorithms have it and others don’t.)

Suppose you’re using this encryption scheme with the padding scheme I described
above: messages are padded with n bytes, each set to the value n, to extend the
message length to a multiple of 16 bytes. We’ll say that there’s always at least
one byte of padding added, so a message that is originally 16 bytes would be
padded to 32 bytes.

Now consider an attacker who has some ciphertext which is an encryption of a
message _m_, and the attacker wants to learn what _m_ is. The attacker can
request decryptions for an arbitrary number of ciphertexts to learn if the
underlying plaintexts have valid padding -- this is the padding oracle. Here’s
the attacker’s algorithm:

* First, we need to figure out what the padding value is. The attacker will do
  this by searching for how to transform the original message into a message
  that has only one byte of padding. The attacker iteratively chooses a byte _b_
  from `0x01` to `0xff`, and at each iteration, the attacker xors the last byte
  of the ciphertext with _b_ and queries to see if the padding is valid. When
  the attacker finds a value for b that produces valid padding, then the
  attacker has learned that the last byte of the message xored with _b_ is
  (probably[^1]) equal to `0x01`, which allows them to recover the actual last
  byte of the message -- the length of the original message’s padding.
* Suppose the attacker has now discovered that there are 3 bytes of padding in
  the original message, so the last 3 bytes of the message are `0x03`. The
  attacker is now going to search for the value of the 4th-to-last byte of the
  message (which is the last byte of the original unpadded message). They’ll do
  this by transforming the last 3 bytes of the message from `0x03` to `0x04`, and
  then searching for how to transform the 4th-to-last byte to `0x04`:
  * To transform the last 3 bytes of the underlying message from `0x03` to
    `0x04`, the attacker xors the last 3 bytes of the ciphertext with `0x07`
    (because `0x03 ^ 0x07 = 0x04`).
  * Now the attacker again iterates from _b_ = `0x01` to `0xff` and xors the
    4th-to-last byte of the ciphertext with _b_. When the attacker finds a value
    of _b_ that produces valid padding, they know that the 4th-to-last byte’s
    value xored with _b_ is (again, probably) `0x04`, and from that they can
    recover that byte’s value.
* Now it’s a simple matter of iterating this procedure for each remaining byte,
  from last to first, to recover the entire message!

The high-level idea is that if an attacker can transform a ciphertext in a way
that produces a predictable change in the plaintext, they can use this to search
for transformations that produce valid padding. If they know that a transformed
byte of ciphertext produces a valid padding byte in the plaintext, then they
might be able to learn what the original byte of plaintext was by undoing that
transformation.

## Partitioning oracles

Now that I’ve talked so much about padding oracles, you might be a bit peeved to
hear that partitioning oracles are not really like padding oracles at all. I
believe that they both will be classic oracle attacks with significant practical
impact, but that’s mostly where the similarity ends. I mostly explained padding
oracles because they are a relatively straightforward example of what an oracle
is and how it can be used, and because they can help explain partitioning
oracles as a series of contrasts.

### Attack target: plaintext versus password

Whereas a padding oracle attack aims to recover a plaintext from a ciphertext, a
partitioning oracle attack aims to recover the secret key -- which is far more
devastating, because it allows the attacker to subsequently decrypt any message.
Padding oracles are relevant to any cryptosystem that uses a padding scheme, but
partitioning oracle attacks specifically target cryptosystems that derive keys
from low-entropy secrets like passwords.

Password-based cryptosystems are inherently weaker than systems where a key is
chosen at random. The space of passwords is usually smaller than the space of
possible keys, and passwords can be chosen weakly, often coming from a large
dictionary of common passwords. Password-based cryptosystems aim to at least
avoid offline brute force attackers, where an attacker can eavesdrop on some
communication and then try a bunch of different passwords offline. Instead, the
attacker should be forced to interact with the legitimate parties; in the real
world, this is a significant deterrent since the attacker will be subject to
denial-of-service and anti-fraud protections if they have to interact with a
real server, plus the time required to process each guess will be slowed down by
network latency and other factors.

So, the partitioning oracle setting is an online guessing attack. The goal is to
dramatically speed up the attack over trying candidate secrets one-by-one.

### Applicable encryption schemes: malleable encryption versus non-committing authenticated encryption

When I described padding oracle attacks, the encryption scheme had a special
property that allowed the attacker to work: malleability, or the ability to
transform a ciphertext in such a way that produces a predictable transformation
in the plaintext.

In contrast, partitioning oracles target _authenticated encryption_ schemes with
a _different_ specific property.

First, what’s authenticated encryption? In an authenticated encryption scheme,
by definition, once a message has been encrypted, any change to the ciphertext
will cause decryption to fail with an error. (Note that by definition, an
authenticated encryption scheme cannot be malleable.)

Second, what’s this special property we need? It’s called key multi-collisions,
and it’s much less well-known[^2] than malleability. If an authenticated
encryption scheme has key multi-collisions, it means that the attacker can
generate a ciphertext that decrypts without error under several different keys.
The paper calls such schemes _non-committing_.

At first glance, the presence of a key multi-collision might sound in conflict
with the definition of authenticated encryption. But note that the goal here is
not to _transform_ a ciphertext such that it decrypts under several different
keys; rather, it’s to generate a ciphertext from scratch. And it turns out that,
surprisingly, some rather well-known and commonly used authenticated encryption
schemes are non-committing (aka, not resistant to key multi-collisions). That
is, in these schemes, it’s possible (and efficient) to construct a ciphertext
that decrypts without error under many different keys. The attacker might not
know much about what it decrypts _to_, but they know it successfully passes
authentication checks and decrypts to something.

I don’t have a good intuition for what makes an encryption scheme
multi-collision resistant or not. In the partitioning oracles paper, the authors
describe how to construct key multi-collisions for some encryption schemes where
the encryption scheme is a series of algebraic operations, so constructing a key
multi-collision is a “simple” matter of solving a system of linear equations.
I’m not sure if all forms of key multi-collisions will take this same basic
shape or not. This is something I’d like to understand better in future.

### The oracle itself: padding validity versus authentication

As previously discussed, in a padding oracle, the attacker is able to somehow
learn whether a ciphertext decrypts to a message with valid padding. In a
partitioning oracle, the attacker is able to learn something totally different:
whether or not a ciphertext decrypts without error, or, in other words, passes
the encryption scheme's authentication check.

The paper calls this a partitioning oracle because, along with an algorithm for
generating key multi-collisions, the oracle allows the attacker to partition the
set of secrets into those that are possibly the true underlying secrets and
those that are not. In other words, a partitioning oracle and key
multi-collision algorithm allow the attacker to generate a ciphertext that
“guesses” many secrets at once and submit that to the oracle to learn if the
underlying secret is one of those guesses.

For example, if attacking a password-based encryption scheme, the attacker can
choose a set of common passwords from a password dictionary and generate a
ciphertext that decrypts under all the corresponding keys, and submit that
ciphertext to the oracle. If the ciphertext decrypts without error (according to
the oracle), then the attacker can recursively search that set of common
passwords; otherwise, they can recursively search the remainder of the
dictionary. The speed of the attack depends on how much control the attacker has
over the key multi-collision. In the best case, the attacker can generate a key
multi-collision for exactly half the remaining passwords at each step and
perform a binary search.

Where might a partitioning oracle arise in practice? The paper has an in-depth
case study of an encrypted UDP proxy protocol where the client and server share
a password from which they derive an encryption key. The oracles arises from the
fact that, upon receiving an encrypted packet from the client, the proxy server
opens a port (to listen to a response from the target server) if and only if the
packet decrypts without error. Thus, an attacker can send an encrypted packet to
the proxy server and observe (via port scanning) whether a port was opened to
determine whether the ciphertext decrypted without error.

There are quite a few other details in there, both in the cryptography and the
implementation, but that’s the basic idea of the attack. Most interestingly, the
authors show how to construct key multi-collisions that decrypt to messages that
pass format checks. In the encrypted proxy example, the attack only works if the
ciphertext decrypts to a message that starts with 0x01. Constructing this kind
of collision requires some good luck, so it doesn’t scale to arbitrary format
checks. In fact, one of the authors’ suggestions for fixing key multi-collisions
is to change encryption schemes to prepend 16 zero bytes to messages and check
for their presence upon decryption. This is essentially a format check, but a
long enough one that it’s not practical to pass it by luck when constructing key
multi-collisions.

### Can you just tell me what I should look out for?

If you don’t care too much about the details, here’s what to watch out for:

* For padding oracles, if the encryption scheme is malleable and the
  implementation allows attackers to differentiate whether a ciphertext decrypts
  to a message with valid vs. invalid padding, then you are probably going to
  have a bad time.
* For partitioning oracles, if the encryption scheme is based on passwords or
  some other secret that isn’t a randomly generated key, and the encryption
  scheme allows attackers to generate ciphertexts that decrypt without error
  under multiple keys, and the implementation allows attackers to determine
  whether a ciphertext decrypts without error or not, then you are probably
  going to have a bad time.

Unfortunately, partitioning oracle attacks will probably be harder for
non-experts to spot than padding oracles, at least until the concept of key
multi-collision resistance becomes more mainstream. It’s very easy to discover
whether a particular encryption scheme is malleable by googling around for a few
minutes, but this isn’t really the case for key multi-collision resistance.

## A hot take about PAKEs

The partitioning oracles paper also presents another case study attack against a
state-of-the-art Password Authenticated Key Exchange (PAKE). The PAKE they
target allows a client to authenticate to a server with a password, without the
server knowing the actual password. Technically this is called an asymmetric
PAKE (aPAKE). aPAKEs are useful because they keep passwords secret against
phishing servers, and in the event that a legitimate server gets breached.

A weird thing about this aPAKE partitioning oracle attack is that it’s
“backwards” from how we typically think about cryptographic attacks. By
“backwards” I mean that the server is attacking the client (since the client is
the entity that knows the password), whereas a more common attack scenario is
for a client to be attacking a server.

At a high level, in an aPAKE partitioning oracle attack, we assume that an
impersonating server can somehow induce a client to make many login requests. As
part of the protocol for each login request, the fake server provides a key
multi-collision to guess from a set of passwords. The fake server observes the
client’s behavior (which is the partitioning oracle) to determine whether the
real password is in their set of guessed passwords.

My hot take is that these new aPAKE partitioning oracle attacks have little
practical impact. I don’t know of any aPAKE deployment where the client can be
tricked into making many login requests, or even any login requests, without
human user involvement. My extensive research (aka,
[polling](https://twitter.com/estark37/status/1354667657413369858) my Twitter
followers) didn’t turn up many plausible scenarios. The most plausible idea was
a password manager that stores the aPAKE password and automatically logs the
user in upon request. But, password managers usually wouldn’t initiate a login
request without some kind of human interaction, since people usually don’t want
to be identified to servers without their consent. Human interaction on each
guess is downright prohibitive for any guessing attack, even one that is
dramatically sped up over brute-force one-by-one guessing.

In the paper, the authors give the example of malicious JavaScript on an
attacker-controlled web server that initiates many aPAKE login requests. This
follows the example of
[BEAST](https://en.wikipedia.org/wiki/Security_of_Transport_Layer_Security#BEAST_attack)
and other attacks which initiate multiple requests from JavaScript to attack
session cookies or other secrets. I find this to be an unrealistic attack
setting for aPAKEs on several levels. First, web browsers don’t generally
implement aPAKES, and if they did and integrated them with browser-based
password managers, I imagine that each login request would require user
interaction, as I just explained. Further, I previously wrote up a different
[hot take](https://emilymstark.com/2020/07/30/should-web-apps-use-pakes.html)
about why I don’t think PAKEs are coming to the web or web browsers, and that
further cements my skepticism about the practicality of this attack.

Don’t get me wrong: partitioning oracle attacks in aPAKEs are weaknesses that should be fixed. But, I do think they will remain confined to the world of theoretical attacks. Generally I predict partitioning oracle attacks will have significant practical impact, perhaps as much as padding oracle attacks, but I think that impact will come from settings other than aPAKEs.

[^1]: There could be some unlucky cases where the attack strategy doesn’t
    actually work. For example, suppose the original unpadded plaintext is 15
    bytes long and ends in `0x02`. The padded plaintext’s last two bytes would
    be `0x02 0x01` (the original unpadded message’s last byte, followed by one
    byte of padding). In this case, a choice of _b_ = `0x03` would produce valid
    padding, because the transformed plaintext would end in `0x02 0x01 ^ 0x03` =
    `0x02 0x02` -- a valid padding of two bytes. So the attacker has unwittingly
    transformed the plaintext so that it ends up with two bytes of padding
    instead of one, which is what the attacker was searching for. We could
    construct other cases like this where the attacker accidentally finds valid
    padding that isn’t what they were looking for, due to a fluke in the
    original message. It’s okay to ignore these cases though, because even
    breaking some or most encryptions is enough to declare the encryption scheme
    broken. These weird cases also might be less likely if the application had
    certain format requirements on the plaintext whose encryption the attacker
    is trying to break -- for example, the application might only encrypt
    ASCII-encoded English strings.

[^2]: To put this into context, if you take an undergraduate cryptographic
    course, you will probably learn about malleability early in the semester. I’ve
    taken multiple graduate-level cryptography courses, albeit some years ago, and
    had never heard of multikey collisions until I read this paper last week.