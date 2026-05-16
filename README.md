# 🤖 Imployii — Automated Recruitment Outreach System

> AI-powered, multi-channel recruitment outreach pipeline built on **n8n** — fully automated lead management for employer acquisition.
> 
> A part of demo screenshot: 👇🏿
<img width="1612" height="659" alt="image" src="https://github.com/user-attachments/assets/de7433c4-f931-427d-9f03-57d53dcdde23" />

---

## 📋 Overview

Imployii is a production-grade automated recruitment outreach system that reads employer leads from a Google Sheets database and autonomously contacts them via **Phone (VAPI AI)**, **Email (Microsoft Outlook)**, **WhatsApp**, and **Telegram** — then writes all results back to the sheet.

The AI phone agent, **Elena**, speaks fluent Russian and handles the full conversation lifecycle: qualification, objection handling, job requirement extraction, and CV delivery.

---

## ✨ Features

- 📞 **AI Voice Calls** — VAPI-powered Russian-speaking agent (Elena) with dynamic prompts
- 📧 **14-Step Email Sequences** — Progressive follow-up with escalating tone strategies
- 💬 **WhatsApp Integration** — Via CodingMantra + UltraMSC
- ✈️ **Telegram Integration** — Custom-built Telethon + FastAPI server
- 📱 **SMS Integration** — Fully integrated SMS outreach channel with delivery tracking
- 🧠 **GPT-5 Powered** — AI-generated contextual openers for follow-up calls
- 📄 **Automated CV Sending** — Candidate matching by sector + Google Drive download + email
- 🔄 **Smart Lead Eligibility Engine** — Cooling periods, status checks, attempt tracking
- 📊 **Google Sheets as CRM** — Full read/write sync after every interaction
- 📥 **Automated Lead Import Pipeline** — Raw leads from external sheet auto-processed into main engine with ID assignment and deduplication

---

## 🏗️ Architecture

The system consists of **two independent workflow chains**:

```
CHAIN A — Outbound Outreach (Scheduled: 15:50 daily)
──────────────────────────────────────────────────
Schedule Trigger → Read Google Sheet → Loop Over Leads
  └── Eligibility Check → Route by Channel
        ├── Phone   → Build VAPI Prompt → AI Call
        ├── Email   → Select Template   → Send via Outlook
        ├── WhatsApp → CodingMantra API
        ├── Telegram → Telethon FastAPI Server
        └── SMS      → SMS Gateway API

CHAIN B — Post-Call Processing (Webhook Triggered)
──────────────────────────────────────────────────
VAPI Webhook → Parse Transcript → Update Google Sheet
  └── If CV Needed → Match Candidates → Download from Drive → Email CVs

CHAIN C — Lead Import Pipeline (Scheduled: 17:08 daily)
──────────────────────────────────────────────────────
Schedule Trigger → Get All Existing Leads (find max Lead ID)
  → Read Raw Leads (Lead Management sheet)
  → Set Daily Loop Limit → Loop Over New Leads
      └── Assign Lead ID (maxId + loop_index)
          └── Normalize Phone Number
              ├── Has Phone → Append to Leads Engine (Contact Method: Not selected)
              └── No Phone  → Append to Leads Engine (Contact Method: Email)
          → Archive to "Old Leads data" tab → Wait 10s → Next Lead
```

---

## 🛠️ Tech Stack

| Technology | Purpose |
|---|---|
| **n8n (Self-Hosted)** | Workflow automation engine |
| **Google Sheets** | Master database (Leads, Candidates, Placements) |
| **VAPI AI** | AI voice call platform |
| **OpenAI GPT-5** | Dynamic first-message generation for follow-up calls |
| **Microsoft Outlook** | Email outreach via `elena@imployii.com` |
| **Telethon + FastAPI** | Custom Telegram messaging server |
| **CodingMantra + UltraMSC** | WhatsApp integration |
| **SMS Gateway** | SMS outreach channel integration |
| **Google Drive** | CV and passport file storage |

---

## 📂 Google Sheets Data Model

**Spreadsheet ID:** `1TbaLtv0SX2-8n--sAD5ltjx12IUMC3MXT0b6LqnSTnw`

| Tab | Description |
|---|---|
| `Leads Engine` (gid=0) | Main leads database — all contacts, statuses, call history |
| `Candidates Available` (gid=247118444) | Available candidate profiles with CV links |
| `Placements` (gid=711664441) | Qualified leads and active placement pipeline |
| `Old Leads data` (gid=872591740) | Archive of processed raw leads (written by Import Pipeline) |

**Lead Management Spreadsheet** (separate file, ID: `1lCIhwYIHss2THItaeOubZz9ZqlqepG5DuNZUK3BWWrg`)

| Tab | Description |
|---|---|
| `Raw leads` (gid=0) | Source of new leads — manually populated or externally fed |

---

## 🔄 Workflow Details

### Lead Eligibility Engine

Every lead passes through a JS eligibility check before contact:

| Skip Reason | Condition |
|---|---|
| `NO_LEAD_ID` | Lead ID is blank |
| `STATUS_CLOSED` | Status is DISQUALIFIED, CLOSED, or QUALIFIED |
| `IN_COOLING` | Today is before the Cooling Until date |
| `MAX_ATTEMPTS` | Attempt count reached safety ceiling |

### Email Sequence (14 Steps)

Progressive follow-up with increasing gaps (2 days → 110 days), all written in professional Russian. Leads are automatically deleted after 15 failed attempts.

### CV Matching Logic

- Matches candidate `Sector` field against employer's `Job Sector From Call`
- Candidates with `Sector = "Any"` are always included
- Downloads CV PDFs from Google Drive and attaches them to email

### SMS Integration

SMS is fully integrated as a native outreach channel alongside Phone, Email, WhatsApp, and Telegram. When a lead's `Preferred Contact Method` is set to `SMS`, the system routes to the SMS branch and sends a message via the SMS Gateway API.

- Messages are sent in professional Russian, consistent with all other channels
- Delivery status is written back to the Google Sheet after each send
- Reply detection: if the lead has already responded, re-sending is skipped
- Failed SMS attempts increment the attempt count, same as other channels
- SMS credentials are stored securely in n8n's encrypted credential store


---

## 📥 Lead Import Pipeline

A **separate scheduled workflow** runs daily at **17:08** to automatically import raw leads from an external source sheet into the main Leads Engine.

### How It Works

```
Schedule Trigger (17:08 daily)
  └── Get All Existing Leads → Find Max Lead ID
        └── Read Raw Leads (Lead Management sheet)
              └── Set Loop Limit → Loop Over Items
                    └── Assign New Lead ID
                          └── Normalize Phone Number
                                ├── Has Phone? → Append with Contact Method = "Not selected"
                                └── No Phone?  → Append with Contact Method = "Email"
                                      └── Merge → Wait 10s → Next Lead
```

### Source Sheet

| Field | Details |
|---|---|
| **Spreadsheet** | `Lead Management` (ID: `1lCIhwYIHss2THItaeOubZz9ZqlqepG5DuNZUK3BWWrg`) |
| **Tab** | `Raw leads` (gid=0) |

### What Gets Imported

Each raw lead is mapped into the Leads Engine with these fields on import:

| Field | Value |
|---|---|
| `Lead ID` | Auto-assigned (max existing ID + loop index) |
| `Job Role` | From raw lead's `Job Title` field |
| `Import Date` | Today's date (`MM/DD/YYYY`) |
| `Lead Status` | `New` |
| `Attempt Count` | `0` |
| `Phone Number` | Cleaned & normalized (digits only; leading `8` → `7`) |
| `Email` | From raw lead |
| `Contact Method` | `Not selected` if phone exists, `Email` if phone is missing |

### Phone Normalization Logic

```javascript
// Strips all non-digits
// Replaces leading 8 with 7 (Russian mobile format fix)
// e.g. 89001234567 → 79001234567
phone = '7' + phone.slice(1);
```

### Deduplication & ID Assignment

Before importing, the workflow reads all existing leads and finds the current maximum `Lead ID`. Each new lead is assigned `maxId + loop_index` to guarantee unique, sequential IDs with no collisions.

### Old Leads Archive

Non-email leads removed from the active Leads Engine are archived to the `Old Leads data` tab (gid=872591740) before deletion, preserving full history including `Message History`, all response fields (`Email Response`, `WhatsApp Response`, `Telegram Response`, `SMS Response`), and `Attempt Count`.

### Import Loop Settings

In the `Set daily Loop Limit` code node:
```javascript
const MAX_LOOP_LIMIT = 1; // Change to import more leads per run
```
A 10-second wait between each lead import prevents Google Sheets API rate limits.


---

## ⚙️ Configuration

### Loop Limit (leads per run)

In the `Set 180 Loop Limit2` code node:
```javascript
const MAX_LOOP_LIMIT = 10; // Change this value
// 10 leads ≈ 20 min | 30 leads ≈ 1 hour | 60 leads ≈ 2 hours
```

### Schedule Time

Edit the `Schedule Trigger` node. Default: **15:50 (3:50 PM)** — optimized for Russian business hours.

---

## 🔐 Credentials Required

| Credential | Service |
|---|---|
| Google Sheets OAuth2 | Lead database read/write |
| Google Drive OAuth2 | CV file downloads |
| Microsoft Outlook OAuth2 | Email sending |
| VAPI API Key | AI phone calls |
| OpenAI API Key | GPT-5 first message generation |
| Telethon FastAPI Bearer Token | Telegram server auth |
| CodingMantra / UltraMSC | WhatsApp session |
| SMS Gateway API Key | SMS outreach channel |

> All credentials are stored in n8n's encrypted credential store — never hardcoded.

---

## 🚀 Getting Started

1. Import the n8n workflow JSON into your n8n instance
2. Configure all credentials listed above in n8n's credential manager
3. Set up the Google Sheets with the correct tab structure (Leads Engine, Candidates Available, Placements)
4. Deploy the Telethon + FastAPI server on a VPS (for Telegram support)
5. Add leads to the `Leads Engine` tab from row 3 with at minimum: `Lead ID`, `Phone/Email`, `Preferred Contact Method`
6. Activate the workflow — it will run automatically at 15:50 daily

---

## 🛠️ Troubleshooting

| Issue | Solution |
|---|---|
| Leads not being processed | Check `row_number` equals `loop_index + 2` |
| VAPI calls not connecting | Verify international phone number format (e.g. `79001234567`) |
| Emails not sending | Re-authenticate Microsoft Outlook OAuth2 token |
| Telegram messages failing | SSH into VPS, check FastAPI logs, restart Telethon service |
| WhatsApp not sending | Re-scan QR via UltraMSC dashboard |
| SMS not sending | Check SMS Gateway API key validity and account balance/quota |
| CVs not attaching | Ensure Google Drive files are set to "Anyone with link can view" |
| Sheet not updating after call | Verify VAPI webhook URL matches `Webhook1` node path in n8n |

---

## 📞 Contact & Access

To get access to this system or for support:

**📧 prinommojumder@gmail.com**

---

## 📄 License

Private & Proprietary — Imployii Engineering Team. All rights reserved.
