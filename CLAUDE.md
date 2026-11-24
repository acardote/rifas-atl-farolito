# Bruno Method — project discipline

This project applies the [Bruno Method](https://www.olympum.com/bruno-method/SPEC.md). Configuration lives in `.bruno/config.toml`. The `bruno` skill (`/bruno <subcommand>`) is the operational entry point.

## Five Moves as gates

For non-trivial work, run the moves in order. Skipping moves produces unsound work.

1. **Move 1 — Evidence Before Architecture.** Capture current behavior, recent changes, owner notes, and known incidents as evidence on the parent issue *before* proposing what to build.
2. **Move 2 — Strategy Before Plan.** Commit decisions (architecture, ownership, sequence) as decision records in `decisions/` (start from `decisions/_template.md`), cite each one from the parent with a `Decided-by: D-NNNN` line, and keep the explicit assumption ledger on the parent. Plans before strategy are precise nonsense.
3. **Move 3 — Issues As Memory.** Convert strategy into addressable parent + child issues using `/bruno parent` and `/bruno child`. Work in chat does not exist.
4. **Move 4 — Falsification Before Confidence.** Each child carries ≥1 explicit falsifier. Use `/bruno falsify` to invoke the challenger agent.
5. **Move 5 — Closure Against Reality.** Reconcile acceptance against landed state via `/bruno reconcile` before closing. Closure by fiat is forbidden.

## Refusal modes

- An issue without goal/scope/target is a wish. Refuse to advance.
- A child without ≥1 falsifier is not buildable. Refuse to execute.
- A claim about deployed code that hasn't been resolved against serving state is a confabulation. Refuse to merge.
- **Closing a parent without re-validating its assumption ledger is a confabulation.** All children closed is necessary, not sufficient. Per SPEC §4.2, a parent closes only when every child is reconciled AND the assumption ledger has been re-validated. For each `A<N>` in the parent's ledger: either file a child to validate it, or post an explicit `## Accept-A<N>: <reason>` comment with reasoning the parent's stakeholders would defend. **Default action when an A<N> is open: file the validating child. Acceptance is the exception, not the path of least resistance.**
- **A reconciled parent close is a shipping unit; a disposition close (declined or superseded) never is.** Never count a disposition close as shipped or list it as delivered. It still starts the release-readiness check, just as a reconciled close does, for any commits already on the default branch under it (from its reconciled children, or from merged PRs linked to the declined or superseded issue); those are classified on their own. If this project defines no such check, the check is: (1) take only the closed issue's commits (a reconciled parent's, or for a disposition close the commits above), not everything since the last tag; (2) only if they form a coherent shipping unit (a finished change that stands on its own, not a piece of work still open) propose a semantic-version bump (MAJOR / MINOR / PATCH); (3) surface it to the owner, who decides and tags. A project that has never tagged a release skips the check. See the Releases rule in [`method/owner-disposition.md`](https://github.com/acardote/bruno-method/blob/main/method/owner-disposition.md).

## Anti-patterns (SPEC §11)

These phrases indicate type errors and stop the build:

- "use an agent"
- "make it work"
- "buying agents will not give you agency"
- "demo passed"
- "merged" (as evidence — distinct from a noun-phrase use)
- "it works on my machine"
- "I checked it"
- "we'll test it later"
- "the agent says it's done"
- "trust the model"

If you find one in an issue body or PR description, label `anti-pattern` and stop until resolved.

## Evidence floor (SPEC §3.3)

Stronger ↑ — landed-state reconciled
        — gate evidence
        — RCA + regression
        — repro command
        — linked test run
Weaker ↓ — "it works on my machine"

Bring evidence at or above the floor for each claim. Evidence without provenance does not type-check (§3.2).

See [`method/evidence-kinds.md`](https://github.com/acardote/bruno-method/blob/main/method/evidence-kinds.md) for the six evidence kinds with operational examples per kind, and [`method/falsifier-patterns.md`](https://github.com/acardote/bruno-method/blob/main/method/falsifier-patterns.md) for the falsifier-pattern catalog.

## Reviewer matrix (SPEC §7.4)

| Reviewer | Eligibility |
|---|---|
| Author | Disqualified as adversary |
| Sibling | Static / dynamic adversary on small changes |
| Owner | Required for reality adversary on load-bearing changes |
| Outsider | Brought in to falsify framing on `weight:heavy` |

See [`method/reviewer-matrix.md`](https://github.com/acardote/bruno-method/blob/main/method/reviewer-matrix.md) for objective weight criteria, the eligibility-by-stage-and-weight matrix, and the **solo-mode** substitutes (deferred review, named adversarial agent, ship-and-document-risk caveat).

## Distance self-check (SPEC §12)

For periodic reflection on the six-axis distance metric (falsifiesPlans, findsRootCause, leavesTrail, gatesClaims, slicesWork, reviewsHard), see [`method/distance-self-check.md`](https://github.com/acardote/bruno-method/blob/main/method/distance-self-check.md). Solid+ ratings require linked evidence per axis to resist confirmation bias.

## Operational pointers

- `/bruno status` — current Move position, open falsifiers, lint failures
- `/bruno parent` / `/bruno child` — create issues respecting `.bruno/config.toml` routing
- `/bruno evidence` — append typed, provenanced evidence
- `/bruno reconcile` — Move 5 gate before close
- `/bruno close` — refuses without reconciliation

## Working a parent's sequence map

When iterating a parent's children, drive each child through the full Bruno cycle (claim → falsify → implement → evidence → reconcile → close) without pausing between children for confirmation. Continue until the parent closes, OR until a child surfaces a real question/blocker that requires decision-maker input. Surface progress concisely between children, but do not stop merely to check in when the path forward is already agreed.
