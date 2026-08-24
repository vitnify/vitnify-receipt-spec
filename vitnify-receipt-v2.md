# vitnify-receipt v2

The canonical format for a vitnify execution receipt: one signed, self-verifying
object that binds an agent run — its computation, authority, actions, order, and
identity — so an action cannot be detached from what produced it.

Everything hashes with **BLAKE3**. Signatures are **ed25519** with an embedded
public key, so anyone can verify a receipt offline with no model, no network, and
no secret.

> **Change from v1.** v2 adds `issued_at`, `nonce`, and `run_id` to the signed body
> so a receipt is time-placeable and unique (one run's receipt can no longer stand
> in for another's), specifies **fail-closed** verification and capability
> containment, and defines how a **hosted** model is recorded. Receipt-format v1
> remains frozen and valid for receipts issued under it.
>
> The tier-1 model-computation digest is a **distinct, lower-level version line**. It
> has advanced to its own **v2**, which binds the numerical *regime* — the reduction
> contract the forward pass ran under (see Two tiers) — so a digest records which
> arithmetic produced it, and a deliberate regime change is distinguishable from
> tampering. Tier-1 v1 stays frozen: the historical anchor `9c0754…` still reproduces
> under it.

---

## Two tiers

**Tier 1 — model-computation digest** (produced by the `vitni-tensor` engine).
The per-step commitment to what the model actually computed:

```
model_digest = BLAKE3( "vitnify-receipt v2\x00"
                       || LEB128(|regime|)        || regime
                       || LEB128(n_inputs)        || inputs
                       || LEB128(n_outputs)       || outputs
                       || LEB128(n_ops)           || per-op records
                       || LEB128(n_activations)   || activation records
                       || LEB128(n_interventions) || intervention records )
```

Fields are length-prefixed so concatenation is injective. `regime` (e.g.
`vitni-regime-2`) names the numerical reduction contract the forward pass ran under;
binding it lets a verifier distinguish a **deliberate regime change** ("cannot replay
under this engine") from tampering. Under a fixed regime the digest is reproducible
**bit-for-bit across CPU vendors and instruction sets**.

What the shipped `vitni-receipt` binary commits is an **I/O pair under a fixed
engine**: `inputs = (model_id, weights_hash, arch_hash, prompt_tokens, n_new_tokens)`,
`outputs = (output_tokens, output_tokens_hash)`, with `n_ops = 0`. The per-op /
activation / intervention records are an available mode the shipped binary does not
emit — so a different implementation producing the same tokens under the same regime
yields the same digest.

The tier-1 digest is versioned independently of the receipt format. Tier-1 **v1**
(`"vitnify-receipt v1\x00"`, no regime) is frozen and retained so pre-regime receipts —
including the anchor `9c0754…` — stay reproducible.

**Tier 2 — execution receipt** (produced by the SDK). The full agent-run object,
which *embeds* the tier-1 digests.

---

## The receipt body

The signed digest is `BLAKE3(canonical_json(body))`, where `body` is:

| field | meaning |
|---|---|
| `v` | `"vitnify-receipt v2"` |
| `program_hash` | hash of the exact agent program that ran |
| `capabilities` | sorted list of granted capabilities |
| `event_root` | BLAKE3 Merkle root of the event log (inclusion proofs) |
| `n_events` | event count |
| `head_hash` | head of the hash-chained event log |
| `model_digests` | ordered tier-1 digests, one per model step |
| `issued_at` | ISO-8601 UTC time the receipt was issued (issuer-asserted) |
| `nonce` | random per-receipt value; makes every receipt unique |
| `run_id` | identifier for this run; distinct runs get distinct receipts |

`canonical_json` = UTF-8 JSON, keys sorted, no whitespace.

`issued_at` and `program_hash` are **caller/issuer-asserted**. `issued_at` places
the receipt in time and defeats silent replay, but it is not a trusted timestamp —
non-repudiable time requires an RFC 3161 token or the Verification Authority
countersignature. `program_hash` binds whatever string the issuer supplies; it does
not itself hash the agent's code. Likewise, `model_digests` is empty for a hosted or
non-deterministic backend — an integrity-only receipt that binds the transcript, not
the computation.

### Signature envelope
`sig` (ed25519 over the 32-byte receipt digest), `sig_alg` = `"ed25519"`,
`pubkey` (raw ed25519 public key, hex). HMAC (`sig_alg` = `"hmac-blake2b"`) is a
fallback only. `sig_alg` = `"none"` is **not verifiable** and never yields a valid
verdict (see Verification).

---

## The event log

Every nondeterministic input/output is one canonical, hash-linked `Event`:

```
Event = { i, kind, payload, prev }
hash  = BLAKE3(canonical_json(Event))          # prev = previous event's hash
```

`kind` ∈ { `llm_call`, `tool_call`, `entropy`, `agent_step` }. An `llm_call`
payload carries `{prompt_hash, tokens, seed, model_digest}` — so committing the
event log (via `event_root` + `head_hash`) transitively binds every model step's
computation into the receipt. It **should** also carry `regime` (the tier-1 numerical
regime, e.g. `vitni-regime-2`) and `weights_hash` (the BLAKE3 of the model weights) — the
two engine inputs the `model_digest` depends on but that cannot be read back out of it.
Both are already bound (they are folded into the digest); recording them as payload
fields makes them **readable** too, so a level-2 verifier can report a **regime mismatch**
("issued under regime-1, this engine is regime-2") or **wrong weights** rather than an
opaque digest mismatch indistinguishable from tampering. For a hosted model it may also
carry a `provider` object (see Hosted models).

---

## Verification

Verification **fails closed**: a receipt that cannot be cryptographically verified
never returns a valid verdict. It also fails closed on any **unrecognised
self-declared label** — an unknown event `kind`, `decision`, or `sig_alg` is invalid
— so a check that filters events by such a label can never be sidestepped by
relabelling.

A conformant verifier accepts **every published format version** and reconstructs
each receipt's signed body from its own `v` (v1 binds seven fields; v2 adds the
three above). A frozen format must stay verifiable across verifier upgrades — a
receipt that a newer verifier can't read defeats "verify it years later."

**Level 1 — integrity (no model).**
1. Recompute `event_root`, `head_hash`, `n_events`, and `model_digests` from the
   raw events and confirm they match the receipt.
2. **Signature.** Verify the ed25519 signature against the embedded public key. A
   receipt with `sig_alg` = `"none"`, an unknown algorithm, or an HMAC receipt
   presented without its key is **invalid** — an unverifiable receipt is never
   `ok`.
3. **Capability containment.** Every `tool_call` must name a tool in `capabilities`
   **or** be a clean denial — decision `deny` (case/space-insensitive) with no
   `result`. An ungranted tool that is not cleanly denied — any other decision
   string, or one carrying a result — is invalid. The verifier fails closed on the
   decision label, so the receipt *proves* no ungranted tool executed rather than
   trusting the word "allow".

   **Enforced vs observed.** A decision of `allow`/`deny` was *gated* by the
   capability wall; a decision of `observed` was *recorded* by a passive adapter,
   not enforced. A receipt containing any observed decision is integrity-only for
   containment — the verifier reports `containment_enforced = false` — so it is a
   valid transcript, not proof that containment was applied. A containment *proof*
   requires `ok` **and** `containment_enforced`.

Rejects any edit, reorder, truncation, forgery, or out-of-policy action in the
transcript.

**Level 2 — recomputation (needs the weights).** Re-run each `llm_call` through
`vitni-tensor` and confirm every `model_digest` reproduces bit-for-bit. Available
for local weights only (see Hosted models). Before recomputing, check the payload's
`regime` against the verifying engine's and the payload's `weights_hash` against the
loaded weights: if either differs, report a **regime mismatch** or **wrong weights** —
the receipt was issued under a different numerical contract or against different weights,
which this engine cannot reproduce — rather than a bare digest mismatch, which a regime
change, a weights change, and tampering would all produce.

**Integrity vs. authority — a split verdict.** Level 1 answers two distinct questions,
and a verifier reports them separately rather than collapsing them into one boolean:

- `integrity_ok` — is the transcript internally consistent and validly signed by
  *whoever* signed it (steps 1–3 above)? Computable by **anyone, offline, with no secret**.
- `authority_ok` — was the signer an *approved runtime*? An embedded key proves signer
  *continuity*, not authority, so this needs a trust root and is `true` / `false` / `null`.
  **Signer pinning stays optional:** a verifier MAY supply an allow-list of trusted ed25519
  keys, in which case `authority_ok` is whether the receipt's key is on it; with no
  allow-list it is `null` — **unestablished**, a state a stranger offline can never resolve
  and MUST be told plainly, never as a bare `false` indistinguishable from a forgery.

`ok` = `integrity_ok` **and** an authorised signer. A verifier with no trust root reports
`integrity_ok = true`, `authority_ok = null`: the receipt is verifiable, its provenance
simply unanchored. Pinning closes the re-signing gap (below) in the open-source verifier,
with no managed service.

---

## Hosted models

A hosted API (OpenAI, Anthropic, Gemini, …) returns no reproducible computation —
at most truncated top-N logprobs — so there is **no tier-1 digest** to bind:
`model_digests` is empty (or self-asserted). A receipt over a hosted run is
therefore **integrity-only**: it binds the transcript, the capabilities, and the
identity, but not the model's computation.

Record the provider so drift is not mistaken for tampering. The `llm_call`
`provider` object should carry `{provider, model_version, system_fingerprint,
response_id}`; binding it means a later mismatch is attributable to a backend or
version change rather than forgery.

**Do not replay a hosted receipt as a control.** Replay would re-call the provider,
whose output is not bit-reproducible; a single drifted token would fail
verification identically to tampering. Level 2 applies to local weights only.

---

## Trust boundary (honest limit)

An embedded ed25519 key proves **integrity and signer continuity**, not that the
signer was an authorised runtime. A receipt re-signed by a different key still
self-verifies. Binding execution *provenance* requires a pinned trust anchor — a
known key (supply `pinned_pubkeys` to the verifier), or a TPM/enclave. A
**vitnify-verified** receipt is one countersigned by the official vitnify
Verification Authority against such an anchor.

---

## Reference value

A conformance anchor for the tier-1 model-computation digest, reproducible with the
`vitni-receipt` binary from the `vitni-tensor` engine:

```
model            TinyLlama-1.1B-Chat, Q4_K_M GGUF
model_id         tinyllama-1.1b-chat-Q4_K_M
weights_hash     ab7591e1ec49cb5dce27865d31d436091a433a3e807e9920a572c42dda294b0d
prompt           [1, 9038, 2501, 263, 931, 29892]      ("Once upon a time,")
n_new            20
regime           vitni-regime-2
model_digest     7a2e28c94c351b8e303e0f8c12a719b8aaa2c94e7233a96b4348a7dc6d341730   (tier-1 v2, current)
model_digest_v1  9c0754458633e863e0fb5bb2bd00df0d8b813934687b9a4097a1a9a4179f3b0f   (tier-1 v1, frozen)
```

An implementation is conformant if it reproduces the v2 `model_digest` byte-for-bit —
on any CPU vendor or instruction set — under regime `vitni-regime-2`. The v1 digest is
retained so receipts issued before the regime binding stay verifiable; the same
reference run reproduces it exactly.

Under the prior regime `vitni-regime-1`, this same run's v2 `model_digest` was
`ffebe8620f3a78009317c2d72fcc373b4b3cd5c63ba424f6776a2007e119c88f`; the digest moved
because the regime is bound, which is exactly the property that lets a verifier tell a
regime change from tampering. The v1 digest above is unchanged across the two regimes.
