# Request

*"Finally, can we come up with AI agent commands? For example, (1) /bootstrap-workspace to initialise the Rust project, create crate names and replace project-name placeholders. (2) continue; to direct the agent to complete the next roadmap task, including tests and a local Git commit."*

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
