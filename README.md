# Football prediction market on GenLayer Studionet — notes and a working v0.2 build

**As of 2026-10-21 (UTC):** the hosted GenLayer Studionet (`https://studio.genlayer.com/api`, chain id 61999) accepts
v0.2-style contracts (the server reports GenVM v0.2.16) and rejects the v0.3-style examples currently on `genlayer-studio` (e.g. at commit f799be5fb7) with a bare `invalid_contract` — while the
deployment transaction still finalizes and the CLI still prints "Contract deployed successfully.". This repo contains a prediction-market
contract that deploys and resolves on Studionet, and the evidence behind that statement.

## Observed (2026-10-21)
- The GenVM version in a failing Studionet RPC response is `v0.2.16-x86_64-linux-release`.
- v0.2-style contract (`from genlayer import *`, `gl.Contract`, `# { "Depends": "py-genlayer:1jb45aa8…" }`): loads, deploys, runs.
- v0.3-style contract (`# v0.3.0`, `import genlayer as gl`, `gl.contract.Contract`, `py-genlayer:9b8kjyda…`; the unmodified upstream examples at
  commit `f799be5fb7`): `invalid_contract`; the deploy transaction is `FINALIZED / MAJORITY_AGREE` with `execution_result = ERROR`, and the
  contract does not exist afterwards (`gen_getContractSchema` and `genlayer call` return "not found").
- `contracts/prediction_v02.py` was deployed with `("2026-10-18", "Nottingham Forest", "Arsenal")`; `resolve()` returned
  `{score: "0:1", winner: 2}` and `get_resolution_data()` read back `{has_resolved: true, score: '0:1', winner: 2}`.
  Addresses and transaction hashes: [`DEPLOYMENTS.md`](DEPLOYMENTS.md). Full write-up: [`FINAL_REPORT.md`](FINAL_REPORT.md).

## Not verified
- Whether Studionet rejects the v0.3 examples because of the runner hash, the syntax, or both.
- Where the supported GenVM/runner version of Studionet is documented — none of the pages or RPC methods tried exposed it.
- Nothing was deployed to Bradbury.

## Reproduce (read-only, no transaction, no key)
Check whether a contract file loads on the hosted server:

```bash
python3 - contracts/prediction_v02.py > req.json <<'EOF'
import json, sys
print(json.dumps({"jsonrpc": "2.0", "id": 1, "method": "gen_getContractSchemaForCode",
                  "params": [open(sys.argv[1], encoding="utf-8").read()]}))
EOF
curl -sS -X POST https://studio.genlayer.com/api -H 'content-type: application/json' --data-binary @req.json
```

`params` is the plain source string. A success returns the constructor and method schema; a failure includes
`genvm_log[0].version` and `{"kind":"VM_ERROR","message":"invalid_contract"}`. To judge a *deployment*, use `gen_getContractSchema` on the
returned address rather than the CLI message or `eth_getCode` (observed: `eth_getCode` returns `0x` even for contracts that exist).

## Environment
genlayer CLI `0.39.2` and genlayer-js `1.1.8` (pinned in `package.json`; the CLI was installed with `--ignore-scripts`), Node v24.12.0,
Python 3.14.4, macOS arm64. Deployments used a throwaway account with a 0 GEN balance (Studionet accepted the transactions).

## License and sources
- `contracts/prediction_v02.py` is `genlayerlabs/genlayer-studio` `examples/contracts/football_prediction_market.py` at commit `58293cd30a`
  (2026-03-27), MIT, Copyright (c) 2024 GenLayer Labs Corp. The contract code is unmodified; only the first line (`# v0.2.16`) was removed and
  a provenance note added. See [`contracts/SOURCE.md`](contracts/SOURCE.md) and `contracts/LICENSE.genlayer-studio`.
- The notes and reports in this repo are provided as-is. Not affiliated with GenLayer Labs.
