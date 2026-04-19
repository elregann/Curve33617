# Curve33617

A Montgomery curve defined over the prime field `p = 2^336 - 17`.

| Parameter | Value |
|---|---|
| Field size | 336-bit |
| Byte representation | 42 bytes |
| Security level | ~168-bit |
| Cofactor | 4 (target) |

---

## Prime Field Selection — `p = 2^336 - 17`

The prime `p = 2^336 - 17` was selected through a systematic scan of pseudo-Mersenne primes in the form `2^k - c` for `k ∈ [256, 448]`, filtering for the smallest `c` satisfying both primality and `p ≡ 3 (mod 4)`. Among all candidates in this range, `2^336 - 17` stands out with `c = 17` — one of the smallest reduction constants found — where 17 itself is prime, enabling modular reduction via a single shift-and-add operation: `17x = (x << 4) + x`, with no full multiplication required. The 336-bit field size maps cleanly onto modern architectures: `336 = 6 × 56-bit` or `6 × 64-bit limbs` with no padding waste, making multi-precision arithmetic straightforward to implement efficiently.

This prime occupies a deliberate gap in the landscape of Montgomery curves. No standardized Montgomery curve currently exists at the ~168-bit security level — sitting precisely between Curve25519 (~128-bit) and Curve448 (~224-bit). The selection process is fully reproducible: the SageMath script used to derive this prime is included in this repository, and anyone can independently verify that `2^336 - 17` is the first valid candidate at `k = 336` under the stated criteria. No parameters were chosen arbitrarily.

---

## Prime Derivation Script (SageMath)

The following script was used to derive `p = 2^336 - 17`. It scans all `k ∈ [256, 448]` and finds the smallest `c` such that `2^k - c` is prime and `≡ 3 (mod 4)`.

```python
print(f"{'k':>4} | {'c':>6} | {'bytes':>5} | {'p mod 4':>7} | p")
print("-" * 80)

for k in range(256, 449):
    for c in range(1, 200):
        p = 2^k - c
        if is_prime(p) and p % 4 == 3:
            byte_len = ceil(k / 8)
            print(f"{k:>4} | {c:>6} | {byte_len:>5} | {p % 4:>7} | 2^{k} - {c}")
            break
```

Run this script in [SageMath](https://www.sagemath.org/) to independently reproduce and verify the prime selection. The output at `k = 336` confirms `c = 17` as the smallest valid reduction constant at this field size.

---

## FPOW — Fixed-Point One-Way Wrap

FPOW is an additional hardness layer on top of scalar multiplication, designed to protect `k_raw` even if an adversary solves the ECDLP — for example via Shor's algorithm.

### Core Construction

```
secret    = SHA-512(k_raw)                    — internal, never exposed
k_wrapped = k_raw + H(secret ‖ k_raw) mod n  — transformed scalar
PublicKey = k_wrapped × G                     — only this leaves the function
```

`H` is a double SHA-512 construction binding `secret` and `k_raw` through two independent hash paths. The entire operation is atomic — no intermediate value is ever exposed.

### The Fixed-Point Equation

An attacker who recovers `k_wrapped` via ECDLP must still solve:

```
k_raw = k_wrapped − H(SHA-512(k_raw) ‖ k_raw) mod n
```

`k_raw` appears on both sides. `secret` is derived from `k_raw` itself — creating a circular dependency with no known algebraic shortcut.

### Attack Resistance — Verified via SageMath

| Attack | Method | Result |
|---|---|---|
| Fixed-point iteration | `x = k_wrapped − H(SHA-512(x) ‖ x)` × 200 | failed — no convergence |
| Meet-in-the-middle | 100k left vs 100k right table, find collision | failed — no collision found |
| Related-key delta | 10 adjacent `k_raw`, search for pattern in output | failed — no pattern detected |
| Linearity test | `wrap(a) + wrap(b) ≡ wrap(a+b)?` × 20 | failed — 0/20 linear hits |
| Known-plaintext correlation | 10 trials × 2000 samples, search for bias | failed — noise only, no genuine correlation |

Full verification: `dev/unit/fpow_curve33617.sage`

### Security Model

```
Classical adversary:
  PublicKey → k_wrapped   requires solving ECDLP        (~2^168)
  k_wrapped → k_raw       requires solving fixed-point  (~2^336)

Quantum adversary (Shor):
  PublicKey → k_wrapped   recovered
  k_wrapped → k_raw       circular, Grover: ~2^168 — still infeasible
```

FPOW does not claim to be a post-quantum primitive. It is an additional hardness layer — an attacker must break both ECDLP and the SHA-512 fixed-point inversion independently. No known combined shortcut exists.

> **Note:** FPOW is a novel construction not found in surveyed literature at time of writing. Presented as a research contribution pending formal peer review.
