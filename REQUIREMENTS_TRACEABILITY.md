# Requirements Traceability Matrix
## Exam Security and Student Support System

This document maps all functional requirements (REQ-1 through REQ-44) to their system modules, database tables, service layers, and verification methods.

| Req ID | Description | Target Module / Class | DB Tables Involved | Verification Method |
|---|---|---|---|---|
| **REQ-1** | Student registration with field validations (unique roll_no, email, strong password) | `AuthService`, `StudentDao`, `Validator` | `students` | Unit + Integration Test |
| **REQ-2** | Student login with roll number and password | `AuthService`, `StudentDao`, `PasswordHasher` | `students`, `integrity_ledger` | Unit + Integration Test |
| **REQ-3** | Faculty/Staff login with Staff ID; credential source determines role strictly | `AuthService`, `StaffDao`, `StudentDao` | `staff`, `students`, `integrity_ledger` | Unit + Integration Test |
| **REQ-4** | Staff first-run setup CLI (`--create-staff`) / first-launch dialog; no hardcoded prod credentials; staff can add staff | `Main`, `AuthService`, `StaffDao` | `staff`, `integrity_ledger` | CLI / Integration Test |
| **REQ-5** | PBKDF2WithHmacSHA256 hashing (16+ byte salt, >=210,000 iterations, constant-time compare) | `PasswordHasher` | `students`, `staff` | Unit Test |
| **REQ-6** | Account lockout after 5 failed attempts for 5 minutes; generic error messages; reset on success | `AuthService`, `StudentDao`, `StaffDao` | `students`, `staff` | Unit + Integration Test |
| **REQ-7** | `SessionManager` tracking active user, role, login time; 15-minute inactivity timeout | `SessionManager`, `LoginController` | In-memory session | Unit Test + UI Event Watcher |
| **REQ-8** | `AccessGuard` enforcing role authorization in service layer; raises `AccessDeniedException` & logs ledger | `AccessGuard`, Service layer | `integrity_ledger` | Unit + Integration Test |
| **REQ-9** | Ledger recording for login success & failure without leaking passwords | `AuthService`, `LedgerService` | `integrity_ledger` | Integration Test |
| **REQ-10** | Student dashboard welcome banner & verified real-time security indicators | `StudentDashboardView`, `StudentController` | `integrity_ledger`, `exam_papers` | UI Verification |
| **REQ-11** | Student dashboard cards (Exam Portal, Integrity Ledger read-only, Support) | `StudentDashboardView`, `RoundedCard` | N/A | UI Verification |
| **REQ-12** | Exam Portal table with state badges (LOCKED/UNLOCKED/CLOSED/SUBMITTED) & live countdown | `ExamPortalView`, `ExamService`, `TimeLockService` | `exams`, `submissions` | UI + Unit Test |
| **REQ-13** | Grievance history list showing only the logged-in student's requests | `SupportHistoryView`, `GrievanceService` | `grievances` | Integration Test |
| **REQ-14** | Universal logout button clearing session across all screens | Top navigation bar / Views | In-memory session | UI Test |
| **REQ-15** | Faculty/Staff dashboard overview metric cards & ledger status summary | `StaffDashboardView`, `StaffController` | `exams`, `submissions`, `grievances`, `integrity_ledger` | UI + Integration Test |
| **REQ-16** | Comprehensive Staff navigation (Create, Manage, Submissions, Ledger, Grievances, Reports, Keywords, Add Staff) | `StaffDashboardView`, Navigation router | N/A | UI Verification |
| **REQ-17** | Staff route & service enforcement restricted strictly to authenticated staff | `AccessGuard`, Controllers | N/A | Unit + Service Tests |
| **REQ-18** | Create Exam form: title, subject, instructions, duration (1-600m), future unlock time, file picker | `CreateExamView`, `ExamService`, `Validator` | `exams` | UI + Unit Test |
| **REQ-19** | Plain text `.txt` (UTF-8, <=1MB) paper parser supporting MCQ & TEXT formats, with fallback mode | `PaperParser`, `Question` | File I/O | Unit Tests (valid, fallback, rejection) |
| **REQ-20** | Atomic exam creation: fresh AES-256 data key, AES-GCM (12-byte IV), SHA-256 plaintext hash, key wrapping, ledger records | `ExamService`, `CryptoService`, `KeyStoreManager`, `LedgerService` | `exams`, `exam_papers`, `integrity_ledger` | Unit + Integration Test |
| **REQ-21** | Plaintext paper never persisted to DB or disk; zeroed in memory | `ExamService`, `PaperService`, `CryptoService` | `exam_papers` | Code Audit + Test |
| **REQ-22** | Manage Exams: view status, cancel prior to unlock, update unlock time prior to unlock (all ledger-audited) | `ExamService`, `ExamDao`, `LedgerService` | `exams`, `integrity_ledger` | Integration Test |
| **REQ-23** | Papers strictly inaccessible to students prior to `unlock_at` | `TimeLockService`, `PaperService` | `exams`, `exam_papers` | Unit + Integration Test |
| **REQ-24** | Service-level time-lock: verifies DB server time `SELECT UTC_TIMESTAMP(6)`, validates window, decrypts & checks SHA-256 | `TimeLockService`, `PaperService`, `CryptoService` | `exams`, `exam_papers`, `DatabaseManager` | Unit + Integration Test |
| **REQ-25** | First paper access logs `EXAM_UNLOCKED` (once) & `PAPER_ACCESSED` (per student) | `PaperService`, `ExamAccessDao`, `LedgerService` | `exam_access`, `integrity_ledger` | Integration Test |
| **REQ-26** | Exam taking UI: countdown timer, MCQ radio group, TEXT answer area, confirm prompt | `TakeExamView`, `ExamController` | In-memory exam session | UI Verification |
| **REQ-27** | Single-submission enforcement (`UNIQUE(exam_id, student_id)`), window bounds check, atomic save with answers & ledger | `SubmissionService`, `SubmissionDao`, `SubmissionAnswerDao` | `submissions`, `submission_answers`, `integrity_ledger` | Integration Test |
| **REQ-28** | Graceful crypto & tampering error handling; GCM auth tag failure prevents partial plaintext display | `CryptoService`, `PaperService`, `CustomExceptionHandler` | `exam_papers` | Unit + Integration Test |
| **REQ-29** | Staff submission viewer: student answer display, auto-mark MCQs only when `ANSWER: <X>` key present | `SubmissionReviewView`, `SubmissionService` | `submissions`, `submission_answers` | Unit + UI Test |
| **REQ-30** | Append-only `integrity_ledger` table; application has zero UPDATE/DELETE methods | `LedgerDao`, `DatabaseManager` | `integrity_ledger` | Code Audit + DAO Test |
| **REQ-31** | Ledger record structure: `seq_no`, `event_uuid`, `event_type`, `actor_id`, `actor_role`, `exam_id`, `description`, `event_time`, `prev_hash`, `curr_hash` | `LedgerEntry`, `LedgerDao` | `integrity_ledger` | Integration Test |
| **REQ-32** | Canonical hash formula using delimiter `U+001F`, genesis 64 zeros, lowercase hex | `HashUtil`, `LedgerService` | `integrity_ledger` | Unit Test |
| **REQ-33** | Concurrency-safe ledger append using pessimistic lock `FOR UPDATE` in atomic transaction | `LedgerDao`, `LedgerService`, `DatabaseManager` | `integrity_ledger` | Concurrency Test |
| **REQ-34** | Full coverage of mandatory audit events across authentication, exam, paper, support, and ledger operations | All services | `integrity_ledger` | Integration Tests |
| **REQ-35** | `LedgerService.verify()` full chain validation: detect `MODIFIED_RECORD`, `BROKEN_LINK`, `MISSING_RECORD`, `BAD_GENESIS` | `LedgerService`, `LedgerView` | `integrity_ledger` | Unit + Integration Test |
| **REQ-36** | Ledger anchor export & external anchor verification detecting tail truncation | `LedgerService`, `LedgerView` | `integrity_ledger`, Anchor File | Unit + Integration Test |
| **REQ-37** | Developer tamper demo tool (CLI / test script) for controlled ledger row alteration | `TamperDemoTool` | `integrity_ledger` | Script Test |
| **REQ-38** | Student Support portal: categories, 10-2000 char message, neutral receipt message, no diagnostic labels, exams unblocked | `SupportView`, `GrievanceService` | `grievances` | UI + Integration Test |
| **REQ-39** | `KeywordScanner`: case-insensitive, Unicode-normalized, whole-word/phrase detection, decoupled & unit-testable | `KeywordScanner` | `support_keywords` | Unit Tests |
| **REQ-40** | Grievance submission workflow: validation -> scan -> AES-GCM encryption -> store -> ledger (no raw text in ledger) | `GrievanceService`, `CryptoService`, `LedgerService` | `grievances`, `integrity_ledger` | Integration Test |
| **REQ-41** | Staff grievance review: flagged-first order, decrypt message for staff only, disclaimer banner, audit review notes | `GrievanceReviewView`, `GrievanceService` | `grievances`, `grievance_reviews`, `integrity_ledger` | UI + Integration Test |
| **REQ-42** | Keyword management UI: activate/deactivate/add keywords, ledger audited | `KeywordSettingsView`, `KeywordService`, `LedgerService` | `support_keywords`, `integrity_ledger` | UI + Integration Test |
| **REQ-43** | Privacy preservation: zero grievance text in logs, reports, ledger, or dashboard overviews | All layers | N/A | Code Audit |
| **REQ-44** | Staff Reports & CSV export: exam list, submission summary, ledger stats, grievance counts, activity audit | `ReportService`, `Report` interface, `ReportView` | Multiple tables | Unit + UI Test |
