Yes. We'll start with the **`docs/` documentation**, before touching the README.

We'll do it **one document at a time**, and every document will describe only what we actually built.

## Documentation — File 1

Create:

```text
docs/
└── architecture.md
```

Paste this into `architecture.md`:

````markdown
# System Architecture

## Automated Phishing Detection Workflow

The Automated Phishing Detection Workflow is an n8n-based cybersecurity automation pipeline designed to detect, analyze, classify, and respond to potentially malicious phishing emails.

The workflow combines Gmail, Google Gemini, VirusTotal, Slack, Jira, and Supabase to automate the email security investigation and response process.

---

## High-Level Architecture

```text
                         ┌──────────────────────┐
                         │     Gmail Trigger    │
                         │   New Email Received  │
                         └──────────┬───────────┘
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │    Edit Fields       │
                         │  Prepare Message ID  │
                         └──────────┬───────────┘
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │     Get a Message    │
                         │  Retrieve Full Email │
                         └──────────┬───────────┘
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │      AI Agent #1     │
                         │  Email Content       │
                         │     Extraction       │
                         └──────────┬───────────┘
                                    │
                   ┌────────────────┴────────────────┐
                   │                                 │
                   ▼                                 ▼
        ┌──────────────────────┐          ┌──────────────────────┐
        │      URL Branch      │          │   Attachment Branch │
        │                      │          │                      │
        │     Split Out        │          │         IF           │
        │         │            │          │         │            │
        │         ▼            │          │         ▼            │
        │  VirusTotal URL      │          │      Split Out       │
        │      Request         │          │         │            │
        │         │            │          │         ▼            │
        │         ▼            │          │  VirusTotal File     │
        │       Wait            │          │      Upload          │
        │         │            │          │         │            │
        │         ▼            │          │         ▼            │
        │ VirusTotal URL        │          │       Wait            │
        │ Analysis Result       │          │         │            │
        │         │            │          │         ▼            │
        │         ▼            │          │ VirusTotal File       │
        │ Aggregate URL        │          │ Analysis Result       │
        │ Results              │          │         │            │
        └─────────┬────────────┘          │         ▼            │
                  │                       │ Aggregate File       │
                  │                       │ Results              │
                  │                       └─────────┬────────────┘
                  │                                 │
                  └──────────────┬──────────────────┘
                                 │
                                 ▼
                       ┌──────────────────────┐
                       │        Merge         │
                       │ Combine VT Results  │
                       └──────────┬───────────┘
                                  │
                                  ▼
                       ┌──────────────────────┐
                       │       Merge1         │
                       │ Email + VT Results  │
                       └──────────┬───────────┘
                                  │
                                  ▼
                       ┌──────────────────────┐
                       │      AI Agent #2     │
                       │     SOC Analyst      │
                       │                      │
                       │ Google Gemini        │
                       │ + Structured Output  │
                       └──────────┬───────────┘
                                  │
                                  ▼
                       ┌──────────────────────┐
                       │       Switch         │
                       │    AI Verdict        │
                       └──────────┬───────────┘
                                  │
             ┌────────────────────┼────────────────────┐
             │                    │                    │
             ▼                    ▼                    ▼
       ┌─────────────┐      ┌─────────────┐      ┌─────────────┐
       │  MALICIOUS  │      │ SUSPICIOUS  │      │    CLEAN    │
       └──────┬──────┘      └──────┬──────┘      └──────┬──────┘
              │                    │                    │
              ▼                    ▼                    ▼
       Gmail Trash            Slack Alert          Supabase
              │                    │                 Log Event
              ▼                    ▼
        Slack Alert            Jira Ticket
              │                    │
              ▼                    ▼
        Jira Ticket           Supabase
              │               Log Event
              ▼
          Supabase
          Log Event
````

---

## Workflow Components

### 1. Gmail Trigger

The workflow starts when a new email is received in the monitored Gmail account.

The Gmail Trigger provides the initial email information and downloaded attachments.

---

### 2. Get Full Email

The workflow retrieves the complete Gmail message using the message ID.

This provides the information required for downstream analysis, including:

* Subject
* Sender
* Recipient
* Email body
* URLs
* Attachments
* Message metadata

---

### 3. AI Agent #1 — Email Content Extraction

Google Gemini analyzes the email and extracts security-relevant information.

The agent produces structured data containing:

* Subject
* Sender
* Recipient
* Email body
* URLs
* Attachment names
* Suspicious indicators

The agent uses a Structured Output Parser to ensure the extracted information follows a predictable JSON structure.

---

### 4. VirusTotal URL Analysis

URLs extracted from the email are separated into individual items.

Each URL is submitted to VirusTotal for analysis.

The workflow then retrieves the corresponding VirusTotal analysis result and aggregates the results before sending them to the SOC Analyst.

---

### 5. VirusTotal Attachment Analysis

The workflow checks whether the incoming email contains attachments.

If attachments are present:

1. Attachments are separated into individual items.
2. Each file is submitted to VirusTotal.
3. The workflow waits for the analysis.
4. The analysis result is retrieved.
5. File analysis results are aggregated.

This allows the SOC Analyst to consider attachment reputation as part of the final assessment.

---

### 6. Result Aggregation

The URL and attachment analysis results are combined with the extracted email information.

The merged dataset provides the SOC Analyst with a complete view of the email and available threat-intelligence results.

---

### 7. AI Agent #2 — SOC Analyst

The second Google Gemini AI Agent acts as the SOC Analyst.

It evaluates:

* Email content
* Sender and recipient information
* Suspicious indicators
* VirusTotal URL results
* VirusTotal attachment results

The agent produces:

```text
Risk Score
Verdict
Recommended Action
Reasons
URL Assessment
Attachment Assessment
```

The verdict is restricted to:

```text
MALICIOUS
SUSPICIOUS
CLEAN
```

The recommended action is restricted to:

```text
QUARANTINE
ALERT
ALLOW
```

A Structured Output Parser ensures that the SOC Analyst response follows the expected JSON format.

---

## 8. Verdict-Based Routing

A Switch node routes the SOC Analyst result according to the generated verdict.

### MALICIOUS

The malicious branch performs:

```text
Gmail → Trash Thread
       ↓
Slack → Security Alert
       ↓
Jira → Create Issue
       ↓
Supabase → Create Security Event
```

The email is moved to the Gmail trash, the security team is notified through Slack, a Jira investigation ticket is created, and the event is recorded in Supabase.

---

### SUSPICIOUS

The suspicious branch performs:

```text
Slack → Security Alert
       ↓
Jira → Create Issue
       ↓
Supabase → Create Security Event
```

The email remains available for review while the security team receives an alert and a Jira investigation ticket is created.

---

### CLEAN

The clean branch performs:

```text
Supabase → Create Security Event
```

No destructive Gmail action is performed. The email remains in the inbox and the analysis result is recorded for auditing.

---

## 9. Supabase Security Event Logging

All verdict paths are recorded in the Supabase `security_events` table.

The table stores:

* Email message ID
* Subject
* Sender
* Recipient
* Email body
* Risk score
* Verdict
* Recommended action
* AI reasoning
* URL assessments
* Attachment assessments
* Action taken
* Event timestamp

This provides a persistent audit trail of processed email security events.

---

## Technology Flow

```text
Gmail
  ↓
n8n
  ↓
Google Gemini
  ↓
VirusTotal
  ↓
Google Gemini SOC Analysis
  ↓
Switch
  ├── MALICIOUS → Gmail + Slack + Jira + Supabase
  ├── SUSPICIOUS → Slack + Jira + Supabase
  └── CLEAN → Supabase
```

---

## Design Objective

The workflow is designed to reduce the amount of manual effort required to investigate phishing emails by combining automated extraction, threat-intelligence enrichment, AI-based classification, automated response actions, and centralized security-event logging.

```

