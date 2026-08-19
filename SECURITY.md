# Security Policy

This repository is the normative definition of the **vitnify execution receipt** format.
It ships no runtime code, but a flaw in the *specification* is a real security issue: if
the format lets a receipt be forged, verified against the wrong key, or reduced to an
ambiguous set of bytes, every conforming implementation inherits that weakness. We treat
reports against the spec as security reports.

## Reporting a vulnerability

**Email [security@vitnify.com](mailto:security@vitnify.com).** Please do **not** open a
public GitHub issue, pull request, or discussion for a suspected weakness in the format —
that discloses it before there is a corrected version.

A private GitHub Security Advisory (Security ▸ *Report a vulnerability*) is also fine.

A useful report includes:

- the exact section and version (e.g. `vitnify-receipt-v1.md`, "The event log",
  "Verification");
- the property you believe is violated and a concrete scenario — ideally two byte
  sequences, or two logs, that a conforming verifier would treat identically or
  incorrectly;
- whether the issue is in the format itself or in the description being ambiguous enough
  that independent implementations would diverge.

## What's in scope

Weaknesses in the *design or wording* of the format, including:

- **Canonicalization ambiguity** — any way two distinct inputs can serialize to the same
  canonical bytes (a hash or Merkle-root collision), or any place the canonical form is
  underspecified enough that two honest implementations disagree on the bytes to hash.
- **Verification gaps** — a defined receipt that passes the level-1 checks despite an
  edit, reorder, truncation, or substitution of the transcript; a signature envelope that
  can be satisfied by a key other than the one that signed.
- **Downgrade / substitution** — a way to strip or swap a field (capabilities, model
  digests, the signature envelope) without detection, or to pass off a weaker construction
  (e.g. the HMAC fallback) as an ed25519-signed receipt.
- **Under-stated trust assumptions** beyond the honest limit already documented in the
  spec.

## Not in scope

- The trust-boundary limit that the spec already states plainly: an embedded ed25519 key
  proves integrity and signer continuity, **not** that the signer was an authorized
  runtime. Binding provenance requires a verifier-supplied trust anchor. That is a
  documented limitation, not a vulnerability.
- Bugs in a specific implementation — report those to the relevant repo
  ([`vitni-tensor`](https://github.com/vitnify/vitni-tensor) or
  [`vitnify`](https://github.com/vitnify/vitnify)), or to their `security@vitnify.com`
  contact if sensitive.
- Typos or editorial suggestions — please just open a normal PR for those.

## Response expectations

- We will **acknowledge your report within 3 business days**.
- We will assess the issue and, if confirmed, decide whether it can be clarified in place
  or requires a new format version, typically within 10 business days.
- We will coordinate disclosure with you and are happy to credit you unless you prefer to
  remain anonymous.

## Safe harbor

We will not pursue or support legal action against anyone who, in good faith, follows this
policy while investigating or reporting a weakness in the format — giving us reasonable
time to respond before public disclosure. We consider good-faith security research to be
authorized conduct and will work with you. If in doubt, ask first at security@vitnify.com.
