# Agent commands

These are the commands to give the agent in Google Antigravity. Slash commands run a skill from `.agents/skills/`; plain words such as "continue" trigger the same skills through their descriptions.

## At a glance

| Command | What the agent does | Changes files? | Commits? |
| --- | --- | --- | --- |
| `/bootstrap-workspace` | Initialises the Rust project: chooses the project and crate names, replaces every name placeholder, creates the workspace, CI and license policy | Yes | Yes, one bootstrap commit; asks before pushing |
| `continue` (or `/next-task`) | Completes the next roadmap task: plan, code, tests, `/verify`, roadmap tick, progress entry | Yes | Yes, one local commit per task |
| `continue with M1.2` | The same, for a named task | Yes | Yes |
| `/status` | Reports the current milestone, open tasks, open decisions, last session and git state | No | No |
| `/verify` | Format, clippy, all tests and license checks; fixes failures | Only to fix failures | No |
| `/accuracy-check` | Residual, orthogonality and LAPACK agreement across the test matrices | No | No |
| `/benchmark` | Release benchmark against LAPACK and faer, CSV results, linear and log-log plots | Results only (`results/`) | No |
| `/add-kernel` | The procedure for writing or changing a CubeCL kernel, with tests against the f64 reference | Yes | Within the task's commit |
| `/milestone-gate` | Proves the milestone's gate, writes the evidence file, runs the release checklist at M2 | Yes | Yes; asks before publishing or tagging |
| `/record-decision` | Adds, accepts or supersedes a decision and updates what depends on it | Yes | Yes |
| `/upgrade-cubecl` | Moves to a new CubeCL release on its own branch, with before-and-after checks | Yes | Yes, on the upgrade branch |

## Getting started

**`/bootstrap-workspace`** runs once, in an empty repository that holds only this kit. It:

1. checks git (offers `git init -b main`, and to commit the planning files first);
2. asks for a project name, suggests three if you have none, and checks crates.io that `<name>` and `<name>-bench` are free;
3. replaces every project-name and crate-prefix placeholder in `AGENTS.md`, `README.md`, `LICENSE-MIT`, `docs/` and `.agents/`, then greps to prove none are left;
4. creates the Cargo workspace (library crate, bench crate, xtask), `rust-toolchain.toml`, CI and `deny.toml` from the templates;
5. confirms the Proposed decisions with you: CubeCL version (D-012), LAPACK backends (D-013), license (D-017), name (D-018);
6. runs `cargo xtask verify`, ticks M0.1, logs progress and commits `chore: bootstrap cargo workspace`;
7. asks before pushing; after the first CI run it settles D-016 (GPU tests in CI).

## The daily loop

**`continue`** is the command you will use most. Each time, the agent:

1. checks for uncommitted work from a previous session and offers to finish it first;
2. reads the last progress entries and finds the first unticked task of the current milestone (or the task you named);
3. splits the task first if it is too big for one session;
4. writes a plan (files, API changes, formulas, launch grid, tests) and waits for your approval only if it changes a public API, adds a dependency, loosens a tolerance or contradicts a decision;
5. implements it in small compiling steps, using `/add-kernel` and `/accuracy-check` where they apply;
6. runs `/verify` until green, without weakening tests;
7. ticks the task in `docs/roadmap.md`, appends to `docs/progress.md`, and makes a local commit with a Conventional Commits message, for example `feat(kernels): add make_reflector`;
8. names the next task, or suggests `/milestone-gate` when the milestone is complete.

It never pushes, and never starts the next milestone, unless you ask.

Useful variations:

| Say | Effect |
| --- | --- |
| `continue` | The next task |
| `continue with M2.3` | A specific task |
| `continue, plan only` | Stop after the plan, for review |
| `/status` then `continue` | Check where things stand before starting a session |

## Checking quality

- **`/verify`**: run any time; `continue` runs it before every commit. Set `QR_GPU_TESTS=1` if GPU tests are gated in CI (D-016).
- **`/accuracy-check`**: after changing anything that affects results, and at every gate. Reports ρ_res, ρ_orth, δ_R and δ_τ per test matrix.
- **`/benchmark`**: from milestone M2. Flags such as `--sizes`, `--panel-widths`, `--cube-dims` and `--baselines` select a sweep; results go to `results/run_<timestamp>/`.

## Milestones and decisions

- **`/milestone-gate`** when every task of a milestone is ticked. It writes `docs/milestones/<id>-gate.md` with the evidence, and at M2 prepares release 0.1 up to, but not including, `cargo publish`.
- **`/record-decision`** to settle a Proposed decision, add a new one or, with your approval, supersede an Accepted one. For example: `/record-decision accept D-014 with nb = 64`.

## Maintenance

- **`/upgrade-cubecl`** when you want a newer CubeCL. It works on its own branch, fixes known API changes, re-runs `/accuracy-check`, compares benchmarks, and asks before merging.

## What the agent will always ask you about

- Pushing, force-pushing, tagging, publishing or changing remotes.
- Anything that contradicts an Accepted decision.
- A dependency outside the license allowlist, or any C dependency in the library crate.
- Loosening an accuracy threshold or test.
- Work from a later milestone.
