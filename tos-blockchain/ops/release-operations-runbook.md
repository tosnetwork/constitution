# Release Operations Runbook

Operational notes for running TOS validator infrastructure through release,
testnet, and mainnet phases. Collected from the 2026-08 pre-mainnet readiness
assessment and the dark-feature inventory review.

This complements [../tos-release-policy.md](../tos-release-policy.md) (surface
stability policy) and [../tos-upgrade-process.md](../tos-upgrade-process.md)
(change process). Those documents say what may change and how; this one says
what operators must watch while the network runs.

## 1. Dark-Feature Locks Are a Release Invariant

The mainnet binary ships five features dark: AIPoW native issuance, TOS DNS,
Native Registry, the cross-chain bridges, and pooled staking economics. The
current security posture ("no live vulnerability") holds **if and only if
these stay dark**. Each is held by a different kind of lock:

| Feature | Lock | Operator rule |
|---|---|---|
| AIPoW issuance | `capAipow` unset + ConfigParam 90–93 absent + contracts undeployed | Never vote these on outside a planned activation |
| TOS DNS | ConfigParam 4 absent | Absence is intentional; clients fail closed |
| Native Registry | Registry contract undeployed (SHA256C opcode itself is live at version 14) | Do not deploy registry code outside a planned activation |
| Cross-chain bridge | No ConfigParam, no genesis deployment, no production oracle set | See §6 before any deployment |
| Pooled staking | ConfigParam 17 `max_stake_factor = 1` | Raising it is a governance action gated on validator count |

Operator consequences:

- Treat any config-parameter vote touching these locks as a **protocol
  activation**, not a routine parameter change. Activation requires a
  frozen-version external audit covering that feature first.
- Alert on unexpected appearance of ConfigParam 4, 90–93, or changes to
  ConfigParam 17. On this network, "parameter appeared" equals "feature went
  live".
- The config-master key can change these parameters unilaterally, bypassing
  the validator vote. That key must sit under the same custody discipline as
  the genesis/minter keys (multisig or time-lock, documented key ceremony).

## 2. Validator Log Management

At current default verbosity each validator produces roughly **10 GiB of text
logs per day** (measured: 44.9 GiB per node in 4.7 days). Unmanaged, logs
dominate the data directory within a week.

What is safe to clean, and how:

- Engine text logs live directly in the node data directory: `log`,
  `log.thread<N>.log`, `session-logs`, and `error/log.txt`. These are the
  only files log cleanup may touch.
- The engine holds these files open. **Truncate in place
  (`truncate -s 0 <file>`); never delete them** — deleting an open file frees
  no space until the process exits, and loses the write target.
- **Never touch numbered `*.log` files under subdirectories** (`celldb/`,
  `state/`, `consensus/`, `adnl/`, `overlays/`, `files/`, `archive/`,
  `wc0-index/`, `dht-*/`). Those are database write-ahead logs; removing one
  corrupts the store.
- No restart is needed for truncation. If a restart is ever required, control
  nodes through the service manager (`systemctl`), never by killing
  processes.

Standing policy for testnet and mainnet:

- Lower the engine verbosity in the service configuration; the current level
  is a debugging setting, not an operating one.
- Add rotation (e.g. `logrotate` with `copytruncate`) sized so that the log
  share of the data directory stays under a few GiB per node.
- Before truncating, snapshot the tail of each file if an investigation is
  open; memory-statistics sampling lines (`JEMALLOC_STATS`) live in these
  logs and are the input to the resource baseline in §4.

## 3. Election and Validator-Set Rollover Watch

A long-running network was observed with ConfigParam 34 (current validator
set) **expired for more than three days without rollover** — the election
path silently stopped being exercised and nothing alarmed.

Required monitoring:

- Alert when `utime_until` of the active validator set is within one election
  cycle of expiry and no successor set is staged.
- Alert immediately when the active set's `utime_until` is in the past.
- Track that elections actually occur each cycle (Elector state changes), not
  merely that blocks are produced. Block production continuing on a stale set
  masks a dead election path.

## 4. Resource Baseline and Alerting

Before mainnet, capture a continuous **≥ 7 day** resource baseline on the
frozen release version (it can be taken during the public-testnet soak):

- jemalloc `allocated` from the engine's periodic `JEMALLOC_STATS` log lines
- `RssAnon` per process from `/proc/<pid>/status`
- data-directory disk usage, with the log share separated out (§2)

Derive mainnet memory and disk alert thresholds from this baseline rather
than from ad-hoc observation. Any sustained growth rate with no plateau needs
allocation-site attribution before release, not after.

## 5. TOS DNS: Operational Discipline After Activation

The DNS resolution trust chain has known upstream-inherited limitations that
are **covered operationally, not in code** (2026-08-24 disposition): official
wallets do not verify Merkle proofs of resolution results, the lite-client
trusts server-returned resolution, resolver records survive domain ownership
transfer, and the config-master key can rewrite the DNS root parameter.

The operational coverage that must therefore hold whenever DNS is lit:

- Official wallets and clients connect only to endpoints the project
  operates; those endpoints are part of the trusted computing base and sit
  under infrastructure monitoring and access control.
- Endpoint credentials and the config-master key follow the governance
  custody discipline of §1.
- Product documentation states plainly that a `.tos` name can change hands
  and that a resolution result is only as fresh as the record behind it.
- Client-side proof verification remains a tracked long-term improvement, not
  an activation blocker.

## 6. Bridge Deployment Discipline

The EVM vote digests now bind both the bridge contract address and the
EIP-1344 chain ID, so cross-chain signature replay is cryptographically
excluded. Two operational rules remain mandatory for any deployment:

- All target chain IDs across every deployment must be **pairwise distinct**.
- Run an independent indexer that reconciles lock/mint and burn/release
  totals across both sides; alert on any divergence.
- A bridge deployment is itself a feature activation under §1: frozen-version
  external audit first, and a documented production oracle set.

## 7. Scope

This runbook is Level 3 (experimental) under the release policy's stability
levels: it reflects the current pre-mainnet operating knowledge and should be
revised as testnet operations produce better numbers.
