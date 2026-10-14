# LedgerParity Product Statement

Document version: 1.1.0
Evaluation Date: October 14, 2026
Target System: LedgerParity CLI, Core, and Connectors

## What LedgerParity Is

LedgerParity is a dedicated, read-only reconciliation tool that compares internal application payment records against on-chain Stellar classic payment operations. It identifies matching settlements, flags accounting discrepancies, and isolates ambiguous or unobserved transactions using exact integer-stroop arithmetic and deterministic classification rules.

## Implemented Capabilities

1. **Exact Classic Payment Reconciliation**:
   - Matches internal payment records against Horizon ledger operations using exact decimal amounts (7 decimal places, stroop-level precision).
   - Enforces triple asset identity (AssetType, AssetCode, and AssetIssuer), preventing spoofed or counterfeit token matches.
   - Evaluates network passphrases strictly, ensuring Testnet operations are never confused with Mainnet settlements.
   - Recommends 64-bit Stellar Operation IDs for disambiguation in multi-operation transactions and under incomplete coverage windows, rather than universally requiring them for all single classic payments.

2. **Conservative Discrepancy Classification**:
   - Classifies records as `EXACT` when amount, asset, sender, recipient, and exact timestamp match.
   - Classifies records as `TOLERANT` when identity matches and timestamps align within the configured timeframe tolerance.
   - Identifies discrepancies: `AMOUNT_MISMATCH` (differing delivered value when identity is established), `MISSING_ON_CHAIN` (expected settlement absent from an asserted complete window), and `DUPLICATE_INTERNAL` (repeated IDs within the source export).
   - Preserves `UNKNOWN` status whenever coverage is incomplete, an account is unverified, an amount mismatch lacks explicit transaction identity, or multiple candidate operations exist without an operation ID to distinguish them.

3. **Standardized Input Adapters**:
   - Ingests canonical JSON and CSV files following an explicit, safe schema.
   - Ingests payment CSV exports from the Stellar Disbursement Platform (SDP 7.0.0 specification) via `ledger-parity-connectors/pkg/sdp`, discarding sensitive contact fields and enforcing partial-scope safety.

4. **Portable Offline Evidence and Replay**:
   - Captures self-contained evidence bundles (`--bundle`) containing internal records, observed on-chain operations, and coverage assertions.
   - Generates local SHA-256 checksum proof files (`--verify auto`, creating `.proof.json`).
   - Supports offline replay (`--replay`), enabling third-party auditors to reproduce byte-identical reconciliation reports without database access or internet connectivity.

5. **Deterministic Exit Codes**:
   - Exit `0`: No discrepancies or unknowns (`TotalDiscrepancies == 0 && TotalUnknown == 0`). Full parity.
   - Exit `1`: Execution error (configuration, validation, network, or file system failure).
   - Exit `2`: Discrepancies without unknowns (`TotalDiscrepancies > 0 && TotalUnknown == 0`).
   - Exit `3`: One or more unknowns, including mixed results (`TotalUnknown > 0`).

6. **Zero-Dependency Offline Dashboard**:
   - Provides a standalone HTML/JavaScript viewer (`dashboard/index.html`) that parses reports locally in any browser.
   - Supports search across internal and on-chain identifiers, candidate operation inspection, and clear visual discrepancy reporting.

## What LedgerParity Does Not Do (Explicit Boundaries)

- **No Transaction Submission or Signing**: LedgerParity does not hold private keys, manage custody, or submit transactions to the network.
- **No Path Payments or Soroban Events**: Ingestion is currently restricted to classic payment operations (`payment`). Path payments and Soroban smart contract invocations are outside current scope.
- **No Cryptographic Blockchain Verification**: LedgerParity queries Horizon HTTP endpoints. Horizon is an off-chain API indexing service, not an infallible cryptographic oracle. Local `.proof.json` files authenticate only local report file integrity against modification; they do not authenticate the truth of the underlying payment records or cryptographically verify the blockchain.
- **No Automated Dispute Resolution**: The tool reports discrepancies; it does not alter database states or trigger automated refunds.
- **No Dependency on Smart Contracts**: While an optional Soroban report-hash contract exists in a separate repository (`ledger-parity-contract`), the CLI does not invoke it. Hash registration on-chain proves only an authorized digest commitment, not the factual truth of underlying payment data.
