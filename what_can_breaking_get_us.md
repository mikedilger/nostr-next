# What Can We Get by Breaking NOSTR?

"What if we just started over? What if we took everything we have learned while building nostr and did it
all again, but did it right this time?"

That is a question I've heard quite a number of times, and it is a question I have pondered quite a lot myself.

My conclusion (so far) is that I believe that we can fix everything without starting over. There are different levels of breakage, starting over is the most extreme of them. In this post I will describe these levels of breakage and what each one could buy us.


# Cryptography

Your keypair is the most fundamental part of nostr. That is your portable identity.

If the cryptography changed from secp256k1 to ed25519, all current nostr identities would not be usable.

This would be a complete start over, not just a hard fork.

Every other break listed in this post could be done as well to no additional detriment (save for reuse of some existing code) because we would be starting over.

Why would anyone suggest making such a break? What does this buy us?

- [Curve25519 is a safe curve](https://safecurves.cr.yp.to/) meaning a bunch of specific cryptography things that us mortals do not understand but we are assured that it is somehow better.
- Ed25519 is more modern, said to be faster, and has more widespread code/library support than secp256k1.
- Nostr keys could be used as TLS server certificates. [TLS 1.3](https://tools.ietf.org/html/rfc8446#section-4.4.2) using [RFC 7250 Raw Public Keys](https://tools.ietf.org/html/rfc7250) allows raw public keys as certificates. No DNS or certification authorities required, removing several points of failure. These ed25519 keys could be used in TLS, whereas secp256k1 keys cannot as no TLS algorithm utilizes them AFAIK. Note however that we do not need to break nostr cryptography to go down this path - we could just as well keep secp256k1 keys for users and use ed25519 keys for relays.

Besides breaking everything, another downside is that people would not be able to manage nostr keys with bitcoin hardware.

I am fairly strongly against breaking things this far. I don't think it is worth it.

# Signature Scheme and Event Structure

Event structure is the next most fundamental part of nostr. Although events can be represented in many ways (clients and relays usually parse the JSON into data structures and/or database columns), the nature of the content of an event is well defined as seven particular fields. If we changed those, that would be a hard fork.

This break is quite severe. All current nostr events wouldn't work in this hard fork. We would be preserving identities, but all content would be starting over.

It would be difficult to bridge between this fork and current nostr because the bridge couldn't create the different signature required (not having anybody's private key) and current nostr wouldn't be generating the new kind of signature. Therefore any bridge would have to do identity mapping just like bridges to entirely different protocols do (e.g. mostr to mastodon).

What could we gain by breaking things this far?

- We could have a faster event hash and id verification: the current signature scheme of nostr requires lining up 5 JSON fields into a JSON array and using that as hash input. There is a performance cost to copying this data in order to hash it.
- We could introduce a subkey field, and sign events via that subkey, while preserving the pubkey as the author everybody knows and searches by. Note however that we can already get a remarkably similar thing using something like NIP-26 where the actual author is in a tag, and the pubkey field is the signing subkey.
- We could refactor the kind integer into composable bitflags (that could apply to any application) and an application kind (that specifies the application).
- Surely there are other things I haven't thought of.

I am currently against this kind of break. I don't think the benefits even come close to outweighing the cost. But if I learned about other things that we could "fix" by restructuring the events, I could possibly change my mind.

# Protocol Messaging and Transport

The protocol messages of nostr are the next level of breakage. We could preserve keypair identities, all current events, but just break the protocol of how clients and relay communicate this data.

This would not necessarily break relay and client implementations at all, so long as the new protocol were opt-in.

What could we get?

- The new protocol could transmit events in binary form for increased performance.
- It could have clear expectations of who talks first, and when and how AUTH happens, avoiding a lot of current miscommunication between clients and relays.
- We could introduce bitflags for feature support so that new features could be added later and clients would not bother trying them (and getting an error or timing out) on relays that didn't signal support. This could replace much of NIP-11.
- We could then introduce something like negentropy or negative filters (but not that... probably something else solving that same problem) without it being a breaking change.

The downsides are just that if you want this new stuff you have to build it. It makes the protocol less simple, having now multiple protocols, multiple ways of doing the same thing.

Nonetheless, this I am in favor of. I think the trade-offs are worth it. I will be pushing a draft PR for this soon.

# The path forward

I propose then the following path forward:

1. A new nostr protocol over websockets binary (draft PR to be shared soon)
2. Subkeys brought into nostr via NIP-26 (but let's use a single letter tag instead, ok?) via a
   big push to get all the clients to support it (that will be slow and painful as people
   on clients that don't support it yet will not recognize your subkey as you and will miss
   all of your messages during the transition).
3. We assign ed25519 keys to relays, and relays publish their IP address to nostr events.
   This allows us to discard DNS and certificates. Relay identities would however not be nostr
   identities.

We sacrifice these:

1. Faster event hash/verification
2. Composable event bitflags
2. Safer faster more well-supported crypto curve
3. Nostr keys as TLS 1.3 certificates
