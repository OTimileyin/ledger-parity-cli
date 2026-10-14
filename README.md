# ledger-parity-cli

![LedgerParity](assets/lp-banner.png)

[![Go](https://img.shields.io/badge/Go-1.22.2%2B-3FE0C4?style=flat&logo=go&logoColor=white&labelColor=0B0E1E)](https://go.dev/dl/)
[![License](https://img.shields.io/github/license/LedgerParity/ledger-parity-cli?style=flat&color=7A5CFF)](LICENSE)
[![CI](https://img.shields.io/github/actions/workflow/status/LedgerParity/ledger-parity-cli/ci.yml?branch=main&style=flat&label=CI&logo=github&labelColor=0B0E1E)](.github/workflows/ci.yml)

See the [LedgerParity documentation](https://ledgerparity.vercel.app/) (also available at [ledgerparity.github.io](https://ledgerparity.github.io/)) for the full organization overview, concepts, quick start, and evidence discipline.

A read-only tool for Stellar payment operators to compare an application export with ordinary classic Stellar payment operations. It helps investigate settlement notifications that were missed, incorrect amounts and duplicate application records. Developer preview: no demonstrated operator adoption or production-readiness claim.

Download [v0.3.2-preview](https://github.com/LedgerParity/ledger-parity-cli/releases/tag/v0.3.2-preview) for Windows, Linux or macOS, with SHA-256 checksums and the offline dashboard included. See [release notes](RELEASE_NOTES.md) for installation and validation limits.

The [delivery record](DELIVERY.md) lists completed software, verification evidence and the external gates remaining before production readiness.

Version 0.3.0-preview adds a release-scoped **SDP 7.0.0 CSV workflow** and **offline evidence replay**. See [runnable SDP example](docs/SDP_EVIDENCE.md): export mapping requires independently known sender/network/settlement scope, preserves business references and leaves export completeness unproven. `--bundle new-file.json` captures evidence alongside a report; `--replay new-file.json` verifies and re-evaluates it offline. These are implemented features with synthetic contract tests, not deployed SDP or production validation.

From this repository alone, install Go 1.22.2+ (prefer a currently supported Go release), then:

```sh
go test ./...
go vet ./...
go build -o ledger-parity ./cmd/ledger-parity
./ledger-parity --demo --format json --out -
```

On Windows use `go build -o ledger-parity.exe ./cmd/ledger-parity`, then `.\ledger-parity.exe --demo --format json --out -`. If Go is installed but unavailable in your terminal, add its bin directory to the current shell PATH or use its full executable path.

The demo is deterministic and offline: **1 match, 4 discrepancies, 3 unknowns**, exit **3**. It includes an exact settlement, a one-unit amount difference, a missing expected operation, duplicate internal IDs, a pending record and an ambiguous batch. Duplicate input IDs prevent a definitive orphan conclusion. These synthetic addresses and transaction identifiers are deliberately not live ledger evidence.

Use the same file-input pipeline directly:

```sh
./ledger-parity --config examples/config.json --format both --out report.json
```

Configuration is strict JSON, not YAML. Source paths and config output paths resolve relative to the config file; `--out` resolves relative to the current directory. CLI flags override config values. `--out - --format json` emits only JSON, suitable for scripts. `--demo` is explicit; running without config does not silently switch to demo.

Exit codes: **0** no discrepancies or unknowns, **1** execution error (config/input/network/output), **2** discrepancies without unknowns, **3** one or more unknowns, including mixed results (takes precedence when unknowns exist). Read report totals for mixed results. Ingestion errors produce no new report and a nonzero exit; an older report file may still exist, so always check the exit code. `go run` can wrap nonzero exit statuses; use the built binary in automation.

A **[reconciliation dashboard](dashboard/)** is included: open `dashboard/index.html` in a browser and drag-drop the JSON report to visualize matches, discrepancies, and unknowns with search and charts.

### Report verification

Save a local SHA-256 checksum after generating a report:

```sh
./ledger-parity --config examples/config.json --format json --out report.json \
  --verify auto
# writes report.json.proof.json with SHA-256 hash
```

Verify a report against a saved proof:

```sh
./ledger-parity --verify-check report.json.proof.json report.json
# prints VERIFIED or MISMATCH
```

`--verify auto` creates a new `report.json.proof.json`; an explicit path can replace `auto`. It requires JSON file output (`json` or `both`) and refuses to overwrite an existing proof or an input, report, or bundle. Reconciliation exit codes still apply. Replay cannot be combined with `--verify`. `--verify-check` accepts exactly one report path and returns 0 for matching bytes or 1 for a mismatch/error.

Local checksums compare bytes against a saved hash; they do not authenticate reports, metadata, or ledger history. Someone who replaces both files can make them agree. The separate [Soroban contract](https://github.com/LedgerParity/-ledger-parity-contract) provides hash storage code; the CLI does not call it or submit transactions.

For live read-only ingestion, copy examples/config.json, replace `stellar.on_chain_path` with `stellar.horizon_url`, set the exact network passphrase and monitored accounts, and provide a canonical application export. Always use an explicit RFC3339 start/end window. The CLI expands the Horizon scan by the time tolerance. `target_app.complete` asserts that the export contains every relevant ordinary payment for those accounts/window; leave false unless you can establish that. No secrets or wallets are needed. An optional public-testnet smoke check is available with `powershell -File scripts/Test-LiveRead.ps1` after a Windows build; it synthesizes an expected row from public testnet data and documents that evidence limit.

The canonical export contract lives in [connectors](https://github.com/LedgerParity/ledger-parity-connectors), matching/coverage semantics in [core](https://github.com/LedgerParity/ledger-parity-core). Required fields include network passphrase, operation_type `payment`, exact sender/recipient, amount as a decimal string, asset/type/issuer, timestamp (or explicit settlement interval) and status. Operation ID is recommended; reference ID means transaction hash only. Exact amounts, case-sensitive asset codes and issuers, network and direction are never bypassed by a reference. Non-completed statuses without observed settlement remain UNKNOWN. Examples are bundled into the binary for standalone use.

Supported: canonical JSON/CSV and release-scoped SDP CSV input, ordinary Horizon payment operations, bounded pagination/retries, explicit coverage and versioned offline evidence replay. Expectations use a timestamp or explicit settlement interval, never both. Unsupported: runtime Soroban/RPC ingestion, path payments, account creation/merge settlement, memo matching, durable automatic checkpoints, signing or fund movement. Named Stellopay/Facil-Pay/Trustless Work packages remain experimental examples, not CLI adapters. Provider completeness is trusted evidence, not a cryptographic proof. StrKey checksums and general muxed/base-account equivalence are not validated. Review reports before operational decisions.

[Reviewer walkthrough](docs/REVIEWER_WALKTHROUGH.md) · [verification](PROJECT_HANDOFF.md) · [roadmap](ROADMAP.md) · [appeal draft](docs/DRIPS_SUBMISSION_PREP.md) · [contributing](CONTRIBUTING.md) · [security](SECURITY.md) · [MIT](LICENSE).

## Stellar Wave submission preparation

See the [submission brief](docs/SUBMISSION.md), [verification record](docs/VERIFICATION_OCT09.md),
[maintainers](MAINTAINERS.md), and [focused contributor backlog](docs/WAVE_BACKLOG.md).
