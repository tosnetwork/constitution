# Workchain Execution Registry

The workchain execution registry is the host-chain mechanism that maps a workchain descriptor from `ConfigParam 12` to a local execution engine.

## Current Scope

A production node registers the native TVM execution path only. One additional
engine is approved in direction but is not registered, not enabled and not
deployable; it is described under "Approved Engine" below.

| Engine | Descriptor key | Status |
|---|---|---|
| TVM | `wfmt_basic`, `vm_version = -1` | registered, active |
| Confidential-amount workchain engine | BlockTransition, workchain 2 | approved in direction, not registered |

No other engine key is registered or advertised by the node. A test-only counter
engine exists behind a build macro that is off in release builds and logs a
"not for deployment" warning when compiled in; it is a fixture, not a policy
exception.

## Behavior

When a non-masterchain workchain appears in `ConfigParam 12`, the validator resolves its descriptor through the registry. If the local binary has no matching engine, validation fails rather than falling back to TVM.

This keeps unknown execution domains fail-closed.

## Approved Engine: Confidential Amounts on Workchain 2

The one approved non-TVM direction is a BlockTransition engine on workchain 2
that hides transfer amounts. Sender, receiver and the transfer graph stay
public; there is no anonymity set. Internally this work is tracked under the
name Uno, which is why that identifier is permitted inside the paths listed by
the CI scan below.

Approval is a direction, not a deployment permission. Three things follow.

**It is not a second asset.** There is no separate token, no independent
issuance, no mining and no block subsidy. A confidential balance is a claim on
the same native coin, backed one-to-one by custody. The retired independent-asset
implementation and the retired shielded-pool implementation are not revived by
this approval and remain banned, including inside the approved paths.

**It is not registered.** The engine has no production registration today. Host
integration, admission, fee settlement and independent validator replay are
separate work. Component code existing in the tree is not registration, and
registration is not activation.

**Enabling the config gates ahead of the executors is unsafe.** The engine is
gated on global version 15, on global version 16 for the storage-participant
boundary, and on the `capBlockTransition` capability. When the network
configuration enables a version or capability beyond what a binary supports,
the collator and validator only log an error and continue; they do not refuse.
The version number is therefore a release-process proxy for execution
capability, not a code-level guarantee. Neither gate may be enabled
network-wide before the executing binaries are widely deployed, and any
rehearsal must include the case where the gate is on but the executor is not.

Before registration, this engine must still satisfy every requirement in
"Future Engine Policy" below.

## Future Engine Policy

Any engine outside the native TVM surface must define:

- descriptor key and versioning rules
- account execution policy
- config validation
- compute semantics
- migration rules for active networks
- tests for collator and validator paths

Until such an engine exists and is registered, this binary executes native TVM only.

AI actor roadmap work must not add an execution engine as a shortcut for model,
service, or agent integration. Those integrations should be represented as
native service actors, off-chain workers with on-chain authorization, or proof
adapters. The approved confidential-amount engine is not an exception to this
rule: it exists to hide amounts, not to execute agent workloads.

## Cross-Chain Bridge Contracts

The `crosschain/` directory vendors bridge smart contracts whose Solidity half is deployed on external counterparty chains (Ethereum, BNB Smart Chain, ...), never inside the TOS node. The TOS-side half of each bridge is ordinary native TVM code selected through masterchain ConfigParams. Vendoring these sources does not register an execution engine, add a workchain descriptor, or weaken the policy above.

## CI Enforcement

`scripts/check-no-removed-execution-domains.sh` in the chain repository enforces
this document. It runs two independent scans.

The removed-execution-domain scan keeps EVM banned everywhere outside
`third-party/`, and excludes `crosschain/` and the bridge build, test and verify
entry points for the reason given above. It permits the approved engine's
identifier only inside an explicit list of paths covering the engine tree, the
workchain host and its wiring, and the corresponding tests and documents.

The retired-symbol scan bans identifiers belonging to the removed
independent-asset and shielded-pool implementations. It applies to code and
build text everywhere, including the approved paths, and including
`crosschain/`. It does not apply to Markdown, because a record of what was
removed has to be able to name it. Two narrow exceptions are bound to a specific
path and a specific word rather than to a whole file: the standard mnemonic
wordlists, which legitimately contain one of the banned words, and the
dependency gate that must name the banned crates in order to assert their
absence.

A scan that reads no files fails rather than reporting success.
