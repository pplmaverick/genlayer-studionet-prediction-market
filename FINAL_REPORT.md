# Final report — football prediction market on GenLayer Studionet (2026-10-21, UTC)

**Conclusion.** Hosted Studionet (chain id 61999) runs GenVM `v0.2.16` (observed in its RPC error log). v0.2-style contracts deploy and run;
the v0.3-style examples on `genlayer-studio` `main` and the latest SDK docs are rejected with `invalid_contract`, yet the deployment
transaction finalizes and the CLI still reports success. A prediction-market contract in v0.2 style
(`contracts/prediction_v02.py`, MIT, from `genlayer-studio@58293cd30a`) was deployed, resolved for *Nottingham Forest v Arsenal (2026-10-18)* and read back
as `{has_resolved: true, score: '0:1', winner: 2}`. Not done: Bradbury.

Tooling: genlayer CLI 0.39.2 (pinned, installed with `--ignore-scripts`), genlayer-js 1.1.8, Node v24.12.0, Python 3.14.4, macOS arm64.
Throwaway account, 0 GEN balance throughout. Addresses, transaction hashes and per-transaction evidence: [`DEPLOYMENTS.md`](DEPLOYMENTS.md).

## 1. Deployment summary (Studionet)
| # | Contract | Address | Exists |
|---|---|---|---|
| 1 | unmodified upstream `football_prediction_market.py` @ `f799be5fb7` (v0.3 style) | `0x0A665ac055eC92dF80D2D7f362Ff0a40Dc838221` | no (`invalid_contract`) |
| 2 | unmodified upstream `storage.py` @ `f799be5fb7` (v0.3 style) | `0xA6d6b925d77231582d81187FB71Dfab584c430b4` | no (`invalid_contract`) |
| 3 | docs "first contract" example (v0.2 style) | `0xd3643f8AdA1b0D6447E4b305b13DC0a2D72e43C8` | yes; `run()` → `Hello, world` |
| 4 | `contracts/prediction_v02.py` | `0x08FfbC04A794E8D2Bcba7355E46eC9bDc2Bfb834` | yes; resolved, state read back |
| 5 | code block of the docs `examples/prediction` page (v0.2 style) | `0x7800A2a89c5CB3b1a05DD6005eF319b07Ba8Da2a` | yes; `resolve()` ran |

"Exists" is judged with `gen_getContractSchema` / `genlayer call`; `eth_getCode` returns `0x` even for existing contracts on Studionet (observed).

## 2. Why the v0.3-style deployments failed
**Observed**
- `gen_getContractSchemaForCode` on the hosted server accepts v0.2-style sources (`from genlayer import *`, `gl.Contract`,
  `py-genlayer:1jb45aa8…`) and rejects the v0.3-style examples (`# v0.3.0`, `import genlayer as gl`, `gl.contract.Contract`,
  `py-genlayer:9b8kjyda…`) with `invalid_contract`.
- The same CLI, SDK, account and balance produced both outcomes, so neither the tooling version, the 0 GEN balance, nor the contract logic is
  what separates them.
- On the rejected deployments, every validator receipt has `execution_result = ERROR`, result `\x02invalid_contract`, empty `stderr`; the
  transaction is nonetheless `FINALIZED` / `MAJORITY_AGREE`.
- `genlayer-studio` `main` pins GenVM `v0.6.0-rc0` (`third_party/genvm/version`); the SDK migration guide (v0.3.0-rc10) says v0.3 is not
  compatible with v0.2.x (imports, `gl.Contract` → `gl.contract.Contract`, `u256(...)` no longer callable).
- No page or RPC method I tried states the GenVM/runner version Studionet supports (`web3_clientVersion`, `gen_getVersion`, `sim_getVersion`,
  `sim_version`, `gen_version`, `sim_getConfig`, `gen_clientVersion`, `rpc.discover` → method not found; `/health` has no version field).

**Unverified**
- Whether the rejection is caused by the runner hash, the syntax, or both (the hash-only variant I tried still carried the `# v0.3.0` first line).
- Whether Studionet will move to v0.3.

## 3. Findings by status
### Already reported by others (as found by searching `genlayerlabs` on 2026-10-21)
| Topic | Existing reports |
|---|---|
| `invalid_contract` on StudioNet 61999 with CLI 0.39.2, including a minimal probe | genvm-manager#50 (closed), genvm-manager#42, genlayer-studio#1757 |
| `invalid_contract` carries no details | genlayer-studio#1609 (open), #1612, #1627 |
| Success printed although the contract does not exist | genlayer-docs#487 (open), genlayer-studio#1276 (merged 2025; current behavior differs — unverified whether it regressed) |
| `latest` CLI/SDK still use the old Bradbury RPC/addresses; fix only in rc | genlayer-js#236, genlayer-cli#427 (merged), genlayer-cli#425, genlayer-docs#486, #514 |
| No `--gas`; exact `eth_estimateGas` limit reverts | genlayer-cli#402, genlayer-js#205, genlayer-cli#404 |
| Bradbury per-transaction gas ceiling 2^24 | genlayer-cli#419 |

### New (no matching issue found)
| Finding | Kind |
|---|---|
| Hosted Studionet (v0.2.16) and `genlayer-studio` `main` examples / latest SDK docs (v0.3) disagree; supported version not stated or exposed | docs / infrastructure |
| `genlayer deploy` prints success from a decided consensus status without checking the execution result (CLI bundle `dist/index.js` ~54909–55032; SDK wait ~51726–51770) | CLI |
| Docs prediction page: `winner`/`score` not declared, no read method, "no Read Methods" sentence, GitHub link points to `main` (now v0.3 style) | docs |
| Faucet requires ≥ 0.01 ETH on Ethereum mainnet per its UI; the faucet README and `networks.md` do not state the threshold | docs |
| `sim_fundAccount` returned a finalized transaction but balance stayed 0; Studionet accepted deployments at 0 balance | possible bug (observed) |
| Docs list `account create --output`; CLI 0.39.2 has `--name/--password/--overwrite/--no-set-active`; network name is `testnet-bradbury`, not `bradbury` | docs |
| npm package `genlayer@0.39.2`: no `files` field (262 files), `postinstall` copies a Studio `.env.example`, `vitest` in `dependencies`; `npm audit`: 6 vulnerabilities (2 critical) through it | packaging |
| CLI keystore files are created with mode 0644 | CLI |

### Unverified / open
- Runtime effect of the undeclared `winner`/`score` in the docs page version (it ran; the fields cannot be read back).
- Whether the Phase 2 rc packages deploy to Bradbury (not installed, not attempted).
- Why `sim_fundAccount` does not change the displayed balance.
- Observations not analysed: validators' `stdout` is exactly 5000 characters; an `UserWarning: Detected pickling storage class. Reading
  storage in nondet mode is not supported` appears when a non-deterministic function reads `self.*`.

## 4. Bradbury (read-only findings only)
Chain id and genesis hash match the docs. The CLI/SDK `latest` default RPC is the old domain `rpc-bradbury.genlayer.com`; its built-in
`mainContract` `0x0112Bf6e83497965A5fdD6Dad1E447a6E004271D` and `stakingContract` `0x4A4449E617F8D10FDeD0b461CadEf83939E821A5` have no
bytecode on `rpc.testnet-chain.genlayer.com`, while the docs' AddressManager `0x3F71AE080BD6270A91dBcc7960d88797828538E0` does. No transaction was sent.
