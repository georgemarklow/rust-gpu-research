# Agent guide

This repository builds a **Rust library for QR decomposition on GPUs with CubeCL**. Version 1 is a blocked Householder QR of square, column-major f32 matrices (n = 128 to 4096) on wgpu/Metal, benchmarked against LAPACK `sgeqrf` and the pure-Rust CPU library faer. Later versions add explicit Q, least squares, tall-skinny and batched inputs, and f64 on CUDA/ROCm.

- **Project name:** TBD (chosen in M0.1 by `/bootstrap-workspace`)
- **Repository:** TBD
- **Crate prefix:** TBD, written `<prefix>` in these files (the library is `<prefix>`, the benchmark and LAPACK-comparison crate is `<prefix>-bench`)
- **Developer:** one person, evenings and weekends, on an Apple Silicon Mac. Small, finished steps beat big, half-done ones.

## Where things are

| What | Where | Read it when |
| --- | --- | --- |
| Full design, maths and reasons | @docs/design-plan.md | Before any design-level change, and before writing a kernel |
| Binding decisions | @docs/decisions.md | Before proposing anything that might contradict one |
| Milestones, tasks, gates | @docs/roadmap.md | At the start of every session |
| What has been done | @docs/progress.md | At the start of every session; append after every task |
| Commands the human uses | @docs/commands.md | When unsure what a command should do |
| Detailed rules | `.agents/rules/` | Loaded automatically by file type or topic |
| Repeatable procedures | `.agents/skills/` | Invoke with `/<skill-name>` |

## Skills (slash commands)

The full command reference, with what each command changes and commits, is @docs/commands.md.

- `/bootstrap-workspace`: one-time setup: project and crate names, placeholder replacement, Cargo workspace, xtask, CI and license policy.
- `/next-task`: the default development loop. Use it whenever the user says "continue" or "next".
- `/status`: read-only report of the milestone, open tasks, open decisions and git state.
- `/verify`: format, lint, tests and license checks. Run before calling anything done.
- `/add-kernel`: the procedure for writing or changing a CubeCL kernel.
- `/accuracy-check`: residual, orthogonality and LAPACK agreement for a factorization.
- `/benchmark`: timed runs against LAPACK and faer, CSV results and plots.
- `/milestone-gate`: checks a milestone's gate and records the evidence.
- `/record-decision`: adds, accepts or supersedes a decision consistently.
- `/upgrade-cubecl`: the procedure for moving to a new CubeCL release.

## Invariants

These hold everywhere. The rules in `.agents/rules/` explain each in detail.

1. **Crates:** `<prefix>` is the library: pure Rust plus CubeCL, no C dependencies. `<prefix>-bench` holds the benchmark binary and the comparison tests, and is the only crate that may depend on the baselines: LAPACK, faer (D-019) and, behind its `cuda` feature, cuSOLVER (D-020). `xtask` holds developer tasks.
2. **Column-major everywhere.** Element (i, j) of a `DeviceMatrix` is at `offset + j * ld + i`. No row-major or general strides.
3. **LAPACK conventions.** Reflectors follow `slarfg` exactly (sign of β, τ = 0 for an already reduced column), and the compact output matches the `sgeqrf` layout: R on and above the diagonal, reflectors below it, τ separately.
4. **Only `gemm.rs` calls `cubecl-matmul`.** Every other module goes through its wrapper functions.
5. **Device first.** The core API takes and returns `DeviceMatrix` values that stay on the device. Host slices appear only in `host.rs`.
6. **Generic precision.** Kernels are generic over `F: Float`. Unsupported precisions (f64 on Metal) return `QrError::UnsupportedPrecision` before any launch.
7. **Errors, not panics.** Library code returns `Result<_, QrError>` for anything that can be checked before launching: shapes, configuration, precision, rank deficiency.
8. **f64 oracles.** Accuracy is always judged by f64 CPU code (the reference implementation and the test ratios), never by comparing one GPU result with another.
9. **CubeCL is pinned.** The version in `Cargo.toml` changes only through `/upgrade-cubecl`.
10. **The CubeCL CPU runtime is a test target only.** Never restructure or slow down a GPU kernel to make it compile on the CPU runtime; mark such tests ignored there instead.

## Definition of done

A task is done when all of these are true:

- `/verify` passes with no new warnings.
- New behavior has tests; every bug fix has a regression test.
- Any change to a factorization kernel passes `/accuracy-check`.
- Public items have rustdoc; changed behavior is reflected in `docs/`.
- The task is ticked in `docs/roadmap.md` and an entry is appended to `docs/progress.md`.
- Changes are committed locally with a Conventional Commits message (for example `feat(kernels): add make_reflector`).

## Stop and ask the human when

- A change would contradict `docs/decisions.md`.
- A new dependency has a license outside the policy in `.agents/rules/dependencies.md`, or the library crate would gain a C dependency.
- A public API would break after release 0.1.
- A test tolerance or accuracy threshold would have to be loosened to make a test pass.
- Work from a later milestone seems necessary before the current gate passes.

## Do not

- Bump CubeCL on its own; use `/upgrade-cubecl`.
- Call `cubecl-matmul` outside `gemm.rs`.
- Silence clippy with `allow` attributes unless a comment explains why.
- Report benchmark numbers without warm-up runs, or from a debug build.
- Start the next milestone's tasks without being asked.
- Push, force-push, rewrite git history or change remotes without the human's explicit go-ahead. Local commits are fine.
