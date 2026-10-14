# Operator Pilot Runbook and Safe Input Contract

Document version: 1.1.0
Date: October 14, 2026
Repository: LedgerParity/ledger-parity-cli

## 1. Overview and Scope

This runbook defines the operational procedure for executing an external operator reconciliation pilot using LedgerParity CLI. It reconciles the requirements in `docs/OPERATOR_VALIDATION.md` and `docs/SDP.md` with the underlying Go source code in `ledger-parity-core` and `ledger-parity-connectors`.

The reconciliation tool is read-only. It inspects Stellar classic payment operations and compares them with internal application payment records. It never signs transactions, never requests private keys, and never initiates funds movement.

## 2. Minimum Safe Sanitized Export Contract

To prevent leaks of Customer Proprietary Network Information (CPNI), Personally Identifiable Information (PII), or confidential banking details, the operator must export ONLY the following technical settlement fields. All customer names, email addresses, phone numbers, bank account numbers, physical addresses, and memos must be omitted or redacted.

### Canonical JSON Export Schema

A JSON array of payment records matching this exact contract:

```json
[
  {
    "id": "internal-payment-uuid-or-id",
    "network": "Public Global Stellar Network ; September 2015",
    "operation_type": "payment",
    "sender": "GXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX",
    "recipient": "GYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYY",
    "amount": "150.5000000",
    "asset": "USDC",
    "asset_type": "credit_alphanum4",
    "asset_issuer": "GA5ZSEJYB37JRC5AVCIA5MOP4RHTM335X2KGX3IHOJAPP5RE34K4KZVN",
    "status": "completed",
    "timestamp": "2026-09-01T12:00:00Z",
    "operation_id": "19878182387720193",
    "reference_id": "4c94f1c1a9..."
  }
]
```

### Canonical CSV Export Schema

A UTF-8 CSV with the following mandatory header order or names:

```csv
id,network,operation_type,sender,recipient,amount,asset,asset_type,asset_issuer,status,timestamp,operation_id,reference_id
pay-001,Public Global Stellar Network ; September 2015,payment,GXXX...,GYYY...,150.5000000,USDC,credit_alphanum4,GA5Z...,completed,2026-09-01T12:00:00Z,19878182387720193,4c94f1c1a9...
```

### Stellar Disbursement Platform (SDP 7.0.0) Payment CSV Contract

For operators running SDP release 7.0.0 (revision `14704274467d267b6c6679052d7251ef6050d118`):
- Format flag: `target_app.format = "sdp-csv"`
- Implementation reference: Parsing logic resides in `github.com/LedgerParity/ledger-parity-connectors/pkg/sdp` (in repository `ledger-parity-connectors`), which is invoked by `ledger-parity-cli` in `cmd/ledger-parity/main.go`.
- Standard headers consumed: `ID`, `Amount`, `Asset.Code`, `Asset.Issuer`, `Status`, `CreatedAt`, `UpdatedAt`, `StellarTransactionID`, `ExternalPaymentID`.
- PII handling: SDP contact phone/email fields present in raw exports are strictly discarded during import.
- Completeness safety: Because SDP exports can be filtered or partial, the CLI strictly forbids `target_app.complete = true` for SDP CSV exports. Any missing observation remains `UNKNOWN` rather than assumed missing.

## 3. Pre-Reconciliation Scope and Completeness Declaration

Before running reconciliation, the operator and evaluator must agree on:
1. **Network Passphrase**: Exact passphrase (`Public Global Stellar Network ; September 2015` or `Test SDF Network ; September 2015`).
2. **Account Scope**: Explicit list of issuing or distribution public keys to ingest from Horizon.
3. **Time Window**: Closed UTC interval (`start` to `end`) with an explicit tolerance buffer (`timeframe_tolerance_sec`, typically 300 to 600 seconds).
4. **Completeness Assertion (`complete`)**:
   - **Default: `false`**. Unobserved payments are classified conservatively as `UNKNOWN`.
   - **Requirements to set `complete: true`**:
     Setting `complete: true` asserts that the export contains every single payment transaction conducted by the subject account(s) within the declared interval, with zero omissions. To legitimately assert completeness, the operator must provide:
     - Verified audit of the database extraction query confirming that all payment state transitions for the account were extracted without user-level, status-level, or batch-level filtering.
     - Contiguous internal transaction sequence verification or external source-funding bank balance reconciliation.
     - Confirmation that the export format permits complete assertions (for SDP CSV exports, `complete: true` is strictly rejected by the parser).

## 4. Test Controls and Expected Classifications

To verify that the engine behaves accurately on real data, every pilot run must establish clear control baselines. Classifications depend strictly on timestamp tolerance, explicit operation identity, and coverage completeness:

### 1. Positive Controls
- **Exact Timestamp Match**:
  Internal record matches on-chain payment in amount, asset, sender, recipient, and the timestamp matches exactly.
  Expected classification: Status `EXACT`, discrepancy `NONE`.
- **Tolerant Timestamp Match**:
  Internal record matches on-chain payment in amount, asset, sender, and recipient, but timestamps differ within `timeframe_tolerance_sec`.
  Expected classification: Status `TOLERANT`, discrepancy `NONE`, with `time_delta_sec` populated.
- **Incomplete Coverage Effect on Positive Controls**:
  If explicit `operation_id` is omitted AND the coverage window is incomplete (`complete: false`), the engine classifies the candidate observation as `UNKNOWN` ("Candidate observed but full matching window is unproven; supply operation ID"). Therefore, under default incomplete coverage (`complete: false`), positive controls require an explicit `operation_id` to achieve `EXACT` or `TOLERANT` matching.

### 2. Negative Controls (Intentional Discrepancies)
- **Delivered Amount Mismatch**:
  Internal record claims an amount that differs from the on-chain delivered amount.
  - If explicit identity (`operation_id` or `reference_id` transaction hash) is supplied: Expected classification: Status `DISCREPANCY`, discrepancy `AMOUNT_MISMATCH`.
  - If explicit identity is omitted: Expected classification: Status `UNKNOWN`, discrepancy `UNRESOLVED` ("Nearby payment has a different amount; no explicit identity proves its relationship").
- **Missing On-Chain Payment**:
  Internal record marks a payment completed, but no corresponding operation exists on-chain.
  - If coverage is complete (`complete: true`): Expected classification: Status `DISCREPANCY`, discrepancy `MISSING_ON_CHAIN`.
  - If coverage is incomplete (`complete: false`, the default): Expected classification: Status `UNKNOWN`, discrepancy `UNRESOLVED` ("No settlement found; status or coverage does not justify absence").
- **Duplicate Internal Submission**:
  The export contains multiple internal records sharing the same internal ID.
  Expected classification: Status `DISCREPANCY`, discrepancy `DUPLICATE_INTERNAL`.

### 3. Ambiguity Controls
- **Multiple Candidate Operations**:
  Two or more on-chain operations match the payment parameters within the timeframe tolerance, and the internal record lacks an `operation_id`.
  Expected classification: Status `UNKNOWN`, discrepancy `UNRESOLVED` ("Multiple candidate operations; supply an operation ID").
- **Multiple Internal Claims**:
  Two or more internal records claim the same on-chain operation.
  Expected classification: Status `UNKNOWN`, discrepancy `UNRESOLVED` ("Multiple internal records claim this operation").

### 4. Separation of Artificial Test Discrepancies from Operator Findings
The discrepancies observed in the built-in offline demo (`--demo`) and SDP test fixtures (`examples/sdp/`) are artificial regression fixtures engineered to verify engine logic. They must never be described as genuine operator findings. Genuine operator findings can only be documented after a consenting operator runs their actual export through this runbook.

## 5. Execution Commands and Procedure

### Step 1: Prepare the Pilot Configuration

Create `pilot-config.json` with `complete: false` as the default:
```json
{
  "target_app": {
    "name": "pilot-operator-payouts",
    "format": "json",
    "source_path": "sanitized-export.json",
    "complete": false
  },
  "stellar": {
    "network": "Public Global Stellar Network ; September 2015",
    "accounts": [
      "GOPERATOR_DISTRIBUTION_ACCOUNT_PUBKEY"
    ]
  },
  "reconciliation": {
    "start": "2026-10-01T00:00:00Z",
    "end": "2026-10-01T23:59:59Z",
    "timeframe_tolerance_sec": 600
  }
}
```

### Step 2: Execute Reconciliation and Bundle Evidence

Run the LedgerParity binary against Horizon:

```bash
ledger-parity \
  --config pilot-config.json \
  --format both \
  --bundle pilot-evidence.json \
  --out pilot-report.json \
  --verify auto
```

### Step 3: Interpret CLI Exit Codes

The tool outputs specific exit codes that must be checked by the operator:
- Exit `0`: No discrepancies or unknowns (`TotalDiscrepancies == 0 && TotalUnknown == 0`). Full parity achieved.
- Exit `1`: Execution error (configuration error, validation failure, network timeout, or file system failure).
- Exit `2`: Discrepancies without unknowns (`TotalDiscrepancies > 0 && TotalUnknown == 0`).
- Exit `3`: One or more unknowns, including mixed results (`TotalUnknown > 0`).

### Step 4: Verify Offline Evidence and Replay

To verify that the evidence bundle is portable and auditable by third-party reviewers:

```bash
# 1. Verify SHA-256 local proof file integrity
ledger-parity --verify-check pilot-report.json.proof.json pilot-report.json

# 2. Replay evidence in an air-gapped or offline environment
ledger-parity --replay pilot-evidence.json --format json --out replayed-report.json

# 3. Confirm byte-identical output
diff -u pilot-report.json replayed-report.json
```

Note: The local `.proof.json` file authenticates only the byte-level integrity of the local report against post-run file modification. It does not authenticate the underlying truth of internal payment records or cryptographically verify the blockchain.

### Step 5: Visual Inspection via Dashboard

Open `dashboard/index.html` in a web browser and load `pilot-report.json`. The operator must verify:
1. Total internal payment count matches source export row count.
2. Total on-chain operation count matches Horizon ledger query count.
3. Discrepancy counts accurately reflect known positive and negative controls.
4. Candidate operation IDs resolve correctly for all matched payments.

## 6. Operator Confirmation Checklist

Before concluding the pilot, the operator must complete and sign off on this checklist:
- [ ] No customer PII, memos, or confidential credentials were included in the export file.
- [ ] Network passphrase strictly matched the target ledger.
- [ ] Source account public keys and time window were verified independently.
- [ ] Known positive controls were classified as `EXACT` or `TOLERANT` (with explicit operation IDs when coverage is incomplete).
- [ ] Known negative controls were classified as `DISCREPANCY` (or `UNKNOWN` when explicit IDs/complete coverage were omitted).
- [ ] Offline replay produced byte-identical output with matching SHA-256 local proof.
- [ ] Total integration and execution time was recorded.
- [ ] False positive and false negative rates were documented.
- [ ] No synthetic regression fixtures were cited as live operator findings.
