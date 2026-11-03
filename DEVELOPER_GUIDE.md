# Developer & Architecture Guide
## Exam Security and Student Support System

> Written for software engineering students, security architects, and backend maintainers.

---

## 1. Architectural Philosophy

The **Exam Security and Student Support System** is designed with three core engineering principles:

1. **Defense in Depth & Zero Trust:** We never trust the client workstation's system clock, we never trust database administrators not to attempt SQL-level modifications, and we never rely on third-party security frameworks when the standard Java Cryptography Architecture (JCA) provides battle-tested primitives.
2. **Deterministic & Decoupled Design:** High-assurance components (such as `KeywordScanner`, `HashUtil`, `PaperParser`, and `PasswordHasher`) have **zero** database or UI dependencies. They can be unit-tested in isolation in milliseconds.
3. **Strict Separation of Concerns:**
   - `model`: Pure domain entities and enums.
   - `dao`: SQL queries, transactions, and result-set mapping.
   - `service`: Business validation, access control, time-locking, and ledger auditing.
   - `security`: Low-level cryptographic primitives and session state.
   - `view`: Java Swing user interface components and event listeners.

---

## 2. Directory & Package Structure

```
com.examsecurity/
├── dao/               # JDBC Data Access Objects
│   ├── ExamDao.java            # Exam queries & cancellation
│   ├── ExamAccessDao.java      # Student paper access auditing
│   ├── GrievanceDao.java       # Encrypted grievance persistence
│   ├── GrievanceReviewDao.java # Staff review history
│   ├── KeywordDao.java         # Configurable keyword queries
│   ├── LedgerDao.java          # Append-only ledger with pessimistic locking
│   ├── PaperDao.java           # Encrypted question paper storage
│   ├── StaffDao.java           # Staff account management
│   ├── StudentDao.java         # Student account management
│   ├── SubmissionDao.java      # Exam submissions
│   └── SubmissionAnswerDao.java# Student question answers
├── database/          # Database connection pooling
│   └── DatabaseManager.java    # Connection management & SQL script execution
├── model/             # Domain entities and enums
│   ├── Exam.java               # Exam entity & state computation
│   ├── ExamPaper.java          # Ciphertext, IV, wrapped key entity
│   ├── Grievance.java          # Support request entity
│   ├── LedgerEntry.java        # Immutable hash-chained ledger row
│   ├── Question.java           # Parsed question (MCQ or TEXT)
│   ├── Role.java               # STUDENT, STAFF
│   └── ExamState.java          # LOCKED, UNLOCKED, CLOSED
├── security/          # Cryptographic primitives
│   ├── AccessGuard.java        # Role enforcement and audit hooks
│   ├── CryptoService.java      # AES-256-GCM encryption, decryption, key-wrapping
│   ├── HashUtil.java           # SHA-256 & canonical ledger hash calculation
│   ├── KeyStoreManager.java    # PKCS12 KeyStore & Master Key management
│   ├── PasswordHasher.java     # PBKDF2WithHmacSHA256 (210,000 iterations)
│   └── SessionManager.java     # Thread-safe in-memory session tracker
├── service/           # Business logic & workflows
│   ├── AuthService.java        # Registration, login, lockout tracking
│   ├── ExamService.java        # Scheduled exam creation & paper encryption
│   ├── GrievanceService.java   # Support message encryption & review workflow
│   ├── KeywordScanner.java     # Unicode-normalized whole-word regex engine
│   ├── KeywordService.java     # Keyword management & audit logging
│   ├── LedgerService.java      # Chain walking, verification, anchor export
│   ├── PaperParser.java        # Question paper text parser
│   ├── PaperService.java       # Access verification, paper decryption
│   ├── ReportService.java      # 5 core system reports & CSV export
│   ├── SubmissionService.java  # Answer submission & MCQ auto-marking
│   └── TimeLockService.java    # Server-time-based access gate
├── util/              # Utilities & helpers
│   ├── BackgroundTask.java     # Non-blocking SwingWorker wrapper
│   ├── TamperDemoTool.java     # Developer utility to simulate SQL tampering
│   └── Validator.java          # Field and input validation rules
└── view/              # Java Swing GUI views
    ├── MainFrame.java          # Root JFrame, CardLayout, security top bar
    ├── components/             # Reusable UI widgets (AccentButton, ThemedTable, etc.)
    └── theme/                  # Theme tokens (colors, typography)
```

---

## 3. Cryptography Pipeline Walkthrough

### 3.1 AES-256-GCM Envelope Encryption (Question Papers)
When faculty uploads a question paper:
1. `PaperParser` parses and validates the UTF-8 text file (max 1 MB).
2. `HashUtil.sha256Hex()` computes the SHA-256 hash of the raw plaintext bytes.
3. `KeyStoreManager.generateDataKey()` generates a fresh 256-bit AES key (`dataKey`).
4. `CryptoService.encrypt()` encrypts the paper bytes using `AES/GCM/NoPadding` with a fresh 12-byte random IV and AAD bound to the exam ID (`"EXAM_" + examId`).
5. `CryptoService.wrapDataKey()` wraps the `dataKey` under the application Master Key (loaded from the PKCS12 KeyStore) using AES-GCM with AAD `"KEY_WRAP_V1"`.
6. `CryptoService.wipeBytes()` zeroes the plaintext bytes in memory.
7. The ciphertext, IV, wrapped data key, wrap IV, and plaintext SHA-256 hash are saved atomically to `exam_papers` inside a database transaction.
8. The plaintext paper is **never** written to disk or stored in the database.

---

### 3.2 Time-Lock Verification & Decryption
When a student attempts to access a question paper:
1. `TimeLockService.assertPaperAccessible(exam)` queries the database server time using `SELECT UTC_TIMESTAMP(6)`.
2. If `serverTime < exam.getUnlockAt()`, access is denied immediately with `PaperLockedException`.
3. If `serverTime > exam.getUnlockAt() + exam.getDurationMinutes()`, access is denied (`ExamExpiredException`).
4. If within the valid window:
   - The master key is retrieved from the KeyStore.
   - `CryptoService.unwrapDataKey()` unwraps the data key.
   - `CryptoService.decrypt()` decrypts the ciphertext.
   - The SHA-256 hash of the decrypted bytes is checked against `plaintext_sha256`. If they do not match, a tampering alert is raised.
   - If this is the student's first access, an `EXAM_UNLOCKED` (once per exam) and `PAPER_ACCESSED` (per student) event is written to the ledger.

---

## 4. SHA-256 Integrity Ledger Architecture

### 4.1 Canonical Hash Formula
Each ledger entry's `curr_hash` is computed as:
$$\text{curr\_hash} = \text{SHA-256}(\text{seq\_no} \parallel \text{event\_uuid} \parallel \text{event\_type} \parallel \text{actor\_id} \parallel \text{actor\_role} \parallel \text{exam\_id} \parallel \text{description} \parallel \text{event\_time} \parallel \text{prev\_hash})$$

- **Delimiter:** ASCII Unit Separator `\u001F` (delimiter cannot collide with user text).
- **Null handling:** Null fields (such as `exam_id`) are encoded as the literal string `"NULL"`.
- **Timestamp:** Formatted to ISO-8601 UTC with microsecond precision: `yyyy-MM-dd'T'HH:mm:ss.SSSSSS'Z'`.
- **Genesis Record:** The very first entry (`seq_no = 1`) has `prev_hash` equal to **64 lowercase zeros** (`"0000000000000000000000000000000000000000000000000000000000000000"`).

### 4.2 Pessimistic Locking Concurrency
To ensure that concurrent users (e.g., 500 students submitting answers simultaneously) do not fork the hash chain, `LedgerDao.appendEvent` executes inside an atomic database transaction:
```sql
SELECT seq_no, curr_hash FROM integrity_ledger ORDER BY seq_no DESC LIMIT 1 FOR UPDATE;
```
This locks the tail row, ensures that each transaction reads the true latest sequence number and previous hash, and appends the new record sequentially.

### 4.3 Chain Verification & Failure Reasons
`LedgerService.verify()` walks the chain from sequence 1 to $N$. If any inconsistency is found, it reports the failing sequence number and exact violation code:
- `BAD_GENESIS`: Sequence 1 does not have 64 zeros as its previous hash.
- `MISSING_RECORD`: Gap in sequence numbers (e.g. sequence jumps from 4 to 6).
- `BROKEN_LINK`: `prev_hash` of entry $i$ does not match `curr_hash` of entry $i-1$.
- `MODIFIED_RECORD`: The recomputed hash of entry $i$ does not match its stored `curr_hash`.

### 4.4 External Anchor Verification (Tail Truncation Detection)
A corrupt DBA could theoretically delete the last 10 rows of the ledger; all remaining rows would still form a valid chain. To counter this:
- Staff can export an anchor file via `LedgerService.exportAnchor()`.
- During verification, `LedgerService.verify(anchorFile)` verifies that the database contains at least as many rows as the anchor file and that the anchor's sequence number matches the anchor's hash.

---

## 5. Ethical Student Support Keyword Scanner

The `KeywordScanner` implements a decoupled, ethical wellness detection engine:
- **Normalization:** Cleans text with `Normalizer.normalize(text, Normalizer.Form.NFC)` and converts to lowercase.
- **Word Boundary Regex:** Compiles each keyword with `\b` boundaries:
  ```java
  Pattern pattern = Pattern.compile("\\b" + Pattern.quote(keyword) + "\\b", Pattern.CASE_INSENSITIVE | Pattern.UNICODE_CASE);
  ```
- **Privacy Enforcement:**
  - The student's message is encrypted with AES-256-GCM using AAD `"GRIEVANCE_" + studentId`.
  - The plaintext message is **never** written to log files, the integrity ledger, or system reports.
  - The student is **never** informed their message was flagged, preserving psychological safety.
  - Exam access is **never** blocked.

---

## 6. Extending the System

### Adding a New Report
To add a new administrative report:
1. Open [ReportService.java](file:///d:/Exam%20security/exam-security-system/src/main/java/com/examsecurity/service/ReportService.java).
2. Implement a method returning a `Report`:
   ```java
   public Report generateCustomReport() {
       AccessGuard.requireStaff();
       List<List<String>> rows = ...;
       return new SimpleReport("Custom Report Title", "Description", List.of("Col1", "Col2"), rows);
   }
   ```
3. Add the report name to the dropdown in [ReportsView.java](file:///d:/Exam%20security/exam-security-system/src/main/java/com/examsecurity/view/ReportsView.java).
   - CSV export works automatically because the `Report` interface includes a default RFC 4180 `toCsv()` implementation!
