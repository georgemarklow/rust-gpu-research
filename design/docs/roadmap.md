# Roadmap

Milestones run in order. `/next-task` picks the first unticked task of the first unfinished milestone. Each task should fit one or two evening sessions; split bigger ones into lettered sub-tasks (M2.5a, M2.5b) before starting. The drivers accept m ≥ n from M1 on; version 1 benchmarks square matrices.

Status: **not started.**

---

## M0 Foundations

**Gate:** a matrix uploaded to the wgpu device comes back unchanged, and the f64 reference QR matches LAPACK `sgeqrf` and passes the test ratios at n = 4, 128 and 1024.

- [ ] **M0.1 Bootstrap.** Choose the project name and check the crates.io names, then create the workspace, crates, xtask, CI and license policy (`/bootstrap-workspace`). Confirm D-012 (CubeCL version), D-013 (LAPACK backends), D-016 (GPU tests in CI), D-017 (license) and D-018 (name). *Done when:* `cargo xtask verify` passes locally and in CI, and those decisions are Accepted or replaced.
- [ ] **M0.2 Device matrix and errors.** `DeviceMatrix` (handle, offset, rows, cols, ld) with upload, download and views; `QrError`; `QrConfig` with defaults and validation (`cube_dim` a power of two, `panel_width` ≥ 1). *Done when:* a round-trip test passes on wgpu and on the CPU runtime, views address the right elements, and invalid shapes and configs return the right `QrError`.
- [ ] **M0.3 f64 reference QR.** A pure-Rust f64 module (`reference.rs`, behind `cfg(any(test, feature = "reference"))`): `slarfg`, unblocked `geqr2`, `larft`, forming Q from v and τ, and applying Qᵀ. *Done when:* unit tests show ‖A − QR‖ and ‖I − QᵀQ‖ near f64 rounding for random and edge-case matrices.
- [ ] **M0.4 Test matrices and ratios.** Seeded generators for every case in the design plan's test-matrix table (random, partial last panel, zero column, identity and diagonal, ill-conditioned with known singular values, huge and tiny entries), plus `rho_res`, `rho_orth`, `delta_r` and `delta_tau`. *Done when:* each generator and ratio has a unit test.
- [ ] **M0.5 Baseline harness.** In `<prefix>-bench`: safe wrappers for LAPACK `sgeqrf` and `sorgqr`, and for faer's QR (D-019) returning Q and R; a test comparing the f64 reference (rounded to f32 input) with `sgeqrf`. *Done when:* the reference matches `sgeqrf` within D-010's tolerances for n = 4 to 1024, which proves the `slarfg` convention, and faer's Q and R pass ρ_res and ρ_orth < 30 at the same sizes.
- [ ] **M0.6 Gate.** `/milestone-gate`.

## M1 Unblocked QR on the GPU

**Gate:** `geqr2` on wgpu passes `/accuracy-check` and matches `sgeqrf` at n = 4, 16 and 128.

- [ ] **M1.1 make_reflector kernel.** One cube per column: scaled two-pass norm, then β, τ and v in place (`/add-kernel`). *Done when:* single-column results match the f64 reference for random columns, a zero column (τ = 0), a column already reduced, and entries near 10³⁰ and 10⁻³⁰.
- [ ] **M1.2 Panel reflector application.** `panel_dots` (wᵀ = vᵀA₂, one reduction per column) and `panel_update` (A₂ ← A₂ − τ v wᵀ). *Done when:* applying one reflector to a block matches the reference.
- [ ] **M1.3 Unblocked driver.** `geqr2` column loop producing `QrFactors` (matrix plus τ buffer); `host::sgeqrf` calls it for now. Accuracy tests go in `crates/<prefix>/tests/accuracy.rs`. *Done when:* `/accuracy-check` passes at n = 4, 16 and 128 and for a 300 × 100 matrix.
- [ ] **M1.4 CPU-runtime pass.** Run the M1 tests on the CubeCL CPU runtime; mark any that its compiler rejects as ignored on that runtime, with the reason. *Done when:* every M1 test either passes on the CPU runtime or is ignored there with a comment, and `docs/progress.md` lists which.
- [ ] **M1.5 LAPACK agreement.** `<prefix>-bench` tests (`tests/lapack_agreement.rs`) compare GPU `geqr2` with `sgeqrf` on the same f32 input. *Done when:* δ_R and δ_τ are within D-010's tolerances at n = 4, 16 and 128.
- [ ] **M1.6 Gate.** `/milestone-gate`.

## M2 Blocked QR (version 1)

**Gate:** blocked `geqrf` passes `/accuracy-check` for n = 100 to 4096 (including sizes that are not multiples of nb), and a benchmark CSV with plots compares it with LAPACK `sgeqrf` and faer at n = 128, 256, 512, 1024, 2048 and 4096.

- [ ] **M2.1 cubecl-matmul spike.** Find how the pinned `cubecl-matmul` multiplies strided, possibly transposed f32 matrices on wgpu. *Done when:* a decision entry records the call pattern, and a throwaway test multiplies two column-major views correctly.
- [ ] **M2.2 gemm wrapper.** `gemm.rs` with the two products the blocked algorithm needs, W = VᵀA₂ and A₂ ← A₂ − VW, on `DeviceMatrix` views. *Done when:* both match f64 CPU products for square and rectangular shapes, including views with ld > rows.
- [ ] **M2.3 V and T.** `materialize_v` (unit lower trapezoidal copy of the panel) and `build_t` (S = VᵀV through `gemm.rs`, then the `slarft` recurrence in one cube). *Done when:* T matches the reference `larft` for b = 1, 7, 32 and 64.
- [ ] **M2.4 Triangular multiply.** `trmm_t`, W ← TᵀW with T upper triangular. *Done when:* it matches a CPU product, including b not a power of two.
- [ ] **M2.5 Blocked driver.** `geqrf` panel loop with the workspace (V, T, W) allocated once, a partial last panel, and the trailing update; `host::sgeqrf` switches to it. *Done when:* `/accuracy-check` passes for n = 100, 128, 1000, 1024, 2048 and 4096, and the result matches the M1 unblocked driver.
- [ ] **M2.6 Benchmark harness.** `/benchmark`: the bench binary with the LAPACK and faer baselines, warm-up, 10 timed repetitions, the CSV schema in `.agents/rules/benchmarking.md`, and linear and log-log plots. Sweep nb = 32, 64, 128 and cube sizes 64, 128, 256. *Done when:* a full run writes the CSV and plots, and D-014 is Accepted or replaced.
- [ ] **M2.7 Gate.** `/milestone-gate`, including the benchmark evidence.

## M3 Using the factors

**Gate:** explicit Q passes the orthogonality ratio and least-squares solutions match LAPACK `sgels`, for square and tall matrices up to 2048 rows.

- [ ] **M3.1 Apply Q and Qᵀ.** `apply_qt` and `apply_q` (ormqr): panels forward with Tᵀ, backward with T. Decide D-015 (keep or rebuild T). *Done when:* QᵀA reproduces R and Q(QᵀC) reproduces C within the residual threshold.
- [ ] **M3.2 Explicit Q.** `q(thin)` (orgqr) from the identity block, panels backward. *Done when:* `rho_orth` < 30 and Q·R reproduces A, for thin and full Q.
- [ ] **M3.3 R and back substitution.** `r()` and a `trsm` kernel for R₁X = C with several right-hand sides; `QrError::RankDeficient` for a zero or tiny diagonal entry. *Done when:* solutions match the f64 reference, and a rank-deficient input returns the error.
- [ ] **M3.4 Least squares and host wrappers.** `solve_least_squares`, `host::lstsq` and `host::sorgqr`. *Done when:* solutions and residual norms match `sgels` for m = n and m = 2n, up to 2048 rows.
- [ ] **M3.5 Gate.** `/milestone-gate`.

## M4 Performance

**Gate:** the benchmark at n = 4096 is measurably faster than the M2 result, with the launch count per column reduced.

- [ ] **M4.1 Launch cost.** Measure the fixed cost of one kernel launch on wgpu/Metal (1,000 empty launches). *Done when:* a decision entry records the number and updates the performance model in the design plan.
- [ ] **M4.2 Fused panel kernel.** Factor a whole column, or a whole panel, in fewer launches. *Done when:* `/accuracy-check` still passes everywhere and the launches per column drop.
- [ ] **M4.3 Re-benchmark.** Repeat the M2.6 sweep. *Done when:* the CSV and plots show the change against M2, and D-014 is revisited.
- [ ] **M4.4 Gate.** `/milestone-gate`.

## Backlog (not scheduled)

- TSQR for tall-skinny matrices, also usable as the panel factorization.
- Batched QR for many small matrices, one cube per matrix in shared memory.
- f64 on CUDA or ROCm, with a CI runner that has such a GPU. Includes the cuSOLVER `geqrf` baseline through `cudarc` behind the bench crate's `cuda` feature (D-020).
- QR with column pivoting (QRCP) as a separate function.
