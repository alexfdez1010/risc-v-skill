# J — Crypto, Codecs, Compression, Hashing & DB/Packet Processing

35 ideas exploiting Zbb/Zba/Zbc/Zbs, Zbkb/Zbkc/Zbkx, Zk* (Zkne/Zknd/Zknh/Zksed/
Zksh), Zvk* (Zvkn/Zvks/Zvkb/Zvkg/Zvkh), and RVV 1.0 base.

---

## Symmetric crypto — scalar (Zk family)

**J1. AES-128/256 via Zkne/Zknd with round-key interleaving.** `aes64es` /
`aes64esm` / `aes64ds` / `aes64dsm` encode one full AES round. Schedule 2–4
independent blocks back-to-back to fill the throughput gap → ~1 block per 10
cycles on dual-issue X60. Round keys pre-loaded in x-registers.

**J2. AES-GCM via Zkne + Zbc clmul GHASH.** AES-CTR with `aes64esm`; GHASH
with `clmul` + `clmulh` and Karatsuba reduction modulo the GCM polynomial.
Batch 4 GHASH updates with the H, H², H³, H⁴ table held in 8 scalar regs.

**J4. SHA-256/512 via Zknh.** `sha256sig0/1` / `sha256sum0/1` collapse each σ/Σ
chain (4–8 rotates+XORs) into one instruction. Schedule 4 lanes for
dual-issue.

**J5. SHA-3 / Keccak via Zbb (or custom).** Software path: bit-interleaved
state + lane complementing; `rev8` for output byte-swap; `ror`/`rol` for ρ
rotations. Custom path: 1-round `xkeccakp` opcode → 46× over vanilla.

**J6. SM4 via Zksed.** `sm4ed` / `sm4ks` embed the GF(256) S-box; eliminates
software T-table cache-timing channel.

**J7. SM3 via Zksh.** `sm3p0`/`sm3p1` collapse P0/P1 permutation (4 XOR + 2
ROL) to one instruction each. Vector counterpart via Zvksh.

**J10. Plantard PQC modular mult via Zbb.** Replaces Montgomery's MULH+MULD+
shift with MULH+shift+cond-add — fills both dual-issue slots, ~15 % faster
NTT butterfly than scalar Montgomery on X60.

**J12. DOM-masked AES on Zkne.** Two-share Domain-Oriented Masking; schedule
both shares' `aes64esm` chains cycle-by-cycle so timing is identical between
shares. 0 % timing overhead on CV32E40S.

**J14. CRC32C via Zbc clmul folding.** 64-bit-at-a-time `clmul`/`clmulh`
against precomputed fold constants; Barrett-reduced 32-bit residue in ~120
instructions. ~8× vs naive byte-at-a-time.

**J16. xxHash3 via Zbb.** `mul` + `ror` + `andn` chain; vector path keeps 8
lanes in registers, `vmul.vv` + `vxor.vv` + `vror.vi`, final `vredxor.vs`.

**J18. Huffman canonical decode via Zbb CLZ + `vrgather`.** `clz` for code
length detection (replaces 3–8 compare chain); 256-entry symbol table in `m2`
group looked up via `vrgather.vx`. Chain bit-buffer refill with `sll` + `or`.

**J19. zstd FSE/ANS decode via Zbb REV8 + RVV.** `rev8` replaces the 4-inst
software byte-swap; multi-stream ANS state lookup via `vluxei8.v` over
parallel decode states.

**J30. Protobuf varint decode.** `clz` (Zbb) gives length without branches.
Batch: `vle8.v` packed varints, `vmsne.vi` detects continuation bits,
`viota.m` computes byte offsets, `vluxei8.v` gathers each varint's start.

**J32. Population count.** Scalar `cpop` (Zbb) replaces 5-instr Hamming-weight
idiom on RV64. Bulk: `vcpop.m` over LMUL=m8 groups counts 8·VLEN bits per
instruction. 4 MB Bloom filter probe: ~500 K → ~2 K instructions.

**J33. Bit-extract via Zbs + RVV.** Scalar `bext` replaces `srl + and`; bulk
extract via `vsrl.vv` + `vand.vi` + `vcompress.vm` for non-zero packing.
Parquet RLE_DICTIONARY decode is the hot site.

---

## Symmetric crypto — vector (Zvk family)

**J3. AES on Zvkned.** `vaesef.vv` / `vaesem.vv` / `vaesdf.vv` / `vaesdm.vv` at
LMUL=m2 encrypt 2–4 blocks per `vsetvli`. CTR nonces via `vadd.vi` on a counter
vector; expanded key via `vaeskf1/2.vi`. Throughput scales linearly with VLEN.

**J8. ChaCha20-Poly1305 on RVV + Zvkb.** Quarter-round: `vadd.vv` +
`vxor.vv` + `vrol.vv` (Zvkb). 4 keystream blocks at LMUL=m4 @ VLEN=256 → 512
B/iter. Poly1305 via scalar `clmul`.

**J9. Kyber NTT/INTT on RVV.** 16-bit Montgomery reduction: `vmul.vv` +
`vsra.vi` + `vssub.vv` over n=256 coefficients. Barrett via `vmulh.vv`.
Pre-load 4 levels of twiddles via `vluxei`; 16 butterflies per vector iter
replacing 256-iter scalar loop.

**J11. Bitsliced AES via RVV mask registers.** 128 AES instances bit-sliced;
each plane → one mask register. SubBytes = `vmand.mm` / `vmxor.mm` /
`vmnot.m` chains. ShiftRows / MixColumns = register-rename permutations. No
data-dependent branches, no table loads → constant-time.

**J13. GF(2⁸) Reed-Solomon via Zvkb + `vrgather`.** Split-radix nibble tables
preloaded as vector regs; two 16-entry `vrgather` lookups + `vxor.vv` realises
GF mul. RS(255, k) symbol row at LMUL=m4. ~2.6× over scalar.

**J15. BLAKE3 leaf hashing via Zvknh + Zvkb.** Compression function ≡
ChaCha20 quarter-round on 16-word u32 state. Map each leaf onto a separate
vector lane; `vadd.vv` / `vxor.vv` / `vrol.vv` execute quarter-rounds across
leaves in parallel. LMUL=m4 → 8 leaves/cycle on VLEN=256.

**J20. simdjson-style JSON parsing.** `vle8.v` 64-byte chunks; `vmsne.vi`
for each structural char; OR masks; `vcpop.m` counts structures; `vfirst.m`
locates first delimiter; `vmsbf` / `vmsif` / `vmsof.m` for string-interior
masking.

**J21. UTF-8 validation + transcoding.** Two-stage nibble table via `vsrl` +
`vand` + `vrgather.vv` (replaces PSHUFB); overlong/surrogate detection via
`vand.vv` + `vmseq.vi`. UTF-8→UTF-32 via masked `vsext.vf4`; `vcompress.vm`
packs valid code points.

**J22. Parquet bit-packing (FOR + BP).** Per-chunk min via `vredmin.vs`;
`vsub.vv` to zero base; pack into bit_width fields via `vsll` + `vor`
interleave. Decode: `vsra.vi` + `vand.vi`. Width-agnostic (any bit width
without recompile).

**J23. Hash-join probe via `vluxei`.** Gather hash-table entries with `vluxei32/
64.v`; compare keys with `vmseq.vv`; `vcompress.vm` collects matches. Chain
multiple hash-key loads across LMUL to overlap hash compute with memory.

**J24. WHERE-clause predicate evaluation.** `vmsgt.vi` / `vmslt.vi` /
`vmseq.vv` chains; accumulate masks via `vmand.mm`; `viota.m` + `vcompress.vm`
projects passing row indices. Branchless; 64 int32 rows / iter at
VLEN=256/m2.

**J25. DPDK packet classification.** Zbb `andn` / `orn` per-field masks; bulk
mbuf scan via `vlse32.v` strided load over the descriptor array; `vmsne.vv` +
`viota.m` extract matched flow IDs across a 32-packet burst.

**J26. Parallel xorshift / Philox PRNG.** VLEN/64 independent xorshift128+
states in two `m1` registers; `vsll` + `vxor` + `vsrl` + `vxor` advances 8
lanes/iter at VLEN=256. Philox-4×32-10: 10 fixed `vmul`/`vxor`/`vror` rounds
in vector regs.

**J27. AES-CTR for Classic McEliece via Zvkned.** Multi-block CTR with
expanded keys in `m1` regs; `vaesem.vv` across 4 state regs; counter increment
via `vadd.vi`. Reported 5–8× over scalar AES for McEliece keygen.

**J28. Parquet DELTA_BINARY_PACKED via prefix scan.** Hillis-Steele on delta
block: `vle32.v` → `vslideup` + `vadd.vv` doublings → `vadd.vx` block base.
128 values in 7 slide passes vs 128 scalar adds.

**J29. JPEG AAN IDCT via int16 RVV.** 8 rows × `m1` int16; 11 `vmul.vx`
multiplies + 29 `vadd`/`vsub` + `vsra.vi` descale per 1D IDCT. 8 row + 8
column passes per 8×8 block in ~112 vector instructions.

**J31. SNOW-V via Zvkned LFSR + AES round.** Map 16×128-bit LFSR onto Zvkned
regs; `vaesem.vv` for FSM update; `vxor.vv` for keystream XOR. 8 LFSR steps
pipelined ahead of 1 AES round (8:1 ratio). Targets 5G UPF/gNB.

**J34. Constant-time discipline for RVV crypto.** Always use `ta` / `ma`
policies; never branch or loop on `vl` derived from secret-length material;
`vfirst.m` only on non-secret masks. Avoids new RVV-specific timing oracle
where variable-length semantics leak through `vsetvli vl`.

**J35. AES-GCM batched authentication via Zvkg.** `vghsh.vv` / `vgmul.vv`
encode GF(2¹²⁸) MAC directly in vector regs. At LMUL=m1 eew=128 process
VLEN/128 independent GHASH steps per instruction; 4 multi-session H values
in one register group.

---

## Routing inside this file

| Domain | Ideas |
| --- | --- |
| TLS / VPN encrypt | J1, J3, J8, J35 |
| Hashing (crypto) | J4, J5, J7, J15 |
| Hashing (non-crypto) | J16 (xxHash3), J32 (popcount) |
| Post-quantum | J9 (Kyber NTT), J10 (Plantard), J27 (McEliece) |
| Constant-time / side channel | J11 (bitsliced AES), J12 (DOM), J34 (VLA timing) |
| Erasure / FEC | J13 (Reed-Solomon), J14 (CRC32C) |
| Compression | J17 (LZ4), J18 (Huffman), J19 (zstd FSE) |
| Database / analytics | J22, J23, J24, J28, J33 |
| Codec / streaming | J20 (JSON), J21 (UTF-8), J29 (JPEG IDCT) |
| Networking | J25 (DPDK), J30 (protobuf), J31 (SNOW-V) |

## Extension → idea index

| Extension | Ideas |
| --- | --- |
| Zkne / Zknd | J1, J3, J12, J27 |
| Zbc (clmul) | J2, J8, J14 |
| Zknh | J4 |
| Zksh / Zksed | J6, J7 |
| Zbb | J5, J10, J16, J18, J19, J22, J25, J30, J32, J33 |
| Zbs | J33 |
| Zbkb / Zvkb | J8, J15, J16, J26 |
| Zvkned | J3, J27, J31 |
| Zvknh | J4, J15 |
| Zvksed / Zvksh | J6, J7 |
| Zvkg | J35 |
| Base RVV (no crypto ext) | J9, J11, J13, J17, J18, J20–J24, J28–J30, J32, J34 |

**J17. LZ4 match-copy via `vlse` + `vslideup`.** Literal phase: bulk `vle8.v`
loads. Match phase: when offset < VL, `vslide1up`/`vslideup` replicates the
short-match pattern across the output; for offset ≥ VL, direct `vle8.v` copy.
~80 % of typical match offsets >32 B fall into the bulk-copy path.
