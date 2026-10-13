# LedgerParity Product Statement

Document version: 1.0.0
Evaluation Date: October 13, 2026
Target System: LedgerParity CLI, Core, and Connectors

## What LedgerParity Is

LedgerParity is a dedicated, read-only reconciliation tool that compares internal application payment records against on-chain Stellar classic payment operations. It identifies matching settlements, flags accounting discrepancies, and isolates ambiguous or unobserved transactions with strict mathematical precision.

## Implemented Capabilities

1. **Exact Classic Payment Reconciliation**:
   - Matches internal payment records against Horizon ledger operations using exact decimal amounts (7 decimal places, stroop-level precision).
   - Enforces triple asset identity (AssetType, AssetCode, and AssetIssuer), preventing spoofed or counterfeit token matches.
   - Evaluates network passphrases strictly, ensuring Testnet operations are never confused with Mainnet settlements.

2. **Conservative Discrepancy Classification**:
   - Classifies records as `EXACT` when amount, asset, sender, recipient, and direction match.
   - Identifies discrepancies: `AMOUNT_MISMATCH` (differing delivered value), `MISSING_ON_CHAIN` (expected settlement absent from covered window), and `DUPLICATE_INTERNAL` (repeated IDs within the source export).
   - Preserves `UNKNOWN` status whenever coverage is incomplete, an account is unverified, or multiple candidate operations exist without an operation ID to distinguish them.

3. **Standardized Input Adapters**:
   - Ingests canonical JSON and CSV files following an explicit, safe schema.
   - Ingests payment CSV exports from the Stellar Disbursement Platform (SDP 7.0.0 specification), discarding sensitive contact fields and enforcing partial-scope safety.

4. **Portable Offline Evidence and Replay**:
   - Captures self-contained evidence bundles (`--bundle`) containing internal records, observed on-chain operations, and coverage assertions.
   - Generates local SHA-256 cryptographic checksum proofs (`--verify auto`, creating `.proof.json`).
   - Supports offline replay (`--replay`), enabling third-party auditors to reproduce byte-identical reconciliation reports without database access or internet connectivity.

5. **Zero-Dependency Offline Dashboard**:
   - Provides a standalone HTML/JavaScript viewer (`dashboard/index.html`) that parses reports locally in any browser.
   - Supports search across internal and on-chain identifiers, candidate operation inspection, and clear visual discrepancy reporting.

## What LedgerParity Does Not Do (Explicit Boundaries)

- **No Transaction Submission or Signing**: LedgerParity does not hold private keys, manage custody, or submit transactions to the network.
- **No Path Payments or Soroban Events**: Ingestion is currently restricted to classic payment operations (`payment`). Path payments and Soroban smart contract invocations are outside current scope.
- **No Automated Dispute Resolution**: The tool reports discrepancies; it does not alter database states or trigger automated refunds.
- **No Dependency on Smart Contracts**: While an optional Soroban report-hash contract exists in a separate repository, the CLI does not invoke it. Reconciliation truth is determined by Horizon ledger state and local cryptographic hashing, not contract storage.
