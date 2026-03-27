# A Comprehensive Journey into Secure Communication

## Overview

This lab covered cryptographic principles from classical ciphers to post-quantum security. The goal was to understand how encryption, hashing, and digital signatures protect data, identify weaknesses in outdated algorithms, and implement secure communication protocols using Python and OpenSSL.

## Objectives

- Decrypt classical ciphers manually and through brute force
- Implement symmetric encryption with different modes (ECB, CBC, GCM)
- Understand hash functions, collisions, and integrity verification
- Generate and use asymmetric key pairs for encryption and signatures
- Analyze PKI trust chains and certificate validation
- Identify weak cipher suites and broken cryptographic implementations
- Explore post-quantum cryptography and zero-knowledge proofs

## Lab Environment

- **OS**: Kali Linux
- **Tools**: OpenSSL, Python 3, GnuPG
- **Python Libraries**: cryptography, pycryptodome, bcrypt, hashlib

## Tools Used

- openssl - TLS/SSL toolkit for encryption, key generation, and certificate management
- python3 - Scripting language for implementing crypto algorithms
- gnupg - File encryption and digital signature verification
- hashpump - Hash length extension attack tool
- nmap - Cipher suite enumeration

![Installation of required tools](images/tool-installation.png)

## Methodology

### Level 1: Classical Cryptography

#### Exercise 1: Manual Caesar Cipher Decryption

I needed to decrypt the ciphertext "WTAAD" using a shift of k=15.

**Understanding the Caesar cipher:**
Each letter shifts backward by 15 positions in the alphabet. W becomes L, T becomes I, and so on.

**Manual decryption:**
- W - 15 = L
- T - 15 = I
- A - 15 = P
- A - 15 = P
- D - 15 = S

Result: LIPPS

![Manual Caesar decryption](images/caesar-manual.png)

#### Exercise 2: Caesar Brute Force Attack

I intercepted the ciphertext "XAB" and needed to try all 25 possible shifts to find the English word.

**Why brute force works on Caesar:**
Only 25 possible keys exist (shifts 1-25). Testing all of them takes seconds.

I wrote a Python script that tested every shift:
- Shift 1: WZA
- Shift 2: VYZ
- Shift 3: UXY
- ...
- Shift 23: ADE

Only one shift produced a recognizable English word.

![Caesar brute force output](images/caesar-brute.png)

#### Exercise 3: Python Caesar Cipher Implementation

I implemented a full Caesar cipher that handles both encryption and decryption with user input.

**Design decisions:**
- The script asks for plaintext or ciphertext first
- User specifies shift value (1-25)
- Mode selection (encrypt/decrypt) determines direction
- Encryption shifts forward, decryption shifts backward by making shift negative

**How the algorithm works:**
For each character, if it's a letter:
1. Convert to uppercase
2. Get its position (A=0, B=1... Z=25)
3. Apply shift with modulo 26 to wrap around
4. Convert back to character

Non-alphabetic characters pass through unchanged.

![Python Caesar cipher code](images/caesar-code.png)

**Testing with "CRYPTOGRAPHY" and shift 7:**
- C + 7 = J
- R + 7 = Y
- Y + 7 = F
- ...

Result: JYFWAVNYHWOF

![Caesar encryption test](images/caesar-encrypt.png)

**Decryption verification:**
Running the same ciphertext with decrypt mode and shift 7 returned "CRYPTOGRAPHY", confirming the implementation works correctly.

![Caesar decryption test](images/caesar-decrypt.png)

#### Exercise 4: Manual Vigenère Cipher Decryption

I needed to decrypt "LXFOPVEFRNHR" using the key "LEMON".

**How Vigenère differs from Caesar:**
Instead of one fixed shift, each letter uses a different shift based on the key. The key repeats:
- L (shift 11), E (shift 4), M (shift 12), O (shift 14), N (shift 13)

**Decryption process:**
```
Ciphertext: L X F O P V E F R N H R
Key:        L E M O N L E M O N L E
Shift:      11 4 12 14 13 11 4 12 14 13 11 4
Plaintext:  A T T A C K A T D A W N
```

Each ciphertext letter - corresponding key shift = plaintext letter

Result: ATTACKATDAWN

#### Exercise 5: Python Vigenère Implementation

I wrote a script that takes ciphertext and a key, then performs Vigenère decryption.

**Algorithm logic:**
1. Convert key to uppercase and repeat it to match ciphertext length
2. For each character in ciphertext:
   - Get the shift from the corresponding key character
   - Subtract that shift (with modulo 26 wrap)
   - Build the plaintext character by character

**Key index tracking:**
The `key_index` variable cycles through the key, resetting to 0 when it reaches the end. This allows the key to repeat for longer messages.

![Vigenère cipher code](images/vigenere-code.png)

**Test with LXFOPVEFRNHR and key LEMON:**

![Vigenère decryption output](images/vigenere-output.png)

### Level 2: Symmetric Key Cryptography

#### Exercise 6: OpenSSL AES-128-CBC with Wrong Password

I encrypted a file with AES-128-CBC, then attempted decryption with an incorrect password to observe the behavior.

**Encryption command:**
```bash
openssl enc -aes-128-cbc -salt -in wrong_secret.txt -out w_encrypted.bin -k fatai
```

The `-salt` flag adds random data to prevent rainbow table attacks on the password.

**Decryption with wrong password:**

When I used the wrong password, OpenSSL created the output file and began writing decrypted data immediately. It only detected the error at the final block when checking the padding.

**Why the error appears late:**
AES processes data in 16-byte blocks. OpenSSL decrypts each block and writes output as it goes. Padding validation happens at the very end - if the padding bytes don't match the expected pattern, it throws "bad decrypt".

This means an attacker could partially decrypt a file with a wrong password and not know it failed until the end.

![Wrong password decryption attempt](images/aes-wrong-password.png)

#### Exercise 7: ECB vs CBC Mode Comparison

I encrypted the same BMP image using AES-256-ECB and AES-256-CBC to compare their security properties.

**ECB encryption:**
```bash
openssl enc -aes-256-ecb -salt -in image.bmp -out ecb.bin -k fatai
```

**Why ECB is weak:**
ECB (Electronic Codebook) encrypts each 16-byte block independently with the same key. Identical plaintext blocks produce identical ciphertext blocks.

**Result:**
When I examined ecb.bin, repeated patterns from the original image were still visible in the encrypted output. If the image had large areas of the same color (like a white background), those areas would produce repeating ciphertext blocks.

![ECB encrypted output showing patterns](images/ecb-pattern.png)

**CBC encryption:**
```bash
openssl enc -aes-256-cbc -salt -in image.bmp -out cbc.bin -k fatai
```

**Why CBC is secure:**
CBC (Cipher Block Chaining) XORs each plaintext block with the previous ciphertext block before encryption. This means identical plaintext blocks produce different ciphertext, eliminating the pattern problem.

**Result:**
The cbc.bin output looked like random data with no visible patterns, even if the original image had repeated sections.

![CBC encrypted output - no patterns](images/cbc-random.png)

**Security implication:**
An attacker with an ECB-encrypted image can identify boundaries between objects without decrypting anything. With CBC, the entire image appears as noise.

#### Exercise 8: Python AES-GCM Implementation

I implemented AES-256-GCM encryption in Python, which provides both confidentiality and authentication.

**Why GCM matters:**
GCM (Galois/Counter Mode) adds an authentication tag to detect tampering. Without it, an attacker could flip bits in ciphertext and corrupt specific parts of the decrypted message.

**Implementation:**
```python
from cryptography.hazmat.primitives.ciphers.aead import AESGCM
import os

# Generate 256-bit key (32 bytes)
key = AESGCM.generate_key(bit_length=256)

# Generate 96-bit nonce (12 bytes) - must be unique per encryption
nonce = os.urandom(12)

# Encrypt
aesgcm = AESGCM(key)
ciphertext_with_tag = aesgcm.encrypt(nonce, user_text.encode(), None)
```

The `encrypt()` call returns ciphertext + a 16-byte authentication tag appended to the end.

![AES-GCM implementation](images/gcm-code.png)

**Test output:**
Encrypting "Authenticated Data" produced:
- Nonce (IV): 26b71286ea6f2ab08fe1007a
- Ciphertext: 878543ec5484163ad8e5149aadbea9f20a69
- Auth Tag: ddce06380e79ce266b678a008908dead

![GCM encryption output](images/gcm-output.png)

If even one bit changes in the ciphertext, decryption fails with "authentication tag mismatch".

#### Exercise 9: Secure File Encryption Tool

I built a Python tool that encrypts files using AES-256-GCM with password-based key derivation.

**Security features implemented:**
1. **PBKDF2HMAC** for key derivation - prevents rainbow table attacks on passwords
2. **100,000 iterations** - slows down brute force attempts
3. **Random salt and nonce** - ensures same password produces different ciphertext each time
4. **Authentication tag** - detects file tampering

**Workflow:**
```
User password → PBKDF2-HMAC-SHA256 → 256-bit AES key
Random nonce (12 bytes) → stored with ciphertext
Random salt (16 bytes) → stored for key derivation
```

The encrypted file stores: `salt + nonce + ciphertext + tag`

![File encryption tool code](images/file-encrypt-code.png)

**Test - successful decryption:**
1. Created test_file.txt with content "Authenticating Data"
2. Encrypted with password "fatai"
3. Decrypted with same password
4. File contents restored correctly

![Successful file decryption](images/file-decrypt-success.png)

**Test - wrong password:**
Attempted decryption with password "fa34" resulted in:
```
ACCESS DENIED: Incorrect password or corrupted file.
Reason: GCM Authentication Tag mismatch.
```

This confirms the authentication prevents silent corruption from wrong passwords.

![Failed decryption with wrong password](images/file-decrypt-fail.png)

### Level 3: Hashing and Message Integrity

#### Exercise 10: Computing Hash Values

I computed MD5 and SHA-256 hashes for three strings to observe their characteristics.

**Test strings:**
- "Hello World"
- "Secure Hashing"
- "INT306 Cryptography"

**Commands:**
```bash
echo -n "Hello World" | openssl md5
echo -n "Hello World" | openssl sha256
```

**Results for "Hello World":**
- MD5: ed076287532e86365e841e92bfc50d8c (32 hex characters = 128 bits)
- SHA-256: (64 hex characters = 256 bits)

![MD5 hash output](images/hash-md5-hello.png)

**Results for "Secure Hashing":**
- MD5: 95a5bc5ee6b414cfcca1c67ba6f76eed

![MD5 hash output](images/hash-md5-secure.png)

**Results for "INT306 Cryptography":**
- MD5: 1d5a7039d3c83aed53aa21215e795fe8
- SHA-256: dd743610105be6f4e74d4d6aedfd09f999fe0cf0a8a60fcf9f98d3c92ec3ba572

![SHA-256 hash output](images/hash-sha256-int306.png)

**Key difference:**
MD5 produces 128-bit hashes (32 hex chars) while SHA-256 produces 256-bit hashes (64 hex chars). SHA-256's longer output makes collisions exponentially harder to find - approximately 2^128 times harder than MD5.

#### Exercise 11: File Integrity Verification

I generated a SHA-256 checksum for image.bmp and verified its integrity, then tested what happens when the file is modified.

**Step 1: Generate hash and save to file**
```bash
sha256sum image.bmp > sha_image.txt
```

This creates a file containing:
```
21041583c5896e0f0bc1b25c753f814f569e90691a56a64b8ba77cba73de21  image.bmp
```

![Generate SHA-256 hash](images/sha-generate.png)

**Step 2: View the hash file**
```bash
cat sha_image.txt
```

The format is: 64-character hash, two spaces, filename

![View hash file contents](images/sha-view.png)

**Step 3: Verify unmodified file**
```bash
sha256sum -c sha_image.txt
```

Output: `image.bmp: OK`

This means the computed hash matches the stored hash - file is intact.

![Successful integrity check](images/sha-verify-ok.png)

**Step 4: Modify file and verify again**

After changing even one byte in image.bmp:
```bash
sha256sum -c sha_image.txt
```

Output:
```
image.bmp: FAILED
sha256sum: WARNING: 1 computed checksum did NOT match
```

![Failed integrity check](images/sha-verify-failed.png)

This proves SHA-256 detects any modification - changing a single bit completely changes the hash output.

#### Exercise 12: Avalanche Effect Demonstration

I created two text files differing by one character to demonstrate how hash functions amplify tiny input changes.

**Step 1: Create original.txt**
```bash
echo "The quick brown fox jumps over the lazy dog." > original.txt
sha256sum original.txt > original_hash.txt
```

Hash: b47cc0f104b62d4c7c30bcd08fd8e67613e287dc4ad8c310ef10cbadea9c4380

![Original file hash](images/avalanche-original.png)

**Step 2: Create modified.txt (changed "dog" to "cog")**
```bash
echo "The quick brown fox jumps over the lazy cog." > modified.txt
sha256sum modified.txt > modified_hash.txt
```

Hash: 8fd09c5556e586329a3bbc226ba1abff3f4a2b6967beb34c3fab0f5d28800581

![Modified file hash](images/avalanche-modified.png)

**Comparison:**
```
Original:  b47cc0f104b62d4c7c30bcd08fd8e67613e287dc4ad8c310ef10cbadea9c4380
Modified:  8fd09c5556e586329a3bbc226ba1abff3f4a2b6967beb34c3fab0f5d28800581
```

Changing one letter ('d' to 'c') caused 32 out of 64 hex characters to change - approximately 50% of the hash is different.

**Why this matters for security:**
The avalanche effect ensures attackers can't make small, controlled changes to files while keeping the same hash. Even knowing the original hash provides zero information about nearby hashes.

SHA-256 achieves this through bitwise operations (rotate right, shift right, XOR) that propagate changes throughout the entire hash, making the output unpredictable even for single-bit input changes.

#### Exercise 13: HMAC Generation

I generated an HMAC-SHA256 for a financial transaction message using a secret key.

**Command:**
```bash
echo -n "Financial Transaction: $500" | openssl sha256 -hmac "super-secret-key-123"
```

**What HMAC adds:**
Unlike a plain hash, HMAC (Hash-based Message Authentication Code) includes a secret key in the calculation. Without the key, you cannot produce the correct output.

**Output:**
```
SHA2-256(stdin)= d096dff186ab14c32b59b0602733acc1ac8e83063efd38a6c227f13ee80afac9
```

![HMAC generation](images/hmac-output.png)

**Security property:**
If someone intercepts the transaction message and modifies "$500" to "$5000", they cannot generate a valid HMAC without knowing "super-secret-key-123". The recipient recomputes the HMAC with their copy of the key - if it doesn't match, the message was tampered with.

This is how banking APIs verify that transaction data hasn't been modified in transit.

#### Exercise 14: Password Hashing with bcrypt

I implemented a Python password manager using bcrypt, which is designed specifically for password storage.

**Why bcrypt instead of SHA-256:**
- bcrypt includes automatic salt generation
- Adjustable work factor (rounds=12 means 2^12 iterations)
- Intentionally slow to resist brute force
- Memory-hard algorithm resists GPU cracking

**Implementation:**
```python
import bcrypt

# Registration - hash password with random salt
password = input("Set password: ").encode()
hashed_password = bcrypt.hashpw(password, bcrypt.gensalt(rounds=12))

# Login - verify attempt against stored hash
attempt = input("Enter password: ").encode()
if bcrypt.checkpw(attempt, hashed_password):
    print("ACCESS GRANTED")
else:
    print("ACCESS DENIED")
```

![bcrypt implementation code](images/bcrypt-code.png)

**Test - successful login:**
1. Set password: "fatai"
2. Stored hash: $2b$12$xmJeXiRmNKK4lb2lzCAN6.jc2OIsf0kkUQPgjZtDqeE5Vt3B5hLQS
3. Login attempt: "fatai"
4. Result: ACCESS GRANTED

![Successful password verification](images/bcrypt-success.png)

**Test - failed login:**
1. Login attempt: "fat"
2. Result: ACCESS DENIED

![Failed password verification](images/bcrypt-fail.png)

**Why bcrypt is preferred:**
If an attacker steals the password database, bcrypt's slow hashing (100-300ms per attempt) makes brute forcing millions of passwords impractical. SHA-256 can compute billions of hashes per second on a GPU.

#### Exercise 15: Rainbow Table Concept

A rainbow table is a precomputed database that trades computation time for storage space in password cracking.

**How it works:**

**Step 1: Build the table**
- Start with millions of common passwords (123456, password, qwerty, etc.)
- Hash each password using the target algorithm (e.g., MD5)
- Apply a "reduction function" that converts the hash back into a different plaintext string
- Hash that string again, creating a chain
- Store only the first plaintext and the final hash

Example chain:
```
password123 → hash → reduce → apple45 → hash → reduce → final_hash
Store: (password123, final_hash)
```

This chain represents thousands of intermediate hashes while only storing two values.

**Step 2: Crack a stolen hash**
- Look for the stolen hash at the end of any chain
- If found, recalculate that chain from the beginning
- The plaintext that produces the stolen hash is the password

**Why salting defeats rainbow tables:**
```
Unsalted: MD5("password") = 5f4dcc3b5aa765d61d8327deb882cf99
With salt: MD5("password" + "a7f3k2p9") = completely different hash
```

An attacker would need a separate rainbow table for every possible salt value. With a 128-bit random salt, that means 2^128 different tables - storing those would require more data than exists in the universe.

#### Exercise 16: MD5 Collision Demonstration

I downloaded two files (hello1.exe and hello2.exe) that produce identical MD5 hashes despite having different contents.

**Download the collision files:**
```bash
wget https://www.win.tue.nl/hashclash/SoftIntCodeSign/HelloWorld-colliding.exe -O hello1.exe
wget https://www.win.tue.nl/hashclash/SoftIntCodeSign/GoodbyeWorld-colliding.exe -O hello2.exe
```

![Download collision files](images/collision-download.png)

**Verify the collision:**
```bash
md5sum hello1.exe
md5sum hello2.exe
```

Both output: `0c30c8116ff7e621164cbabfc7359cd4`

![Identical MD5 hashes](images/collision-md5.png)

**Security implications:**

**Digital signatures broken:**
In a signature system, you sign the hash of a document, not the document itself. If I can create two documents with the same MD5 hash - one legitimate ("I owe you $10") and one malicious ("I transfer my house to you") - then your signature on the good document is mathematically valid for the evil one.

**Software integrity destroyed:**
If I publish a legitimate software update with MD5 checksum `5f4dcc...`, an attacker can create malware with the same checksum. Users download the malware, verify the MD5, see it matches the official website, and install it thinking it's safe.

**Why this matters in 2026:**
Some legacy systems still use MD5 for file integrity checks. Any security system relying on MD5 can be bypassed using collision techniques.

#### Exercise 17: Rainbow Table Creation

I implemented a simple rainbow table in Python that stores MD5 hashes of common passwords, then attempts to crack a given hash.

**Code structure:**
```python
import hashlib

# Build table
passwords = input("Enter passwords: ").split(",")
rainbow_table = {}

for pwd in passwords:
    hash_value = hashlib.md5(pwd.encode()).hexdigest()
    rainbow_table[hash_value] = pwd

# Crack hash
target_hash = input("Enter MD5 hash to crack: ")
if target_hash in rainbow_table:
    print(f"Password found: {rainbow_table[target_hash]}")
else:
    print("No match in table")
```

![Rainbow table implementation](images/rainbow-code.png)

**Test:**
Passwords stored: fatai, password, 1234

Attempting to crack: `88a5d978cad92b881c91f2d9d299e3a` (hash of "fatai")

Result: MATCH FOUND - The password is 'fatai'

![Successful rainbow table crack](images/rainbow-crack-success.png)

Attempting to crack: `1ff4cb63830672fd08060bebf59be569b` (not in table)

Result: NO MATCH - This hash is not in your current table

![Failed rainbow table crack](images/rainbow-crack-fail.png)

**Limitations:**

**Memory vs Scope:**
This table stores only 3 passwords. A real rainbow table needs billions of entries, requiring terabytes of storage.

**Algorithm Specificity:**
This table only works for MD5. If the target uses SHA-1 or bcrypt, the table is useless.

**Complexity:**
A password like "Xj8#kL!p2" would never appear in a common password table. Storing all possible 8-character combinations (62^8 = 218 trillion) is impractical.

**How salting prevents this attack:**
```
Hash("password") = 5f4dcc...  ← in rainbow table
Hash("password" + "a7f3k2p9") = 94e3b2... ← not in table
```

With random salts, an attacker needs a different rainbow table for each user. 10,000 users = 10,000 tables = storage requirements become impossible.

#### Exercise 18: Hash Functions in Git (Research Report)

**Role of Hash Functions**

Git doesn't track files by name. Every commit, file (blob), and directory (tree) is identified by the hash of its content. When you run `git commit`, Git computes a SHA-1 hash of your files, metadata, and the hash of the previous commit. This creates an immutable chain where changing any historical commit would require recalculating every subsequent commit hash.

**Specific Algorithms Used**

Git used SHA-1 (160-bit) exclusively until 2020. Modern Git versions support SHA-256 (256-bit) as an alternative object format. Users can initialize new repositories with `git init --object-format=sha256` for high-security environments.

**Benefits**

Git organizes hashes in a Merkle Tree structure. Each commit hash includes the hash of all files in that commit plus the hash of the parent commit. This creates an unbreakable audit trail - to alter a file from three years ago, an attacker must recompute hashes for every commit since then. The computational cost makes Git's history effectively tamper-proof, which is why blockchain technology adopted the same pattern.

Hash-based storage also enables deduplication. If two branches contain identical files, Git stores only one copy since both have the same hash.

**Vulnerabilities and Real-World Implications**

The SHAttered attack in 2017 demonstrated a practical SHA-1 collision: two different PDF files with identical hashes. An attacker could theoretically craft malicious code that shares a hash with legitimate code, tricking Git into accepting the bad version during a merge.

While accidental collisions remain astronomically unlikely (1 in 10^48), the transition to SHA-256 protects against state-sponsored actors with massive computing resources.

**Conclusion**

Hashing is Git's identity system. Every object is verified by content, not metadata. For developers, the hash is a receipt proving that code retrieved today is byte-for-byte identical to code committed yesterday.

#### Exercise 19: Hash Table Implementation

I implemented a hash table class in Python with insert, search, delete, and collision resolution through chaining.

**Code structure:**
```python
class HashTable:
    def __init__(self, size=10):
        self.size = size
        self.table = [[] for _ in range(size)]  # List of buckets
    
    def _hash(self, key):
        # Simple modulo hash
        return sum(ord(char) for char in str(key)) % self.size
    
    def insert(self, key, value):
        hash_key = self._hash(key)
        # Check if key exists, update or append
        for i, kv in enumerate(self.table[hash_key]):
            if kv[0] == key:
                self.table[hash_key][i] = (key, value)
                return
        self.table[hash_key].append((key, value))
```

![Hash table implementation](images/hashtable-code.png)

**Test - inserting 5 entries:**
```
Insert: 00 → encryption
Insert: 01 → fatai
Insert: 01 → icdfa  (collision with 00)
Insert: 02 → cybersecurity
Insert: 03 → hashing
```

![Hash table insertions](images/hashtable-insert.png)

**Viewing internal structure:**
```
Bucket 0: [('04', 'encryption')]
Bucket 1: [('00', 'fatai')]
Bucket 2: [('01', 'icdfa')]
Bucket 3: [('02', 'cybersecurity')]
Bucket 4: [('03', 'hashing')]
```

![Hash table structure](images/hashtable-structure.png)

**Test - search operation:**
Searching for key "00" returns: fatai
Searching for key "03" returns: hashing

![Hash table search](images/hashtable-search.png)

**Test - delete operation:**
Deleting key "04" removes "encryption" from bucket 0

![Hash table delete](images/hashtable-delete.png)

**How the hash function contributes to efficiency:**

The `_hash()` method converts strings to integers mod table size. This distributes keys across buckets:
- "00" → sum of ASCII values = 96 → 96 % 10 = bucket 6
- "encryption" → different sum → different bucket

Average case: O(1) lookup (direct bucket access)
Worst case: O(n) if all keys collide into one bucket

**Collision resolution through chaining:**
Each bucket is a list. When two keys hash to the same bucket, both are stored in that list. Search operation loops through the bucket to find the matching key.

This is simpler than open addressing (probing for empty slots) but requires extra memory for lists.

### Level 4: Asymmetric Key Cryptography

#### Exercise 20: Manual RSA Calculation

Given primes p=3 and q=11, I performed RSA encryption and decryption by hand.

**Step 1: Calculate n**
```
n = p * q = 3 * 11 = 33
```

**Step 2: Calculate φ(n)**
```
φ(n) = (p-1) * (q-1) = (3-1) * (11-1) = 2 * 10 = 20
```

**Step 3: Choose e and find d**
Given e = 3, find d such that (e * d) mod φ(n) = 1
```
3 * d ≡ 1 (mod 20)
Testing values:
3 * 7 = 21
21 mod 20 = 1 ✓

Therefore d = 7
```

**Step 4: Encrypt M=5**
```
C = M^e mod n
C = 5^3 mod 33
C = 125 mod 33
C = 26
```

**Step 5: Decrypt C=26**
```
M = C^d mod n
M = 26^7 mod 33

Breaking it down:
26^2 = 676 mod 33 = 16
26^4 = 16^2 = 256 mod 33 = 25
26^7 = 26^4 * 26^2 * 26 = 25 * 16 * 26 mod 33

25 * 16 = 400 mod 33 = 4
4 * 26 = 104 mod 33 = 5

Therefore M = 5 ✓
```

The encryption and decryption cycle returned the original message, proving the math works.

#### Exercise 21: Python RSA Encryption

I generated an RSA key pair, encrypted "Top Secret" with the public key, then decrypted with the private key.

**Step 1: Generate 2048-bit RSA private key**
```bash
openssl genrsa -out private.pem 2048
```

![Generate RSA private key](images/rsa-genkey.png)

**Step 2: Extract public key**
```bash
openssl rsa -in private.pem -pubout -out public.pem
```

The public key contains only (n, e) while the private key contains (n, d).

![Extract RSA public key](images/rsa-pubkey.png)

**Step 3: Encrypt message**
```bash
echo -n "Top Secret" > message.txt
openssl pkeyutl -encrypt -pubin -inkey public.pem \
  -in message.txt -out message.enc \
  -pkeyopt rsa_padding_mode:oaep \
  -pkeyopt rsa_oaep_md:sha256
```

**Why OAEP padding:**
Raw RSA without padding is vulnerable to chosen-ciphertext attacks. OAEP (Optimal Asymmetric Encryption Padding) adds randomness so encrypting "Top Secret" twice produces different ciphertexts.

![RSA encryption with OAEP](images/rsa-encrypt.png)

**Step 4: Decrypt ciphertext**
```bash
openssl pkeyutl -decrypt -inkey private.pem \
  -in message.enc \
  -pkeyopt rsa_padding_mode:oaep \
  -pkeyopt rsa_oaep_md:sha256
```

Output: Top Secret

![RSA decryption](images/rsa-decrypt.png)

The decryption succeeded because only the private key can reverse the operation performed by the public key.

#### Exercise 22: Diffie-Hellman Key Exchange Simulation

I implemented a Python simulation where Alice and Bob establish a shared secret without transmitting it.

**Public parameters (known to everyone):**
```python
p = 24251  # Large prime
g = 97     # Generator
```

**Private keys (kept secret):**
- Alice chooses: a = 123
- Bob chooses: b = 456

**Public key exchange:**
Alice computes: `A = g^a mod p = 97^123 mod 24251 = 19672`
Bob computes: `B = g^b mod p = 97^456 mod 24251 = 16606`

Alice sends 19672 to Bob (publicly)
Bob sends 16606 to Alice (publicly)

![Public key exchange](images/dh-exchange.png)

**Shared secret derivation:**
Alice computes: `secret = B^a mod p = 16606^123 mod 24251 = 13852`
Bob computes: `secret = A^b mod p = 19672^456 mod 24251 = 13852`

![Shared secret derivation](images/dh-secret.png)

**Both arrived at 13852 without ever transmitting it.**

**Why private keys are never exchanged:**

If Alice sent her private key `a = 123` to Bob, any eavesdropper (Eve) could intercept it and compute the secret herself:
```
Eve sees: g=97, p=24251, a=123, B=16606
Eve computes: secret = B^a mod p = 16606^123 mod 24251 = 13852
```

**The security comes from the discrete logarithm problem:**
Eve sees g=97, p=24251, and A=19672. To find Alice's private key, Eve must solve:
```
97^a ≡ 19672 (mod 24251)
```

No efficient algorithm exists for this, even with modern computers. This is why Eve cannot reverse-engineer the private keys from the public exchange.

**The mathematical magic:**
```
(g^b)^a = g^(b*a) = g^(a*b) = (g^a)^b
```

Both Alice and Bob end up computing g^(ab) mod p, but neither one reveals their exponent to the other.

### Level 5: Digital Signatures and PKI

#### Exercise 23: Tamper Detection with Digital Signatures

I signed a file, modified it, then attempted verification to observe tamper detection.

**Step 1: Create and sign file**
```bash
echo "Hello World" > hello.txt
openssl dgst -sha256 -sign private.pem -out signature.bin hello.txt
```

This computes SHA-256(hello.txt) and encrypts the hash with my private key.

![Sign file with private key](images/sign-create.png)

**Step 2: Verify signature**
```bash
openssl dgst -sha256 -verify public.pem -signature signature.bin hello.txt
```

Output: `Verified OK`

![Successful signature verification](images/sign-verify-ok.png)

**Step 3: Modify file (changed "Hello" to "Bello")**
```bash
nano hello.txt  # Edit content
openssl dgst -sha256 -verify public.pem -signature signature.bin hello.txt
```

Output:
```
Verification Failure
error:1C800064:Provider routines::bad decrypt
```

![Failed verification after tampering](images/sign-verify-fail.png)

**Why verification failed:**

The signature contains the hash of "Hello World" encrypted with the private key. When verifying:
1. Compute SHA-256 of current file (now "Bello World")
2. Decrypt the signature with public key to get original hash
3. Compare: new hash ≠ original hash → FAILURE

Even changing one character breaks the signature because of the avalanche effect in SHA-256.

#### Exercise 24: Self-Signed Certificate Creation

I created an X.509 certificate for "mysite.local" valid for 365 days.

**Command:**
```bash
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem \
  -days 365 -nodes -subj '/CN=mysite.local'
```

**Parameters explained:**
- `-x509`: Create a certificate instead of a signing request
- `-newkey rsa:4096`: Generate 4096-bit RSA key simultaneously
- `-days 365`: Certificate expires in one year
- `-nodes`: Don't encrypt the private key (no passphrase)
- `-subj '/CN=mysite.local'`: Common Name in certificate

![Self-signed certificate creation](images/cert-selfsigned.png)

**Why self-signed certificates aren't trusted by default:**

When you visit `https://mysite.local`, the browser asks: "Who vouches for this certificate?" With a proper certificate, the answer is a trusted Certificate Authority (CA) like DigiCert. With a self-signed certificate, the answer is "The website signed its own certificate."

This creates a trust problem: anyone can create a certificate claiming to be "google.com" and sign it themselves. Browsers reject these by default.

Self-signed certificates work for:
- Internal development servers
- Testing TLS implementations
- Private networks where all clients manually trust the certificate

#### Exercise 25: Trust Chain Analysis

I examined Google's certificate chain to identify the Root CA and Intermediate CA.

**Command:**
```bash
openssl s_client -connect google.com:443 -showcerts
```

![Certificate chain output](images/cert-chain.png)

**Certificate 0 (Leaf Certificate):**
```
Subject: CN=*.google.com
Issuer: CN=WE2, O=Google Trust Services
Valid: Feb 23 2026 - May 18 2026
```

This is Google's actual server certificate. The asterisk (*) allows it to cover www.google.com, mail.google.com, etc.

![Leaf certificate details](images/cert-leaf.png)

**Certificate 1 (Intermediate CA):**
```
Subject: CN=WE2, O=Google Trust Services
Issuer: CN=GTS Root R4, O=Google Trust Services LLC
Valid: Dec 13 2023 - Feb 20 2029
```

WE2 is the intermediate CA that signed Google's server certificate.

![Intermediate CA certificate](images/cert-intermediate.png)

**Certificate 2 (Root CA):**
```
Subject: CN=GTS Root R4, O=Google Trust Services LLC
Issuer: CN=GlobalSign Root CA, O=GlobalSign nv-sa
Valid: Nov 15 2023 - Jan 28 2028
```

GTS Root R4 is Google's root CA, but it's cross-signed by GlobalSign.

![Root CA certificate](images/cert-root.png)

**Why the cross-signature exists:**

Google created their own Root CA (GTS Root R4) relatively recently. Modern devices trust it, but older Android phones or Windows PCs might not have it in their trust store yet.

By having GlobalSign (established since 1996) sign Google's root, the chain works for everyone:
- New devices: Trust GTS Root R4 directly
- Old devices: Trust GlobalSign → which trusts GTS Root R4 → which trusts WE2 → which trusts *.google.com

This creates backward compatibility while transitioning to Google's own PKI infrastructure.

### Level 6: Real-World Protocols and Advanced Topics

#### Exercise 26: Cipher Suite Inspection

I used nmap to enumerate supported cipher suites on google.com and identify weak ones.

**Command:**
```bash
nmap --script ssl-enum-ciphers -p 443 google.com
```

![Nmap cipher suite scan](images/nmap-ciphers.png)

**TLSv1.0 and TLSv1.1 findings:**

Both protocols were found with warning: `64-bit block cipher 3DES vulnerable to SWEET32 attack`

**Example weak ciphers:**
- TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA (ecdh_x25519)
- TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA (ecdh_x25519)
- TLS_RSA_WITH_AES_128_CBC_SHA (rsa 2048)
- TLS_RSA_WITH_3DES_EDE_CBC_SHA (rsa 2048)

![Weak cipher suites](images/weak-ciphers.png)

**Why these are weak:**

**TLSv1.0/1.1:** Both are deprecated as of 2020. They're vulnerable to BEAST and POODLE attacks. Modern browsers disable them by default.

**3DES:** Uses 64-bit blocks instead of 128-bit. The SWEET32 attack can recover plaintext after observing 32GB of encrypted data on the same connection.

**CBC mode without AEAD:** Ciphers like AES-128-CBC-SHA don't include authentication. An attacker can flip bits in ciphertext (padding oracle attacks) and decrypt messages.

**Secure alternatives:**
- TLS 1.3 with AES-256-GCM
- TLS 1.3 with ChaCha20-Poly1305
- TLS 1.2 with ECDHE-RSA-AES256-GCM-SHA384

These use authenticated encryption (GCM/Poly1305) and forward secrecy (ECDHE), preventing both tampering and retroactive decryption.

#### Exercise 27: GPG File Encryption

I generated a GPG key pair, encrypted a file, and decrypted it to verify the workflow.

**Step 1: Generate GPG key**
```bash
gpg --full-generate-key
```

**Configuration:**
- Key type: (9) ECC (sign and encrypt) - default
- Elliptic curve: (1) Curve 25519
- Expiration: 60 days
- User ID: Fatai <fatai@mail>
- Passphrase: [set during generation]

![GPG key generation](images/gpg-gen.png)

**Step 2: Encrypt file**
```bash
gpg --encrypt --recipient "Fatai" Secret.txt
```

This creates Secret.txt.gpg - an encrypted file that only my private key can decrypt.

![GPG file encryption](images/gpg-encrypt.png)

**Step 3: Decrypt file**
```bash
gpg --decrypt Secret.txt.gpg
```

GPG prompts for the passphrase, then outputs the original content.

![Passphrase prompt](images/gpg-passphrase.png)

Output: `Fatai's Secret`

![GPG decryption output](images/gpg-decrypt.png)

**Why GPG uses hybrid encryption:**

GPG doesn't encrypt the file directly with your public key (RSA/ECC encryption is slow for large files). Instead:
1. Generate random 256-bit AES session key
2. Encrypt file with AES-256 (fast)
3. Encrypt the session key with recipient's public key (small, one-time operation)
4. Store both in .gpg file

This combines the speed of symmetric encryption with the key-distribution benefits of asymmetric encryption.

### Level 7: Broken Crypto & Cryptanalysis

#### Exercise 28: Hash Length Extension Attack

I used a Python script to forge a valid hash without knowing the secret key.

**Scenario:**
Given:
- Original hash: `88a5d978cad92b881c91f2d9d299e3a` (MD5 of secret+"user=admin")
- Known data: "user=admin"
- Goal: Generate hash for secret+"user=admin&append=true" without knowing secret

**How the attack works:**

MD5 processes data in 512-bit blocks. When you hash secret+"user=admin":
1. MD5 pads the input to 512-bit boundary
2. Processes it block by block
3. Outputs 128-bit hash

The attack exploits that the final hash becomes the starting state for the next block. By treating the original hash as an intermediate state, we can append data and continue hashing.

![Hash length extension code](images/hashlength-code.png)

**Attack execution:**
```python
# 1. Original hash from server
original_hash = "88a5d978cad92b881c91f2d9d299e3a"

# 2. We append our data
append_data = "&role=root"

# 3. Calculate padding that MD5 added
key_length = 32  # Guessed secret length

# 4. Forge new hash
forged_hash = hashpump.perform_attack(original_hash, "user=admin", append_data, key_length)
```

Output:
```
FORGED HASH: 25eafccdcfdf637a8ebbcbf08ee390d6494d
RAW PAYLOAD: fatai%80%00%00%00%00%00%00%00%00%00%00%28%01%00%00%00%00%00%00%00%00%00%00root
```

![Hash length extension output](images/hashlength-output.png)

**The forged payload includes:**
- Original data: "fatai" (the secret we guessed)
- Padding bytes: %80%00... (null bytes MD5 added)
- Appended data: "root"

If the server uses: `Hash(secret + user_input)` for authentication, we can append arbitrary data without knowing the secret.

**Why HMAC prevents this:**
HMAC uses: `Hash(key + Hash(key + message))`

The key appears twice, and the outer hash prevents extending the inner one.

### Level 8: Future of Cryptography

#### Exercise 29: Zero-Knowledge Proofs Research

**The Ali Baba Cave Analogy:**

Imagine a ring-shaped cave with a single entrance and a secret door halfway around. Only someone who knows the password can open this door.

Peggy (Prover) claims to know the password. Victor (Verifier) wants proof, but Peggy won't reveal the password itself.

**The protocol:**
1. Victor waits outside while Peggy enters the cave and chooses left or right path
2. Victor enters and randomly shouts "Exit from the left!" or "Exit from the right!"
3. If Peggy knows the password, she can open the door to switch sides and always exit from Victor's chosen path
4. Repeat 20 times

**The proof:**
If Peggy guesses correctly once, she has a 50% chance (lucky coin flip). But guessing correctly 20 times in a row has probability (1/2)^20 = 1 in 1,048,576. Victor becomes mathematically certain Peggy knows the password, yet he never learns what it is.

**Why the secret isn't revealed:**
Victor only sees Peggy exiting from random sides. He can't determine whether she used the left path, right path, or the secret door. The protocol proves knowledge without transferring knowledge.

**Real-world applications:**
- Blockchain: Prove you own cryptocurrency without revealing your private key
- Authentication: Log in without transmitting your password
- Privacy: Prove you're over 18 without revealing your birthdate

#### Exercise 30: Post-Quantum Cryptography

The NIST Post-Quantum Cryptography competition was launched to find algorithms that resist quantum computer attacks. Quantum computers can solve the integer factorization problem (breaking RSA) and discrete logarithm problem (breaking Diffie-Hellman and ECC) in polynomial time using Shor's algorithm.

**Three standardized algorithms:**

**CRYSTALS-Kyber (ML-KEM):**
- Purpose: Key encapsulation for encryption
- Security basis: Learning With Errors (LWE) problem on lattices
- Replaces: RSA and ECDH for key exchange
- Status: NIST FIPS 203 standard

**CRYSTALS-Dilithium (ML-DSA):**
- Purpose: Digital signatures
- Security basis: Module Lattice problem
- Replaces: RSA and ECDSA signatures
- Status: NIST FIPS 204 standard

**SPHINCS+ (SLH-DSA):**
- Purpose: Stateless hash-based signatures
- Security basis: Hash function security (conservative fallback)
- Replaces: RSA/ECDSA in applications requiring long-term security
- Status: NIST FIPS 205 standard

**Why the transition is urgent:**
Adversaries are harvesting encrypted data today with the assumption they'll decrypt it later when quantum computers become practical ("Harvest Now, Decrypt Later" attacks). Data encrypted with RSA in 2026 might be decrypted by 2035.

### Mastery Challenge: The Cryptographic Gauntlet

#### Challenge 1: Key Management System

**Storage Architecture:**

Instead of storing RSA-4096 keys in files on disk, production systems use Hardware Security Modules (HSMs) or cloud Key Management Services (KMS).

**HSM approach:**
- Keys generated inside tamper-resistant hardware (FIPS 140-2 Level 3)
- Private keys never leave the device in plaintext
- Encryption/signing operations happen on-chip
- Physical security: device erases keys if opened

**Key Rotation Strategy:**

**90-day automated rotation:**
```
Day 0: Generate key v2, tag as "ACTIVE"
Day 1-90: v1 marked "DECOMMISSIONING" (still decrypts old data)
Day 90: v2 becomes primary, v1 marked "ARCHIVED"
Day 120: v1 deleted after 30-day grace period
```

**Why rotation matters:**
If an attacker compromises v1 on day 89, they can only decrypt messages from the past 89 days. Messages encrypted with v2 (day 90+) remain secure.

**Version tagging:**
Every encrypted file includes a header: `KEY_VERSION=v2`. During decryption, the system selects the correct key automatically.

#### Challenge 2: Secure Channel Implementation

I implemented a Python simulation of the TLS handshake process.

**Protocol steps:**

**Step 1: Key Exchange**
- Server generates SECP384R1 private/public key
- Client generates SECP384R1 private/public key
- Both exchange public keys

![TLS key exchange](images/tls-exchange.png)

**Step 2: Shared Secret Derivation**
- Server: `shared = client_public ^ server_private`
- Client: `shared = server_public ^ client_private`
- Both derive identical 384-bit shared secret

Result: Both parties established the same shared secret

![Shared secret derivation](images/tls-shared.png)

**Step 3: Session Key Derivation**
- Use HKDF (HMAC-based Key Derivation Function)
- Input: shared secret
- Output: 256-bit AES session key

Session key: `97afb712cb037fb072...`

![Session key derivation](images/tls-session.png)

**Step 4: Message Encryption**
- Encrypt "Top Secret Message" using AES-256-GCM
- Nonce: randomly generated per message
- Authentication tag: appended to ciphertext

![Message encryption](images/tls-encrypt.png)

**Step 5: Audit Trail**
- Compute SHA3-256 hash of ciphertext
- Sign hash with server's private key
- Store: ciphertext + signature + hash

This creates a tamper-evident record. If anyone modifies the ciphertext, the signature verification fails.

![Audit trail creation](images/tls-audit.png)

**Step 6: Decryption and Verification**
- Verify signature using server's public key
- Decrypt ciphertext with session key
- Authentication tag automatically verified by GCM

Output: `Top Secret Message`

![Message decryption](images/tls-decrypt.png)

**Why this mimics TLS:**
1. Ephemeral key exchange (ECDH) provides forward secrecy
2. Session key derived from shared secret (not transmitted)
3. Authenticated encryption (GCM) prevents tampering
4. Digital signature creates audit trail

If the server's long-term private key is compromised tomorrow, messages encrypted today remain secure because the session keys were ephemeral.

#### Challenge 3: Quantum Readiness Migration

**Phase 1: Hybrid Key Exchange**

Implement dual encryption:
```
Classical: ECDH-P384 → shared_secret_1
Post-quantum: ML-KEM-768 → shared_secret_2
Combined: HKDF(shared_secret_1 || shared_secret_2)
```

If a quantum computer breaks ECDH, ML-KEM still protects the session. If ML-KEM has an undiscovered weakness, ECDH still provides security.

**Phase 2: Signature Algorithm Transition**

Replace RSA signatures in audit logs:
```
Old: SHA-256 + RSA-4096
New: SHA3-256 + ML-DSA-87 (Dilithium)
```

Documents signed in 2026 will remain verifiable in 2040 even if quantum computers can break RSA.

**Phase 3: Crypto-Agility**

Update banking software to recognize Object Identifiers (OIDs) for PQC algorithms:
```
TLS: {1.3.6.1.4.1.2.267.7.4.4} = ML-KEM-512
Signatures: {1.3.6.1.4.1.2.267.7.6.5} = ML-DSA-65
```

This allows swapping algorithms instantly if NIST identifies a weakness in any PQC candidate.

**Migration timeline:**
- 2026: Deploy hybrid mode in production
- 2027: Begin signing new documents with ML-DSA
- 2028: Deprecate RSA/ECDSA for new connections
- 2029: Full transition to post-quantum only

## Findings

**Classical Cryptography:**
- Caesar cipher brute force succeeds because keyspace is only 25 possibilities
- Vigenère provides stronger security than Caesar but manual decryption remains feasible with key length known
- Both ciphers fall to frequency analysis when key length unknown

**Symmetric Encryption:**
- ECB mode leaks information through identical ciphertext blocks for identical plaintext
- CBC mode eliminates patterns but wrong passwords create partial output before failure
- GCM mode adds authentication, preventing silent corruption from wrong passwords
- Password-based encryption requires PBKDF2 with high iteration counts to resist brute force

**Hashing:**
- MD5 produces 128-bit hashes, SHA-256 produces 256-bit hashes
- Avalanche effect causes 50% of hash bits to change from single-character input modifications
- bcrypt's intentional slowness (rounds=12) makes password cracking impractical
- Rainbow tables trade computation for storage but salting defeats them completely
- MD5 collisions exist in practice, destroying its integrity guarantees

**Asymmetric Cryptography:**
- RSA encryption works but is slow - production systems use it only for key exchange
- Diffie-Hellman allows shared secret derivation without transmitting the secret
- Digital signatures detect tampering but provide no confidentiality
- Self-signed certificates work technically but browsers reject them for trust reasons

**PKI:**
- Certificate chains require multiple signatures to establish trust
- Cross-signing allows new CAs to work with old devices
- Weak cipher suites (TLS 1.0/1.1, 3DES) remain enabled for backward compatibility but create vulnerabilities

**Advanced Topics:**
- Hash length extension attacks work on MD5/SHA-1/SHA-256 but HMAC prevents them
- Zero-knowledge proofs enable authentication without revealing secrets
- Post-quantum algorithms replace RSA/ECC to resist quantum computer attacks

## Challenges Faced

**AES decryption with wrong password:**
OpenSSL's behavior surprised me - it created the output file and wrote partial plaintext before detecting the error. I expected immediate failure. This taught me that padding validation happens at the final block, not continuously.

**bcrypt rounds parameter:**
Initially used rounds=4 (16 iterations), which ran instantly. Increasing to rounds=12 (4096 iterations) slowed each hash to 200ms. I learned that security requires intentional computational cost.

**RSA manual calculation:**
Computing 26^7 mod 33 by hand was tedious. Breaking it into smaller exponentiations (26^2, 26^4, then combining) made it manageable. This showed why computers handle these operations instead of humans.

**Hash length extension attack:**
Understanding the padding mechanism took multiple reads of the RFC. The attack works because MD5's internal state becomes exposed as the final hash, allowing continuation. HMAC prevents this by hashing twice with the key.

**GPG passphrase lockout:**
Entered the wrong passphrase three times and GPG wouldn't decrypt. Had to regenerate the key pair because I forgot the passphrase I set during generation. This highlighted the importance of password management.

## Key Takeaways

- **Mode selection matters more than algorithm strength:** AES-256-ECB is weaker than AES-128-GCM because ECB leaks patterns while GCM provides authentication.

- **Hashing ≠ Encryption:** Hashes are one-way functions for integrity, not confidentiality. MD5 collisions make it unsuitable for any security application.

- **Key derivation requires work:** Raw passwords are weak. PBKDF2 with 100,000 iterations transforms them into strong encryption keys while resisting brute force.

- **Forward secrecy is non-negotiable:** Using ephemeral Diffie-Hellman keys means past sessions remain secure even if long-term keys are compromised later.

- **Quantum computers break current cryptography:** RSA, ECDH, and ECDSA will become insecure within 10-15 years. Transitioning to lattice-based algorithms now prevents "harvest now, decrypt later" attacks.

- **Authentication prevents tampering:** AES-CBC without HMAC allows bit-flipping attacks. GCM's built-in authentication tag makes this impossible.

- **Salting defeats precomputation:** Rainbow tables require terabytes of storage for unsalted hashes but become useless when each password has a unique random salt.

- **Digital signatures prove integrity, not secrecy:** A signed document proves it hasn't been modified and came from the claimed sender, but anyone can read it.

## Disclaimer

This lab was performed in a controlled Kali Linux environment for educational purposes as part of the ICDFA Cryptography course. All cryptographic operations were conducted on local systems with no real-world data. Weak algorithms (MD5, 3DES) were analyzed to understand their vulnerabilities, not to implement them in production systems.