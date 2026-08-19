# Contributing to the vitnify receipt spec

This repository is the **normative** definition of the vitnify execution-receipt format.
There is no code to build — the deliverable is prose precise enough that two independent
implementations produce byte-identical, mutually verifiable receipts. Changes are held to
that bar.

## What kinds of change look like what

- **Editorial** (typos, clarified wording that does not change any byte on the wire,
  better examples): open a PR directly. Explain in the description why the meaning is
  unchanged.
- **Clarification of an ambiguity** (the format was always intended one way, but the text
  allowed two readings): open an issue first so we can confirm the intended reading, then a
  PR. Note which implementations are affected.
- **Normative change** (anything that alters the bytes that get hashed or signed — a field,
  the canonical-JSON rules, the digest construction, the signature envelope): this is a
  **new format version**, never an edit to a frozen one. Open an issue with the rationale,
  a migration/compatibility note, and the security reasoning before writing the spec text.

## Versioning policy

- A published version file (e.g. `vitnify-receipt-v1.md`) is **frozen** once
  implementations ship against it. Do not change its wire meaning; fix only genuine
  editorial errata.
- A breaking or wire-visible change lands as a new file (`vitnify-receipt-v2.md`) with its
  own `v` string. The README's "Current version" link is updated in the same PR.
- The spec is the source of truth for the two reference implementations, the engine
  ([`vitni-tensor`](https://github.com/vitnify/vitni-tensor)) and the SDK
  ([`vitnify`](https://github.com/vitnify/vitnify)). A normative change is not done until
  those repos are updated to match; call out the cross-repo work in the PR.

## The reference value is a conformance anchor — do not change it casually

`vitnify-receipt-v1.md` pins a tier-1 model-computation digest:

```
model         TinyLlama-1.1B-Chat, Q4_K_M GGUF
model_id      tinyllama-1.1b-chat-Q4_K_M
prompt        [1, 9038, 2501, 263, 931, 29892]   ("Once upon a time,")
n_new         20
model_digest  9c0754458633e863e0fb5bb2bd00df0d8b813934687b9a4097a1a9a4179f3b0f
```

An implementation is conformant if it reproduces this digest byte-for-bit on any CPU
vendor or instruction set. **Do not edit this value.** It is fixed by the engine's actual
computation; changing the number here without a corresponding, agreed engine change would
declare every conforming implementation non-conformant. If a versioned change to the
computation is ever made, the new anchor is updated across the engine, SDK, and this spec
together — never here alone.

## Proposing a change, mechanically

1. Fork and branch.
2. Edit the relevant version file (or add a new one, per the versioning policy).
3. Keep Markdown tidy — the CI runs a link check; make sure any links you add resolve.
4. Open a PR describing the change, its category (editorial / clarification / normative),
   affected implementations, and security implications.

## Sign your commits (DCO)

We use the [Developer Certificate of Origin](https://developercertificate.org/). Sign off
every commit:

```bash
git commit -s -m "your message"
```

This adds a `Signed-off-by: Your Name <you@example.com>` line. Commits without it will be
asked to amend.

## Reporting a flaw vs. a vulnerability

A weakness that would let a receipt be forged or verified incorrectly is a **security**
issue — **do not** open a public issue; email
[security@vitnify.com](mailto:security@vitnify.com). See [SECURITY.md](SECURITY.md).

By contributing you agree your contributions are licensed under Apache-2.0. The **vitnify**
and **vitnify-verified** marks are trademarks — see [TRADEMARKS.md](TRADEMARKS.md).
