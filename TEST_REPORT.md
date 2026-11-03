# Test Execution Report
## Exam Security and Student Support System

**Report Date:** 2026-10-19  
**Build Status:** **BUILD SUCCESS**  
**Total Tests:** 40  
**Failures:** 0  
**Errors:** 0  
**Skipped:** 0  
**Success Rate:** **100%**  
**Execution Environment:** Windows 11, Oracle OpenJDK 25.0.3 (Target: Java 17 bytecode release), Apache Maven 3.9.9

---

### 1. Test Suite Summary

| Test Class | Package | Tests Run | Failures | Errors | Time (s) | Status |
|---|---|:---:|:---:|:---:|:---:|:---:|
| `CryptoServiceTest` | `com.examsecurity.security` | 6 | 0 | 0 | 0.550s | **PASSED** |
| `PasswordHasherTest` | `com.examsecurity.security` | 3 | 0 | 0 | 3.090s | **PASSED** |
| `SessionAndAccessGuardTest` | `com.examsecurity.security` | 2 | 0 | 0 | 0.064s | **PASSED** |
| `GrievanceWorkflowTest` | `com.examsecurity.service` | 2 | 0 | 0 | 0.553s | **PASSED** |
| `KeywordScannerTest` | `com.examsecurity.service` | 5 | 0 | 0 | 0.055s | **PASSED** |
| `LedgerServiceTest` | `com.examsecurity.service` | 6 | 0 | 0 | 0.248s | **PASSED** |
| `PaperParserTest` | `com.examsecurity.service` | 4 | 0 | 0 | 0.072s | **PASSED** |
| `ReportServiceTest` | `com.examsecurity.service` | 1 | 0 | 0 | 0.027s | **PASSED** |
| `TimeLockServiceTest` | `com.examsecurity.service` | 5 | 0 | 0 | 0.074s | **PASSED** |
| `ValidatorTest` | `com.examsecurity.util` | 5 | 0 | 0 | 0.113s | **PASSED** |
| `SystemIntegrationTest` | `com.examsecurity` | 1 | 0 | 0 | 0.901s | **PASSED** |
| **TOTAL** | | **40** | **0** | **0** | **5.747s** | **100% PASS** |

---

### 2. Critical Security & Requirement Verifications

#### A. Cryptography & Key Management (REQ-20, REQ-21, REQ-28)
- **AES-256-GCM Encryption & Decryption:** Validated 12-byte random IV generation and 128-bit GCM authentication tags.
- **AAD Context Binding:** Tested context tampering (altering AAD) correctly results in complete decryption rejection (`CryptoException`).
- **Ciphertext Tampering:** 1-bit alteration of ciphertext results in GCM authentication failure; zero partial plaintext is leaked.
- **Key Wrapping & Unwrapping:** Data key wrapped with AES-GCM under Master Key using `"KEY_WRAP_V1"` AAD.
- **Memory Wiping:** Verified `CryptoService.wipeBytes()` zeroes sensitive byte arrays immediately after use.

#### B. Credential Security & Lockout (REQ-5, REQ-6)
- **PBKDF2WithHmacSHA256:** Verified 210,000 iterations and 16-byte cryptographically secure random salts.
- **Constant-Time Comparison:** Evaluated using `MessageDigest.isEqual` to prevent timing attacks.
- **Account Lockout:** Verified 5 failed login attempts result in a 5-minute lockout; successful login clears the failed counter.

#### C. Time-Lock Enforcement (REQ-23, REQ-24)
- **Server Clock Authority:** Time checks strictly evaluated against database UTC timestamp (`SELECT UTC_TIMESTAMP(6)`), never the client OS clock.
- **Access Gate:** Paper access throws `PaperLockedException` when `now < unlock_at`. Access is granted once `now >= unlock_at` and `now <= unlock_at + duration`.
- **Expired Exam Window:** Paper access rejected when `now > unlock_at + duration`.

#### D. Hash-Chained Integrity Ledger (REQ-30 through REQ-37)
- **Canonical Hash Formula:** Verified using Unit Separator (`\u001F`) delimiter, 64-zero genesis previous hash, and lowercase SHA-256 hex encoding.
- **Chain Verification:** `LedgerService.verify()` successfully validates full chain integrity.
- **Tamper Detection:**
  - `MODIFIED_RECORD`: Recomputing altered text detects tampering at the exact sequence number.
  - `BROKEN_LINK`: Corrupting previous hash links detects link disruption.
  - `BAD_GENESIS`: Altering genesis previous hash detects non-conforming root.
- **External Anchor & Tail Deletion:** Exporting anchor file and deleting/truncating tail records detected with `MISSING_RECORD`.

#### E. Ethical Student Support & Privacy (REQ-38 through REQ-43)
- **Whole-Word & Phrase Matching:** `KeywordScanner` matches whole words (`"stress"`, `"anxiety"`, `"help"`) while rejecting substrings (e.g., `"distress"` when scanning for `"stress"` alone). Case-insensitive and Unicode normalized.
- **Message Privacy:** Student message encrypted with AES-256-GCM. Raw plaintext message is NEVER stored in database columns, logs, reports, or the ledger.
- **Ethical Human-in-the-Loop:** Students are never told their message was flagged; exam taking is never blocked.
- **Staff Review:** Authorized staff review decrypts on the fly, logs review notes, and transitions status (`UNDER_REVIEW`, `REVIEWED`, `CLOSED`).

#### F. Reporting & RFC 4180 CSV Export (REQ-44)
- **Extensible Architecture:** Interface-driven report generation (`Report`) supports independent addition of reports.
- **Privacy Enforcement:** Grievance report aggregates metric counts only; zero private student text is exposed.
- **CSV Formatting:** Output strictly adheres to RFC 4180 standard (escapes commas, quotes, line breaks).

---

### 3. End-to-End System Integration Test (`SystemIntegrationTest`)
The end-to-end integration test executed the full 10-step lifecycle in sequence:
1. Staff account creation and login (`PROF201`, Dr. Grace Hopper)
2. Student registration and login (`CS2026-999`, Ada Lovelace)
3. Exam creation with scheduled AES-256-GCM paper encryption
4. Time-lock enforcement (access blocked with `PaperLockedException` before unlock time; access granted once time arrives)
5. Student paper decryption with unwrapped data key & SHA-256 plaintext integrity validation
6. Answer submission and single-submission constraint check
7. Student support message submission with distress keyword scanning (flagged for review)
8. Staff review queue inspection, on-the-fly decryption, and review decision recording
9. Report generation and RFC 4180 CSV export
10. Full SHA-256 integrity ledger walk & external anchor export/verification

---

### 4. How to Reproduce and Run Tests

Run the full automated test suite from the project root:
```bash
cd "d:\Exam security\exam-security-system"
.\mvnw.cmd test
```

To run a specific test class:
```bash
.\mvnw.cmd test -Dtest=SystemIntegrationTest
.\mvnw.cmd test -Dtest=CryptoServiceTest
.\mvnw.cmd test -Dtest=LedgerServiceTest
```
