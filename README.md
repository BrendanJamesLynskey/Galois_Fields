# Galois Fields in Hardware

An educational resource covering the theory of Galois Fields (finite fields) and their fundamental role in digital hardware design. This repository contains an interactive Reveal.js presentation and supporting documentation that bridge the gap between abstract algebra and practical hardware implementations such as CRC generators, LFSRs, AES encryption, and error-correcting codes.

**Author**: Brendan Lynskey 2025

---

## Overview

Galois Fields underpin a remarkable range of hardware subsystems — from the CRC check that validates every Ethernet frame to the AES cipher that encrypts network traffic, and from the LFSR that generates pseudo-random test patterns to the Reed-Solomon codes that recover data on scratched Blu-ray discs. Despite their importance, finite field theory is often presented in a purely mathematical context that can feel disconnected from the hardware engineer's day-to-day work.

This article is aimed at hardware engineers, FPGA developers, and computer science students who want to understand *why* XOR gates keep appearing in their error-detection and cryptographic circuits, and how the elegant algebraic structure of Galois Fields makes all of it work.

The material assumes familiarity with basic digital logic (AND, XOR gates) and binary arithmetic. No prior knowledge of abstract algebra is required — the necessary concepts are developed from first principles.

---

## Background Theory

### What is a Finite Field / Galois Field?

A **field** is an algebraic structure consisting of a set of elements together with two operations (addition and multiplication) that satisfy a collection of axioms: closure, associativity, commutativity, existence of identity elements, existence of inverses, and distributivity of multiplication over addition. A **finite field** is simply a field with a finite number of elements.

The foundational result of finite field theory is that a finite field exists if and only if its number of elements is a prime power $p^n$, where $p$ is a prime and $n$ is a positive integer. Furthermore, all finite fields of a given order are isomorphic — there is essentially only one finite field of each valid size, up to relabelling of elements. These fields are denoted $GF(p^n)$ or $\mathbb{F}_{p^n}$.

For hardware applications, the prime $p = 2$ is overwhelmingly dominant, because the elements of $GF(2)$ map directly to the binary values 0 and 1, and the field operations map directly to logic gates.

### Évariste Galois — Historical Context

The fields bear the name of **Évariste Galois** (1811–1832), a French mathematician whose tragically short life produced some of the most profound insights in the history of algebra. Born in Bourg-la-Reine near Paris, Galois demonstrated extraordinary mathematical talent as a teenager, developing the foundations of what is now called Galois theory — the study of the symmetries of polynomial equations.

Galois was a politically active Republican during a turbulent period of French history. He was expelled from the École Normale, imprisoned twice, and lived a volatile existence. On 30 May 1832, at the age of twenty, Galois was mortally wounded in a duel — the exact circumstances of which remain debated by historians. The night before the duel, aware that he might not survive, he frantically wrote down his mathematical ideas in a letter to his friend Auguste Chevalier, scrawling in the margins: *"Je n'ai pas le temps"* ("I have not the time").

His manuscripts were eventually published in 1846 by Joseph Liouville, and their importance was gradually recognised. Galois's work laid the groundwork for group theory, field theory, and much of modern abstract algebra. The finite fields that now carry his name are among the most practically consequential structures in all of mathematics.

### GF(2) — The Binary Field

The simplest Galois Field is $GF(2) = \{0, 1\}$, the field with exactly two elements. Its arithmetic is defined by:

**Addition in GF(2):**

| + | 0 | 1 |
|---|---|---|
| **0** | 0 | 1 |
| **1** | 1 | 0 |

**Multiplication in GF(2):**

| × | 0 | 1 |
|---|---|---|
| **0** | 0 | 0 |
| **1** | 0 | 1 |

Addition in $GF(2)$ is the XOR operation, and multiplication in $GF(2)$ is the AND operation. This is not a coincidence or an analogy — it is an exact correspondence. Every XOR gate in a digital circuit is literally performing addition in $GF(2)$, and every AND gate is performing multiplication in $GF(2)$.

This correspondence is what makes Galois Field theory so directly applicable to hardware design. The abstract algebraic properties of the field (closure, inverses, distributivity) translate into concrete guarantees about the behaviour of hardware circuits.

### GF(2^n) — Extension Fields

While $GF(2)$ is foundational, most practical applications require larger fields. The extension fields $GF(2^n)$ contain $2^n$ elements and are constructed by adjoining a root of an irreducible polynomial of degree $n$ over $GF(2)$.

Elements of $GF(2^n)$ are represented as polynomials of degree at most $n-1$ with coefficients in $GF(2)$:

$$a_{n-1}x^{n-1} + a_{n-2}x^{n-2} + \cdots + a_1x + a_0$$

where each $a_i \in \{0, 1\}$. This means each element can be represented as an $n$-bit binary string:

| Polynomial | Binary | Decimal |
|-----------|--------|---------|
| $0$ | 000 | 0 |
| $1$ | 001 | 1 |
| $x$ | 010 | 2 |
| $x + 1$ | 011 | 3 |
| $x^2$ | 100 | 4 |
| $x^2 + 1$ | 101 | 5 |
| $x^2 + x$ | 110 | 6 |
| $x^2 + x + 1$ | 111 | 7 |

The table above shows all eight elements of $GF(2^3)$, represented as polynomials of degree at most 2 with binary coefficients.

### Arithmetic in GF(2^n)

**Addition** in $GF(2^n)$ is straightforward: add corresponding polynomial coefficients modulo 2. Since addition modulo 2 is XOR, addition in $GF(2^n)$ is simply a bitwise XOR of the binary representations.

For example, in $GF(2^3)$:

$$(x^2 + x) + (x^2 + 1) = (1 \oplus 1)x^2 + (1 \oplus 0)x + (0 \oplus 1) = x + 1$$

In binary: $110 \oplus 101 = 011$.

**Multiplication** is more involved. Elements are multiplied as polynomials, and the result is reduced modulo an irreducible polynomial $p(x)$ of degree $n$. This modular reduction is what keeps the result within the field.

For example, in $GF(2^3)$ with irreducible polynomial $p(x) = x^3 + x + 1$:

$$(x^2 + x) \times (x + 1) = x^3 + x^2 + x^2 + x = x^3 + x$$

Since $x^3 + x$ has degree $\geq 3$, we reduce modulo $p(x)$:

$$x^3 + x \equiv x^3 + x - (x^3 + x + 1) = 1 \pmod{x^3 + x + 1}$$

(Remembering that subtraction and addition are the same in $GF(2)$, since $-1 = 1$.)

So $(x^2 + x) \times (x + 1) = 1$ in $GF(2^3)$. This also tells us that $x^2 + x$ and $x + 1$ are multiplicative inverses of each other — a key property that $GF(2^3)$ guarantees for every non-zero element.

### Irreducible (Primitive) Polynomials

An **irreducible polynomial** over $GF(2)$ is a polynomial that cannot be factored into the product of two non-trivial polynomials over $GF(2)$. These play a role analogous to prime numbers in integer arithmetic. A **primitive polynomial** is an irreducible polynomial whose root generates all non-zero elements of the field — that is, the root is a primitive element (generator) of the multiplicative group.

Standard irreducible polynomials used in practice include:

| Application | Field | Polynomial |
|------------|-------|------------|
| CRC-8 | $GF(2^8)$ | $x^8 + x^2 + x + 1$ |
| CRC-16-CCITT | $GF(2^{16})$ | $x^{16} + x^{12} + x^5 + 1$ |
| CRC-32 (Ethernet) | $GF(2^{32})$ | $x^{32} + x^{26} + x^{23} + x^{22} + x^{16} + x^{12} + x^{11} + x^{10} + x^8 + x^7 + x^5 + x^4 + x^2 + x + 1$ |
| AES (Rijndael) | $GF(2^8)$ | $x^8 + x^4 + x^3 + x + 1$ |

The choice of irreducible polynomial affects the specific mapping between polynomial and binary representations, but not the algebraic structure of the field (all fields of the same order are isomorphic). In practice, polynomials are chosen for desirable properties such as low weight (few non-zero terms) for efficient hardware implementation, or maximum-length sequence generation.

### Key Properties

The following properties of $GF(2^n)$ are directly exploited by hardware implementations:

- **Closure**: The sum or product of any two field elements is always another field element. In hardware terms, an $n$-bit XOR or a multiply-and-reduce circuit always produces a valid $n$-bit result.
- **Associativity and commutativity**: Operations can be reordered and regrouped freely, enabling pipeline-friendly architectures.
- **Additive inverse**: Every element is its own additive inverse ($a + a = 0$ in $GF(2^n)$), meaning subtraction is identical to addition (XOR).
- **Multiplicative inverse**: Every non-zero element has a unique multiplicative inverse. This is critical for AES (the SubBytes step computes $GF(2^8)$ inversions) and for decoding Reed-Solomon codes.
- **Distributive law**: $a \cdot (b + c) = a \cdot b + a \cdot c$. This enables matrix formulations of encoding and decoding.

---

## Hardware Applications

### CRC — Cyclic Redundancy Check

CRC is an error-detection scheme used in virtually all digital communication protocols (Ethernet, USB, PCIe, SATA, etc.). A CRC computation is a polynomial division in $GF(2)$: the message is treated as a polynomial over $GF(2)$, divided by a fixed generator polynomial, and the remainder is appended as the check value.

The generator polynomial defines the specific CRC variant and determines its error-detection capabilities. For example, CRC-32 can detect all single-bit errors, all double-bit errors, all odd-number-of-bit errors, and any burst error of length 32 or less.

In hardware, CRC computation is typically implemented using a Linear Feedback Shift Register (LFSR) or a parallelised XOR network derived from the LFSR structure.

See: [CRC Implementation Repository](https://github.com/BrendanJamesLynskey/CRC)

### LFSR — Linear Feedback Shift Register

An LFSR is a shift register whose input is a linear function (XOR) of selected bits from the register. LFSRs implement multiplication by $x$ in $GF(2^n)$, with the feedback taps defined by the irreducible polynomial. A maximum-length LFSR (one using a primitive polynomial) cycles through all $2^n - 1$ non-zero elements of the field before repeating.

LFSRs are used for:
- **Pseudo-random sequence generation** (PRBS) for hardware testing (BERT)
- **Scrambling** in communication protocols (e.g., 64b/66b encoding in 10G Ethernet)
- **CRC computation** (serial implementation)
- **Built-in self-test** (BIST) pattern generation

See: [LFSR Implementation Repository](https://github.com/BrendanJamesLynskey/LFSR)

### AES — Advanced Encryption Standard

The AES block cipher operates on bytes as elements of $GF(2^8)$ with the irreducible polynomial $x^8 + x^4 + x^3 + x + 1$ (0x11B in hexadecimal). The SubBytes transformation — the core non-linear step of AES — computes the multiplicative inverse of each byte in $GF(2^8)$ (with 0 mapped to 0), followed by an affine transformation.

In hardware implementations, the $GF(2^8)$ inversion can be computed by:
- A lookup table (S-box) of 256 entries
- Composite field arithmetic, decomposing $GF(2^8)$ into $GF((2^4)^2)$ or $GF(((2^2)^2)^2)$ for area-efficient implementations
- Direct inversion using the extended Euclidean algorithm or exponentiation ($a^{-1} = a^{254}$ by Fermat's little theorem)

### Reed-Solomon and BCH Codes

Reed-Solomon (RS) codes are a class of error-correcting codes built entirely on $GF(2^m)$ arithmetic. They operate on symbols (multi-bit units) rather than individual bits, making them particularly effective against burst errors.

Applications include:
- **Optical storage**: CD, DVD, and Blu-ray disc error correction
- **QR codes**: error recovery from damaged or partially obscured codes
- **Deep-space communication**: NASA's Voyager missions, Mars rovers
- **Flash memory / SSDs**: ECC for NAND flash reliability
- **DVB / ATSC**: Digital television broadcast error correction

BCH (Bose–Chaudhuri–Hocquenghem) codes are a related family of cyclic error-correcting codes also based on finite field arithmetic.

### Elliptic Curve Cryptography over Binary Fields

Elliptic Curve Cryptography (ECC) can be defined over $GF(2^n)$, using curves of the form $y^2 + xy = x^3 + ax^2 + b$ where $a, b \in GF(2^n)$. Binary field ECC was historically favoured in some hardware applications because addition is carry-free (XOR), which simplifies timing and avoids side-channel leakage. The NIST curves B-233, B-283, B-409, and B-571 are defined over binary extension fields.

---

## Presentation

The interactive presentation is built with [Reveal.js](https://revealjs.com/) and can be viewed in any modern web browser.

### Viewing Locally

1. Clone this repository:
   ```bash
   git clone https://github.com/BrendanJamesLynskey/Galois_Fields.git
   cd Galois_Fields
   ```

2. Open `index.html` directly in a browser:
   ```bash
   # On Linux
   xdg-open index.html

   # On macOS
   open index.html

   # On Windows
   start index.html
   ```

   Alternatively, serve it with any local HTTP server:
   ```bash
   python3 -m http.server 8000
   # Then navigate to http://localhost:8000
   ```

3. Navigate slides with arrow keys. Press `Esc` for the slide overview, `S` for speaker notes, and `F` for fullscreen.

### Dependencies

All dependencies (Reveal.js 4.6.1, KaTeX, Google Fonts) are loaded via CDN. No local installation or build step is required.

---

## References

### Textbooks

- Lidl, R. and Niederreiter, H. *Finite Fields*. Cambridge University Press, 1997.
- McEliece, R.J. *Finite Fields for Computer Scientists and Engineers*. Kluwer Academic Publishers, 1987.
- Lin, S. and Costello, D.J. *Error Control Coding*. Pearson, 2nd edition, 2004.
- Menezes, A.J., van Oorschot, P.C., and Vanstone, S.A. *Handbook of Applied Cryptography*. CRC Press, 1996.

### Standards and Specifications

- NIST FIPS 197: *Advanced Encryption Standard (AES)*, 2001.
- IEEE 802.3: *Ethernet Standard* — defines CRC-32 polynomial and usage.
- ITU-T V.41: *CRC-16-CCITT specification*.

### Online Resources

- [Finite Field Arithmetic — Wikipedia](https://en.wikipedia.org/wiki/Finite_field_arithmetic)
- [Galois Field in Cryptography — Christof Paar's lectures](https://www.youtube.com/c/ChristofPaarCryptography)
- [AES S-Box and GF(2^8) — Sam Trenholme](https://www.samiam.org/galois.html)
- [CRC Implementation Repository](https://github.com/BrendanJamesLynskey/CRC)
- [LFSR Implementation Repository](https://github.com/BrendanJamesLynskey/LFSR)

---

## License

This project is licensed under the MIT License. See [LICENSE](LICENSE) for details.
