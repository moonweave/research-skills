---
name: arxiv-monitor
description: "Fetches new arXiv papers matching a research watchlist, scores relevance with LLM, and delivers a filtered daily digest. Invoke when the user wants to check new arXiv papers on a topic, set up a daily paper monitoring routine, asks 'what's new on arXiv about X?', or wants to scan recent preprints in their field. Also use as a Claude Code Routine template for scheduled daily briefings. Do NOT invoke for finding a specific known paper (use ref-verify) or for general web research."
---

# arxiv-monitor — Daily arXiv Research Digest

Searches arXiv for papers matching your watchlist, scores each paper's relevance to your specific research focus, and returns a ranked digest. Designed for both on-demand queries and as a Claude Code Routine running on a daily schedule.

---

## Watchlist format

Before running, identify the user's watchlist. If not provided, ask for:

1. **Keywords** — search terms for arXiv queries (e.g., "ionic polymer actuator", "dielectric elastomer", "soft robotics strain")
2. **Research focus** — 2-3 sentences describing the user's specific interest (used for LLM relevance scoring, not just keyword matching)
3. **Authors to track** — specific researchers whose new work should always surface
4. **Lookback window** — how many days back to search (default: 7 for on-demand, 1 for daily Routine)

If the user has used this skill before, check if a watchlist already exists at `~/.arxiv-monitor-watchlist.json`. If it does, use it unless the user asks to update it.

---

## Search protocol

### Step 1 — Query arXiv

For each keyword group, query the arXiv API:

```
https://export.arxiv.org/api/query?search_query={query}&sortBy=submittedDate&sortOrder=descending&max_results=20&start=0
```

Build queries by combining keywords logically. For a user studying IPMC and soft actuators:
- Query 1: `all:ionic+polymer+metal+composite+actuator`
- Query 2: `all:dielectric+elastomer+actuation`
- Query 3: `all:electroactive+polymer+strain`

For author tracking:
- Query: `au:{lastname}_{firstname_initial}`

Fetch all queries, then deduplicate by arXiv ID.

### Step 2 — Filter by date

Keep only papers submitted within the lookback window. The `<published>` field in the API response is the submission date in ISO format.

Discard any paper with a submission date older than the lookback window. If the API returns fewer papers than expected, note this — arXiv has submission gaps on weekends.

### Step 3 — Score relevance

For each remaining paper, score relevance 1-5 based on the user's research focus:

- **5 — Core match**: directly addresses the user's specific research question or materials system
- **4 — Adjacent**: same domain, different angle (e.g., user studies actuation, paper is about sensing with same material)
- **3 — Related**: same broad field, some overlap with user's work
- **2 — Peripheral**: same general area but minimal overlap
- **1 — Off-topic**: matched a keyword incidentally but not relevant

Score based on title + abstract together. Do not score based on the keyword query alone — a paper matching "soft robot" may be entirely about locomotion control with no materials content.

Discard papers scoring 1. Surface all papers scoring 2+ but clearly separate by relevance tier.

### Step 4 — Format digest

Produce one entry per paper, sorted by relevance score (5→2):

```
─────────────────────────────────────────────────────
[5] Highly stretchable ionic polymer actuator with self-healing Nafion membrane
    Authors: Kim J., Park S., Lee H.W.
    Submitted: 2026-05-31 | arXiv: 2605.12345
    Abstract highlight: "achieves 18% tip deflection at 2V, recovers 95% performance after damage"
    Why relevant: Direct match — Nafion-based IPMC with self-healing, same voltage range as your work.
─────────────────────────────────────────────────────
[4] Dielectric elastomer sensors for proprioceptive feedback in soft grippers
    Authors: Chen X., Wang Y.
    Submitted: 2026-05-30 | arXiv: 2605.11892
    Abstract highlight: "capacitance change of 340% at 150% strain"
    Why relevant: Same material class (DEA), sensing application adjacent to actuation.
─────────────────────────────────────────────────────
```

At the end, a summary line:
```
Found: 12 papers in window | Scored ≥2: 7 | Score 5: 1 | Score 4: 2 | Score 3: 3 | Score 2: 1 | Discarded: 5
```

---

## Saving the watchlist

After the first successful run, offer to save the watchlist for future use:

```json
{
  "keywords": ["ionic polymer actuator", "dielectric elastomer actuation"],
  "research_focus": "IPMC actuators based on Nafion with platinum electrodes, focusing on strain output, blocking force, and long-term stability in air. Comparing against dielectric elastomers for wearable soft robotics.",
  "track_authors": ["Shahinpoor M", "Bar-Cohen Y"],
  "lookback_days": 7,
  "last_run": "2026-06-01",
  "seen_ids": []
}
```

Save to `~/.arxiv-monitor-watchlist.json`. On future runs, skip papers whose arXiv ID is already in `seen_ids`, then append new IDs. This prevents the same paper appearing in multiple weekly digests.

---

## As a Claude Code Routine

To run this automatically every morning, the user can set up a Routine:

**Schedule**: Daily at 08:00 KST (23:00 UTC previous day)
**Prompt**: "Run arxiv-monitor with my saved watchlist at ~/.arxiv-monitor-watchlist.json and send the digest to [Slack/Obsidian/email per user preference]"

The skill works identically in Routine mode — it loads the watchlist, searches with lookback_days=1, scores, formats, and outputs. The Routine trigger handles scheduling.

---

## Output routing

When running as a Routine or when the user specifies a destination:

**Obsidian**: Write to `{vault}/inbox/arxiv-{date}.md` with the digest as a note
**Slack**: Format as a Slack message block (truncate abstracts to 280 chars)
**Plain file**: Write to `~/arxiv-digest-{date}.txt`
**Terminal only** (default): Output directly in the conversation

If no destination is specified, output in the conversation and offer to save.

---

## Low-coverage domain detection

Some research domains publish primarily in journals and post rarely or never on arXiv. This is common in:
- Ionic polymer / IPMC actuators (Smart Materials and Structures, Sensors and Actuators A)
- Polymer synthesis (Macromolecules, ACS Applied Materials)
- Biomedical devices (many clinical journals)

If the most recent paper for a keyword query is older than 30 days, the domain likely has low arXiv coverage. In this case:

1. Note it explicitly: *"This domain has low arXiv coverage — most recent paper is X days old. Supplementing with Semantic Scholar is recommended."*
2. Run a Semantic Scholar fallback search:
   `https://api.semanticscholar.org/graph/v1/paper/search?query={keywords}&fields=title,authors,year,publicationDate,abstract&limit=10&sort=publicationDate`
3. Include any Semantic Scholar results from the lookback window in the digest, labeled `[S2]` instead of `[arXiv]`

**Author tracking fallback**: If an author has zero arXiv presence after multiple query variants, recommend Semantic Scholar author alerts instead:
`https://api.semanticscholar.org/graph/v1/author/search?query={name}&fields=name,paperCount,papers`
Note: "Author X has no arXiv presence. Set up a Semantic Scholar alert at semanticscholar.org/author/{id} to track their journal publications."

---

## Edge cases

**Weekend gap**: arXiv doesn't process submissions on weekends. Monday queries often return Friday's papers. If no papers found for a 1-day window, automatically extend to 3 days before reporting "no new papers."

**Preprint versions**: If a paper appears as v2 or later of an already-seen arXiv ID, skip it unless the title changed significantly (indicates a major revision worth noting).

**Rate limiting**: arXiv API allows ~3 requests/second. Add a 0.4s gap between queries if running more than 5 keyword groups.

**arXiv category filter**: For more precise results, add a category constraint to queries. For soft actuator research: `cat:cond-mat.soft+OR+cat:cond-mat.mtrl-sci+OR+cat:physics.app-ph`. Add to query string: `&search_query=({keywords})+AND+(cat:cond-mat.soft+OR+cat:physics.app-ph)`.
