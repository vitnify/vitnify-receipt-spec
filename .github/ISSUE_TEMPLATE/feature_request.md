---
name: Format proposal
about: Propose a normative change or a new version of the receipt format
title: "[proposal] "
labels: enhancement
---

## What should the format be able to express

Describe the new capability or guarantee (e.g. a new field, a countersignature envelope,
a new verification level).

## Change category

- [ ] Clarification (intended meaning unchanged; text made precise)
- [ ] Normative change (alters bytes that get hashed/signed — requires a **new version**)

If normative, this becomes `vitnify-receipt-vN.md`; an existing frozen version is not
edited. See CONTRIBUTING.md.

## Wire format

Sketch the exact serialization / hashing / signing change. Note whether it affects the
tier-1 model-computation digest, the receipt body, the event log, or the signature
envelope.

## Compatibility & security

- How do existing (previous-version) receipts continue to verify?
- What is the security rationale? Does it change any trust assumption?
- Which implementations (`vitni-tensor`, `vitnify`) need to change to match?

## Alternatives considered

What else did you weigh, and why this?
