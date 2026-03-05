# Post-Quantum Cryptography

The cryptographic algorithms that secure the internet — RSA, elliptic curve Diffie-Hellman, ECDSA — share a common dependency: the difficulty of problems that quantum computers solve efficiently. Specifically, Shor's algorithm reduces integer factorisation and discrete logarithm to polynomial time on a sufficiently capable quantum machine. The security of everything built on those assumptions reduces with it.

This is not a new observation. Peter Shor published his algorithm in 1994. The cryptographic community has spent the intervening decades designing, analysing, and standardising replacements. NIST ran a formal competition from 2016 to 2024 and has now published three standards, with a fourth in progress.

This book covers the territory systematically: the threat model, the mathematics, the competing algorithm families, the standards that emerged, and the migration work required. It assumes familiarity with classical cryptography — if RSA and Diffie-Hellman are not already in your mental model, the appendices of any cryptography textbook will fill that gap quickly.

## What is not covered

This book does not cover quantum key distribution (QKD). QKD is a physically interesting idea that requires fibre runs between endpoints, has no mechanism for authentication without a pre-shared key, and solves a narrower problem than post-quantum cryptography at significantly higher cost. It is also not in any NIST standard. It is therefore outside the scope of this book.

This book also does not attempt to predict when cryptographically relevant quantum computers will exist. The range of credible estimates is wide enough to be unhelpful, and the conclusion — begin migration now — is the same regardless of where in that range the answer falls.

## A note on notation

Mathematical objects are typeset in standard LaTeX throughout. Vectors are bold lowercase (**a**), matrices are bold uppercase (**A**). Security parameters are written as λ. Modular arithmetic is written as operations over ℤ_q. Algorithm names use their official designations: ML-KEM rather than Kyber, ML-DSA rather than Dilithium, except when discussing the pre-standardisation designs where the original names are more recognisable.
