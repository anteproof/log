# Anteproof forecast log — anchors

This repository is the public, append-only record that Anteproof's forecast
log existed, in exactly its published form, on the dates it says it did.
It is written by an automated job once a day and by nothing else.

Every forecast Anteproof publishes is a row in a hash chain: each row's
`row_hash` is the SHA-256 of its immutable fields plus the previous row's
hash. That makes the log tamper-*evident* — but only if you can trust that
a given head hash existed before the forecasts it covers resolved. These
receipts make that trust unnecessary: each day's head hash is committed to
the Bitcoin blockchain and to an independent RFC 3161 timestamp authority,
neither of which Anteproof controls.

## What is in `receipts/`

For each UTC date `D`:

| file | what it is |
|---|---|
| `D.json` | the receipt: chain head hash, row count, latest `issued_at`, and the SHA-256 of the previous receipt (receipts chain too) |
| `D.json.ots` | OpenTimestamps proof that `sha256(D.json)` existed before a given Bitcoin block |
| `D.json.tsr` | RFC 3161 timestamp token over `sha256(D.json)` from a public TSA (DigiCert, or Sigstore as fallback) |

A receipt looks like:

```json
{
  "schema": "anteproof-receipt/1",
  "date": "2026-09-02",
  "head_hash": "…64 hex chars…",
  "head_id": 12345,
  "n_forecasts": 12345,
  "max_issued_at": "2026-09-02T00:11:52.019384+00:00",
  "prev_receipt": "2026-09-01.json",
  "prev_receipt_sha256": "…64 hex chars…",
  "generated_at": "2026-09-02T00:20:03.117205+00:00"
}
```

## Verifying — no Anteproof software required

You need `sha256sum` (or `shasum -a 256`), `openssl`, and for the Bitcoin
proof the OpenTimestamps client (`pip install opentimestamps-client`).

### 1. The RFC 3161 timestamp (seconds)

```sh
D=receipts/2026-09-02.json
openssl ts -verify -data "$D" -in "$D.tsr" -CAfile /etc/ssl/certs/ca-certificates.crt
openssl ts -reply -in "$D.tsr" -text | grep -E 'Time stamp|TSA|Policy'
```

`Verification: OK` means a DigiCert TSA signed a statement that this exact
file existed at the printed time. (Use your platform's CA bundle path;
`python -c 'import certifi; print(certifi.where())'` works anywhere. A
token from Sigstore's TSA instead chains to Sigstore's own root, which you
fetch from their TUF repository rather than a system bundle.)

### 2. The Bitcoin timestamp (needs one confirmation, typically a few hours after the receipt)

```sh
ots verify receipts/2026-09-02.json.ots
```

With a local Bitcoin node this prints `Success! Bitcoin block N attests
existence as of <date>`. Without one, run `ots --no-bitcoin verify …`: it
prints the block height and merkle root to check against any block
explorer. If it reports `Pending confirmation`, the proof has not been
upgraded yet — `ots upgrade` fetches the completed proof from the public
calendars, and this repo's next commit will carry it too.

### 3. The receipt chain

Each receipt names the previous one and carries its SHA-256:

```sh
sha256sum receipts/2026-09-01.json
grep prev_receipt_sha256 receipts/2026-09-02.json
```

Removing or rewriting any day's receipt breaks every later link.

### 4. The head against the live log

The public API exposes the chain itself:

- `GET /v1/chain/head` — the current `head_hash`, `head_id`, `n_forecasts`, `max_issued_at`
- `GET /v1/chain/rows?after_id=0&limit=1000` — every row's hashed fields with `prev_hash` and `row_hash`, in chain order

To recompute a hash: take the row's ten hashed fields (`question_id`,
`p`, `p_raw`, `issued_at`, `samples`, `model_config`,
`calibration_version`, `trace_uri`, `is_backtest`, `cutoff_at`) plus
`prev_hash` (hex string, or null for the first row), serialise them as
JSON with keys sorted, no whitespace (`,` and `:` separators), timestamps
as ISO-8601 in UTC exactly as the API returns them, and SHA-256 the UTF-8
bytes. In Python:

```python
import hashlib, json

payload = {k: row[k] for k in FIELDS} | {"prev_hash": row["prev_hash"]}
h = hashlib.sha256(json.dumps(payload, sort_keys=True, separators=(",", ":")).encode()).hexdigest()
assert h == row["row_hash"]
```

Walk the rows from id 1, checking each `prev_hash` equals the previous
`row_hash`, until you reach the receipt's `head_id`: its `row_hash` must
equal the receipt's `head_hash`. Every row you passed is therefore covered
by that receipt — it existed, with exactly those fields, no later than the
receipt's timestamps.

(The reference implementation of this walk is
`foresight_anchor.head.verify_chain` in the Anteproof source; floats are
serialised with Python's shortest round-trip repr, which is what the API
emits.)

## What this does and does not prove

It proves a forecast (its probability, the question it answers, when it
was issued) existed by a certain time. It does not by itself prove the
forecast was *good* — that is what the resolution record and the public
calibration history are for. Nor can a receipt cover rows appended after
it; those are covered by the next day's receipt.
