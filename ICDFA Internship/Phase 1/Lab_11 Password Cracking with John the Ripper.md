# Security Portfolio: Password Cracking with John the Ripper
---

## 1. Engagement Overview

A controlled password cracking assessment was conducted in a local Kali Linux
environment using John the Ripper v1.9.0-Jumbo-1. The lab covered six cracking
techniques across multiple hash types: raw MD5, raw SHA1, NTLM, and
Yescrypt. Each technique was tested against purpose-built hash files containing
known and custom password values. The assessment progressed from basic
wordlist attacks through to rule-based mutations and brute force, with an
additional exercise generating and cracking a custom MD5 hash using Python.

---

## 2. Objectives

- Identify hash types using John the Ripper's format detection and prefix
  recognition
- Crack raw MD5 and SHA1 hashes using the rockyou.txt wordlist and a custom
  wordlist
- Perform incremental brute force against MD5 hashes and measure practical
  speed
- Crack NTLM hashes representative of Windows credential storage
- Apply rule-based mutations to a wordlist to crack case-modified passwords
- Generate a custom MD5 hash with Python and confirm it is crackable with a
  standard wordlist
- Assess the practical crackability of passwords and document the conditions
  under which each technique succeeds or fails

---

## 3. Scope

**In-Scope:**
- Hash files created and controlled within the local lab environment
- Hash types tested: Raw-MD5, Raw-SHA1, NT (NTLM), Yescrypt
- Wordlists: `/usr/share/wordlists/rockyou.txt`, `custom_wordlist.txt`,
  `wordlist.txt`
- Cracking modes: wordlist, incremental (brute force), rule-based (ShiftToggle)

**Out-of-Scope:**
- Live system account cracking
- Any hash files from real users or production systems
- Network-based authentication attacks

**Authorization Statement:**
> All hash files tested in this assessment were created within an isolated local
> Kali Linux lab environment for educational purposes. No real user credentials,
> live systems, or unauthorized infrastructure were accessed at any point. All
> activities were conducted under authorized course instruction.

---

## 4. Methodology

### Phase 1 — Hash Identification
John the Ripper was used to identify hash types both by format prefix (`$1$`
for MD5Crypt, `$y$` for Yescrypt, `$6$` for SHA-512) and by raw hash length
for unsalted MD5 and SHA1 values. Yescrypt was noted as non-auto-detectable
in raw format due to its mathematical construction.

### Phase 2 — Wordlist Attack (rockyou.txt)
A raw MD5 hash file (`hash.txt`) was targeted using:
```

john --wordlist=/usr/share/wordlists/rockyou.txt --format=raw-md5 hash.txt
```
The rockyou.txt wordlist contains over 14 million entries sourced from real
credential breaches. The attack completed in approximately 3 seconds.

### Phase 3 — Custom Wordlist Attack
A custom wordlist (`custom_wordlist.txt`) containing 7 targeted entries was run
against `hash.txt` using the same raw-md5 format. This tested whether a small
but well-targeted list could crack multiple hashes without a full dictionary scan.

### Phase 4 — Brute Force (Incremental Mode)
Incremental mode was run against a 4-hash MD5 file using:
```
john --incremental --format=raw-md5 hash.txt
```
To keep the brute force within a practical time window, the hash file was
pre-populated with hashes of short, common values (`1234`, `123`, `abcd`).
The attack completed in 4 seconds and cracked 3 hashes.

### Phase 5 — NTLM Hash Cracking
A Windows NTLM hash stored in `win_hash.txt` was attacked using:
```

john --format=nt --wordlist=/usr/share/wordlists/rockyou.txt win_hash.txt
```
NTLM is the hash format used by Windows to store local account credentials.

### Phase 6 — Rule-Based Attack
A rule-based attack using John's built-in `ShiftToggle` ruleset was applied
against `hash.txt` with `wordlist.txt` as the base:
```
john --wordlist=wordlist.txt --rules=ShiftToggle --format=raw-md5 hash.txt
```
ShiftToggle generates case-alternated mutations of every wordlist entry,
targeting passwords where users have manually modified capitalisation.

### Phase 7 — Custom Hash Generation and Cracking
A raw MD5 hash of the word `mypassword` was generated using Python:
```bash
python3 -c "import hashlib; print(hashlib.md5(b'mypassword').hexdigest())" \
>> my_hash.txt
```
The resulting hash was then cracked with rockyou.txt to confirm the full
workflow from hash generation to recovery.

---

## 5. Vulnerability Summary

> **Note:** This lab focuses on technique demonstration rather than live system
> assessment. The table below documents the hash weakness findings rather than
> application vulnerabilities.

| ID | Finding | Severity | Hash Type / Context |
|----|---------|----------|---------------------|
| 01 | Common Password Cracked via Wordlist (raw-md5) | High | Raw-MD5 / hash.txt |
| 02 | Multiple Passwords Cracked via Custom Wordlist | High | Raw-MD5 / hash.txt |
| 03 | Short Passwords Cracked via Brute Force in 4s | High | Raw-MD5 / hash.txt |
| 04 | NTLM Hash Cracked via Wordlist (Password123) | High | NTLM / win_hash.txt |
| 05 | Case-Modified Passwords Cracked via Rules | Medium | Raw-MD5 / hash.txt |
| 06 | Custom MD5 Hash of Common Word Cracked in 0s | High | Raw-MD5 / my_hash.txt |

---

## 6. Detailed Findings

---

### Finding 01 — Common Password Cracked via Wordlist Attack

#### Severity
High

#### Affected Hash Type
Raw-MD5 (`hash.txt`)

#### Description
A raw MD5 hash was cracked using the rockyou.txt wordlist in under 3 seconds.
MD5 is an unsalted, fast hashing algorithm not designed for password storage.
Its speed makes it practical for high-throughput wordlist attacks — the session
reached a sustained rate of approximately 3,622 kilo-candidates per second. The
recovered password was a single common dictionary word, which appeared
directly in rockyou.txt without any mutation needed.

#### Proof of Concept

John the Ripper command and terminal output confirming successful crack:



![John Wordlist Attack - rockyou.txt Against Raw-MD5 Hash](image.jpg)



Command used:

```
john --wordlist=/usr/share/wordlists/rockyou.txt --format=raw-md5 hash.txt
```

Output:
```
Loaded 2 password hashes with no different salts (Raw-MD5 [MD5 256/256 AVX2 8x3])
password      (?)
1g 0:00:00:03 DONE 0.2525g/s 3622Kp/s 3622Kc/s 3622KC/s fuckyooh21..*7¡Vamos!
Session completed.
```

#### Impact
Any MD5-hashed password that appears in rockyou.txt is recoverable in seconds
on standard consumer hardware. In a real credential database where passwords
are stored as unsalted MD5 values, a single SQL injection or database exposure
incident would result in full plaintext recovery for the majority of user accounts
within minutes.

#### Remediation
- Do not use MD5 for password hashing in any context, including legacy systems
- Replace MD5 with bcrypt (minimum cost 12), Argon2id, or scrypt for all
  password storage
- Apply a per-user random salt before hashing to defeat precomputed attacks
  even when a stronger algorithm is not immediately available

---

### Finding 02 — Multiple Passwords Cracked via Custom Wordlist

#### Severity
High

#### Affected Hash Type
Raw-MD5 (`hash.txt`)

#### Description
A custom wordlist containing 7 targeted entries cracked 4 of 7 loaded hashes
immediately, completing in under 1 second. Cracked passwords included
`password`, `fatai`, `james`, and `icdfa`. This demonstrates that an attacker
with any knowledge of the target organisation, usernames, or common naming
conventions can build a small, efficient wordlist that outperforms even large
dictionaries when the password space is predictable.

#### Proof of Concept

John the Ripper run with custom wordlist against the same hash file:



![John Custom Wordlist Attack - 4 of 7 Hashes Cracked](image.jpg)



Command used:
```
john --wordlist=custom_wordlist.txt --format=raw-md5 hash.txt
```

Output:
```
Loaded 7 password hashes with no different salts (Raw-MD5 [MD5 256/256 AVX2 8x3])
password      (?)
fatai         (?)
james         (?)
icdfa         (?)
4g 0:00:00:00 DONE 57.14g/s 57.14p/s 57.14c/s 400.0C/s password..icdfa
Session completed.
```

#### Impact
Organisation-specific wordlists built from company names, product names,
employee usernames, and public information are highly effective against
passwords chosen by staff. Four targeted entries were sufficient to crack 4 out
of 7 hashes. A real-world attacker building a list from LinkedIn profiles, company
documentation, and previous breach data would achieve far higher coverage.

#### Remediation
- Prohibit passwords that contain the company name, product names, or any
  string that appears in the organisation's public presence
- Enforce a minimum password length of 14 characters with complexity
  requirements that make dictionary entries ineffective as base values
- Use have-I-been-pwned-style blocklists to reject passwords known to appear
  in public wordlists at account creation time

---

### Finding 03 — Short Passwords Cracked via Brute Force in 4 Seconds

#### Severity
High

#### Affected Hash Type
Raw-MD5 (`hash.txt`)

#### Description
Incremental brute force mode was run against an MD5 hash file containing
hashes of short, common numeric and alphabetic values. Three hashes were
cracked in 4 seconds at a sustained rate of approximately 139,432 candidates
per second. Cracked values were `1234`, `123`, and `abcd`. The speed
confirmed that passwords of 4 characters or fewer are recoverable in seconds
by brute force against unsalted MD5, regardless of whether the value appears
in any wordlist.

#### Proof of Concept

John the Ripper incremental mode output:



![John Incremental Brute Force - 3 Hashes Cracked in 4 Seconds](image.jpg)



Command used:
```
john --incremental --format=raw-md5 hash.txt
```

Output:
```
Loaded 4 password hashes with no different salts (Raw-MD5 [MD5 256/256 AVX2 8x3])
Remaining 3 password hashes with no different salts
1234      (?)
123       (?)
abcd      (?)
3g 0:00:00:04 DONE 0.6048g/s 139432p/s 139432c/s 141445C/s all$..amyg
Session completed.
```

#### Impact
Brute force against MD5 is theoretically 100% successful given enough time.
Against short passwords of 4 characters or fewer, it is practically instantaneous
on standard hardware. This means any 4-character or shorter password stored
as MD5 provides no meaningful protection — the hash is effectively equivalent
to storing the password in plaintext for an attacker with the hash file.

#### Remediation
- Enforce a minimum password length of at least 12 characters, preferably 16
  or more, at the application layer
- The length increase from 8 to 16 characters increases the brute force search
  space by a factor that makes the attack computationally impractical, even
  against MD5
- Combine minimum length enforcement with a memory-hard hash function
  to multiply the time cost per candidate and reduce throughput to hundreds
  of hashes per second rather than hundreds of millions

---

### Finding 04 — NTLM Hash Cracked via Wordlist (Password123)

#### Severity
High

#### Affected Hash Type
NT / NTLM (`win_hash.txt`)

#### Description
A Windows NTLM hash was cracked using rockyou.txt in under 1 second,
recovering the password `Password123`. NTLM is the native password hashing
format used by Windows for local account credential storage. Like MD5, NTLM
is fast and unsalted, making it highly vulnerable to wordlist attacks. The
recovered password used a capital first letter and appended digits, a pattern
often perceived by users as providing meaningful complexity. The password was
present in rockyou.txt because `password123` with a capital P is a common
variation documented in real breach data.

#### Proof of Concept

John the Ripper NTLM crack against win_hash.txt:



![John NTLM Wordlist Attack - Password123 Recovered](image.jpg)

Command used:
```
john --format=nt --wordlist=/usr/share/wordlists/rockyou.txt win_hash.txt
```

Output:
```
Loaded 1 password hash (NT [MD4 256/256 AVX2 8x3])
Password123    (?)
1g 0:00:00:00 DONE 25.00g/s 840000p/s 840000c/s 840000C/s coco21..181193
Session completed.
```

#### Impact
NTLM hashes are extracted routinely from Windows systems via tools such as
Mimikatz, the SAM registry hive, or pass-the-hash techniques during
post-exploitation. Once extracted, a hash like this is cracked before the incident
response team is even notified. `Password123` satisfies many organisations'
complexity policies (uppercase, lowercase, digits) while being trivially crackable,
demonstrating that complexity policies without minimum entropy requirements
provide a false sense of security.

#### Remediation

- Replace NTLM authentication with NTLMv2 or Kerberos where possible,
  and disable NTLM entirely in environments that support modern protocols
- Enforce passphrases of 16 or more characters rather than relying on
  character-class complexity rules
- Block known common passwords including `Password123` and its variants
  using a deny-list at account creation and password change

---

### Finding 05 — Case-Modified Passwords Cracked via Rule-Based Attack

#### Severity
Medium

#### Affected Hash Type
Raw-MD5 (`hash.txt`)

#### Description
The `ShiftToggle` rule set was applied to `wordlist.txt`, generating alternating-
case mutations of each base word. Two hashes were cracked immediately:
`FaTaI` and `PassWord`. Both values are case-alternated versions of base words
(`fatai` and `password`) that would not appear verbatim in a standard wordlist.
Without rule-based mutation, these hashes would have survived a plain wordlist
attack. This demonstrates that user-applied capitalisation transforms perceived
as adding complexity do not meaningfully increase resistance against rule-based
cracking.

#### Proof of Concept

John the Ripper rule-based attack using ShiftToggle:



![John Rule-Based Attack - ShiftToggle Cracking FaTaI and PassWord](image.jpg)



Command used:
```
john --wordlist=wordlist.txt --rules=ShiftToggle --format=raw-md5 hash.txt
```

Output:
```
Loaded 2 password hashes with no different salts (Raw-MD5 [MD5 256/256 AVX2 8x3])
FaTaI       (?)
PassWord    (?)
2g 0:00:00:01 DONE 1.754g/s 253.5p/s 253.5c/s 507.0C/s password..PASSWORD
Session completed.
```

#### Impact
Rule-based attacks systematically apply every known human capitalisation
pattern to a wordlist. Any password that is a transformed version of a dictionary
word — regardless of how unusual the capitalisation pattern appears — is
vulnerable. Common rules cover `l33t` substitutions, digit appending, reversed
strings, and mixed-case alternation. A complete ruleset attack recovers the
majority of passwords that pass standard complexity requirements.

#### Remediation
- A 16-character passphrase of random words provides stronger resistance
  than a short word with complex capitalisation, because it increases the base
  entropy rather than applying predictable transforms to a low-entropy root
- Advise users that capitalisation patterns like `CamelCase` and alternating
  case do not add meaningful security against rule-based attacks
- Implement account lockout and alerting at the authentication layer to detect
  online rule-based guessing before a hash file exposure is required

---

### Finding 06 — Custom MD5 Hash of Common Word Cracked in 0 Seconds

#### Severity
High

#### Affected Hash Type
Raw-MD5 (`my_hash.txt`)

#### Description
A custom MD5 hash was generated directly from the string `mypassword` using
Python's `hashlib` module and saved to a file. John the Ripper cracked the hash
against rockyou.txt in under 1 second, returning `mypassword`. This exercise
confirmed the complete workflow from hash generation to recovery and
demonstrated that the word `mypassword` is present in rockyou.txt.

The custom hash value generated:
```
34819d7beeabb9260a5c854bc85b3e44
```

#### Proof of Concept

Python hash generation command:
```bash
python3 -c "import hashlib; print(hashlib.md5(b'mypassword').hexdigest())" \
>> my_hash.txt
```

Hash file contents confirmed via `cat`:
```
34819d7beeabb9260a5c854bc85b3e44
34819d7beeabb9260a5c854bc85b3e44
```



![Python MD5 Hash Generation and cat Output](image.jpg)



John cracking the custom hash:



![John Cracking Custom MD5 Hash - mypassword Recovered in Under 1 Second](image.jpg)



Command used:
```
john --format=raw-md5 --wordlist=/usr/share/wordlists/rockyou.txt my_hash.txt
```

Output:
```
Loaded 1 password hash (Raw-MD5 [MD5 256/256 AVX2 8x3])
mypassword    (?)
1g 0:00:00:00 DONE 11.11g/s 25600p/s 25600c/s 25600C/s amore..abcdefgh
Session completed.
```

#### Impact
This exercise confirms that generating an MD5 hash of a password does not
protect it in any meaningful way if the underlying word is common. The hash
`34819d7beeabb9260a5c854bc85b3e44` looks opaque, but it maps directly and
immediately to `mypassword` via rockyou.txt. Any developer or system
administrator who stores user passwords as plain MD5 values is effectively
storing crackable plaintext.

#### Remediation
- Never use Python's `hashlib.md5()` or `hashlib.sha256()` for password
  storage. These are general-purpose cryptographic hash functions designed
  for speed, which is the opposite of what password hashing requires.
- Use Python's `bcrypt` library or `argon2-cffi` for password hashing:
```python
import bcrypt
hash = bcrypt.hashpw(b'mypassword', bcrypt.gensalt(rounds=12))
```

- The bcrypt output includes a per-password salt automatically, making each
  hash unique even for identical passwords

---

## 7. Tools Used

- Kali Linux (lab environment)
- John the Ripper v1.9.0-Jumbo-1+git20211102-0kali10
- rockyou.txt (`/usr/share/wordlists/rockyou.txt`)
- Python 3 (`hashlib` module for custom hash generation)

---

## 8. Challenges Encountered

- **Yescrypt auto-detection not possible in raw format:** John the Ripper cannot
  auto-detect Yescrypt hashes when provided in raw format because of its
  mathematical construction. The `--format` flag must be specified explicitly for
  Yescrypt targets, unlike MD5 and SHA1 which can be identified by hash length.
- **Brute force time management:** Running incremental mode against longer or
  more complex passwords would have required hours or days. The hash file was
  pre-populated with short, common values to produce a result within the lab
  session. In a real engagement, brute force is reserved for short passwords only
  and combined with targeted wordlist and rule-based attacks for longer ones.
- **OpenMP warning on all sessions:** Every John session produced the warning
  `no OpenMP support for this hash type, consider --fork=2`. This is a
  performance notice, not an error. Using `--fork=2` would split the work across
  CPU cores for faster throughput on supported hash types.

---

## 9. Key Takeaways

- **Hash algorithm selection determines crackability more than password choice
  does:** A common password stored with bcrypt and cost factor 12 takes
  significantly longer to crack than a complex password stored as MD5. The
  algorithm is the first line of defence, not the password itself.
- **rockyou.txt covers more ground than expected:** `Password123`, `mypassword`,
  and single common words were all present in rockyou.txt without any mutation.
  This wordlist contains over 14 million entries from real breach data, meaning
  any password a user thinks is private has likely been used before.
- **Rules close the gap between wordlists and complex passwords:** `FaTaI` and
  `PassWord` would survive a plain wordlist attack but fell immediately to
  ShiftToggle. Organisations that consider capitalisation a meaningful complexity
  requirement are overestimating the protection it provides.
- **Salts defeat precomputed attacks but not targeted cracking:** If the accounts
  table from Finding 05 of the previous SQL injection lab had used salted hashes
  instead of plaintext, rainbow tables would have been useless. However, John
  would still crack individual hashes given the hash file — salts slow attackers
  down but do not stop them if the underlying password is weak.
- **Password length is the most durable defence:** Adding characters to a
  password increases the search space exponentially. A 4-character password
  is brute-forceable in seconds. A 16-character passphrase of random words
  would take longer than the age of the universe to brute force against bcrypt,
  regardless of the wordlist available.

---

## 10. Disclaimer

> This assessment was conducted entirely within an isolated local Kali Linux
> lab environment using hash files created specifically for this exercise. No
> credentials from real users, live systems, or production databases were
> accessed, stored, or cracked at any point. John the Ripper and all associated
> wordlists were used for **educational purposes only**. Password cracking
> techniques demonstrated in this report must only be applied against systems
> and credentials for which explicit written authorization has been obtained.

---