# Optimization Catalog — References

Primary sources for the 275 ideas across `generic.md`, `compiler.md`,
`microarch.md`, `nn-llm.md`, `crypto-codecs.md`, `scientific.md`,
`alignment.md`. The bulk of these references were mined using
[`paperhound`](https://github.com/alexfdez1010/paperhound).

This file is **only** loaded when chasing a citation. The skill content itself
is citation-free so it stays compact in the agent's context.

---

## G — Compiler & Codegen

- G1, G12. Bataev. *LLVM Compiler for RISC-V Architecture* (Apress / Maker
  Innovations, 2025). DOI 10.1007/979-8-8688-2169-1_4.
- G2. Shih et al. "Register-Pressure Aware Predicator for Length Multiplier of
  RVV." LCPC 2022. DOI 10.1145/3547276.3548513.
- G3, G23. Lai, Lee, Hwang. "Enhancing LLVM Optimizations for Linear Recurrence
  Programs on RVV." ICPP-W 2023. DOI 10.1145/3605731.3605904.
- G4. Peccia, Haxel, Bringmann. "Tensor Program Optimization for the RISC-V
  Vector Extension Using Probabilistic Programs." ICCAD 2025. arXiv 2507.01457.
- G5. Yang et al. "Auto-tuning Fixed-point Precision with TVM on RISC-V Packed
  SIMD." ACM TODAES 2023. DOI 10.1145/3569939.
- G6. Ahmad et al. "Accelerating GenAI Workloads by Enabling RISC-V Microkernel
  Support in IREE." arXiv 2508.14899 (2025).
- G7. Lin, Yang, Lai, Lee. "Rewriting and Optimizing Vector Length Agnostic
  Intrinsics from Arm SVE to RVV." ICPP-W 2024. DOI 10.1145/3677333.3678151.
- G8. Fu, Hsu. "Translating Traditional SIMD Instructions to Vector
  Length Agnostic Architectures." CGO 2019. DOI 10.1109/cgo.2019.8661195.
- G9. van Kempen et al. "muRISCV-NN: Challenging Zve32x Autovectorization."
  CF 2024. DOI 10.1145/3637543.3652878.
- G10. Xu et al. "Zoozve: A Strip-Mining-Free RISC-V Vector Extension." LCTES
  2025. DOI 10.1145/3735452.3735526.
- G11. Shi, Schieffer et al. "High-performance Vector-length Agnostic Quantum
  Circuit Simulations on ARM Processors." arXiv 2602.09604 (2026).
- G13. Nunes et al. "Accelerating ML with RVV and Auto-Vectorization." ISCAS
  2025. DOI 10.1109/iscas56072.2025.11043225.
- G14. Volokitin et al. "Improved Vectorization of OpenCV Algorithms for
  RISC-V CPUs." arXiv 2311.12808 (2023).
- G15. Merouani et al. "LOOPer: A Learned Automatic Code Optimizer." arXiv
  2403.11522 (2024).
- G16. Heakl et al. "From CISC to RISC: LLM-Guided Assembly Transpilation."
  arXiv 2411.16341 (2024).
- G17. Cummins et al. "Large Language Models for Compiler Optimization." arXiv
  2309.07062 (2023).
- G18. Grubisic et al. "Compiler Generated Feedback for LLMs." arXiv
  2403.14714 (2024).
- G19. Tang. "An Open-Source RISC-V Vector Math Library." IEEE ARITH 2024.
  DOI 10.1109/arith61463.2024.00019.
- G20–G22, I33, H31. Poveda Rodrigo et al. "V-Seek: Accelerating LLM Reasoning
  on Open-hardware Server-class RISC-V Platforms." arXiv 2503.17422 (2025).
- G24. He, Markidis. "High-Performance FFT Code Generation via MLIR Linalg
  Dialect and SIMD Micro-Kernels." IEEE CLUSTER 2024. DOI
  10.1109/cluster59578.2024.00021.
- G25. Liu et al. "TinyIREE: An ML Execution Environment for Embedded Systems."
  IEEE Micro 2022. DOI 10.1109/mm.2022.3178068.
- G26, H19. Titopoulos et al. "Register Dispersion." CF 2025. DOI
  10.1145/3719276.3725181.
- G27. Lee, Jamieson, Brown. "Backporting RISC-V Vector Assembly." Euro-Par
  2023. DOI 10.1007/978-3-031-40843-4_32.
- G28. Alaejos et al. "Micro-kernels for portable and efficient matrix
  multiplication in deep learning." J. Supercomputing 2022. DOI
  10.1007/s11227-022-05003-3.
- G29. Martínez et al. "Inference with Transformer Encoders on ARM and
  RISC-V Multicore Processors." Euro-Par 2024. DOI
  10.1007/978-3-031-69766-1_26.
- G32. Szafraniec et al. "Code Translation with Compiler Representations."
  arXiv 2207.03578 (2022).
- G33. Das, Mannarswamy. "ML-driven Hardware Cost Model for MLIR." arXiv
  2302.11405 (2023).
- G34. Razilov, Matus, Fettweis. "Communications Signal Processing Using
  RVV." IWCMC 2022. DOI 10.1109/iwcmc55113.2022.9824961.
- G31. Bramas. "Inastemp." Scientific Programming 2017. DOI 10.1155/2017/5482468.

## H — Microarch / Energy / Autotune

- H1, H9. Brown et al. SC'23 workshop. arXiv 2309.00381; Lee et al. "RVV
  HPC survey." arXiv 2304.10319; SpacemiT K1 TRM (vendor).
- H2. Perotti et al. "Ara2." IEEE Trans. Comput. 2024. DOI
  10.1109/tc.2024.3388896.
- H3. He et al. "Prefetcher design for C908/C910." Electronics 2026. DOI
  10.3390/electronics15020319.
- H4. Gupta et al. arXiv 2311.05284.
- H5. Cavalcante et al. "Spatz." ICCAD 2022. DOI 10.1145/3508352.3549367;
  Perotti et al. (Spatz follow-up) IEEE TCAD 2025. DOI
  10.1109/tcad.2025.3528349.
- H6. SiFive X280 ISA manual (vendor).
- H7. Lee et al. arXiv 2304.10319; Lei (PhD thesis 2024) DOI
  10.4995/thesis/10251/212297.
- H8. Morgado et al. "CARM." IISWC 2024. DOI 10.1109/iiswc63097.2024.00016;
  Batashev. PaCT 2025. DOI 10.1007/978-3-032-06751-7_11.
- H10. Patterson & Williams (Berkeley PhD thesis 2008); Lei PhD thesis 2024.
- H11. Carneiro et al. arXiv 2604.04599 ("LP-GEMM").
- H12. Mach et al. "FPnew." IEEE TVLSI 2021. DOI 10.1109/tvlsi.2020.3044752.
- H14. Tan et al. IJCNN 2025. DOI 10.1109/ijcnn64981.2025.11228861; Liu &
  Karanth HiPC 2021. DOI 10.1109/hipc53243.2021.00037.
- H15. Brown arXiv 2508.13840 (SG2044); Brown & Jamieson LNCS 2024. DOI
  10.1007/978-3-031-73716-9_25.
- H16. Gupta et al. arXiv 2311.05284 (Winograd co-design).
- H17. Minervini et al. "Vitruvius+." ACM TACO 2022. DOI 10.1145/3575861.
- H18. Brown et al. SC'23 workshop. DOI 10.1145/3624062.3624234; Venieri et
  al. LNCS 2025 (Monte Cimone v2). DOI 10.1007/978-3-032-07612-0_44.
- H20. Titopoulos et al. "Vectorised FlashAttention on RVV." J.
  Supercomputing 2026. DOI 10.1007/s11227-026-08322-x.
- H22. Ashouri et al. ACM CSUR 2018. DOI 10.1145/3197978; Jaber & Jaber
  "AutoKernel." arXiv 2603.21331; Guerreiro et al. PDP 2015. DOI
  10.1109/pdp.2015.44.
- H25. Su & Zhang. "Lock-free Data Pipeline." Inf. Sci. 2026. DOI
  10.1016/j.ins.2026.123171.
- H28. Diehl et al. "HPX runtime on RISC-V." DOI 10.1145/3624062.3624230.
- H30. RAJA Performance Suite (LLNL, github.com/LLNL/RAJAPerf).
- H32. Mallasén et al. "PERCIVAL." IEEE TETC 2022. DOI 10.1109/tetc.2022.3187199.

## I — NN / LLM

- I1. Dao et al. "Flash-Decoding." https://crfm.stanford.edu/2023/10/12/
  flashdecoding.html; "FastAttention." arXiv 2410.16663.
- I3. "FireQ." arXiv 2505.20839; "EliteKV." arXiv 2503.01586.
- I4. "Effectively Compress KV Heads." arXiv 2406.07056; "MLKV." arXiv
  2406.09297.
- I5. "DeepSeek-V2." arXiv 2405.04434; "DeepSeek-V3." arXiv 2412.19437;
  "MC-MLA." DOI 10.1109/iscsic67494.2025.11351976.
- I6. "Context Parallelism." arXiv 2411.01783.
- I7. "KIVI." arXiv 2402.02750; "RotateKV." arXiv 2501.16383; "ZipCache."
  arXiv 2405.14256.
- I8. "ZipCache." arXiv 2405.14256; "Scissorhands." arXiv 2305.17118.
- I9. "vAttention." arXiv 2405.04437; "PagedEviction." arXiv 2509.04377.
- I10. "XQuant." arXiv 2510.11236; "KVTuner." arXiv 2502.04420.
- I11. "AWQ." arXiv 2306.00978; "Efficient LLM Inference on CPUs." arXiv
  2311.00502.
- I12. "SmoothQuant." arXiv 2211.10438; "SmoothQuant+." arXiv 2312.03788.
- I13. "SqueezeLLM." arXiv 2306.07629.
- I14. "QMoE." arXiv 2310.16795.
- I15. "DeepSeek-V3 Tech Report." arXiv 2412.19437.
- I16. "Efficient Routing in Sparse MoE." IJCNN 2024; "QMoE."
- I17. "Mamba." arXiv 2312.00752; "RWKV." arXiv 2305.13048.
- I18. "RWKV." arXiv 2305.13048; "GoldFinch." arXiv 2407.12077.
- I19. "Hydra." arXiv 2402.05109; "EAGLE-2." DOI
  10.18653/v1/2024.emnlp-main.422; "Recurrent Drafter." arXiv 2403.09919.
- I20. "EAGLE-2." Batch Speculative Decoding. arXiv 2510.22876.
- I21. "Sarathi-Serve." arXiv 2403.02310; "RAPID-Serve." arXiv 2601.11822.
- I22. "Relax." arXiv 2311.02103.
- I23. "BigBird." arXiv 2007.14062; "SWAT." arXiv 2405.17025; "LycheeDecode."
  arXiv 2602.04541.
- I25. "DejaVu." arXiv 2310.17157; "ShadowLLM." arXiv 2406.16635.
- I26. "PowerInfer." SOSP 2024. DOI 10.1145/3694715.3695964.
- I27. SwiGLU (Shazeer 2020); GELU. arXiv 1606.08415; "Integer SWIN."
  arXiv 2402.01169.
- I28. GELU (1606.08415); "GELU Analysis." arXiv 2305.12073.
- I29. "NOVA." arXiv 2512.18453.
- I31. "Relax." arXiv 2311.02103.
- I34. "MLKV." arXiv 2406.09297.
- I35. "RotateKV." arXiv 2501.16383; "XQuant." arXiv 2510.11236.

## J — Crypto / Codecs / DB

- J1. Marshall, Page, Pham. HASP 2020. DOI 10.1145/3458903.3458904.
- J1, J6, J12, J34. Kassimi et al. *Cryptography* 2026. DOI
  10.3390/cryptography10010006.
- J2. Jankowski, Laurent. IEEE TC 2011. DOI 10.1109/tc.2010.147; RISC-V Zbc
  spec v1.0 (github.com/riscv/riscv-crypto).
- J3, J35. Szymkowiak, Isufi, Saarinen. "Marian." CCS 2024. DOI
  10.1145/3658644.3691394; OpenSSL Zvkned (github.com/openssl/openssl).
- J4. Li, Mentens, Picek. DATE 2023. DOI 10.23919/date56975.2023.10137009;
  Bolat et al. ISVLSI 2025. DOI 10.1109/isvlsi65124.2025.11130308.
- J5, J9, J10. Zhang, Yan, Huang, Koç. TCHES 2025. DOI
  10.46586/tches.v2025.i1.632-655; Miteloudi et al. LNCS 2024. DOI
  10.1007/978-3-031-54409-5_10.
- J6. RISC-V Zksed/Zvksed spec v1.0; Liu et al. ACM 2025. DOI
  10.1145/3723890.3723919.
- J7. RISC-V Zksh/Zvksh spec.
- J8. Zinzindohoué et al. "HACL*." CCS 2017. DOI 10.1145/3133956.3134043;
  RISC-V Zvkb spec; BoringSSL RVV ChaCha20.
- J11. Grosso et al. FSE 2015. DOI 10.1007/978-3-662-46706-0_2; Rădulescu,
  Choudary. *Cryptography* 2022. DOI 10.3390/cryptography6030031.
- J13. Bhaskar et al. SPAA 2003. DOI 10.1145/777412.777458; Intel ISA-L
  (github.com/intel/isa-l).
- J14. Intel "Fast CRC Computation … PCLMULQDQ" (White Paper, 2011); Linux
  lib/crc32.c.
- J15. O'Connor, Hart, Aumasson, Samuel. BLAKE3 (github.com/BLAKE3-team/BLAKE3).
- J16. xxHash (github.com/Cyan4973/xxHash).
- J17. LZ4 (github.com/lz4/lz4); Sitaridi PhD 2016. DOI 10.7916/d8fn16bz.
- J18. Cameron. PPoPP 2008. DOI 10.1145/1345206.1345222; RISC-V Zbb spec.
- J19. BtrBlocks: Kuschewski et al. SIGMOD 2023. DOI 10.1145/3589263; zstd
  (github.com/facebook/zstd).
- J20. Lemire et al. "simdjson." VLDB 2019; simdjson (github.com/simdjson/
  simdjson).
- J21. Clausecker, Lemire. SPE 2023. DOI 10.1002/spe.3261; simdutf
  (github.com/simdutf/simdutf).
- J22. Hildebrandt, Habich, Lehner. "BOUNCE." 2023. DOI 10.1007/
  s10619-023-07426-0.
- J23. Shanbhag, Yogatama, Yu, Madden. SIGMOD 2022. DOI 10.1145/3514221.3526132;
  DuckDB (github.com/duckdb/duckdb).
- J24. Liu, Zeng, Zhang. "LeCo." SIGMOD 2024. DOI 10.1145/3639320.
- J25. Cui, Li, Qian. IEEE Access 2023. DOI 10.1109/access.2023.3246491; DPDK.
- J26. Salmon et al. SC 2011; Google Highway library.
- J27. Namazi Rizi, Zidarič, Batina, Mentens. DDECS 2024. DOI
  10.1109/ddecs60919.2024.10508919.
- J28. Apache Parquet format spec.
- J29. Volokitin et al. arXiv 2311.12808 (2023); libjpeg-turbo PR #706.
- J31. Ekdahl, Johansson, Maximov, Yang. ToSC 2019. DOI
  10.46586/tosc.v2019.i3.1-42.
- J32. Muła, Lemire. JCSE 2018. arXiv 1611.07612; CRoaring
  (github.com/RoaringBitmap/CRoaring).
- J33. Lemire, Suxiu. "Roaring Bitmap." SPE 2016.

## K — Scientific / HPC

- K1. Monakov et al. (2010) auto-tuning SpMV; Besta et al. "SlimSell." arXiv
  2010.09913.
- K2. Bell, Garland. SC 2009; Rodrigues et al. Euro-Par 2023.
- K3. Srivastava et al. "MatRaptor." MICRO 2020.
- K4. El Maarouf et al. Euro-Par 2025; Lu et al. ICPP 2020.
- K5. Bertolacci et al. PPoPP 2015; Matsumura et al. "AN5D." CGO 2020.
- K6. Williams et al. "LBM auto-tuning." SC 2011.
- K8. Williams et al. SC 2011; Calore et al. Concurr. Comput. 2016.
- K9. Oyarzun PhD thesis 2015 (heterogeneous CFD).
- K10. Teyssier "RAMSES." A&A 2002; Rudi et al. SC 2015.
- K11. Kronbichler, Wall. SIAM J. Sci. Comput. 2018; Kronbichler, Sashko,
  Munch. IJHPCA 2022.
- K12. Yang. Lect. Notes Comput. Sci. Eng. 2006; Rudi et al. SC 2015.
- K13. Kronbichler, Sashko, Munch. IJHPCA 2022.
- K14. Demmel, Hoemmen "CA-CG" PhD 2010.
- K15. Ghysels, Vanroose. Parallel Comput. 2014.
- K17. Li et al. "AutoFFT." IEEE TPDS 2020.
- K18. FFTW3 (Frigo, Johnson) Proc. IEEE 2005.
- K20. Yokota et al. CPC 2009; Schaller et al. "SWIFT." MNRAS 2024.
- K21. Singh et al. JPDC 1995.
- K22. Hess et al. "GROMACS 4." JCTC 2008; Abraham et al. "GROMACS 5."
  SoftwareX 2015.
- K23. Abraham et al. SoftwareX 2015; Simmonett, Brooks. JCP 2021.
- K24. Matsumoto, Nishimura. ACM TOMACS 1998.
- K26. Besta et al. arXiv 2010.09913; Beamer, Asanovic, Patterson. SC 2012.
- K27. "Fast triangle counting with SIMD." Adv. Comput. 2023.
- K28. Bailey "double-double" LBL 1995; Hida, Li, Bailey. ARITH-15 2001;
  Kouya. ARITH 2021.
- K29. Rump "INTLAB" 1999.
- K30. Knuth TAOCP Vol 2.
- K31. Baboulin et al. CPC 2008; Vieuble PhD 2022.
- K33. Low et al. "Analytical Modeling for BLIS." ACM TOMS 2016; Xu, Van Zee,
  van de Geijn. ICS 2023.
- K34. Buttari et al. Parallel Comput. 2008.
- K35. Stock et al. IPDPS 2011; Kühne et al. "CP2K." 2020.
- K36. Heinecke et al. "LIBXSMM." 2016.
- K39. Warren, Salmon. CPC 1995.

## L — Sequence Alignment

- L1, L33. Farrar. Bioinformatics 2007. DOI 10.1093/bioinformatics/btl582.
- L1, L36. Zhao et al. PLOS ONE 2013. DOI 10.1371/journal.pone.0082138
  (libssw).
- L7, L40. Wozniak. Comput. Appl. Biosci. 1997.
- L8, L20, L34. Rognes. "SWIPE." BMC Bioinformatics 2011. DOI
  10.1186/1471-2105-12-221.
- L9, L36. Daily. "Parasail." BMC Bioinformatics 2016. DOI
  10.1186/s12859-016-0930-z.
- L13, L41. Suzuki, Kasahara. "libgaba." BMC Bioinformatics 2018. DOI
  10.1186/s12859-018-2014-8.
- L15, L16. Li. "ksw2 / minimap2." Bioinformatics 2018. DOI
  10.1093/bioinformatics/bty191.
- L17. Marco-Sola et al. "WFA." Bioinformatics 2021; BiWFA 2023.
- L24, L25. Myers. "Bit-parallel edit distance." J. ACM 1999; Hyyrö. (LCS).
- L24. Šošić. "edlib." Bioinformatics 2017.
- L19. Liu et al. "block-aligner." 2023.
- L26, L27. Eddy. "HMMER3." 2011.
- L29. Standard SW traceback; libssw 2-bit pack.
- L30. Hirschberg. CACM 1975.
- L31. Carneiro et al. (Pair-HMM in GATK).
- L39. Ahmed et al. "GASAL2." Bioinformatics 2019; "mm2-ax." 2021.
- L42. Perotti et al. Ara2 IEEE TC 2024; Vizcaino et al. SC'23 workshop. DOI
  10.1145/3624062.3624231.

## RISC-V Vector / Hardware Platform References

- RVV 1.0 ratified spec. github.com/riscv/riscv-v-spec.
- RVV / RVA22 ISA. github.com/riscv/riscv-isa-manual.
- Ara2. Perotti et al. IEEE TC 2024. DOI 10.1109/tc.2024.3388896.
- Vitruvius+. Minervini et al. ACM TACO 2022. DOI 10.1145/3575861.
- Spatz. Cavalcante et al. ICCAD 2022 + IEEE TCAD 2025.
- Sophon SG2042. Brown et al. SC'23 workshop. DOI 10.1145/3624062.3624234;
  Brown, Jamieson. LNCS 2024.
- Sophon SG2044. Brown. arXiv 2508.13840 (2025).
- Monte Cimone v2. Venieri et al. LNCS 2025. DOI 10.1007/978-3-032-07612-0_44.
- V-Seek (LLM on RISC-V). Poveda Rodrigo et al. arXiv 2503.17422.
- EoK / IntrinTrans / LLM-assisted RVV. Chen 2025 (arXiv 2509.14265); Han
  2026 (arXiv 2510.10119); Pan 2024 (DOI 10.1145/3597503.3639226).

## Methodology

Six parallel sub-agents ran ~10–16 paperhound searches across distinct topic
clusters (compiler/codegen, NN/LLM, crypto/codecs, scientific/HPC,
microarchitecture/energy/autotuning, SSW/bioinformatics) and fetched full-text
Markdown for the highest-signal papers via `paperhound get -o`. Reference
repositories inspected: libssw, parasail, ksw2, WFA2-lib, edlib, libgaba,
llama.cpp, IREE, OpenSSL Zvkned.
