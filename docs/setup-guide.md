# Setup Guide

---

## Prerequisites

- Windows, Mac, or Linux, with at least ~4 GB of free disk space
- A Gmail account with **2-Step Verification** enabled
- An OpenAI account with API access
- A Google account (for Google Sheets + Google Cloud Console)
- A Slack workspace (a free personal one is fine for testing)

---

## 1. Run n8n Locally with Docker

n8n runs inside a Docker container, so it works the same way on any machine and
doesn't install anything permanent on your system outside of Docker itself.

### 1.1 Install Docker Desktop
1. Download **Docker Desktop** from docker.com for your OS
2. Run the installer
   - **Windows:** it will prompt to enable **WSL 2** — accept this, then restart your
     computer if asked. If Docker later complains that WSL isn't installed, run
     `wsl --install` in an **administrator** PowerShell window, then restart again.
3. Open Docker Desktop and wait until it shows as running (the whale icon stops
   animating)

**Verify it worked:**
```bash
docker --version
```

### 1.2 Create the project folder and Compose file
```bash
mkdir invoice-automation
cd invoice-automation
```

Create a file named exactly `docker-compose.yml` in that folder:
```yaml
services:
  n8n:
    image: docker.n8n.io/n8nio/n8n
    restart: unless-stopped
    ports:
      - "5678:5678"
    volumes:
      - n8n_data:/home/node/.n8n

volumes:
  n8n_data:
```

### 1.3 Start it
```bash
docker compose up -d
```
Open **http://localhost:5678** in your browser. The first time, you'll be asked to
create an owner account (just a local email + password).

### Everyday commands
| Action | Command |
|---|---|
| Start n8n | `docker compose up -d` |
| Stop n8n | `docker compose down` |
| Check it's running | `docker ps` |
| View logs (troubleshooting) | `docker compose logs -f n8n` |
| Update to the latest version | `docker compose pull` then `docker compose up -d` |

---

## 2. Configure the Email Trigger

This project uses **IMAP**, not Gmail's full OAuth, since it's much simpler to set
up and needs no Google Cloud Console project.

### 2.1 Generate a Gmail App Password
1. Go to **myaccount.google.com → Securit**
2. Confirm **2-Step Verification** is **On** (turn it on first if it isn't)
3. Search the page for **"App passwords"** (or search
   **app passwords**)
4. Create one named e.g. `n8n-invoice-project`
5. Copy the 16-character password shown — you'll only see it once

### 2.2 Add the node
1. In n8n, add an **Email Trigger (IMAP)** node
2. Turn on **Download Attachments**
3. Under **Credentials**, create a new one:

| Field | Value |
|---|---|
| User | your full Gmail address |
| Password | the 16-character App Password (not your real password) |
| Host | `imap.gmail.com` |
| Port | `993` |
| SSL/TLS | On |

### 2.3 Point it at a dedicated folder
Create a Gmail label (e.g. `Invoices`) and set the node's **Mailbox** field to it (Default: INBOX),
rather than the main inbox. This keeps testing predictable and matches how you'd
deploy this for a real client — watching one dedicated folder, not someone's whole
inbox.

---

## 3. Connect OpenAI, Google Sheets, and Slack Credentials

### 3.1 OpenAI
1. Get a key from **platform.openai.com/api-keys** → **Create new secret key**
2. In n8n, when adding the **OpenAI Chat Model** node (feeding the Information
   Extractor), create a new credential and paste the key in
3. Set the model to `gpt-4o-mini`

### 3.2 Google Sheets (Service Account method)
This avoids Google's OAuth consent-screen flow entirely, which is normally the
most error-prone part of connecting Google services.

1. **console.cloud.google.com** → create/select a project → enable the
   **Google Sheets API**
2. **IAM & Admin → Service Accounts → Create Service Account**, name it, skip the
   optional role step, click **Done**
3. Open it → **Keys** tab → **Add Key → Create new key → JSON** → a file downloads.
   Keep it private; never commit it to a repo.
4. Open that JSON file, copy the `client_email` value
5. **Share your Google Sheet** with that email address, giving it **Editor** access
   — this step is easy to miss and the most common cause of a "permission denied"
   error later
6. In n8n's **Google Sheets** node, create a credential of type
   **Google Sheets (Service Account)**, and paste in:
   - **Service Account Email** → the `client_email` field
   - **Private Key** → the `private_key` field (the whole block, including the
     `-----BEGIN/END PRIVATE KEY-----` lines)

### 3.3 Slack
1. **api.slack.com/apps → Create New App → From scratch**, pick your workspace
2. **OAuth & Permissions → Bot Token Scopes → Add an OAuth Scope → `chat:write`**
3. Scroll up → **Install to Workspace → Allow**
4. Copy the **Bot User OAuth Token** (`xoxb-...`)
5. In n8n's **Slack** node, create a credential of type **Access Token** (not
   OAuth2), paste the token in
6. In Slack itself, invite the bot into your alert channel:
   `/invite @YourBotName`

---

## 4. Required Spreadsheet Columns

Create a Google Sheet (any name) with exactly this header row in the first row of
the first tab:

```
vendor_name | invoice_number | invoice_date | due_date | subtotal | tax_amount | total_amount | currency | processed_date | status
```

The workflow's **Auto-map Input Data to Columns** option matches by these exact
column names, so they need to match precisely — same spelling, same case,
underscores not spaces.

---

## 5. Configure the Error-Handler Workflow

This is a **separate** workflow from the main one, built once and left running in
the background.

1. In n8n, create a new workflow: `Invoice Automation - Error Handler`
2. Add an **Error Trigger** node — this only fires when another workflow fails
3. Add a **Slack** node after it (reuse the same Slack credential from 3.3), with
   a message like:
   ```
   🚨 Invoice automation workflow failed!
   Workflow: {{ $json.workflow.name }}
   Failed node: {{ $json.execution.lastNodeExecuted }}
   Error: {{ $json.execution.error.message }}
   ```
4. Save this workflow, then **Publish** it (top right dropdown button) — it must be active
   to listen for failures
5. Open your **main** workflow → **⋯ menu → Settings → Error Workflow** → select
   this error-handler workflow → save

From now on, any unhandled failure in the main pipeline (a bad PDF, an API outage,
a malformed response) sends a Slack alert instead of failing silently.

---

## 6. How to Test the Workflow Safely

A few habits that keep testing from causing real-world side effects or confusing
results:

- **Use a dedicated test label/folder in Gmail** (2.3), never your real inbox —
  this avoids accidentally processing unrelated personal email, and lets you control
  exactly what the trigger sees.
- **Use a separate test channel in Slack** (e.g. `#invoice-alerts-test`) while
  building, so test alerts don't mix with real ones later.
- **Watch for pinned data.** If you ever click the thumbtack icon on a node's output
  to freeze a test value (useful for forcing a specific scenario), remember to
  **unpin it afterward** — a forgotten pin silently feeds stale data into every
  future run of that node until removed.
- **Test each branch on purpose, not just the easy path.** For each IF node,
  deliberately create an input that goes down *both* outputs at least once:
  - Missing field → temporarily blank a field in a test item
  - Duplicate → manually add a matching row to the sheet first, then remove it after
  - Threshold → temporarily lower the dollar threshold to something tiny (e.g. `1`)
    to force the "True" path, then set it back
- **Test the error-handler deliberately once.** Temporarily break something small
  and obvious (e.g. a typo'd field reference in one node), run the workflow, confirm
  the Slack alert fires, then undo the change.
- **Be mindful of real API costs while testing.** Every test run that reaches the
  OpenAI or Google Sheets nodes makes a real, billed API call. Costs are small with
  `gpt-4o-mini`, but avoid running the full workflow dozens of times unnecessarily —
  use **Test step** on individual nodes where possible instead of always running the
  entire pipeline end to end.
- **Clear test rows from your sheet before considering the project "done."** Leftover
  manually-added test rows (like one used to simulate a duplicate) should be removed
  so your final sheet reflects only real, automated runs.
