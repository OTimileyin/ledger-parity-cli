# SDP export verification and evidence replay

Build the CLI from this repository alone. On Windows use ledger-parity.exe and `.\ledger-parity.exe`; the commands below use the Unix binary name.

```sh
go build -o ledger-parity ./cmd/ledger-parity
./ledger-parity --config examples/sdp/config.json --bundle sdp-evidence.json --out sdp-report.json
./ledger-parity --replay sdp-evidence.json --out replayed-report.json
```

Both reconciliation and replay exit **3**: the synthetic example has **1 match, 1 amount discrepancy, 1 unknown batch**. Other exits remain 0 (no discrepancies or unknowns), 1 (execution error), 2 (discrepancies without unknowns), and 3 (one or more unknowns, including mixed results). An unknown is not a financial discrepancy. A bundle path must be new; choose another name on repeated runs. Replay uses no configuration, original source files or network calls. It verifies the bundle digest, reruns the engine and requires the stored report to agree. A changed engine that produces different findings fails replay explicitly.

The CSV is shaped against SDP release 7.0.0, not exported from a deployed operator. Addresses, references, amounts and findings in examples/sdp are synthetic. The source contract is documented in [connectors](https://github.com/LedgerParity/ledger-parity-connectors/blob/main/docs/SDP.md).

For a real export set `target_app.format` to `sdp-csv`, a stable deployment `name`, and `source_path`. The required `sdp` object contains:

```json
{
  "release": "7.0.0",
  "sender": "INDEPENDENTLY_KNOWN_STELLAR_ACCOUNT",
  "settlement_start": "2026-09-01T11:00:00Z",
  "settlement_end": "2026-09-01T13:00:00Z",
  "assertion": "Explain the single-account export scope and independently established expected settlement interval."
}
```

Use actual verified scope; the placeholder is not a valid Stellar account. Sender must be monitored, and the explicit interval must fit the report window. The same interval applies to every row in this first adapter. Creation/update timestamps are not settlement timestamps. If you cannot justify a shared interval, obtain per-payment expectations via a separately verified export rather than guessing. Circle routes, missing receiver addresses, unknown schemas and unsupported states fail instead of silently skipping rows. `complete` must remain false because export filtering/visibility does not establish all account activity. Shared transaction hashes remain ambiguous when multiple operations fit.

To use Horizon replace stellar.on_chain_path with stellar.horizon_url and set the exact network and accounts. The bundle preserves the resulting scoped normalized observations and coverage. A Horizon error produces neither a new report nor a bundle; existing files may remain from previous runs. Always check the exit code.

## Evidence format and privacy

`ledgerparity-evidence/v1` wraps normalized expectations, observations, report, time window, tolerance/coverage, source byte hashes, build/module revisions and SDP assertions. Its SHA-256 covers the serialized data. Raw files, config paths and raw provider URLs are not embedded; live provider identity is reduced to its host. Source hashes identify the bytes actually parsed. The report retains its original generated time so replay is deterministic. Existing report JSON remains a report, not a wrapped bundle; use --bundle for the separate versioned artifact.

The CLI drops arbitrary internal metadata and chain memos; the SDP adapter drops receiver contacts. It retains accounts, amounts and business IDs needed for review. These may still be sensitive: use non-sensitive source names/assertions and review bundles before sharing. Output files use mode 0600 where supported. Inputs/bundles are bounded to 32 MiB; JSON config, canonical JSON, observations and replay reject duplicate keys and excessive nesting. The reusable connectors library has its own validation and does not inherit all CLI input limits.

A hash is not an authenticated source signature, proof of ledger inclusion or guarantee that inputs are truthful. Someone can edit and reseal a bundle. Coverage remains a provider/operator assertion; offline replay cannot refresh it. Historical results are not production-readiness evidence. Durable ingestion checkpoints and general RPC/Soroban ingestion remain unsupported.

## Operator validation gate

No real operator has validated this workflow yet. With consent, obtain a sanitized independently produced export, release/version, sender/network scope, time semantics, one known issue and a clean control. Record setup effort, findings the operator confirms, false positives and unresolved cases. A second reviewer should reproduce the result from the evidence bundle without editing Go code. If there is no additional value over existing SDP tools, revise the workflow. No outreach, adoption claim or public case study is authorized by running these examples.
