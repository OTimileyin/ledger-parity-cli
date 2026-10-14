# Problem Validation: Operational Reconciliation on Stellar

Document version: 1.1.0
Date: October 14, 2026
Repository: LedgerParity/ledger-parity-cli

## 1. Exact Target User

The primary users of LedgerParity are:
- Financial operations engineers, settlement clerks, and treasury teams operating automated disbursement or payment pipelines on the Stellar network.
- Organizations running the Stellar Disbursement Platform (SDP) for aid distribution, cash assistance, or bulk payouts.
- Stellar anchors, on/off-ramps, cross-border remittance providers, and crypto-fiat payout services executing high-volume classic payment operations.
- External financial auditors and compliance teams conducting post-settlement audits of Stellar-based fund flows against internal general ledgers.

These operators maintain an internal business database (recording user accounts, initiated payout instructions, approval states, and intended amounts) alongside an automated transaction submitter that communicates with Horizon or a Stellar Core node.

## 2. Operational Failure Scenarios

When an application dispatches classic payments to Stellar, operational divergence occurs in several documented failure modes:

1. **Unconfirmed Submission / Phantom Completion**:
   An application initiates a batch of payments and marks records as "completed" or "submitted" in its local SQL database based on network dispatch. If the transaction submission times out, fails Horizon validation (such as unexpected fee spikes, invalid sequence numbers, or insufficient balance), or drops from the transaction queue, the database asserts settlement that never occurred on-chain.

2. **Divergent Amounts and Rounding Variances**:
   If internal systems calculate deductions, conversions, or network fees inconsistently, delivered amounts on-chain diverge from internal accounting ledgers. Classic payment operations require exact stroop-level precision (1 stroop = 0.0000001 XLM or unit of asset). A variance of even a single stroop creates an accounting imbalance across tens of thousands of payouts.

3. **Accidental Duplicate Payouts**:
   Retry logic without strict idempotency keying or misconfigured cron workers can submit two or more distinct Stellar transactions for a single internal payment obligation. The internal database might record one payout, while on-chain ledger records reveal two separate operations debiting the organization's account.

4. **Crash Before Local Persistence**:
   A transaction successfully commits to a Stellar ledger, but the application server crashes, loses its database connection, or encounters an unhandled exception before writing the transaction hash or status back to the local database. The internal state remains "pending" or "failed", prompting manual re-execution by operators.

5. **Counterfeit Asset Ambiguity**:
   An operator intends to pay in an anchor-issued stablecoin (such as USDC issued by Centre/Circle). Without strict asset identity checking (asset code plus asset issuer account ID), an operator can send or accept an identically named asset issued by an untrusted or fraudulent address.

## 3. Ordinary Workarounds

In standard operational environments without automated reconciliation tooling, teams rely on the following manual workarounds:

1. **Ad-hoc SQL and CSV Dumps**:
   Engineers export payment rows from Postgres or MySQL into spreadsheets.
2. **Manual Horizon API Queries**:
   Engineers write one-off bash scripts with `curl` or Python scripts querying `/accounts/{account_id}/payments` to collect paginated JSON.
3. **Spreadsheet VLOOKUP / Index-Match**:
   Finance teams merge internal exports with Horizon CSV exports using Excel or Google Sheets, comparing amounts and destination addresses manually.
4. **Manual Explorer Lookups**:
   Disputed payouts or missing transfers are reviewed individually on public block explorers like StellarExpert by pasting transaction hashes or public keys.

## 4. Operational Friction and Overhead

Manual reconciliation workflows introduce significant operational overhead and tangible risks:

1. **Engineering and Operational Triage Burden**:
   Resolving payment discrepancies manually requires cross-functional coordination between support clerks, database administrators, and blockchain engineers to inspect Horizon logs, raw transaction envelopes, and database state.
2. **Extended Investigation Cycles**:
   Manual investigation of unconfirmed or disputed payouts routinely extends resolution cycles by 24 to 72 hours, delaying beneficiary access to funds and increasing support backlog.
3. **Irreversible Capital Losses**:
   On non-custodial blockchains like Stellar, duplicate payouts or overpayments caused by uncoordinated retries are generally irreversible unless the recipient voluntarily returns the funds. Undetected duplicate payments lead directly to unrecoverable capital losses.

## 5. Why Stellar-Specific Identifiers Matter

Generic reconciliation algorithms that treat payments as simple strings fail on Stellar due to native protocol semantics:

1. **Network Passphrases**:
   Every Stellar transaction envelope is cryptographically bound to a network passphrase (`Public Global Stellar Network ; September 2015` for Mainnet, `Test SDF Network ; September 2015` for Testnet). An export claiming settlement on Mainnet must never match an observation from Testnet. LedgerParity treats the network passphrase as an immutable component of payment identity.

2. **Operation IDs vs Transaction Hashes**:
   A single Stellar transaction can bundle up to 100 distinct operations. Identifying payments solely by transaction hash causes ambiguity when multiple payments are grouped into a single batch transaction. 
   Operation IDs are not universally required for all single classic payments with distinct amounts and counterparties. However, the 64-bit Stellar Operation ID (encoding ledger sequence, transaction index, and operation index) is strongly recommended for disambiguation. Without an explicit operation ID, multi-operation batch payments or payments evaluated under incomplete coverage cannot be unambiguously matched and must remain UNKNOWN.

3. **Triple Asset Identity (Type, Code, Issuer)**:
   Unlike single-token chains, Stellar assets are uniquely defined as a triple:
   - Asset Type: `native` (XLM), `credit_alphanum4` (1-4 characters), or `credit_alphanum12` (5-12 characters).
   - Asset Code: Ticker symbol (such as `USDC` or `EURT`).
   - Asset Issuer: The 56-character Ed25519 public key of the issuing anchor account. Native XLM has an empty issuer.
   LedgerParity validates all three dimensions, preventing token spoofing or mismatched issuer discrepancies.

4. **Exact Decimal Precision**:
   Stellar uses 7 decimal places (stroops). Floating-point conversions in general-purpose software introduce rounding errors. LedgerParity parses, compares, and formats all amounts as exact scaled integers or decimal strings without float conversion.

## 6. Why Explorers and Horizon Alone Do Not Reconcile

Public explorers (such as StellarExpert) and raw Horizon endpoints provide views of what happened on the ledger, but they cannot perform reconciliation:

- **Zero Visibility into Business Intent**:
  Horizon knows only what was submitted. It has no knowledge of internal database IDs, intended payment schedules, unsubmitted obligations, canceled payments, or internal batch boundaries.
- **No Discrepancy Detection**:
  Horizon does not know if an observed 10 XLM payment was meant to be 11 XLM. It only records that 10 XLM was transferred.
- **No Orphan or Absence Determination**:
  If a payment is missing from the chain, Horizon returns nothing. Determining whether the payment is truly missing versus outside the queried window requires an engine that calculates bounded coverage assertions over closed intervals.
- **No Offline Verifiable Evidence Bundles**:
  Horizon queries are live, unindexed by internal ID, and subject to node retention pruning. Explorers cannot generate self-contained, offline-replayable evidence bundles for compliance auditors.

## 7. Limits of Proof and Completeness

LedgerParity enforces strict epistemic boundaries:
- **Read-Only Inspection**:
  The tool never submits transactions, moves funds, or modifies databases.
- **Coverage Assumptions**:
  Reconciliation depends on the validity of the operator's declared scope (account list, time window, and whether the internal export is asserted as complete). The default completeness assertion is false; it requires explicit verification of database extraction completeness to be set to true.
- **Explicit UNKNOWN States**:
  If an internal payment has no matching chain record, but the chain query window does not fully cover the settlement timeframe tolerance, LedgerParity classifies the outcome as `UNKNOWN`, not as an authoritative discrepancy. Similarly, if multiple candidate operations match identical parameters without an operation ID to disambiguate them, the tool outputs `UNKNOWN`.
- **Horizon Node Trust and Integrity Boundaries**:
  The tool relies on the configured Horizon server for ledger observations. Horizon is an off-chain API indexing service, not an infallible cryptographic oracle. Local SHA-256 proof files (`.proof.json`) authenticate only the byte-level integrity of the generated local report against post-run tampering; they do not authenticate the truth of the underlying payment records or cryptographically verify the Stellar blockchain.

## 8. What Makes LedgerParity Reusable

LedgerParity is decoupled into distinct modules:
1. **Core Matching Engine**:
   The engine in `ledger-parity-core` (`github.com/LedgerParity/ledger-parity-core/pkg/engine`) is completely isolated from storage backends, HTTP frameworks, and user interfaces. It accepts canonical memory structures and executes deterministic integer-stroop comparisons.
2. **Connector Ecosystem**:
   The connectors library in `ledger-parity-connectors` (`github.com/LedgerParity/ledger-parity-connectors`) provides standardized adapters for canonical CSV and JSON formats, as well as third-party contracts such as the Stellar Disbursement Platform (SDP 7.0.0 CSV specification in `pkg/sdp`).
3. **CLI and Replayable Evidence**:
   The CLI tool in `ledger-parity-cli` (`github.com/LedgerParity/ledger-parity-cli/cmd/ledger-parity`) outputs portable evidence bundles (`--bundle`) that can be transferred to an auditor and replayed (`--replay`) without network access or database credentials, yielding byte-identical reports.
4. **Offline Dashboard**:
   A zero-dependency, single-file HTML viewer (`dashboard/index.html`) runs directly in any modern browser without an active web server, allowing finance teams to inspect reports, search candidate operations, and review coverage without technical setup.
5. **Decoupled Hash Registry**:
   An optional Soroban report-hash contract (`https://github.com/LedgerParity/-ledger-parity-contract`) exists as an independent external registry. The CLI does not invoke this contract during reconciliation, preserving the principle that on-chain hash registration proves only an administrator's record commitment, not the factual accuracy of the underlying payment data.

## 9. Comparison with Existing Stellar Tooling

| Capability | LedgerParity | Stellar Disbursement Platform (SDP) | StellarExpert | Raw Horizon Queries |
|---|---|---|---|---|
| **Primary Focus** | Application vs on-chain reconciliation | Disbursement scheduling and execution | Public ledger explorer and directory | HTTP API for ledger data access |
| **Input Source** | Application export plus Horizon API | Internal database and CSV upload | Ledger directly | Ledger directly |
| **Application vs Chain Matching** | Deterministic comparison of internal records with on-chain operations | Manages internal disbursement state machine | None (displays on-chain events only) | None (serves raw transaction/op streams) |
| **Identifies Missing Payments** | Flags `MISSING_ON_CHAIN` when coverage is complete | Tracks SDP disbursement states (such as pending/failed) | None | None |
| **Identifies Duplicate Submissions** | Flags `DUPLICATE_INTERNAL` and shared-operation claims | Enforces disbursement idempotency keys | None | None |
| **Identifies Amount Discrepancies** | Flags `AMOUNT_MISMATCH` with exact stroop deltas | None (assumes SDP payout amount is intended) | None | None |
| **Candidate Disambiguation** | Recommends Operation IDs; flags ambiguous candidates as `UNKNOWN` | Uses SDP internal disbursement ID | Displays all operations in a transaction | Returns operations list |
| **Epistemic Classification** | Conservative UNKNOWN outcomes on incomplete coverage | Binary internal statuses | N/A | Paginated queries only |
| **Offline Replayable Evidence** | Portable JSON evidence bundle with local SHA-256 report proof | CSV export of disbursement records | None | None |

### Sources and Verification Dates (Checked October 2026)
1. Stellar Disbursement Platform Backend Architecture and Export Handler:
   Source: `https://github.com/stellar/stellar-disbursement-platform-backend/tree/v7.0.0`
   Checked: October 2026. Verified export headers and payment state machine definitions.
2. Stellar Developers Documentation on Operations and Transactions:
   Source: `https://developers.stellar.org/docs/fundamentals-and-concepts/stellar-data-structures/operations-and-transactions`
   Checked: October 2026. Verified 100 operations limit per transaction and 64-bit operation ID structure.
3. Stellar Horizon API Specification:
   Source: `https://developers.stellar.org/api/horizon/resources/operations`
   Checked: October 2026. Verified payments endpoint pagination and filtering constraints.
