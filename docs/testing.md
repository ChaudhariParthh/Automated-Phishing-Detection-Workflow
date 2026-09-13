````markdown
# Testing Guide

## 1. Purpose

The Automated Phishing Detection Workflow should be tested end-to-end to verify that:

- Gmail emails are detected correctly.
- AI Agent #1 extracts email content and security-relevant information.
- URLs are submitted to VirusTotal and their analysis results are collected.
- Attachments are submitted to VirusTotal and their analysis results are collected.
- AI Agent #2 (SOC Analyst) evaluates the complete email context.
- The Switch node correctly routes the email according to the SOC verdict.
- CLEAN emails are logged in Supabase and remain in the inbox.
- SUSPICIOUS emails generate Slack and Jira notifications and are logged in Supabase.
- MALICIOUS emails are quarantined, generate Slack and Jira notifications, and are logged in Supabase.

---

## 2. Testing Strategy

Testing should be performed in stages:

1. CLEAN email test
2. SUSPICIOUS email test
3. MALICIOUS email test
4. Attachment processing test
5. URL reputation test
6. Supabase logging verification
7. Slack notification verification
8. Jira issue creation verification
9. Gmail quarantine verification
10. Final end-to-end validation

---

## 3. CLEAN Email Test

### Objective

Verify that a normal email containing safe content and safe URLs is classified as `CLEAN`.

### Example Test Email

**Subject:**
```text
Security phishing detection automation test
````

**Body:**

```text
Hello,

This is a phishing detection automation test.

Visit:
https://example.com
https://www.google.com

Regards,
Security Testing
```

### Expected Result

AI Agent #1 should extract:

* Subject
* Sender
* Recipient
* Email body
* URLs
* Attachment names
* Suspicious indicators

The URLs should be processed through VirusTotal.

AI Agent #2 should return a classification similar to:

```json
{
  "verdict": "CLEAN",
  "recommended_action": "ALLOW"
}
```

The Switch node should route the item through the `CLEAN` output.

### Expected Actions

* Email remains in the Gmail inbox.
* No Slack alert is generated.
* No Jira issue is created.
* A record is inserted into Supabase.
* `action_taken` should be:

```text
ALLOWED
```

---

## 4. SUSPICIOUS Email Test

### Objective

Verify that an email containing suspicious characteristics but without confirmed malicious evidence is classified as `SUSPICIOUS`.

### Example Test Email

**Subject:**

```text
Account verification required
```

**Body:**

```text
Hello,

Your account requires verification.

Please review the information using the link below:

https://example.com/account-verification

Please complete the verification as soon as possible.

Regards,
Support Team
```

### Expected Result

The SOC Analyst should evaluate:

* Urgency
* Suspicious wording
* External URLs
* Sender information
* VirusTotal results
* Other phishing indicators

If the evidence is concerning but insufficient to classify the email as malicious, the expected result is:

```json
{
  "verdict": "SUSPICIOUS",
  "recommended_action": "ALERT"
}
```

### Expected Actions

The `SUSPICIOUS` branch should execute:

```text
SUSPICIOUS
    ↓
Slack
    ↓
Jira
    ↓
Supabase
```

The original email should remain in the inbox.

Supabase should contain:

```text
verdict = SUSPICIOUS
action_taken = ALERTED_AND_TICKET_CREATED
```

---

## 5. MALICIOUS Email Test

### Objective

Verify that an email containing strong malicious indicators is classified as `MALICIOUS`.

For safety, use a controlled security-testing sample or an approved phishing/malware simulation rather than sending real malicious content.

### Expected Result

The SOC Analyst should produce:

```json
{
  "verdict": "MALICIOUS",
  "recommended_action": "QUARANTINE"
}
```

The Switch node should route the email through the `MALICIOUS` output.

### Expected Actions

The malicious branch should execute:

```text
MALICIOUS
    ↓
Gmail Thread Trash
    ↓
Slack
    ↓
Jira
    ↓
Supabase
```

### Gmail

The Gmail Thread Trash node should move the malicious email thread to Trash.

### Slack

A security alert should be generated containing:

* Subject
* Sender
* Risk score
* Verdict
* Recommended action
* Reasons
* Confirmation that the email was quarantined

### Jira

A Task should be created containing:

* Email details
* SOC Analyst assessment
* Risk score
* Verdict
* Recommended action
* Reasons
* URL assessment
* Attachment assessment
* Automated response

### Supabase

The event should be logged with:

```text
verdict = MALICIOUS
action_taken = QUARANTINED
```

---

## 6. URL Testing

URL testing verifies the URL extraction and VirusTotal integration.

The workflow should process URLs as follows:

```text
Email
  ↓
AI Agent #1
  ↓
Extract URLs
  ↓
Split Out
  ↓
VirusTotal URL Submission
  ↓
VirusTotal Analysis Result
  ↓
Aggregate URL Results
```

### Verify

For every URL:

* The URL is extracted correctly.
* The URL is submitted to VirusTotal.
* A VirusTotal analysis ID is returned.
* The analysis result is retrieved.
* The result is included in the aggregated URL data.
* The SOC Analyst receives the URL intelligence.

If multiple URLs exist, each URL should be processed individually.

---

## 7. Attachment Testing

Attachment testing verifies that email attachments are detected and processed.

The workflow should process attachments as follows:

```text
Gmail Trigger
  ↓
IF
  ↓
Split Out
  ↓
VirusTotal File Upload
  ↓
Wait
  ↓
VirusTotal File Analysis Result
  ↓
Aggregate File Analysis Results
```

### Verify

* The attachment is detected by the IF node.
* Each attachment is split into an individual item.
* The binary file is uploaded to VirusTotal.
* VirusTotal analysis is allowed time to complete.
* The analysis result is collected.
* Multiple attachments are aggregated correctly.
* The SOC Analyst receives the attachment assessment.

For safe testing, use benign test files rather than real malware.

---

## 8. Supabase Verification

After each test, open the `security_events` table and verify that a corresponding record exists.

Verify the following fields:

| Field                    | Expected Value                      |
| ------------------------ | ----------------------------------- |
| `email_message_id`       | Gmail message ID                    |
| `subject`                | Extracted email subject             |
| `sender`                 | Extracted sender                    |
| `recipient`              | Extracted recipient                 |
| `email_body`             | Extracted email body                |
| `risk_score`             | SOC Analyst risk score              |
| `verdict`                | CLEAN / SUSPICIOUS / MALICIOUS      |
| `recommended_action`     | ALLOW / ALERT / QUARANTINE          |
| `reasons`                | SOC Analyst reasons                 |
| `url_assessments`        | URL analysis results                |
| `attachment_assessments` | Attachment analysis results         |
| `action_taken`           | Final automated action              |
| `created_at`             | Automatically generated by Supabase |

---

## 9. Slack Verification

For `SUSPICIOUS` and `MALICIOUS` tests, verify that the Slack node executes successfully.

The notification should contain:

* Email subject
* Sender
* Risk score
* Verdict
* Recommended action
* Reasons for the classification
* Automated action taken

CLEAN emails should not generate Slack notifications.

---

## 10. Jira Verification

For `SUSPICIOUS` and `MALICIOUS` tests, verify that the Jira node successfully creates a Task.

The issue should contain:

* Email details
* Risk score
* Verdict
* Recommended action
* SOC Analyst reasons
* URL assessment
* Attachment assessment
* Automated response

CLEAN emails should not create Jira issues.

---

## 11. Gmail Quarantine Verification

Only the `MALICIOUS` branch should execute the Gmail Thread Trash operation.

Verify that:

1. The malicious test email is detected.
2. The Switch routes the email to `MALICIOUS`.
3. The Gmail Thread Trash node receives the correct `threadId`.
4. The email thread is moved to Trash.
5. The Slack notification is generated.
6. The Jira Task is created.
7. The Supabase record is created.

CLEAN and SUSPICIOUS emails should remain in the inbox.

---

## 12. Final End-to-End Validation

A successful end-to-end test should demonstrate:

```text
                    Gmail
                      │
                      ▼
              Email Extraction
                      │
          ┌───────────┴───────────┐
          ▼                       ▼
        URLs                  Attachments
          │                       │
          ▼                       ▼
     VirusTotal              VirusTotal
          │                       │
          └───────────┬───────────┘
                      ▼
                 SOC Analyst
                      │
                      ▼
                    Switch
          ┌───────────┼───────────┐
          ▼           ▼           ▼
      MALICIOUS   SUSPICIOUS     CLEAN
          │           │           │
          ▼           ▼           ▼
    Gmail Trash     Slack       Supabase
          │           │
          ▼           ▼
        Slack        Jira
          │           │
          ▼           ▼
        Jira       Supabase
          │
          ▼
       Supabase
```

The workflow is considered successfully tested when all three verdict paths execute the expected actions and the corresponding Supabase records accurately reflect the final automated response.

---

## 13. Testing Notes

* Use controlled test emails.
* Do not use real credentials or sensitive information in test emails.
* Do not commit API keys, tokens, or credentials to GitHub.
* Use benign files when testing attachment processing.
* VirusTotal results may initially remain queued; allow sufficient time for analysis.
* Re-submitting an already submitted file may return `409 AlreadySubmittedError`. This indicates that VirusTotal has already received the file and does not necessarily indicate an API credential problem.
* AI-generated classifications should be validated against the actual email and available threat-intelligence results.
* SPF/DKIM verification is not implemented in the current workflow and should not be included as completed test coverage.

---

## 14. Test Completion Checklist

* [ ] Gmail Trigger receives the test email.
* [ ] AI Agent #1 extracts the email correctly.
* [ ] URLs are processed by VirusTotal.
* [ ] Attachments are processed by VirusTotal when present.
* [ ] AI Agent #2 generates a valid SOC assessment.
* [ ] Switch correctly routes `CLEAN`.
* [ ] Switch correctly routes `SUSPICIOUS`.
* [ ] Switch correctly routes `MALICIOUS`.
* [ ] CLEAN events are logged as `ALLOWED`.
* [ ] SUSPICIOUS events generate Slack + Jira and are logged as `ALERTED_AND_TICKET_CREATED`.
* [ ] MALICIOUS events are quarantined.
* [ ] MALICIOUS events generate Slack + Jira.
* [ ] MALICIOUS events are logged as `QUARANTINED`.
* [ ] Supabase records contain the expected event information.
* [ ] No credentials or secrets are exposed in the repository.

---

