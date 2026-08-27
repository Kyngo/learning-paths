---
title: "Information Theory"
weight: 11
---

# Information Theory

Information theory, founded by Claude Shannon in 1948, quantifies information, uncertainty, and communication. It provides the mathematical framework for compression, error correction, cryptography, and — surprisingly — the loss functions used in machine learning.

---

## Information Content

How much "information" does an event carry? Intuitively, rare events are more informative than common ones. "The sun rose this morning" is unsurprising (low information). "An earthquake just hit" is surprising (high information).

### Self-Information

The information content of an event with probability p:

```
I(x) = -log₂(p(x))  bits
```

| Event | Probability | Information |
|-------|------------|-------------|
| Fair coin lands heads | 1/2 | 1 bit |
| Roll a 6 on a fair die | 1/6 | 2.58 bits |
| Specific card from a deck | 1/52 | 5.7 bits |
| Winning the lottery | ~10⁻⁸ | ~26.6 bits |
| Certain event | 1 | 0 bits |

**Key insight:** Information is measured in **bits** because each bit doubles the number of distinguishable messages. A 1-bit message can encode 2 possibilities; an n-bit message can encode 2ⁿ.

---

## Entropy

**Entropy** H(X) measures the average information content (average surprise) of a random variable — the average number of bits needed to encode a sample.

```
H(X) = -Σ p(x) log₂(p(x))
```

### Examples

**Fair coin (p=0.5, 0.5):**

```
H = -0.5 log₂(0.5) - 0.5 log₂(0.5) = 0.5 + 0.5 = 1 bit
```

**Biased coin (p=0.9, 0.1):**

```
H = -0.9 log₂(0.9) - 0.1 log₂(0.1) = 0.137 + 0.332 = 0.469 bits
```

The biased coin has lower entropy — it is more predictable, so outcomes carry less information on average.

### Properties of Entropy

| Property | Statement |
|----------|-----------|
| Non-negative | H(X) ≥ 0 |
| Maximum at uniform | H(X) ≤ log₂(n) for n outcomes; equality when all equally likely |
| Zero when certain | H(X) = 0 when one outcome has probability 1 |
| Additive for independent vars | H(X, Y) = H(X) + H(Y) if X, Y independent |

### Entropy in Engineering

| Application | Role of Entropy |
|-------------|----------------|
| Compression | Entropy is the theoretical minimum bits per symbol (Shannon limit) |
| Password strength | H = log₂(keyspace) — entropy measures brute-force difficulty |
| Decision trees | Splitting on the feature with highest information gain (entropy reduction) |
| Language modelling | Lower perplexity (2^H) = better model |
| Random number generation | A good RNG should produce maximum entropy output |

### Password Entropy Examples

| Password Type | Keyspace | Entropy (bits) |
|--------------|----------|---------------|
| 4-digit PIN | 10⁴ | 13.3 |
| 8 lowercase letters | 26⁸ | 37.6 |
| 8 mixed case + digits | 62⁸ | 47.6 |
| 8 printable ASCII | 95⁸ | 52.6 |
| 4 random words (Diceware) | 7776⁴ | 51.7 |
| 6 random words (Diceware) | 7776⁶ | 77.5 |

---

## Cross-Entropy

Cross-entropy measures the average number of bits needed to encode samples from distribution p using an encoding optimised for distribution q:

```
H(p, q) = -Σ p(x) log₂(q(x))
```

### Why Cross-Entropy Matters in ML

In classification, p is the true label distribution (one-hot) and q is the model's predicted probability:

```
For a single sample with true class c:
H(p, q) = -log(q(c))
```

If the model predicts 90% for the correct class: loss = -log(0.9) = 0.105
If the model predicts 10% for the correct class: loss = -log(0.1) = 2.303

Cross-entropy loss penalises confident wrong predictions heavily and rewards confident correct predictions.

### Relationship

```
H(p, q) = H(p) + D_KL(p || q)
```

Cross-entropy equals the true entropy plus the KL divergence. Since H(p) is fixed for a given dataset, minimising cross-entropy is equivalent to minimising KL divergence — making q as close to p as possible.

---

## KL Divergence

The **Kullback-Leibler divergence** measures how one probability distribution differs from another:

```
D_KL(p || q) = Σ p(x) log(p(x) / q(x))
```

### Properties

| Property | Statement |
|----------|-----------|
| Non-negative | D_KL(p \|\| q) ≥ 0 (Gibbs' inequality) |
| Zero iff equal | D_KL(p \|\| q) = 0 ⟺ p = q |
| Not symmetric | D_KL(p \|\| q) ≠ D_KL(q \|\| p) in general |
| Not a metric | Violates triangle inequality and symmetry |

### Applications

| Application | What D_KL Measures |
|-------------|-------------------|
| Training neural networks | Distance between predicted and true distributions (via cross-entropy) |
| Variational autoencoders (VAE) | Regularisation term forcing latent distribution toward prior |
| Policy gradient (RL) | Constraining policy updates (PPO, TRPO) |
| Model comparison | How different two models' predictions are |
| Feature selection | Information gain = D_KL(posterior || prior) |

---

## Mutual Information

**Mutual information** I(X; Y) measures how much knowing X reduces uncertainty about Y (and vice versa):

```
I(X; Y) = H(X) + H(Y) - H(X, Y)
         = H(X) - H(X | Y)
         = D_KL(p(x,y) || p(x)p(y))
```

| Value | Meaning |
|-------|---------|
| I(X; Y) = 0 | X and Y are independent |
| I(X; Y) = H(X) | Y completely determines X |
| I(X; Y) = H(Y) | X completely determines Y |

**Applications:** Feature selection (which features are most informative about the target), channel capacity, clustering.

---

## Compression

### Shannon's Source Coding Theorem

No lossless compression can encode messages from a source with entropy H at fewer than H bits per symbol on average. Optimal compression approaches H.

### Huffman Coding

Assigns shorter codes to more frequent symbols:

| Symbol | Frequency | Code |
|--------|-----------|------|
| A | 0.40 | 0 |
| B | 0.30 | 10 |
| C | 0.20 | 110 |
| D | 0.10 | 111 |

Average code length: 0.4(1) + 0.3(2) + 0.2(3) + 0.1(3) = 1.9 bits/symbol
Entropy: H ≈ 1.85 bits/symbol

Huffman coding is close to optimal but requires knowing the frequency distribution in advance.

### Arithmetic Coding

Encodes the entire message as a single number in [0, 1). Achieves closer to entropy than Huffman, especially for skewed distributions. Used in modern compression (JPEG, H.264, zstd).

### Lempel-Ziv (LZ77/LZ78)

Dictionary-based: replaces repeated patterns with references to earlier occurrences. Foundation of gzip, PNG, zip. Does not require prior knowledge of symbol frequencies — adapts to the data.

### The Limits of Compression

- Random data (maximum entropy) cannot be compressed
- A file that has already been compressed cannot be compressed further (it is near-maximum entropy in its encoding space)
- The pigeonhole principle proves this: 2ⁿ possible n-bit files cannot all map to fewer than 2ⁿ shorter representations

---

## Error Correction

### Shannon's Channel Coding Theorem

For any channel with capacity C, it is possible to transmit data at any rate below C with arbitrarily low error probability — but not above C.

Channel capacity is measured in bits per channel use and depends on signal-to-noise ratio.

### Hamming Distance

The number of positions where two binary strings differ:

```
hamming("1011101", "1001001") = 2
```

A code with minimum Hamming distance d can:
- Detect up to d-1 errors
- Correct up to ⌊(d-1)/2⌋ errors

### Common Error-Correcting Codes

| Code | Distance | Capability | Use |
|------|----------|-----------|-----|
| Parity bit | 2 | Detect 1 error | Simple checks |
| Hamming(7,4) | 3 | Correct 1 error | ECC memory |
| Reed-Solomon | Configurable | Burst error correction | CDs, QR codes, deep space |
| LDPC | Configurable | Near Shannon limit | 5G, WiFi 6, SSD |
| Turbo codes | Configurable | Near Shannon limit | 3G/4G cellular |

### Checksums and Hashes

| Type | Purpose | Properties |
|------|---------|-----------|
| CRC-32 | Error detection | Fast, catches burst errors, not cryptographic |
| MD5 | Fingerprinting (legacy) | Broken for security, fast for non-adversarial use |
| SHA-256 | Cryptographic integrity | Collision-resistant, preimage-resistant |
| HMAC | Authenticated integrity | Combines hash with secret key |

---

## Information Theory in ML

| Concept | ML Application |
|---------|---------------|
| Cross-entropy | Standard classification loss function |
| KL divergence | VAE regularisation, policy gradient constraints |
| Entropy | Decision tree splitting criterion, exploration in RL |
| Mutual information | Feature selection, contrastive learning (InfoNCE) |
| Bits-per-character | Language model evaluation (lower = better) |
| Perplexity | 2^(cross-entropy); measures how "confused" a language model is |
| Minimum description length | Model selection (Occam's razor formalised) |

### Perplexity

```
Perplexity = 2^H(p, q)
```

Where H(p, q) is the cross-entropy of the model q against the true distribution p.

A model with perplexity 30 is "as confused as if choosing uniformly among 30 options at each step." Lower is better.

GPT-class models achieve perplexity in the range of 10-20 on standard English text benchmarks.

---

## Key Takeaways

- Information content is -log₂(p): rare events carry more information. Entropy is the average information content.
- Entropy sets the theoretical minimum for compression. You cannot beat the Shannon limit.
- Cross-entropy is the standard loss function for classification. Minimising it is equivalent to minimising KL divergence from the true distribution.
- KL divergence measures how different two distributions are. It is not symmetric and not a metric, but it is the central quantity in variational inference and policy optimisation.
- Error-correcting codes (Hamming, Reed-Solomon, LDPC) add controlled redundancy to survive noise — the theoretical limit is the channel capacity.
- Password strength, compression ratios, model perplexity, and hash collision resistance are all measured in bits — information theory provides the unified framework.
