# MeetMind — AI-Powered Meeting Assistant

> Transform meeting notes into actionable tasks — automatically tracked, assigned, and delivered.

**Live Website:** [meetmind-hackathon-takim11.github.io](https://meetmind-hackathon-takim11.github.io)
**Live Website:** Youtube Link

---

## What is MeetMind?

MeetMind is a meeting productivity tool that turns raw meeting notes into structured tasks using Google Gemini AI. Tasks are automatically synced to Google Sheets, assigned to team members via Gmail, and scheduled in Google Calendar — all from a single paste.

Built with a fully static frontend (vanilla HTML/CSS/JS) hosted on GitHub Pages, powered by n8n automation workflows in the backend.

---

## Features

- **AI Task Extraction** — Paste meeting notes, Gemini extracts tasks, people, deadlines, and priorities automatically
- **Human-in-the-Loop Approval** — Review and confirm AI-extracted tasks before anything is saved or sent
- **Live Dashboard** — KPI cards, risk meter, mood analysis, person distribution, all synced from Google Sheets
- **Task Management** — Filter, sort, and update task statuses directly from the dashboard
- **Calendar View** — Deadline tracking with an interactive monthly calendar
- **Meeting Archive** — Browse past meetings and their associated tasks
- **Voice Notes** — Send a Telegram voice message instead of typing — Whisper AI transcribes it automatically
- **Weekly Digest** — Automated Monday morning email summarizing completed and pending tasks
- **Status Sync** — Update task status in the dashboard and it syncs back to Google Sheets in real time

---

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | Vanilla HTML, CSS, JavaScript |
| Hosting | GitHub Pages |
| Automation | n8n (Cloud) |
| AI — Task Extraction | Google Gemini API |
| AI — Voice Transcription | OpenAI Whisper API |
| Database | Google Sheets + SheetDB |
| Email | Gmail via n8n |
| Calendar | Google Calendar via n8n |
| Voice Interface | Telegram Bot |

---

## Architecture

```
┌─────────────────────────────────────────────────────┐
│                   GitHub Pages                       │
│                  (index.html)                        │
│                                                      │
│  ┌─────────┐  ┌─────────┐  ┌──────────┐  ┌──────┐  │
│  │Dashboard│  │ Submit  │  │  Tasks   │  │ More │  │
│  └────┬────┘  └────┬────┘  └────┬─────┘  └──────┘  │
└───────┼────────────┼────────────┼───────────────────┘
        │            │            │
        │ SheetDB    │ Webhook    │ Webhook
        │ (GET)      │ (POST)     │ (POST)
        ▼            ▼            ▼
┌───────────────────────────────────────────────────┐
│                    n8n Cloud                       │
│                                                   │
│  Workflow 1          Workflow 2      Workflow 3   │
│  Main Flow           Weekly Email   Telegram Bot  │
│                                                   │
│  Workflow 4                                       │
│  Status Update                                    │
└───────┬───────────────────────────────────────────┘
        │
        ▼
┌───────────────────────────────┐
│  Google Sheets (SheetDB)      │
│  Google Calendar              │
│  Gmail                        │
│  Telegram                     │
└───────────────────────────────┘
```

---

## n8n Workflows

### Workflow 1 — Main Meeting Flow

The core workflow. Handles both the analysis phase and the confirmation phase via a single webhook, distinguished by an `action` field.

**Analysis phase** (`action: "analyze"`):
```
Webhook → Metin Temizle → IF (action == confirm?)
                               ↓ FALSE
                          Gemini Body Hazırla → HTTP Request (Gemini API)
                          → JSON Çözümle → Veri Hazırla
                          → Respond to Webhook (returns extracted tasks to frontend)
```

**Confirmation phase** (`action: "confirm"`):
```
Webhook → Metin Temizle → IF (action == confirm?)
                               ↓ TRUE
                          Confirm Hazırla → Sheets (append row)
                                         → Email Template → IF Has Email → Gmail
                                         → IF Has Deadline → Color Map → Calendar Event
                                         → Get Follow Up Date → Calendar Event
                          → Respond to Webhook
```

**Trigger:** POST webhook from frontend  
**Inputs:** `meeting_title`, `meeting_notes`, `meeting_date`, `action`, `action_items`  
**Outputs:** Gemini JSON (analyze) / Sheets + Gmail + Calendar (confirm)

---

### Workflow 2 — Weekly Monday Email

Runs automatically every Monday morning. Fetches all tasks from Google Sheets, summarizes completed and pending items, and sends a motivational digest email to the team.

```
Cron (Every Monday) → HTTP Request (SheetDB — fetch all tasks)
→ Code (summarize completed / pending / overdue)
→ Gmail (send weekly digest to team)
```

**Trigger:** Cron schedule (Monday 09:00)  
**Outputs:** Weekly summary email to all team members

---

### Workflow 3 — Telegram Voice Note

Allows sending meeting notes as voice messages via Telegram instead of typing. Whisper AI transcribes the audio, then the same Gemini pipeline processes the text.

```
Telegram Trigger (voice message received)
→ Download Audio File
→ HTTP Request (OpenAI Whisper API — transcribe)
→ Metin Temizle → Gemini Body Hazırla → HTTP Request (Gemini API)
→ JSON Çözümle → Veri Hazırla
→ Telegram (send confirmation message back to user)
→ [User replies "Evet" to confirm]
→ Sheets + Gmail + Calendar
```

**Trigger:** Telegram bot voice message  
**Outputs:** Transcription → task extraction → same as Workflow 1 confirmation

---

### Workflow 4 — Status Update

Triggered when a user changes a task's status dropdown in the dashboard. Updates the corresponding row in Google Sheets in real time.

```
Webhook (POST) → Google Sheets (Update Row by Görev column)
→ Respond to Webhook ({"success": true})
```

**Trigger:** POST webhook from frontend status dropdown  
**Inputs:** `id` (task name), `status` (new status value)  
**Outputs:** Updated `Durum` column in Google Sheets

---

## Google Sheets Structure

The sheet must have a tab named `Log` with these exact column headers:

| Column | Description |
|---|---|
| `ID` | Unique task identifier |
| `Toplantı Adı` | Meeting name |
| `Sorumlu` | Assigned person |
| `Görev` | Task description |
| `Deadline` | Due date (YYYY-MM-DD) |
| `Öncelik` | Priority (Yüksek / Orta / Düşük) |
| `Durum` | Status (Bekliyor / Devam Ediyor / Tamamlandı) |
| `Tarih` | Meeting date |
| `E-Posta` | Assignee email address |

---

## Setup

### Prerequisites

- n8n Cloud account (free tier works)
- Google account (Sheets, Gmail, Calendar)
- Gemini API key ([aistudio.google.com](https://aistudio.google.com))
- SheetDB account ([sheetdb.io](https://sheetdb.io))
- Telegram Bot (optional, for voice notes)
- OpenAI API key (optional, for voice transcription)

### Steps

**1. Google Sheets**
- Create a new Google Sheet
- Add a tab named `Log`
- Add the column headers listed above in row 1
- Connect to SheetDB → copy the API URL

**2. n8n**
- Import the 4 workflow JSON files from the `/workflows` folder
- Add your credentials (Google, Gemini, Gmail, Telegram)
- Activate all workflows
- Copy the webhook URLs from Workflow 1 and Workflow 4

**3. Frontend Configuration**
- Open `index.html`
- Find the `CONFIG` object at the top and fill in your values:

```javascript
const CONFIG = {
  webhookUrl:        'YOUR_N8N_WORKFLOW1_WEBHOOK_URL',
  confirmWebhookUrl: 'YOUR_N8N_WORKFLOW1_WEBHOOK_URL', // same URL, action field differentiates
  sheetDbUrl:        'YOUR_SHEETDB_API_URL',
  updateWebhookUrl:  'YOUR_N8N_WORKFLOW4_WEBHOOK_URL',
  voiceWebhookUrl:   'YOUR_N8N_WORKFLOW3_WEBHOOK_URL',
  refreshInterval:   30000,
};
```

**4. Deploy**
- Push `index.html` to your GitHub repository
- Enable GitHub Pages in repository Settings → Pages
- Set source to main branch, root folder

---

## How It Works — User Flow

```
1. Open dashboard → meetmind-hackathon-takim11.github.io
2. Click "Toplantı Ekle" or go to "Toplantı Gönder"
3. Enter meeting name, date, and paste meeting notes
4. Click "Gönder" — Gemini analyzes the notes
5. Review extracted tasks in the approval panel
6. Click "Onayla" — tasks saved to Sheets, emails sent, calendar events created
7. Dashboard refreshes automatically — tasks appear in all views
```

Alternatively, send a voice message to the Telegram bot — the rest of the flow is identical.

---

## Project Structure

```
/
├── index.html          # Full frontend — dashboard, submit, tasks, calendar, notes, voice
├── README.md
└── workflows/
    ├── workflow1-main.json
    ├── workflow2-weekly-email.json
    ├── workflow3-telegram-voice.json
    └── workflow4-status-update.json
```

---

## Team

Built for Hackathon by **Takım 11**
Kerziban Sicim | Özde Hava Sinecek | Enes Koyuncu | Göktuğ Kocatürk
---
