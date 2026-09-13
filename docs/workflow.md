````markdown
# Workflow Documentation

## Automated Phishing Detection Workflow

This document describes the n8n workflow and the purpose of each major node used in the Automated Phishing Detection Workflow.

---

## 1. Gmail Trigger

**Node:** Gmail Trigger

**Purpose:**  
Starts the workflow when a new email is received.

The trigger provides the initial Gmail message information and downloads email attachments for further processing.

### Key Configuration

- Gmail account connected through n8n credentials
- Attachment downloading enabled
- Attachment prefix configured as `attachment_`

---

## 2. Edit Fields

**Node:** Edit Fields

**Purpose:**  
Prepares the Gmail message data required by downstream nodes.

The Gmail message ID is retained so that the workflow can retrieve the complete message and reference the original email during response actions.

### Main Field

```text
id
{{ $json.id }}
````

---

## 3. Get a Message

**Node:** Gmail — Get a message

**Purpose:**
Retrieves the complete Gmail message using the message ID.

This provides the email information required for AI-based extraction and analysis.

---

# Email Content Extraction

## 4. AI Agent — Email Content Extraction

**Node:** AI Agent

**Model:** Google Gemini

**Purpose:**
Extracts security-relevant information from the email.

The agent produces structured output containing:

```text
subject
sender
recipient
body
urls
attachment_names
suspicious_indicators
```

### Structured Output

The agent uses a Structured Output Parser to maintain a consistent JSON structure.

Example:

```json
{
  "subject": "Security phishing mail check",
  "sender": "sender@example.com",
  "recipient": "recipient@example.com",
  "body": "Email body",
  "urls": [
    "https://example.com"
  ],
  "attachment_names": [],
  "suspicious_indicators": [
    "Contains generic external links"
  ]
}
```

---

# Attachment Analysis

## 5. IF — Attachment Check

**Node:** IF

**Purpose:**
Determines whether the incoming email contains attachments.

The workflow checks the number of binary attachment objects.

### Condition

```text
{{ Object.keys($binary || {}).length }}
```

The condition checks whether the number of attachments is greater than:

```text
0
```

If attachments exist, the workflow continues through the attachment-analysis branch.

---

## 6. Split Out — Attachments

**Node:** Split Out

**Purpose:**
Separates multiple email attachments into individual workflow items.

### Field

```text
$binary
```

This allows each attachment to be submitted to VirusTotal individually.

---

## 7. VirusTotal HTTP Request — File Upload

**Node:** HTTP Request

**Purpose:**
Uploads each email attachment to VirusTotal for security analysis.

### Configuration

```text
Method: POST

Endpoint:
https://www.virustotal.com/api/v3/files
```

The attachment is submitted as multipart form data using the n8n binary file.

VirusTotal returns an analysis identifier that is used to retrieve the analysis status and results.

---

## 8. Wait

**Node:** Wait

**Purpose:**
Provides time for VirusTotal to process the submitted file.

The workflow currently uses a time interval before requesting the analysis result.

---

## 9. VirusTotal — File Analysis Result

**Node:** HTTP Request

**Purpose:**
Retrieves the VirusTotal analysis result using the analysis ID returned by the upload request.

The result contains information such as:

* Analysis status
* Malicious detections
* Suspicious detections
* Harmless results
* Undetected results
* Other analysis statistics

---

## 10. Aggregate File Analysis Results

**Node:** Aggregate

**Purpose:**
Combines the individual attachment analysis results into a single collection.

The results are stored together so that the SOC Analyst can evaluate all analyzed attachments.

---

# URL Analysis

## 11. Split Out1 — URLs

**Node:** Split Out

**Purpose:**
Separates every URL extracted by the first AI Agent into an individual workflow item.

### Field

```text
output.urls
```

This allows every extracted URL to be analyzed independently.

---

## 12. VirusTotal — URL Reputation

**Node:** HTTP Request

**Purpose:**
Submits each extracted URL to VirusTotal for reputation analysis.

### Configuration

```text
Method: POST

Endpoint:
https://www.virustotal.com/api/v3/urls
```

The URL is submitted as form data.

VirusTotal returns an analysis ID for the submitted URL.

---

## 13. VirusTotal — URL Analysis Result

**Node:** HTTP Request

**Purpose:**
Retrieves the VirusTotal analysis result for the submitted URL.

### Endpoint

```text
https://www.virustotal.com/api/v3/analyses/{{ $json["data"]["id"] }}
```

The workflow retrieves the analysis status and detection statistics.

---

## 14. Aggregate URL Results

**Node:** Aggregate

**Purpose:**
Combines all URL analysis results into a single collection.

The aggregated data is passed to the result-merging stage.

---

# Result Combination

## 15. Merge

**Node:** Merge

**Purpose:**
Combines the VirusTotal URL analysis results and VirusTotal attachment analysis results.

This creates a consolidated threat-intelligence dataset.

---

## 16. Merge1

**Node:** Merge1

**Purpose:**
Combines:

1. Email extraction results from the first AI Agent
2. Aggregated VirusTotal analysis results

The resulting dataset provides the SOC Analyst with the email context and available threat-intelligence information.

---

# AI SOC Analysis

## 17. AI Agent1 — SOC Analyst

**Node:** AI Agent1

**Model:** Google Gemini

**Purpose:**
Performs the final security assessment of the email.

The agent evaluates:

* Email content
* Sender
* Recipient
* Suspicious indicators
* VirusTotal URL results
* VirusTotal attachment results

### Output

The SOC Analyst produces:

```text
risk_score
verdict
recommended_action
reasons
url_assessment
attachment_assessment
```

### Verdict Values

The verdict is restricted to:

```text
MALICIOUS
SUSPICIOUS
CLEAN
```

### Recommended Actions

The recommended action is restricted to:

```text
QUARANTINE
ALERT
ALLOW
```

The `risk_score` represents the AI-generated risk assessment on a 0–100 scale.

---

## 18. Structured Output Parser1

**Node:** Structured Output Parser

**Purpose:**
Ensures that the SOC Analyst output follows the expected structure.

The parser produces a predictable JSON object that can be consumed by the Switch node.

---

# Verdict Routing

## 19. Switch

**Node:** Switch

**Mode:** Rules

**Purpose:**
Routes the workflow according to the SOC Analyst's verdict.

### Input

```text
{{ $json.output.verdict }}
```

### Routing Rules

```text
MALICIOUS
SUSPICIOUS
CLEAN
```

Each verdict follows a different response path.

---

# MALICIOUS Response

## 20. Gmail — Trash a Thread

**Purpose:**
Moves the malicious email thread to Gmail Trash.

The workflow uses the original Gmail thread ID:

```text
{{ $('Gmail Trigger').item.json.threadId }}
```

This provides a recoverable quarantine-style response rather than permanently deleting the message.

---

## 21. Slack — Send Message

**Purpose:**
Sends a security alert to the configured Slack security channel.

The alert includes information from the SOC Analyst such as:

* Subject
* Sender
* Risk score
* Verdict
* Recommended action
* Reasons

---

## 22. Jira — Create Issue

**Purpose:**
Creates a Jira investigation ticket for the malicious email.

The ticket contains the SOC Analyst findings and relevant email information.

### Issue Type

```text
Task
```

### Summary Format

```text
[TriageForge] Malicious Email Detected - <email subject>
```

The Jira ticket provides a persistent investigation record for the detected threat.

---

## 23. Supabase — Create Row

**Purpose:**
Records the malicious email event in the `security_events` table.

The event records the email information, AI assessment, threat-intelligence results, and response action.

### Action Taken

```text
QUARANTINED
```

---

# SUSPICIOUS Response

## 24. Slack — Send Message

**Purpose:**
Alerts the security team that an email requires attention.

The message contains the SOC Analyst assessment and relevant findings.

---

## 25. Jira — Create Issue

**Purpose:**
Creates a Jira Task for further investigation.

The ticket provides the security team with the email context and AI-generated findings.

---

## 26. Supabase — Create Row

**Purpose:**
Records the suspicious email event in the centralized security-event database.

### Action Taken

```text
ALERTED_AND_TICKET_CREATED
```

The email is not automatically quarantined by this branch.

---

# CLEAN Response

## 27. Supabase — Create Row

**Purpose:**
Records the clean email analysis for auditing.

No destructive Gmail action is performed.

The email remains in the inbox.

### Action Taken

```text
ALLOWED
```

---

# Database Logging

All three verdict paths eventually record an event in:

```text
security_events
```

The database stores:

```text
email_message_id
subject
sender
recipient
email_body
risk_score
verdict
recommended_action
reasons
url_assessments
attachment_assessments
action_taken
created_at
```

The `created_at` field is generated automatically by Supabase.

---

# End-to-End Data Flow

```text
Incoming Gmail Email
        │
        ▼
Get Full Message
        │
        ▼
AI Email Extraction
        │
        ├───────────────┐
        │               │
        ▼               ▼
     URLs          Attachments
        │               │
        ▼               ▼
   VirusTotal       VirusTotal
   URL Analysis     File Analysis
        │               │
        └───────┬───────┘
                ▼
             Merge
                │
                ▼
          SOC Analyst AI
                │
                ▼
             Switch
        ┌───────┼───────┐
        ▼       ▼       ▼
    Malicious Suspicious Clean
        │       │       │
        ▼       ▼       ▼
      Gmail   Slack   Supabase
        │       │
        ▼       ▼
      Slack   Jira
        │       │
        ▼       ▼
      Jira   Supabase
        │
        ▼
    Supabase
```

---

# External Integrations

| Integration   | Role                                                     |
| ------------- | -------------------------------------------------------- |
| Gmail         | Email trigger, retrieval, and malicious-email quarantine |
| Google Gemini | Email extraction and SOC analysis                        |
| VirusTotal    | URL and attachment threat intelligence                   |
| Slack         | Security notifications                                   |
| Jira          | Investigation ticket creation                            |
| Supabase      | Security-event audit logging                             |
| n8n           | Workflow orchestration                                   |

---

