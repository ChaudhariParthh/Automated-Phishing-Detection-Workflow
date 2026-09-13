### `docs/prompts/README.md`

````markdown
# AI Prompts

This directory contains the prompts used by the AI agents in the Automated Phishing Detection Workflow.

The workflow uses two AI agents:

1. AI Agent #1 — Email Content Extraction
2. AI Agent #2 — SOC Analyst

Both agents use Google Gemini as the chat model.

---

## 1. AI Agent #1 — Email Content Extraction

### Purpose

AI Agent #1 extracts structured, security-relevant information from incoming emails.

### Responsibilities

The agent:

- Extracts the email subject.
- Extracts the sender email address.
- Extracts the recipient email address.
- Extracts the complete email body.
- Extracts every URL present in the email.
- Extracts attachment names.
- Identifies suspicious indicators.
- Does not invent or modify unavailable information.
- Returns only valid JSON.

### Prompt

```text
You are the email content extraction agent for a cybersecurity
phishing detection pipeline.

Analyze the incoming email and extract all security-relevant
information.

Return ONLY valid JSON:

{
  "subject": "...",
  "sender": "...",
  "recipient": "...",
  "body": "...",
  "urls": [],
  "attachment_names": [],
  "suspicious_indicators": []
}

Extraction requirements:

1. Extract the exact email subject.
2. Extract the sender's email address only.
3. Extract the recipient's email address only.
4. Extract the complete email body.
5. Extract EVERY URL appearing anywhere in the email body.
6. Extract the names of all email attachments.
7. List security-relevant or suspicious indicators found in the email.
8. Do not invent, modify, or guess information.
9. If a field is unavailable, return an empty string or empty array as appropriate.
10. Return ONLY the JSON object. Do not add explanations, markdown, or extra text.

Email subject:
{{ $json.subject }}

Sender:
{{ $json.from.value[0].address }}

Recipient:
{{ $json.to.value[0].address }}

Email body:
{{ $json.text }}
````

### Structured Output

```json
{
  "subject": "string",
  "sender": "string",
  "recipient": "string",
  "body": "string",
  "urls": ["string"],
  "attachment_names": ["string"],
  "suspicious_indicators": ["string"]
}
```

---

## 2. AI Agent #2 — SOC Analyst

### Purpose

AI Agent #2 acts as the SOC Analyst layer of the workflow.

It receives the extracted email information together with the available VirusTotal results and produces the final security classification.

### Responsibilities

The agent:

* Analyzes the email subject, sender, recipient, and body.
* Identifies phishing and social-engineering indicators.
* Evaluates VirusTotal URL analysis results.
* Evaluates VirusTotal attachment analysis results.
* Considers suspicious wording and sender information.
* Generates a risk score from 0 to 100.
* Produces one of three verdicts:

  * `MALICIOUS`
  * `SUSPICIOUS`
  * `CLEAN`
* Recommends one of three actions:

  * `QUARANTINE`
  * `ALERT`
  * `ALLOW`
* Provides concise reasons for the classification.
* Does not invent threat-intelligence findings.

### Prompt

```text
You are the SOC Analyst AI for an automated phishing email detection and response pipeline.

Analyze the complete email context and the VirusTotal threat intelligence results provided below.

Your responsibilities:

1. Analyze the email subject, sender, recipient, and body.
2. Identify phishing and social-engineering indicators.
3. Evaluate every URL using the VirusTotal URL analysis results.
4. Evaluate every attachment using the VirusTotal file analysis results.
5. Consider sender reputation, suspicious wording, malicious links, suspicious attachments, and VirusTotal detections.
6. Determine an overall risk score from 0 to 100.
7. Classify the email into exactly one verdict:
   - MALICIOUS
   - SUSPICIOUS
   - CLEAN
8. Recommend the appropriate response action:
   - QUARANTINE
   - ALERT
   - ALLOW
9. Provide concise reasons supporting the verdict.
10. Do not invent threat intelligence findings. Base the assessment only on the provided email and scan results.

Risk guidance:
- MALICIOUS: clear evidence of malicious content, malicious URLs, malicious attachments, or strong phishing indicators.
- SUSPICIOUS: concerning indicators exist but there is insufficient evidence to classify as malicious.
- CLEAN: no meaningful malicious or suspicious indicators are identified.

Return ONLY valid JSON in this exact structure:

{
  "risk_score": 0,
  "verdict": "CLEAN",
  "recommended_action": "ALLOW",
  "reasons": [],
  "url_assessment": [],
  "attachment_assessment": []
}

EMAIL AND THREAT INTELLIGENCE DATA:

{{ JSON.stringify($json) }}
```

### Structured Output Example

```json
{
  "risk_score": 75,
  "verdict": "SUSPICIOUS",
  "recommended_action": "ALERT",
  "reasons": [
    "The email contains suspicious indicators"
  ],
  "url_assessment": [
    "URLs were analyzed by VirusTotal"
  ],
  "attachment_assessment": [
    "Attachments were analyzed by VirusTotal"
  ]
}
```

---

## 3. AI Decision Flow

```text
                    Incoming Email
                          │
                          ▼
              AI Agent #1
          Email Content Extraction
                          │
                          ▼
              Structured Email Data
                          │
              ┌───────────┴───────────┐
              ▼                       ▼
             URLs                Attachments
              │                       │
              ▼                       ▼
         VirusTotal              VirusTotal
              │                       │
              └───────────┬───────────┘
                          ▼
                    AI Agent #2
                    SOC Analyst
                          │
                          ▼
                    Risk Assessment
                          │
                          ▼
                       Verdict
             ┌────────────┼────────────┐
             ▼            ▼            ▼
         MALICIOUS    SUSPICIOUS      CLEAN
```

---

## 4. Design Principles

### Structured Output

Both AI stages use structured JSON output so downstream n8n nodes can reliably access individual fields.

### Separation of Responsibilities

AI Agent #1 focuses on **email extraction**.

AI Agent #2 focuses on **security analysis and classification**.

### No Fabricated Intelligence

The SOC Analyst is instructed not to invent VirusTotal findings or other threat-intelligence information.

### Deterministic Routing

The final verdict is restricted to:

```text
MALICIOUS
SUSPICIOUS
CLEAN
```

This allows the n8n Switch node to route the result into the appropriate response branch.

---

## 5. Current Scope

The AI agents currently analyze:

* Email content
* Sender and recipient information
* URLs
* VirusTotal URL analysis
* Attachments
* VirusTotal file analysis
* Suspicious indicators
* Overall risk

SPF/DKIM verification is **not implemented** in the current workflow.

````

### `docs/integrations.md`

```markdown
# Integrations

The Automated Phishing Detection Workflow connects n8n with multiple external services for email ingestion, threat intelligence, AI analysis, notifications, incident tracking, and event logging.

---

## Integration Architecture

```text
                         ┌─────────────────┐
                         │      Gmail      │
                         │ Email Ingestion │
                         └────────┬────────┘
                                  │
                                  ▼
                         ┌─────────────────┐
                         │      n8n        │
                         │  Orchestration  │
                         └────────┬────────┘
                                  │
             ┌────────────────────┼────────────────────┐
             ▼                    ▼                    ▼
        Google Gemini        VirusTotal             Gmail
         AI Analysis       Threat Intel         Quarantine
             │                    │                    │
             └────────────────────┼────────────────────┘
                                  │
                                  ▼
                            SOC Decision
                                  │
                    ┌─────────────┼─────────────┐
                    ▼             ▼             ▼
                  Slack         Jira         Supabase
                Alerting     Incident Task    Logging
````

---

## 1. Gmail

### Purpose

Gmail is the email source for the workflow.

The Gmail Trigger starts the automation when a new email is received.

### Usage

Gmail is used for:

* Detecting incoming emails.
* Retrieving complete email information.
* Reading the subject, sender, recipient, and body.
* Accessing email attachments.
* Obtaining the Gmail message ID.
* Obtaining the Gmail thread ID.
* Moving malicious email threads to Trash.

### Main Flow

```text
Gmail Trigger
     ↓
Get a message
     ↓
Email Processing
```

For malicious emails:

```text
MALICIOUS
     ↓
Gmail Thread Trash
```

---

## 2. Google Gemini

### Purpose

Google Gemini provides the AI capabilities used by the workflow.

Gemini is used by both AI agents.

### AI Agent #1

**Role:** Email Content Extraction

Extracts:

* Subject
* Sender
* Recipient
* Body
* URLs
* Attachment names
* Suspicious indicators

### AI Agent #2

**Role:** SOC Analyst

Produces:

* Risk score
* Verdict
* Recommended action
* Reasons
* URL assessment
* Attachment assessment

### Verdicts

```text
MALICIOUS
SUSPICIOUS
CLEAN
```

### Recommended Actions

```text
QUARANTINE
ALERT
ALLOW
```

---

## 3. VirusTotal

### Purpose

VirusTotal provides threat-intelligence analysis for URLs and email attachments.

The workflow uses separate processing paths for URLs and files.

### URL Analysis

```text
Extracted URL
     ↓
VirusTotal URL Submission
     ↓
Analysis ID
     ↓
VirusTotal Analysis Result
     ↓
Aggregate URL Results
```

The URL is submitted using:

```text
POST /api/v3/urls
```

The analysis result is retrieved using:

```text
GET /api/v3/analyses/{analysis_id}
```

Multiple URLs are processed individually and aggregated before reaching the SOC Analyst.

### File Analysis

```text
Attachment
     ↓
VirusTotal File Upload
     ↓
Wait
     ↓
VirusTotal File Analysis Result
     ↓
Aggregate File Results
```

The file upload endpoint is:

```text
POST /api/v3/files
```

The workflow allows time for VirusTotal to process the submitted file before retrieving its analysis result.

### Important Behavior

VirusTotal may return:

```text
409 AlreadySubmittedError
```

when the same file has already been submitted.

This means the file has already been submitted to VirusTotal and does not necessarily indicate an invalid API credential.

---

## 4. Slack

### Purpose

Slack provides real-time security notifications for analyst awareness.

Slack notifications are generated for:

* `MALICIOUS`
* `SUSPICIOUS`

CLEAN emails do not generate Slack alerts.

### MALICIOUS Alert

The alert contains:

* Subject
* Sender
* Risk score
* Verdict
* Recommended action
* Reasons
* Confirmation that the email was quarantined

Example:

```text
🚨 TriageForge Security Alert

Malicious email detected.

Subject: [email subject]
Sender: [sender]

Risk Score: [score]/100
Verdict: MALICIOUS

Recommended Action: QUARANTINE

Reasons:
[reasons]

Action Taken: Email quarantined.
```

---

## 5. Jira

### Purpose

Jira is used to create a Task for emails requiring analyst attention.

Jira issues are created for:

* `MALICIOUS`
* `SUSPICIOUS`

CLEAN emails do not create Jira issues.

### MALICIOUS Issue

The issue contains:

* Email details
* Risk score
* Verdict
* Recommended action
* SOC reasons
* URL assessment
* Attachment assessment
* Automated response

Current summary format:

```text
[TriageForge] Malicious Email Detected - [Subject]
```

### SUSPICIOUS Issue

Suspicious emails generate a Jira Task containing the relevant email and SOC assessment information.

---

## 6. Supabase

### Purpose

Supabase provides persistent storage for security events generated by the workflow.

The workflow stores events in:

```text
security_events
```

### Stored Information

| Field                    | Purpose                   |
| ------------------------ | ------------------------- |
| `id`                     | Unique event identifier   |
| `email_message_id`       | Gmail message identifier  |
| `subject`                | Email subject             |
| `sender`                 | Sender address            |
| `recipient`              | Recipient address         |
| `email_body`             | Email body                |
| `risk_score`             | SOC risk score            |
| `verdict`                | Final classification      |
| `recommended_action`     | AI recommended response   |
| `reasons`                | SOC reasoning             |
| `url_assessments`        | URL intelligence          |
| `attachment_assessments` | File intelligence         |
| `action_taken`           | Actual automated response |
| `created_at`             | Event creation timestamp  |

### Action Values

CLEAN:

```text
ALLOWED
```

SUSPICIOUS:

```text
ALERTED_AND_TICKET_CREATED
```

MALICIOUS:

```text
QUARANTINED
```

The `created_at` timestamp is generated automatically by Supabase.

---

## 7. n8n

### Purpose

n8n is the central orchestration platform.

It coordinates:

* Gmail ingestion
* Email retrieval
* AI processing
* URL extraction
* Attachment processing
* VirusTotal analysis
* Data aggregation
* SOC classification
* Conditional routing
* Gmail quarantine
* Slack notifications
* Jira ticket creation
* Supabase logging

### High-Level Flow

```text
Gmail
  ↓
n8n
  ↓
AI Extraction
  ↓
VirusTotal
  ↓
SOC Analyst
  ↓
Switch
  ├── MALICIOUS → Gmail + Slack + Jira + Supabase
  ├── SUSPICIOUS → Slack + Jira + Supabase
  └── CLEAN → Supabase
```

---

## 8. Complete Integration Data Flow

```text
1. Gmail receives an email.

2. Gmail Trigger starts the n8n workflow.

3. n8n retrieves the complete email.

4. Gemini AI Agent #1 extracts:
   - Email information
   - URLs
   - Attachments
   - Suspicious indicators

5. URLs are submitted to VirusTotal.

6. Email attachments are submitted to VirusTotal when present.

7. VirusTotal analysis results are collected and aggregated.

8. Gemini AI Agent #2 acts as the SOC Analyst.

9. The SOC Analyst produces:
   - Risk score
   - Verdict
   - Recommended action
   - Reasons
   - URL assessment
   - Attachment assessment

10. n8n Switch routes the verdict.

11. MALICIOUS emails:
    - Move to Gmail Trash
    - Generate Slack alert
    - Create Jira Task
    - Log the event in Supabase

12. SUSPICIOUS emails:
    - Generate Slack alert
    - Create Jira Task
    - Log the event in Supabase

13. CLEAN emails:
    - Remain in the inbox
    - Are logged in Supabase
```

---

## 9. Credentials and Security

Credentials should be stored using n8n's credential system rather than directly inside workflow logic or documentation.

The following must never be committed to GitHub:

* Gmail credentials
* Gemini API keys
* VirusTotal API keys
* Slack credentials
* Jira API tokens
* Supabase secrets
* Passwords
* OAuth secrets

The exported n8n workflow should be checked before committing to ensure sensitive credential values are not exposed.

---

