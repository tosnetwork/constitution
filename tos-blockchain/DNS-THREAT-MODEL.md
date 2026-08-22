# TOS DNS v1 Threat Model

- **Scope:** TIP-1 implementation candidate
- **Date:** 2026-08-22
- **Deployment status:** not deployed on mainnet
- **Independent review status:** required and not yet complete

This document turns the security requirements in [DNS.md](DNS.md) into explicit
assets, trust boundaries, attacker goals, controls, and release evidence. It is
an internal threat model, not an independent audit report.

## 1. Security invariants

1. Configuration parameter 4 selects the only root; absence fails as
   `DNS unavailable`.
2. One resolution uses one finalized checkpoint, visits at most eight resolvers,
   rejects cycles and non-component boundaries, and validates every record tag.
3. A Domain Item is usable only when its Collection, label index, deterministic
   address, auction state, and 366-day renewal clock all verify.
4. A `.tos` name is never an identity or authorization principal. Native Agent
   and Capability use requires address-to-object-ID re-derivation and finalized
   typed state. Purchases, signatures, sessions, receipts, and journals bind IDs.
5. Root, Collection, and item code are immutable. An upgrade deploys a new Root
   and Collection and requires a critical parameter-4 vote.
6. ConfigParam 80 is absent in v1. Auction proceeds are permanently locked.

## 2. Assets and trust boundaries

| Asset | Boundary | Required trust |
|---|---|---|
| Namespace root | finalized masterchain ConfigParam 4 | consensus plus critical governance |
| Name ownership and records | immutable Collection and Domain Item state | finalized chain state and pinned code/artifact hashes |
| Resolution result | resolver/client boundary | assurance class reported exactly (`evaluated`, `state_proved`, `chain_anchored`, or `quorum_agreed`) |
| Agent/Capability identity | DNS-to-Native boundary | deterministic Native address and finalized typed Registry state |
| Wallet transfer | user-signing boundary | raw address, network, checkpoint, auction and renewal disclosure |
| Registrar | browser/wallet boundary | non-custodial transaction construction; owner signs exact payload |
| Index/cache | chain-to-projection boundary | checkpoint provenance, bounded retention, rollback/invalidation |

Gateways, RPC endpoints, registrars, explorers, DNS text records, reverse lookup,
and displayed aliases are untrusted inputs or replaceable projections.

## 3. Threats and controls

| Threat | Failure | Mandatory control | Release evidence |
|---|---|---|---|
| Root substitution | every name redirects | parameter 4 critical; pin expected network and code hashes; announce activation | critical-vote rehearsal, artifact manifest, client mismatch test |
| Malicious/compromised RPC | forged record or checkpoint | strict majority of independent endpoints or proof-verified client; never upgrade provenance label | disagreement, stale/future checkpoint, and mixed-block tests |
| Resolver loop/depth bomb | resource exhaustion | cycle check before contact; eight contacts maximum; strictly shorter remainder | 8-hop success, 9/100-hop refusal, repeated-address test |
| Consumed-bit confusion | skip/split labels | positive byte-aligned count at a component boundary, including before/after NUL | worked-trace and adversarial boundary vectors |
| Category confusion | wallet/Agent record decoded as another type | exact category hash and TL-B tag; no trailing data | mismatch/unknown-tag/trailing-data tests |
| Fake Domain Item | attacker-controlled resolver impersonates item | canonical third hop; `get_nft_data`, Collection, slice-hash index and Collection mapping all match | substitution tests at one checkpoint |
| Auction/expiry reuse | stale record accepted while ownership is unsafe | reject every non-zero auction end, missing clock, and `now > last_fill+31622400` | exact deadline and ended-unfinalized tests |
| Record survives transfer/release | old owner endpoint remains active | display inheritance; new owner reviews/replaces records; clients still apply lifecycle gate | transfer/re-auction record-retention tests |
| Name-to-Native substitution | name grants Agent/Capability authority | resolve address, decode typed state, recover ID, re-derive address, then use ID only | mismatch, revoke, tombstone, transfer tests |
| Cache poisoning/amplification | stale authority or memory exhaustion | checkpoint/lifecycle-keyed bounded cache; 1024 entries; 256 in-flight names; 64 waiters/name; request coalescing | eviction, deadline, concurrency and recovery tests |
| Reorg or record mutation mid-TTL | stale result remains until TTL | event-driven subtree invalidation and rollback by checkpoint | **open release gate**; integration evidence required |
| Registrar phishing/XSS | user signs wrong name/bid/address | lowercase canonical preview, raw payload, network, CSP, no custody, untrusted text escaping | CSP review, signer/UI tests, homograph warnings |
| Output/token file attack | credential disclosure or evidence overwrite | owner-private regular token file, reject symlink, create evidence with `O_EXCL` mode 0600 | OpenFox CLI unit tests |
| Supply-chain drift | source/vector/artifact mismatch | pinned TON commit, parity CI, canonical corpus checksum, two independent reproducible builders | parity report and **open two-builder gate** |
| ConfigParam 80 surprise | hidden seizure/reservation power | keep absent; validator rejects undeclared 80; later enablement requires new TIP | absent-param tests and configuration inspection |
| Locked-fund misconception | operators expect auction treasury funds | disclose permanent lock; no withdrawal/admin key | source review and public economics disclosure |

## 4. Abuse and availability limits

Long labels, deep delegation, repeated cache misses, concurrent lookups, malformed
BOCs, extreme VM integers, and RPC fan-out are attacker-controlled. All parsing
fails closed, integer conversions are checked, encoded input is at most 127
bytes, resolver contact and concurrency budgets are fixed, and caches are
bounded. A malformed name or unavailable DNS must not prevent raw-address wallet
use or Native-ID workflows.

## 5. Privacy

DNS records, bids, ownership, and queries to public RPC endpoints are public.
DNS must not store private endpoints, prekeys, prompts, session identifiers, or
service secrets. Messenger records point only to an Agent; signed expiring DHT
and Contact Descriptor mechanisms carry mutable contact material.

## 6. Residual risks and release gates

- Event-driven record/delegation invalidation and reorg rollback are not yet
  implemented for every cache consumer. The current 300-second cap reduces but
  does not eliminate the stale-result window.
- Contract code has not completed an independent review. Authors of this model
  and implementation do not satisfy that requirement.
- iOS changes have not been compiled on macOS/Xcode in this Linux environment.
- Public multi-validator testnet evidence must cover auction, refunds,
  finalization, inherited records, renewal, release, and re-auction.
- The 2027-01-01 timestamp is conditional. If release gates are not complete in
  time, governance must publish a revised timestamp and regenerated artifacts.

Mainnet activation is blocked until these items and DNS.md gates G1–G10 have
commit-bound evidence.

