# Request

*"In my original request I asked to use CubeCl. Do you think this is the best Rust library to use for QR with GPU, or are there better options now?"*

# Response

Yes, for what you're building, CubeCL is still the best choice. I searched for current alternatives, and none of them covers your combination of requirements: GPU QR on your Mac now, the same code on NVIDIA and AMD later, f64 when the hardware allows it, and pure Rust.

## How the alternatives compare

| Option | Runs on your Mac's GPU? | QR on the GPU? | Verdict |
| --- | --- | --- | --- |
| **CubeCL** | Yes (wgpu → Metal) | You write it; matmul is provided | **Best fit:** one codebase for Metal, CUDA, ROCm and Vulkan |
| cuSOLVER via [cudarc](https://docs.rs/crate/cudarc/0.19.9) or [baracuda-cusolver](https://docs.rs/baracuda-cusolver) | No, NVIDIA only | Yes, NVIDIA's own `geqrf` | Fastest on NVIDIA, but cudarc only exposes it as raw C bindings and baracuda is alpha. Not usable on your machine |
| [oxicuda-solver](https://docs.rs/crate/oxicuda-solver/0.1.0) | No, CUDA | Its own blocked Householder | Young, CUDA-only |
| MLX ([mlxrs](https://docs.rs/crate/mlxrs/0.1.0/source/src/ops/linalg_full.rs)) | Yes | **No**: QR is "not yet supported on the GPU" and runs on the CPU | Doesn't solve the problem |
| Metal Performance Shaders | Yes | As far as I know, only LU and Cholesky, no QR | Doesn't solve the problem |
| [numr](https://dev.to/farhansyah/numr-050-the-rust-numerical-computing-library-that-doesnt-make-you-choose-cpp) | Yes (WebGPU) | Lists GPU "decompositions", but QR isn't confirmed | Worth watching; too new to build on |
| Raw wgpu + WGSL | Yes | You write it all, matmul included | What CubeCL generates for you, with more work and no CUDA path |
| Burn, Candle | Burn yes, Candle partly | No GPU QR found | Tensor frameworks for ML, not linear algebra |

The surprising result is that **there's no ready-made GPU QR for Apple Silicon in Rust**, or even in Apple's own MLX. Only a small third-party project, [qr-apple-silicon](https://c0rmac.github.io/qr-apple-silicon/), offers one, and it's built on MLX rather than Rust. So writing your own kernels isn't a detour; on a Mac it's the only route. Among the ways to write your own, CubeCL saves you the most work: you write each kernel once in Rust, and you get a tuned matrix multiply.

## CubeCL's weaknesses (the design already covers them)

- **API churn between releases:** the design pins the version and has the `/upgrade-cubecl` skill.
- **An immature CPU runtime:** the design uses it only as a test target.
- **Launch overhead on wgpu:** the performance model and milestone M4 deal with it.

None of these is a reason to switch; they're why those parts of the design exist.

## Two additions I'd suggest

1. **[faer](https://docs.rs/faer) as a second CPU baseline.** It's a pure-Rust linear algebra library with a very fast blocked QR. Comparing against both LAPACK and faer shows where your GPU code stands against the best CPU code written in Rust as well as in Fortran and C.
2. **cuSOLVER as the CUDA baseline later.** When you reach the f64-on-CUDA backlog item, cuSOLVER `geqrf` through cudarc is the natural yardstick for the CUDA backend. It would sit in the bench crate only, like LAPACK.

## Sources

- [GitHub topic: qr-decomposition](https://github.com/topics/qr-decomposition)
- [numr 0.5.0 announcement](https://dev.to/farhansyah/numr-050-the-rust-numerical-computing-library-that-doesnt-make-you-choose-cpp)
- [nalgebra QR](https://michael-f-bryan.github.io/constraints/nalgebra/linalg/struct.QR.html)
- [ndarray-linalg QR](https://docs.rs/ndarray-linalg/latest/ndarray_linalg/qr/index.html)
- [cudarc 0.19.9](https://docs.rs/crate/cudarc/0.19.9)
- [atomr-accel-cuda cuSOLVER source](https://docs.rs/crate/atomr-accel-cuda/0.4.2/source/src/sys/cusolver.rs)
- [baracuda-cusolver](https://docs.rs/baracuda-cusolver)
- [baracuda-kernels QR plan](https://docs.rs/crate/baracuda-kernels/latest/source/src/linalg/qr.rs)
- [oxicuda-solver](https://docs.rs/crate/oxicuda-solver/0.1.0)
- [NVIDIA cuSOLVER API reference](https://docs.nvidia.com/cuda/archive/13.4.1/cusolver/index.html)
- [mlxrs linalg source](https://docs.rs/crate/mlxrs/0.1.0/source/src/ops/linalg_full.rs)
- [MLX issue #1392: Metal SVD](https://upd.dev/ml-explore/mlx/issues/1392)
- [qr-apple-silicon](https://c0rmac.github.io/qr-apple-silicon/)
- [Portable GPU SVD kernels (arXiv 2508.06339)](https://arxiv.org/html/2508.06339v1)
- [OminiX-MLX framework docs](https://www.mintlify.com/OminiX-ai/OminiX-MLX/concepts/mlx-framework)


# Kit update: faer and cuSOLVER baselines

Both baselines are now in the kit. The updated `cubecl-qr-antigravity-kit.zip` replaces the copy in the `lu-decomp-gpu` folder.

## What's new

**Two new decisions:**

- **D-019 faer (Accepted).** faer 0.24, MIT-licensed, becomes a second CPU baseline next to LAPACK. It's compared on speed and on the ρ_res and ρ_orth ratios, not entry by entry: faer's sign convention for R may differ from LAPACK's `slarfg`, so LAPACK stays the only reference for matching R and τ.
- **D-020 cuSOLVER (Accepted).** cuSOLVER `geqrf`, through `cudarc` 0.19 (MIT OR Apache-2.0), is the yardstick for the later CUDA backend. It sits behind an opt-in `cuda` feature in the bench crate that's never enabled on macOS, in CI or by default. It's built together with the f64-on-CUDA backlog item, not before.

**Where the changes landed:**

| File | Change |
| --- | --- |
| `AGENTS.md` | Invariant 1: the bench crate is the only home for LAPACK, faer and cuSOLVER |
| `docs/roadmap.md` | M0.5 adds a faer wrapper, checked with the test ratios; M2.6 and the M2 gate benchmark against faer as well as LAPACK; the CUDA backlog item includes the cuSOLVER baseline |
| `docs/design-plan.md` | A "Baselines" row in the decisions table, `faer.rs` and `cusolver.rs` in the workspace layout, and an updated architecture diagram and roadmap summary |
| `.agents/rules/dependencies.md` | The faer and cudarc rules, including "never enable `cuda` on macOS" |
| `.agents/rules/benchmarking.md` | New CSV backend values (`faer`, `cusolver`, `cubecl-cuda`), faer's thread count, and the timing rule for cuSOLVER |
| `.agents/skills/benchmark` | A `--baselines` flag (default `lapack,faer`) and the steps for a CUDA run |
| `.agents/skills/accuracy-check` | Why faer is checked with ratios only |
| Bootstrap templates | `faer` and optional `cudarc` (with `cusolver`) in the workspace and bench `Cargo.toml` |
| `docs/progress.md` | An entry recording the review of alternatives and why CubeCL stays |

**Checked:**

- All 20 decisions and 28 tasks referenced across the kit exist.
- All six TOML templates parse.
- The design plan's 21 formulas still pass the brace check.
- faer and cudarc are both on the licence allowlist.

