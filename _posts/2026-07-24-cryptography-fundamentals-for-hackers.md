---
title: "Cryptography Fundamentals for Hackers and Security Engineers"
date: 2026-07-24 10:00:00 +0530
categories: [Cybersecurity, Cryptography]
tags: [cryptography, modular-arithmetic, rsa, diffie-hellman, ctf, math-for-hackers]
description: "A deep dive into the mathematical foundations of modern cryptography — modular arithmetic, GCD, Euler's theorem, CRT, and the discrete log problem — explained with real examples and CTF applications."
image:
  path: /assets/img/social/cryptography-fundamentals.png
  alt: "Cryptography fundamentals for security engineers: modular arithmetic, RSA, CRT, and discrete logarithms"
toc: true
comments: true
---

## Introduction

If you are reading this, you probably already know that cryptography is everywhere in cybersecurity.

HTTPS, SSH, PGP, Signal, Wi-Fi encryption, TLS certificates — every single one of them is built on a few core mathematical ideas.

But here is the uncomfortable truth:

Most people who "understand" cryptography at a surface level do not actually understand it. They know that RSA uses "big prime numbers" and that Diffie-Hellman uses "modular exponentiation." But when a CTF challenge asks them to recover a message given `e = 3` and a small ciphertext, or when they need to explain why a weak `g` in Diffie-Hellman is dangerous, they freeze.

This article exists to fix that.

I am not going to throw formulas at you and call it a day. I am going to explain each concept from first principles. I will show you the math, explain why it matters, and demonstrate how attackers abuse weak implementations in the real world.

By the time you finish reading this, you should be able to:

- Explain modular arithmetic intuitively
- Compute GCDs and modular inverses by hand using the Extended Euclidean Algorithm
- Understand why Fermat's Little Theorem makes RSA decryption possible
- Recognize when the Chinese Remainder Theorem can break RSA
- Explain the Discrete Logarithm Problem and why it is the foundation of modern key exchange

Let us begin.

---

## 1. Modular Arithmetic — The Language of Cryptography

### What Is Modular Arithmetic?

Imagine a clock.

It reads 1 through 12, and after 12, it wraps back to 1.

If it is 10 o'clock and you add 4 hours, you get 2 o'clock. Not 14 o'clock.

That is modular arithmetic.

When we say:

```
a ≡ b (mod n)
```

We mean: when you divide `a` by `n`, the remainder is `b`.

Formally:

```
a = q * n + b
```

where `q` is the quotient and `b` is the remainder (always in the range `0` to `n - 1`).

Examples:

```
25 ≡ 1 (mod 12)   because 25 = 2*12 + 1
100 ≡ 4 (mod 8)   because 100 = 12*8 + 4
7  ≡ 7 (mod 13)   because 7 = 0*13 + 7
```

This is called **modular congruence**. The number `n` is called the **modulus**.

### Why Should a Hacker Care?

Modern cryptography operates entirely inside modular arithmetic. Every operation — every addition, multiplication, exponentiation — happens in a finite space defined by a modulus. There is no "outside."

When you read that RSA uses:

```
c = m^e mod n
```

The `mod n` part is not optional. It is the entire reason RSA works.

In the real numbers, computing `m^e` for large numbers is impossible — the result has more digits than there are atoms in the universe. In modular arithmetic, the result always fits within the range of the modulus. This makes the computation feasible.

But more importantly:

> Modular arithmetic is where the hard problems live.

Multiplying two numbers and taking the remainder is easy.

But given a remainder, figuring out the original number — that is hard. This asymmetry is what cryptographers exploit.

### Modular addition

```
(a + b) mod n = ((a mod n) + (b mod n)) mod n
```

Example:

```
(17 + 25) mod 12
= (5 + 1) mod 12
= 6 mod 12
= 6
```

### Modular multiplication

```
(a * b) mod n = ((a mod n) * (b mod n)) mod n
```

Example:

```
(17 * 25) mod 12
= (5 * 1) mod 12
= 5 mod 12
= 5
```

### Modular exponentiation

This is the trickiest one.

```
a^b mod n
```

You cannot compute `a^b` first and then take the modulus. The intermediate result would be astronomically large.

Instead, we use **modular exponentiation**, also called **square-and-multiply**.

To compute `5^13 mod 7`:

1. Write the exponent in binary: `13 = 1101` (which is `8 + 4 + 1`)
2. Repeatedly square:
   - `5^1 = 5 mod 7 = 5`
   - `5^2 = 25 mod 7 = 4`
   - `5^4 = 4^2 = 16 mod 7 = 2`
   - `5^8 = 2^2 = 4 mod 7 = 4`
3. Multiply the needed terms: `5^8 * 5^4 * 5^1 = 4 * 2 * 5 = 40 mod 7 = 5`

So `5^13 mod 7 = 5`.

Every modern cryptosystem depends on this property.

### The Additive and Multiplicative Groups

When working modulo `n`, the numbers `0` through `n - 1` form an algebraic structure.

Under addition, every number has an inverse. For any `a`, the number `(n - a) mod n` satisfies:

```
a + (n - a) ≡ 0 (mod n)
```

Under multiplication, not every number has an inverse. Only numbers that are **coprime** to `n` — meaning `gcd(a, n) = 1` — have a multiplicative inverse.

This distinction is critical. It is why we care about GCD and the Extended Euclidean Algorithm, which we will cover next.

### The Key Intuition

Here is what you must internalize:

- Modular arithmetic is finite — everything wraps around.
- The modulus defines your universe.
- Addition, multiplication, and exponentiation are well-defined and computable.
- Finding inverses requires special algorithms.
- Exponentiation is easy. The reverse (finding the exponent given the result) is hard.

The entire field of public-key cryptography flows from these observations.

---

## 2. The Euclidean and Extended Euclidean Algorithms

### The Greatest Common Divisor (GCD)

The GCD of two numbers is the largest integer that divides both of them without leaving a remainder.

```
gcd(12, 8) = 4
gcd(17, 5) = 1
gcd(100, 25) = 25
```

If `gcd(a, n) = 1`, we say `a` and `n` are **coprime** (or **relatively prime**).

### Why GCD Matters in Cryptography

In RSA, you generate two large primes `p` and `q`, then compute `n = p * q`.

You compute the totient `φ(n) = (p - 1) * (q - 1)`.

You choose a public exponent `e` such that:

```
gcd(e, φ(n)) = 1
```

This ensures that `e` has a multiplicative inverse modulo `φ(n)`. That inverse is your private exponent `d`.

Without `gcd(e, φ(n)) = 1`, you cannot compute `d`. Without `d`, you cannot decrypt.

### The Euclidean Algorithm

The Euclidean Algorithm finds the GCD efficiently without factoring the numbers.

The rule is:

```
gcd(a, b) = gcd(b, a mod b)
```

Repeat until the remainder is zero. The last non-zero remainder is the GCD.

Example — Find `gcd(1071, 462)`:

```
1071 = 2 * 462 + 147
 462 = 3 * 147 + 21
 147 = 7 * 21  + 0
```

```
gcd(1071, 462) = 21
```

This is extremely fast. Even for numbers thousands of digits long, the Euclidean Algorithm completes in milliseconds.

### The Extended Euclidean Algorithm

The Extended Euclidean Algorithm does more. It finds integers `x` and `y` such that:

```
a * x + b * y = gcd(a, b)
```

When `gcd(a, n) = 1`, this gives us:

```
a * x ≡ 1 (mod n)
```

The value `x` is the **modular multiplicative inverse** of `a` modulo `n`. This is the single most important computation in public-key cryptography.

Let me walk through an example.

Find the inverse of `3` modulo `7`:

We want `x` such that:

```
3 * x ≡ 1 (mod 7)
```

Step 1: Run the standard Euclidean Algorithm:

```
7 = 2 * 3 + 1
3 = 3 * 1 + 0
```

GCD is `1`. An inverse exists.

Step 2: Back-substitute to express `gcd(3, 7) = 1`:

```
1 = 7 - 2 * 3
```

Step 3: Rewrite in terms of the original numbers:

```
1 = 7 - 2 * 3
```

Take this modulo 7:

```
1 ≡ (-2) * 3 (mod 7)
```

Since `-2` is equivalent to `5` modulo 7:

```
1 ≡ 5 * 3 (mod 7)
```

The inverse is `5`.

Verification:

```
3 * 5 = 15 ≡ 1 (mod 7)
```

Correct.

### Table-Based Method (Simpler for Humans)

There is a cleaner way to perform the Extended Euclidean Algorithm using a table. It avoids the back-substitution mess.

Let us find the inverse of `3` modulo `7` using the table method:

| Step | r | q | x | y |
|------|---|---|---|---|
| 0 | 7 | - | 1 | 0 |
| 1 | 3 | - | 0 | 1 |
| 2 | 1 | 2 | 1 - (2 * 0) = 1 | 0 - (2 * 1) = -2 |
| 3 | 0 | 3 | | |

At each step, `q = r_{i-2} / r_{i-1}`.

The new `x` is `x_{i-2} - q * x_{i-1}`.
The new `y` is `y_{i-2} - q * y_{i-1}`.

When `r = 1`, the GCD is `1`. The `y` value is the modular inverse.

The inverse of `3 mod 7` is `-2 mod 7 = 5`.

### CTF Applications

Many CTF challenges ask you to compute a modular inverse. You can do this using Python:

```python
from math import gcd

def extended_gcd(a, b):
    if a == 0:
        return b, 0, 1
    g, x1, y1 = extended_gcd(b % a, a)
    x = y1 - (b // a) * x1
    y = x1
    return g, x, y

g, x, y = extended_gcd(3, 7)
print(f"Inverse: {x % 7}")  # 5
```

Python 3.8+ has a built-in:

```python
inverse = pow(3, -1, 7)
print(inverse)  # 5
```

The `pow(a, -1, n)` syntax directly computes the modular inverse. This is available in Python 3.8 and later. If you do not have this, you must implement the Extended Euclidean Algorithm yourself.

### Why This Matters for RSA

When you decrypt RSA:

```
m = c^d mod n
```

The exponent `d` is the modular inverse of `e` modulo `φ(n)`:

```
d ≡ e^(-1) (mod φ(n))
```

Without the Extended Euclidean Algorithm, you cannot compute `d`. Without `d`, you cannot decrypt. This single algorithm is literally the key to RSA.

---

## 3. Fermat's Little Theorem & Euler's Totient Theorem

### Fermat's Little Theorem (FLT)

Fermat's Little Theorem states:

If `p` is a prime number and `a` is not divisible by `p`, then:

```
a^(p - 1) ≡ 1 (mod p)
```

This is one of the most beautiful and useful results in all of cryptography.

Example: `p = 7, a = 3`

```
3^(7 - 1) = 3^6 = 729
729 mod 7 = ?
```

```
7 * 104 = 728
729 - 728 = 1
```

So `3^6 ≡ 1 (mod 7)`.

### Why FLT Works (Intuitive Explanation)

Consider the numbers `1, 2, 3, ..., p-1` modulo `p`.

Multiply each by `a`:

```
a * 1, a * 2, a * 3, ..., a * (p-1)
```

When `p` is prime and `a` is not divisible by `p`, these products are just a permutation of `1, 2, 3, ..., p-1`.

So the product of both sets must be equal:

```
1 * 2 * 3 * ... * (p-1) ≡ (a*1) * (a*2) * (a*3) * ... * (a*(p-1)) (mod p)
```

The left side is `(p-1)!`.

The right side is `a^(p-1) * (p-1)!`.

```
(p-1)! ≡ a^(p-1) * (p-1)! (mod p)
```

Since `(p-1)!` is not divisible by `p`, we can cancel it:

```
1 ≡ a^(p-1) (mod p)
```

That is the proof.

### Using FLT to Simplify Massive Exponents

This is where FLT becomes a practical tool.

Suppose you need to compute:

```
3^100 mod 7
```

Without FLT, you would need to compute a 48-digit number and then divide. With FLT:

```
3^6 ≡ 1 (mod 7)
```

So:

```
3^100 = 3^(6 * 16 + 4) = (3^6)^16 * 3^4 ≡ 1^16 * 3^4 = 81 ≡ 4 (mod 7)
```

Reducing the exponent modulo `p - 1` makes the computation trivial.

### Euler's Totient Theorem

Euler generalized Fermat's Little Theorem for composite moduli.

First, the **Euler totient function** `φ(n)` counts the numbers between `1` and `n - 1` that are coprime to `n`.

For a prime `p`:

```
φ(p) = p - 1
```

For a product of two distinct primes `p` and `q`:

```
φ(p * q) = (p - 1) * (q - 1)
```

For `n = p^k`:

```
φ(p^k) = p^k - p^(k-1) = p^(k-1) * (p - 1)
```

**Euler's Theorem** states:

If `gcd(a, n) = 1`, then:

```
a^(φ(n)) ≡ 1 (mod n)
```

Notice that Fermat's Little Theorem is just a special case of Euler's Theorem where `n` is prime.

### Why This Makes RSA Work

RSA picks two primes `p` and `q`, and computes:

```
n = p * q
φ(n) = (p - 1) * (q - 1)
```

The public exponent `e` and private exponent `d` satisfy:

```
e * d ≡ 1 (mod φ(n))
```

This means:

```
e * d = k * φ(n) + 1
```

Now, encryption is:

```
c = m^e mod n
```

Decryption is:

```
m' = c^d mod n
```

Let us trace through what happens:

```
m' = (m^e)^d = m^(e*d) = m^(k * φ(n) + 1) = m^(k * φ(n)) * m
```

By Euler's Theorem:

```
m^(k * φ(n)) = (m^(φ(n)))^k ≡ 1^k = 1 (mod n)
```

So:

```
m' ≡ 1 * m = m (mod n)
```

That is the proof. RSA decryption recovers the original message because Euler's Theorem guarantees that raising to the totient exponent returns 1.

### CTF Application: RSA with Small e

If `e = 3` and the plaintext `m` is small enough that `m^3 < n`, then:

```
c = m^3 mod n = m^3 (no wrapping!)
```

You can just take the cube root of `c` to recover `m`.

```python
from gmpy2 import iroot

c = 123456789  # example
m, exact = iroot(c, 3)
if exact:
    print(f"Plaintext: {m}")
```

This is the "cube root attack." It works because there was no modular reduction — the message was small enough that exponentiation stayed within the range of `n`.

---

## 4. The Chinese Remainder Theorem (CRT)

### What Is CRT?

The Chinese Remainder Theorem solves a system of congruences:

```
x ≡ a1 (mod n1)
x ≡ a2 (mod n2)
...
x ≡ ak (mod nk)
```

where all the moduli are pairwise coprime (no two share a common factor).

CRT states that there exists a unique solution modulo `N = n1 * n2 * ... * nk`.

In plain English:

> If you have remainders for different moduli that share no common factors, you can reconstruct the original number uniquely.

### How CRT Works: Step by Step

Let's solve:

```
x ≡ 2 (mod 3)
x ≡ 3 (mod 5)
x ≡ 2 (mod 7)
```

Step 1: Compute `N = 3 * 5 * 7 = 105`.

Step 2: For each modulus `ni`, compute `Ni = N / ni`:

```
N1 = 105 / 3 = 35
N2 = 105 / 5 = 21
N3 = 105 / 7 = 15
```

Step 3: Find the multiplicative inverse `yi` of `Ni` modulo `ni`:

```
35 * y1 ≡ 1 (mod 3) → 2 * y1 ≡ 1 (mod 3) → y1 = 2
21 * y2 ≡ 1 (mod 5) → 1 * y2 ≡ 1 (mod 5) → y2 = 1
15 * y3 ≡ 1 (mod 7) → 1 * y3 ≡ 1 (mod 7) → y3 = 1
```

Step 4: Combine:

```
x = (a1 * N1 * y1 + a2 * N2 * y2 + a3 * N3 * y3) mod N
x = (2 * 35 * 2 + 3 * 21 * 1 + 2 * 15 * 1) mod 105
x = (140 + 63 + 30) mod 105
x = 233 mod 105
x = 23
```

Verification:

```
23 mod 3 = 2 ✓
23 mod 5 = 3 ✓
23 mod 7 = 2 ✓
```

### Why CRT Matters for Hackers

#### Application 1: Hastad's Broadcast Attack

Suppose someone sends the same message `m` encrypted to three different recipients, each using RSA with `e = 3` but different moduli.

```
c1 = m^3 mod n1
c2 = m^3 mod n2
c3 = m^3 mod n3
```

If `n1, n2, n3` are pairwise coprime (which they usually are), you can use CRT to recover `m^3`.

Since `m^3 < n1 * n2 * n3` (assuming `m` is smaller than each modulus), the modular reduction never happens. You can just take the cube root.

This is one of the most famous RSA attacks in CTFs:

```python
from sympy.ntheory.modular import crt

moduli = [n1, n2, n3]
remainders = [c1, c2, c3]
m_cubed, _ = crt(moduli, remainders)
m = int(round(m_cubed ** (1/3)))
```

Use `gmpy2.iroot` for the cube root instead of floating-point math to avoid precision errors.

#### Application 2: Speeding Up RSA Decryption

RSA decryption normally requires:

```
m = c^d mod n
```

The exponent `d` is huge (roughly the size of `n`). The modulus `n` is about 2048 bits.

If instead you decrypt modulo `p` and modulo `q` separately, the exponents are half the size and the computation is roughly 4 times faster.

This is how real RSA implementations work internally. They use CRT to speed up decryption without changing the result.

The steps:

```
m1 = c^(d mod (p-1)) mod p
m2 = c^(d mod (q-1)) mod q
m = CRT(m1, m2, p, q)
```

#### Application 3: Fault Attacks

If a hardware fault corrupts one of the CRT components during decryption (say `m1` is computed correctly but `m2` is garbled), an attacker can factor `n` from the faulty result.

Given:

```
correct = m mod p   (but not mod q, because m2 was wrong)
faulty  = m' (general)
```

```
gcd(n, correct - faulty) = p or q
```

This is the Bellcore attack. It is a practical attack against smart cards and HSMs.

### The Intuition

Think of CRT as the inverse of modular arithmetic.

Modular arithmetic takes a number and gives you remainders modulo different bases.

CRT takes those remainders and gives you the original number back. No information is lost as long as the number is smaller than the product of the moduli.

This property makes CRT invaluable for both cryptanalysis (attacks) and optimization (implementations).

---

## 5. The Discrete Logarithm Problem (DLP)

### What Is the Discrete Logarithm Problem?

Consider modular exponentiation:

```
result = base^exponent mod modulus
```

Given `base` and `exponent`, computing the result is easy. We already covered this — use square-and-multiply.

The Discrete Logarithm Problem is the reverse:

Given `base`, `result`, and `modulus`, find `exponent`:

```
base^exponent ≡ result (mod modulus)
```

That `exponent` is called the **discrete logarithm**.

In the real numbers, logarithms are easy (use a calculator). In modular arithmetic, they are computationally hard.

This asymmetry — exponentiation is easy, inverse is hard — is the foundation of:

- Diffie-Hellman key exchange
- Digital Signature Algorithm (DSA)
- ElGamal encryption
- Elliptic Curve Cryptography

### Why Is It Hard?

To find a discrete logarithm, the naive approach is to try every possible exponent:

```python
def brute_force_dlp(base, result, modulus):
    val = 1
    for exp in range(modulus):
        if val == result:
            return exp
        val = (val * base) % modulus
    return None
```

For a 2048-bit modulus, the loop would run up to `2^2048` iterations. That is more operations than there are particles in the observable universe.

Even the best known algorithms (Index Calculus, Number Field Sieve) are subexponential. For elliptic curves, the best algorithms are fully exponential.

### A Concrete Example

Take a small prime: `p = 23`, generator `g = 5`.

```
5^x mod 23 = ?
```

Let's compute:

```
5^1 = 5 mod 23
5^2 = 25 mod 23 = 2
5^3 = 2 * 5 = 10 mod 23
5^4 = 10 * 5 = 50 mod 23 = 4
...
```

Now, given `5^x ≡ 8 mod 23`, can you find `x`?

You could enumerate all 23 possibilities. But when `p` is a 2048-bit prime, enumeration is impossible.

### The Diffie-Hellman Key Exchange

This is the most famous use of DLP.

Alice and Bob want to share a secret over an insecure channel.

Public values:

- A large prime `p`
- A generator `g` (a primitive root modulo `p`)

Alice picks a random secret `a`, computes `A = g^a mod p`, sends `A`.

Bob picks a random secret `b`, computes `B = g^b mod p`, sends `B`.

Alice computes `s = B^a mod p = (g^b)^a = g^(ab) mod p`.

Bob computes `s = A^b mod p = (g^a)^b = g^(ab) mod p`.

Both now share `g^(ab) mod p`. No one else can compute it without finding `a` or `b`.

The security of Diffie-Hellman rests entirely on the assumption that DLP is hard.

### The Small Subgroup Attack

This is a practical attack you need to know.

If the modulus `p` is chosen such that `p - 1` has small factors, an attacker can restrict the possible values of the shared secret by sending a generator from a small subgroup.

Example: If `g` has order `q` where `q` divides `p - 1`, then the possible values of `g^x` are limited to `q` possibilities.

The attacker can force the exponent to leak information modulo the small factor. With enough such attacks, the full exponent can be recovered using CRT.

This is why safe primes `p = 2q + 1` (where `q` is also prime) are used in practice. The only subgroups are of orders `1, 2, q, 2q`. The large prime subgroup of order `q` prevents the attack.

### DLP in Elliptic Curves

Elliptic curve cryptography (ECC) uses a different algebraic structure, but the same principle applies:

- Point addition on an elliptic curve is analogous to multiplication in the multiplicative group.
- Scalar multiplication (`k * G`) is analogous to exponentiation.
- The discrete logarithm (finding `k` given `k*G` and `G`) is even harder than in the integer case.

This is why ECC can use smaller key sizes (256-bit ECC = 3072-bit RSA equivalent security).

### DLP-Based Attacks in CTFs

The most common DLP-related attacks you will encounter:

1. **Small prime** — If `p` is small, just brute force.

2. **Smooth order** — If `p - 1` has only small factors, use the Pohlig-Hellman algorithm with CRT.

3. **Pohlig-Hellman Algorithm**: If the order of the group has factorization `q1^e1 * q2^e2 * ... * qk^ek`, DLP can be solved in each subgroup independently, then combined using CRT. The running time depends on the largest prime factor, not the group size.

   ```python
   from sympy.ntheory import factorint
   
   # If n = p - 1 has small factors only
   # Pohlig-Hellman will solve DLP quickly
   # Use sage: discrete_log(h, Mod(g, p))
   ```

4. **Reused nonces** in DSA — If the same `k` is used twice, you can recover the private key.

5. **Weak generators** — If `g` generates a small subgroup, exhaustive search becomes feasible.

### The Big Picture

Here is what you must understand about DLP:

| Problem | Easy Direction | Hard Direction |
|---------|---------------|----------------|
| Integer multiplication | Multiply | Factor |
| Modular exponentiation | Compute `g^x mod p` | Find `x` given `g^x` |
| Elliptic curve scalar multiplication | Compute `k*G` | Find `k` given `k*G` |

The entire field of public-key cryptography exploits this asymmetry.

Encryption is the easy direction. Decryption (for the authorized party) uses a trapdoor — additional information that makes the hard direction easy.

The attacker only has the public information, so they face the hard direction.

---

## Putting It All Together

Let me show you how these concepts connect in a real cryptosystem.

### RSA End to End

1. Choose two large primes `p` and `q`.
2. Compute `n = p * q`.
3. Compute `φ(n) = (p - 1) * (q - 1)` — Euler's totient.
4. Choose `e` such that `gcd(e, φ(n)) = 1` — uses Euclidean Algorithm.
5. Compute `d = e^(-1) mod φ(n)` — uses Extended Euclidean Algorithm.
6. Encrypt: `c = m^e mod n` — modular exponentiation.
7. Decrypt: `m = c^d mod n` — works because of Euler's Theorem.
8. Speed up decryption using CRT: compute `m_p = c^d mod p`, `m_q = c^d mod q`, combine via CRT.

Every concept we covered is used in this single cryptosystem.

### Diffie-Hellman End to End

1. Choose a large prime `p` and generator `g`.
2. Alice picks `a`, sends `A = g^a mod p`.
3. Bob picks `b`, sends `B = g^b mod p`.
4. Shared secret: `s = g^(ab) mod p`.
5. Security relies on DLP — attacker cannot find `a` from `g^a mod p`.
6. Use safe prime `p = 2q + 1` to prevent small subgroup attacks.
7. Use large subgroup of order `q` to prevent Pohlig-Hellman.

---

## Summary for the Practicing Hacker

You do not need to be a mathematician to be good at cryptography. But you do need to understand these five concepts at a deep, intuitive level.

Here is what you should be able to do after reading this article:

| Concept | What to Know | Attack to Recognize |
|---------|-------------|-------------------|
| Modular arithmetic | Numbers wrap around. Exponentiation is easy, inverse is hard. | Recognize when no wrap happened (cube root attack). |
| Extended Euclidean | Compute GCD. Find modular inverses. | RSA insecure when `p` and `q` are too close (Fermat factorization). |
| FLT / Euler's Theorem | `a^(φ(n)) ≡ 1 (mod n)`. This is why RSA decryption works. | Small `e` attacks, understanding what `d` actually does. |
| CRT | Reconstruct numbers from remainders. | Hastad's broadcast, fault attacks, multi-prime RSA. |
| DLP | Exponentiation is easy, finding the exponent is hard. | Small subgroup attacks, Pohlig-Hellman, weak `g`. |

Each of these concepts is a tool in your toolbelt. The more comfortable you are with them, the more attacks you will be able to recognize and exploit.

---

## Where to Go Next

If you want to practice these concepts:

1. **Implement RSA from scratch** — Do not use a library. Write `keygen()`, `encrypt()`, `decrypt()` yourself in Python. This will force you to use every concept in this article.

2. **Solve CTF challenges** — Cryptography challenges on platforms like HackTheBox, CTFtime, and PicoCTF will test your understanding of these concepts under time pressure.

3. **Read the source code of real implementations** — Look at how OpenSSL or Python's `cryptography` library implements RSA and Diffie-Hellman. Pay attention to how they handle CRT and safe primes.

4. **Break things** — Take a correct implementation and introduce one weakness at a time (small `e`, shared primes, weak `g`, nonce reuse). See how far you can go.

Cryptography is not magic. It is math applied carefully, with assumptions that must be verified. Once you understand the assumptions, you can learn to break them.

And that is where the real power lies.

For an applied example of signatures, claims, and verification failures in web authentication, continue with the [JWT Security Deep Dive]({% post_url 2026-07-21-jwt-security-deep-dive %}).
