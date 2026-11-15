# QR decomposition with Rust and CubeCL: planning kit

This folder is the starting point for a Rust library that computes the QR decomposition on GPUs with CubeCL. It holds the design, the decisions, the roadmap and the agent rules and skills that Google Antigravity uses to build the library task by task. It follows the same structure as the Tetraglass project.

## What is inside

| Path | Contents |
| --- | --- |
| `AGENTS.md` | The agent guide: project summary, invariants, definition of done, when to stop and ask |
| `docs/design-plan.md` | The full design with the maths (GitHub `math` blocks) and the architecture diagram (Mermaid) |
| `docs/decisions.md` | Numbered decisions D-001 to D-020, Accepted or Proposed, including the LAPACK, faer and cuSOLVER baselines |
| `docs/roadmap.md` | Milestones M0 to M4 with tasks, completion criteria and gates |
| `docs/progress.md` | The progress log, with its first entry |
| `.agents/rules/` | Rules loaded by file type or topic: Rust code, CubeCL kernels, numerics, testing, dependencies, benchmarking |
| `.agents/skills/` | Slash-command procedures: bootstrap-workspace, next-task, status, verify, add-kernel, accuracy-check, benchmark, milestone-gate, record-decision, upgrade-cubecl |
| `docs/commands.md` | The command reference: what each command does, changes and commits |
| `LICENSE-MIT` | Licence text; the bootstrap fills in the project name |

## How to start

1. Create an empty git repository and copy the contents of this folder into it.
2. Open the folder in Google Antigravity.
3. Ask the agent to run `/bootstrap-workspace`. It asks for a project name, checks crates.io, replaces the name placeholders, creates the Cargo workspace and confirms the Proposed decisions with you.
4. From then on, say "continue" (or run `/next-task`) at the start of each session. The agent takes the next roadmap task, plans it, implements it with tests, runs `/verify`, records progress and makes a local commit. `/status` shows where things stand.

`docs/commands.md` lists every command.

The agent asks before anything that contradicts a decision, adds a dependency outside the licence policy, loosens an accuracy threshold, or pushes to a remote.
