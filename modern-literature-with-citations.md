# Literature overview: QR decomposition on GPUs with Rust and CubeCL

As of 15 November 2026. This overview collects the literature behind the design: Householder QR and its blocked form, how it is tested, how it maps onto GPUs, the newer mixed-precision and randomized variants, portable GPU programming, and Rust. Every entry has a citation; the matching BibTeX entries are in `qr-references.bib`.

**Verification key.** ✔ = title, authors and venue checked online on 15 November 2026. ◇ = a standard reference cited from memory; check the details (especially page numbers and DOIs) against the publisher's page before submitting.

## 1. Foundations of Householder QR

Householder introduced the reflector that annihilates a column below its first entry [1]. Golub turned it into the standard method for linear least squares [2]. The textbooks of Golub and Van Loan [3] and of Trefethen and Bau [4] give the algorithm in the form our design follows: unblocked `geqr2`, the sign choice for β, and least squares through R. Higham proves that Householder QR is backward stable, with error bounds proportional to the dimension times the unit roundoff [5, ch. 19], which is why our residual and orthogonality ratios use a fixed threshold.

1. ◇ A. S. Householder. Unitary triangularization of a nonsymmetric matrix. *Journal of the ACM*, 5(4):339–342, 1958. doi:10.1145/320941.320947
2. ◇ G. H. Golub. Numerical methods for solving linear least squares problems. *Numerische Mathematik*, 7(3):206–216, 1965. doi:10.1007/BF01436075
3. ◇ G. H. Golub and C. F. Van Loan. *Matrix Computations*, 4th edition. Johns Hopkins University Press, 2013. ISBN 978-1-4214-0794-4
4. ◇ L. N. Trefethen and D. Bau III. *Numerical Linear Algebra*. SIAM, 1997. doi:10.1137/1.9780898719574
5. ◇ N. J. Higham. *Accuracy and Stability of Numerical Algorithms*, 2nd edition. SIAM, 2002. doi:10.1137/1.9780898718027

## 2. Blocked QR and the compact WY form

Bischof and Van Loan showed that a product of Householder reflectors can be written as I + WYᵀ, so a panel's reflectors can be applied with matrix multiplies [6]. Schreiber and Van Loan reduced the storage to the compact form I − VTVᵀ with T upper triangular [7]; this is exactly the `larft`/`larfb` scheme in our design and in LAPACK `sgeqrf` [8]. Elmroth and Gustavson showed that recursive panel factorization improves performance further [9], an option for the fused panel kernel in milestone M4.

6. ◇ C. Bischof and C. Van Loan. The WY representation for products of Householder matrices. *SIAM Journal on Scientific and Statistical Computing*, 8(1):s2–s13, 1987. doi:10.1137/0908009
7. ◇ R. Schreiber and C. Van Loan. A storage-efficient WY representation for products of Householder transformations. *SIAM Journal on Scientific and Statistical Computing*, 10(1):53–57, 1989. doi:10.1137/0910005
8. ◇ E. Anderson, Z. Bai, C. Bischof, S. Blackford, J. Demmel, J. Dongarra, J. Du Croz, A. Greenbaum, S. Hammarling, A. McKenney and D. Sorensen. *LAPACK Users' Guide*, 3rd edition. SIAM, 1999. doi:10.1137/1.9780898719604
9. ◇ E. Elmroth and F. G. Gustavson. Applying recursion to serial and parallel QR factorization leads to better performance. *IBM Journal of Research and Development*, 44(4):605–624, 2000. doi:10.1147/rd.444.0605

## 3. Safe norms and accuracy testing

The scaled norm in `make_reflector` follows the line of work on overflow-safe Euclidean norms that began with Blue [10] and was revisited for the reference BLAS by Anderson [11]. The test ratios ρ_res and ρ_orth are those used by LAPACK's own QR tests (routine `sqrt01` in the LAPACK test suite) [8], and their thresholds rest on Higham's backward-error analysis [5].

10. ◇ J. L. Blue. A portable Fortran program to find the Euclidean norm of a vector. *ACM Transactions on Mathematical Software*, 4(1):15–23, 1978. doi:10.1145/355769.355771
11. ◇ E. Anderson. Algorithm 978: Safe scaling in the Level 1 BLAS. *ACM Transactions on Mathematical Software*, 44(1), article 12, 2017. doi:10.1145/3061665

## 4. Dense linear algebra on GPUs

Volkov and Demmel's study of GPU matrix multiply and factorizations established that blocked algorithms reach high efficiency on GPUs when the trailing update is a large GEMM [12], the basis of our reliance on `cubecl-matmul`. The MAGMA project brought LAPACK-style factorizations, QR included, to hybrid CPU–GPU systems [13, 14], and the tiled algorithms of PLASMA [15] show the alternative of dividing the matrix into tiles scheduled as a task graph.

12. ◇ V. Volkov and J. W. Demmel. Benchmarking GPUs to tune dense linear algebra. In *Proceedings of the 2008 ACM/IEEE Conference on Supercomputing (SC08)*, 2008. doi:10.1109/SC.2008.5214359
13. ◇ S. Tomov, J. Dongarra and M. Baboulin. Towards dense linear algebra for hybrid GPU accelerated manycore systems. *Parallel Computing*, 36(5–6):232–240, 2010. doi:10.1016/j.parco.2009.12.005
14. ◇ E. Agullo, J. Demmel, J. Dongarra, B. Hadri, J. Kurzak, J. Langou, H. Ltaief, P. Luszczek and S. Tomov. Numerical linear algebra on emerging architectures: The PLASMA and MAGMA projects. *Journal of Physics: Conference Series*, 180:012037, 2009. doi:10.1088/1742-6596/180/1/012037
15. ◇ A. Buttari, J. Langou, J. Kurzak and J. Dongarra. A class of parallel tiled linear algebra algorithms for multicore architectures. *Parallel Computing*, 35(1):38–53, 2009. doi:10.1016/j.parco.2008.10.002

## 5. Communication-avoiding QR and tall-skinny matrices

TSQR, our backlog item for tall-skinny input, comes from the communication-avoiding QR work of Demmel, Grigori, Hoemmen and Langou, which proves communication lower bounds and gives optimal algorithms [16]. Anderson, Ballard, Demmel and Keutzer implemented it on GPUs [17]. Ballard and colleagues survey the communication lower bounds across numerical linear algebra [18]. A March 2026 study by Thies and Röhrig-Zöllner compares TSQR with Gram-matrix methods on current NVIDIA GPUs and finds TSQR competitive but demanding to optimize [19].

16. ◇ J. Demmel, L. Grigori, M. Hoemmen and J. Langou. Communication-optimal parallel and sequential QR and LU factorizations. *SIAM Journal on Scientific Computing*, 34(1):A206–A239, 2012. doi:10.1137/080731992
17. ◇ M. Anderson, G. Ballard, J. Demmel and K. Keutzer. Communication-avoiding QR decomposition for GPUs. In *Proceedings of the IEEE International Parallel & Distributed Processing Symposium (IPDPS)*, 2011. doi:10.1109/IPDPS.2011.15 ([ResearchGate](https://www.researchgate.net/publication/224257718_Communication-Avoiding_QR_Decomposition_for_GPUs))
18. ◇ G. Ballard, E. Carson, J. Demmel, M. Hoemmen, N. Knight and O. Schwartz. Communication lower bounds and optimal algorithms for numerical linear algebra. *Acta Numerica*, 23:1–155, 2014. doi:10.1017/S0962492914000038
19. ✔ J. Thies and M. Röhrig-Zöllner. Implementation of QR factorization of tall and very skinny matrices on current GPUs. arXiv:2603.20889, March 2026. [arxiv.org/abs/2603.20889](https://arxiv.org/abs/2603.20889)

## 6. Batched QR for many small matrices

Haidar, Dong, Luszczek, Tomov and Dongarra developed batched Cholesky, LU and QR that run entirely on the GPU as sequences of batched BLAS calls, reporting up to 5× over cuBLAS batched QR on a K40 [20]. This is the reference for our "batched small matrices" backlog item.

20. ✔ A. Haidar, T. Dong, P. Luszczek, S. Tomov and J. Dongarra. Batched matrix computations on hardware accelerators based on GPUs. *International Journal of High Performance Computing Applications*, 29(2):193–208, 2015. doi:10.1177/1094342014567546 ([ORNL record](https://www.ornl.gov/publication/batched-matrix-computations-hardware-accelerators-based-gpus))

## 7. Mixed precision, tensor cores and randomized QR

This is the most active area. Zhang, Baharlouei and Wu found that a plain Householder QR does not benefit much from tensor cores, and that recursive Gram-Schmidt with scaling and refinement reaches 3.0×–14.6× over cuSOLVER [21, 22]. A later IEEE paper extends tensor-core QR to FP32 and FP64 inputs, switching to Householder to keep orthogonality [23]. Benner, Köhler, Liu and Oktay analyse block Householder QR in reduced and mixed precision and show that norm computations need fp32 accumulation to avoid overflow and underflow [24], which supports our scaled-norm rule. Randomization is a second direction: randomized Householder QR halves the cost of Householder QR with one synchronization per step [25], a 2025 survey covers randomized orthogonalization [26], and a randomized preconditioner makes Cholesky-QR stable on GPUs at high condition numbers [27]. Higham and Mary [28] and Abdelfattah and colleagues [29] survey mixed precision in numerical linear algebra. CholeskyQR2 [30, 31] is the GEMM-heavy alternative those GPU papers compare against.

21. ✔ S. Zhang, E. Baharlouei and P. Wu. High accuracy matrix computations on neural engines: A study of QR factorization and its applications. In *Proceedings of the 29th International Symposium on High-Performance Parallel and Distributed Computing (HPDC '20)*, pages 17–28. ACM, 2020. doi:10.1145/3369583.3392685 (author order as given by the [LATER project page](https://www2.cs.uh.edu/~panruowu/later.html); one catalogue lists Baharlouei first)
22. ✔ S. Zhang and P. Wu. High accuracy low precision QR factorization and least square solver on GPU with TensorCore. arXiv:1912.05508, 2019. [arxiv.org/abs/1912.05508](https://arxiv.org/abs/1912.05508)
23. ✔ (authors not verified) High performance Householder QR factorization on emerging GPU architectures using tensor cores. IEEE Xplore document 10816084. [ieeexplore.ieee.org/document/10816084](https://ieeexplore.ieee.org/document/10816084). The title and abstract were confirmed; check the authors, venue and year on IEEE Xplore before citing.
24. ✔ P. Benner, M. Köhler, X. Liu and E. Oktay. Reduced and mixed-precision block Householder QR factorization. arXiv:2610.38537, October 2026. [arxiv.org/abs/2610.38537](https://arxiv.org/abs/2610.38537)
25. ✔ L. Grigori and E. Timsit. Randomized Householder QR. arXiv:2405.10923, 2024. [arxiv.org/abs/2405.10923](https://arxiv.org/abs/2405.10923)
26. ✔ J. G. de Damas, L. Grigori, I. Simunec and E. Timsit. Randomized orthogonalization and Krylov subspace methods: principles and algorithms. arXiv:2512.15455, December 2025. [arxiv.org/abs/2512.15455](https://arxiv.org/abs/2512.15455) (first-name initials of the first and third authors from memory)
27. ✔ J. E. Garrison, C. Chen and I. C. F. Ipsen. Numerically stable Cholesky-QR on GPU via mixed-precision randomized preconditioning. arXiv:2606.18411, June 2026. [arxiv.org/abs/2606.18411](https://arxiv.org/abs/2606.18411)
28. ◇ N. J. Higham and T. Mary. Mixed precision algorithms in numerical linear algebra. *Acta Numerica*, 31:347–414, 2022. doi:10.1017/S0962492922000022
29. ◇ A. Abdelfattah, H. Anzt, E. G. Boman, E. Carson, T. Cojean, J. Dongarra, A. Fox, M. Gates, N. J. Higham, X. S. Li, J. Loe, P. Luszczek, S. Pranesh, S. Rajamanickam, T. Ribizel, B. F. Smith, K. Swirydowicz, S. Thomas, S. Tomov, Y. M. Tsai and U. M. Yang. A survey of numerical linear algebra methods utilizing mixed-precision arithmetic. *International Journal of High Performance Computing Applications*, 35(4):344–369, 2021. doi:10.1177/10943420211003313
30. ◇ T. Fukaya, Y. Nakatsukasa, Y. Yanagisawa and Y. Yamamoto. CholeskyQR2: A simple and communication-avoiding algorithm for computing a tall-skinny QR factorization on a large-scale parallel system. In *Proceedings of the 5th Workshop on Latest Advances in Scalable Algorithms for Large-Scale Systems (ScalA)*, 2014. doi:10.1109/ScalA.2014.11
31. ◇ Y. Yamamoto, Y. Nakatsukasa, Y. Yanagisawa and T. Fukaya. Roundoff error analysis of the CholeskyQR2 algorithm. *Electronic Transactions on Numerical Analysis*, 44:306–326, 2015.

## 8. Portable GPU programming

The closest published precedent for our approach, one kernel code base across GPU vendors and precisions, is the unified SVD kernels of Ringoot, Alomairy, Churavy and Edelman, written in Julia and presented at ICPP 2025 [32]. Kokkos is the established C++ portability layer for the same goal [33]. CubeCL itself has no paper; cite the repository with the version used [34]. Its wgpu backend targets the WebGPU standard and its WGSL shading language [35].

32. ✔ E. Ringoot, R. Alomairy, V. Churavy and A. Edelman. Performant unified GPU kernels for portable singular value computation across hardware and precision. In *Proceedings of the 54th International Conference on Parallel Processing (ICPP 2025)*, pages 500–510. ACM, 2025. arXiv:2508.06339. [arxiv.org/abs/2508.06339](https://arxiv.org/abs/2508.06339)
33. ◇ C. R. Trott, D. Lebrun-Grandié, D. Arndt, J. Ciesko, V. Dang, N. Ellingwood, R. Gayatri, E. Harvey, D. S. Hollman, D. Ibanez, N. Liber, J. Madsen, J. Miles, D. Poliakoff, A. Powell, S. Rajamanickam, M. Simberg, D. Sunderland, B. Turcksin and J. Wilke. Kokkos 3: Programming model extensions for the exascale era. *IEEE Transactions on Parallel and Distributed Systems*, 33(4):805–817, 2022. doi:10.1109/TPDS.2021.3097283
34. ✔ Tracel AI. CubeCL: multi-platform high-performance compute language extension for Rust. Software; this project uses version 0.7 (give the release date of the version you cite). [github.com/tracel-ai/cubecl](https://github.com/tracel-ai/cubecl)
35. ◇ W3C GPU for the Web Working Group. *WebGPU* and *WebGPU Shading Language (WGSL)*. W3C Candidate Recommendations. [w3.org/TR/webgpu](https://www.w3.org/TR/webgpu/), [w3.org/TR/WGSL](https://www.w3.org/TR/WGSL/)

## 9. Rust for scientific and GPU computing

Perkel's *Nature* feature documents why scientists adopt Rust: memory safety without a garbage collector and C-like speed [36]. RustBelt gives the formal foundation of Rust's safety guarantees, including for `unsafe` code wrapped in safe abstractions [37], which is the pattern our `launch_unchecked` wrappers follow. faer, our second CPU baseline, is described in its JOSS paper [38]. The Rust book is the standard language reference [39].

36. ◇ J. M. Perkel. Why scientists are turning to Rust. *Nature*, 588:185–186, 2020. doi:10.1038/d41586-020-03382-2
37. ◇ R. Jung, J.-H. Jourdan, R. Krebbers and D. Dreyer. RustBelt: Securing the foundations of the Rust programming language. *Proceedings of the ACM on Programming Languages*, 2(POPL), article 66, 2018. doi:10.1145/3158154
38. ✔ S. Quiñones El Kazdadi. faer: A linear algebra library for the Rust programming language. *Journal of Open Source Software*, 11(123):6099, 2026. doi:10.21105/joss.06099
39. ◇ S. Klabnik and C. Nichols. *The Rust Programming Language*. No Starch Press, 2nd edition, 2023. Also at [doc.rust-lang.org/book](https://doc.rust-lang.org/book/)

## 10. Baseline libraries and vendor documentation

40. ◇ LAPACK: Linear Algebra PACKage, version 3.12. Software, netlib. [netlib.org/lapack](https://www.netlib.org/lapack/) (see also [8])
41. ✔ NVIDIA. cuSOLVER API Reference, CUDA Toolkit 13.4. [docs.nvidia.com/cuda/archive/13.4.1/cusolver](https://docs.nvidia.com/cuda/archive/13.4.1/cusolver/index.html)
42. ✔ cudarc: safe Rust wrappers for the CUDA driver, cuBLAS and cuSOLVER. Software, version 0.19. [docs.rs/cudarc](https://docs.rs/crate/cudarc/0.19.9)
43. ◇ Apple. Accelerate framework documentation (BLAS and LAPACK). [developer.apple.com/documentation/accelerate](https://developer.apple.com/documentation/accelerate)

## How the sources support the design

| Design element | Sources |
| --- | --- |
| Householder reflectors and the `slarfg` sign choice | [1], [3], [8] |
| Backward stability and fixed test thresholds | [5], [8] |
| Compact WY form (`larft`, `larfb`) and blocking | [6], [7], [8] |
| Recursive or fused panel factorization (M4) | [9], [21], [23] |
| Scaled norm in `make_reflector` | [10], [11], [24] |
| Blocked trailing update as large GEMMs on the GPU | [12], [13] |
| TSQR for tall-skinny input (backlog) | [16], [17], [18], [19] |
| Batched QR (backlog) | [20] |
| Mixed precision and tensor cores (future work) | [21]–[24], [28], [29] |
| Randomized and Cholesky-based alternatives | [25], [26], [27], [30], [31] |
| Portable kernels across vendors and precisions | [32], [33], [34], [35] |
| Rust, safety and the faer baseline | [36], [37], [38], [39] |
| LAPACK and cuSOLVER baselines | [8], [40], [41], [42], [43] |

## Notes on what the search found

- No peer-reviewed paper describes CubeCL; cite its repository and version [34].
- No paper or library provides GPU QR in Rust or on Apple Silicon (see the separate CubeCL alternatives review), so a design like ours fills a gap.
- The 2026 arXiv preprints [19], [24], [27] are recent and not yet peer reviewed; cite them as preprints and check for published versions later.


# References


