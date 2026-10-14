# Operator validation guide

This guide explains how a real Stellar payment operator can validate
LedgerParity against their own data. This is the final acceptance gate:
proof that the tool works on genuine operator exports, not just synthetic
fixtures.

## What the operator needs to provide

### 1. Sanitized application export

A JSON or CSV file containing your application's payment records. Each record
must include:

**Required fields (JSON array of objects or CSV with these headers):**

| Field | Description |
|---|---|
| `id` | Unique record identifier (your internal ID) |
| `network` | Exact network passphrase (e.g., `"Test SDF Network ; September 2015"` or `"Public Global Stellar Network ; September 2015"`) |
| `operation_type` | Must be `"payment"` |
| `sender` | Sender's Stellar public key (starting with `G`) |
| `recipient` | Recipient's Stellar public key (starting with `G`) |
| `amount` | Positive decimal string (e.g., `"100.50"`) |
| `asset` | Asset code (e.g., `"XLM"`, `"USDC"`) |
| `asset_type` | `"native"` for XLM, `"credit_alphanum4"` or `"credit_alphanum12"` for tokens |
| `status` | `"completed"`, `"success"`, or `"settled"` (case-insensitive) |
| `timestamp` | RFC3339 timestamp (e.g., `"2026-09-01T12:00:00Z"`) |

**Optional fields:**

| Field | Description |
|---|---|
| `asset_issuer` | Issuer public key for credit assets (required if `asset_type` is not `"native"`) |
| `operation_id` | Operation ID (recommended for disambiguation; not universally required for unique payments under complete coverage) |
| `reference_id` | Transaction hash (not memo) |
| `settlement_start` | Start of explicit settlement interval (RFC3339) |
| `settlement_end` | End of explicit settlement interval (RFC3339) |
| `business_reference` | Application-specific reference (separate from `reference_id`) |

**Example (JSON):**

```json
[
  {
    "id": "app-tx-001",
    "network": "Test SDF Network ; September 2015",
    "operation_type": "payment",
    "sender": "GABC...",
    "recipient": "GDEF...",
    "amount": "250.0000000",
    "asset": "USDC",
    "asset_type": "credit_alphanum4",
    "asset_issuer": "Gxxx...",
    "status": "completed",
    "timestamp": "2026-09-01T12:00:00Z"
  }
]
```

**Example (CSV):**

```csv
id,network,operation_type,sender,recipient,amount,asset,asset_type,asset_issuer,status,timestamp
app-tx-001,Test SDF Network ; September 2015,payment,GABC...,GDEF...,250.0000000,USDC,credit_alphanum4,Gxxx...,completed,2026-09-01T12:00:00Z
```

### 2. Configuration

Provide the time window and accounts to reconcile:

```json
{
  "target_app": {
    "name": "your-app-name",
    "format": "json",
    "source_path": "path/to/your-export.json",
    "complete": false
  },
  "stellar": {
    "network": "your-network-passphrase",
    "accounts": ["YOUR_SENDER_ACCOUNT_G..."]
  },
  "reconciliation": {
    "start": "2026-09-01T00:00:00Z",
    "end": "2026-09-02T00:00:00Z",
    "timeframe_tolerance_sec": 600
  }
}
```

> **Note on `complete` assertion:**
> The default completeness assertion is `complete: false`, which conservatively treats unobserved payments as `UNKNOWN`. Setting `complete: true` asserts that the export contains every single payment transaction conducted by the subject account(s) within the declared interval, with zero omissions. To legitimately assert completeness, the operator must provide independent proof of exhaustive database extraction confirming that all payment state transitions for the account were extracted without user-level, status-level, or batch-level filtering.

### 3. Known discrepancy (optional but valuable)

If you already know of a specific discrepancy (e.g., a missed settlement,
incorrect amount, or duplicate record), note it separately. This helps verify
that LedgerParity catches real issues.

## How to run the validation

### Step 1: Export your data

Export your application's payment records for the agreed time window. Sanitize
any sensitive fields (customer names, memos, etc.) but preserve all
Stellar-relevant identity fields exactly.

### Step 2: Share the export

Send the export file and config to the LedgerParity maintainer. The export
should be a clean file with no application secrets, API keys, or customer PII.

### Step 3: Run reconciliation

The maintainer will run:

```sh
ledger-parity --config operator-config.json \
  --bundle operator-evidence.json --out operator-report.json
```

### Step 4: Review results

The report will show:
- **Matches**: payments found in both your export and on-chain
- **Discrepancies**: payments where amounts, assets, or identity differ
- **Unknowns**: payments that couldn't be verified (incomplete coverage,
  ambiguous candidates, etc.)

### Step 5: Verify a known discrepancy

If you noted a known discrepancy, confirm that LedgerParity identified it.
This is the key validation: the tool catches real issues, not just synthetic
ones.

## What this proves

- LedgerParity works on real operator data, not just test fixtures
- The reconciliation engine handles genuine export formats
- The tool correctly identifies matches, discrepancies, and unknowns
- The evidence bundle is reproducible and auditable

## Limitations

- This is read-only validation; LedgerParity never signs or moves funds
- Coverage trusts the operator's export completeness; it does not prove
  cryptographic history
- A single operator validation does not prove universal applicability
- The operator's export must be sanitized to remove PII before sharing

## Contact

To participate in operator validation, open a GitHub issue with the label
`operator-validation` or contact the maintainer directly. Provide:

1. Your role (payment operator, exchange, issuer, etc.)
2. Network (testnet or mainnet)
3. Approximate number of payments in the time window
4. Whether you have a known discrepancy to verify
