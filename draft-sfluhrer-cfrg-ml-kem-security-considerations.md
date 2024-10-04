---
title: ml-kem-security-considerations
abbrev: "ml-kem security"
category: info

docname: draft-sfluhrer-cfrg-ml-kem-security-considerations-latest
submissiontype: IRTF
number:
date:
consensus: true
v: 3
area: "IRTF"
workgroup: "Crypto Forum"

author:
    fullname: Scott Fluhrer
    organization: Cisco Systems
    email: sfluhrer@cisco.com

author:
    fullname: Quynh Dang
    organization: National Institute of Standards and Technology
    email: Quynh.Dang@nist.gov

informative:


--- abstract

NIST standardized ML-KEM as FIPS 203 in August 2024.  This document discusses how to use ML-KEM - that is, what problem it solves, and how to use it securely.

--- middle

# Introduction

A large reliable Quantum Computer (often termed a Cryptographically Relevant Quantum Computer or CRQC) would be able to break protocols which rely on the tradtional RSA, DH or ECDH methods of securely exchanging keys.  Even though we do not believe, at the time of this writing, there exists a CRQC, there still remains the possibility that a adversary may record the protocol exchange, and then later (when they have access to a CRQC) go ahead and read the traffic.

Because of this potential threat, NIST has standardized ML-KEM (Module-Lattice-Based Key-Encapsulation Mechanism), which is standardized in FIPS 203.  ML-KEM is used to generate a shared secret key between two parties. One party (Alice) generates a public/private keypair, and sends the public key to the other side (Bob).  Bob uses the public key and some randomness to generate both the shared secret key and a ciphertext.  Bob then sends the ciphertext to Alice, who uses her private key to generate the same shared secret key.

The fundamental security propery is that someone listening to the exchanges (and thus obtains both the public key and the ciphertext) cannot reconstruct the shared secret key and this is true even if the adversary has access to a CRQC. ML-KEM is IND-CCA2 secure, that is, it remains secure even if an adversary is able to submit arbitrary ciphertexts and observe the resulting shared key. Submitting invalid ciphertexts to a ML-KEM.Decap does not help the attacker obtain information about the decryption key of the PKE-Decrypt function inside the ML-KEM.Decap. Subtituting the public key Alice sends Bob by another public key chosen by the attacker will not help the attacker get any information about Alice's private key, it would just make Alice and Bob not have a same shared secret key.

ML-KEM is what is termed a Key Encapsulation Mechanism. One common misunderstanding of that term is the expectation that Bob freely chooses the shared secret, and encrypts that when sending to Alice. What happens instead is that randomness from both sides are used to contribute to the shared secret.  That is, ML-KEM internally generates the shared secret in a way that Bob cannot select the value. Now, Bob can generate a number of ciphertext/shared secret pairs, and select the shared secret that he prefers, but he cannot freely choose it.

A KEM (such as ML-KEM) sounds like it may be a drop-in replacement for Diffie-Hellman, however there is one scenario where this doesn't work. If the protocol uses DH in a 'static-static' configuration, that is, if both sides have long-term public keys, then ML-KEM is not suitable. That is because the ciphertext is necessarily a function of Alice's public key, and thus can only be useful only with that specific public key.

# Using ML-KEM

To use ML-KEM, there are three steps involved

## ML-KEM Key Generation

The first step for Alice is to generate a public and private keypair.

In FIPS 203, this function is termed ML-KEM.KeyGen() (see section 7.1).  It internally calls the random number generator for a seed and produces both a public key (termed an encapsulation key in FIPS 203) and a private key (termed a decapsulation key).

The public key can be freely published (and Bob will need it for his part of the process); this step may be performed simply by transmitting the key to Bob.  However, the private key must be kept secret.

## ML-KEM Encapsulation

The second step is for Bob to generate a ciphertext and a shared secret key.

To perform this step, Bob would first run the Encapsulation Key Check on Alice's public key as outlined at the beginning of section 7.2.
If that test passes, then Bob would perform the what FIPS 203 terms as ML-KEM.Encaps() (see section 7.2).  This step takes the validated public key, internally calls the random number generator for a seed, and produces both a ciphertext and a 32 byte shared secret key.

The ciphertext can be transmitted back to Alice; if the exchange is successful, the 32 byte shared secret key will be the key shared with Alice.

It may be that some libraries combine the validation and the encapsulation step; you should check whether the library you are using does.

## ML-KEM Decapsulation

The third and final step is for Alice to take the ciphertext and generate the shared secret key.

To perform this step, Alice would first run the Decapsulation Key Check on Bob's ciphertext as outlined at the beginning of section 7.3.
If that test passes, then Bob would perform the what FIPS 203 terms as ML-KEM.Decaps() (see section 7.3).  This step takes the ciphertext from Bob and the private key that was previously generated by Alice, and produces a 32 byte shared secret key.

If the exchange is successful, the 32 byte key generated on both sides will be the same.

It may be that some libraries combine the validation and the decapsulation step; you should check whether the library you are using does.

## ML-KEM Parameter sets

ML-KEM comes with three parameter sets; ML-KEM-512, ML-KEM-768 and ML-KEM-1024.  It is assumed that Alice and Bob both know which parameter sets they use (either by negotiation or by having one selection fixed in the protocol).

Here is a summary of how those parameter sets differ:

|             | pk size  | sk size | ct size  | ss size  | as strong as |
| :---------- | -------: | ------: | -------: | :------: | :----------: |
| ML-KEM-512  |      800 |    1632 |      768 |       32 |      AES-128 |
| ML-KEM-768  |     1184 |    2400 |     1088 |       32 |      AES-192 |
| ML-KEM-1024 |     1568 |    3168 |     1568 |       32 |      AES-256 |

(pk = public key, sk = private key, ct = ciphertext, ss = shared key, all lengths in bytes)

# Conventions and Definitions

{::boilerplate bcp14-tagged}

I don't know if we need anything in this section.

# KEM Security Considerations

This section pertains to KEM (Key Encapsulation Mechanisms) in general, including ML-KEM

To use a KEM, you need to use a high-quality source of entropy during both the key-pair generation and ciphertext generation steps.  If an adversary can recover the random bits used in either of these processes, they can recover the shared secret.  If an adversary can recover the random bits used during key generation, they can recover the secret key.

Alice needs to keep her private key secret. It is recommended that she zeroize her private key when she will have no further need of it.

A KEM provides no authentication; it is important that the protocol that uses a KEM lets Bob be able to verify that the public key he obtains comes from Alice and that the ciphertext that Alice receives came from Bob (that is, an entity that Alice is willing to communicate with).

# ML-KEM Security Considerations

To use a ML-KEM, you need to use a 32-byte random byte-string which has a security strength equal to greater than the security strength of the KEM during both key generation and encapsulation steps.  If an adversary can recover the 32-byte random byte-string used in either of these processes, he can recover the shared secret key.

Alice must keep her private key secret.  It is recommended that she zeroizes her private key when she will have no further need of it.

The use of ML-KEM described above in this document does not provide authentication of either communitcating party. A protocol that requires authentication must use other cryptographic methods to achieve it such as using digital signatures or a MAC to verify integrity of the protocol exchange transcript.

If the ciphertext that Alice receives from Bob is tampered with (either by small modification or by replacing it with an entirely different ciphertext), the shared secret key that Alice derives will be uncorrelated with the shared secret key that Bob obtains.  An attacker will not be able to determine any information about the correct shared secret key or Alice's private key, even if the attacker obtains Alice's modified shared secret key which is the output of the ML-KEM.Decap function taking the modified ciphertext as input.

It is secure to reuse a public key multiple times.  That is, instead of Alice generating a fresh public and private keypair for each exchange, Alice may generate a public key once, and then publish that public key, and use it for multiple incoming ciphertexts, generating multiple shared secret keys.  While this is safe, it is recommended that if the protocol allows it (if Alice and Bob exchange messages anyways) that Alice generates a fresh keypair each time (and zeroize the private key immediately after) to obtain Perfect Forward Secrecy. Be noted that generally key generation of ML-KEM is very fast. That is, if Alice's system is subverted (either by a hacker or a legal warrent), the previous communications remain secure (because Alice no longer has the information needed to recover the shared secret keys).

The shared secret key for all three parameter sets, ML-KEM-512, ML-KEM-768 and ML-KEM-1024 are 32 bytes which are indistinguishable from 32-byte psedorandom byte-strings of 128, 192 and 256 bits of strengths respectively. As such, it is suitable both to use directly as a symmetric key (for use by a symmetric cipher such as AES or a MAC), and for inserting into a Key Derivation Function. This is in contrast to a Diffie-Hellman (or ECDH) operation, where the output is distinguishable from random.

# IANA Considerations

This document has no IANA actions.

--- back

# Acknowledgments
{:numbered="false"}
