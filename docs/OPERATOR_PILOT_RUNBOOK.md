# Operator Pilot Runbook and Safe Input Contract

Document version: 1.0.0
Date: October 13, 2026
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
- Standard headers consumed: `ID`, `Amount`, `Asset.Code`, `Asset.Issuer`, `Status`, `CreatedAt`, `UpdatedAt`, `StellarTransactionID`, `ExternalPaymentID`.
- PII handling: SDP contact phone/email fields present in raw exports are strictly discarded by `pkg/sdp/parser.go`.
- Confidence flag: Because SDP exports can be filtered or partial, the CLI strictly forbids `target_app.complete = true` for SDP CSV exports. Any missing observation must remain `UNKNOWN` rather than assumed missing.

## 3. Pre-Reconciliation Scope Declaration

Before running reconciliation, the operator and evaluator must agree on:
1. **Network Passphrase**: Exact passphrase (`Public Global Stellar Network ; September 2015` or `Test SDF Network ; September 2015`).
2. **Account Scope**: Explicit list of issuing or distribution public keys to ingest from Horizon.
3. **Time Window**: Closed UTC interval (`start` to `end`) with an explicit tolerance buffer (`timeframe_tolerance_sec`, typically 300 to 600 seconds).
4. **Completeness Assertion**: Whether the export represents all payments made by the sender during the window (`complete: true`) or an arbitrary subset (`complete: false`).

## 4. Test Controls and Baselines

To ensure the reconciliation engine behaves accurately on real data, every pilot run must include:
1. **Positive Controls**:
   At least 3 payments known to have settled successfully on-chain with matching amount, asset, recipient, and timestamp. Expected classification: `EXACT`, discrepancy `NONE`.
2. **Negative Controls (Intentional Discrepancies)**:
   - One record with a known amount discrepancy (e.g., internal says 100, on-chain says 95). Expected classification: `DISCREPANCY`, discrepancy `AMOUNT_MISMATCH`.
   - One record marked completed internally that was intentionally never submitted or submitted to a different recipient. Expected classification: `DISCREPANCY`, discrepancy `MISSING_ON_CHAIN`.
3. **Ambiguity Controls**:
   Two batch payments with identical amounts, sender, recipient, and timestamps lacking explicit operation IDs. Expected classification: `UNKNOWN`, discrepancy `UNRESOLVED` (with candidate operation IDs listed).

## 5. Execution Commands and Procedure

### Step 1: Prepare the Pilot Configuration

Create `pilot-config.json`:
```json
{
  "target_app": {
    "name": "pilot-operator-payouts",
    "format": "json",
    "source_path": "sanitized-export.json",
    "complete": true
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
- Exit `0`: All internal records matched on-chain observations exactly with zero discrepancies and zero unknowns. Complete parity achieved.
- Exit `1`: Fatal configuration, input validation, network timeout, or file system error. Execution aborted.
- Exit `2`: Unused / reserved.
- Exit `3`: Reconciliation completed successfully, but discrepancies (`AMOUNT_MISMATCH`, `MISSING_ON_CHAIN`, `DUPLICATE_INTERNAL`) or `UNKNOWN` states were detected. This is the expected exit code when discrepancies or partial coverage are present.

### Step 4: Verify Offline Evidence and Replay

To verify that the evidence bundle is portable and auditable by third-party reviewers:

```bash
# 1. Verify SHA-256 proof file
ledger-parity --verify-check pilot-report.json.proof.json pilot-report.json

# 2. Replay evidence in an air-gapped / offline environment
ledger-parity --replay pilot-evidence.json --format json --out replayed-report.json

# 3. Confirm byte-identical output
diff -u pilot-report.json replayed-report.json
```

### Step 5: Visual Inspection via Dashboard

Open `dashboard/index.html` in a web browser and load `pilot-report.json`. The operator must verify:
1. Total internal payment count matches source export row count.
2. Total on-chain operation count matches Horizon ledger query count.
3. Discrepancy counts accurately reflect known positive and negative controls.
4. Candidate operation IDs resolve correctly for all matched payments.

## 6. Operator Confirmation Checklist

Before concluding the pilot, the operator must complete and sign off on this checklist:
- [ ] No customer PII or confidential credentials were included in the export file.
- [ ] Network passphrase strictly matched the target ledger.
- [ ] Source account public keys and time window were verified independently.
- [ ] Known positive controls were classified as `EXACT`.
- [ ] Known negative controls were classified as `DISCREPANCY`.
- [ ] Offline replay produced byte-identical output with matching SHA-256 proof.
- [ ] Total integration and execution time was recorded.
- [ ] False positive and false negative rates were documented.
