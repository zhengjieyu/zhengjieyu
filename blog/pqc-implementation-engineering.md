<div align="right">

**English** · [中文](./pqc-implementation-engineering.zh-CN.md)

</div>

# Engineering Post-Quantum Cryptography

## From NIST submissions to implementation pitfalls, side channels, and testing

*September 25, 2026 · @jieyu · 20 min read*

> Most PQC implementation bugs do not live in the mathematics. They appear at the boundary between specification and code—byte order, encoding, bounds checks, and rejection counters—or when optimization subtly changes semantics through overflow, lazy reduction, or compiler-introduced branches and division.

This tutorial is for researchers who can already write AVX2, AVX-512, or NEON code and want a systematic, high-assurance implementation methodology. The failures exposed during the NIST process, Korea's KpqC competition, and the first NGCC round are strikingly similar.

Suggested route: skim Sections 1–2 for context; study Sections 3–5 alongside your own implementation; turn Sections 6–7 into CI; then use Sections 8–9 as a reading list and release checklist.

Links marked as sources were checked while preparing the original article. Recheck primary sources before citing time-sensitive claims.

## 1. The standardization landscape

NIST took eight years—from its 2016 call to FIPS 203, 204, and 205 in 2024—to standardize the first PQC algorithms. Every round eliminated candidates because of either design or implementation problems. The speed of public review during NGCC, increasingly assisted by AI, means that a submission should be treated as immediately auditable.

### 1.1 NIST timeline

| Date | Event | Lesson for implementers |
| --- | --- | --- |
| 2026-07 | HAWK was withdrawn after an AI-assisted attack | Design review is accelerating too |
| 2026-05 | Nine additional-signature candidates advanced to round 3 ([NIST IR 8610](https://csrc.nist.gov/pubs/ir/8610/final)) | Constant-time and physical-attack requirements will tighten |
| 2025-09 | Sixth NIST PQC conference; FIPS 206 progress report | FN-DSA floating- and fixed-point questions remained active |
| 2025-03 | HQC selected as the second KEM | Track drafts until the standard is final |
| 2024-08-13 | FIPS 203 (ML-KEM), 204 (ML-DSA), and 205 (SLH-DSA) finalized | The standardized algorithms differ subtly from Kyber, Dilithium, and SPHINCS+ |
| 2022-07 | Kyber, Dilithium, Falcon, and SPHINCS+ selected | Algorithm agility is essential: SIKE and Rainbow were broken that year |
| 2017-12 | 69 of 82 first-round submissions accepted ([NIST IR 8240](https://nvlpubs.nist.gov/nistpubs/ir/2019/NIST.IR.8240.pdf)) | Public review rapidly finds both attacks and implementation bugs |
| 2016-12 | NIST issued the call for proposals | — |

As of September 2026, FIPS 206 remains unfinished and HQC has been selected but not yet standardized. Verify current status on the [NIST PQC project page](https://csrc.nist.gov/projects/post-quantum-cryptography).

### 1.2 Beyond NIST

- **China's NGCC:** launched in 2025. Its first-round review exposed many implementation, design, and side-channel issues. The [ngcc.dev report index](https://ngcc.dev/reports/index.html) is an unusually useful engineering dataset.
- **Korea's KpqC:** selected HAETAE and AIMer for signatures, and SMAUG-T and NTRU+ for KEMs. Its early use of Valgrind-based constant-time checks, leak detection, and metamorphic testing is a useful model ([ePrint 2023/1437](https://eprint.iacr.org/2023/1437.pdf)).
- **ISO, IETF, and national profiles:** FrodoKEM, Classic McEliece, hybrid TLS groups such as X25519MLKEM768, certificate identifiers, and deployment profiles add interoperability and encoding requirements that a standalone implementation can miss.

## 2. A map of the schemes

Remember families, not just names. Schemes in the same family tend to fail in the same places.

| Family | Representative schemes | Recurring implementation risks |
| --- | --- | --- |
| Module/Ring lattice KEM | ML-KEM, NTRU, NTRU Prime | NTT or polynomial multiplication, compression, FO transform, public-key validation |
| Code-based KEM | HQC, Classic McEliece, BIKE | Constant-time decoding, fixed-weight sampling, failure probability |
| Lattice signature | ML-DSA, FN-DSA | Rejection loops, hint encoding, floating-point or Gaussian sampling |
| Hash-based signature | SLH-DSA, LMS, XMSS | Address encoding, index truncation, state reuse, fault attacks |
| Multivariate signature | UOV, MAYO, QR-UOV, SNOVA | GF(16)/GF(256) tables and constant-time elimination |
| MPC/VOLE-in-the-head | FAEST, MQOM, SDitH | Seed trees, commitments, AES and field arithmetic |
| Isogeny-based | SQIsign | Large-integer and quaternion arithmetic, verifier input validation |

NIST solicited KEMs rather than a separate KEX category. In practice, authenticated key exchange is usually assembled at the protocol layer from a KEM and signatures, as in TLS 1.3, KEMTLS, or PQ-Noise.

## 3. The implementation workflow

Mature projects such as the PQ Code Package, PQClean, pqm4, liboqs, and BoringSSL/AWS-LC converge on the same order:

1. **Read the specification as a contract.** Rewrite every algorithm as pseudocode. Record type, length, range, endianness, validation rules, and domain-separation bytes.
2. **Build an executable specification.** Use Python, SageMath, hacspec, or Rust. Optimize nothing; mirror the specification line by line. Ideally, a different person writes this model.
3. **Write the portable reference implementation.** Use strict C99 warnings, check all external lengths, avoid undefined behavior, and make even the reference path constant-time where secrets are involved.
4. **Freeze behavior with tests.** Add KATs, intermediate-value vectors, negative vectors, and cross-platform checks before optimization.
5. **Optimize one kernel at a time.** Compare AVX2, AVX-512, NEON, Cortex-M, or RISC-V kernels against the reference on random and boundary inputs. Document input/output coefficient ranges.
6. **Harden and verify.** Add constant-time analysis, fuzzing, formal checks, and independent review.
7. **Release with traceability.** State the exact specification version, regenerate vectors after every tweak, publish errata, and keep regression tests for every fix.

### 3.1 Paper version versus standardized version

Subtle changes are security-relevant:

- **ML-KEM vs. Kyber round 3:** domain separation in key generation changed; encapsulation and shared-secret derivation changed; public-key and private-key validation was added.
- **ML-DSA vs. Dilithium round 3:** context strings and pre-hash mode were added; `tr` became 64 bytes; challenge-hash lengths vary by security level; hint decoding gained anti-malleability checks.
- **SLH-DSA vs. SPHINCS+ 3.1:** message hashing, address processing, and context handling changed.

Never label an implementation only with the family name. Pin a precise document revision and vector set.

## 4. Core operators and their invariants

### 4.1 Polynomial multiplication

For NTT-friendly rings such as ML-KEM and ML-DSA:

- State whether coefficients are signed or unsigned, and whether they are in the Montgomery domain.
- Write the maximum coefficient magnitude after every butterfly layer. Lazy reduction is fast, but it makes overflow analysis part of correctness.
- Verify twiddle order, bit reversal, incomplete NTT structure, and the combined inverse-scaling/Montgomery factor against a model.
- Do not mix Montgomery, Barrett, and Plantard reductions without proving the input and output ranges at every boundary.

For NTT-unfriendly rings such as NTRU, sntrup761, and Saber:

- Toom–Cook, Karatsuba, auxiliary-modulus NTTs, CRT, and mixed-radix methods all require explicit reconstruction bounds.
- A modulus that is large enough for honest keys may still fail for coefficients induced by malicious ciphertexts or public keys.
- Use constant-time divsteps or exponentiation for inversion; data-dependent extended Euclid is not acceptable for secrets.

For code-based schemes, sparse-by-dense multiplication must not index memory with secret support positions.

### 4.2 Sampling

- Public-seed uniform rejection sampling may be variable-time, but it must handle additional XOF blocks correctly, and vector tails must consume exactly the same bytes as the scalar implementation.
- Centered-binomial samplers need careful cross-byte handling—especially for parameters such as η=3.
- Fixed-weight sampling from a secret seed must be constant-time. A variable rejection count can reveal the seed.
- Gaussian samplers for FN-DSA-like designs must be checked for both distribution correctness and platform consistency; x87, FMA, subnormals, and soft-float can change output or timing.
- Rejection-loop counters for signature masks must never repeat or wrap.

### 4.3 Encoding and decoding

- Replace secret-dependent division in compression with verified multiply-and-shift sequences where needed.
- Require canonical encodings. A reliable rule is: decode, encode again, then compare byte for byte.
- Explicitly validate padding bits, unused high bits, sorted hint positions, coefficient ranges, and declared lengths.

### 4.4 Hash and XOF use

- Check output lengths against the claimed security level, remembering the birthday bound for collision security.
- Treat every domain-separation byte, nonce position, and length encoding as part of the algorithm.
- Compare every lane of x4/x8 Keccak against the scalar version.
- Test incremental squeezing exactly at SHAKE rate boundaries (168 bytes for SHAKE128, 136 for SHAKE256).

## 5. Failure patterns that repeatedly survive review

| Category | Typical symptom | Defense |
| --- | --- | --- |
| Security budget truncation | A 384/512-bit level uses a 256-bit seed; bit/byte confusion | Unit-bearing names and compile-time assertions |
| Broken FO transform | Partial ciphertext comparison or key selection; rejection key not bound to the full ciphertext | Flip every ciphertext bit and test deterministic implicit rejection |
| Randomness failure | Test DRBG reaches production; first encapsulation repeats | Separate test and production RNGs at build time; test repeated calls |
| Parsing and memory safety | Attacker-controlled count writes past the stack; failure returns success | Fuzz with ASan, UBSan, and MSan; validate before writing |
| Non-canonical encoding | Multiple signatures or public keys decode to the same value | Decode → re-encode → compare |
| Code/spec mismatch | Missing rounding constant, error polynomial, decoder stage, or address byte | Independent model plus intermediate-value differential tests |
| State reuse | Resettable protocol state or repeated one-time index | Make state transitions explicit and destructive |
| Protocol omission | Identity or transcript missing from the KDF | Bind the full transcript and peer identities |
| Debug residue | Secrets printed or attack scripts shipped | Review every file in the release archive |

### 5.1 Optimization-specific failures

1. **Range overflow:** delayed reduction and narrow SIMD lanes pass ordinary tests but fail on adversarial coefficients.
2. **Tail overread/overwrite:** dimensions such as 761 are not multiples of a vector width; padding can hide the bug from basic tests.
3. **Vectorized rejection tails:** shuffle-based compaction writes more lanes than are valid near the end of a buffer.
4. **Internal coefficient order:** an implementation is self-consistent but serializes a shuffled NTT representation and fails interoperability.
5. **Compiler-introduced branches:** source-level masks can become secret branches. Clang 15–18 compiled some Kyber code this way under selected options (CVE-2024-37880).
6. **Variable-time instructions:** division, modulo, early-exit multiplication, and floating-point corner cases can leak secrets.
7. **Undefined behavior:** signed overflow, left-shifting negative values, alignment violations, and strict-aliasing violations let compilers change semantics.
8. **Elided zeroization:** ordinary `memset` can be removed as dead code; use an approved explicit-zero primitive or barrier.
9. **Dispatch inconsistency:** scalar, AVX2, and AVX-512 paths must return identical bytes and error codes for malicious inputs.
10. **Embedded constraints:** stack exhaustion, unchecked TRNG failure, cache leakage, and variable-latency multiplication require target-specific testing.

## 6. Side-channel protection

For software, the minimum is secret-independent control flow, memory addressing, and variable-latency instruction operands—and this must be verified on the produced binary. The [NGCC constant-time review](https://ngcc.dev/constant-time/index.html) shows the same three leak classes repeatedly:

| Leak | Common pattern | Safer construction |
| --- | --- | --- |
| Secret branch | Sign branch, pivot choice, centered reduction, secret-dependent message tail | Arithmetic masks, conditional swaps, constant-time elimination |
| Secret address | S-box/T-table, sparse support, syndrome table, secret permutation | Bitslicing, full-table masked selection, sorting networks |
| Variable work or instruction | Division, modulo, rejection count, decoder iteration count | Multiply-and-shift, fixed iterations, constant-time fixed-weight sampling |

### 6.1 The most dangerous KEM pattern

In a Fujisaki–Okamoto KEM, any small signal correlated with the decrypted message—a branch, division, non-constant comparison, retry, return code, or timing difference—can become a plaintext-checking oracle. Thousands of queries may be enough to recover a private key.

From decryption through final key output, decapsulation should behave like straight-line code. Invalid ciphertexts must not produce a distinguishable observable behavior.

### 6.2 Compiler and microarchitecture

- Use reviewed value barriers for critical masks where the compiler has proved adversarial in practice.
- Test GCC and Clang across `-O0/-O1/-O2/-O3/-Os`, vectorization settings, and LTO.
- Consider speculative execution, frequency-based leakage such as Hertzbleed, and data-dependent prefetchers. Source-level constant time is not the finish line.

### 6.3 Physical attacks

Power, electromagnetic, and fault attacks matter on embedded targets. Mask conversion around compression, comparison, CBD, and rejection sampling is a major cost in lattice schemes. Use TVLA and platforms such as ChipWhisperer for leakage evaluation, and combine masking with fault checks, redundant critical comparisons, and randomized signing where the design permits it.

## 7. Testing, robustness, and specification conformance

A KAT only shows agreement on a small collection of ordinary deterministic inputs. Most adversarial and low-probability boundaries are untouched.

| Layer | Purpose | Useful tools |
| --- | --- | --- |
| 1. KAT | Agreement on normal inputs | NIST `PQCgenKAT` |
| 2. Intermediate vectors | Localize the first divergent step | ACVP, [C2SP CCTV](https://c2sp.org/CCTV/ML-KEM) |
| 3. Negative/boundary vectors | Non-canonical keys, malformed lengths, corrupted hashes, unlucky XOF streams | CCTV, [Wycheproof](https://github.com/C2SP/wycheproof), AWS-LC vectors |
| 4. Accumulated vectors | Millions of deterministic cases summarized by one digest | [Accumulated Test Vectors](https://words.filippo.io/accumulated/) |
| 5. Differential testing | Optimized vs. reference vs. independent model | Custom harness plus Sage/Python model |
| 6. Property/metamorphic tests | Round trips, canonical encoding, mutation rejection, fresh randomness | Property-based framework |
| 7. Fuzzing | Arbitrary-byte robustness and cross-implementation differences | libFuzzer, AFL++, honggfuzz |
| 8. Memory/UB | Bounds, uninitialized reads, integer overflow | Valgrind, ASan, UBSan, MSan |
| 9. Cross-platform | Endianness, word size, compiler differences | QEMU targets including s390x, i386, Arm |
| 10. Constant time | Secret branches, addresses, and variable-latency operands | TIMECOP/ctgrind, dudect, Binsec/Rel, Microwalk |
| 11. Formal verification | Functional correctness, memory safety, constant time | Jasmin/EasyCrypt, CBMC/HOL Light, hax/F*, Cryptol/SAW |

### 7.1 Constant-time tooling

- **ctgrind/TIMECOP:** mark secrets as undefined under Valgrind; report any branch or address depending on them. It is cheap enough for CI.
- **Patched Valgrind for KyberSlash-style issues:** additionally detects secret operands to variable-latency instructions.
- **dudect:** black-box timing and t-tests; useful on final binaries, though coverage depends on input classes.
- **Binary analysis:** Binsec/Rel, Microwalk, and ct-verif inspect behavior below the source level.
- Run every tool across the compiler/optimization matrix, or compiler-created leaks remain invisible.

### 7.2 Practical conformance method

1. Have someone other than the implementation author build the executable model from the specification.
2. Export intermediate state—expanded seeds, matrix, noise, pre/post-compression values, hash inputs—and compare step by step.
3. Turn every “MUST reject” sentence into a negative test and maintain requirements-to-tests traceability.
4. Experimentally check decryption-failure analysis on reduced or noise-amplified variants.
5. Ask an AI assistant to review code against the specification line by line; reviewers already do.

## 8. Reading and resources

If you read only three things, start with the PQClean experience paper ([ePrint 2022/337](https://eprint.iacr.org/2022/337)), Peter Schwabe's [Kyber implementation slides](https://cryptojedi.org/peter/data/cmmrs-20240801.pdf), and the [ngcc.dev findings](https://ngcc.dev/reports/index.html).

Other useful starting points:

- [KyberSlash](https://kyberslash.cr.yp.to/) and [ePrint 2024/1049](https://eprint.iacr.org/2024/1049)
- Filippo Valsorda, [Enough Polynomials and Linear Algebra to Implement Kyber](https://words.filippo.io/kyber-math/)
- [PQClean](https://github.com/PQClean/PQClean), pqm4, liboqs, libjade, mlkem-native, and libcrux
- Jancar et al., “They're not that hard to mitigate,” IEEE S&P 2022
- Bos et al., “Masking Kyber,” TCHES 2021
- Almeida et al., the Jasmin/EasyCrypt Kyber verification work
- The [NIST pqc-forum](https://groups.google.com/a/list.nist.gov/g/pqc-forum), IETF CFRG/TLS/LAMPS lists, PQCA, and Open Quantum Safe discussions

## 9. Pre-release checklist

Every item below corresponds to a failure reported in a public review.

### A. Security budget

- [ ] Seed, hash, challenge, salt, and shared-secret lengths meet the claimed level; collision-related lengths account for the birthday bound.
- [ ] Length constants carry units (`_BYTES`, `_BITS`).
- [ ] Failure-probability analysis matches rounding and compression in the implementation and is backed by experiments.

### B. KEM and FO transform

- [ ] Flipping every ciphertext bit produces a deterministic pseudorandom rejection key, never the valid key.
- [ ] Re-encryption comparison covers every ciphertext byte in constant time; conditional selection covers every key byte.
- [ ] The rejection KDF binds the complete ciphertext.
- [ ] The decrypt-to-output path has no secret branch, table lookup, division, or observable failure behavior.
- [ ] Public-key canonicality/modulus checks are implemented or explicitly justified by the specification.

### C. Signatures

- [ ] Decode → re-encode is byte-identical; padding, hints, and high bits are canonical.
- [ ] Hint counts and indices are validated before any memory write.
- [ ] Zero, all-`0xff`, truncated, and overlong signatures return an error without crashing.
- [ ] Rejection-loop nonces never repeat or wrap.

### D. Randomness and state

- [ ] The KAT DRBG is linked only in test builds; production RNG errors are checked.
- [ ] Consecutive key generation, encapsulation, and ephemeral-key calls produce different outputs.
- [ ] Protocol state cannot be reset or reused; identities and transcript enter the KDF.

### E. API and memory

- [ ] Caller-supplied lengths are validated against constants.
- [ ] Allocation and internal failures never return success; failure outputs are cleared.
- [ ] External input cannot reach `assert` or `abort`.
- [ ] Secret buffers use zeroization that survives optimization.
- [ ] Debug paths, prints, and global secrets are absent from the release archive.

### F. Optimized implementations

- [ ] Every kernel is differentially tested on random, extreme, and adversarial inputs.
- [ ] Input/output range invariants are documented and tested.
- [ ] Vector tails are covered by ASan even when padded buffers could hide an overrun.
- [ ] Reference, AVX2, AVX-512, and embedded paths return identical bytes and status codes.
- [ ] Embedded builds measure stack use and validate RNG and instruction timing assumptions.

### G. Constant time

- [ ] Secret-marking tests pass across the compiler and optimization matrix.
- [ ] Variable-latency instructions are checked at binary level.
- [ ] Every intentionally variable-time public operation is justified in the specification.
- [ ] Decapsulation, verification, and key parsing are continuously fuzzed.

### H. Specification consistency

- [ ] Independent executable model and intermediate-value comparisons pass.
- [ ] Every mandatory rejection rule has a negative test.
- [ ] Changes and errata are public; every fix includes a regression test.

Map this checklist into layers of your own test infrastructure: the baseline harness, independent-model differential tests, and a general robustness suite. Run them in CI—not only before submission.

---

### Primary source index

- [ngcc.dev reports](https://ngcc.dev/reports/index.html) and [constant-time review](https://ngcc.dev/constant-time/index.html)
- [NIST PQC project](https://csrc.nist.gov/projects/post-quantum-cryptography), [NIST IR 8610](https://csrc.nist.gov/pubs/ir/8610/final), and [NIST IR 8240](https://nvlpubs.nist.gov/nistpubs/ir/2019/NIST.IR.8240.pdf)
- [PQClean](https://github.com/PQClean/PQClean), [ePrint 2022/337](https://eprint.iacr.org/2022/337), and [ePrint 2023/1437](https://eprint.iacr.org/2023/1437.pdf)
- [KyberSlash ePrint 2024/1049](https://eprint.iacr.org/2024/1049), [Clangover](https://github.com/antoonpurnal/clangover)
- [C2SP CCTV](https://c2sp.org/CCTV/ML-KEM) and [Accumulated Test Vectors](https://words.filippo.io/accumulated/)

<div align="right">

[Back to profile](../README.md) · [阅读全文（中文）](./pqc-implementation-engineering.zh-CN.md)

</div>
