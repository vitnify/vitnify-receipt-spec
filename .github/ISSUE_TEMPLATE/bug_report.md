---
name: Spec issue
about: An error, ambiguity, or inconsistency in the receipt format spec
title: "[spec] "
labels: bug
---

<!--
SECURITY: if this is a weakness that would let a receipt be forged or verified
incorrectly (a canonicalization collision, a verification gap, a downgrade), do NOT
file it here. Email security@vitnify.com — see SECURITY.md.
-->

## Where

- File and version (e.g. `vitnify-receipt-v1.md`):
- Section (e.g. "The event log", "Verification", "Reference value"):

## The problem

- [ ] Factual error (the spec says something that is wrong)
- [ ] Ambiguity (two implementations could read this differently)
- [ ] Inconsistency (contradicts another section or a reference implementation)
- [ ] Missing detail (something an implementer needs is unspecified)

Describe it clearly.

## Impact

Would this cause independent implementations to produce incompatible or
non-verifying receipts? Which implementations are affected
(`vitni-tensor`, `vitnify`, a third party)?

## Suggested wording

If you have a fix in mind, sketch it here (or open a PR).
