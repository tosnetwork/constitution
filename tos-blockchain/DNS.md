# TOS DNS and `.tos` Naming System

- **Status:** Proposed architecture for review
- **Target:** TOS mainnet and test networks
- **Canonical public suffix:** `.tos`

This document is a target design, not a deployment claim. The inherited
resolver primitives exist. The on-chain naming contracts live in the
`tosnetwork/tos` tree at `crypto/smartcont/dns/`, ported from the TON
reference contracts (`ton-blockchain/dns-contract` remains the upstream
parity source); the registrar and management application lives in the same
tree at `domains/`. Both still require deployment and acceptance evidence. The auction and renewal rules intentionally follow the current TON
contracts; Agent-native APIs and production activation are TOS integrations,
not capabilities inherited from TON.

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
3. keep the auction, renewal, and release state machine aligned with the latest
   official TON DNS contract, with no TOS-only redesign by default;
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

### 3.1 What the current code does *not* provide

The following statements were verified against the working tree and must not be
assumed away by an implementer:

- **Configuration parameter 4 is unset at genesis by default.**
  `crypto/fift/lib/Config.fif` now provides `config.dns_root_smc!` and
  `tos/crypto/smartcont/gen-zerostate.fif` carries a commented activation
  example, but no genesis profile enables it; local test networks may pin it
  through the tostester zerostate generator (`NetworkConfig.dns_root_addr`,
  exercised by `scripts/dns-e2e.py`). On a network built from the default
  zero state, `block::Config::get_dns_root_addr()`
  (`crypto/block/mc-config.cpp:869`) returns
  `configuration parameter 4 ... is absent` and every client-side resolution
  fails closed. A governance action is required on an existing network. Whether
  new genesis/proposal code is required is not yet established: Phase 0 first
  exercises the generic Config Contract and zero-state tooling, then adds only
  the wrappers or helpers that the evidence shows are missing.
- **Parameter 4 is neither mandatory nor critical.** The genesis lists are
  `mandatory_params = (0 1 9 10 12 14 15 16 17 18 20 21 22 23 24 25 28 34)` and
  `critical_params = (-999 -1000 -1001 0 1 3 9 10 12 14 15 16 17 32 34 36)`
  (`gen-zerostate.fif:208-209`); the validator-side mandatory list is
  `{18, 20, 21, 22, 23, 24, 25, 28, 34}` (`crypto/block/block.cpp:1885`).
  Parameter 4 appears in none of them, so replacing the entire `.tos` root
  resolver would today pass under the *ordinary* proposal setup
  `(2 3 2 2 ...)` rather than the critical setup `(4 7 4 2 ...)`. See §5.1.
- **Only two category constants are pinned by code.**
  `sha256("dns_next_resolver")` is fixed in
  `crypto/smc-envelope/ManualDns.h:34`, `crypto/smartcont/dns-manual-code.fc:347`
  and `crypto/smartcont/dns-auto-code.fc:480`; `sha256("site")` is fixed in
  `rldp-http-proxy/DNSResolver.cpp:69`. `wallet`, `storage`, `text`, `agent`,
  `capability`, and `messenger` are **not** referenced anywhere in TOS code and
  are proposals of this document (§7).
- **There is no deployed TOS-adapted production `.tos` collection or domain
  item.** The ported `.tos` sources live in-tree at `crypto/smartcont/dns/`:
  the Root is adapted to the single `.tos` zone, the Collection and Item are
  unchanged from the pinned upstream commit, and the launch timestamp is a
  localnet placeholder pending governance (§6.6). Nothing is deployed.
  Elsewhere `crypto/smartcont/` contains only `dns-manual-code.fc` and
  `dns-auto-code.fc`.
  The DNS collection and item sources under
  `crypto/func/auto-tests/legacy_tests/dns-collection/` and
  `.../tele-nft-item/` are **FunC compiler test fixtures inherited from TON**.
  They are useful as behavioural references (§5.2, §5.4) and are cited as such
  below, but they are not TOS contracts, are not audited, and must not be
  deployed.
- **There is no DNS-specific code-hash policy.** TOS has an on-chain
  audited-code registry only for AIPoW (`ConfigParam 93`). Nothing equivalent
  exists for a root resolver, collection, or item. Production must pin
  reproducible hashes and an upgrade policy, but §10 deliberately leaves open
  whether enforcement is on-chain, in immutable contract policy, or in client
  release manifests.

The reusable idea from TON DNS is the split between a small root resolver, a
TLD resolver/collection, and per-domain resolver contracts. TOS extends that
model for its own Agent-native ecosystem and does not inherit TON deployment
addresses, `.ton` ownership, or governance decisions. Its auction algorithm is
the compatibility baseline; deployment-time timestamps and TOS-denominated
constants remain explicit network choices.

### 3.2 Reuse boundary and delivery profiles

Creating `.tos` does **not**, by itself, require a consensus, TVM, TL-B record,
Lite Server, or validator change. The existing parameter-4 and `dnsresolve` ABI
are already sufficient to route `tos\0` to a collection, and the existing
`dns_smc_address` encoding is sufficient for the proposed `wallet`, `agent`,
`capability`, and `messenger` categories; category names are hashes and do not
require a new `DNSRecord` constructor.

Two precisions, because both are easy to state wrongly:

- **The suffix lives in the Root's code, not its data.** Upstream `root-dns.fc`
  compares the query against literals built in code
  (`store_slice("ton")`, and likewise for `t.me` and `www.ton`) and keeps only
  the three target addresses in storage. A `.tos` Root is therefore a
  source-level adaptation of that contract, not a data configuration of the TON
  Root. It is still not a *core* change: the Root is an ordinary masterchain
  contract maintained under `crypto/smartcont/dns/`.
- **The one upstream feature that would require a core change is ConfigParam 80**
  (§6.6). Launching `.tos` does not need it, and v1 does not depend on it.

Work is divided into two profiles so that inherited compatibility is not
confused with TOS policy:

| Profile | Purpose | Required repositories | Effect on `tos` core |
|---|---|---|---|
| **Baseline port** | Prove that the TON reference Root/Collection/Item model can register and resolve `.tos` on a TOS localnet | `tos` (`crypto/smartcont/dns/`), `TIP`, deployment configuration, one inspection client | No consensus or TVM change. Use existing generic configuration tooling where it is sufficient; add only missing parameter-4 deployment scripts or confirmed boundary fixes. |
| **TOS production profile** | Keep the upstream auction/lifecycle contract unchanged while adding code-hash publication, resolver provenance, safe lifecycle interpretation, and Agent-native integrations | Baseline components plus `domains/` (registrar), wallets, explorer, service and Messenger repositories | No consensus or TVM change. Client/API hardening is gated independently from contract parity. |

The baseline port is an engineering compatibility milestone, not a public
namespace or a promise of mainnet ownership. Production retains the inherited
open ascending auction. Any future auction redesign requires a separate TIP and
must not be smuggled into Agent-native or client work. Provenance results,
code-hash publication, and Agent-native records remain TOS integrations.

Repository ownership follows the same boundary:

- `crypto/smartcont/dns/` in `tosnetwork/tos` owns Root, Collection, Domain
  Item, auction, lifecycle, contract build artifacts, and contract tests;
- the rest of `tosnetwork/tos` owns only inherited platform primitives,
  parameter-4 activation support, generally useful resolver/API fixes, SDKs,
  and operator tooling; and
- `domains/` in `tosnetwork/tos` is a non-custodial application and never
  defines protocol bytes independently of the TIP and shared vector corpus.

## 4. Name Syntax and Canonicalization

### 4.1 Inherited registration labels and UI policy

The Collection keeps the upstream registration rule exactly. `check_domain_string`
(`func/dns-utils.fc`) accepts a byte only if it is `0`-`9`, `a`-`z`, or a hyphen
that is **neither the first nor the last byte** of the label:

```text
valid_char = (is_hyphen & (i > 0) & (i < len - 8))
           | ((char >= 48) & (char <= 57))
           | ((char >= 97) & (char <= 122))
```

Combined with the Collection's length and alignment checks, the on-chain rule is:
byte-aligned, `len > 3 * 8` and `len <= 126 * 8` (that is, 4 through 126 bytes),
lowercase ASCII alphanumeric or interior hyphen. So leading and trailing hyphens
**are** rejected by the contract. What the contract does *not* restrict is
consecutive interior hyphens and the `xn--` prefix, both of which register
normally. Registration software must not describe a stricter frontend convention
as an on-chain rule, nor refuse to resolve a name that the contract validly
registered.

For safer presentation, `tos-domains` and wallets should recommend labels no
longer than 63 bytes, warn on `xn--` and on consecutive hyphens, reserve
confusable Unicode, and show the exact lowercase label before signing. These
are UX protections only. A direct contract caller can register any label that
satisfies the inherited rule, so resolvers and indexers must support it subject
to the full-name encoding bound in Section 4.2.

For lookup, a UI may offer to lowercase human input after showing the
canonicalized result. Signed transactions and durable identifiers use the
exact on-chain label. Empty components, control characters, spaces, `/`, `:`,
NUL, and a trailing dot are rejected by public name-entry interfaces.

### 4.2 Internal representation and the exact length bound

Resolution retains the existing TOS/TON-compatible encoding: split the name on
dots, reverse the components, append a zero byte to every component, and
concatenate them (`DnsInterface::encode_name`,
`crypto/smc-envelope/ManualDns.cpp:625-643`).

```text
translate.alice.tos -> tos\0alice\0translate\0      (20 bytes)
```

**Encoded length is always the dotted length plus one.** A name with `L` labels
whose dotted form is `V` bytes contains `L-1` dots and `sum(label_len) = V-(L-1)`;
the encoded form replaces every dot with a NUL and appends one more, giving
`V-(L-1)+L = V+1` bytes regardless of how the labels are split. The label count
therefore does not affect the total, and the frequently repeated claim that
"each label adds a byte" is wrong.

The binding limit is on the encoded slice:

- `dnsresolve` receives the encoded name as the data bits of a single cell. A
  cell holds at most 1023 bits, so 127 bytes (1016 bits) fit and 128 bytes do
  not.
- Both clients enforce this explicitly:
  `DnsInterface::resolve_args_raw` rejects `encoded_name.size() > 127`
  (`ManualDns.cpp:199`), and `TestNode::dns_resolve_start` rejects
  `qdomain.size() > 127` (`lite-client.cpp:1834`).
- TEP-81 states the same bound (`n` at most 127) for the inherited ABI.

**Normative resolver rule:** the encoded name is at most **127 bytes**, so the
dotted name is at most **126 bytes**. For a direct second-level `.tos` name,
this means at most 122 label bytes (`label.tos` plus the final encoded NUL).
The upstream Collection nevertheless accepts labels up to 126 bytes. Frontends
must refuse to bid on labels that cannot be resolved from the configured root
and explain why; changing the Collection's inherited upper bound would be a
contract fork and is not part of v1.

Two boundary defects in the current clients must be fixed rather than
documented around; both belong to the `tos` row of §11:

- `DnsInterface::get_default_max_name_size()` returns **128**
  (`ManualDns.h:193-195`), and `resolve_args` / `resolve_raw_or_throw` gate on
  it (`ManualDns.cpp:212`, `ManualDns.cpp:512`). A 127- or 128-byte dotted name
  therefore passes the SDK's advertised limit and then fails deeper with
  `DNS encoded name is too long`. The constant must become 126.
- `TestNode::dns_resolve_finish` accepts a consumed-bit count only up to
  `8 * min(qdomain.size(), 126)` (`lite-client.cpp:1958`). A resolver that
  legitimately consumes all 1016 bits of a 127-byte slice in one hop is
  rejected as "too many bits used". The `.tos` chain never reaches this case
  because the root consumes `tos\0` first, but `dnsresolvestep` against a
  domain item can, so the bound must become `qdomain.size()`.

**Trailing dots are not neutral.** `encode_name("alice.tos.")` yields
`\0tos\0alice\0` — a *leading* NUL byte — which is the "resolve relative to this
resolver" form (equivalent to the Lite Client's `mode & 2`, `lite-client.cpp:1826`,
and to Toslib's forced trailing dot on the explicit-resolver path,
`toslib/toslib/ToslibClient.cpp:5551-5553`). `alice.tos` and `alice.tos.` are
therefore *different queries*, not two spellings of one name. v1 rejects a
trailing dot in user input and exposes the leading-NUL form only as an explicit
"resolve as a step against this resolver" flag in tooling.

Boundary vectors that the TIP corpus must contain verbatim:

| Case | Dotted input | Encoded bytes | Expected |
|---|---|---|---|
| Lookup-only short label | `a.tos` | `tos\0a\0` (6) | resolver accepts; Collection registration rejects length `< 4` |
| Maximum | any canonical name of exactly 126 bytes | 127 | accept |
| One over | any canonical name of exactly 127 bytes | 128 | reject, `encoded name too long` |
| Label count invariance | 126-byte name as 2 labels vs. 21 labels | 127 in both | accept both |
| Trailing dot | `alice.tos.` | `\0tos\0alice\0` (11) | reject at input |
| Empty label | `a..tos` | — | reject at input |
| Bare separator | `.` | `\0` (1) | reject at input; valid only as an internal self-query |
| Uppercase | `Alice.tos` | — | reject for registration/mutation; lowercase only for lookup |
| Non-ASCII | `älice.tos` | — | reject in v1 |
| `xn--` | `xn--80ak6aa92e.tos` | valid encoding | contract accepts; public UI warns/refuses by policy |
| Interior hyphen | `ab-c.tos` | `tos\0ab-c\0` (9) | contract accepts |
| Leading/trailing hyphen | `-abc.tos`, `abc-.tos` | valid encoding | **contract rejects** (`check_domain_string`, error 203); resolution of an existing name is unaffected |

The canonical wire bytes, category hashes, auction thresholds, deterministic
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

Configuration parameter 4 is `dns_root_addr:bits256`
(`crypto/block/block.tlb:640`). It carries **only 256 address bits**; the
workchain is not stored and every client hard-codes the masterchain
(`Config::get_dns_root_addr()` returns bare bits, and callers wrap it as
`block::StdAddress(masterchainId, addr)` —
`toslib/toslib/ToslibClient.cpp:5959`, `lite-client/lite-client.cpp:1843-1845`).
The root resolver must therefore live in the masterchain; a workchain-0 root is
unreachable through the existing ABI and would require a versioned parameter
change.

The root contract should do very little:

- implement `dnsresolve`;
- delegate `.tos` to the canonical `.tos` collection/resolver;
- optionally delegate future TOS-owned suffixes after governance approval; and
- expose version and code-hash getters for operational verification.

The root must not sell names or store user records.

**Four distinct change surfaces must not be conflated.** Each has a different
authority, blast radius, and required evidence:

| Change | Mechanism | Blast radius | Required gate |
|---|---|---|---|
| Parameter 4 value | masterchain config proposal | replaces the entire namespace root for every client | explicit mainnet governance decision below |
| Which Collection `tos\0` delegates to | **deploy a new Root and repoint parameter 4** — there is no in-place edit | replaces the `.tos` Collection for every name | announced activation block; audited Root and Collection code hashes; new item-address vectors |
| `.tos` Collection replacement | same as above; a live Collection cannot be upgraded | registrar policy and deterministic item address for names under the new Collection | upstream-parity report, migration plan, new address vectors, explicit governance activation |
| Domain item code in a Collection | immutable code cell used in StateInit | changing it changes every derived item address under that Collection | treat as a new Collection deployment; never mutate or silently substitute it |

**The upstream Root and Collection are immutable by construction, and TOS keeps
them that way.** `root-dns.fc` has an empty `recv_internal`, no `recv_external`,
no owner field, and no `set_code`; its storage holds only the delegated
resolver addresses, while the suffix literals are compiled into the code.
`nft-collection.fc` stores only `(collection_content, nft_item_code)`, handles
exactly `op == 0` (register) and `op::fill_up`, and throws `0xffff` on anything
else — it too has no owner, no admin operation, and no `set_code`. There is
therefore no "root data update" or "collection upgrade" surface at all.
Parameter 4 is the single lever over the whole namespace, which makes the
governance question below sharper rather than softer.

**Parameter 4 is currently an ordinary-vote parameter.** As shown in §3.1 it
appears in neither `mandatory_params` nor `critical_params`, so under the
genesis configuration a proposal that replaces the root resolver clears with
`min_tot_rounds=2, max_tot_rounds=3, min_wins=2, max_losses=2` — a weaker
threshold than the one protecting the config, elector, and fee-collector
addresses (parameters 0, 1, 3). Silently redirecting `.tos` is at least as
severe as changing the fee collector.

**Decision required before mainnet, not before the baseline port:** governance
must either add parameter 4 to `critical_params` (ConfigParam 10, itself a
critical parameter) before or as part of the announced activation sequence, or
publish and approve an alternative root-protection policy with an equivalent
threat analysis. The two configuration changes need not be falsely described
as one atomic proposal if the Config Contract cannot express that operation.
In every case, clients treat an absent parameter 4 as "DNS unavailable", never
as "name not found". Until a policy is selected and activated, this document
makes no claim that the root is governance-protected.

The upstream Collection has no emergency auction pause, and the immutability
above means none can be added to a deployed instance. Adding one to the source
would be a semantic fork and is outside v1.

The inherited Domain Item does implement `process_governance_decision`
(`op = 0x44beae41`): with no sender check at all, **any** caller may execute it,
and the authority comes entirely from an entry for the item's index in
ConfigParam 80 — `config_op = 0` transfers the domain to an address read from
the config entry, `config_op = 1` destroys the item with send mode `128 + 32`.
It additionally requires that no auction is active. §6.6 records why this path
is currently inert on TOS and what enabling it would cost. TOS must disclose
and govern it rather than claim it does not exist, and must not broaden it.
Incident response may protect frontend access and publish warnings, but cannot
invent additional mutation authority.

### 5.2 `.tos` collection and registrar

The `.tos` collection is a direct TOS deployment of the latest reviewed
`ton-blockchain/dns-contract` Collection contract. The in-tree port at
`crypto/smartcont/dns/` is pinned to upstream commit
`d08131031fb659d2826cccc417ddd9b98476f814`
("Merge pull request #2 from ton-blockchain/root2", 2022-10-30); upstream has
only the single branch `main` plus the tags `v1.0`, `root-v1.0`, and
`root-v1.1`. In the port, `nft-item.fc` and `nft-collection.fc` are
byte-identical to upstream; `root-dns.fc` is adapted to the single `.tos`
zone; and `dns-utils.fc` differs only by moving `auction_start_time` into
`tos-config.fc`. Before every release, the port must recompare against
upstream `main` and incorporate newer compatible fixes first.

The collection:

- implements the TOS-TEP-62 collection getters;
- validates a registration label and its minimum opening value;
- checks the optional reserved-name configuration used by the upstream
  contract, if TOS governance chooses and can represent that configuration;
- derives the Domain Item address;
- deploys the Domain Item with the caller as its first bidder; and
- implements `dnsresolve` by returning the derived item as
  `dns_next_resolver`.

It does not keep a second auction state machine, enumerate names, custody
refunds, or carry Agent-native authority.

#### Item index - preserve the upstream rule

The upstream collection computes:

```text
item_index = slice_hash(canonical_label_slice)
```

Here `slice_hash` is the TVM `HASHSU` representation hash of the cell that
holds the label slice. It is not a plain `SHA256(label_bytes)` digest. The TOS
contract, SDKs, wallets, explorer, and shared vectors must reproduce the
upstream operation exactly. Changing it would change every deterministic Domain
Item address and needlessly break compatibility.

The label slice contains the second-level label only, without `.tos` and
without a trailing NUL. The TIP corpus must include:

- the upstream `slice_hash` result for `alice`;
- a negative vector showing that plain SHA-256 yields a different value; and
- the resulting StateInit and Domain Item address.

#### Deterministic item address

The address follows the upstream layout without modification:

```text
item_data      = uint256(item_index) || MsgAddressInt(collection_address)
state_init     = split_depth:nothing  special:nothing
                 code:just(^domain_item_code)  data:just(^item_data)
                 library:nothing
item_address   = addr_std(anycast:nothing, workchain, cell_hash(state_init))
```

The exact item code cell, collection address, item index, workchain, and
StateInit layout determine the address. Mainnet/testnet separation comes from
different collection addresses and roots, not from adding a network identifier
to this derivation.

The upstream Collection stores one item-code cell and exposes
`get_nft_address_by_index(index)`. Clients derive locally with that same code
and require the getter to match. TOS v1 does not add per-item code-version
getters or silently move an existing label to a different item address. A
future item-code migration requires a separate TIP and explicit collection
migration plan.

#### TOS-TEP-62 DNS collection profile

The upstream collection intentionally returns:

```text
get_collection_data() -> (-1, collection_content, addr_none)
```

because item indices are hashes rather than a sequential mint counter. TOS keeps
that behavior. The TOS-TEP-62 DNS profile must document:

1. `next_item_index = -1` means a non-sequential collection;
2. `get_nft_address_by_index(uint256)` is the discovery getter;
3. wallets and explorers must not enumerate such a collection by incrementing
   an index;
4. JSON and SDK representations preserve the complete 256-bit signed/unsigned
   VM integer without narrowing it to `int64`; and
5. ownership and transfer remain NFT-compatible even though enumeration is not.

The existing TOS JSON-RPC narrowing bug is a generic tooling fix in Section 11,
not a reason to change the DNS contract index.

#### Allowed collection differences

A TOS port may change only values that cannot correctly remain TON-specific:

- collection metadata and public `.tos` presentation;
- the Root delegation from `ton\0` to `tos\0`;
- deployment addresses, workchain selection, and build artifacts;
- the launch timestamp and TOS-denominated economic constants approved for the
  deployment; and
- a reservation/configuration identifier only if TOS governance explicitly
  adopts that optional upstream feature.

Auction message formats, operation codes, item storage, item-index hashing,
bid ordering, refund behavior, duration/prolongation formulas, renewal, and
release semantics remain upstream-compatible. Any additional difference
requires a TIP, a source-level upstream diff, new vectors, and independent
review.

### 5.3 Domain NFT item

Each second-level name, such as `alice.tos`, is one permanent smart contract
using the upstream Domain Item storage and interfaces. It implements:

- the editable TOS-TEP-62 NFT item interface;
- the public ascending auction in Section 6;
- owner top-up/renewal and post-year release;
- DNS record set/delete operations;
- `dnsresolve`; and
- ownership transfer.

TOS v1 preserves the upstream getters:

```text
get_nft_data()
get_editor()
get_domain()
get_auction_info()
get_last_fill_up_time()
dnsresolve(slice remaining_name, int category)
```

It does not add `get_domain_state`, `domain_epoch`, `auction_round`, a grace
state, or a second settlement contract. Applications derive the renewal
deadline from `get_last_fill_up_time() + one_year` and determine whether an
auction is present through `get_auction_info()`.

The Domain Item is deployed once at its deterministic address. Initial
registration initializes it and starts its first auction. After a year without
renewal, `dns_balance_release` starts another auction in the same item, so an
ordinary change of owner never redeploys the contract.

Two boundary behaviours complete that statement:

- **A duplicate registration is refunded, not applied.** If the Collection
  deploys onto an item that is already initialized, the item takes the
  `init? & sender == collection_address` branch and returns the whole remaining
  message value to the would-be bidder (`send_msg(from_address, 0, 0, cur_lt(),
  null(), 64)`). It does not restart an auction or overwrite state.
- **Governance destruction is the one case where the address is reused.**
  `process_governance_decision` with `config_op = 1` sends mode `128 + 32`,
  which carries the balance out and deletes the account. The same ConfigParam 80
  entry still causes the Collection to reject registration with error 205, so
  governance must remove that entry before anyone can register the label again.
  Only then can the Collection re-create the item at the same address with fresh
  state and a fresh auction. Any indexer or wallet that assumes an item's history
  is continuous must handle account deletion, configuration removal, and later
  re-creation at the same address as distinct finalized events.

### 5.4 Subdomains - preserve TEP-81 behavior

The upstream Domain Item stores records for the domain itself. For a deeper
query it always requests the `dns_next_resolver` category, consumes the leading
NUL self separator, and returns the delegated resolver stored in that category.
It does not embed a prefix dictionary for arbitrary subdomain records.

Therefore the owner of `alice.tos` can:

- store records for `alice.tos`; and
- store one `dns_next_resolver` record that delegates the namespace beneath
  `alice.tos` to another TEP-81 resolver.

A separately deployed subresolver may implement records such as
`translate.alice.tos`, but it is not a tradable subdomain NFT in v1. Clients
must display that the parent owner can replace or remove the delegation.

**A delegated subresolver must accept a slice that does not begin with NUL.**
The Domain Item consumes exactly the eight bits of its own self separator and
returns `(8, next_resolver)`, so the delegated contract receives `translate\0`
with the separator already consumed. A subresolver copied from `nft-item.fc`
would reject it immediately (`throw_unless(413, starts_with_zero_byte)`).
`dns-manual-code.fc` is safe because it strips an optional leading NUL and
resolves either form (`crypto/smartcont/dns-manual-code.fc:312-317`). This is a
live porting trap, not a theoretical one, and belongs in the shared vectors.

TOS must not modify the Domain Item to serve bounded subdomain records directly
unless a later TIP demonstrates a need. Keeping the inherited eight-bit
self-query behavior avoids a new consumed-bit contract and preserves existing
Lite Client and Toslib recursion.

### 5.5 Worked upstream-compatible resolution trace

`translate.alice.tos` encodes to `tos\0alice\0translate\0`, 20 bytes /
160 bits. With the current upstream Root/Collection/Item behavior:

| Hop | Resolver and input | Consumed | Required result | Remainder |
|---|---|---:|---|---|
| 1 | Root: `tos\0alice\0translate\0` | 24 bits (`tos`, stopping before its separator) | `dns_next_resolver` to the `.tos` Collection | `\0alice\0translate\0` |
| 2 | Collection: `\0alice\0translate\0` | 48 bits (leading NUL plus `alice`, stopping before its separator) | `dns_next_resolver` to the locally derived Domain Item | `\0translate\0` |
| 3 | Domain Item: `\0translate\0` | 8 bits (self separator) | its stored `dns_next_resolver` | `translate\0` |
| 4 | delegated subresolver: `translate\0` | implementation-defined positive component-boundary prefix, ultimately full input | requested terminal record or another valid `dns_next_resolver` | strictly shorter after every partial hop |

For `alice.tos`, hops 1 and 2 leave the one-byte slice `\0`; the Domain Item
consumes 8 bits and returns its own requested record.

A conforming client preserves the inherited TEP-81 checks:

- the consumed count is byte-aligned, positive for a successful hop, and no
  larger than the input;
- a partial answer ends at a component boundary and decodes exactly as
  `dns_next_resolver`;
- the next resolver address is a valid `MsgAddressInt`;
- every remainder is strictly shorter; and
- malformed records or an absent next resolver fail closed.

The production clients may additionally impose the uniform hop budget, cycle
detection, and checkpoint rules in Section 8. Those are client hardening, not a
reason to change the upstream contract ABI.

## 6. TON-Compatible Auction, Renewal, and Release

### 6.1 Compatibility rule

The `.tos` auction and lifecycle contract must track the current official TON
DNS contract, rather than introduce a TOS-specific auction protocol. The
`.tos` port under `crypto/smartcont/dns/` is pinned to upstream
`ton-blockchain/dns-contract` commit
`d08131031fb659d2826cccc417ddd9b98476f814`. Before every DNS contract release,
CI must fetch the current upstream branch, produce a source and generated-code
parity report, and require an explicit review for every difference.

The default is zero semantic difference. In particular, v1 does not add a
commit-reveal phase, bid-vault contract, second-price settlement, auction-round
counter, domain-epoch counter, or renewal grace period. A later TIP may propose
one of these only as a separately versioned protocol with migration and
compatibility evidence; it is not part of this design.

Allowed differences are limited to changes required to deploy the same protocol
on TOS: the `.tos` suffix and presentation strings, TOS network addresses, the
initial auction timestamp, approved TOS-denominated economic constants, build
and deployment metadata, and integrations outside the contracts. Even these
differences require byte-exact tests and must not silently alter the state
machine.

### 6.2 Initial registration

Registration follows the upstream Collection contract:

1. The bidder sends an ordinary internal message in the standard text-comment
   shape — a 32-bit zero opcode followed by the plaintext label — whose value
   covers the minimum opening bid and gas. `read_domain_from_comment` walks the
   single-ref chain, so a long label may span cells.
2. The Collection requires `now() > auction_start_time` (strict), then checks
   `len > 3 * 8`, `len <= 126 * 8`, `mod(len, 8) == 0`, and
   `check_domain_string(label)` in that order (errors 199, 200, 201, 202, 203).
   This is the inherited contract rule; the stricter UI conventions in Section 4
   are not a different auction.
3. The Collection requires `msg_value >= get_min_price(len, now())` (error 204)
   — the length-tiered curve that decays 10% per 30-day month for at most 21
   months and is then flat.
4. It derives `item_index = slice_hash(label)` and, **only if ConfigParam 80
   exists**, rejects a listed index (error 205). On TOS that parameter cannot
   exist today, so this check silently passes for every label; see §6.6.
5. It deploys the deterministic Domain Item, passing the sender as first bidder
   and the attached value as the current maximum bid. The item sets its own
   `auction_end_time` and `last_fill_up_time` on receipt.

The label is public before inclusion, exactly as it is in TON DNS. The auction
is the price-discovery and anti-sniping mechanism; TOS does not claim that it
hides registration intent from block producers or observers.

### 6.3 Open ascending auction

The auction state is held inside the Domain Item and retains the upstream
rules:

- **Initial duration.** Only the *first* auction ramps. With
  `months = min(floor((now() - auction_start_time) / 2592000), 12)`, the
  duration is `604800 - (604800 - 3600) * months / 12`, which is exactly
  `604800 - 50100 * months` seconds — seven days falling to one hour in twelve
  discrete steps of thirty days each, so the ramp completes after 360 days, not
  a calendar year. A release re-auction (§6.5) never uses this curve; it is
  always seven days.
- **Bid threshold.** A replacement bid must satisfy
  `msg_value >= muldiv(max_bid_amount, 105, 100)` (error 407), i.e. at least
  `floor(max_bid * 105 / 100)`, compared inclusively.
- **Outbid refund is capped, and the cap is real.** The item sends
  `min(max_bid_amount, my_balance - min_tons_for_storage())` to the previous
  bidder as `op::outbid_notification` (`0x557cea20`) with **send mode 1**
  (fees paid separately, action errors revert the whole transaction). Because
  `min_tons_for_storage()` is `1e9` base units and `my_balance` already includes
  the incoming bid, the cap normally does not bind — but a client must not
  assume an outbid refund always equals the previous bid.
- **Every outgoing message is non-bounceable.** `send_msg` writes the header
  `0x10`, so nothing sent by the Domain Item can bounce back. There is no
  bounce-recovery path anywhere in this design, and none should be described.
- **Anti-sniping extension.** `delta_time = 3600 - (auction_end_time - now())`;
  if positive, `auction_end_time += delta_time`, so any bid leaves at least one
  hour on the clock.
- **Completion is strict.** `auction_complete = now() > auction_end_time`, so a
  bid arriving exactly at `auction_end_time` is still an ordinary bid.
- **First price.** The highest bidder wins and pays the full winning bid. This
  is not a second-price auction, and there is no bidder iteration.

Balance reservation, exact gas costs, and every other unstated detail are
defined by the pinned upstream FunC source and its tests, not by a
reimplementation from this prose.

### 6.4 Finalization

Auction finalization is lazy and has no dedicated operation. After
`auction_end_time`, the item runs the finalization block *before* dispatching
whatever operation arrived: it forwards
`min(max_bid_amount, (my_balance - msg_value) - min_tons_for_storage())` to the
Collection as `op::fill_up` with **send mode 2** (errors ignored), installs
`max_bid_address` as owner, and clears the auction cell. There is no separate
settlement contract and no loop over bidders.

**Not every message can finalize, and a wrong one silently cannot.** Three
constraints decide this:

- the item reads a 64-bit `query_id` before the finalization block, so any
  non-zero-op body lacking it throws;
- finalization and the requested operation share one transaction, so an
  operation that throws — including any unrecognized opcode, which falls
  through to `throw(0xffff)` — rolls the finalization back with it; and
- an empty body (`op == 0`) never finalizes. It takes the bid/top-up branch,
  where a completed auction demands `sender == owner_address`; the owner is
  still `addr_none` at that moment, so a plain transfer after the auction ends
  always fails with error 406.

The one inherited operation that any address can send and that always completes
is **`op::get_static_data()` (`0x2fcb26a2`)**. It carries a `query_id`, has no
sender check, and its only effect is a `report_static_data` reply with mode 64.
Notably it does **not** refresh `last_fill_up_time`, so using it to finalize
does not extend the renewal deadline. Registrars and CLIs must implement
"finish auction" as exactly this message; the owner-gated operations
(`transfer`, `edit_content`, `change_dns_record`) also finalize, but only for
the winner, and only because finalization installs them as owner first.

Clients must therefore distinguish "auction end time has passed" from "the
finalization transaction has executed", and must read `get_auction_info()`
rather than infer ownership from elapsed time.

### 6.5 Renewal and release

Ownership is kept alive by funding the Domain Item, using the inherited
`last_fill_up_time` lifecycle:

- initial deployment sets it when the first auction starts;
- **every accepted bid refreshes it**, so during an auction the clock tracks the
  last bid rather than the deployment;
- an owner top-up (`op == 0` from the owner, after the auction) refreshes it;
- a successful transfer, `edit_content`, `change_dns_record`, or a release
  refreshes it;
- auction finalization **preserves** the existing value rather than resetting it
  to the win time, and the permissionless finish operation of §6.4 does not
  refresh it either;
- `one_year` is the upstream constant `31,622,400` seconds (366 days), compared
  strictly (`now() - last_fill_up_time > one_year`); and
- there is no grace period.

The consequence is worth stating plainly: **a winner's first term is not 366
days from winning.** It runs from the last bid in the auction that they won, and
if they take no action afterwards the item becomes releasable that much sooner.
Wallets and registrars must compute the deadline from
`get_last_fill_up_time() + 31,622,400` and surface it, never from the purchase
date.

Once `now() - last_fill_up_time > one_year` and no auction is active, anyone may
invoke the inherited balance-release operation with at least the current
minimum price. The item refunds its old releasable balance to the former owner,
clears the owner, and immediately starts a new seven-day open auction with the
caller as its first bidder. Re-auction uses the same permanent Domain Item
address; the item is not redeployed.

Release requires `cell_null?(auction)`, so a name already under auction cannot
be released again, and the refund to the former owner is
`(my_balance - msg_value) - min_tons_for_storage()` sent with **send mode 2**
(errors ignored, non-bounceable): if that send fails, the value stays in the
item and the former owner receives nothing.

**Records survive both release and the change of owner.** Release and
finalization both write the existing `content` cell straight back, and the item
has no path that clears the record dictionary. A new owner therefore inherits
the previous owner's records — including a `dns_next_resolver` delegation to a
resolver the previous owner still controls — and they keep resolving until the
new owner overwrites each one. Registrars must prompt a new owner to review and
replace inherited records as the first post-purchase step.

The raw `dnsresolve` getter does not independently suppress records merely
because the renewal time has elapsed, an auction is active, or ownership is
`addr_none`. There is no `expires_at` getter and no on-chain suppression.
Consequently, wallets, gateways, Agent software, payment flows, and other
security-sensitive consumers must inspect `get_auction_info()` and
`get_last_fill_up_time()` and fail closed for an active auction or an overdue
item. This is a required client safety policy, not a reason to fork the
contract state machine. A later upstream fix should be evaluated and merged
through the parity process in Section 6.1.

### 6.6 Deployment parameters and upstream updates

The inherited minimum-price curve and auction formulas stay unchanged unless
TON upstream changes them. Two deployment values nevertheless force an explicit
decision.

**Launch timestamp.** Upstream hardcodes `auction_start_time = 1659171600`
(30 July 2022). The two available choices are not equivalent and this document
does not pick one:

| Option | Effect on a `.tos` launch today |
|---|---|
| Keep the upstream timestamp | `months` saturates at 12 and 21, so every first auction opens already at the one-hour minimum duration and every label opens at the fully decayed floor price. Anti-sniping and price discovery still work, but there is no extended launch ramp. |
| Use a TOS activation timestamp | The upstream algorithm is preserved exactly and its seven-day-to-one-hour ramp and 21-month price decay restart from `.tos` launch. Because the timestamp is a compile-time constant in `dns-utils.fc`, this is an explicit source/code change that produces a different code hash, not a data-only deployment choice. |

**Unit scale is already identical; do not confuse it with an economic change.**
Upstream `one_ton` and `min_tons_for_storage()` are both `1e9` base units, and
TOS `Tomi` is also `1000000000` (`crypto/fift/lib/Currency.fif:5`). Renaming the
constants is cosmetic. Only the *magnitudes* — the 1000/500/400/300/200/100/50/10
tier table and the one-unit storage reserve — are economic decisions requiring
governance approval, published source diffs, reproducible code hashes, and test
vectors at boundary times and amounts.

**Auction proceeds have no withdrawal path.** Finalization forwards the winning
bid to the Collection as `op::fill_up`, and the upstream Collection handles that
opcode by doing nothing at all (`if (op == op::fill_up) { return (); }`). It has
no owner, no admin operation, no withdrawal, and no `set_code`. Proceeds
therefore accumulate in the Collection's balance permanently — an implicit burn,
not a treasury. TOS must either accept that explicitly as the v1 economic
outcome, or acknowledge that adding a proceeds destination is a semantic fork of
the Collection requiring its own TIP, review, and vectors. A gate that merely
says "proceeds destination approved" without resolving this is not satisfied.

Upstream synchronization is continuous, not a one-time port:

1. monitor official `ton-blockchain/dns-contract` releases and commits;
2. classify each upstream change as security, compatibility, economics, or
   tooling;
3. reproduce its upstream tests and add TOS-network deployment tests;
4. merge semantic fixes by default unless a documented TOS constraint prevents
   them; and
5. publish the remaining diff and deployed code hashes before activation.

**ConfigParam 80 is more than a registration blacklist, and on TOS it cannot
currently exist.** Upstream uses `dns_config_id = 80` in production and `-80` on
testnet. Both effects hang off it: the Collection rejects a new label whose item
index is listed, and any caller can drive an existing Domain Item's
`process_governance_decision` to transfer or destroy it (§5.1).

In the current TOS tree, index 80 is unassigned — `block.tlb` declares
`JettonBridgeParams` at 79, 81, 82, and 83 and skips 80 — and the generated
validator refuses it: `ConfigParam::get_tag` has `case 79` and `case 81` with
`default: return -1` (`crypto/block/block-auto.cpp:19592-19609`), so `validate_skip`
returns false, `check_one_config_param` fails
(`crypto/block/block.cpp:1865-1883`), and `valid_config_data` rejects the entire
configuration. A proposal that sets parameter 80 would therefore be refused by
collation and validation.

The practical consequences, which the design must not blur:

- **Today the mechanism is inert.** `config_param(80)` returns null, so the
  Collection's blacklist check is skipped for every label, and
  `process_governance_decision` throws on `null.begin_parse()`. Launching `.tos`
  neither needs nor gains this feature.
- **Enabling it at index 80 is a TOS core change** — a `block.tlb` declaration
  plus regenerated `block-auto.{h,cpp}` that every validator must run. That is
  precisely the class of change §3.2 says `.tos` does not otherwise require.
- **A negative index avoids the core change.** `check_one_config_param` returns
  true unconditionally for `idx < 0`, which is why upstream's testnet build uses
  `-80`. Adopting a negative `dns_config_id` would make the feature available
  with no TL-B or validator change, at the cost of an unvalidated parameter
  shape.

Choosing among *no blacklist* (v1 default), *negative index*, and *core-declared
index 80* is a governance decision that must be recorded before activation, not
left implicit. Whichever is chosen, tests, wallet warnings, TOSCan, and
governance runbooks must cover both effects, and TOS must not remove or broaden
the path silently; any policy change is a reviewed contract diff and a
governance decision.

No application repository may redefine bid thresholds, auction timing, renewal
deadlines, or item-state interpretation. They consume the contract ABI and the
shared conformance vectors.

## 7. Record Categories

Categories remain `sha256(UTF-8 category name)`. Category zero is not a category
at all: it requests the **complete record dictionary**, returned as
`HashmapE 256 ^DNSRecord` (`crypto/block/block.tlb:980`), which the client walks
itself (`lite-client.cpp:2014-2032`, `ManualDns.cpp:539-556`). Version 1 reuses
the existing TL-B record encodings; new semantics are selected by category
instead of inventing unnecessary wire types.

The **Status** column distinguishes what is already fixed by TOS code from what
this document proposes. Only two category hashes exist in the tree today (§3.1);
every `Proposed` row is a name whose SHA-256 value the TIP must freeze together
with a vector, because once a record is stored under a hash the name that
produced it is unrecoverable.

| Category name | Status | Record encoding | TOS meaning | Required follow-up verification |
|---|---|---|---|---|
| `dns_next_resolver` | **Pinned in code** (`ManualDns.h:34`, `dns-manual-code.fc:347`, `dns-auto-code.fc:480`) | `dns_next_resolver` | Delegated subdomain resolver | Resolver address, cycle, depth, response validation; valid **only** on partial resolution (§5.5) |
| `site` | **Pinned in code** (`DNSResolver.cpp:69`) | `dns_adnl_address` | TOS Site entry point | ADNL identity and protocol support |
| `wallet` | Proposed | `dns_smc_address` | Default payment account | Address/network validation and wallet transaction confirmation |
| `agent` | Proposed | `dns_smc_address` | Finalized Agent account | Address→Agent ID re-derivation (below), live state, controller policy, revocation |
| `capability` | Proposed | `dns_smc_address` | Finalized Capability account | Address→Capability ID re-derivation, owner Agent, version, revocation/tombstone, manifest, policy |
| `messenger` | Proposed | `dns_smc_address` | Agent used for Messaging contact | Address→Agent ID re-derivation, then Endpoint delegation, Contact Descriptor, and DHT locator (§9.2) |
| `storage` | Proposed | `dns_storage_address` | TOS Storage Bag ID | Bag hash/content verification and application policy |
| `text` | Proposed | `dns_text` | Human-readable presentation metadata | Never authoritative; escape before display |

**Name collisions with inherited conventions.** `dns_next_resolver` and `site`
carry their inherited meanings unchanged and must not be redefined.
`wallet`, `storage`, and `text` are inherited *conventions* that no TOS code
implements yet, so TOS is free to define them but must not assume a TON-derived
tool already agrees. `agent`, `capability`, and `messenger` are new to TOS and
have no inherited meaning.

**Strict category-to-record-type checking, failing closed.** A decoder must
reject any record whose TL-B tag is not the one this table assigns to the
requested category — a `dns_adnl_address` returned for `agent`, or a `dns_text`
returned for `wallet`, is a hard failure, never a best-effort interpretation.
Trailing data after a decoded record is likewise a failure. Under category zero
the same rule applies per entry: an entry with an unknown category hash is
ignored, but an entry whose record does not decode as a `DNSRecord` invalidates
the answer. `EntryData::from_cellslice` already fails closed on an unknown tag
(`ManualDns.cpp:131-195`), and every other implementation must match.

### 7.1 Agent, Capability, and Messenger records

`dns_smc_address` is safe for these three categories, because in the actual
Native model both Agents and Capabilities **do** have dedicated deterministic
account addresses: the address is derived from the `StateInit` holding the
reviewed registry code cell and a data cell carrying the network, object kind,
and object ID (`tos-service-spec/docs/NATIVE_IDENTIFIERS_V1.md`, "Deterministic
account"; `tos-service-protocol/pkg/nativecore/locator.go:55-74`). No record
semantics need to be invented.

Two properties of that derivation drive mandatory follow-up checks:

- **The durable identity is the object ID, not the address.** Agents are
  `agent_<sha256 hex>` and Capabilities are `cap_<sha256 hex>`
  (`NATIVE_IDENTIFIERS_V1.md`), and every Native and Messenger interface is
  keyed by that ID — for example `resolver.ResolveAgent(delegation.AgentID)`
  in `tos-messenger/pkg/identity/delegation.go:272`. A consumer must therefore
  read the account at the resolved address, decode its typed state, recover the
  object ID, and **re-derive the address from that ID**, accepting the record
  only if the derivation reproduces it. Resolving a name to an address and
  feeding that address onward is not sufficient.
- **The address is registry-code-version specific.** Because the registry code
  cell is inside the `StateInit`, a registry code upgrade changes the address of
  every object. A `dns_smc_address` record written before such an upgrade points
  at the old account. A consumer must check the code hash against the currently
  accepted registry code and treat a mismatch as "record stale", not as
  "Agent revoked" and not as a reason to trust the old account. The TIP must
  state whether `.tos` records are expected to be rewritten on a registry
  migration, or whether an ID-carrying record type is introduced in v2; v1 does
  not answer this and must not pretend to.

An `agent`, `capability`, or `messenger` record stores an account address, not
an HTTPS URL or mutable private endpoint. A service name normally points to a
Capability through the `capability` category. Multiple services should use
separate subdomains rather than an unbounded list inside one record.

Nothing mutable is duplicated into DNS. Contact Descriptors, prekey sets,
Capability manifests, quote policy, rate cards, and service endpoints stay in
their authoritative locations; DNS holds only the address that leads to them.

Future typed record schemas require a TIP, new positive and adversarial
vectors, and at least two independent decoders before being marked stable.

## 8. Resolution Algorithm and Result Provenance

A conforming resolver performs these steps:

1. parse and canonicalize the name under the v1 rules;
2. obtain a finalized masterchain block and configuration parameter 4;
3. encode the name in reverse zero-delimited form;
4. call `dnsresolve` on the root at that same finalized block;
5. validate the consumed-bit count and component boundary;
6. if partially resolved, require a valid `dns_next_resolver`, detect cycles by
   `(resolver_address, remaining_slice)`, and continue with the unresolved
   suffix;
7. stop after at most eight resolver hops;
8. decode only the requested category and reject trailing or malformed data;
9. confirm that the domain is active at the block time; and
10. return both the value and its provenance.

**Step 7 is now implemented uniformly.** The shared constant
`tos::DNS_MAX_RESOLVER_HOPS = 8` (`crypto/smc-envelope/ManualDns.h`) bounds
every client: the Lite Client threads it through `dns_resolve_send/finish`,
Toslib clamps every caller-supplied ttl to it, `toslib-cli` and
`rldp-http-proxy` pass it directly (historically these were 10, 16,
caller-supplied, and absent). Exhausting the budget is reported as a distinct
error, never as "not found", so an operator can tell a misconfigured
delegation from a missing name. Cycle safety follows from the budget plus the
progress rule: every accepted hop consumes at least one byte of the encoded
name, so a delegation loop terminates as a budget error.

The structured result exposed by SDKs and APIs must include:

```text
canonical_name
category_name and category_hash
record_type and decoded value
root_resolver_address
resolver_path[]
masterchain_block_id / seqno / root hash
domain_item_address
auction_active, max_bidder, max_bid, auction_end_time
last_fill_up_time and derived renewal_deadline
resolved_at_chain_time
provenance_class            (see the table below)
```

Delivered so far: `dns.resolved` in the toslib API now carries the pinned
`block_id`, the ordered `resolver_path`, and a `provenance_class`
(`chain_anchored`), and both `toslib-cli` and the Lite Client print the
pinned block, hop count, and resolver path; the JS SDK's resolver returns
the same structure with `provenance_class = "evaluated"`. The remaining
fields (canonical name, category names, decoded lifecycle/auction state,
chain time) are still assembled by callers from the underlying get-methods;
completing the single structured result stays tracked in the `tos` and
`tos-service-protocol` rows of §11.

Clients resolving Agent-native objects then perform their protocol-specific
finalized-state verification. They must use the account address or object ID,
not the input name, in signatures, nonces, Accepted Quotes, Events, receipts,
and durable journals.

### 8.1 What each reader can actually prove

Four different assurances are routinely conflated. They are not interchangeable,
and a structured result must name which one it carries:

| `provenance_class` | What it means | Who provides it today |
|---|---|---|
| `evaluated` | A get-method was executed against a named block. Says nothing about whether that block's state is genuine. | any `runGetMethod` caller that ignores the returned proof |
| `state_proved` | The account state was verified by Merkle proof against the named block, and the get-method was re-executed locally over the proved state. | **Lite Client** and **Toslib** |
| `chain_anchored` | `state_proved`, plus the named block was reached through a verified block-proof chain from a trusted init block. | **Toslib** |
| `quorum_agreed` | A strict majority of independent RPC endpoints returned the same value. No cryptographic proof. | **`tos-service-protocol`**, **Gateway** |

The distinctions are verifiable in the tree:

- The Lite Client calls `block::AccountState::validate(ref_blk, addr)` before
  running the method (`lite-client.cpp:2233`), which performs
  `check_shard_proof` and `check_account_proof`
  (`crypto/block/check-proof.cpp:224-237`), and then re-executes `dnsresolve`
  in a local VM over the proved state (`lite-client.cpp:2319-2327`). That is a real
  Merkle proof.
- Toslib does the same (`ToslibClient.cpp:1501`) and additionally maintains a
  verified `BlockProofChain` from its init block (`toslib/toslib/LastBlock.cpp`),
  which is what raises it to `chain_anchored`.
- `tos-service-protocol/pkg/toschain` reads over JSON-RPC and agrees by strict
  endpoint majority (`pkg/toschain/quorum.go`, `adapter.go:56`). The word
  `proof` does not occur in the package. It is honest agreement among readers;
  it is **not** a proof, and this document must never describe it as one.
- The Gateway is a transport. It can forward and cache a `quorum_agreed`
  result; it cannot raise its class.

**Anchoring across hops.** All hops of one lookup must be evaluated against one
masterchain checkpoint and the shard states it commits to. The Lite Client
threads a single `blkid` through every recursion, and Toslib now latches the
whole chain to the block the first hop ran at (`finish_dns_resolve` pins
`block_id` once and every later hop reuses it), reporting that block in
`dns.resolved.block_id`. A lookup can no longer straddle two checkpoints.

### 8.2 Caching

Positive cache lifetime is bounded by the earliest of:

- the derived renewal deadline and auction state;
- a record-specific validity limit, if a future record defines one;
- the current finalized-state refresh policy; and
- a conservative client maximum.

Negative results use a short bounded cache. A reorg or a change in the trusted
finalized checkpoint invalidates affected cache entries. Gateways may cache
resolution results but never become the source of authority.

Invalidation must be driven by events, not only by elapsed time. Each of these
invalidates every entry derived from the affected subtree:

| Event | Invalidates |
|---|---|
| auction starts, ends, or is finalized | that name and all its subdomains |
| `last_fill_up_time` changes or the renewal deadline passes | that name and all its subdomains |
| record set or delegation mutated | that name and, for a delegation change, its subtree |
| parameter 4 or a root delegation changed | the entire cache |
| trusted finalized checkpoint moved backwards (reorg) | every entry resolved at or after the abandoned checkpoint |

**Current state of the one cache that exists.** `rldp-http-proxy/DNSResolver`
is bounded at 1024 entries with expired-first eviction. It identifies the
Domain Item as the third canonical `.tos` resolver hop (never blindly as the
last hop, which may be an owner-controlled delegate), verifies through
`get_nft_data` that the item belongs to the Collection in the preceding hop,
and loads that item with `withBlock` at `dns.resolved.block_id`. It then reads
the item's collection and `uint256` index from `get_nft_data`, requiring them
to match the preceding Collection and the canonical second-level label's
`slice_hash`, before reading `get_auction_info` and `get_last_fill_up_time`.
It fails closed on auctioning, unfinalized, overdue, malformed, or unverifiable
names. Every successful
`smc.load` is paired with exactly one `smc.forget`, including all asynchronous
error exits. Cache lifetime is capped at the derived renewal deadline and an
unhealthy refresh immediately evicts the old entry.

Equal in-flight lookups are coalesced; the proxy retains at most 256 distinct
host lookups and 64 waiting callers per host. It does not yet learn of record
updates or checkpoint changes mid-lifetime beyond its base 300-second expiry;
record-update and reorg-driven invalidation remain open work in the `tos` row
of §11.

### 8.3 Reverse lookup

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
  -> `messenger` record -> Agent account address
  -> decode finalized typed state -> agent_<sha256 hex>
  -> re-derive the account address from that ID and require a match   (§7.1)
  -> finalized Agent policy and delegation digest
  -> signed, expiring DHT locator
  -> content-addressed Contact Descriptor
  -> current Messaging Endpoint and prekey generation
```

The third and fourth lines are the step that must not be skipped. Messenger's
delegation verifier is keyed by Agent ID, not by address —
`identity.Delegation.AgentID` is checked against `AgentPattern` and resolved via
`resolver.ResolveAgent(delegation.AgentID)`
(`tos-messenger/pkg/identity/delegation.go:272, 357`) — so a name resolution
that stops at an address has not yet produced anything Messenger can consume.
Recovering the ID from state and re-deriving the address is what prevents a
record from pointing at an account that merely *looks* like a registry object.

DNS must not contain prekeys, private contact graphs, Mailbox bearer tokens, or
long-lived endpoint URLs. In particular the signed Contact Card already carries
exactly one endpoint plus a bounded expiry
(`tos-service-protocol/pkg/agentpacket/contact.go:20-48`); DNS must not
duplicate that field, because a stale DNS copy would outlive the signature's
expiry. A Messenger Event is signed and journaled against the resolved Agent and
Endpoint identities, never against the string `chat.alice.tos`.

### 9.3 TOS Sites and Storage

Existing `site` and `storage` categories remain compatible with the current
TL-B records. `rldp-http-proxy` should accept `.tos` only after it implements
the finalized-root, hop-limit, cycle, lifecycle, and record-validation rules in
this document.

## 10. Security Requirements

Implementations must address at least these threats:

- **observable bidding and front-running:** registration labels and bids are
  public, as in TON DNS; clients must show the current bid, minimum increment,
  and end time and must never promise sealed-bid confidentiality;
- **homographs:** the contract is lowercase ASCII and rejects edge hyphens, but
  permits `xn--` and consecutive interior hyphens; public UIs apply and clearly
  label the stricter presentation policy in §4.1;
- **network confusion:** caches and UI confirmations are bound
  to the network domain tuple;
- **malicious resolvers:** strict consumed-bit, component-boundary, schema,
  hop-limit, and cycle checks;
- **stale authority:** Agent, Capability, and Messaging state is re-read from a
  finalized checkpoint after DNS resolution;
- **overdue or auctioning records:** because the upstream raw getter can retain
  records, security-sensitive clients derive lifecycle state from
  `get_auction_info()` and `get_last_fill_up_time()` and fail closed (§6.5);
- **record substitution:** mutations require the current NFT owner and are
  reflected in finalized state;
- **governance seizure:** ConfigParam 80 can block registration and authorize
  an inherited transfer/destruction operation for an existing item; clients
  disclose this power and governance limits it under the published policy;
- **upgrade substitution:** *required policy, not a required consensus
  feature.* TOS has no DNS-specific code-hash registry (§3.1), and it does not
  need to generalize the AIPoW-specific ConfigParam 93 merely to ship DNS.
  Before activation the TIP and the `crypto/smartcont/dns/` release manifest must pin and
  version the reproducible code hashes for the root, collection, and item and
  define client acceptance rules. Because item code participates in StateInit,
  a code change requires a new Collection and an explicit migration plan. An
  on-chain allowlist is one governance option, not assumed by this document;
  immutable code/data, an authenticated collection policy, or client release
  manifests may satisfy the same threat model if the TIP states their trust and
  upgrade properties explicitly;
- **gateway deception:** clients can reproduce resolution from chain state and
  receive provenance with cached API results;
- **display injection:** text records are untrusted UTF-8 presentation data;
  and
- **payment mistakes:** wallets show the resolved raw address, network,
  checkpoint age, auction state, and derived renewal deadline before signing.

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
| `tosnetwork/tos` (`crypto/smartcont/dns/`, **ported from `ton-blockchain/dns-contract`**) | Track the latest official Root, Collection, Domain Item, auction/renewal logic, and tests. Make only reviewed deployment adaptations for `.tos`, TOS addresses, launch time, and approved TOS-denominated constants; continuously report the remaining upstream diff | Upstream-parity CI; deterministic builds and published code hashes; exact 105% bid, one-hour extension, refund, lazy-finalization, and 366-day release tests; local multi-validator lifecycle evidence |
| `tosnetwork/tos` (confirmed generic fixes) | Fix independently reproducible inherited defects: set `get_default_max_name_size()` to 126 (`ManualDns.h:193`); correct the `min(qdomain.size(), 126)` consumed-bit cap (`lite-client.cpp:1958`); make `getTokenData` `uint256`-safe for hashed indices (`json-rpc-server-token.cpp:318, 368`). These are generic correctness fixes, not evidence that `.tos` requires a consensus fork | Boundary vectors from §4.2 passing in C++; a JSON-RPC test asserting a full 256-bit index round-trips as a decimal string; no consensus-state change |
| `tosnetwork/tos` (production-profile client hardening) | For production clients, add a uniform eight-hop limit and cycle detection to Lite Client, Toslib, `toslib-cli`, and `rldp-http-proxy` (§8); pin every hop to one checkpoint (`ToslibClient.cpp:5468`); bound the proxy cache and make it auction/renewal-aware (§8.2); emit the structured provenance result of §8 | Each change has an independent test and can land without the DNS contracts; hop/cycle, lifecycle cache invalidation, and checkpoint consistency evidence |
| `tosnetwork/tos` (configuration and activation) | First prove whether existing generic genesis and Config Contract proposal tooling can set parameter 4. Both activation paths are rehearsed by `scripts/dns-e2e.py`: genesis (`config.dns_root_smc!`, the commented `gen-zerostate.fif` example, the tostester `dns_root_addr` profile) and an ordinary config-change proposal accepted by validator vote on a running localnet. One operational finding stands: the default ConfigVotingSetup (ConfigParam 11) requires multi-round validator-set rotation, so localnet rehearsals opt into a relaxed override. Treat adding parameter 4 to `critical_params` as an explicit mainnet governance decision, not a prerequisite for the baseline port | A localnet booted with parameter 4 set from genesis and, if governance activation is selected, one where parameter 4 is introduced by proposal; evidence that clients fail closed while it is absent; a recorded decision on critical-parameter policy |
| `tosnetwork/tos` (`sdk/js`) | TypeScript resolver, canonicalization, category hashes, item-address derivation, and `uint256` index handling; align `NftCollection.ts` with the TOS-TEP-62 DNS profile | Consumption of the shared vectors in TypeScript; parity with the Go and C++ implementations |
| `tosnetwork/tos` (`tosctl`) | Add `domain normalize`, `bid`, `auction`, `finish` (sending `op::get_static_data`, per §6.4), `renew/top-up`, `release`, `transfer`, `record set/delete`, `delegate`, `resolve`, and `inspect`, using the inherited message formats with offline signing support | CLI golden vectors, exact bid-boundary and lifecycle interpretation tests, restart-safe transaction tracking, hardware/offline signer tests, and real localnet lifecycle |
| `tosnetwork/tos-service-spec` | Specify how `.tos` aliases may identify Agent, Capability, and Messenger entry points without changing `tos_service_v1` authority | Normative boundary text, negative cases, and shared vectors; no alternate registry semantics |
| `tosnetwork/tos-service-protocol` | Add a Go resolver/verifier library that consumes finalized TOS state, reproduces canonical encoding and category hashes, returns provenance, checks auction/renewal state, and resolves an alias to an address and then to a Native object ID with the §7.1 re-derivation before existing verification; extend `api/tos/service/v1/native.proto` and regenerate `gen/` for any resolution response crossing the Connect boundary | Cross-language vector parity; results labelled `quorum_agreed`, never `proof`; cycle/lifecycle/reorg tests; generated-code and API-compatibility checks |
| `tosnetwork/tos-service-gateway` | Expose bounded read-only resolution and verified aliases in discovery results; cache only within checkpoint and auction/renewal bounds | Gateway restart/cache tests and proof that a Gateway cannot create or mutate name or Native authority |
| `tosnetwork/tos-messenger` | Accept `.tos` contact input, reject auctioning or overdue items, resolve `messenger` to an Agent, then execute the existing finalized delegation -> DHT locator -> Contact Descriptor checks; persist IDs rather than names | Substitution, stale delegation, name transfer, re-auction/overdue state, DHT rotation, and three-transport replay tests |
| `tosnetwork/openfox` | Add name input/display at the human boundary while binding sessions, policy, purchases, and execution to resolved Agent/Capability IDs | Name-transfer and stale-cache tests proving no session or purchase authority follows an old alias |
| `tosnetwork/toscan` | Index domain NFTs, auctions, top-ups, records, transfers, releases, and re-auctions; provide forward-confirmed reverse lookup and domain pages | Reorg-safe index tests, raw-address display, checkpoint provenance, and localnet lifecycle coverage |
| `tosnetwork/ios` | Resolve names for send/contact flows and manage domain NFTs with explicit address/network/auction/renewal confirmation | Unit, UI, signer, and testnet lifecycle tests |
| `tosnetwork/android` | Match the iOS resolver, send protection, and domain-management behavior without trusting inherited TON APIs | Cross-platform vectors, UI tests, and TOS-native API boundary tests |
| `tosnetwork/tos` (`domains/`, **established**) | Provide the public registrar and management web application; use wallet signing and chain APIs without holding owner keys, and consume the upstream-compatible ABI and shared vectors rather than redefining auction rules in the frontend | Public bid/refund/finalization recovery UX, transaction-state recovery, overdue-name warnings, phishing defenses, CSP/security review, shared-vector parity, and testnet acceptance |
| `tosnetwork/toscan` (lifecycle indexing) | Keep ownership and record history by Domain Item address plus transaction logical time, and expose active auction, last top-up, derived renewal deadline, release, and re-auction transitions | An index test that owns a name, lets it become releasable, re-auctions it to a different owner at the same address, and keeps historical and current rows separate |
| shared vector corpus (owned by `tosnetwork/TIP`, consumed everywhere) | Publish one versioned corpus covering §4.2 boundaries, category hashes, `slice_hash` item-index and item-address derivation, bid thresholds, auction durations, renewal deadlines, and the adversarial cases of §13 | The corpus is consumed unmodified by C++, Go, Swift, Kotlin, and TypeScript, and a corpus change fails every consumer build until re-reviewed |
| `tosnetwork/doc` | Maintain this architecture, operator runbooks, category registry links, deployment addresses, and code hashes | Documentation review tied to released commits and deployed network parameters |
| `tosnetwork/docs` | Publish end-user and integrator documentation for `.tos`, including public-auction behavior, renewal/release, wallet confirmation, and raw-address/network/lifecycle warnings | Published pages tied to a released commit; no page describes an unshipped capability as available |

### 11.1 Repositories that should not gain authority

- `tos-ai` may accept already resolved Agent/Capability identities but must not
  perform name-based execution authorization inside the runner. Execution stays
  bound to the Accepted Quote and the object IDs it names; a `.tos` string must
  never reach an authorization decision, a sandbox policy, or an evidence
  record.
- `freecity` may display verified aliases but remains a replaceable projection.
- `tos-homepage` may link to the registrar only after deployment; it must not
  host registrar keys or claim unshipped functionality.
- wallets, explorers, Gateways, Messenger Relays, and DHT nodes never own the
  namespace or override finalized chain state.

## 12. Delivery Plan

### Phase 0 — upstream parity and specification freeze

1. Track the official TON repository as the upstream of
   `crypto/smartcont/dns/` and add CI that reports commit, source,
   generated-code, and ABI differences.
2. Prove parameter-4 activation through existing generic TOS tooling before
   adding specialized Core helpers.
3. Publish the TIP, category registry, exact inherited auction/lifecycle rules,
   and the small allowlist of TOS deployment differences.
4. Freeze normalization, encoding, category hashes, `slice_hash` item index,
   StateInit, operation codes, bid/duration boundary vectors, and code hashes.
5. Approve the `.tos` launch timestamp, TOS-denominated price constants,
   proceeds handling, optional reservation policy, and upgrade governance.
6. Complete threat modeling and independent contract review.

### Phase 1 — contracts and local tooling

1. Deploy the minimally adapted Root, Collection, and permanent Domain Item;
   do not add an auction helper or Bid Vault contract.
2. Reproduce upstream tests, then add TOS tests for every approved diff and for
   client-side overdue/auction fail-closed behavior.
3. Add `tosctl`, Lite Client, Toslib, JSON-RPC, and genesis/config support.
4. Demonstrate register → outbid/refund → extend → finish → resolve → update →
   delegate → transfer → top-up → release → re-auction on a multi-validator
   local network.

### Phase 2 — testnet product

1. Activate the reviewed root through configuration parameter 4.
2. Deploy `tos-domains`, TOSCan indexing, and wallet integrations.
3. Run public testnet auctions with no mainnet ownership promise.
4. Measure gas, state growth, resolver latency, reorg behavior, refund failures,
   lazy finalization, release behavior, and support burden.

### Phase 3 — Agent-native integration

1. Add service-protocol and Gateway alias resolution.
2. Add Messenger and OpenFox contact resolution.
3. Prove that name transfer, overdue state, active re-auction, stale caches,
   revoked delegations, and tombstoned Capabilities all fail closed.
4. Consume the same vectors in C++, Go, Swift, Kotlin, and TypeScript.

### Phase 4 — mainnet activation gates

Every gate must be evidenced against a specific commit:

- **G1 — upstream parity:** the deployed contract release identifies its TON
  upstream commit, and every semantic diff is approved and published.
- **G2 — deterministic artifacts:** Root, Collection, and Item build
  reproducibly to published code hashes on two independent builders.
- **G3 — frozen formats:** the TIP and shared corpus freeze encoding, category
  hashes, `slice_hash` item index, StateInit, ABI, operation codes, exact 105%
  boundary rounding, durations, extension, refund, and release behavior.
- **G4 — economics:** the launch-timestamp choice of §6.6 is recorded with its
  consequences; the price tier magnitudes and storage reserve are approved; and
  the fact that proceeds accumulate irrecoverably in the Collection is either
  accepted in writing as the v1 outcome or replaced through an approved
  Collection fork. No alternate auction model is hidden in economics
  configuration.
- **G5 — implementation review:** two independent reviews cover the registrar
  and item, and an independent resolver reproduces the shared vectors.
- **G6 — lifecycle evidence:** a public testnet demonstrates registration,
  multiple outbids and refunds, anti-sniping extension, lazy finalization,
  transfer, record update, top-up, 366-day release, and re-auction to a different
  owner at the same item address. Accelerated test-only times are permitted.
- **G7 — client safety:** every security-sensitive client rejects raw DNS
  records while an auction is active or renewal is overdue, and displays the
  raw address, network, checkpoint age, auction state, and renewal deadline.
- **G8 — reorg and cache behavior:** resolvers and indexers roll back abandoned
  checkpoints and invalidate affected name, lifecycle, and delegation caches.
- **G9 — governance and operations:** the parameter-4 protection decision of
  §5.1 is active; the ConfigParam 80 choice of §6.6 is recorded, including
  whether any core change was made and who may add an entry; upgrade notice,
  incident response, and a public contact path are published. The runbook states
  that governance destruction deletes the item account but its ConfigParam 80
  entry continues to block registration until governance removes the entry; a
  later registration then re-creates fresh state at the same deterministic
  address.
- **G10 — upstream monitoring:** an owner and response SLA are assigned for new
  official TON DNS changes, including expedited handling of security fixes.

## 13. Required Test Matrix

At minimum, the shared corpus and integration suite cover:

- valid and invalid labels, maximum lengths, case handling, reserved `xn--`,
  and the complete Section 4.2 boundary table;
- reverse encoding, category hashes, `slice_hash(label)`, deterministic
  Collection/Item addresses, and the negative case that plain byte SHA-256 is
  not substituted for TVM slice hashing;
- minimum opening price and decay boundaries, initial-duration boundaries,
  exact 105% minimum-bid rounding, rejection one unit below the threshold,
  highest-bidder replacement, and refund destination/value;
- bids outside the auction window, a bid inside the last hour extending the
  auction to at least one hour, and multiple extensions;
- leading and trailing hyphens rejected on-chain (error 203) while interior and
  consecutive hyphens register, and a duplicate registration onto a live item
  refunding the sender instead of restarting an auction;
- lazy finalization after the end time: the capped amount forwarded to the
  Collection, owner assignment, cleared auction state, no bidder loop, and
  `last_fill_up_time` left unchanged;
- finalization triggered by `op::get_static_data` from a non-winner, versus an
  unknown opcode and a body without a `query_id` both leaving the auction
  unfinalized, and `op == 0` after the end time failing with error 406;
- every accepted bid refreshing `last_fill_up_time`, alongside owner top-up,
  transfer, and record update; rejection at the 366-day boundary and permission
  only after `>` one year;
- balance release refunding the former owner, clearing ownership, opening a
  seven-day auction with the caller as first bidder, and retaining the same
  Domain Item address;
- records surviving release and re-auction so that a new owner inherits the
  previous owner's dictionary and delegation until each entry is overwritten;
- raw-record retention across overdue/re-auction state, together with every
  wallet, Gateway, Messenger, Agent, payment, and Site client failing closed on
  lifecycle checks;
- with ConfigParam 80 absent — the current TOS state — registration succeeding
  for any valid label and `process_governance_decision` throwing;
- if ConfigParam 80 is enabled: destruction deleting the account, the retained
  config entry continuing to reject registration with error 205, removal of the
  entry, and subsequent registration re-creating fresh state at the same item
  address without merging the two account lifetimes;
- record update/delete, NFT transfer, delegated subdomains, partial resolution,
  resolver loops, excess depth, malformed consumed counts, category/type
  mismatches, unknown TL-B tags, and trailing data;
- the Section 5.5 hop trace byte for byte, a delegated subresolver correctly
  handling a slice with **no** leading NUL, and hop exhaustion distinct from
  “not found”;
- finalized checkpoint change, cache invalidation, indexer rollback, and
  historical ownership separation by transaction logical time;
- correct provenance class, with `quorum_agreed` never presented as proof;
- Agent revocation, Capability tombstone/transfer, address-to-object-ID
  re-derivation mismatch, registry-code-version staleness, Messenger delegation
  rotation/revocation, and stale DHT locators; and
- wallet confirmation of raw address, network, auction state, renewal deadline,
  and finalized checkpoint.

## 14. Operational Inspection

Until the new product contracts and APIs are implemented, existing tools can
inspect the inherited resolver interface:

```bash
cd build
./lite-client/lite-client -C /data/tos-global.json
```

```text
getconfig 4
dnsresolve <domain>                          # category 0 -> all records
dnsresolve <domain> dns_next_resolver        # follow the delegation chain
dnsresolve <domain> site                     # a specific category
dnsresolvestep <addr> <domain> <category>    # one hop against one resolver
```

**The Lite Client hashes the category argument.** `<category>` is passed through
`td::sha256_bits256(cat_str)`, and an omitted argument means the all-zero
category (`lite-client.cpp:1028-1030`). There is therefore no `-1` category:
typing `dnsresolve <domain> -1` silently queries `sha256("-1")`, a category no
contract will ever hold, and reports nothing found. Earlier revisions of this
document described `-1` as a diagnostic convention for resolver chaining; that
was wrong. The next-resolver category is the hash of the literal string
`dns_next_resolver`, which is what the inherited contracts write
(`dns-manual-code.fc:347`). The historical signed `-1` and `1`/`2` category
numbers survive only as comments in `crypto/block/block.tlb:989-998`; the
implementation is unsigned 256-bit hashes, with category zero meaning all
records.

`dnsresolvestep` differs from `dnsresolve` in two ways that matter when reading
its output: it does not recurse, and it prepends a NUL to the encoded name
(`mode = 3`, `lite-client.cpp:1826, 1988`), i.e. it asks the named resolver
about the name *relative to itself* (§4.2).

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

TOS code locations established for this work:

- [tosnetwork/tos `crypto/smartcont/dns/`](https://github.com/tosnetwork/tos/tree/main/crypto/smartcont/dns)
  — the `.tos` on-chain naming contracts, ported from
  `ton-blockchain/dns-contract`
- [tosnetwork/tos `domains/`](https://github.com/tosnetwork/tos/tree/main/domains)
  — registrar and management application

Inherited design references:

- [TON DNS documentation](https://github.com/ton-blockchain/docs/blob/main/content/foundations/web3/ton-dns.mdx)
- [TEP-81: TON DNS Standard](https://github.com/ton-blockchain/TEPs/blob/master/text/0081-dns-standard.md)
- [TON DNS reference contracts](https://github.com/ton-blockchain/dns-contract) — the parity source, pinned in §6.1
- [TON DNS frontend](https://github.com/ton-blockchain/dns) — the upstream analogue of `tos-domains`, useful as a feature-scope reference only

These TON references explain the inherited resolver mechanics. This document,
the accepted TOS TIP, and deployed TOS contract code are authoritative for the
`.tos` namespace.
