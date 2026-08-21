# TOS DNS and `.tos` Naming System

- **Status:** Proposed architecture for review
- **Target:** TOS mainnet and test networks
- **Canonical public suffix:** `.tos`

This document is a target design, not a deployment claim. The inherited
resolver primitives exist, but the `.tos` registrar, domain NFT, bid-vault,
cross-repository APIs, and production activation described below still require
implementation and acceptance evidence.

## 1. Design Purpose

TOS DNS gives people and software stable, human-readable names for TOS
accounts, Agents, Capabilities, services, Messaging endpoints, TOS Sites, and
TOS Storage objects.

Examples:

- `alice.tos` resolves to a wallet or Agent account;
- `translate.alice.tos` resolves to a Capability account;
- `chat.alice.tos` resolves to an Agent whose current Messaging Endpoint is
  discovered and verified through the Messenger protocol;
- `www.alice.tos` resolves to a TOS Site ADNL address; and
- `archive.alice.tos` resolves to a TOS Storage Bag ID.

The design has five goals:

1. preserve the hierarchical resolver interface already implemented in TOS;
2. make second-level `.tos` ownership transferable and wallet-compatible by
   representing each name as a TOS-TEP-62 NFT;
3. prevent mempool name theft and reduce permanent squatting through a
   commit-reveal auction and renewable leases;
4. make names useful across the Agent, service, Messenger, wallet, explorer,
   Sites, and Storage layers; and
5. keep the authority boundary strict: a name is an alias and discovery hint,
   never proof of identity, authorization, payment, or execution.

The last point is essential. If `worker.alice.tos` points to an Agent or
Capability, a consumer must still verify the current finalized Agent policy,
delegation, Capability state, Accepted Quote, escrow, Receipt, and settlement
as required by the relevant protocol. DNS must not become a second registry or
an alternate authority path.

## 2. Decisions and Non-Goals

### 2.1 Canonical suffix

The canonical TOS suffix is `.tos`. TOS must not treat `.ton`, `.gram`, or an
existing Internet DNS suffix as an alias for `.tos`:

- `.ton` belongs to the TON ecosystem;
- `.gram` has historical and third-party ambiguity; and
- reusing an Internet suffix creates cookie, origin, and phishing hazards.

Test networks use the same textual suffix but are separated by their network
domain tuple and root resolver. User interfaces must show the active network
beside every registration or mutation. A name owned on testnet conveys no
right on mainnet.

### 2.2 What TOS DNS is

TOS DNS is:

- an on-chain hierarchical name-resolution protocol;
- a renewable ownership system for second-level `.tos` names;
- a delegation mechanism for subdomains; and
- a common human-readable entry point into independently verified TOS
  protocols.

TOS DNS is not:

- Internet DNS or a replacement for ICANN DNS;
- a certificate authority;
- a source of Agent, Capability, Messaging, Quote, or payment authority;
- a place to store private endpoints, prekeys, chat data, prompts, or service
  manifests; or
- a complete search engine for all Agents and Capabilities.

Capability search remains an explicitly bounded catalog function. Messaging
endpoint rotation remains in signed, expiring Contact Descriptors and TOS DHT.
DNS provides a memorable route to the authoritative object, not a copy of its
mutable state.

## 3. Existing TOS Foundation

TOS inherited a useful DNS foundation from TON and already contains:

- configuration parameter 4 for the masterchain root DNS address;
- the `(int, cell) dnsresolve(slice name, int category)` get-method contract;
- reverse-component, zero-delimited name encoding;
- partial resolution through `dns_next_resolver`;
- `dns_smc_address`, `dns_adnl_address`, `dns_storage_address`,
  `dns_next_resolver`, and `dns_text` TL-B records;
- manual and automatically registering resolver contracts;
- Lite Client and Toslib resolution/update support; and
- TOS Sites resolution in `rldp-http-proxy`.

Relevant implementation locations include:

- `tos/crypto/block/block.tlb`;
- `tos/crypto/smartcont/dns-manual-code.fc`;
- `tos/crypto/smartcont/dns-auto-code.fc`;
- `tos/crypto/smc-envelope/ManualDns.*`;
- `tos/lite-client/lite-client.cpp`;
- `tos/toslib/`; and
- `tos/rldp-http-proxy/DNSResolver.*`.

These components provide resolution primitives, not a production `.tos`
property and registrar system. In particular, the legacy automatically
registering resolver is not the proposed `.tos` collection/NFT contract and
must not be activated as the production registrar without a separate security
review.

The reusable idea from TON DNS is the split between a small root resolver, a
TLD resolver/collection, and per-domain resolver contracts. TOS extends that
model for its own Agent-native ecosystem and does not inherit TON deployment
addresses, `.ton` ownership, auction parameters, or governance decisions.

## 4. Name Syntax and Canonicalization

### 4.1 Version 1 labels

Version 1 accepts only lowercase ASCII labels:

```text
label = [a-z0-9] ([a-z0-9-]{0,61} [a-z0-9])?
```

Rules:

- each label is 1 to 63 bytes;
- the complete name is at most 126 bytes before internal encoding;
- uppercase input is converted to lowercase before display or signing;
- leading and trailing hyphens are forbidden;
- empty labels, control characters, spaces, `/`, `:`, and NUL are forbidden;
- `xn--` labels are reserved in v1; and
- Unicode registration is deferred until a reviewed normalization and
  confusable-character policy exists.

Clients must reject invalid names before constructing a transaction. Contracts
must independently enforce the same rules.

### 4.2 Internal representation

Resolution retains the existing TOS/TON-compatible encoding: split the name on
dots, reverse the components, append a zero byte to every component, and
concatenate them.

```text
translate.alice.tos -> tos\0alice\0translate\0
```

The canonical wire bytes, category hashes, commitment hashes, deterministic
contract addresses, and negative cases must be frozen as cross-language test
vectors before mainnet activation.

## 5. On-Chain Architecture

```text
finalized masterchain config parameter 4
                    |
                    v
             Root DNS Resolver
          delegates the `tos` suffix
                    |
                    v
       `.tos` Collection / TLD Resolver
      owns policy and deterministic item code
                    |
                    v
       `<label>.tos` Domain NFT Item
       owns records and subdomain delegation
                    |
          +---------+----------+
          |                    |
          v                    v
    final DNS record      delegated resolver
                               |
                               v
                      deeper subdomains
```

### 5.1 Root DNS resolver

The root resolver resides in the masterchain. Its address is read from
configuration parameter 4 at a finalized masterchain block.

The root contract should do very little:

- implement `dnsresolve`;
- delegate `.tos` to the canonical `.tos` collection/resolver;
- optionally delegate future TOS-owned suffixes after governance approval; and
- expose version and code-hash getters for operational verification.

The root must not sell names or store user records. Changing parameter 4 or a
root delegation is a chain-governance operation with an announced activation
block and audited code hash.

### 5.2 `.tos` collection and registrar

The `.tos` contract is both:

- the TLD resolver for second-level labels; and
- a TOS-TEP-62 NFT collection whose items are domain contracts.

The NFT item index is:

```text
label_hash = sha256(canonical_label_utf8)
item_index = uint256(label_hash)
```

The domain item address is deterministically derived from the collection
address, item index, and pinned item code. Clients must derive this address
locally and reject a collection response that names a different item.

The collection stores or commits:

- the domain-item code and code hash;
- auction and renewal parameter versions;
- the fee-sink address selected by TOS governance;
- a hash commitment to the launch reservation list;
- the minimum registration deposit and resource bounds; and
- the next allowed contract-version activation point.

There is no mutable administrator power to transfer an active user domain.
Emergency response may pause new auctions, but must not silently rewrite
ownership or records.

### 5.3 Domain NFT item

Each second-level name, such as `alice.tos`, is one smart contract implementing:

- the TOS-TEP-62 NFT item interface;
- `dnsresolve`;
- record set/delete operations;
- subdomain resolver delegation;
- auction settlement;
- renewal and expiry; and
- explicit ownership transfer through the NFT interface.

Required getters include:

```text
get_nft_data()
get_domain()
get_domain_state()
get_auction_info()
dnsresolve(slice remaining_name, int category)
```

`get_domain_state()` returns at least the canonical label hash, owner, status,
`expires_at`, grace end, registrar version, and record-set hash. This data must
be sufficient for wallets, explorers, and independent indexers to reproduce
the visible state.

### 5.4 Subdomains

The owner of `alice.tos` can:

- store records directly for `alice.tos`;
- store records for bounded subdomain names in the domain item; or
- place `dns_next_resolver` on a prefix and delegate the remaining namespace
  to another resolver contract.

Subdomains are not separate NFTs in v1. A delegated resolver may implement its
own ownership policy, but clients must display that the parent owner can change
or revoke the delegation. Tradable subdomain wrappers are deferred.

## 6. Ownership, Auction, Renewal, and Expiry

### 6.1 Registration

All generally available second-level names use a commit-reveal auction. This
prevents an observer from copying a valuable plaintext registration from the
mempool. A commitment is held by a small deterministic bid-vault contract,
derived from the commitment and bidder, so the commit transaction does not
expose the label and the collection does not become an unbounded deposit
ledger. On reveal, the collection verifies the vault's StateInit/code hash and
moves the revealed bid into the auction for the label.

The bid commitment is domain-separated and network-bound:

```text
commitment_cell = begin_cell()
  .store_uint(sha256("tos.dns.bid.v1"), 256)
  .store_uint(network_domain_hash, 256)
  .store_uint(label_hash, 256)
  .store_msg_addr(bidder_address)
  .store_coins(maximum_bid)
  .store_uint(salt, 256)
  .end_cell()

commitment = commitment_cell.hash()
```

The exact cell layout is frozen by the TIP and shared vectors; implementations
must not hash ad hoc string concatenation. The auction has fixed commit,
reveal, and settlement windows. The winner is the highest valid revealed bid
and pays the second-highest valid bid plus the configured minimum increment,
capped at the winner's bid. A tie is resolved by the earlier finalized
commitment. Unrevealed commitments incur a bounded penalty; valid losing bids
are refundable through owner-initiated withdrawals. Settlement never loops
over bidders or synchronously pushes every refund.

Exact durations, minimum bids, increments, penalties, and the fee sink are
economic parameters, not constants invented by this document. They must be
approved and frozen before deployment. No auction proceeds may be described as
validator rewards, burns, treasury revenue, or protocol revenue until the
corresponding TOS economic policy explicitly defines that treatment.

### 6.2 Reserved names

Before activation, governance publishes a finite, reviewable reservation list
and commits its Merkle root in the collection configuration. It should contain
only protocol-critical names, security-sensitive names, and names required for
network operation. After activation, governance cannot expand the list for an
already-open namespace without a separately announced contract upgrade.

### 6.3 Lease and renewal

A settled domain receives a one-year lease. The owner can renew during a
bounded renewal window at the configured renewal price.

At `expires_at`:

- resolution stops immediately;
- record changes and transfers stop; and
- a 30-day recovery grace begins, during which only the recorded owner may
  renew.

After grace, the name becomes available for a new commit-reveal auction. The
contract must treat an expired name as unresolved even if physical cleanup has
not yet run. Garbage collection is therefore an optimization, not a security
boundary.

## 7. Record Categories

Categories remain `sha256(UTF-8 category name)` and category zero requests the
complete record dictionary. Version 1 reuses existing TL-B record encodings;
new semantics are selected by category instead of inventing unnecessary wire
types.

| Category name | Record encoding | TOS meaning | Required follow-up verification |
|---|---|---|---|
| `wallet` | `dns_smc_address` | Default payment account | Address/network validation and wallet transaction confirmation |
| `agent` | `dns_smc_address` | Finalized Agent account | Agent code/identity, live state, controller policy, and revocation |
| `capability` | `dns_smc_address` | Finalized Capability account | Owner Agent, version, revocation/tombstone, manifest, and policy |
| `messenger` | `dns_smc_address` | Agent used for Messaging contact | Finalized Agent plus current Endpoint delegation, Contact Descriptor, and DHT locator |
| `site` | `dns_adnl_address` | TOS Site entry point | ADNL identity and protocol support |
| `storage` | `dns_storage_address` | TOS Storage Bag ID | Bag hash/content verification and application policy |
| `dns_next_resolver` | `dns_next_resolver` | Delegated subdomain resolver | Resolver address, cycle, depth, and response validation |
| `text` | `dns_text` | Human-readable presentation metadata | Never authoritative; escape before display |

An `agent`, `capability`, or `messenger` record stores an account address, not
an HTTPS URL or mutable private endpoint. A service name normally points to a
Capability through the `capability` category. Multiple services should use
separate subdomains rather than an unbounded list inside one record.

Future typed record schemas require a TIP, new positive and adversarial
vectors, and at least two independent decoders before being marked stable.

## 8. Resolution Algorithm and Result Provenance

A conforming resolver performs these steps:

1. parse and canonicalize the name under the v1 rules;
2. obtain a finalized masterchain block and configuration parameter 4;
3. encode the name in reverse zero-delimited form;
4. call `dnsresolve` on the root at that same finalized block;
5. validate the consumed-bit count and component boundary;
6. if partially resolved, require a valid `dns_next_resolver`, detect cycles,
   and continue with the unresolved suffix;
7. stop after at most eight resolver hops;
8. decode only the requested category and reject trailing or malformed data;
9. confirm that the domain is active at the block time; and
10. return both the value and its provenance.

The structured result exposed by SDKs and APIs must include:

```text
canonical_name
category_name and category_hash
record_type and decoded value
root_resolver_address
resolver_path[]
masterchain_block_id / seqno / root hash
domain_item_address
domain_expires_at
resolved_at_chain_time
```

Clients resolving Agent-native objects then perform their protocol-specific
finalized-state verification. They must use the account address or object ID,
not the input name, in signatures, nonces, Accepted Quotes, Events, receipts,
and durable journals.

### 8.1 Caching

Positive cache lifetime is bounded by the earliest of:

- domain expiry;
- a record-specific validity limit, if a future record defines one;
- the current finalized-state refresh policy; and
- a conservative client maximum.

Negative results use a short bounded cache. A reorg or a change in the trusted
finalized checkpoint invalidates affected cache entries. Gateways may cache
resolution results but never become the source of authority.

### 8.2 Reverse lookup

Reverse lookup is an indexer projection, not a consensus mapping. An explorer
or wallet may show a primary name for an address only after forward-confirming
that the name currently resolves to that address at a finalized block. A user
must be able to see the raw address and the verification checkpoint.

## 9. Agent, Service, and Messenger Integration

### 9.1 Agent and Capability names

Names improve discovery and presentation, but the Native Registry remains the
authority:

```text
name -> Agent/Capability account address -> finalized typed state
```

A Gateway search result may include a verified `.tos` alias, but ranking,
search metadata, or gateway-local data cannot create ownership or authority.
When a Capability transfers, is revoked, or is tombstoned, the name does not
override that state.

### 9.2 Messenger names

The Messenger resolution chain is:

```text
chat.alice.tos
  -> `messenger` Agent account
  -> finalized Agent policy and delegation digest
  -> signed, expiring DHT locator
  -> content-addressed Contact Descriptor
  -> current Messaging Endpoint and prekey generation
```

DNS must not contain prekeys, private contact graphs, Mailbox bearer tokens, or
long-lived endpoint URLs. A Messenger Event is signed and journaled against
the resolved Agent and Endpoint identities, never against the string
`chat.alice.tos`.

### 9.3 TOS Sites and Storage

Existing `site` and `storage` categories remain compatible with the current
TL-B records. `rldp-http-proxy` should accept `.tos` only after it implements
the finalized-root, hop-limit, cycle, expiry, and record-validation rules in
this document.

## 10. Security Requirements

Implementations must address at least these threats:

- **front-running:** commit-reveal hides the label and bid until reveal;
- **homographs:** v1 is lowercase ASCII only and reserves `xn--`;
- **network confusion:** commitments, caches, and UI confirmations are bound
  to the network domain tuple;
- **malicious resolvers:** strict consumed-bit, component-boundary, schema,
  hop-limit, and cycle checks;
- **stale authority:** Agent, Capability, and Messaging state is re-read from a
  finalized checkpoint after DNS resolution;
- **expired records:** contracts fail closed at `expires_at`, independent of
  garbage collection;
- **record substitution:** mutations require the current NFT owner and are
  reflected in finalized state;
- **admin seizure:** no ordinary administrator transfer path exists;
- **upgrade substitution:** root, collection, and item code hashes are pinned,
  versioned, and announced before activation;
- **gateway deception:** clients can reproduce resolution from chain state and
  receive provenance with cached API results;
- **display injection:** text records are untrusted UTF-8 presentation data;
  and
- **payment mistakes:** wallets show the resolved raw address, network,
  checkpoint age, and domain expiry before signing.

Names must never be used directly as cryptographic principals, database keys
for irreversible actions, or signature-domain inputs. Durable systems store
the canonical resolved identity and may separately retain the name as display
metadata.

## 11. Repositories Requiring Code

The feature is cross-repository. Completion requires the following ownership
map; implementing only the smart contracts is not a complete `.tos` product.

| Repository | Required coding work | Acceptance evidence |
|---|---|---|
| `tosnetwork/TIP` | Publish the normative TOS DNS interface, category registry, lifecycle, operation codes, and canonical vectors | Accepted TIP with frozen hashes and compatibility rules |
| `tosnetwork/tos` | Implement audited root, `.tos` collection, domain NFT, bid-vault, commit-reveal auction, renewal/expiry, and subresolver contracts; pin TL-B/category constants; activate config parameter 4; harden Lite Client, Toslib, `rldp-http-proxy`, JSON-RPC, genesis tooling, and emulator tests | Deterministic builds; contract/unit/adversarial tests; local multi-validator resolution and auction evidence; code hashes and activation plan |
| `tosnetwork/tos` (`tosctl`) | Add `domain normalize`, `commit`, `reveal`, `settle`, `renew`, `transfer`, `record set/delete`, `delegate`, `resolve`, and `inspect` commands with offline signing support | CLI golden vectors, restart-safe transaction tracking, hardware/offline signer tests, and real localnet lifecycle |
| `tosnetwork/tos-service-spec` | Specify how `.tos` aliases may identify Agent, Capability, and Messenger entry points without changing `tos_service_v1` authority | Normative boundary text, negative cases, and shared vectors; no alternate registry semantics |
| `tosnetwork/tos-service-protocol` | Add a Go resolver/verifier library that consumes finalized TOS state, reproduces canonical encoding and category hashes, returns provenance, and resolves aliases to Native object IDs before existing verification | Cross-language vector parity, strict-majority finalized reads, cycle/expiry/reorg tests, and API compatibility |
| `tosnetwork/tos-service-gateway` | Expose bounded read-only resolution and verified aliases in discovery results; cache only with checkpoint and expiry bounds | Gateway restart/cache tests and proof that a Gateway cannot create or mutate name or Native authority |
| `tosnetwork/tos-messenger` | Accept `.tos` contact input, resolve `messenger` to an Agent, then execute the existing finalized delegation -> DHT locator -> Contact Descriptor checks; persist IDs rather than names | Substitution, stale delegation, name transfer, expiry, DHT rotation, and three-transport replay tests |
| `tosnetwork/openfox` | Add name input/display at the human boundary while binding sessions, policy, purchases, and execution to resolved Agent/Capability IDs | Name-transfer and stale-cache tests proving no session or purchase authority follows an old alias |
| `tosnetwork/toscan` | Index domain NFTs, auctions, renewals, records, transfers, and expiries; provide forward-confirmed reverse lookup and domain pages | Reorg-safe index tests, raw-address display, checkpoint provenance, and localnet lifecycle coverage |
| `tosnetwork/ios` | Resolve names for send/contact flows and manage domain NFTs with explicit address/network/expiry confirmation | Unit, UI, signer, and testnet lifecycle tests |
| `tosnetwork/android` | Match the iOS resolver, send protection, and domain-management behavior without trusting inherited TON APIs | Cross-platform vectors, UI tests, and TOS-native API boundary tests |
| new `tosnetwork/tos-domains` | Provide the public registrar and management web application; use wallet signing and chain APIs without holding owner keys | Commit/reveal recovery UX, transaction-state recovery, phishing defenses, CSP/security review, and testnet acceptance |
| `tosnetwork/doc` | Maintain this architecture, operator runbooks, category registry links, deployment addresses, code hashes, and user documentation | Documentation review tied to released commits and deployed network parameters |

### 11.1 Repositories that should not gain authority

- `tos-ai` may accept already resolved Agent/Capability identities but should
  not perform name-based execution authorization inside the runner.
- `freecity` may display verified aliases but remains a replaceable projection.
- `tos-homepage` may link to the registrar only after deployment; it must not
  host registrar keys or claim unshipped functionality.
- wallets, explorers, Gateways, Messenger Relays, and DHT nodes never own the
  namespace or override finalized chain state.

## 12. Delivery Plan

### Phase 0 — specification freeze

1. Publish the TIP and category registry.
2. Freeze normalization, internal encoding, category hashes, operation codes,
   commitment bytes, deterministic addresses, and positive/adversarial vectors.
3. Decide auction parameters, renewal pricing, fee sink, reservation root, and
   upgrade governance.
4. Complete contract threat modeling and independent review.

### Phase 1 — chain and local tooling

1. Implement root, collection, bid-vault, domain item, and delegated resolver
   contracts.
2. Add emulator and property tests for complete lifecycle and malformed cells.
3. Add `tosctl`, Lite Client, Toslib, JSON-RPC, and genesis/config support.
4. Demonstrate register -> resolve -> update -> delegate -> transfer -> expire
   -> grace renew -> re-auction on a multi-validator local network.

### Phase 2 — testnet product

1. Activate the audited root through configuration parameter 4.
2. Deploy `tos-domains`, TOSCan indexing, and wallet integrations.
3. Run public testnet auctions with no mainnet ownership promise.
4. Measure gas, state growth, resolver latency, reorg behavior, abandoned bid
   cleanup, and support burden.

### Phase 3 — Agent-native integration

1. Add service-protocol and Gateway alias resolution.
2. Add Messenger and OpenFox contact resolution.
3. Prove that name transfer, expiry, stale caches, revoked delegations, and
   tombstoned Capabilities all fail closed.
4. Consume the same vectors in C++, Go, Swift, Kotlin, and TypeScript.

### Phase 4 — mainnet activation

Mainnet activation requires:

- published source and reproducible contract code hashes;
- at least two independent security reviews of registrar and item contracts;
- a second independent resolver implementation;
- wallet and explorer support for raw-address confirmation;
- a frozen reservation root and economic parameters;
- upgrade and incident runbooks;
- testnet lifecycle evidence covering expiry and re-auction; and
- an announced configuration-parameter-4 activation block.

## 13. Required Test Matrix

At minimum, the shared corpus covers:

- valid and invalid labels, maximum lengths, case folding, and reserved `xn--`;
- internal reverse encoding and every standard category hash;
- deterministic collection/item addresses across mainnet and testnet;
- commit copying, salt substitution, bid substitution, non-reveal, ties, refund
  replay, and settlement replay;
- record update/delete, NFT transfer, delegated subdomains, partial resolution,
  resolver loops, excess depth, malformed consumed-bit counts, and trailing data;
- expiry, grace renewal, cleanup delay, and re-auction;
- finalized checkpoint change and indexer rollback;
- Agent revocation, Capability tombstone/transfer, Messaging delegation
  rotation/revocation, and stale DHT locator;
- malicious Gateway cache responses and independently reproduced resolution;
  and
- wallet confirmation of raw address, network, and expiry.

## 14. Operational Inspection

Until the new product contracts and APIs are implemented, existing tools can
inspect the inherited resolver interface:

```bash
cd build
./lite-client/lite-client -C /data/tos-global.json
```

```text
getconfig 4
dnsresolve <domain> <category>
dnsresolve <domain> -1
```

Category `-1` in the Lite Client is a diagnostic convention for resolver
chaining; protocol categories are unsigned 256-bit hashes, with category zero
meaning all records.

Operators must resolve against a recent finalized configuration, inspect the
resolver path, and verify the final record type. A successful DNS lookup alone
does not authorize a transfer, Agent action, Messenger session, or service
purchase.

## 15. Related Documents and References

TOS documents:

- [ConfigParam.md](ConfigParam.md)
- [TosSites.md](TosSites.md)
- [tos-tep-token-standards.md](tos-tep-token-standards.md)
- [tos-account-permission-model.md](tos-account-permission-model.md)
- [ai-actors.md](ai-actors.md)
- [ai-inference-sharing-tos-domains.md](ai-inference-sharing-tos-domains.md)
- [TOS Agent-native Messenger architecture](https://github.com/tosnetwork/tos-service-spec/blob/main/docs/AGENT_NATIVE_MESSENGER_V1.md)

Inherited design references:

- [TON DNS documentation](https://github.com/ton-blockchain/docs/blob/main/content/foundations/web3/ton-dns.mdx)
- [TEP-81: TON DNS Standard](https://github.com/ton-blockchain/TEPs/blob/master/text/0081-dns-standard.md)
- [TON DNS reference contracts](https://github.com/ton-blockchain/dns-contract)

These TON references explain the inherited resolver mechanics. This document,
the accepted TOS TIP, and deployed TOS contract code are authoritative for the
`.tos` namespace.
