---
title: "Notes for the Paper \"DECO: Liberating Web Data Using Decentralized Oracles for TLS\""
date: 2023-12-03T01:47:22+0800
lastmod: 2023-12-14T07:34:59+08:00
categories: [computer, paper-reading]
tags: [blockchain, zero-knowledge]
series: arxiv.1909.00938
---

DECO is a privacy-preserving, decentralized oracle scheme based on TLS.
It allows users to prove that a piece of data accessed via TLS came from a particular website,
and optional keeping the data itself secret.
And it requires no trusted hardware or server-side modifications. \
\
In this article, I will read the paper from an engineer's perspective, focus on its implementation;
theory details, such as proofs, will be ignored.
<!--more-->

<!-- TODO & FIXME
    - FIXME(MathJax): underline after non-alphanumeric; ref: jeffreytse/jekyll-spaceship#94
    - FIXME(PlantUML): rendering math formula failed
  -->

## Preface

Before I begin, let me make it clear:

> DECO is currently in the Proof of Concept (PoC) stage with a couple of projects.

When you join [Chainlink official discord server][Chainlink Discord] and ask the state of DECO,
a bootcamp member will answer you the above sentences. (until Dec 3, 2023)

So I didn't find any specific implementation.

The following discussion is just my own understanding after reading relevant resources.

## Background

<div id="customized-mta-protocol" />

### TLS Protocol

<div id="customized-hmac-and-prf" />

#### HMAC and The Pseudorandom Function (PRF) in TLS protocol [^101]

TLS's PRF takes as input a secret, a seed, and an identifying label and
produces an output of arbitrary length.

- Define a data expansion function:

  ```
  P_hash(secret, seed) = HMAC_hash(secret, A(1) + seed) +
                         HMAC_hash(secret, A(2) + seed) +
                         HMAC_hash(secret, A(3) + seed) + ...
  ```

  where `+` indicates concatenation.

  And `A()` is defined as:
  ```
  A(0) = seed
  A(i) = HMAC_hash(secret, A(i-1))
  ```

- TLS's PRF is created by applying `P_hash` to the secret as:

  ```
  PRF(secret, label, seed) = P_hash(secret, label + seed)
  ```

<div id="customized-key-calculation" />

#### Key Calculation in TLS protocol [^102]

In TLS protocol, when keys and MAC keys are generated, the master secret is
used as an entropy source to generate the key material, compute

```
key_block = PRF(SecurityParameters.master_secret,
                "key expansion",
                SecurityParameters.server_random,
                SecurityParameters.client_random)
```

until enough output has been generated.

Then, the `key_block` is partitioned as keys and initial vectors for both client and server.

<div id="customized-tls-handshake" />

#### Handshake in TLS protocol [^103]

```plantuml
!theme sketchy-outline
scale 560 height
skinparam ParticipantPadding 80
skinparam SequenceMessageAlign direction

title Message flow for a full handshake

Client    -> Server: ClientHello
Client <-  Server: ServerHello
Client <--- Server: Certificate*
Client <--- Server: ServerKeyExchange*
Client <--- Server: CertificateRequest*
Client <-   Server: ServerHelloDone
Client  ---> Server: Certificate*
Client    -> Server: ClientKeyExchange
Client  ---> Server: CertificateVerify*
Client    -> Server: [ChangeCipherSpec]
Client    -> Server: Finished
Client <-  Server: [ChangeCipherSpec]
Client <-  Server: Finished
Client <---> Server: Application Data

footer
Char '*' means optional.
end footer
```

- `ClientHello` contains 32 bits UNIX timestamp and 28 bytes random bytes.
  Let's denote the whole random data as $r_c$.
- `ServerHello` contains
  - 32 bits UNIX timestamp and 28 bytes random bytes.
    Let's denote the whole random data as $r_s$.
  - a signature (if requred).
    Let's denote it as $sig$.
- `Certificate` contains the server’s certificate chain, denoted as $cert$.
- `ServerKeyExchange` contains the server's public key, denoted as $Y_S$.
- (_In this article, other messages won't be discussed._)

**The notations above will be used later.**

### :construction: [WIP] Other Protocols

#### Multiplicative to Additive (MtA) share conversion protocol [^104]

Let $E_{pk}(x)$ denote the encryption algorithm using public key $pk$.

- Given ciphertexts $c_{1} = E_{pk}(a)$ and $c_{2} = E_{pk}(b)$.

  Computable function $+\_{E}$ such that: $c_{1} +\_{E} c_{2} = E_{pk}(a + b \mod{N})$.

- Given an integer $a \in N$ and a ciphertext $c = E_{pk}(m)$, then

  Scalar multiplication operation $\times_{E}$ such that:
  $a \times_{E} c_{i} = E_{pk}(am \mod{N})$.

The MtA protocol as follows:

- At the beginning,
  - Alice holds the secret $a \in Z_{q}$.
  - Bob holds the secret $b \in Z_{q}$.
  - $a$ and $b$ are multiplicative shares of a secret $s = ab \mod{q}$.
- Alice initiates the protocol by
  - sending $c_{A} = E_{A}(a)$ to Bob.
- Bob
  - chooses a random $\beta' \in Z_{q^{5}}$.
  - computes the ciphertext $c_{B} = b \times_{E} c_{A} +\_{E} E_{A}(\beta')$.
  - sets his share to $\beta = -\beta' \mod q$.
  - sending $c_{B}$ to Alice.
- Alice
  - decrypts $c_{B}$ to obtain $\alpha'$.
  - sets $\alpha = \alpha' \mod q$.
- As a result,
  - Alice holds $a$ and $\alpha$.
  - Bob holds $b$ and $\beta$.
  - $\alpha + \beta = s = ab \mod{q}$

---

## The DECO Protocol

### Overview

In the original paper, the authors build the full protocol from a strawman protocol, increase features step by step.
This article will skip those steps and focus on the final full protocol.

There are different approaches for each phase in the original paper.
I won't talk about all of them.
For exposition, I will only talk about all 1st choices in the original paper.

_p.s. In the future, I may discuss other parts when I have time._

The following list shows which will mentioned in this article.
- [Three-party handshake](#three-party-handshake)
  - TLS protocol
    - AES cipher
      - [x] CBC-HMAC mode
      - [ ] GCM mode (Sec. 4.1.2 in [DECO])
    - TLS version
      - [x] TLS 1.2
      - [ ] TLS 1.3 (Sec. 4.1.2 in [DECO])
  - [Key exchange](#key-exchange)
  - [Key derivation](#key-derivation)
    - [Convert shares][ECtF]
  - [Full three-party handshake protocol][3P-HS]
- Query execution
  - [x] CBC-HMAC (:construction: Planned, but incomplete.)
  - [ ] AES-GCM (Sec. 4.2.2 in [DECO])
- Proof generation
  - [x] Selective opening
    - [x] CBC-HMAC (:construction: Planned, but incomplete.)
    - [ ] GCM (Sec. 5.1.2 in [DECO])
  - [ ] Context integrity [^201] (Sec. 5.2 in [DECO])

### Three-party handshake

#### Key exchange

The steps of a standard TLS key exchange (TLS 1.2, CBC-HMAC mode) are as follows:

```plantuml
!theme sketchy-outline
title Standard TLS Key Exchange

Client <-> Server: Step 1. Agree on

  note over Client: Step 2. Choose a random secret
/ note over Server: Step 2. Choose a random secret

Client <-> Server: Step 3. Exchange public keys

  note over Client: Step 4. Compute the shared secret
/ note over Server: Step 4. Compute the shared secret
```

- Step 1: Client and Server agree on a generator $G$ and a modulus $p$.
- Step 2: Choose random secrets.
  - Client samples a random secret $a$.
  - Server samples a random secret $b$.
- Step 3: Exchange keys.
  - Client sends its public key $A \equiv G^a \mod{p}$ to Server.
  - Server sends its public key $B \equiv G^b \mod{p}$ to Client.
- Step 4: Compute the shared secret $s$.
  - Client computes $S \equiv B^a \equiv g^{b^a} \equiv g^{ab} \mod{p}$.
  - Server computes $S \equiv B^b \equiv g^{a^b} \equiv g^{ab} \mod{p}$.
- Finally,
  - Client knows $a$ and $s$, but doesn't know $b$.
  - Server knows $b$ and $s$, but doesn't know $a$.

The main idea of DECO is **split session key between Prover and Verifier.**

<div id="customized-3p-key-exchange" />

```plantuml
!theme sketchy-outline
title DECO Three-Party Key Exchange

participant Verifier
participant Prover
participant Server

Prover  <--> Server : Step 1. Start a standard TLS key exchange
Verifier <-  Prover : Step 2. Sync server's public key
  note over Verifier: Step 3. Choose a random secret
Verifier  -> Prover : Step 4. Send public key
  note over Prover  : Step 5. Choose a random secret
  note over Prover  : Step 6. Compute a new shared public key
Prover  <--> Server : Step 7. Continue the standard TLS key exchange
```

- Step 1: Prover starts a [standard TLS key exchange until message `ServerHelloDone`][TLS-Handshake]. \
  Then, Prover has $Y_S$.
- Step 2: Prover sends the server's publick key $Y_S$ to Verifier.
- Step 3: Verifier samples a random secret $s_V$.
- Step 4: Verifier sends its public key $Y_V = s_V \cdot G$ to Prover.
- Step 5: Prover samples a random secret $s_P$.
- Step 6: Prover
  - computes a public key $Y_{P'} = s_P \cdot G$,
  - computes the combined public key $Y_P = Y_{P'} + Y_V$.
- Step 7: Prover uses $Y_P$ as its own public, then continues the TLS key exchange with Server.
- Finally,
  - Server knows its own secret $s_S$, and the secret $Z = s_S \cdot Y_P$.
  - Prover knows its own secret $s_P$, and its share of $Z$ as $Z_P = s_P \cdot Y_S$.
  - Verifier knows its own secret $s_V$, and its share of $Z$ as $Z_V = s_V \cdot Y_S$.
  - Neither Prover nor Verifier knows the $Z$. \
    Note that $Z = Z_V + Z_P$ where $+$ is the group operation of $EC(\mathbb{F}\_p)$.

#### Key derivation

Now that Prover and Verifier have established additive shares of $Z$, they proceed to
derive session keys by evaluating the [TLS-PRF] keyed with the $x$ corrdinate of $Z$.

<div id="customized-ectf" />

##### Converting shares in $EC(\mathbb{F}\_p)$ to shares in $\mathbb{F}\_p$ (ECtF)

DECO uses ECtF protocol, for converting shares of EC points in $EC(\mathbb{F})$
to shares of coordinates in $\mathbb{F}$.

- Step 1:
  - Prover   samples $P_1 = (x_1, y_1) \in EC(\mathbb{F}\_p)$,
  - Verifier samples $P_2 = (x_2, y_2) \in EC(\mathbb{F}\_p)$.
- Step 2:
  - Prover   samples $\rho_1 \gets{$} \mathbb{Z}\_p$,
  - Verifier samples $\rho_2 \gets{$} \mathbb{Z}\_p$,
  - Then compute $\alpha_1,\alpha_2 := \mathrm{MtA}((-x_1,\rho_1),(\rho_2,x_2))$.
- Step 3:
  - Prover   computes $\delta_1 = -x_1 \rho_1 + \alpha_1$,
  - Verifier computes $\delta_2 =  x_2 \rho_2 + \alpha_2$.
- Step 4:
  - Prover   sends $\sigma_1$ to Verifier,
  - Verifier sends $\sigma_2$ to Prover,
  - Both of Prover and Verifier compute $\delta = \delta_1 + \delta_2$.
- Step 5:
  - Prover   computes $\eta_1 = \rho_1 \cdot \delta^{-1}$,
  - Verifier computes $\eta_2 = \rho_2 \cdot \delta^{-1}$,
  - Then compute $\beta_1,\beta_2 := \mathrm{MtA}((-y_1,\eta_1),(\eta_2,y_2))$.
- Step 6:
  - Prover   computes $\lambda_1 = -y_1 \eta_1 + \beta_1$,
  - Verifier computes $\lambda_2 =  y_2 \eta_2 + \beta_2$,
  - Then compute $\gamma_1,\gamma_2 := \mathrm{MtA}(\lambda_1, \lambda_2)$.
- Step 7:
  - Prover   computes $s_1 = 2\gamma_1 + \lambda_1^2 - x_1$,
  - Verifier computes $s_2 = 2\gamma_2 + \lambda_2^2 - x_2$,
- Finally, Prover and Verifier output $s_1$ and $s_2$ such that $s_1 + s_2 = x$
  where $(x,y) = P_1 + P_2$ in $EC$.

<div id="customized-3p-handshake" />

<div id="todo" />

#### :construction: [WIP] The full three-party handshake (3P-HS) protocol

Let's combine all steps above:
- Step 1: Prover and Verifier start [three-party key exchange][3P-KeyExchange] and stop after its step 2.
- Step 2: Both Prover and Verifier check the $cert$ and $sig$.
- Step 3: Prover and Verifier continue [three-party key exchange][3P-KeyExchange].
- Step 4: Prover and Verifier use [ECtF protocol][ECtF] to compute a sharing
  of the x-coordinate of $Z$, denote the results as $z_P$ and $z_V$.

:construction: Writing ...

### :construction: [TODO] Query execution

After the handshake, Prover sends its query to Server as a standard TLS client,
but it requires help from Verifier.

### :construction: [TODO] Proof generation

#### :construction: [TODO] Selective opening

### References

- [DECO: Liberating Web Data Using Decentralized Oracles for TLS][DECO]

## Appendix

#### Errors in Original Paper

_p.s. Possible errors, just my own opinion._

- In A.1 Figure 6, it said 256 random bits in `ClientHello`.

  [I think it's incorrect.][TLS-Handshake]

- In A.1 Figure 6, the last $z_v$ should be $z_V$ (uppercase).

#### Additional Resources

- Chainlink Blogs
  - [DECO Series](https://blog.chain.link/tag/deco-series/)
  - [ZK Series](https://blog.chain.link/tag/zk-series/)
- Chainlink Videos
  - [SmartCon 2022: Privacy-Preserving Oracles With DECO (Dahlia Malkhi and Osama Khan)](https://www.youtube.com/watch?v=eJqZQ2_VBzo)
  - [SmartCon 2023: An Update on DECO (Alexandru Topliceanu)](https://www.youtube.com/watch?v=j4PpAloF75Y)
  - :+1: [Ari Juels DECO Presentation: Liberating Web Data Using Decentralized Oracles for TLS](https://www.youtube.com/watch?v=zWTx1iQOCDM)
- [Chainlink DECO Playground](http://chn.lk/deco-playground)
- Related Papers
  - [PECO: methods to enhance the privacy of DECO protocol](https://eprint.iacr.org/2022/1774)

<!-- anchors -->
[MtA]: #customized-mta-protocol
[TLS-PRF]: #customized-hmac-and-prf
[TLS-KeyCal]: #customized-key-calculation
[TLS-Handshake]: #customized-tls-handshake
[3P-KeyExchange]: #customized-3p-key-exchange
[3P-HS]: #customized-3p-handshake
[ECtF]: #customized-ectf

<!-- footnotes -->
[^101]: Section 5 in [RFC 5246: The Transport Layer Security (TLS) Protocol Version 1.2][Ref.36].
[^102]: Section 6.3 in [RFC 5246: The Transport Layer Security (TLS) Protocol Version 1.2][Ref.36].
[^103]: Section 7.3 in [RFC 5246: The Transport Layer Security (TLS) Protocol Version 1.2][Ref.36].
[^104]: Section 3 in [Fast multiparty threshold ECDSA with fast trustless setup][Ref.42].
[^201]: TODO: It's very important feature, and I will discuss in another post.

<!-- external -->
[Chainlink Discord]: https://discordapp.com/invite/aSK4zew
[DECO]: https://arxiv.org/abs/1909.00938
[Ref.36]: https://www.rfc-editor.org/info/rfc5246
[Ref.42]: https://eprint.iacr.org/2019/114