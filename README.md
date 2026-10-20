# Lab test results assistant (n8n + WhatsApp + AI agents)

An n8n workflow for a medical lab results service: doctors and patients sign up through web forms, the lab uploads test result PDFs, and patients or their doctors get the PDFs on WhatsApp by chatting with an AI assistant that first checks who they are and what they are allowed to see.

![Tour of the workflow: overview, then each of the four flows](docs/demo.gif)

> The GIF is a tour of the imported workflow in a local n8n instance (overview, then the four flows). It does not show a live run: running it needs Google Sheets, Google Cloud Storage, WhatsApp Business and OpenAI credentials, which are not part of this repo. The user-facing texts are in Romanian; the assistant answers in the user's language.

## The problem

I built this for a clinic. The idea: patients ask for their lab results on WhatsApp ("send me everything after 1 August"), doctors get the results of their own patients the same way, and nobody can get someone else's results. The clinic stopped the collaboration before the workflow was connected to their systems or to any real data.

## What it does

The workflow has four flows (one per colour on the canvas):

1. **Doctor registration** (form): name, date of birth, email, phone. Checks that the doctor does not exist yet, gives them an ID (`D-001`, ...) and saves them.
2. **Patient registration** (form): the same fields plus "your doctor's name". An AI agent matches the free-text doctor name against the doctors list (handles diacritics, swapped first/last name, "Dr.", typos; returns NONE below 0.75 confidence). Then the patient gets an ID built from their initials (`P-ABC-001`) and is linked to that doctor.
3. **Upload test results** (form): patient ID plus one or more PDF/DOC files. Checks that the patient exists, uploads each file to Google Cloud Storage under `patient-results/<patient_id>/`, gives each result an ID and records it.
4. **WhatsApp assistant**:
   - only text messages are accepted; the user immediately gets a "preparing the answer" message
   - the AI agent (gpt-5-mini, with memory per phone number) first checks if this WhatsApp number is already authenticated; if not, it asks for date of birth and email and matches them, together with the phone number, against the patients and doctors sheets; authentication is valid for 30 days
   - **patients** can only get their own PDFs and their own name, date of birth and email
   - **doctors** can list their assigned patients and get those patients' PDFs, never other doctors' patients
   - requests can be filtered by date ("before", "after", "between"); ambiguous ones ("from 15 August") get a clarifying question, and requests for two people at once are refused
   - when the agent decides to send, the workflow lists the patient's files in the bucket, applies the date filter, downloads each PDF, cleans the file name and sends it as a WhatsApp document

The agent always answers with a fixed JSON structure (reply, intent, role, patient ID, date filter, ...), validated by an output parser, so the rest of the workflow branches on fields instead of free text.

## Architecture

![The whole workflow on the n8n canvas](docs/workflow-canvas.png)

```mermaid
flowchart LR
    F1[Doctor form] --> A1[ID agent] --> S[(Google Sheets<br/>doctors, patients, results,<br/>counters, authentications)]
    F2[Patient form] --> M[Doctor-name matcher agent] --> A2[ID agent] --> S
    F3[Upload form] --> G[(Google Cloud Storage<br/>patient-results/ID/)]
    F3 --> A3[Result ID agent] --> S
    W[WhatsApp message] --> AG[WhatsApp agent<br/>auth + permissions + intent]
    AG <-->|Sheets tools| S
    AG -->|send_pdfs + filter| L[List files, filter by date] --> G
    L --> D[Download PDF, send as WhatsApp document]
```

| Patient registration | Upload test results |
|---|---|
| ![Patient registration flow](docs/patient-registration.png) | ![Upload flow](docs/file-upload.png) |

| Doctor registration | WhatsApp assistant |
|---|---|
| ![Doctor registration flow](docs/doctor-registration.png) | ![WhatsApp flow](docs/whatsapp-chatbot.png) |

## Tech stack

n8n (Form Trigger, AI Agent, Structured Output Parser with auto-fix, Simple Memory, Google Sheets and Google Sheets tool nodes, Google Cloud Storage, WhatsApp Business Cloud trigger and send, Code node, If/Filter/Loop), OpenAI API (gpt-5-mini, gpt-4.1-mini, gpt-4o).

## How to run it

1. Import `workflow/lab-results-assistant.workflow.json` into n8n (Workflows > Import from file). I checked the import on n8n 2.4.7.
2. Create credentials and select them in the nodes: OpenAI, Google Sheets (OAuth), Google Cloud Storage, WhatsApp Business Cloud (trigger and send).
3. Create a Google Sheet with the tabs `doctori`, `pacienti`, `rezultate-analize`, `counters`, `authentications` (column names are visible in the Sheets nodes) and select it where you see `<LONG_ID_1>`. Create a bucket and set its name in the Cloud Storage nodes.
4. Put your WhatsApp Business phone number ID in the WhatsApp nodes (`<PHONE_…>` placeholders) and in the form completion messages.
5. Activate the workflow, register a doctor and a patient through the forms, upload a PDF, then write to the WhatsApp number.

### What was removed from this export

Exported with an allowlist: I started from an empty copy (nodes, names, connections) and restored only fields worth showing, after a scan. Credentials, webhook IDs, pinned data, the spreadsheet ID, phone numbers, emails and URLs were removed or replaced by placeholders such as `<LONG_ID_1>` or `<PHONE_2>`; the same original value always gets the same placeholder. Some example IDs in the prompts (like `P-ABC-001`) were also caught by the scan and show up as `P-<JIRA_KEY_n>`. The prompts and the node logic are otherwise as I left them.

## My contribution

Solo project: I built this workflow in n8n, including the agents' prompts.

**About this repository:** the workflow is mine. I exported and packaged it for GitHub (this README, screenshots, demo GIF, removing private data) with help from an AI assistant, which is why the single commit lists Claude as co-author.

## Status and limitations

- **Never used with real people or real data.** Only I tested it, with test data.
- In this export the "Get all Results" tool is disabled and the prompt says rate limiting is "currently disabled".
- **Authentication and permissions are decided by the AI agent, following the prompt.** The Sheets tools themselves do not enforce roles: for example, the doctor filter in "Get all Patients" is a value the model fills in. A successful prompt injection could make the agent read data it should not. The prompt is strict about this, but it is not a guarantee.
- **IDs are allocated by an LLM** reading a counters sheet and returning the next number. That is not deterministic and two sign-ups at the same time could get the same number.
- Medical data (names, dates of birth, phone numbers, result files) lives in Google Sheets and a bucket; there is no consent flow, retention policy or audit log.
- The date filter uses the file's upload/storage time in the bucket, not the date of the test.
- Phone normalization assumes Romanian numbers (country code 40).

## What I would change for production

- Move authentication and authorization into code: a one-time code sent on WhatsApp, a session table, and tools that only accept the authenticated user's ID (never a value chosen by the model).
- Allocate IDs with code (a Code node or a database sequence), not with a model.
- Replace Google Sheets with a real database (for example Postgres) with encryption, access logs, consent and retention rules suitable for health data (GDPR).
- Store the test date as metadata and filter on it.
- Add evaluation conversations (impersonation attempts, two patients in one message, ambiguous dates) and run them after every prompt change.
