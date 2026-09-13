````markdown
# Setup Guide

## Automated Phishing Detection Workflow

This guide explains how to configure the services and credentials required to run the Automated Phishing Detection Workflow in n8n.

---

## 1. Prerequisites

Before importing the workflow, make sure you have access to:

- n8n
- Gmail account
- Google Gemini API access
- VirusTotal API access
- Slack workspace
- Jira Cloud project
- Supabase project

The workflow uses these services for email ingestion, AI analysis, threat-intelligence enrichment, automated notifications, ticket creation, and event logging.

---

## 2. n8n Setup

Create or open an n8n workspace.

Import the workflow file located at:

```text
n8n/Automated Phishing Detection Workflow.json
````

After importing the workflow, review every credential reference and replace it with credentials belonging to your own accounts.

---

## 3. Gmail Setup

### Purpose

Gmail is used to:

1. Detect newly received emails.
2. Retrieve complete email messages.
3. Access downloaded attachments.
4. Move malicious email threads to Trash.

### Credential

Create/connect a Gmail credential in n8n.

The Gmail account must have permission to:

* Read email messages
* Access attachments
* Modify email threads

### Gmail Trigger

The Gmail Trigger starts the workflow when a new email is received.

Attachments are downloaded by the trigger so they can be passed to the VirusTotal file-analysis branch.

---

## 4. Google Gemini Setup

Google Gemini is used by two AI Agents.

### AI Agent 1 — Email Content Extraction

The first Gemini Agent extracts:

```text
subject
sender
recipient
body
urls
attachment_names
suspicious_indicators
```

A Structured Output Parser is connected to ensure the result follows the expected JSON structure.

---

### AI Agent 2 — SOC Analyst

The second Gemini Agent evaluates the complete email and available VirusTotal results.

It generates:

```text
risk_score
verdict
recommended_action
reasons
url_assessment
attachment_assessment
```

The supported verdicts are:

```text
MALICIOUS
SUSPICIOUS
CLEAN
```

The supported recommended actions are:

```text
QUARANTINE
ALERT
ALLOW
```

---

## 5. VirusTotal Setup

### Purpose

VirusTotal provides threat-intelligence analysis for:

* URLs extracted from emails
* Email attachments

Create a VirusTotal API credential and configure the required API authentication in the HTTP Request nodes.

### URL Analysis

The URL branch submits extracted URLs to:

```text
POST https://www.virustotal.com/api/v3/urls
```

The returned analysis ID is then used to retrieve the analysis result.

### File Analysis

Email attachments are submitted to:

```text
POST https://www.virustotal.com/api/v3/files
```

The returned analysis ID is used to retrieve the file-analysis result.

### API Considerations

VirusTotal analysis can be asynchronous.

The workflow therefore includes a Wait node before retrieving analysis results.

The exact time required for an analysis can vary depending on VirusTotal processing and API availability.

---

## 6. Slack Setup

### Purpose

Slack is used to notify the security team when an email requires attention.

Slack notifications are generated for:

### MALICIOUS

A security alert is sent after a malicious verdict.

### SUSPICIOUS

A security alert is sent to notify the team that further review may be required.

Create/connect a Slack credential in n8n and select the security notification channel used by the workflow.

---

## 7. Jira Cloud Setup

### Purpose

Jira is used to create investigation tickets.

The workflow uses:

```text
Resource: Issue
Operation: Create
Issue Type: Task
```

The Jira Cloud credential requires:

* Atlassian account email
* Jira API token
* Jira Cloud domain

Example domain format:

```text
https://your-site.atlassian.net
```

The Jira project used by the workflow must allow the authenticated account to create issues.

### Malicious Jira Ticket

The malicious branch creates an investigation ticket containing information such as:

* Email subject
* Sender
* Recipient
* Risk score
* Verdict
* Recommended action
* AI-generated reasons
* URL assessment
* Attachment assessment

### Suspicious Jira Ticket

The suspicious branch also creates a Jira Task so that the security team can investigate the email.

---

## 8. Supabase Setup

### Purpose

Supabase provides persistent security-event logging.

Create a Supabase project and connect the Supabase credential in n8n.

The workflow writes events to:

```text
security_events
```

### Security Events Table

The table contains:

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

The `created_at` field is generated automatically by the database.

### Action Values

#### MALICIOUS

```text
QUARANTINED
```

#### SUSPICIOUS

```text
ALERTED_AND_TICKET_CREATED
```

#### CLEAN

```text
ALLOWED
```

---

## 9. Credential Security

Never commit real credentials or API keys to GitHub.

Do not store the following in the repository:

```text
Gmail passwords
Gmail OAuth secrets
Gemini API keys
VirusTotal API keys
Slack tokens
Jira API tokens
Supabase service-role keys
.env files containing secrets
```

Use n8n's credential management system instead.

---

## 10. Workflow Import

After the required services and credentials are available:

1. Open n8n.
2. Import:

```text
n8n/Automated Phishing Detection Workflow.json
```

3. Open each node requiring authentication.
4. Select the appropriate credential.
5. Verify the configured Gmail, Gemini, VirusTotal, Slack, Jira, and Supabase connections.
6. Save the workflow.

---

## 11. Configuration Verification

Before testing the complete workflow, verify:

### Gmail

* Gmail Trigger is connected.
* Full message retrieval works.
* Attachments are downloaded.

### Gemini

* Both AI Agents have a Gemini model connected.
* Structured Output Parsers are connected.
* Both agents return the expected JSON structures.

### VirusTotal

* URL submission works.
* File submission works.
* Analysis results can be retrieved.

### Slack

* Security notification channel is configured.
* Message sending works.

### Jira

* Jira Cloud credential connects successfully.
* The selected project is accessible.
* The selected issue type is available.
* The account can create issues.

### Supabase

* Supabase credential connects successfully.
* `security_events` exists.
* The n8n credential has permission to insert rows.

---

## 12. Recommended Testing Order

The complete workflow should be tested using controlled test emails.

Test the following scenarios:

```text
1. CLEAN
2. SUSPICIOUS
3. MALICIOUS
4. Email with attachments
5. Email containing multiple URLs
```

Verify both the automated response and the Supabase audit record for each scenario.

Detailed testing procedures are documented separately in:

```text
docs/testing.md
```

---

