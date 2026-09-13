# Automated Phishing Detection Workflow

An enterprise-grade automated phishing detection and response system powered by AI and threat intelligence. This workflow uses n8n orchestration with Google Gemini AI agents and VirusTotal analysis to detect, classify, and respond to phishing emails in real-time.

---

## Table of Contents

- [Overview](#overview)
- [Key Features](#key-features)
- [Architecture](#architecture)
- [System Components](#system-components)
- [AI Agents](#ai-agents)
  - [Agent #1: Email Content Extraction](#agent-1-email-content-extraction)
  - [Agent #2: SOC Analyst](#agent-2-soc-analyst)
- [Integration Stack](#integration-stack)
- [Security Classifications](#security-classifications)
- [Data Flow](#data-flow)
- [Configuration](#configuration)
- [Security Considerations](#security-considerations)
- [Documentation](#documentation)

---

## Overview

The Automated Phishing Detection Workflow provides a complete solution for identifying and responding to phishing threats. It combines:

- Automated email analysis using AI
- Threat intelligence from VirusTotal
- Intelligent security classification
- Automated response actions
- Real-time alerting and incident tracking
- Comprehensive event logging

This system is designed to operate continuously on incoming emails, reducing response time and human analyst burden while maintaining accuracy and reducing false positives.

---

## Key Features

**Automated Email Processing**
- Real-time email ingestion from Gmail
- Intelligent extraction of email metadata and content
- URL and attachment identification

**AI-Powered Threat Detection**
- Two-stage AI pipeline using Google Gemini
- Email content analysis and extraction
- Security assessment and risk scoring
- Phishing pattern recognition

**Threat Intelligence Integration**
- VirusTotal URL analysis
- File attachment scanning
- Reputation-based threat detection
- Multiple vendor detection consensus

**Intelligent Response Actions**
- Automated quarantine of malicious emails
- Real-time Slack alerts for security teams
- Automatic Jira ticket creation for incidents
- Persistent audit logging

**Comprehensive Audit Trail**
- Complete event logging to Supabase
- Traceability of all security decisions
- Historical analysis and reporting capability

---

## Architecture

```
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
           ┌──────────────────┼──────────────────┐
           ▼                  ▼                  ▼
      Google Gemini      VirusTotal           Gmail
       AI Analysis      Threat Intel         Quarantine
           │                  │                  │
           └──────────────────┼──────────────────┘
                              │
                              ▼
                        SOC Decision
                              │
                    ┌─────────┼─────────┐
                    ▼         ▼         ▼
                  Slack      Jira    Supabase
                 Alerting   Incident  Logging
                           Tracking
```

---

## System Components

| Component | Purpose | Role |
|-----------|---------|------|
| **Gmail** | Email Source | Ingests incoming emails and executes quarantine actions |
| **n8n** | Workflow Orchestration | Coordinates all system components and data flow |
| **Google Gemini** | AI Engine | Powers email extraction and security analysis |
| **VirusTotal** | Threat Intelligence | Analyzes URLs and file attachments |
| **Slack** | Alert Channel | Delivers real-time security notifications |
| **Jira** | Incident Management | Creates and tracks security incidents |
| **Supabase** | Event Storage | Maintains comprehensive audit logs |

---

## AI Agents

### Agent #1: Email Content Extraction

**Purpose:** Extracts structured, security-relevant information from incoming emails.

**Responsibilities:**
- Extract email subject
- Extract sender email address
- Extract recipient email address
- Extract complete email body
- Extract all URLs present in the email
- Extract attachment names
- Identify suspicious indicators
- Return structured JSON output

**Output Structure:**
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

**Design Principles:**
- No fabrication or modification of data
- Returns only available information
- Structured JSON output for reliable downstream processing
- Complete accuracy in extraction

---

### Agent #2: SOC Analyst

**Purpose:** Analyzes email content and threat intelligence data to produce final security classification.

**Responsibilities:**
- Analyze email subject, sender, recipient, and body
- Identify phishing and social-engineering indicators
- Evaluate VirusTotal URL analysis results
- Evaluate VirusTotal attachment analysis results
- Consider sender reputation and suspicious wording
- Generate risk score (0-100)
- Produce security verdict
- Recommend response action
- Provide reasoning for classification

**Output Structure:**
```json
{
  "risk_score": 0,
  "verdict": "CLEAN",
  "recommended_action": "ALLOW",
  "reasons": ["string"],
  "url_assessment": ["string"],
  "attachment_assessment": ["string"]
}
```

**Verdict Types:**
- **MALICIOUS:** Clear evidence of malicious content, malicious URLs, malicious attachments, or strong phishing indicators
- **SUSPICIOUS:** Concerning indicators exist but insufficient evidence for malicious classification
- **CLEAN:** No meaningful malicious or suspicious indicators identified

**Recommended Actions:**
- **QUARANTINE:** Isolate email immediately
- **ALERT:** Flag for analyst review
- **ALLOW:** Permit email delivery

---

## Integration Stack

### Gmail Integration

**Email Ingestion:** Gmail Trigger initiates workflow on new email arrival

**Email Processing:** Retrieves complete email including:
- Subject and body content
- Sender and recipient addresses
- Attachments and metadata
- Message and thread IDs

**Response Actions:**
- Move malicious emails to Trash
- Maintain audit trail of actions

**Flow:**
```
Gmail Trigger → Get Message → Email Processing
                                      ↓
                            (for MALICIOUS)
                                      ↓
                            Gmail Thread Trash
```

---

### Google Gemini Integration

**AI Agent #1 (Extraction):** Processes raw email data and produces structured extraction

**AI Agent #2 (Analysis):** Receives structured data and threat intelligence, produces classification

**Model:** Google Gemini Chat Model

**Features:**
- Structured JSON output
- Deterministic routing based on verdicts
- No hallucinated intelligence
- Consistent classification

---

### VirusTotal Integration

**URL Analysis Pipeline:**
```
Extracted URL
    ↓
URL Submission (POST /api/v3/urls)
    ↓
Analysis ID Assignment
    ↓
Result Retrieval (GET /api/v3/analyses/{analysis_id})
    ↓
Aggregation for SOC Review
```

**File Analysis Pipeline:**
```
Email Attachment
    ↓
File Upload (POST /api/v3/files)
    ↓
Processing Wait Period
    ↓
Result Retrieval
    ↓
Aggregation for SOC Review
```

**Error Handling:**
- 409 AlreadySubmittedError indicates previous submission (not a credential error)
- Workflow gracefully handles duplicate submissions
- Results are aggregated before SOC analyst stage

---

### Slack Integration

**Alert Generation:** Triggered for MALICIOUS and SUSPICIOUS emails

**Alert Contents:**
- Email subject and sender
- Risk score (0-100)
- Security verdict
- Recommended action
- Supporting reasons
- Automated response confirmation

**Clean Emails:** No alert generated

---

### Jira Integration

**Ticket Creation:** Triggered for MALICIOUS and SUSPICIOUS emails

**Ticket Contents:**
- Email details and headers
- Risk score and verdict
- Recommended action
- SOC analyst reasoning
- URL assessment results
- Attachment assessment results
- Automated response taken

**Summary Format:**
```
[TriageForge] Malicious Email Detected - [Subject]
```

**Clean Emails:** No ticket created

---

### Supabase Integration

**Event Storage Table:** `security_events`

**Stored Fields:**

| Field | Type | Purpose |
|-------|------|---------|
| `id` | UUID | Unique event identifier |
| `email_message_id` | string | Gmail message ID |
| `subject` | string | Email subject |
| `sender` | string | Sender address |
| `recipient` | string | Recipient address |
| `email_body` | text | Complete email body |
| `risk_score` | integer | SOC risk assessment (0-100) |
| `verdict` | enum | Classification (CLEAN, SUSPICIOUS, MALICIOUS) |
| `recommended_action` | enum | Recommended response (ALLOW, ALERT, QUARANTINE) |
| `reasons` | array | SOC reasoning for classification |
| `url_assessments` | array | VirusTotal URL analysis results |
| `attachment_assessments` | array | VirusTotal file analysis results |
| `action_taken` | string | Actual automated response executed |
| `created_at` | timestamp | Event creation time (auto-generated) |

**Action Values:**
- **CLEAN:** `ALLOWED`
- **SUSPICIOUS:** `ALERTED_AND_TICKET_CREATED`
- **MALICIOUS:** `QUARANTINED`

---

## Security Classifications

### CLEAN Emails

**Handling:**
- Remain in inbox
- Logged to Supabase
- No alerts or tickets generated

### SUSPICIOUS Emails

**Handling:**
- Generate Slack alert
- Create Jira ticket
- Log to Supabase
- Remain in inbox for analyst review

**Example Alert:** "Email contains suspicious indicators requiring analyst attention"

### MALICIOUS Emails

**Handling:**
- Move to Gmail Trash
- Generate Slack alert
- Create Jira ticket
- Log to Supabase

**Example Alert:** "Malicious email detected and quarantined automatically"

---

## Data Flow

### Complete Processing Pipeline

```
1. Gmail receives an email
    ↓
2. Gmail Trigger starts n8n workflow
    ↓
3. n8n retrieves complete email message
    ↓
4. Gemini AI Agent #1 extracts:
   - Email information
   - URLs
   - Attachments
   - Suspicious indicators
    ↓
5. URLs submitted to VirusTotal
    ↓
6. Attachments submitted to VirusTotal (if present)
    ↓
7. VirusTotal analysis results collected and aggregated
    ↓
8. Gemini AI Agent #2 (SOC Analyst) receives:
   - Extracted email data
   - URL threat intelligence
   - File threat intelligence
    ↓
9. SOC Analyst produces:
   - Risk score (0-100)
   - Verdict (MALICIOUS, SUSPICIOUS, CLEAN)
   - Recommended action (QUARANTINE, ALERT, ALLOW)
   - Reasons for classification
   - URL assessments
   - Attachment assessments
    ↓
10. n8n Switch routes based on verdict
    ↓
11a. MALICIOUS Path:
     - Move email to Gmail Trash
     - Send Slack alert
     - Create Jira task
     - Log event to Supabase
    ↓
11b. SUSPICIOUS Path:
     - Send Slack alert
     - Create Jira task
     - Log event to Supabase
    ↓
11c. CLEAN Path:
     - Log event to Supabase only
```

---

## Configuration

### Prerequisites

Before deploying this workflow, ensure you have:

**Gmail:**
- Gmail account with API access enabled
- OAuth2 authentication configured
- Permissions for reading, modifying, and deleting messages

**Google Gemini:**
- Google Cloud project with Gemini API enabled
- Valid API key
- Appropriate rate limits configured

**VirusTotal:**
- Active VirusTotal account
- Valid API key
- Sufficient API quota for URL and file submissions

**Slack:**
- Slack workspace administrator access
- Bot token with message posting permissions
- Target channel configured

**Jira:**
- Jira Cloud or Server instance
- API token or OAuth2 credentials
- Appropriate project and issue type permissions

**Supabase:**
- Supabase project initialized
- `security_events` table created with documented schema
- Database connection string

**n8n:**
- n8n instance deployed and accessible
- All external credentials configured in n8n credential management
- Webhook URL available for Gmail trigger (if using webhooks)

### Credential Management

All credentials must be stored in n8n's credential system:

1. Navigate to n8n Credentials panel
2. Create credential entries for:
   - Gmail OAuth2
   - Google Gemini API
   - VirusTotal API
   - Slack Bot Token
   - Jira API Token
   - Supabase Connection String

3. Reference credentials by name in workflow nodes

**Critical:** Never hardcode credentials in exported workflows. Verify before GitHub commits.

### Deployment Checklist

- [ ] Gmail credentials configured and tested
- [ ] Gemini API key verified and quota checked
- [ ] VirusTotal API key validated
- [ ] Slack webhook verified and channel selected
- [ ] Jira project and issue type configured
- [ ] Supabase table schema created
- [ ] n8n workflow imported and credentials linked
- [ ] Test email processed end-to-end
- [ ] Alerts received in Slack
- [ ] Jira ticket created successfully
- [ ] Event logged to Supabase
- [ ] Workflow monitoring and alerting configured

---

## Security Considerations

### Credential Security

The following must NEVER be committed to version control:

- Gmail credentials and OAuth tokens
- Google Gemini API keys
- VirusTotal API keys
- Slack credentials and tokens
- Jira API tokens
- Supabase secrets and connection strings
- Database passwords
- OAuth secrets

**Best Practice:** Use n8n's credential management system exclusively. Before exporting workflows, verify that all credentials are properly referenced and not embedded in export files.

### Data Privacy

- Email bodies and attachments are processed for threat analysis
- Personally identifiable information (PII) is handled per organizational policy
- All events are logged with timestamps for audit purposes
- Access to Supabase security_events table should be restricted to authorized personnel

### API Rate Limiting

- VirusTotal: Monitor URL and file submission limits
- Google Gemini: Track API quota and concurrent requests
- Gmail: Respect Gmail API quotas for label operations
- Slack: Configure webhook rate limiting appropriately

### Email Handling

- Malicious emails are moved to Trash, not permanently deleted
- Emails can be recovered from Trash within 30 days
- All email processing is logged for audit trail
- Original email content is preserved in Supabase logs

### False Positive Management

The SOC Analyst is designed to:
- Avoid false positives through multi-factor analysis
- Consider sender reputation and email authenticity
- Flag uncertain cases as SUSPICIOUS rather than MALICIOUS
- Provide detailed reasoning for all classifications

**Recommendation:** Monitor SUSPICIOUS classifications and adjust thresholds as needed.

---

## Documentation

### Repository Structure

```
Automated-Phishing-Detection-Workflow/
├── README.md                    # This file
├── LICENSE                      # MIT License
├── docs/
│   ├── prompts/
│   │   └── README.md           # AI Agent prompts documentation
│   └── integrations.md         # Integration architecture
├── n8n/
│   └── workflow.json           # Exported n8n workflow
├── database/
│   └── schema.sql              # Supabase table schema
└── assets/
    └── [architecture diagrams]
```

### Key Documentation Files

- **`docs/prompts/README.md`** - Detailed AI Agent specifications and prompts
- **`docs/integrations.md`** - Complete integration architecture and data flows
- **`n8n/workflow.json`** - Importable n8n workflow configuration

### Additional Resources

For detailed information on:

- **AI Agent Implementation** - See `docs/prompts/README.md`
- **Integration Details** - See `docs/integrations.md`
- **Workflow Configuration** - See `n8n/workflow.json`
- **Database Schema** - See `database/schema.sql`

---

## Current Capabilities

The workflow currently analyzes:

- Email subject and body content
- Sender and recipient information
- URLs and hyperlinks
- Email attachments
- VirusTotal URL reputation
- VirusTotal file reputation
- Phishing and social engineering indicators
- Overall risk assessment

### Planned Enhancements

- SPF/DKIM signature verification
- DMARC policy evaluation
- Header analysis and spoofing detection
- Machine learning-based pattern recognition
- Custom threat intelligence feeds
- Advanced attachment sandboxing

---

## License

This project is licensed under the MIT License. See LICENSE file for details.

---

## Support and Contribution

For issues, questions, or contributions:

1. Review existing documentation in `docs/`
2. Check n8n workflow logs for error details
3. Verify all credentials are properly configured
4. Consult Supabase logs for event history

For questions about specific integrations, refer to:
- Gmail API documentation
- Google Gemini API documentation
- VirusTotal API documentation
- n8n documentation
- Slack API documentation
- Jira API documentation
- Supabase documentation

---

## Important Notes

**VirusTotal 409 Errors:** A 409 AlreadySubmittedError from VirusTotal indicates the file was previously submitted and is not necessarily a credential error.

**Email Extraction Principles:** AI Agent #1 is designed to extract only factual information. It does not invent or modify data; empty fields are returned as empty strings or arrays.

**Deterministic Routing:** The three-verdict model (MALICIOUS, SUSPICIOUS, CLEAN) ensures consistent, predictable response routing through n8n Switch nodes.

**No Hallucinated Intelligence:** The SOC Analyst is explicitly instructed never to fabricate threat intelligence findings. All assessments are based on provided data only.

---

Last Updated: September 2026  
Repository: [Automated-Phishing-Detection-Workflow](https://github.com/ChaudhariParthh/Automated-Phishing-Detection-Workflow)
