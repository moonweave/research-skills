---
name: contradiction-finder
description: "Proactively detects contradictions between claims across a set of papers. Invoke when the user wants to find conflicts in a research area, asks 'do these papers agree?', 'what's the debate around X?', 'find contradictions in my literature set', provides a list of DOIs or a search query for conflict analysis, or is writing a literature review and wants to identify contested claims. Do NOT invoke for summarizing papers, finding papers on a topic (use arxiv-monitor), or verifying a single citation (use ref-verify)."
---

# contradiction-finder — Cross-Paper Conflict Detector

Identifies genuine contradictions between claims across a set of papers. The goal is not to summarize what each paper says in isolation, but to surface pairwise conflicts: where Paper A makes a claim that Paper B directly disagrees with, quantitatively contradicts, or meaningfully qualifies.

This is different from what Elicit and Semantic Scholar do — those tools show you the papers. This skill shows you the *conflicts between them*.

---

## What counts as a contradiction

Three types, in order of confidence:

**DIRECT** — Paper A explicitly asserts X; Paper B explicitly asserts not-X.
Example: A says "the primary mechanism is cation migration"; B says "the primary mechanism is electroosmotic water flow."

**QUANTITATIVE** — Both papers measure the same quantity under comparable conditions but report substantially different values.
Example: A reports "blocking force of 0.3 mN at 3V"; B reports "blocking force of 2.1 mN at 3V" for the same material class.
Note: values differing by ≤20% may reflect measurement uncertainty rather than genuine conflict — flag but note this.

**SCOPE** — A makes a general claim; B shows it holds only under specific conditions, or vice versa.
Example: A says "IPMC actuators degrade in air"; B says "ionic liquid-based IPMCs operate stably in air for 180,000 cycles."
This is often not a true contradiction but a boundary condition — note this distinction.

What is NOT a contradiction:
- Papers measuring different materials, fabrication methods, or test conditions (different systems, not conflicting claims)
- Papers at different scales (single-cell vs tissue-level, nanoparticle vs bulk)
- Papers from different eras where the later one has better data
- Disagreement about interpretation while agreeing on the data

When in doubt between SCOPE and DIRECT, prefer SCOPE — methodological differences explain more than they appear to.

---

## Protocol

### Step 1 — Build the paper set

**If DOIs provided**: First confirm each DOI resolves to a real paper before using it. Fetch via CrossRef (`https://api.crossref.org/works/{DOI}`) — a 404 means the DOI does not exist, and you must not invent an abstract for it. Then fetch each abstract via CrossRef and Semantic Scholar (same protocol as ref-verify, Layer 3). If a DOI is invalid, report it and exclude it. If fewer than 3 valid papers remain, warn the user that contradiction detection requires at least 3 sources.

**If search query provided**: Search arXiv and Semantic Scholar for the topic, fetch top 6-8 papers sorted by citation count (established papers are more likely to represent real positions in a debate). Fetch abstracts.

Limit: 8 papers maximum. Beyond this, the pairwise comparison becomes unwieldy (28 pairs for 8 papers). If more papers are provided, ask the user to select the most representative ones.

### Step 2 — Extract atomic claims

From each abstract, extract specific, falsifiable claims. Focus on:
- Mechanism claims ("the mechanism is X")
- Quantitative results ("property Y = Z under conditions W")
- Causation claims ("factor A causes effect B")
- Universality claims ("always", "all", "never", "generally")

Do not extract:
- Background statements ("X is important for Y applications")
- Motivation ("this study aims to...")
- Vague claims that cannot be falsified

Mark each claim with its source paper and verbatim excerpt. Example:
```
[CLAIM] Paper 1 (DOI:...): "the dominant actuation mechanism is cation migration toward the cathode"
— verbatim: "our results demonstrate that Cs⁺ migration is the primary contributor to bending deformation"
```

### Step 3 — Pairwise comparison

Compare each paper's claims against every other paper's claims. For each pair, ask:
1. Do any claims make assertions about the same physical quantity, mechanism, or relationship?
2. If yes: do they agree, conflict, or qualify each other?
3. If they conflict: is this a genuine contradiction or a methodological difference?

### Step 4 — Classify and filter

Assign conflict severity:
- **HIGH**: Direct contradiction on a core claim, same material system, no obvious methodological explanation
- **MEDIUM**: Quantitative disagreement beyond measurement uncertainty, or SCOPE conflict where the scope distinction matters for practical use
- **LOW**: Apparent contradiction likely explained by methodology, scale, or era difference

Discard LOW-severity conflicts from the main report — list them only in an appendix.

---

## Output Format

```
CONTRADICTION ANALYSIS
Papers analyzed: [N]
Date: [date]
────────────────────────────────────────────────────

HIGH-SEVERITY CONFLICTS
═══════════════════════

[1] DIRECT CONTRADICTION — [Short description]
────────────────────────────────────────────────────
Claim A: "[verbatim quote from abstract]"
  Source: [Author et al. (year)] | DOI: ... | arXiv: ...

Claim B: "[verbatim quote from abstract]"
  Source: [Author et al. (year)] | DOI: ... | arXiv: ...

Conflict: [1-2 sentence explanation of why these contradict]
Could explain: [Possible reasons: different cation types? different humidity? different electrode prep?]
────────────────────────────────────────────────────

[2] QUANTITATIVE DISAGREEMENT — [Short description]
...

MEDIUM-SEVERITY CONFLICTS
══════════════════════════
[list with abbreviated format]

SUMMARY
────────────────────────────────────────────────────
High:   [N] conflicts
Medium: [N] conflicts
Low:    [N] (listed in appendix, likely methodological)

Most contested claim: [the claim that appears in the most conflicts]
Suggested resolution: [What experiment or measurement would resolve the main conflict?]
────────────────────────────────────────────────────
```

---

## Handling common pitfalls

**Era mismatch**: A 2005 paper and a 2024 paper may appear to contradict when the 2024 paper simply has better instrumentation. Flag the year difference and note this as a possible explanation.

**Different material systems labeled similarly**: "dielectric elastomer" can mean VHB, silicone, polyurethane, or custom chemistries. If two papers use the same umbrella term but different materials, the "contradiction" may not be one.

**Selective abstract reporting**: Authors choose what to include in abstracts. A claim absent from an abstract doesn't mean the paper disagrees — it means the paper doesn't address that claim. Do not infer disagreement from silence.

**Systematic vs. anecdotal evidence**: A single-experiment result vs. a meta-analysis covering 50 studies is not a fair contradiction. Note the evidence level difference.

---

## Minimum viable output

If abstracts are available for fewer than 3 papers, or if no contradictions are found, still produce a useful output:
- List the claims extracted from each paper
- State explicitly: "No contradictions found between these papers at the abstract level. Either the papers address compatible questions, or the conflicts are in the body text (not the abstract)."
- Suggest: "To find deeper conflicts, provide full-text access or look for papers that specifically rebut each other."
