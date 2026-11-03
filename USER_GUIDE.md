# User Guide & Operations Manual
## Exam Security and Student Support System

Welcome to the **Exam Security and Student Support System** user guide. This document provides step-by-step instructions for both **Students** and **Faculty/Staff Administrators**.

---

## Part 1: Student Guide

### 1.1 Student Registration
1. Launch the application and select the **Student** radio button on the login screen.
2. Click **"Register as Student"**.
3. Fill in your university details:
   - **Roll Number:** e.g., `CS2026-004` (must follow alphanumeric university roll format).
   - **Full Name:** Your complete legal name.
   - **Email Address:** Your university email address.
   - **Password:** Minimum 8 characters, containing at least one letter and one number.
   - **Confirm Password:** Re-enter your password to prevent typos.
4. Click **"Create Account"**. Your account is secured with PBKDF2WithHmacSHA256 (210,000 hashing rounds), and an audit entry is created in the ledger.

---

### 1.2 Signing In
1. On the login screen, select **Student**.
2. Enter your **Roll Number** and **Password**.
3. Click **"Sign In"**.
   > **Note on Account Lockout:** If an incorrect password is typed 5 consecutive times, the account enters a 5-minute lockout period for brute-force protection.

---

### 1.3 The Student Dashboard
Upon signing in, you are greeted by the Student Dashboard:
- **Security Indicators (Top Right):**
  - `● AES-256-GCM Active`: Confirms exam paper encryption is engaged.
  - `● Ledger Integrity Ready`: Confirms the SHA-256 audit log is online.
  - `● Audit Log Online`: Confirms event logging is functioning.
- **Three Core Action Cards:**
  1. **Exam Portal:** Browse examinations, view countdown timers, and take exams.
  2. **Integrity Ledger:** Inspect the transparent, tamper-evident audit ledger.
  3. **Student Support:** Confidential portal to raise concerns, exam queries, or personal check-in requests.
- **My Support Requests (Bottom):** Displays a history of your past support submissions and their status (`New`, `Under Review`, `Reviewed`, `Closed`).

---

### 1.4 Viewing & Taking Examinations
1. Click **"Go to Exam Portal"**.
2. The exam table lists all scheduled examinations:
   - **LOCKED:** The paper is encrypted and cannot be opened until the scheduled start time. A live countdown displays the exact minutes and seconds remaining until release.
   - **UNLOCKED:** The examination is actively open. The countdown displays the remaining test window.
   - **CLOSED:** The examination window has elapsed.
   - **SUBMITTED:** You have already submitted your answers for this exam.
3. Select an **UNLOCKED** exam and click **"Take Selected Exam"**.
4. **During the Exam:**
   - Review questions: Multiple Choice Questions (MCQs) provide radio button options; descriptive questions provide text areas.
   - Watch the timer in the top header.
   - When finished, click **"Submit Final Answers"**.
   - Confirm your submission when prompted.
   > **Important Security Rule:** Each student is permitted strictly **one** final submission per examination. Once submitted, answers cannot be edited or resubmitted.

---

### 1.5 Submitting Support Messages or Exam Concerns
If you face distress, need accommodation, or encounter a technical issue:
1. Navigate to **Student Support** from the dashboard.
2. Select an appropriate category:
   - *Exam Concern*
   - *Grievance*
   - *Personal Check-in*
   - *Other*
3. Enter a clear Subject line and type your message (10 to 2,000 characters).
4. Click **"Submit Message"**.
5. You will receive a calm, neutral confirmation receipt.
   > **Ethical Protection:** Your message is encrypted with AES-256-GCM and visible only to authorized human staff. Your examination access is **never** blocked, and your submission is never penalized.

---

## Part 2: Faculty & Staff Guide

### 2.1 First-Launch Staff Initialization
If no staff accounts exist in the database:
1. When selecting **Faculty / Staff** on the login screen, a prompt will ask if you wish to initialize the first Staff Administrator.
2. Alternatively, run from the command line:
   ```bash
   java -jar target/exam-security-system.jar --create-staff
   ```
3. Enter your Staff ID (e.g., `PROF101`), Full Name, Email, and strong password.

---

### 2.2 Staff Dashboard Overview
The Staff Dashboard provides eight live metric overview counters:
- **Total Exams, Locked Papers, Unlocked & Open, Closed Exams**
- **Total Submissions, Flagged for Review, Ledger Status, Master Encryption**

Navigation buttons provide one-click access to all administrative capabilities.

---

### 2.3 Creating a Scheduled Exam with Time-Locked Paper
1. Click **"Create Exam"** from the dashboard.
2. Fill in the examination details:
   - **Exam Title:** e.g., `Database Systems & Concurrency`
   - **Subject Code:** e.g., `CS302`
   - **Exam Duration:** Duration in minutes (1 to 600 minutes).
   - **Scheduled Unlock Time:** Exact date and time when the paper should unlock. Must be in the future.
   - **Instructions:** Instructions displayed to students.
3. Click **"Browse Question Paper (.txt)"** and select your formatted UTF-8 plain text paper (maximum 1 MB).
4. Click **"Create & Time-Lock Exam"**.
   - *What happens under the hood:* The system generates a fresh random 256-bit AES data key, encrypts the paper with AES-256-GCM, wraps the data key under the application Master Key, records the plaintext SHA-256 hash, and zeroes the plaintext from memory immediately.

---

### 2.4 Managing Examinations (Rescheduling & Cancellation)
1. Click **"Manage Exams"**.
2. Select any exam in the table.
3. **Reschedule Unlock Time:**
   - If an exam has not yet unlocked, click **"Reschedule Unlock Time"** to choose a new future release timestamp.
   - Audited automatically to the integrity ledger.
4. **Cancel Exam:**
   - If an exam has not yet unlocked, click **"Cancel Exam"**.
   > **Immutability Rule:** Once an exam reaches its unlock time, it can no longer be rescheduled or cancelled.

---

### 2.5 Reviewing Submissions & Auto-Marking MCQs
1. Click **"Review Submissions"**.
2. Select the examination from the dropdown menu to view all student submissions.
3. Select a submission and click **"Inspect Submission"**.
4. The inspection window displays:
   - Student's response for each question.
   - For MCQs where an `ANSWER: <X>` key was provided in the paper, the system automatically grades the response, calculates marks, and highlights **✔ CORRECT** in green or **✘ INCORRECT** in red.
   - For descriptive questions, student answers are displayed for manual grading.

---

### 2.6 Reviewing Student Support Grievances
1. Click **"Review Grievances"**.
2. In accordance with ethical human-in-the-loop oversight, grievances matched by wellness keywords appear at the top of the queue tagged with a **FLAGGED** indicator.
3. Select a grievance and click **"Decrypt & Review Grievance"**.
4. The dialog decrypts the student's message using the master key.
5. Review the message, select an updated status (`Under Review`, `Reviewed`, `Closed`), and type a review note detailing the support action taken.
6. Click **"Submit Review"**.

---

### 2.7 Configuring Support Keywords
1. Click **"Keyword Settings"**.
2. The table displays all active and inactive support keywords and categories (`STRESS`, `ACADEMIC`, `EMERGENCY`, `ACCOMMODATION`, `TECHNICAL`).
3. To toggle a keyword, select it and click **"Toggle Active / Inactive"**.
4. To add a keyword, click **"Add New Keyword"**, enter the word or phrase, and select the category.

---

### 2.8 System Reports & RFC 4180 CSV Export
1. Click **"System Reports"**.
2. Select from the five reports:
   - *1. Examination Status & Time-Lock Overview*
   - *2. Student Submission Summary per Exam*
   - *3. SHA-256 Ledger Chain Statistics*
   - *4. Support Grievances Metric Summary (Counts Only)*
   - *5. Exam Activity Audit Trail*
3. Click **"Export to CSV"** to save standard RFC 4180 formatted CSV data to your computer.

---

### 2.9 Verifying the Integrity Ledger & External Anchors
1. Click **"Integrity Ledger"**.
2. Click **"Verify Ledger Integrity"**.
   - The system walks the cryptographic chain from genesis (64 zeros) through every record.
   - If valid, a prominent green banner displays: `✔ INTEGRITY VERIFIED`.
   - If any row was tampered with, a red banner displays: `✖ TAMPER DETECTED`, specifying the exact sequence number and failure reason.
3. **Exporting Anchors:** Click **"Export Anchor"** to save the latest sequence number and hash to an external file.
4. **Verifying with External Anchor:** Click **"Verify with External Anchor"** and choose a previous anchor file. This detects if any malicious administrator has truncated or deleted rows from the end of the table.
