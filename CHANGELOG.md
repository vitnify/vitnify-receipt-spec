# Changelog

Version files in this spec are **frozen once published**; a new version is a new
file. This log records the lineage.

## vitnify-receipt v2 — 2026-08-19

[`vitnify-receipt-v2.md`](vitnify-receipt-v2.md) — current version.

- **Added** to the signed body: `issued_at` (issuer-asserted UTC), `nonce`, and
  `run_id`, so a receipt is time-placeable and unique — one run's receipt can no
  longer be presented as evidence for another.
- **Specified fail-closed verification:** a receipt with `sig_alg="none"`, an
  unknown algorithm, or keyless HMAC is never `ok`.
- **Specified capability containment:** every *allowed* `tool_call` must name a
  tool in the declared `capabilities`, so the receipt proves containment held.
- **Added optional signer pinning** (`pinned_pubkeys`) to the open-source verifier.
- **Added a Hosted models section:** hosted receipts are integrity-only (no tier-1
  digest); record provider identity; do not replay as a control.
- **Unchanged:** the tier-1 model-computation digest, its `"vitnify-receipt v1\x00"`
  domain separator, and the conformance anchor `9c0754…f3b0f`.

## vitnify-receipt v1 — 2026-08-19

[`vitnify-receipt-v1.md`](vitnify-receipt-v1.md) — frozen. The initial format:
BLAKE3 + ed25519, two-tier (a model-computation digest embedded in a signed
receipt), offline-verifiable.
