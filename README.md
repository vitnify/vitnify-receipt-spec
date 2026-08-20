# vitnify-receipt-spec

The canonical format for a **vitnify execution receipt** — one signed, self-verifying
object that binds an AI-agent run (its model computation, granted capabilities, tool
calls, results, and order) so an action cannot be detached from what produced it.

**Current version:** [`vitnify-receipt-v2.md`](vitnify-receipt-v2.md) · prior (frozen): [`v1`](vitnify-receipt-v1.md)

Everything hashes with BLAKE3; signatures are ed25519 with an embedded public key, so
any party can verify a receipt offline — no model, no network, no secret.

## Reference implementations
- **engine** (the model-computation digest): [`vitni-tensor`](https://github.com/vitnify/vitni-tensor)
- **SDK** (the full receipt): [`vitnify`](https://github.com/vitnify/vitnify)

## Conformance
This spec has an executable definition: [`conformance`](https://github.com/vitnify/conformance)
— reference vectors and forgery probes with a runner that points at any implementation
and reports whether it reproduces the vectors and rejects the attacks. An implementation
is conformant only if it passes. When the spec and the kit disagree, that is a bug in one
of them — open an issue.

## License
Apache-2.0. **"vitnify"** / **"vitnify-verified"** are trademarks — see [TRADEMARKS.md](TRADEMARKS.md).

## Part of Vitnify
This spec is one of the open repos:

- **[vitni-tensor](https://github.com/vitnify/vitni-tensor)** — the engine (tier-1 model-computation digest).
- **[vitnify](https://github.com/vitnify/vitnify)** — the SDK (the full receipt).
- **[conformance](https://github.com/vitnify/conformance)** — the conformance kit (vectors + probes + runner).
- **[vitnify.com](https://vitnify.com)** — the project.
