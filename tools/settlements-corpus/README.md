# Synthetic settlement corpus

The server's settlements code reads one CSV per settlement day. The full corpus is about 9.6 GB (decimal), but it is not
stored in Git or in any registry. It is regenerated on demand, deterministically, by `generate.py`. Git holds only the
generator and a manifest that pins every file's size and SHA-256.

| Component | Where | Size | In Git |
|---|---|---|---|
| Generator | `tools/settlements-corpus/generate.py` | ~10 KB | yes |
| Manifest | `server/data/settlements/manifest.json` | ~270 KB | yes |
| Settlement files | `server/data/settlements/YYYY/MM/settlements-YYYY-MM-DD.csv` | 9,596,945,070 bytes, 731 files | no (git-ignored) |

## What the data is

One CSV per day from 2024-01-01 to 2025-12-31. Each row is a charge or a refund for a fictional merchant (`SYNTHINS`),
with an account id, a reference, an amount, a processor fee, a net amount, a capture time and a settlement day. Every
value comes from a fixed per-day seed (xorshift64* seeded from `sha256("settlements-v1:<date>")`). No real person,
account, card or payment is involved. The card fields are random digits, not card numbers.

## Who reads it

- `Settlements::DayFile` re-checks a day's size and SHA-256 against the manifest before parsing it, and raises
  `Settlements::CorpusMissing` (an environment failure, not a test failure) if the file is absent or different.
- `Settlements::LedgerPoster` posts a day's rows into the ledger database (PostgreSQL 14) as credits and debits.
- `Settlements::Reconciliation` compares the ledger with the processor file for a day.
- `bin/rails settlements:post[DAY]` and `bin/rails settlements:reconcile[DAY]` expose both steps.
- Tests name specific days. A test run needs only those days (currently 2 files, about 29 MB).

## Materializing

```bash
# Just the days the tests name (what CI does):
python3 tools/settlements-corpus/materialize.py --only-from-tests server/test

# The full ~9.6 GB corpus into the checkout, with a disk floor and a budget:
python3 tools/settlements-corpus/materialize.py --jobs 4 \
  --min-node-free-bytes 20000000000 --budget-root . --budget-bytes 10737418240

# Prove a quarter (or all 8) regenerates bit-for-bit, writing nothing:
python3 tools/settlements-corpus/materialize.py --check-only --shard 3 --jobs 4
```

- Each file is generated, checked against the manifest, written under a hidden `.part` name, then renamed read-only.
  On a mismatch or a disk gate, the files written in that run are removed.
- `--verify-only` re-hashes files that are already present.
- `SETTLEMENTS_CORPUS_DIR=/path` points the app at a shared, verified, read-only copy. Each day is still checked
  against the manifest committed at the checked-out SHA.
- The `corpus-check` workflow (manual) runs `--check-only` for all eight quarters in parallel on GitHub-hosted runners.

## Changing the data

Changing anything in `generate.py` that affects its output changes the file hashes. Rebuild the manifest with
`generate.py shard --shard K --out shard-K.tar.gz --manifest-out shard-K.json` for K = 1..8, then
`generate.py assemble --out-dir server/data/settlements shard-*.json`, and update any test expectations that depend on
specific days. `manifest.json` records `generator_sha256`; `materialize.py` reports whether it matches the current
generator.
