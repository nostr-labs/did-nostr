# A `did:nostr` Primer

> A Nostr keypair is already a decentralized identifier. `did:nostr` gives it a W3C DID
> form: the identifier **is** the public key, the DID document can be generated from it
> **offline**, and everything else — profile, relays, social graph — is optional
> enrichment derived from signed Nostr events.

This primer is for two audiences: people from the **DID / Verifiable Credentials world**
who don't know Nostr, and people from the **Nostr world** who don't know DIDs. It answers
the questions implementers actually ask (several come straight from the
[CCG mailing-list thread](https://lists.w3.org/Archives/Public/public-credentials/2026Jul/0001.html)
on the v0.1.0 announcement). It is **informative** and complements the normative
[specification](https://nostrcg.github.io/did-nostr/).

## If you know DIDs but not Nostr

Nostr is a pub/sub protocol over WebSockets. Users hold a secp256k1 keypair, sign small
JSON **events** (a post, a profile, a contact list), and publish them to **relays** —
simple, interchangeable servers that store and forward events. Clients can read a user's
events back from *any* relay that has a copy; no relay is authoritative, and relays can
be added or dropped freely. Every event carries the author's public key and a BIP-340
Schnorr signature, so authenticity is checked by the reader, not trusted from the server.

Events have a `kind` number. Three kinds matter for this method:

| Kind | Contents | Feeds into the DID document as |
| --- | --- | --- |
| 0 | profile metadata (name, about, picture) | `profile` |
| 3 | contact list | `follows` |
| 10002 | preferred relays | `service` (type `Relay`) |

[NIP-01](https://github.com/nostr-protocol/nips/blob/master/01.md) is the short technical
overview of the protocol; for a higher-level introduction there is a good
[video overview by Jack Dorsey and Rabble](https://www.youtube.com/watch?v=NS5JI-ksaXs).

## If you know Nostr but not DIDs

A [Decentralized Identifier](https://www.w3.org/TR/did-core/) (DID) is a W3C-standard
identifier that resolves to a **DID document** — a small JSON-LD document describing the
identity: its public keys (*verification methods*), what they may be used for
(*authentication*, *assertionMethod*), and optional *service endpoints*. DIDs are the
identifier layer used by Verifiable Credentials, Solid/Linked Web Storage, and a growing
ecosystem of wallets and resolvers.

`did:nostr` is the bridge: your existing npub, expressed in a form that W3C tooling
understands. You don't register anything anywhere — if you have a Nostr key, you already
have a `did:nostr`.

## The identifier

```
did:nostr:<64-character lowercase hex public key>
```

For example:

```
did:nostr:124c0fa99407182ece5a24fad9b7f6674902fc422843d3128d38a0afbee0fdd2
```

The identifier uses the **raw hex** public key, not the Bech32 `npub` form. The two
encode the same key; `npub` is display-only:

- Raw key: `124c0fa99407182ece5a24fad9b7f6674902fc422843d3128d38a0afbee0fdd2`
- DID: `did:nostr:124c0fa9…bee0fdd2`
- Display npub: `npub1cpxejnc58zpcuyh0pt8gvkzpv34qxceu0sqp7jec2nk9nut7p5zs4zyx4c`

Because the key is *in* the identifier, the binding between identifier and key is
permanent — there is no registry to consult and nothing to squat. (The flip side —
what happens if the key is compromised — is covered [below](#what-the-method-does-not-do-yet).)

## The DID document

The minimal document is a pure function of the public key. Given only the identifier,
any resolver produces:

```json
{
    "@context": ["https://www.w3.org/ns/did/v1", "https://www.w3.org/ns/cid/v1", "https://w3id.org/nostr/context"],
    "id": "did:nostr:124c0fa99407182ece5a24fad9b7f6674902fc422843d3128d38a0afbee0fdd2",
    "type": "DIDNostr",
    "verificationMethod": [
        {
            "id": "did:nostr:124c0fa99407182ece5a24fad9b7f6674902fc422843d3128d38a0afbee0fdd2#key1",
            "type": "Multikey",
            "controller": "did:nostr:124c0fa99407182ece5a24fad9b7f6674902fc422843d3128d38a0afbee0fdd2",
            "publicKeyMultibase": "fe70102124c0fa99407182ece5a24fad9b7f6674902fc422843d3128d38a0afbee0fdd2"
        }
    ],
    "authentication": ["#key1"],
    "assertionMethod": ["#key1"]
}
```

Reading it line by line:

- **`id`** — the DID itself.
- **`type: "DIDNostr"`** — types the document explicitly for linked-data processing
  (see the [resource model note](./resource-model.md)).
- **`verificationMethod`** — the same public key, re-encoded as a
  [Multikey](https://www.w3.org/TR/cid-1.0/): parity byte `02` + multicodec prefix
  `e701` + the x-only key, multibase-encoded with `f` (base16-lower). This is what makes
  the key usable by W3C Data Integrity tooling (see the
  [parity model note](./parity-model.md) for the `02`/`03` details).
- **`authentication` / `assertionMethod`** — the key may be used to log in and to sign
  claims (e.g. Verifiable Credentials).

An **enriched** document adds material projected from the owner's signed events:
`service` entries for relays (kind 10002), a `profile` (kind 0), a `follows` list
(kind 3), optional `alsoKnownAs` links to the same identity elsewhere (a WebID, a
fediverse handle, another DID), and a `modified` timestamp — the most recent
`created_at` across the signed parts, so consumers can detect staleness.

## Resolution: an offline core with optional enrichment

This is the part that most often surprises DID implementers, so it is worth stating
plainly: **the baseline resolver needs no network at all.**

**Layer 0 — offline (the conformance floor).** Parse the pubkey out of the identifier
and construct the minimal document above. A conforming resolver MUST support this. It
already provides full cryptographic functionality — authentication and signature
verification work with no relay, no domain, no lookup.

**Layer 1 — relay enrichment (optional).** Query Nostr relays for the user's kind 0 / 3
/ 10002 events, verify their signatures, and project them into the document as
`profile`, `follows`, and `service`. Which relays? A local cache, a set of defaults, or
relays already discovered for that key — since events are signed, *any* relay with a
copy is as good as any other.

**Layer 2 — HTTP / `.well-known` (optional).** Any HTTP host can serve a pre-built
document at:

```
https://<domain>/.well-known/did/nostr/<pubkey>.json
```

This makes resolution a cacheable HTTP GET — CDN-friendly, no WebSocket needed — with
normal HTTP caching (`ETag`, `Cache-Control`) on top.

### "How do I know which domain to query?"

You don't have to — that is the point of the layering. There is no canonical domain for
a `did:nostr`, because the identifier is self-certifying and the baseline document is
generated locally (Layer 0). A `.well-known` host is a **cache, not a root of trust**:
anyone can mirror documents, and a consumer can always verify the signed source events
or fall back to Layers 0–1. Domains enter the picture only when you have one in hand —
your own resolver, a mirror you operate or trust, or a hint such as a `didResolver`
field in the user's [NIP-05](https://github.com/nostr-protocol/nips/blob/master/05.md)
identifier. If you have no domain, you have lost nothing: resolve offline and enrich
from relays.

### "Are updates guaranteed to reach resolvers?"

No — and the method is designed so that they don't have to be. Updates happen at the
event layer: publishing a newer signed kind 0 / 3 / 10002 event changes what the
document projects on the *next* resolution. Relay propagation is **eventually
consistent** — an event published to one relay reaches others only if someone
rebroadcasts it, and a resolver querying a different relay set may see older state for a
while. The spec therefore keeps every guarantee on the layer that can honour it:

- The **core document** (identifier, verification method) never depends on relay data,
  so it can never be stale or wrong.
- The **enrichment** is best-effort and self-describing: each part carries its
  `created_at`, the document carries `modified` (the max across parts), and every part
  is signed, so a consumer can always tell *which* state it is looking at and refuse to
  overwrite newer state with older.

If your application needs stronger freshness, publish to (and resolve from) an
overlapping relay set, or serve your own `.well-known` document — both are deployment
choices, not protocol changes.

## What the method does not do (yet)

Honesty section. The identifier is permanently bound to one key, so:

- **Key rotation / deactivation** — no in-method mechanism yet. If the key is lost or
  compromised there is no in-method recovery. Semantics (e.g. `alsoKnownAs`-based
  migration, deactivation signals) are under discussion in
  [#73](https://github.com/nostrcg/did-nostr/issues/73).
- **Key recovery** — exploratory, tracked in
  [#137](https://github.com/nostrcg/did-nostr/issues/137).

Until then, key management advice is the standard Nostr advice: guard the private key,
back it up, and treat it as the identity.

## Try it

- Generate an identity: `npx create-agent`
  ([create-agent](https://github.com/melvincarvalho/create-agent))
- Resolve in code or CLI: the [`did-nostr`](https://github.com/melvincarvalho/did-nostr)
  npm package — offline, `.well-known`, and relay resolution, plus a
  [DIF `did-resolver`](https://github.com/decentralized-identity/did-resolver) driver
- Browse documents visually: the
  [DID:nostr Explorer](https://nostrapps.github.io/did-explorer/)
- A production `.well-known` endpoint:
  [nostr.rocks resolver](https://nostr.rocks/.well-known/did/nostr/32e1827635450ebb3c5a7d12c1f8e7b2b514439ac10a67eef3d9fd9c5c68e245.json)
- Conformance: the [test vectors](./test-vectors) used by implementations

## References

- [did:nostr specification](https://nostrcg.github.io/did-nostr/) — the normative text
- [NIP-01](https://github.com/nostr-protocol/nips/blob/master/01.md) — Nostr protocol basics
- [DID Core](https://www.w3.org/TR/did-core/) — W3C Decentralized Identifiers v1.0
- [Controlled Identifiers v1.0](https://www.w3.org/TR/cid-1.0/) — defines `Multikey` / `publicKeyMultibase`
- [`parity-model.md`](./parity-model.md) — companion note: x-only keys and Multikey parity
- [`resource-model.md`](./resource-model.md) — companion note: subject vs document, provenance
- [CCG thread on v0.1.0](https://lists.w3.org/Archives/Public/public-credentials/2026Jul/0001.html) — where these questions were asked
