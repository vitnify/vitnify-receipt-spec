# vitnium-receipt-spec

The canonical format for a **vitnium execution receipt** — one signed, self-verifying
object that binds an AI-agent run (its model computation, granted capabilities, tool
calls, results, and order) so an action cannot be detached from what produced it.

**Current version:** [`vitnium-receipt-v1.md`](vitnium-receipt-v1.md)

Everything hashes with BLAKE3; signatures are ed25519 with an embedded public key, so
any party can verify a receipt offline — no model, no network, no secret.

## Reference implementations
- **engine** (the model-computation digest): [`vitni-tensor`](https://github.com/vitnium/vitni-tensor)
- **SDK** (the full receipt): [`vitnium`](https://github.com/vitnium/vitnium)

## License
Apache-2.0. **"vitnium"** / **"vitnium-verified"** are trademarks — see [TRADEMARKS.md](TRADEMARKS.md).
