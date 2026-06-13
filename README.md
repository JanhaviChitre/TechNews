# Tech News Agent
 
 **Tech News Agent** is an n8n workflow that runs on a schedule, pulls fresh articles from several technology news RSS feeds, and processes them through a single AI agent call to produce a structured category and a short summary for each article. The processed results are recorded in Google Sheets for historical logging and compiled into a readable, category-grouped report that is sent out to three different channels: email (Gmail), team chat (Slack), and a Telegram chat.

### Prerequisites
- n8n installation (self-hosted or cloud) with import capability.
- Groq API key and n8n Groq credential.
- Google Sheets OAuth2 credential with access to the target spreadsheets.
- Gmail OAuth2 credential (if you want email delivery).
- Slack OAuth2 credential and channel ID (for Slack notifications).
- Telegram bot credentials and target chat ID (for Telegram notifications).

### Quick setup
1. Import the JSON workflow into n8n (Workflow -> Import -> paste the JSON file).
2. In n8n, create credentials for: Groq, Google Sheets, Gmail, Slack, Telegram and attach them to the respective nodes.
3. Replace placeholder IDs and addresses:
   - Google Sheets `documentId` fields: set your spreadsheet IDs for both Sheets nodes.
   - Gmail `sendTo`: set the recipient email address (notes in Gmail node).
   - Slack `channelId`: pick or paste the channel ID and ensure the app is invited.
   - Telegram `chatId`: set the target chat id.
4. Configure the Groq node credential (Groq API key). The workflow expects a Groq model (example: `llama-3.3-70b-versatile`).
5. Review Schedule Trigger node: adjust the scheduled hour as desired.
6. Adjust `Limit for the News` node `maxItems` to control how many articles are processed per run.

### What the workflow does?
- Feeds: example feeds included (TechCrunch, Ars Technica, Hindustan Times Tech).
- Merge & limit: items from feeds are merged and limited per run.
- AI Agent: sends article title + snippet to a Groq LLM agent which must return a JSON with `category` and `summary`.
- Sheets: appends rows containing Title, Link, Published, Category, AI Summary to configured Google Sheets.
- Notifications: composes a daily report and sends it via Gmail, Slack, and Telegram.

## Workflow Architecture
 
```
                ┌────────────────────┐
                │  Schedule Trigger   │
                └──────────┬─────────┘
                            │
        ┌───────────────────┼───────────────────┐
        ▼                   ▼                    ▼
┌───────────────┐ ┌──────────────────┐  ┌─────────────────────────┐
│RSS: TechCrunch│ │ RSS: Ars Technica│  │RSS: Hindustan Times Tech│
└───────┬───────┘ └─────────┬────────┘  └────────────┬────────────┘
        └───────────────────┼────────────────────────┘
                            ▼
                ┌────────────────────────────── ┐
                │Merge the contents of RSS Feeds│
                └───────────────┬───────────────┘
                                 ▼
                     ┌──────────────────────┐
                     │  Limit for the News  │
                     └───────────┬──────────┘
                                 ▼
        ┌─────────────────────────────────────────────────┐
        │   AI Agent - Categorize & Summarize (Groq LLM)   │
        │   + Structured Output Parser                     │
        └───────────┬───────────────────────┬──────────────┘
                     ▼                       ▼
       ┌───────────────────────── ┐   ┌──────────────────────┐
       │ Sheet for Gmail and Slack│   │   Limit for Telegram  │
       └────────────┬─────────────┘   └───────────┬───────────┘
                     ▼                              ▼
   ┌──────────────────────────────────┐  ┌──────────────────────┐
   │  Code in JS for Gmail and Slack  │  │   Sheet for Telegram │
   │       (grouped report A)         │  └───────────┬──────────┘
   └─────────────┬────────────────────┘              ▼
              ┌──────┴──────┐               ┌────────────────────────┐
              ▼             ▼               │ Code in JS for Telegram│
          ┌───────┐     ┌───────┐           │   (grouped report B)   │
          │ Gmail │     │ Slack │           └───────────┬────────────┘
          └───────┘     └───────┘                       ▼
                                                  ┌──────────────┐
                                                  │   Telegram   │
                                                  └──────────────┘
```
#### Stage 1 — Trigger
- **Node**: `Schedule Trigger`
- **Operation**: Initiates the workflow at a configured time interval (e.g., daily).

#### Stage 2 — Fetch RSS Feeds
- **Nodes**: `RSS Feed - TechCrunch2`, `RSS Feed - Ars Technica2`, `RSS Feed - Hindustan Times Tech2`
- **Operation**: Each node independently fetches the latest items from its configured RSS feed URL.

#### Stage 3 — Merge Feeds
- **Node**: `Merge the contents of RSS Feeds`
- **Operation**: Combines the three separate article lists into a single unified list of items.

#### Stage 4 — Limit Batch Size
- **Node**: `Limit for the News`
- **Operation**: Truncates the merged list to a fixed maximum number of articles (configurable, eg., 10) to control cost and message length.

#### Stage 5 — AI Categorization & Summarization
- **Nodes**: `AI Agent - Categorize & Summarize`, `Groq LLM` (language model), `Structured Output Parser`
- **Operation**: For each article, the AI Agent sends the title and content snippet to the Groq-hosted LLM with a system prompt instructing it to (a) classify the article into one of a fixed set of technology categories and (b) write a concise 2–3 sentence summary. The Structured Output Parser enforces that the LLM's response is returned as JSON with `category` and `summary` fields.

#### Stage 6 — Persist to Google Sheets (Dual Logging)
- **Nodes**: `Sheet for Gmail and Slack`, `Limit for Telegram`, `Sheet for Telegram`
- **Operation**:
  - `Sheet for Gmail and Slack` appends a new row (Title, Link, Published date, Category, AI Summary) to a Google Sheet used by the Gmail/Slack reporting branch.
  - In parallel, the AI Agent's output also flows through `Limit for Telegram` (a second limit, allowing the Telegram branch to use a different/smaller batch size if desired) into `Sheet for Telegram`, which appends the same style of row to a separate Google Sheet dedicated to the Telegram branch.

#### Stage 7 — Build Category-Grouped Reports
- **Nodes**: `Code in JS for Gmail and Slack`, `Code in JS for Telegram`
- **Operation**: Each Code node receives the logged article items, groups them by their `Category` field, and constructs a single formatted text string (`report`) with a heading, date, and per-category sections — each listing the article's title, published date, AI summary, and link.

#### Stage 8 — Deliver the Digest
- **Nodes**: `Gmail`, `Slack`, `Telegram`
- **Operation**:
  - `Gmail` sends the report from Code in JS for Gmail and Slack as an email to a configured recipient.
  - `Slack` posts the same report from Code in JS for Gmail and Slack as a message to a configured Slack channel.
  - `Telegram` sends the report from Code in JS for Telegram as a message to a configured Telegram chat.

### Key fields & output:
- **Title, Link, Published:** taken from the RSS item.
- **Category:** One of [AI & Machine Learning, Cybersecurity, Cloud & Infrastructure, Startups & Funding, Hardware & Devices, Software Development, Big Tech & Policy, Other].
- **AI Summary:** 2–3 sentence concise summary produced by the AI agent.

### Customization tips
- To add/remove feeds: edit or duplicate RSS Feed nodes and connect them into the merge node.
- To change summary behavior: edit the AI Agent node `systemMessage` or `prompt` text.
- To change destinations: remove or add nodes (Google Sheets, Slack, Telegram, Gmail) and wire outputs.

### Security & notes
- Do not commit API keys or OAuth tokens to version control. Use n8n credentials.
- Ensure the Google Sheets have header columns exactly: Title, Link, Published, Category, AI Summary.

### Troubleshooting
- If no data appears in Sheets, verify Sheets credential, documentId, and that header row exists.
- If AI output parsing fails, adjust the Structured Output Parser `jsonSchemaExample` and the agent prompt to match the expected JSON shape.


