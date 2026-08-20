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
> containment, and defines how a **hosted** model is recorded. The tier-1
> model-computation digest is a distinct, lower-level version line and is
> **unchanged** — it and the conformance anchor below are identical to v1. v1
> remains frozen and valid for receipts issued under it.

---

## Two tiers

**Tier 1 — model-computation digest** (produced by the `vitni-tensor` engine).
The per-step commitment to what the model actually computed:

```
model_digest = BLAKE3( "vitnify-receipt v1\x00"
                       || LEB128(n_inputs)        || inputs
                       || LEB128(n_outputs)       || outputs
                       || LEB128(n_ops)           || per-op records
                       || LEB128(n_activations)   || activation records
                       || LEB128(n_interventions) || intervention records )
```

Fields are length-prefixed so concatenation is injective. This digest is
reproducible **bit-for-bit across CPU vendors and instruction sets**. Its domain
separator stays `"vitnify-receipt v1\x00"`: the tier-1 digest is versioned
independently of the receipt and did not change in v2.

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

`issued_at` is **issuer-asserted**: it places the receipt in time and defeats
silent replay, but it is not a trusted timestamp. Non-repudiable time requires an
RFC 3161 token or the Verification Authority countersignature.

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
computation into the receipt. For a hosted model it may also carry a `provider`
object (see Hosted models).

---

## Verification

Verification **fails closed**: a receipt that cannot be cryptographically verified
never returns a valid verdict.

**Level 1 — integrity (no model).**
1. Recompute `event_root`, `head_hash`, `n_events`, and `model_digests` from the
   raw events and confirm they match the receipt.
2. **Signature.** Verify the ed25519 signature against the embedded public key. A
   receipt with `sig_alg` = `"none"`, an unknown algorithm, or an HMAC receipt
   presented without its key is **invalid** — an unverifiable receipt is never
   `ok`.
3. **Capability containment.** Every *allowed* `tool_call` in the log must name a
   tool in `capabilities`. A receipt whose log allowed an undeclared tool is
   invalid, so the receipt *proves* containment held rather than merely listing it.

Rejects any edit, reorder, truncation, forgery, or out-of-policy action in the
transcript.

**Level 2 — recomputation (needs the weights).** Re-run each `llm_call` through
`vitni-tensor` and confirm every `model_digest` reproduces bit-for-bit. Available
for local weights only (see Hosted models).

**Signer pinning (optional).** An embedded key proves signer *continuity*, not
*authority*. A verifier may supply an allow-list of trusted ed25519 keys; the
receipt's key must be on it. This closes the re-signing gap (below) in the
open-source verifier, with no managed service.

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

A conformance anchor for the tier-1 model-computation digest (unchanged from v1),
reproducible with the `vitni-receipt` binary from the `vitni-tensor` engine:

```
model         TinyLlama-1.1B-Chat, Q4_K_M GGUF
model_id      tinyllama-1.1b-chat-Q4_K_M
prompt        [1, 9038, 2501, 263, 931, 29892]      ("Once upon a time,")
n_new         20
model_digest  9c0754458633e863e0fb5bb2bd00df0d8b813934687b9a4097a1a9a4179f3b0f
```

An implementation is conformant if it reproduces this digest byte-for-bit — on any
CPU vendor or instruction set.
