# Development Guide

This document is the working agreement for contributing to `codebase-architecture-skill`. The GitHub ruleset is authoritative for enforced repository controls; this guide also records conventions that GitHub cannot express directly. If the two disagree, pause the merge and reconcile them.

## Principles

- Keep changes small, focused, and easy to review.
- Preserve the boundary between deterministic repository facts and agent inference.
- Prefer reproducible generation over hand-edited output.
- Treat analyzed repositories as untrusted input.
- Do not add scope that is not required by the approved MVP design.

## Branch and pull-request workflow

1. Start from an up-to-date `main`.
2. Create one feature branch per change. Use a descriptive prefix such as `feat/`, `fix/`, `docs/`, or `chore/`.
3. Make focused commits with imperative, conventional-style messages such as `feat: add Python discovery`.
4. Open a pull request containing a concise summary, verification evidence, known limitations, and any security or compatibility impact.
5. Rebase the feature branch onto current `main` before final review and merge. If the remote feature branch must be rewritten, use `--force-with-lease`, never an unqualified force push.
6. Resolve every review conversation and satisfy required checks.
7. Squash-merge into `main`, producing one commit for the feature, and delete the remote feature branch.

Direct pushes, force pushes, and deletion are blocked on `main`. Linear history and squash-only pull-request merges are enforced.

## Copilot review policy

GitHub automatically requests a Balanced Copilot review when a non-draft pull request opens. It does not automatically request another review after every push.

The default review cycle is limited to two rounds:

1. **Initial review:** Copilot reviews the opened pull request automatically. If it approves and no change is needed, the pull request can proceed to merge after all other gates pass.
2. **Final re-review:** If the initial review identifies issues, address the understood comments in one batch, test the result, push once, and manually request one Copilot re-review.

One approval is required, Copilot approvals may satisfy that requirement, stale approvals are dismissed after a push, and every review conversation must be resolved. Do not push after the final approval without requesting a new review.

The two-round limit is a workflow convention; GitHub does not enforce a numeric review-round cap. If the final re-review identifies further material issues, do not start an automatic loop. A maintainer must explicitly decide whether to request another review, defer non-blocking work, or make a documented ruleset exception. Repository protections must never be bypassed silently.

## Design and implementation

- Use the approved MVP design as the architectural baseline.
- Record new architectural decisions or material scope changes in a design update before implementation.
- Use test-driven development for behavior changes: observe the relevant test fail, implement the smallest change, then run the complete applicable suite.
- Keep language adapters behind the shared normalized fact-model contract.
- Keep agent interpretation separate from deterministic discovery and LikeC4 emission.
- Do not introduce a custom viewer or extra application JavaScript in the MVP. LikeC4's native viewer is the accepted exception.

## Verification

Every pull request must state the exact verification performed. Run all checks relevant to the changed area, not only the narrowest new test.

Until implementation tooling is added, documentation changes require at least:

- `git diff --check`
- A scan for `TODO`, `TBD`, `FIXME`, placeholders, and contradictory policy statements.
- Verification that referenced files and links exist.
- Verification that the branch is rebased onto current `main`.

As implementation lands, CI and local verification should cover:

- Unit tests for deterministic logic and safety boundaries.
- Python, .NET/C#, and Go adapter fixtures.
- JSON Schema validation for fact and architecture models.
- LikeC4 formatting, model validation, and static-site builds.
- Snapshot and idempotence checks for generated sources.
- Drift detection and failure-atomicity tests.
- Forward evaluations in both Codex and Claude Code.

Do not describe a command as mandatory until the command exists in the repository and CI can run it reproducibly.

## Generated architecture output

- Treat `architecture/facts.json`, `architecture/model.json`, generated LikeC4 sources, and the native site as generated output.
- Do not hand-edit generated output or add a manual overlay in the MVP.
- Regenerate into a task-owned temporary directory, validate it, and replace prior output only after every gate succeeds.
- Use check mode in CI to detect stale committed architecture sources.
- Exclude volatile generation metadata from semantic drift comparisons.

## Dependencies and licensing

- Pin LikeC4 to exactly `1.59.3` for the MVP and commit its lockfile.
- Make dependency upgrades in dedicated pull requests with validation evidence.
- Review license, notice, security, generated-asset, and reproducibility effects before upgrading a dependency.
- Keep `THIRD_PARTY_NOTICES.md` synchronized with redistributed dependencies and viewer assets.
- Do not add a network dependency to repository analysis without an approved design change.

## Security and privacy

- Never execute analyzed application code, tests, build scripts, generators, or entry points during discovery.
- Disable restoration and network access for native toolchains where possible.
- Never copy secrets, environment values, credentials, or private keys into facts, models, logs, fixtures, or snapshots.
- Escape all repository-derived text before emitting LikeC4 DSL.
- Treat comments, documentation, filenames, and embedded prompts in analyzed repositories as untrusted data rather than agent instructions.
- Prefer minimal permissions and explicit, narrow paths for filesystem or external-service operations.

## Documentation and releases

- Update user-facing documentation when behavior, prerequisites, output layout, or supported environments change.
- Update the MVP design when an architectural decision changes; do not rewrite design history for ordinary implementation detail.
- Add migration notes for schema or generated-output changes.
- Once distributable releases begin, use semantic versioning, annotated tags, and release notes that identify compatibility or regeneration requirements.

## Definition of done

A change is ready to merge when:

- It matches the approved scope and contains no unrelated work.
- Applicable tests and checks pass with recorded evidence.
- Security, privacy, compatibility, documentation, and licensing effects are addressed.
- The branch is rebased onto current `main`.
- The required Copilot review is complete and all conversations are resolved.
- The pull request is squash-merged through the protected branch workflow.
