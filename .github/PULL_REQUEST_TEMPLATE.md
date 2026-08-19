<!-- Thanks for improving the vitnify receipt spec. Please complete the checklist. -->

## What this changes

Briefly describe the change.

## Category

- [ ] Editorial (typo / wording; no change to any byte on the wire)
- [ ] Clarification (removes an ambiguity; intended meaning unchanged)
- [ ] Normative (alters hashed/signed bytes — lands as a **new version file**, not an
      edit to a frozen one; issue linked below)

## Checklist

- [ ] I did **not** change the pinned reference digest
      (`9c0754458633e863e0fb5bb2bd00df0d8b813934687b9a4097a1a9a4179f3b0f`) — it is fixed
      by the engine's computation and only ever changes in lockstep across the engine,
      SDK, and spec.
- [ ] For a normative change: a new `vitnify-receipt-vN.md` was added, the README's
      "Current version" link updated, and the affected implementations
      (`vitni-tensor` / `vitnify`) are being updated to match.
- [ ] Links resolve (the CI link check passes).
- [ ] Commits are signed off (DCO): `git commit -s`

## Related issues

Closes #
