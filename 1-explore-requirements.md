# Request 1

*"I would like to design a library using Rust and CubeCL to perform QR decomposition. Please help me with the design decisions. What approach would work? How should the interface work? What are the key functions to implement first? Please grill me until you have enough information to suggest a design."*

# QR Decomposition Library with Rust and CubeCL: Design Notes

These notes record the design discussion for a QR decomposition library built with Rust and [CubeCL](https://github.com/tracel-ai/cubecl): the questions asked, the answers given, and the resulting design.

It builds on lessons from the earlier LU decomposition benchmark project:

- **CubeCL CPU runtime bugs:** the CPU runtime (`cubecl-cpu` 0.7) cannot compile kernels with nested loops, loops inside `if` blocks, or several loops in one kernel. It also defers loads, which breaks in-place swaps.
- **Launch overhead:** on the GPU, kernel launch overhead dominated small matrix sizes.
- **Blocking:** a blocked algorithm, which turns most of the work into a matrix multiply, gave the biggest speed-up.

---

## Round 1: Scope

### What is the QR decomposition mainly for?

| Option | Description |
|---|---|
| ✅ Benchmark vs LAPACK | Same setup as the LU project: compare against LAPACK `sgeqrf` for speed and accuracy. |
| ✅ Least-squares solves | Solve min ‖Ax − b‖. Needs R and the ability to apply Qᵀ to a vector, not Q itself. |
| ✅ Orthonormal basis | The explicit Q matrix is needed, for example for projections or Krylov methods. |
| ✅ Building block | Used inside a bigger algorithm such as eigenvalues (QR iteration) or SVD, so it is called many times. |

**Answer:** all four.

### What matrix shapes will it see most?

| Option | Description |
|---|---|
| ✅ Square, 128–4096 | Like the LU benchmark: one fairly large square matrix at a time. |
| ✅ Tall-skinny | Many more rows than columns, e.g. 100,000 × 64. Needs a different algorithm (TSQR) to run well on a GPU. |
| ✅ Many small matrices | Batches of thousands of small matrices, e.g. 32 × 32. Needs batched kernels, because one launch per matrix would be far too slow. |

**Answer:** all three.

### Which outputs must the library return?

| Option | Description |
|---|---|
| ✅ LAPACK-style compact | R in the upper triangle plus Householder vectors and τ below it, like `sgeqrf`. Smallest and fastest; Q is applied implicitly. |
| ✅ Explicit Q | Form Q as a full matrix (like `sorgqr`). Costs roughly as much again as the factorization. |
| ❌ R only | Only the triangular factor is needed, so nothing about Q has to be stored. |

**Answer:** LAPACK-style compact form and explicit Q.

### Which backends and precisions must it support?

| Option | Description |
|---|---|
| ✅ wgpu / Metal, f32 | The Mac's GPU. wgpu on Metal has no f64, so this means single precision only. |
| ✅ CubeCL CPU runtime | Same kernels on the CPU. Given its compiler bugs, this could constrain every kernel to very simple control flow. |
| ✅ CUDA / ROCm | NVIDIA or AMD GPUs. Allows f64 and larger cubes, but can't be tested on the Mac. |
| ✅ f64 as well | Double precision. Only available on CUDA, ROCm or the CPU runtime, not on Metal. |

**Answer:** all four.

> Choosing every option is a valid goal for the library as a whole, but those combinations pull in different directions. Many small matrices want one launch per batch, tall-skinny wants TSQR, and the CPU runtime wants trivial kernels. The next round set priorities.

---

## Round 2: Priorities

### What should version 1 do well? (The rest can come later on the same design.)

| Option | Description |
|---|---|
| ✅ **Square vs LAPACK** | Blocked Householder QR on one square matrix, benchmarked against `sgeqrf`. Reuses the LU benchmark setup. |
| Tall-skinny (TSQR) | Matrices like 100,000 × 64. Needs a reduction-tree algorithm; the most GPU-friendly case. |
| Batched small | Thousands of small matrices in one call. One cube per matrix, all in shared memory. |
| Least-squares solver | A `solve(A, b)` API first; the factorization is an internal detail. |

**Answer:** square matrices, benchmarked against LAPACK.

### Where should matrices live between calls?

| Option | Description |
|---|---|
| ✅ **Both, device first** | The core API takes GPU-resident tensors so calls can be chained without copying; thin host-slice wrappers on top for convenience and benchmarks. |
| Host slices only | Like the LU code: `&mut [f32]` in and out. Simplest, but every call pays two transfers, which hurts the building-block and batched cases. |
| Device tensors only | No host convenience layer; callers manage uploads and downloads themselves. |

**Answer:** both, with the device API first.

### How first-class should the CubeCL CPU runtime be, given its compiler bugs?

| Option | Description |
|---|---|
| ✅ **Test target only** | Run correctness tests on it where kernels compile; don't let it constrain GPU kernels or performance work. |
| First-class | Every kernel must compile on it. Forces one-loop, no-branch kernels everywhere, which costs GPU speed. |
| Separate CPU kernels | Like the LU project: dedicated CPU-friendly kernels behind the same API. |
| Fall back to LAPACK | On the CPU, call LAPACK directly; CubeCL is used only for GPUs. |

**Answer:** test target only.

### For the large matrix-multiply updates in blocked QR, what should be used?

| Option | Description |
|---|---|
| ✅ **cubecl-matmul** | CubeCL's own tuned matmul crate (in the default features since 0.7). Fastest route; adds a dependency whose API may change between versions. |
| Own tiled kernel | Like `gemm_update` in the LU code. Full control and fewer dependencies; slower than a tuned matmul. |
| Pluggable | A small trait so either can be used; start with our own kernel and swap later. |

**Answer:** `cubecl-matmul`.

---

## Round 3: Details that are hard to change later

### Is column pivoting (QR with column pivoting, LAPACK `sgeqp3`) needed for rank-deficient matrices?

| Option | Description |
|---|---|
| ✅ **Not in v1** | Plain QR like `sgeqrf`. Keep the API open so pivoting can be added later as a separate function. |
| Yes, needed | Rank-revealing QR. Much harder to block on a GPU, since each step needs updated column norms. |

**Answer:** not in version 1.

### What memory layout should the API accept?

| Option | Description |
|---|---|
| ✅ **Column-major only** | Same as LAPACK and the LU code. Results compare directly with `sgeqrf`; one layout to optimise. |
| Strided (any) | Accept row-major and views with arbitrary strides. More flexible, but every kernel needs stride handling. |

**Answer:** column-major only.

### How should errors be reported?

| Option | Description |
|---|---|
| ✅ **Result + error enum** | Return `Result<_, QrError>` for bad shapes, unsupported precision on a device, and launch failures. |
| Panic | Like the LU code's asserts. Simpler, but a library user can't recover. |

**Answer:** `Result` with an error enum.

### What makes a factorization "correct" in the tests?

| Option | Description |
|---|---|
| ✅ Residual ‖A − QR‖ | Backward error, like the LU residual check: about n·ε relative to ‖A‖. |
| ✅ Orthogonality ‖QᵀQ − I‖ | Checks Q is orthonormal. Householder keeps this near ε; Gram-Schmidt methods can lose it. |
| ✅ Match `sgeqrf` | Compare R (up to signs of rows) and τ against LAPACK on the same input. |

**Answer:** all three.

---

## Summary of decisions

| Decision | Answer |
|---|---|
| Version 1 | Square matrices from 128 to 4096, benchmarked against LAPACK `sgeqrf` |
| Later | Least squares, explicit Q, tall-skinny matrices, batched small matrices, use inside other algorithms |
| Outputs | LAPACK-style compact form (R plus Householder vectors and τ), and explicit Q |
| Targets | wgpu/Metal in f32 first; CUDA/ROCm and f64 later; the CubeCL CPU runtime for tests only |
| Data | Core API on GPU-resident data, with host-slice wrappers on top |
| Matrix multiply | `cubecl-matmul` |
| Layout and errors | Column-major only; return `Result` with an error enum |
| Pivoting | Not in version 1 |
| Correctness | Residual, orthogonality of Q, and agreement with `sgeqrf` |

---

## Proposed design

### Algorithm: blocked Householder QR with the compact WY form

This is the same structure LAPACK's `sgeqrf` uses. It suits the goals for three reasons:

- **Orthogonality:** Householder keeps ‖QᵀQ − I‖ close to machine precision. Gram-Schmidt methods (CGS, MGS) lose orthogonality on badly conditioned matrices, and orthogonality is one of the tests.
- **Matching LAPACK:** if the library follows LAPACK's sign convention for each reflector (`slarfg`), its R and τ should match `sgeqrf` up to rounding, not just up to the signs of rows.
- **Fast large updates:** almost all the arithmetic happens in a block update made of three matrix multiplies, which `cubecl-matmul` can do well. That's the blocking lesson from the LU work again.

For each panel of `nb` columns (64 to start with):

```
1. Factor the panel (column by column):
     for each column j in the panel:
       norm of A[j:, j]                  (reduction)
       build reflector v_j and τ_j       (slarfg)
       apply it to the rest of the panel (w = vᵀA, then A -= τ·v·wᵀ)
2. Build T, an nb × nb triangular matrix (slarft), so that the panel's
   nb reflectors combine into one: H = I − V·T·Vᵀ
3. Update everything to the right (slarfb):
     W = Vᵀ · A₂            matmul
     W = Tᵀ · W             small triangular multiply
     A₂ = A₂ − V · W        matmul
```

Version 1 accepts about three kernel launches per column during the panel factorization, much like the LU code. The trailing update is where the time goes for large n. Factoring tall panels with TSQR later will reduce the launch count, and that same code then gives the tall-skinny case.

### Interface (device first)

```rust
/// Column-major matrix on a device. `ld` (leading dimension) allows
/// submatrix views (panel, trailing block) without copying.
pub struct DeviceMatrix<R: Runtime, F: Float> {
    handle: Handle, rows: usize, cols: usize, ld: usize, _f: PhantomData<(R, F)>,
}

pub struct QrConfig { pub panel_width: u32, pub cube_dim: u32 }

/// Compact factorization: R in the upper triangle, reflectors below it, plus τ.
pub struct QrFactors<R: Runtime, F: Float> { a: DeviceMatrix<R, F>, tau: Handle, k: usize }

pub fn geqrf<R: Runtime, F: Float>(
    client: &ComputeClient<R::Server, R::Channel>,
    a: DeviceMatrix<R, F>,          // consumed: overwritten in place
    config: &QrConfig,
) -> Result<QrFactors<R, F>, QrError>;

impl<R: Runtime, F: Float> QrFactors<R, F> {
    pub fn r(&self) -> Result<DeviceMatrix<R, F>, QrError>;            // copy of R
    pub fn q(&self, thin: bool) -> Result<DeviceMatrix<R, F>, QrError>; // orgqr
    pub fn apply_qt(&self, b: &mut DeviceMatrix<R, F>) -> Result<(), QrError>; // ormqr
    pub fn solve_least_squares(&self, b: DeviceMatrix<R, F>)
        -> Result<DeviceMatrix<R, F>, QrError>;                        // QᵀB, then solve with R
}

pub enum QrError {
    InvalidShape { rows: usize, cols: usize, reason: &'static str },
    UnsupportedPrecision { device: String },   // e.g. f64 on Metal
    InvalidConfig(&'static str),               // e.g. cube_dim not a power of two
}

/// Host wrappers with LAPACK-like signatures, for benchmarks and quick use.
pub mod host {
    pub fn sgeqrf(a: &mut [f32], m: usize, n: usize) -> Result<Vec<f32>, QrError>; // returns τ
    pub fn sorgqr(a: &[f32], tau: &[f32], m: usize, n: usize) -> Result<Vec<f32>, QrError>;
    pub fn lstsq(a: &[f32], b: &[f32], m: usize, n: usize, nrhs: usize) -> Result<Vec<f32>, QrError>;
}
```

Why it's shaped this way:

- **One generic `F: Float`, checked at runtime.** f64 support costs nothing to design in, but Metal can't do it, so `geqrf` checks the device and returns `UnsupportedPrecision`.
- **Compact factors as the main result.** Explicit Q, applying Qᵀ, and least squares are all built on them, which matches how LAPACK separates `geqrf`, `orgqr` and `ormqr`.
- **One small wrapper module around `cubecl-matmul`.** Its API changes between releases, and keeping every call in one place limits upgrades to that file. The wrapper describes a column-major matrix as a strided tensor with shape `[rows, cols]` and strides `[1, ld]`, so a transpose is just a swap of strides. That's a second reason for `ld`.
- **Some errors can't be returned as `Result`.** In CubeCL 0.7 a launch failure is a panic inside CubeCL, so the error enum only covers what can be checked before launching.

### Crate layout

```
cubecl-qr/
  src/lib.rs            public API, QrError, QrConfig
  src/matrix.rs         DeviceMatrix, upload/download, views
  src/kernels/
    householder.rs      column norm, slarfg, apply one reflector
    larft.rs            build T for a panel
    trsm.rs             triangular solve (least squares)
  src/gemm.rs           the only file that calls cubecl-matmul
  src/blocked.rs        geqrf driver (panel loop)
  src/apply.rs          larfb, ormqr, orgqr
  src/host.rs           host-slice wrappers
  tests/                residual, orthogonality, vs-LAPACK checks
  benches/              port of the LU benchmark harness
```

### Build order

1. **`DeviceMatrix` and `QrError`:** upload, download and views, plus a test that copies a matrix to the device and back unchanged.
2. **Unblocked QR (`geqr2`):** the three Householder kernels and a column loop. Test against `sgeqrf` at n = 4, 16 and 128, and check the residual and orthogonality. To compute Q for those tests, start with a simple CPU function in f64 that builds Q from v and τ, so the factorization can be tested before `orgqr` exists.
3. **Blocked `geqrf`:** `larft`, plus `larfb` through `cubecl-matmul`. Then the benchmark against LAPACK, which is the goal of version 1.
4. **`orgqr`, `ormqr`, the triangular solve and `lstsq`:** all of them reuse `larfb`.
5. **Later:** TSQR panels and the tall-skinny API, batched kernels for small matrices, and f64 on CUDA.

### Lessons carried over from the LU work

- **CPU runtime:** run only the unblocked-path tests there, and mark tests whose kernels the 0.7 CPU compiler rejects as `#[ignore]` on that runtime. Unpivoted QR never swaps rows, so the deferred-load swap bug won't come up.
- **Warm-up:** do one untimed call for each configuration before timing, because of kernel compilation.
- **Norms in f32:** a plain sum of squares can overflow or underflow for extreme values. LAPACK's `snrm2` uses scaling to avoid that. It doesn't matter for [0, 1) test matrices, but use a scaled or two-pass norm before anyone else relies on the library.

### Open questions

- **Crate name:** not decided yet.
- **Testing f64:** there's no CUDA device on the Mac, so f64 needs other hardware or a CI runner.

Neither of these blocks milestones 1–3.




