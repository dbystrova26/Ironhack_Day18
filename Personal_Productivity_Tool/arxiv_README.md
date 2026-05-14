# arXiv Research Summarizer — n8n + Notion

## What This Project Does

This workflow automatically processes academic papers from arXiv. You send it a paper URL, and it:
1. Fetches the paper metadata from the arXiv API
2. Parses the XML response to extract title, authors, and abstract
3. Generates a plain-language summary using the Anthropic Claude AI
4. Saves everything to a Notion database for future reference

## Repository Structure

```
├── README.md                    # This file
├── lab_summary.md               # One-paragraph conceptual summary
└── screenshots/
    ├── 01_webhook_receiving_data.png      # Webhook receiving arXiv URL
    ├── 02_paper_id_extracted.png          # Code node extracting paper ID
    ├── 03_arxiv_data_fetched.png          # HTTP Request fetching XML from arXiv
    ├── 04_xml_parsed.png                  # Code node parsing XML to structured data
    ├── 05_summary_generated.png           # Anthropic API generating summary
    ├── 06_notion_page_created.png         # Notion API creating database entry
    ├── 07_notion_database.png             # Notion database with papers
    └── 08_full_workflow_canvas.png        # Complete n8n workflow canvas
```

---

## How to Run This Workflow

### Prerequisites
- n8n instance (cloud or self-hosted)
- Notion account with a database set up
- Anthropic API key (or OpenAI API key)

### Setup Steps

**1. Create Notion Database**
- Create a new page in Notion
- Add an inline database with these columns:
  - `Title` (Title type)
  - `Authors` (Text type)
  - `url` (URL type)
  - `Category` (Select type — options: ML, Physics, CS, Math, Other)
  - `Summary` (Text type)
  - `Date Added` (Date type)
  - `Read?` (Checkbox type)
- Connect your Notion integration to the database via "..." → Connections

**2. Get Your Credentials**
- Notion: Go to notion.so/my-integrations → Personal access tokens → New access token
- Anthropic: Get API key from console.anthropic.com

**3. Import Workflow to n8n**
- Open n8n → New Workflow
- Build nodes as described below
- Add your credentials to the relevant HTTP Request nodes

**4. Send a Test Request**
Use ReqBin (reqbin.com) or any API tool to send:
```
POST https://your-n8n-instance/webhook/arxiv-summarizer
Content-Type: application/json

{
  "arxiv_url": "https://arxiv.org/abs/2301.07041"
}
```

---

## Workflow Architecture

```
Webhook → Code (extract ID) → HTTP Request (arXiv API) → Code (parse XML)
       → HTTP Request (Anthropic API) → Code (extract summary)
       → Code (build Notion body) → HTTP Request (Notion API)
       → Respond to Webhook
```

### Node-by-Node Explanation

---

### Node 1: Webhook
**What it is:** A trigger node that listens for incoming HTTP POST requests.

**Why we use it:** The workflow needs a way to receive paper URLs from outside. The Webhook node creates a public URL endpoint that accepts POST requests containing the arXiv paper URL.

**Configuration:**
- HTTP Method: POST
- Path: `arxiv-summarizer`
- Respond: Using Respond to Webhook Node

**Input it receives:**
```json
{
  "arxiv_url": "https://arxiv.org/abs/2301.07041"
}
```

---

### Node 2: Code in JavaScript (Extract Paper ID)
**What it does:** Extracts the paper ID from the full arXiv URL.

**Why we need it:** The arXiv API requires just the paper ID (e.g. `2301.07041`), not the full URL. This node strips everything else out.

**Code logic:**
```javascript
const url = $input.first().json.body.arxiv_url;
const paperId = url.split('/abs/')[1];
return [{ json: { paper_id: paperId, arxiv_url: url } }];
```

**Output:**
```json
{
  "paper_id": "2301.07041",
  "arxiv_url": "https://arxiv.org/abs/2301.07041"
}
```

---

### Node 3: HTTP Request (Fetch from arXiv)
**What it does:** Calls the arXiv API to get paper metadata in XML format.

**Why arXiv returns XML:** arXiv uses the Atom XML format (a standard for academic APIs). Unlike REST APIs that return JSON, arXiv's API returns structured XML containing the paper's title, authors, abstract, and category.

**Configuration:**
- Method: GET
- URL: `https://export.arxiv.org/api/query?id_list={{ $json.paper_id }}&start=0&max_results=1`
- Response Format: Text (because we need raw XML, not parsed JSON)
- Headers: `User-Agent: n8n-arxiv-summarizer/1.0`

**⚠️ Rate Limiting:** arXiv limits requests to ~3 per second per IP address. If you send too many requests during testing, your IP gets temporarily blocked for 15-60 minutes. Always wait between test runs.

---

### Node 4: Code in JavaScript1 (Parse XML)
**What it does:** Parses the raw XML response to extract structured fields.

**Why we need it:** The arXiv API returns raw XML text. n8n can't automatically understand XML structure, so we use JavaScript regex to extract the specific fields we need.

**Fields extracted:**
- `title` — Paper name
- `abstract` — Full paper abstract
- `authors` — Comma-separated list of author names
- `category` — arXiv subject category (e.g. cs.CR, cs.AI)
- `arxiv_url` — Clean URL to the paper

---

### Node 5: HTTP Request1 (Anthropic API — Generate Summary)
**What it does:** Sends the paper abstract to Claude AI and gets back a plain-language summary.

**Why Anthropic instead of OpenAI node:** The n8n OpenAI node had an SSL certificate error on this server (`ERR_TLS_CERT_ALTNAME_INVALID`). Using a direct HTTP Request to the Anthropic API bypassed this issue entirely.

**Configuration:**
- Method: POST
- URL: `https://api.anthropic.com/v1/messages`
- Headers:
  - `x-api-key`: your Anthropic API key (no "Bearer" prefix needed)
  - `anthropic-version`: `2023-06-01`
  - `Content-Type`: `application/json`
- Body:
```json
{
  "model": "claude-haiku-4-5-20251001",
  "max_tokens": 150,
  "messages": [{
    "role": "user",
    "content": "Summarize this research paper abstract in 2-3 sentences for non-experts: {{ $json.abstract }}"
  }]
}
```

---

### Node 6: Code in JavaScript2 (Extract Summary Text)
**What it does:** Pulls the summary text out of the Anthropic API response structure.

**Why we need it:** The Anthropic API returns a nested JSON object. This node navigates the structure to extract just the plain text summary, and also carries forward all other fields (title, authors, etc.) for the next node.

---

### Node 7: Code in JavaScript3 (Build Notion Request Body)
**What it does:** Constructs the properly formatted JSON body for the Notion API.

**Why we need it:** The Notion API requires a very specific JSON structure with property types explicitly declared. Building this in a Code node allows us to:
- Clean strings (remove newlines, escape quotes)
- Truncate long text to Notion's 2000 character limit
- Format each property type correctly (title, rich_text, select, url)

**Key challenge:** Papers with many authors (like GPT-4 with 100+ authors) exceed Notion's 2000 character limit, so truncation is essential.

---

### Node 8: HTTP Request2 (Create Notion Page)
**What it does:** Calls the Notion API to create a new page (row) in the database.

**Why direct HTTP Request instead of Notion node:** The built-in Notion n8n node returned an error: "Databases with multiple data sources are not supported in this API version." Using a direct HTTP Request to `https://api.notion.com/v1/pages` bypassed this limitation.

**Configuration:**
- Method: POST
- URL: `https://api.notion.com/v1/pages`
- Headers:
  - `Authorization`: `Bearer your-notion-token`
  - `Notion-Version`: `2022-06-28`
  - `Content-Type`: `application/json`
- Body: `{{ $json.notionBody }}` (the pre-built JSON from previous node)

---

### Node 9: Respond to Webhook
**What it does:** Sends a success response back to whoever triggered the webhook.

**Why we need it:** When using "Respond to Webhook Node" mode, the workflow waits until this node executes before sending any response. This means the caller gets a confirmation only after the paper has been successfully saved to Notion.

**Response:**
```json
{
  "success": true,
  "message": "Paper added successfully"
}
```

---

## What is ReqBin and Why We Use It

**ReqBin** (reqbin.com) is a free online tool for sending HTTP requests directly from your browser — like Postman but without installing anything.

### Why We Need It
The workflow is triggered by a **POST request** (not a simple browser visit). You can't trigger a POST request just by typing a URL in your browser — browsers only send GET requests when you type a URL. ReqBin lets us:
- Choose the HTTP method (GET, POST, PUT, etc.)
- Add headers like `Content-Type: application/json`
- Write a JSON body with our arXiv URL
- Send the request and see the response

### How We Use It
1. Go to reqbin.com
2. Set method to **POST**
3. Enter the webhook URL: `https://your-n8n-instance/webhook-test/arxiv-summarizer`
4. Click **Body** tab → select **JSON**
5. Paste: `{"arxiv_url": "https://arxiv.org/abs/2301.07041"}`
6. Click **Send**

### Alternatives to ReqBin
- **Postman** — desktop app, more features
- **curl** — command line tool
- **Browser console** — using `fetch()` in developer tools
- **VS Code REST Client** — extension for VS Code

---

## Key Challenges Encountered

### 1. arXiv Rate Limiting
**Problem:** arXiv blocked our n8n server IP after too many test requests.
**Solution:** Wait 30-60 minutes between test sessions. In production, add retry logic with exponential backoff.

### 2. Notion "Multiple Data Sources" Error
**Problem:** The built-in Notion n8n node couldn't write to the database.
**Solution:** Used direct HTTP Request to Notion API instead of the n8n Notion node.

### 3. SSL Certificate Error with OpenAI Node
**Problem:** `ERR_TLS_CERT_ALTNAME_INVALID` when using n8n's OpenAI node.
**Solution:** Switched to Anthropic API via direct HTTP Request node.

### 4. Author List Too Long
**Problem:** GPT-4 paper has 100+ authors, exceeding Notion's 2000 character limit.
**Solution:** Added truncation function in Code in JavaScript3.

### 5. XML Parsing
**Problem:** arXiv returns XML, not JSON, which n8n can't parse automatically.
**Solution:** Used JavaScript regex in a Code node to extract fields from raw XML.

---

## Papers Added to Notion

| Title | Authors | Category |
|---|---|---|
| Verifiable Fully Homomorphic Encryption | Alexander Viand, Christian Knabenhans, Anwar Hithnawi | CS |
| GPT-4 Technical Report | OpenAI (multiple authors) | CS |
| LLaMA 2 / Code Llama | Meta AI Research | CS |
