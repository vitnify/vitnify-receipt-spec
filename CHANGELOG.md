# Changelog

Version files in this spec are **frozen once published**; a new version is a new
file. This log records the lineage.

## tier-1 digest v2 (regime binding) — 2026-08-20

The **tier-1 model-computation digest** advances to its own **v2** — a version line
distinct from the receipt format (both now read "v2", independently). See the updated
Two-tiers and Reference-value sections of [`vitnify-receipt-v2.md`](vitnify-receipt-v2.md).

- **Bound the numerical `regime`** into the digest under a new `"vitnify-receipt v2\x00"`
  domain. The engine previously had no recorded regime/arithmetic version, so a
  deliberate reduction change would break level-2 replay for every prior receipt with
  no way to distinguish "regime moved" from "tampered". A v2 digest now records which
  regime produced it (`vitni-regime-1`).
- **Tier-1 v1 frozen and retained.** `"vitnify-receipt v1\x00"` (no regime) still
  computes the historical anchor `9c0754…`; the same reference run reproduces it exactly.
- **New v2 anchor:** `ffebe862…9c88f` (TinyLlama-1.1B Q4_K_M, "Once upon a time,", 20
  new tokens, regime `vitni-regime-1`).
- **Clarified** that the shipped `vitni-receipt` binary commits an I/O pair under a
  fixed engine; per-op/activation records are an available mode it does not emit.

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
- **Made explicit:** a conformant verifier accepts **every published format
  version** and reconstructs each receipt's signed body from its own `v`, so a
  frozen format stays verifiable across verifier upgrades.
- **Tightened capability containment:** every `tool_call` must be within
  `capabilities` **or** a clean denial (decision `deny`, no `result`); the verifier
  fails closed on the decision label instead of trusting the word "allow", so a
  verifying receipt proves no ungranted tool executed.
- **Fail closed on unrecognised labels:** an unknown event `kind`, `decision`, or
  `sig_alg` is invalid, so a check that filters events by such a label cannot be
  sidestepped by relabelling.
- **Enforced vs observed containment:** a receipt with any `observed` tool decision
  is integrity-only for containment (`containment_enforced = false`); a containment
  proof requires `ok` and `containment_enforced`. Noted `program_hash` is
  caller-asserted and empty `model_digests` (hosted) is integrity-only.
- **Tier-1 unchanged at v2's release** (`"vitnify-receipt v1\x00"`, anchor `9c0754…f3b0f`).
  *Superseded 2026-08-20:* the tier-1 digest later advanced to its own v2 with a bound
  regime — see the entry above; tier-1 v1 stays frozen and reproduces `9c0754…`.

## vitnify-receipt v1 — 2026-08-19

[`vitnify-receipt-v1.md`](vitnify-receipt-v1.md) — frozen. The initial format:
BLAKE3 + ed25519, two-tier (a model-computation digest embedded in a signed
receipt), offline-verifiable.
