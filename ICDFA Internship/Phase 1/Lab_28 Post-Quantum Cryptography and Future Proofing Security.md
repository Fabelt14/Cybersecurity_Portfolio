# Post-Quantum Cryptography and Future Proofing Security

## Overview

This lab focused on implementing NIST-standardized post-quantum cryptographic algorithms to prepare for the quantum computing threat. The goal was to generate quantum-resistant key pairs, perform digital signatures resistant to Shor's algorithm, implement key encapsulation mechanisms, and understand hybrid cryptography migration strategies.

## Objectives

- Install and configure the Open Quantum Safe (OQS) provider for OpenSSL 3.x
- Generate ML-KEM (Kyber-768) key pairs for quantum-resistant key exchange
- Generate ML-DSA (Dilithium3) key pairs for quantum-resistant digital signatures
- Compare key sizes between PQC algorithms and traditional RSA/ECC
- Implement digital signatures using lattice-based cryptography
- Demonstrate Key Encapsulation Mechanisms (KEM) for secure key exchange
- Understand hybrid cryptography as a migration strategy
- Analyze the "Harvest Now, Decrypt Later" threat model

## Lab Environment

- **OS**: Kali Linux
- **OpenSSL Version**: 3.5.5 (January 27, 2026)
- **Required Libraries**: liboqs, cmake, make

## Tools Used

- openssl - Cryptographic operations and key generation
- git - Cloning the liboqs repository
- cmake - Building liboqs for system architecture
- make - Compiling liboqs into machine code
- sha256sum - Binary file comparison via cryptographic hashing

## Methodology

### Step 1: Verifying OpenSSL Version and Provider Architecture

I started by checking the OpenSSL version to ensure it was 3.x or higher, which introduced the provider architecture necessary for loading post-quantum algorithms.

**Why OpenSSL 3.x is required:**
The provider architecture enables the dynamic loading of cryptographic algorithm modules without requiring the recompilation of OpenSSL itself. This is critical for PQC because it enables us to add Kyber, Dilithium, and SPHINCS+ support by installing the OQS provider, rather than rebuilding the entire cryptographic library.

**Command used:**
```bash
openssl version
```

**Output:**
```
OpenSSL 3.5.5 27 Jan 2026 (Library: OpenSSL 3.5.5 27 Jan 2026)
```

![OpenSSL version verification](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/28.1%20OpenSSL%20version%20verification.jpg)

This confirmed I had OpenSSL 3.5.5, which supports the provider model needed for post-quantum algorithms.

### Step 2: Installing the Open Quantum Safe (OQS) Library

I cloned the liboqs repository from GitHub, which contains NIST-standardized post-quantum algorithms.

**Why liboqs is necessary:**
OpenSSL doesn't include PQC algorithms by default. The Open Quantum Safe project provides open-source implementations of ML-KEM, ML-DSA, and other NIST-approved algorithms that can be loaded as an OpenSSL provider.

**Command used:**
```bash
git clone https://github.com/open-quantum-safe/liboqs.git
cd liboqs
ls
```

![Cloning liboqs repository](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/28.2%20Cloning%20liboqs%20repository.jpg)

The repository contained algorithm implementations, build scripts, and test files.

### Step 3: Building liboqs for System Architecture

I used cmake to configure the build for my specific hardware, then compiled it with make.

**Why build from source:**
Pre-compiled binaries might not match your CPU architecture or might lack optimizations for your specific processor. Building from source ensures the cryptographic operations use the fastest available CPU instructions (like AVX2 or AES-NI).

**Commands used:**
```bash
cmake -S . -B build
cd build
make
```

![Building liboqs with cmake](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/28.3%20Building%20liboqs%20with%20cmake.jpg)

The build process compiled the lattice-based math into machine code that OpenSSL can load and execute.

### Step 4: Generating ML-KEM (Kyber-768) Key Pair

I generated a quantum-resistant key pair for key encapsulation using ML-KEM-768 (formerly Kyber-768).

**NIST naming update:**
During this lab, I discovered NIST renamed Kyber-768 to ML-KEM-768 (Module-Lattice-based Key Encapsulation Mechanism) in the final standardization. The OQS provider uses the updated naming.

**Why ML-KEM for key exchange:**
ML-KEM solves the key distribution problem without being vulnerable to Shor's algorithm. It's designed to protect against quantum computers breaking Diffie-Hellman or RSA key exchanges.

**Generating the private key:**
```bash
openssl genpkey -algorithm MLKEM768 -out pqc_kem_private.pem
```

**Extracting the public key:**
```bash
openssl pkey -in pqc_kem_private.pem -pubout -out pqc_kem_public.pem
```

![ML-KEM key generation](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/28.4%20ML-KEM%20key%20generation.jpg)

**Private key contents:**
```bash
cat pqc_kem_private.pem
```

![ML-KEM private key](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/28.5%20ML-KEM%20private%20key.jpg)

The private key is encoded in base64 but represents the lattice-based secret values.

**Public key contents:**
```bash
cat pqc_kem_public.pem
```

![ML-KEM public key](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/28.6%20ML-KEM%20public%20key.jpg)

The public key is significantly larger than traditional ECC keys due to the lattice structure.

### Step 5: Generating ML-DSA (Dilithium3) Key Pair

I generated a quantum-resistant key pair for digital signatures using ML-DSA-65 (formerly Dilithium3).

**NIST naming update:**
Dilithium3 was renamed to ML-DSA-65 (Module-Lattice-based Digital Signature Algorithm, security level 3). The "65" refers to the parameter set.

**Why ML-DSA for signatures:**
ML-DSA provides digital signatures that remain secure even against quantum computers. Traditional RSA and ECDSA signatures can be forged by quantum computers using Shor's algorithm.

**Generating the private key:**
```bash
openssl genpkey -algorithm MLDSA65 -out pqc_sig_private.pem
```

![ML-DSA private key generation](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/28.7%20ML-DSA%20private%20key%20generation.jpg)

**Private key contents:**
```bash
cat pqc_sig_private.pem
```

![ML-DSA private key](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/28.8%20ML-DSA%20private%20key.jpg)

**Extracting the public key:**
```bash
openssl pkey -in pqc_sig_private.pem -pubout -out pqc_sig_public.pem
```

![ML-DSA public key generation](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/28.9%20ML-DSA%20public%20key%20generation.jpg)

**Public key contents:**
```bash
cat pqc_sig_public.pem
```

![ML-DSA public key](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/28.10%20ML-DSA%20public%20key.jpg)

### Step 6: Comparing Key Sizes - PQC vs Traditional Cryptography

I used `ls -lh` to compare file sizes between PQC keys and traditional RSA/ECC keys.

**Command used:**
```bash
ls -lh *.pem
```

**Findings:**
- Traditional RSA-2048 private key: ~1.8KB
- Traditional RSA-2048 public key: ~451 bytes
- ML-KEM-768 private key: ~1.4KB
- ML-KEM-768 public key: ~1.7KB
- ML-DSA-65 private key: ~3.9KB
- ML-DSA-65 public key: ~2.7KB

![Key size comparison](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/28.11%20Key%20size%20comparison.jpg)

**Why PQC keys are larger:**
Lattice-based cryptography relies on mathematical structures (polynomials, matrices) that require more data to represent than simple prime numbers used in RSA. The added size is the trade-off for quantum resistance.

**Security equivalence:**
Just as AES-256 is more secure than AES-128 due to larger key size, PQC keys are larger because they encode more complex mathematical problems that quantum computers cannot efficiently solve.

### Step 7: Understanding Network Impact of Larger Keys

I analyzed how the larger PQC keys affect network protocols.

**Impact on TLS (over TCP):**
When PQC keys are transmitted over HTTPS using TLS:
1. TCP segments the large keys into multiple packets
2. Each packet requires acknowledgment before sending the next
3. This back-and-forth increases latency compared to smaller ECC keys
4. However, TCP guarantees delivery, so the connection will eventually complete

**Impact on IKEv2 (over UDP):**
When PQC keys are transmitted in VPN connections using IKEv2:
1. The large keys exceed the standard MTU (Maximum Transmission Unit) of 1500 bytes
2. This forces IP fragmentation, splitting the key into multiple IP fragments
3. UDP does not guarantee delivery - if even one fragment is lost, the entire key is unusable
4. This causes frequent connection failures in VPN handshakes

**Mitigation strategies:**
- For VPNs: Increase MTU size, use TCP-based VPN protocols, or implement hybrid approaches
- For web servers: Accept the latency increase as acceptable for quantum security

### Step 8: Creating and Signing a Document with ML-DSA

I created a message file and signed it using the ML-DSA private key to demonstrate quantum-resistant digital signatures.

**Creating the message:**
```bash
echo "This message is protected by NIST-standardized Post-Quantum Cryptography." > pqc_message.txt
cat pqc_message.txt
```

![Creating message file](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/28.12%20Creating%20message%20file.jpg)

**Signing the document:**
```bash
openssl dgst -sign pqc_sig_private.pem -out pqc_message.sig pqc_message.txt
```

![Signing document with ML-DSA](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/28.13%20Signing%20document%20with%20ML-DSA.jpg)

**Signature file contents:**
```bash
cat pqc_message.sig
```

![Digital signature file](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/28.14%20Digital%20signature%20file.jpg)

The signature is encoded in base64 and represents the lattice-based mathematical proof that this message was signed by the holder of the private key.

**How ML-DSA differs from RSA signatures:**
RSA signatures rely on the difficulty of factoring large numbers. ML-DSA signatures rely on the Module Learning With Errors (MLWE) problem - given a mathematical operation with intentional noise added, it's computationally infeasible to reverse-engineer the original inputs, even with a quantum computer.

### Step 9: Verifying the Digital Signature

I verified the signature to prove the message hasn't been tampered with and was signed by the correct private key.

**Command used:**
```bash
openssl dgst -verify pqc_sig_public.pem -signature pqc_message.sig pqc_message.txt
```

**Output:**
```
Verified OK
```

![Signature verification](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/28.15%20Signature%20verification.jpg)

**What this proves:**
- The message was signed by the holder of pqc_sig_private.pem
- The message has not been modified since signing
- The signature is mathematically valid according to the ML-DSA algorithm

**Quantum resistance:**
Even if an attacker has a quantum computer and intercepts this signature, they cannot:
1. Forge a signature for a different message
2. Extract the private key from the signature
3. Modify the message without detection

This is because the MLWE problem remains hard even for quantum computers.

### Step 10: Key Encapsulation - Generating a Shared Secret

I simulated secure key exchange using ML-KEM's Key Encapsulation Mechanism.

**The process:**
1. Sender uses the receiver's public key to generate a random symmetric password
2. That password is "locked" using the public key and sent as the encapsulated key
3. Only the receiver with the matching private key can unlock and retrieve the password

**Encapsulation command:**
```bash
openssl pkeyutl -encap -pubin -inkey pqc_kem_public.pem -secret shared_secret.bin -out encapsulated_key.bin
```

![Key encapsulation](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/28.16%20Key%20encapsulation.jpg)

**Shared secret (the raw password):**
```bash
cat shared_secret.bin
```

![Shared secret contents](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/28.17%20Shared%20secret%20contents.jpg)

This is the symmetric key that will be used to encrypt the actual message (like an AES key).

**Encapsulated key (the locked password):**
```bash
cat encapsulated_key.bin
```

![Encapsulated key contents](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/28.18%20Encapsulated%20key%20contents.jpg)

This is what gets transmitted over the network. It can only be unlocked with the private key.

### Step 11: Key Decapsulation - Recovering the Shared Secret

I reversed the process by using the private key to unlock the encapsulated key and recover the symmetric password.

**Decapsulation command:**
```bash
openssl pkeyutl -decap -inkey pqc_kem_private.pem -in encapsulated_key.bin -out recovered_secret.bin
```

![Key decapsulation](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/28.19%20Key%20decapsulation.jpg)

**Recovered secret:**
```bash
cat recovered_secret.bin
```

![Recovered secret contents](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/28.20%20Recovered%20secret%20contents.jpg)

### Step 12: Verifying the Shared Secrets Match

I used cryptographic hashing to mathematically prove that both keys are identical.

**Why sha256sum instead of diff:**
The diff command compares text files. Cryptographic keys are binary data. Using sha256sum generates a unique fingerprint for each file - if the fingerprints match, the files are 100% identical.

**Command used:**
```bash
sha256sum shared_secret.bin recovered_secret.bin
```

**Output:**
```
4eb1d9ae30e25fbeec4a74b5dc72833b5d41974f2e8a3274f8a29dfd70c282f2  shared_secret.bin
4eb1d9ae30e25fbeec4a74b5dc72833b5d41974f2e8a3274f8a29dfd70c282f2  recovered_secret.bin
```

![SHA256 hash comparison](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/28.21%20SHA256%20hash%20comparison.jpg)

The identical hashes prove both symmetric keys are the same. The sender and receiver now have a shared AES key without ever transmitting it in the clear.

**Why KEM is better than Diffie-Hellman for PQC:**
Traditional Diffie-Hellman requires both parties to perform interactive math that produces the same result. With lattice-based crypto using MLWE (which intentionally adds mathematical noise), interactive exchange becomes unreliable because the noise compounds and prevents both sides from reaching the same answer. KEM solves this: one party generates the key, locks it with the other party's public key, and sends it. No interactive math needed, no noise-compounding issues.

## Findings

**OpenSSL Provider Architecture:**
- OpenSSL 3.x introduced modular providers, allowing PQC algorithms to be added without rebuilding OpenSSL
- The OQS provider successfully loaded ML-KEM and ML-DSA algorithms
- Algorithm names were updated from Kyber-768 to ML-KEM-768 and Dilithium3 to ML-DSA-65 during NIST standardization

**Key Size Analysis:**
- ML-KEM-768 keys are approximately 3x larger than RSA-2048 keys
- ML-DSA-65 keys are approximately 5x larger than RSA-2048 keys
- Larger keys are necessary to encode the lattice structures that provide quantum resistance

**Network Protocol Impact:**
- TCP-based protocols (TLS, HTTPS) experience latency increases but maintain reliability
- UDP-based protocols (IKEv2, VPNs) suffer from fragmentation issues and potential connection failures
- MTU adjustments or TCP-encapsulation are required for production VPN deployments

**Digital Signature Verification:**
- ML-DSA signatures successfully verified message integrity and authenticity
- The MLWE problem (intentional mathematical noise) makes signatures quantum-resistant
- Signature sizes are larger but provide future-proof security

**Key Encapsulation Mechanism:**
- KEM successfully generated and transmitted symmetric keys without interactive exchange
- Decapsulation recovered the exact same key (verified via SHA-256 hash)
- KEM avoids the noise-compounding problems that affect lattice-based Diffie-Hellman

**Hybrid Cryptography Strategy:**
- Combining classical algorithms (X25519) with PQC algorithms (Kyber) provides dual-layer protection
- If quantum computers break classical crypto, PQC maintains security
- If vulnerabilities are found in new PQC algorithms, classical crypto provides fallback protection
- Hybrid approach maintains FIPS compliance while transitioning to PQC

## Challenges Faced

**Algorithm naming confusion:**
Initially attempted to use `kyber768` as the algorithm name, but received errors. Research revealed NIST renamed algorithms during final standardization: Kyber became ML-KEM, Dilithium became ML-DSA. The OQS provider uses the updated names.

**Build dependencies:**
The cmake build initially failed due to missing compiler toolchains. Had to install build-essential and cmake packages before liboqs would compile successfully.

**Binary file comparison:**
Initially tried using `diff` to compare shared_secret.bin and recovered_secret.bin, but diff is designed for text files. Learned that binary files require hash-based comparison using sha256sum for mathematical proof of identity.

**Understanding KEM vs Diffie-Hellman:**
Conceptually struggled with why lattice-based systems use KEM instead of traditional key exchange. Research clarified that MLWE (the noise intentionally added for quantum resistance) makes interactive exchange unreliable, forcing the use of one-way key encapsulation instead.

## Key Takeaways

- **Provider architecture enables PQC adoption:** OpenSSL 3.x allows quantum-resistant algorithms to be added dynamically, avoiding the need to rebuild cryptographic infrastructure from scratch.
- **Key size is the price of quantum resistance:** Lattice-based keys are 3-5x larger than classical keys because they encode more complex mathematical structures that resist quantum attacks.
- **Network protocols need adjustment:** Larger keys cause TCP latency and UDP fragmentation. Production deployments require MTU tuning or protocol changes.
- **MLWE provides quantum resistance:** Module Learning With Errors (intentional mathematical noise) makes it computationally infeasible for even quantum computers to reverse-engineer lattice-based cryptographic operations.
- **KEM solves the interactive exchange problem:** Traditional Diffie-Hellman requires both parties to perform symmetrical math. Lattice noise makes this unreliable, so KEM uses one-way encapsulation instead.
- **Hybrid cryptography is the migration path:** Running both classical and PQC algorithms simultaneously provides protection against both current and future threats while maintaining regulatory compliance.
- **HNDL attacks make migration urgent:** Adversaries are harvesting encrypted data today to decrypt in the future when quantum computers are available. Organizations must migrate to PQC now to protect data that requires long-term confidentiality.
- **NIST standardization is ongoing:** Algorithm names and parameters are still being finalized. Staying current with NIST updates is critical for production implementations.

## Disclaimer

This lab was performed in a controlled Kali Linux environment for educational purposes as part of the ICDFA Cryptography course. All cryptographic operations were conducted on a local system with proper authorization. Post-quantum algorithms used are NIST-standardized but should be deployed in production only after thorough security review and compliance verification.
