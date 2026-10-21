# Deployments (GenLayer Studionet, chain id 61999)

All on `https://studio.genlayer.com/api`, 2026-10-21 (UTC), genlayer CLI 0.39.2 (genlayer-js 1.1.8), from a throwaway account with a
0 GEN balance. "Exists" is judged with `gen_getContractSchema` / `genlayer call` — not with the CLI message and not with `eth_getCode`
(observed: `eth_getCode` returns `0x` even for contracts that exist). Nothing was deployed to Bradbury.

| # | Time (UTC) | Contract | Address | Deploy tx | Result | Exists |
|---|---|---|---|---|---|---|
| 1 | 12:37 | unmodified upstream example `football_prediction_market.py` at `genlayer-studio@f799be5fb7` (v0.3 style) | `0x0A665ac055eC92dF80D2D7f362Ff0a40Dc838221` | `0x23cc9cd2dcd041f35ac1f7711c45529641e3f49010a07de9f7ed4cad6f9dc4d7` | `FINALIZED` / `MAJORITY_AGREE`, but `execution_result = ERROR`, `invalid_contract` | **no** |
| 2 | 13:00 | unmodified upstream example `storage.py` at `genlayer-studio@f799be5fb7` (v0.3 style) | `0xA6d6b925d77231582d81187FB71Dfab584c430b4` | `0x10520771987b0cf77665542282b80bbf9362f7cc86154b927fd45d8e806d9ae6` | same as #1 | **no** |
| 3 | 13:11 | docs.genlayer.com "first contract" example, unmodified (v0.2 style) | `0xd3643f8AdA1b0D6447E4b305b13DC0a2D72e43C8` | `0x73098544e23df76e843f3e0798d92657c2e0be4ab35e94e977b29f6444f255e7` | `SUCCESS`; `run()` → `Hello, world` | yes |
| 4 | 13:14 | `contracts/prediction_v02.py` (upstream `58293cd30a`) | `0x08FfbC04A794E8D2Bcba7355E46eC9bDc2Bfb834` | `0xd7778097271463e31b0c5419cd90da3fca94f22c71562c915995855356875e71` | `SUCCESS`; initial state `{has_resolved: false, score: '', winner: 0}` | yes |
| 5 | 13:30 | code block of docs.genlayer.com `examples/prediction` page, unmodified (v0.2 style; block sha256 `fa7090708ec8f4c17bbd1c490807fed03ab53a76400a29ab4690c43eacaf7fe5`) | `0x7800A2a89c5CB3b1a05DD6005eF319b07Ba8Da2a` | `0x4c01d9897d05b50fcbb04e517a28ba45d1f42b7082c425bdfe873e35540b4d17` | `SUCCESS`; schema has only `resolve` | yes |

Constructor arguments for #1, #4 and #5: `"2026-10-18" "Nottingham Forest" "Arsenal"`. #2 used `"hello"`, #3 used `"world"`.

## Failed deployments (#1, #2) — on-chain evidence
`eth_getTransactionByHash`: `status = FINALIZED`, `result = 6` (`MAJORITY_AGREE`), 1 round. The leader and every validator receipt has
`execution_result = "ERROR"` and `result = AmludmFsaWRfY29udHJhY3Q=` (= `\x02invalid_contract`), `gas_used = 0`, empty `stdout`, `stderr` and
`error_code`. Afterwards `gen_getContractSchema` and `genlayer call <address> <method>` return `Contract <address> not found` (code `-32001`).
The CLI printed "Contract deployed successfully." and an address for both.

## `resolve()` calls
| Contract | tx | Result |
|---|---|---|
| #4 | `0x76d50e6eca7ca70305d8d694c58bd40be2b08ff47b7d20c4c0123889e862a2c3` (13:16 UTC) | `FINALIZED` / `MAJORITY_AGREE`; leader and 3 validators `SUCCESS`, 2 validators `ERROR`/`idle` (`Validator execution cancelled after quorum`, error code `CONSENSUS_VALIDATOR_QUORUM_REACHED`). Return value `{score: "0:1", winner: 2}`. Read back with `get_resolution_data()`: `{has_resolved: true, score: '0:1', winner: 2}` — as expected for Nottingham Forest 0–1 Arsenal on 2026-10-18 (BBC Sport). |
| #5 | `0x7ea247d8e57ed9b4ac5018b05b6497d9a5bcd9e9ad745eb315520db93f34c4ae` (13:41 UTC) | `FINALIZED` / `MAJORITY_AGREE`; leader and 3 validators `SUCCESS`, 2 validators `idle`. Return value `{score: "0:1", winner: 2}`. |

`winner` encoding in this contract: team 1 = `1`, team 2 = `2`, draw = `0`; `score` is `"<team1>:<team2>"`.

### State checks for #5 (read-only)
This version has no view method and no Studio RPC exposes contract state. A write *simulation* of `resolve` (`gen_call` with `type: "write"`,
`simulateWriteContract` in genlayer-js; no transaction, no key) returned `"Already resolved"`, so `has_resolved` was persisted; the contract's
transaction count stayed at 2 (deploy + resolve) before and after.
**Unverified:** that `winner` and `score` were stored (they are not declared in that version and cannot be read back).

### Observations (not analysed)
- Validators' `stderr` for #5 contained `UserWarning: Detected pickling storage class. Reading storage in nondet mode is not supported`
  (that version reads `self.*` inside the non-deterministic function; `contracts/prediction_v02.py` copies the values to locals first).
- `stdout` for each successful validator was exactly 5000 characters (possible truncation — unverified).

## Funding
`sim_fundAccount` for the throwaway account returned a `FINALIZED` transaction (value `100000000000000000000`), but `eth_getBalance` and
`genlayer account show` stayed at 0. Deployments at this balance both failed (v0.3-style, #1–#2) and succeeded (v0.2-style, #3–#5), so the balance is not what separates them.
