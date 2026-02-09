# WAYPOINT: AI-Powered Email-to-Task Intelligence Pipeline

WAYPOINT is a modular, AI-driven intelligence system built on **n8n**. It transforms unstructured emails into actionable tasks using thread-aware LLM analysis, high-density Telegram notifications, and an interactive **Human-in-the-Loop (HITL)** approval flow.

---

## 🚀 Features

- **Signal Acquisition**: Automated Gmail ingestion with OAuth2 and intelligent noise filtering.
- **Intelligence Navigation**: Thread-aware intent classification (Actionable/FYI/Forward) using Google Gemini.
- **Interactive HITL**: Telegram-based approval interface—you decide which tasks sync to your calendar.
- **Native Sync**: Seamless integration with **Google Tasks** for approved action items.
- **Stateless Privacy**: Built with privacy in mind, ensuring no personal data persists beyond processing.

---

## 🏗 Architecture

The system is composed of three primary modular workflows:

1.  **Waypoint-mail-retriever**: 
    - Ingests unread emails from Gmail.
    - Uses LLM (Google Gemini) to extract "Waypoints" (subject, due date, stakeholders, action items).
    - Buffers data in n8n Data Tables for thread-level context.
2.  **Waypoint-Collect_TG**:
    - Aggregates processed tasks into high-density Telegram reports.
    - Triggers interactive notifications with "Approve" and "Decline" navigation buttons.
3.  **Waypoint-Telegram-Approve Or Decline Flow**:
    - Acts as a webhook receiver for Telegram button interactions.
    - Synchronizes approved tasks directly to Google Tasks.

---

## 📋 Prerequisites

Before importing the workflows, ensure you have the following:

- **n8n Instance**: Self-hosted via Docker (Recommended).
- **Google Cloud Project**:
  - Enable **Gmail API** and **Google Tasks API**.
  - Configure **OAuth2 Credentials** with scopes: `https://www.googleapis.com/auth/gmail.modify` and `https://www.googleapis.com/auth/tasks`.
- **Telegram Bot**:
  - Create a bot via [@BotFather](https://t.me/botfather) to get your **Bot Token**.
- **AI Provider**:
  - A **Google Gemini API Key** (or OpenAI API Key).
- **n8n Data Tables**:
  - Ensure the following tables are created in your n8n environment: `Mail_Table`, `LLM_Table`, `Task_Table`.

---

## 🛠 Installation & Setup

### 1. Import Workflows
1.  Download the JSON files from the `workflows/` directory.
2.  In n8n, go to **Workflows** > **Add Workflow** > **Import from File** for each JSON:
    - `Waypoint-mail-retriever.json`
    - `Waypoint-Collect_TG.json`
    - `Waypoint-Telegram-Approve Or Decline Flow.json`

### 2. Configure Credentials
Update the following credentials in n8n for each workflow node:
- **Gmail OAuth2 API**: Link to your Google Cloud credentials.
- **Google Tasks OAuth2 API**: Link to the same or dedicated Google Cloud credentials.
- **Telegram Bot API**: Enter your Bot Token.
- **Google Gemini(PaLM) API**: Enter your API Key.

### 3. Setup Environment Variables
To ensure **Stateless Processing** and proper timezone handling, add these to your n8n environment:
```env
GENERIC_TIMEZONE=Asia/Taipei
N8N_EXECUTION_PROCESS_DATA_PRUNING=true
```

---

## 📖 Usage

1.  **Activate Workflows**: Turn on all three workflows in n8n.
2.  **Email Processing**: The `mail-retriever` runs on a schedule (default: every hour) or can be triggered manually.
3.  **Telegram Navigation**:
    - You will receive a summary of new emails in your Telegram chat.
    - Click **[Approve]** to sync the task to Google Tasks.
    - Click **[Decline]** to archive the suggestion.
4.  **Error Monitoring**: Check the n8n execution log if a task fails to sync.

---

## 🛡 Compliance

WAYPOINT adheres to strict privacy standards:
- **Stateless Execution**: Raw email content is deleted after classification.
- **Audit Logs**: Only non-PII metadata is logged for system monitoring.

---

**Developed for AI Agent Hackathon 2026**
