# Project Specification: rss-research-agent

Status: Draft
Target Platform: Docker Compose container / Podman nodejs service
Dependencies:
  - OpenRouter
  - Docker Compose
  - Freshrss
  - Signal

 
## 1. Executive Summary
The rss-research-agent is a personal knowledge synthesis engine. It transforms passive content consumption (Starring an article) into an active research and publication workflow. It utilizes a state-persistent architecture to ensure resilience and a multi-model approach to balance reasoning quality with cost-efficiency.

## 2. System Architecture & State Model

### 2.1 The SQLite Ledger

The `tasks` table acts as the transactional record.

| Column | Type | Purpose |
| :--- | :--- | :--- |
| `id` | UUID | Primary Key. |
| `status` | TEXT | `PENDING`, `TRIAGED`, `AWAITING_GUIDANCE`, `RESEARCHING`, `SYNTHESIZING`, `REVIEW`, `COMPLETED`. |
| `source_url` | TEXT | Original URL from FreshRSS. |
| `title` | TEXT | Article title. |
| `planner_json` | TEXT | JSON output from the Triage Prompt (the 3 paths). |
| `selected_path`| INT | The path Neal selected via Signal or Web UI (1, 2, or 3). |
| `raw_research` | TEXT | Raw data returned from Kagi FastGPT. |
| `draft_md` | TEXT | Final synthesized Markdown before user edits. |
| `user_commentary`| TEXT | Neal’s manual notes added via Web UI or Signal. |
| `slug` | TEXT | URL-friendly title for the wiki and picks feed. |

## 3. Prompt Engineering (The "Brain")

These prompts utilize XML tagging to provide clear boundaries for the Cloud LLMs, ensuring structural integrity and preventing hallucinations.

### 3.1 The Triage Prompt (The Planner)

Model: Claude 3.5/4.7 Sonnet
State Transition: PENDING => TRIAGED

```
<system_role>
You are the Lead Research Planner. Your goal is to analyze an article and provide 
3 distinct, high-value research trajectories that expand upon the original content.
</system_role>

<article_input>
Title: {{title}}
Content: {{full_text}}
</article_input>

<instructions>
1. Analyze the core thesis of the article.
2. Generate a 2-sentence executive summary.
3. Propose 3 mutually exclusive research paths:
   - Path A: Technical/Implementation Deep Dive.
   - Path B: Contextual/Historical Impact.
   - Path C: Synthesis/Contradiction (Conflicting viewpoints).
4. For each path, generate a Kagi FastGPT optimized search query.
</instructions>

<output_schema>
Return ONLY JSON:
{
  "summary": "...",
  "paths": [
    { "id": 1, "label": "Technical", "description": "...", "kagi_query": "..." },
    { "id": 2, "label": "Contextual", "description": "...", "kagi_query": "..." },
    { "id": 3, "label": "Synthesis", "description": "...", "kagi_query": "..." }
  ]
}
</output_schema>
```

### 3.2 The Research Synthesis Prompt (The Worker)

Model: Claude 3.5 Opus
State Transition: RESEARCHING => SYNTHESIZING

```
<context>
Article: {{article_title}}
Selected Path: {{path_description}}
Kagi Research Data: {{kagi_results}}
</context>

<instructions>
1. Synthesize the Kagi data with the original article context.
2. Maintain an even, objective, and realistic tone.
3. Format in Markdown for a technical wiki.
4. Use headers (##, ###) and bullet points for scannability.
5. If data is missing or irrelevant, tag the top with [NEAL_GUIDANCE_REQUIRED].
</instructions>

<output_format>
Markdown content only. No conversational filler.
</output_format>
```

### 3.3 The Judge Prompt (The Watchdog)

Model: Gemini 3.1 Flash
```
State Transition: SYNTHESIZING => REVIEW
<draft_report>
{{final_markdown}}
</draft_report>

<audit_criteria>
- Is the tone objective (not obsequious or hyper-positive)?
- Does the research directly answer the chosen path?
- Are there any hallucinated links or placeholders?
</audit_criteria>

<output_format>
Return "PASSED" or "FAILED: [Specific Reason]".
</output_format>
```

## 4. Interaction Channels

### 4.1 Signal (Mobile Workflow)

- Discovery: Agent sends the summary and paths via "Note to Self."
- Action: You reply "1", "2", or "3". The Agent updates SQLite and starts the RESEARCHING state.
- Final Review: Agent sends a link to the Web UI or a snippet for approval.

### 4.2 Web UI (Editorial Desk)
- Side-by-Side View: Agent Research (Left) | Neal's Commentary Markdown (Right).
- Picks Integration: When Neal writes in the Right pane, it populates the user_notes column.
- Publish: Finalizes the /wiki/picks/[slug] page and pushes to Atom feeds.

### 5. Resilience & Reliability
- Atomic Host Compatibility: Podman volumes use the :Z flag for Bazzite/SELinux compliance.
- OpenRouter Failover: If the primary model (Claude) fails, the system retries with a secondary model (GPT-4o) automatically.
- State Persistence: Every LLM response is written to SQLite before the next step is triggered, ensuring no work is lost on restart.

===

Below is original source from Claude, which was more deterministic and needs to be clearned up & some of this needs to be re-intergrated into above

# freshrss-agent

Star an article in NewsBlur → FreshRSS picks it up → research agent runs →
wiki page published → two RSS feeds updated.

## What it produces

```
/wiki/picks/          Neal's Picks index (public)
/wiki/picks/[slug]    Individual pick page (public, editable by you)
/wiki/research/       Research Reports index (public)  
/wiki/research/[slug] Individual report page (public)
/feeds/picks.xml      Neal's Picks Atom feed
/feeds/research.xml   Research Reports Atom feed
/edit/[slug]          Edit commentary (basic auth required)
```

## Setup

```bash
cp .env.example .env
# fill in .env values

docker compose up -d
```

Then in Nginx Proxy Manager:

- Add a proxy host → `http://agent:3000`
- Enable SSL (Let’s Encrypt)
- Done

Add `https://wiki.yourdomain.com/feeds/picks.xml` to FreshRSS as your
**research feed**. Copy its stream ID into `FRESHRSS_RESEARCH_FEED_ID`
in `.env` — this is the recursion guard.

## FreshRSS API password

Settings → Profile → API management → set a password there.
This is separate from your login password.

## Editing commentary

Visit `/edit/[slug]` in a browser. Basic auth prompt appears.
Write Markdown. Save. Pick page and picks feed regenerate immediately.

## Adding real search

In `src/research-agent.js`, replace `stubSearch()` with a real call:

```js
// Brave Search API (good for self-hosters, $3/month free tier)
async function realSearch(query) {
  const res = await fetch(
    `https://api.search.brave.com/res/v1/web/search?q=${encodeURIComponent(query)}&count=3`,
    { headers: { "X-Subscription-Token": process.env.BRAVE_API_KEY } }
  );
  const data = await res.json();
  return JSON.stringify({
    query,
    results: data.web.results.map(r => ({ url: r.url, snippet: r.description }))
  });
}
```

## Adding real images

Replace `stubFetchImage()` with Wikimedia Commons API or Unsplash:

```js
// Wikimedia Commons (free, CC licensed)
async function realFetchImage(query, slug, index) {
  const res = await fetch(
    `https://commons.wikimedia.org/w/api.php?action=query&generator=search&gsrsearch=${encodeURIComponent(query)}&gsrnamespace=6&prop=imageinfo&iiprop=url|extmetadata&format=json`
  );
  // ... save first result to data/wiki/media/[slug]/
}
```

## Data

Everything lives in `./data/` which is mounted as a volume.
SQLite db: `data/reading-log.db`
Wiki HTML: `data/wiki/`
Media:     `data/wiki/media/[slug]/`

Safe to `docker compose down && docker compose up -d` — data persists.
