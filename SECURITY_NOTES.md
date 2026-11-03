# Security Architecture & Cryptographic Threat Model
## Exam Security and Student Support System

---

## 1. Threat Model & Mitigations

| Threat Vector | Attack Scenario | System Mitigation & Defense |
|---|---|---|
| **T1: Pre-Exam Question Paper Leakage** | An attacker gains read access to the database or filesystem days before the exam. | **AES-256-GCM Envelope Encryption:** Plaintext papers are never stored in the database or written to disk. The paper is encrypted with a fresh random 256-bit AES data key. The data key is wrapped under an application master key kept in a PKCS12 KeyStore. |
| **T2: Local Client Clock Manipulation** | A student adjusts their workstation's system clock back or forward to force paper release or extend duration. | **Server Clock Authority:** Time-lock checks never consult the local machine time (`Instant.now()`). The system queries the MySQL server's monotonic clock using `SELECT UTC_TIMESTAMP(6)`. Manipulating the student's PC clock has zero effect on paper accessibility. |
| **T3: Database Tampering by Malicious DBA** | A database administrator or attacker with SQL access modifies exam answers, changes student scores, or deletes audit logs. | **SHA-256 Hash-Chained Integrity Ledger:** Every event is cryptographically linked to the previous row's hash. Any modification to a past row breaks all subsequent hashes (`MODIFIED_RECORD` / `BROKEN_LINK`). External anchor exports detect tail truncation. |
| **T4: Credential Brute-Force & Rainbow Tables** | An attacker attempts automated dictionary attacks or rainbow table lookups against user passwords. | **PBKDF2WithHmacSHA256:** Passwords are hashed using 210,000 iterations and a 16-byte cryptographically secure random salt. Comparisons use `MessageDigest.isEqual` to prevent timing attacks. Accounts lock for 5 minutes after 5 consecutive failed attempts. |
| **T5: Duplicate Answer Resubmission** | A student attempts to submit multiple times or modify their submission after leaving the exam room. | **Single-Submission Enforcement:** Database table enforces a composite `UNIQUE(exam_id, student_id)` constraint. The service layer verifies that no submission exists before writing, within an atomic transaction. |
| **T6: Privacy Invasion of Student Support Messages** | An unauthorized staff member or database eavesdropper reads private student support requests or mental health concerns. | **Context-Bound AES-256-GCM Encryption:** Support messages are encrypted before saving, with AAD bound to the student ID (`"GRIEVANCE_" + studentId`). Plaintext messages never appear in database columns, logs, the ledger, or reports. Decryption requires authorized staff credentials. |
| **T7: Algorithmic Stigmatization / Student Panic** | An automated keyword scanner alerts the student that they were flagged as "at risk" or blocks them from taking their exam. | **Ethical Support Philosophy:** Keyword scanning operates purely as a routing mechanism for human review. Students are **never** informed their message was flagged, and examination access is **never** blocked. |

---

## 2. Cryptographic Justifications

### 2.1 Why AES-256-GCM (Galois/Counter Mode)?
Traditional AES modes like CBC (Cipher Block Chaining) or ECB (Electronic Codebook) are vulnerable to serious attacks:
- **ECB Mode:** Does not hide data patterns.
- **CBC Mode:** Requires padding (susceptible to padding oracle attacks) and does **not** provide data integrity or authenticity. An attacker can flip bits in the ciphertext to predictably alter plaintext.
- **AES-GCM Solution:** GCM is an Authenticated Encryption with Associated Data (**AEAD**) algorithm. It computes a 128-bit cryptographic authentication tag over the ciphertext and Additional Authenticated Data (AAD). If an attacker alters even a single bit of the ciphertext or AAD, decryption immediately fails, preventing any partial or corrupt plaintext from leaking.

### 2.2 Why PBKDF2WithHmacSHA256 with 210,000 Iterations?
- Standard cryptographic hash functions (such as MD5, SHA-1, and basic SHA-256) are designed to be fast, making them vulnerable to modern GPU-based brute-force attacks (billions of hashes per second).
- **PBKDF2 (Password-Based Key Derivation Function 2):** Applies a pseudorandom function (HMAC-SHA256) repeatedly. NIST guidelines recommend >= 210,000 iterations for PBKDF2 with SHA-256. This imposes a significant computational cost on attackers while taking < 100 milliseconds on a standard desktop during normal user login.
- **Salt:** A fresh 16-byte random salt generated from `java.security.SecureRandom` guarantees that identical passwords generate completely different hashes, neutralizing pre-computed rainbow table attacks.

### 2.3 Why Canonical Delimiter `\u001F` in Ledger Hashes?
In hash-chaining systems, standard delimiters like commas `,` or colons `:` can lead to **canonicalization / field injection attacks**:
- For example, if fields `"Alice"` and `"Staff,Creator"` are concatenated as `Alice:Staff,Creator`, an attacker might shift boundaries to forge an identical hash.
- To prevent this, our ledger formula uses the ASCII **Unit Separator (`\u001F`)**, an unprintable control character designed specifically for separating fields in structured data streams. It cannot naturally appear in human-entered text strings.

### 2.4 Mathematical Solution to Tail Deletion (Anchor Verification)
A common weakness in hash-chained databases is that while modifying an intermediate record breaks the chain, **deleting the last $K$ records** from the tail of the table leaves the remaining chain mathematically valid!
- **Mitigation:** The system provides cryptographic anchor export (`LedgerService.exportAnchor()`).
- The anchor contains the tuple $(\text{seq\_no}, \text{curr\_hash}, \text{timestamp})$.
- When `LedgerService.verify(anchorFile)` is invoked, it verifies that the database contains at least as many records as the anchor file and that the hash at that sequence number matches the anchor. If a malicious DBA deleted rows, a `MISSING_RECORD` alert is triggered immediately.

---

## 3. Ethical Student Support Design Principles

1. **Non-Diagnostic:** The keyword scanner does not attempt to diagnose mental health conditions, calculate "risk scores," or replace professional medical or counseling staff. It simply identifies keywords indicating distress or accommodation needs.
2. **Non-Punitive:** Examination access and student academic standing are completely decoupled from support requests. Under no circumstances does a grievance submission restrict access to taking an exam or viewing grades.
3. **Confidentiality by Default:** No administrator, professor, or proctor can browse private support messages in bulk. Each message is decrypted on-the-fly only when an authorized staff member opens the review dialog.
4. **Transparent Accountability:** All staff reviews and status changes are audited to the SHA-256 integrity ledger, creating a permanent, tamper-evident record of faculty responsiveness to student wellness needs.
