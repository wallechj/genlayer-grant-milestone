# Test flow (GenLayer Studio, hosted)

Paste `contracts/grant_milestone.py` into the Studio editor and deploy. You'll need
two Studio accounts: A (program owner) and B (builder). All payment values below
are in wei.

## 0. Prerequisite: fixtures must be live first

Validators fetch the demo deliverables from public URLs during verification, so
push `examples/` to GitHub BEFORE running this flow and check these URLs open in
a browser:

- https://raw.githubusercontent.com/wallechj/genlayer-grant-milestone/main/examples/landing.html
- https://raw.githubusercontent.com/wallechj/genlayer-grant-milestone/main/examples/docs.md
- https://raw.githubusercontent.com/wallechj/genlayer-grant-milestone/main/examples/docs_partial.md

If your default branch isn't `main`, adjust the URLs in the sample JSONs first.

## 1. Create a program (account A)

`create_program` with:

- title: `Acme Grants - Q1 builder program`
- description: `Small grants for independent builders shipping public goods. Two milestones, paid on verified delivery.`
- rules: `Deliverables must be public and fetchable. Partial delivery earns partial payment. One submission per milestone per builder; resubmission allowed only after an incomplete verdict.`

Expect: returns `0`.

Negative checks: empty title → `UserError: title is required`.

## 2. Create milestones (account A)

Milestone 0, `create_milestone`:

- program_id: `0`
- title: `Landing page`
- criteria: `A public landing page that names the product, lists at least 3 features, and offers a signup or waitlist action.`
- full_payment: `3000000000000000000` (3 GEN)
- partial_payment: `1500000000000000000`

Expect: returns `0`.

Milestone 1, `create_milestone`:

- program_id: `0`
- title: `Docs + changelog`
- criteria: `Public docs covering setup and usage, plus a changelog with at least one released version.`
- full_payment: `2000000000000000000`
- partial_payment: `1000000000000000000`

Expect: returns `1`.

Negative checks: account B creating a milestone → `UserError: only the program owner can do this`.

## 3. Sequential rule: milestone 1 is locked (account B)

Before milestone 0 is fulfilled, `submit_work` to milestone 1:

- milestone_id: `1`
- title: `Docs + changelog delivery`
- description: `Docs covering setup and usage, plus a changelog with v0.1.0.`
- demo_url: ``
- repo_url: `https://raw.githubusercontent.com/wallechj/genlayer-grant-milestone/main/examples/docs.md`

Expect: `UserError: earlier milestones must be fulfilled first`. No submission is created.

## 4. Submit work for milestone 0 (account B)

Valid delivery, `submit_work`:

- milestone_id: `0`
- title: `Acme Widget landing page`
- description: `Landing page for Acme Widget: names the product, lists 4 features, and has a waitlist signup form.`
- demo_url: `https://raw.githubusercontent.com/wallechj/genlayer-grant-milestone/main/examples/landing.html`
- repo_url: ``

Expect: returns `0`.

Dead-demo delivery, `submit_work`:

- milestone_id: `0`
- title: `Acme Widget landing page (mirror)`
- description: `Mirror of the landing page.`
- demo_url: `https://example.invalid/landing`
- repo_url: ``

Expect: returns `1`.

Negative checks: account A (owner) submitting → `UserError: program owner cannot submit work`.
Both URLs empty → `UserError: at least one deliverable URL is required`.
Non-URL deliverable → `UserError: deliverable URLs must be http(s) URLs`.

## 5. Verify milestone 0 (any account)

Watch the validator logs on each call: every validator fetches the submitted
deliverables before grading.

`verify_submission(1)` — dead demo first, so the milestone stays open. Expect
`"incomplete"`, `grade: "incomplete"`, `decided_payment: 0`, and
`get_submission_evidence(1)` noting the demo material could not be retrieved.
A delivery nobody can fetch is never a passing grade.

`verify_submission(0)` — expect `"complete"`, `grade: "complete"`,
`decided_payment: 3000000000000000000` (full), and `get_submission_evidence(0)`
naming the fetched landing content (product name, features, waitlist action).
`get_milestone_status(0)` → `"fulfilled"`.

Negative checks: submit to milestone 0 now → `UserError: milestone is already fulfilled`.
Re-running `verify_submission(0)` → `UserError: submission has already been verified`.

## 6. Milestone 1 unlocks: partial delivery (account B)

`submit_work`:

- milestone_id: `1`
- title: `Docs + changelog delivery`
- description: `Docs covering setup and usage; changelog section exists but no version released yet.`
- demo_url: ``
- repo_url: `https://raw.githubusercontent.com/wallechj/genlayer-grant-milestone/main/examples/docs_partial.md`

Expect: returns `2`.

`verify_submission(2)` — expect `"partial"`, `grade: "partial"`,
`decided_payment: 1000000000000000000` (partial), and evidence noting the docs
are present but the changelog has no released version. Partial delivery earns
partial payment — and still fulfills the milestone.
`get_milestone_status(1)` → `"fulfilled"`.

## 7. Lifecycle + payment trail (account A)

- `pause_program(0)`, then submit as B (to milestone 1) → `UserError: program is not open for submissions` (the program check fires before the milestone check).
- `reopen_program(0)`.
- `mark_paid(0)` → `get_submission(0).status == "paid"`.
- `mark_paid(1)` → `UserError: only verified (complete/partial) submissions can be marked paid`.
- `mark_paid` as account B (on submission 0) → `UserError: only the program owner can mark a submission paid`.
- `close_program(0)` ends the program.

## 8. Views

`get_program(0)` returns the full program incl. milestone_count `2`;
`get_milestone(0)` / `get_milestone(1)` return criteria, payment tables, and
`"fulfilled"` statuses; `get_program_count()` → `1`; `get_milestone_count()` → `2`;
`get_submission_count()` → `3`.
`get_submission_evidence(n)` returns each verdict's one-sentence evidence note.
