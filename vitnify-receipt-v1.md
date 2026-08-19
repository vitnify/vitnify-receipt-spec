# vitnify-receipt v1

The canonical format for a vitnify execution receipt: one signed, self-verifying
object that binds an agent run — its computation, authority, actions, and order —
so an action cannot be detached from what produced it.

Everything hashes with **BLAKE3**. Signatures are **ed25519** with an embedded
public key, so anyone can verify a receipt offline with no model, no network, and
no secret.

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
reproducible **bit-for-bit across CPU vendors and instruction sets**.

**Tier 2 — execution receipt** (produced by the SDK). The full agent-run object,
which *embeds* the tier-1 digests.

---

## The receipt body

The signed digest is `BLAKE3(canonical_json(body))`, where `body` is:

| field | meaning |
|---|---|
| `v` | `"vitnify-receipt v1"` |
| `program_hash` | hash of the exact agent program that ran |
| `capabilities` | sorted list of granted capabilities |
| `event_root` | BLAKE3 Merkle root of the event log (inclusion proofs) |
| `n_events` | event count |
| `head_hash` | head of the hash-chained event log |
| `model_digests` | ordered tier-1 digests, one per model step |

`canonical_json` = UTF-8 JSON, keys sorted, no whitespace.

### Signature envelope
`sig` (ed25519 over the 32-byte receipt digest), `sig_alg` = `"ed25519"`,
`pubkey` (raw ed25519 public key, hex). HMAC is a fallback only.

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
computation into the receipt.

---

## Verification

**Level 1 — integrity (no model).** Recompute `event_root`, `head_hash`,
`n_events`, and `model_digests` from the raw events; confirm they match the
receipt; verify the ed25519 signature against the embedded public key. Rejects any
edit, reorder, truncation, or forgery of the transcript.

**Level 2 — recomputation (needs the weights).** Re-run each `llm_call` through
`vitni-tensor` and confirm every `model_digest` reproduces bit-for-bit.

---

## Trust boundary (honest limit)

An embedded ed25519 key proves **integrity and signer continuity**, not that the
signer was an authorised runtime. A receipt re-signed by a different key still
self-verifies. Binding execution *provenance* requires a pinned trust anchor —
a known key, or a TPM/enclave — supplied by the verifier. A **vitnify-verified**
receipt is one countersigned by the official vitnify Verification Authority
against such an anchor.

---

## Reference value

A conformance anchor for the tier-1 model-computation digest, reproducible with the
`vitni-receipt` binary from the `vitni-tensor` engine:

```
model         TinyLlama-1.1B-Chat, Q4_K_M GGUF
model_id      tinyllama-1.1b-chat-Q4_K_M
prompt        [1, 9038, 2501, 263, 931, 29892]      ("Once upon a time,")
n_new         20
model_digest  9c0754458633e863e0fb5bb2bd00df0d8b813934687b9a4097a1a9a4179f3b0f
```

An implementation is conformant if it reproduces this digest byte-for-bit — on any
CPU vendor or instruction set.
