# Waypoint 
This is an n8n-based automated email management system.
It automatically retrieves emails from Gmail, uses AI (OpenAI) for intelligent classification—such as identifying actionable tasks, emails to be forwarded, or messages for reference only—and seamlessly syncs tasks that require action to Google Tasks.

## 1. Waypoint-mail-retriever

**Function:**  Periodically scans the Gmail inbox.

**Logic:** Verifies if the database `Waypoint_Mail_DB` exists before proceeding.

**Deduplication Mechanism:** Uses the `threadId` to ensure only new emails are recorded.

**Status:** New entries are marked as `unprocessed` by default.


## 2. Waypoint-classifier

**Function:** Processes unprocessed emails stored in the database.

**AI Engine:** Utilizes OpenAI models via an AI Agent.

**Classification Tags:**  
- **Draft Task:** For emails requiring specific action or follow-up.  
- **FYI:** For informational content, such as policy updates or general reading.  
- **Forward:** For emails that need to be redirected to other parties.  
- **Noise:** For advertisements and automated system notifications.

**Deadline Extraction:** The AI automatically identifies and extracts deadlines from the email body.

**Output:** Saves the classification results and metadata into `Waypoint_Ptmp_DB`.

---

## 3. Waypoint-task

**Function:** Scans `Waypoint_Ptmp_DB` for records tagged as **Draft Task**.

**Logic:** If a `due_date` is identified, the task is created with that specific deadline.

**Integration:** Automatically creates tasks in a specified Google Tasks list, including the email subject and content snippets.
