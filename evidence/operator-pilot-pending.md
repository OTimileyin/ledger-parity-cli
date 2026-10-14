# Operator Pilot Status: Pending External Data

Status: PENDING
Gate Classification: CONDITIONAL
Evaluation Date: October 14, 2026
Repository: LedgerParity/ledger-parity-cli

## 1. Pilot Execution Assessment

In accordance with Phase 2 project rules:
- No live operator, customer, enterprise partner, or production dataset has been fabricated.
- As of October 14, 2026, no consenting external payment operator has supplied an authorized, independently created, sanitized production export for live pilot execution.
- Therefore, the operator pilot execution step is preserved as **PENDING**, and the phase evaluation is **CONDITIONAL**.

## 2. Separation of Artificial Test Discrepancies from Genuine Operator Findings

It is essential to distinguish between synthetic regression fixtures and genuine operator findings:
- **Artificial Regression-Test Discrepancies**:
  - The deterministic CLI demo (`--demo`) produces 1 match, 4 discrepancies (`AMOUNT_MISMATCH`: 1, `MISSING_ON_CHAIN`: 1, `DUPLICATE_INTERNAL`: 2), and 3 unknowns (pending: 1, ambiguous candidate operations: 1, unclaimed on-chain: 1), exiting with code `3`.
  - The SDP test fixture in `examples/sdp/` produces 1 match, 1 amount discrepancy, and 1 unknown batch, exiting with code `3`.
  - These are synthetic test fixtures hardcoded to verify the parser, reconciler, and output formatting. They validate code execution paths, but they do NOT represent genuine operator findings or production validation.
- **Genuine Operator Findings**:
  - Real anomalies discovered during active operation of an enterprise payment pipeline (such as actual double payouts, unsubmitted batch drops, or rounding errors).
  - To date, **zero genuine operator findings** have been captured because no external operator export has been supplied.

## 3. Missing Requirements and External Dependencies

Execution of an authoritative real-world pilot requires the following external inputs from a consenting operator:

1. **Sanitized Application Payment Export**:
   - A genuine export from an active payment system (such as an SDP deployment, payment gateway database, or anchor service) covering an agreed-upon historical window.
   - Required fields from the operator: internal transaction UUIDs, actual disbursement amounts, asset types and issuer addresses, and internal settlement statuses.
   - PII redaction: Removal of recipient names, phone numbers, email addresses, and memos while preserving the 56-character Stellar public keys (`G...`) and transaction hashes.

2. **Operator Authorization and Scope Confirmation**:
   - Formal consent to run reconciliation against the operator's distribution account.
   - Independent declaration of sender account public keys, target network passphrase (Mainnet or Testnet), and settlement window boundaries.
   - Default completeness assertion: `complete: false`. To set `complete: true`, the operator must provide independent proof of exhaustive database extraction without filtering.

3. **Known Positive and Negative Controls**:
   - Operator-verified confirmation of at least 1 known settled payment (positive control) and at least 1 known operational discrepancy (negative control) to test classification fidelity.

4. **Measured Integration Metrics**:
   - Operator recording of actual integration time, setup friction, false positive rates, and operator review duration.

## 4. Playbook Findings Classification (Classes A through E)

In alignment with the Stellar Project Approval Playbook and review standards, project findings and gaps are classified into five distinct severity tiers:

| Class | Severity | Category | Playbook Definition | LedgerParity Finding / Status | Carried Phase |
|---|---|---|---|---|---|
| **Class A** | **Critical** | External Adoption | Defect, authorization bypass, or missing evidence that blocks submission approval. Must be resolved before full acceptance. | **External Operator Validation**: Tooling is verified on synthetic fixtures, but independent real-world operator validation remains pending. Gate is **CONDITIONAL**. | Phase 5 / Phase 6 |
| **Class B** | **High** | Data Ingestion | Major architectural risk or upcoming ecosystem deprecation requiring a migration plan. | **Horizon Ingestion Deprecation**: Horizon is slated for eventual deprecation in favor of Stellar RPC. Current RPC lacks long-term indexed payment history parity; a future ingestion connector strategy must be designed without breaking current Horizon stability. | Phase 3 |
| **Class C** | **Medium** | Scope Boundary | Bounded protocol capabilities that must be explicitly documented to prevent reviewer misunderstanding. | **Classic Payment Scope**: Ingestion is intentionally limited to classic `payment` operations. Path payments, Soroban contract balance events, and account creations/merges are outside core scope. | Phase 3 / Phase 4 |
| **Class D** | **Low** | Validation Ergonomics | Ergonomic improvements, edge-case input handling, or secondary validations. | **Public Key and Account Parsing**: Adding StrKey checksum validation on public keys in CLI config (currently length/prefix checked); full bidirectional multiplexed account (`M...`) equivalence. | Phase 4 |
| **Class E** | **Informational** | Scope Boundary | Documentation precision and repository boundary separation. | **Decoupled Soroban Registry**: The optional report-hash contract (`ledger-parity-contract`) must remain decoupled; documentation must reinforce that hash registration does not prove payment truth. | Phase 5 |

## 5. Conditional Gate Justification

Under the project assessment rubric:
- `PASS` strictly requires independent practical-use evidence from an external operator.
- `STOP` is triggered by architectural invalidity, fabricated evidence, or broken functionality.
- `CONDITIONAL` is the required outcome when the software implementation, tests, and verification procedures are fully reproducible and sound, but the external real-world operator validation remains dependent on a consenting external partner.

Because LedgerParity CLI, Core, and Connectors have passed all unit tests, static analysis, binary builds, offline demo runs, and byte-identical evidence replays without defects, the gate is classified as **CONDITIONAL**, with the named dependency being external operator data submission.
