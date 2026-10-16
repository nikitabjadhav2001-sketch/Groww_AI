# Review Intelligence → Weekly Product Pulse + Fee Explainer (n8n)

An n8n workflow that turns raw app-store reviews into:

1. a **weekly internal product pulse** (≤250 words), and
2. a **customer-facing fee explainer** (≤6 bullets, 2 official source links, "Last checked" date),

then, **only after human approval**, logs both to a Google Doc and creates a Gmail draft (nothing is auto-sent).

Product analysed: **Groww** (mutual funds), the same product chosen in Milestone 1.

---

## Fee issue identified from the reviews

**Exit load.** In the reviews, users report being surprised by a deduction when they redeem mutual fund units. The common complaints are:

- the exit load is not shown clearly before redemption,
- the holding-period rules are hard to understand (especially for SIP instalments),
- support replies do not explain the charge in plain terms.

The weekly pulse lists "Exit Load Transparency" as the top theme, and the fee explainer is generated from this issue.

> If you re-run the workflow on a different review export and the model identifies a different fee, update the sources in the **Fee Explainer** prompt to match.

---

## Workflow

```
Manual Trigger
  → HTTP Request (download reviews.csv from GitHub raw URL)
  → Extract Reviews (CSV → one item per review)
  → Prepare Reviews (Code: combine into one prompt-ready text)
  → Review Intelligence (LLM chain + Structured Output Parser)
        ├→ Weekly Product Pulse (LLM chain) → Format Weekly Pulse
        └→ Fee Explainer (LLM chain)        → Format Fee Explainer
  → Merge Outputs
  → Human Approval (Gmail: Send and Wait for Response)   ← approval gate
  → Approval Check (If: approved = true)
        ├→ Update a document (Google Docs: append entry)
        └→ Create Gmail Draft (no auto-send)
```

### Step 1: Review Intelligence
- Clusters reviews into a maximum of 5 themes.
- Picks the top 3 themes.
- Extracts 3 real, word-for-word user quotes.
- Flags 1 recurring fee/charge confusion with a supporting quote.
- Output is forced into JSON by a Structured Output Parser.

### Step 2: Weekly Product Pulse (internal)
Maximum 250 words: top themes, supporting quotes, a key observation, and 3 action ideas.

### Step 3: Fee Explainer (customer-facing)
Maximum 6 bullets, neutral and facts-only, 2 official source links, ending with `Last checked: <date>`. Built from the fee issue found in Step 1, with a prompt that restricts it to the listed sources.

---

## Where the approval (MCP-style gate) happens

**Node: `Human Approval`** (Gmail → *Send message and wait for response*, response type *Approval*).

- The workflow **pauses** and emails the pulse and fee explanation to the reviewer.
- The reviewer clicks **Approve** or **Disapprove** in the email.
- The next node, **`Approval Check`**, lets the run continue only if `approved` is `true`.
- If disapproved, nothing is written and no draft is created.

This is the simulated approval step allowed by the assignment: no action happens before a human approves.

### Actions after approval
1. **Update a document**: appends an entry to the Google Doc `Product Pulse Log` with:
   `date, top_themes, weekly_pulse, identified_fee_issue, explanation_bullets, source_links`
2. **Create Gmail Draft**: subject `Weekly Product Pulse + Customer Clarification — <date>`, body = weekly pulse + fee explanation (a reusable support snippet). **Not sent.**

---

## How to run

### Prerequisites
- An n8n account (Cloud or self-hosted)
- A **Groq API key**
- A Google account with Gmail and Google Docs access (OAuth credentials in n8n)
- A Google Doc named `Product Pulse Log` (blank)

### Setup
1. Upload `reviews.csv` to a **public GitHub repo** and copy the **Raw** URL
   (`https://raw.githubusercontent.com/<user>/<repo>/main/reviews.csv`).
2. In n8n, import `workflow.json` (or rebuild from the diagram above).
3. In **HTTP Request**, paste your raw GitHub URL (Response Format: File, field `data`).
4. In the three **Groq Chat Model** nodes, select your Groq credential and a chat model.
5. In **Human Approval** and **Create Gmail Draft**, select your Gmail credential and set your email address.
6. In **Update a document**, select your Google Docs credential and paste the URL of your `Product Pulse Log` doc.
7. Make sure the two official source links are present in the **Fee Explainer** prompt and in the **Update a document** text.

### Run
1. Click **Execute workflow**.
2. Wait for the approval email, read it, and click **Approve**.
3. Check Gmail → **Drafts** for the new draft and the Google Doc for the new entry.
4. To test the gate, run again and click **Disapprove**. Nothing should be created.

---

## Input data: `reviews.csv`

| Column | Description |
|---|---|
| Date | Review date (YYYY-MM-DD), covering about 8–12 weeks |
| User | Shortened display name |
| Rating | Star rating, 1–5 |
| Review | Review text |

Source: public Google Play Store reviews, exported on **[DATE]** using the `google-play-scraper` Python library. Date range: **[START DATE] to [END DATE]**. User names are shortened and personal data was removed.

---

## Deliverables in this repo

| File | Description |
|---|---|
| `README.md` | This file |
| `reviews.csv` | Reviews input sample |
| `workflow.json` | Exported n8n workflow |
| `weekly_pulse.md` | Generated weekly product pulse |
| `screenshots/` | Workflow canvas, approval email, Gmail draft, Doc entry |

---

## Sources

Official and regulatory sources used for the fee explanation:

1. Nippon India Mutual Fund investor education, Exit Load: https://mf.nipponindiaim.com/investoreducation/financial-term-of-the-week-exit-load
2. SEBI board meeting paper (September 2025), exit load limits: https://www.sebi.gov.in/sebi_data/meetingfiles/sep-2025/1758696854726_1.pdf
3. Edelweiss Mutual Fund, What is exit load: https://edelweissmf.com/investor-insights/mutual-fund-investment-tips-and-articles/what-is-exit-load-in-mutual-funds

---

## Notes and limitations

- **Approval is simulated** with n8n's built-in "send and wait" step, not a separate MCP server. Actions only run after approval.
- The LLM output varies between runs. Check that quotes are word for word from the CSV and that every fee-explainer bullet is supported by the listed sources before sending anything to customers.
- The SEBI link is a board meeting paper about exit load limits, not a current circular. For production use, cite the latest SEBI mutual fund circular.
- Groq free-tier rate limits can cause errors when several model nodes run at once. Using different models per node and enabling *Retry On Fail* helps.
- Review text is truncated and capped (most recent 150 reviews) to stay within token limits.
