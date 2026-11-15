# Progress log

Newest entries at the bottom. `/next-task` appends one entry per finished task using this template:

```
## YYYY-MM-DD: <task id> <title>
- Done:
- Tests:
- Decisions:
- Dependencies added:
- Follow-ups:
```

## 2026-11-15: Planning package imported
- Done: design plan, decisions, roadmap, Antigravity rules and skills added to the repository. The design comes from the QR design discussion and builds on the LU decomposition project (LAPACK vs CubeCL on wgpu and the CubeCL CPU runtime).
- Tests: none yet.
- Decisions: D-001 to D-011 Accepted; D-012 to D-018 Proposed, to confirm in the tasks they name.
- Dependencies added: none.
- Follow-ups: run `/bootstrap-workspace` (M0.1), which also chooses the project name.

## 2026-11-15: Baselines added to the plan
- Done: after reviewing alternatives to CubeCL (no Rust library offers GPU QR on Apple Silicon; cuSOLVER is NVIDIA-only; MLX runs QR on the CPU), CubeCL stays. faer is added as a second CPU baseline (D-019) and cuSOLVER `geqrf` as the baseline for the later CUDA backend (D-020), both in `<prefix>-bench` only.
- Tests: none yet.
- Decisions: D-019, D-020.
- Dependencies added: none yet; `faer` 0.24 and optional `cudarc` 0.19 are listed in the bootstrap templates.
- Follow-ups: the faer wrapper is built in M0.5; the cuSOLVER wrapper with the CUDA backlog item.
