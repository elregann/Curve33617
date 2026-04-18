# Curve33617

A Montgomery curve defined over the prime field `p = 2^336 - 17`.

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
