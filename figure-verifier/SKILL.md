---
name: figure-verifier
description: "Verifies whether a scientific figure actually supports quantitative claims in the paper text. Invoke when the user asks 'does Figure X show Y?', 'the paper claims Z% — does the figure actually show that?', 'is this graph consistent with the abstract?', 'check this figure against the results', or provides a figure image with a specific claim to verify. Also catches wrong figure pointers (value is correct but cited in wrong figure number) and characterizes whether a figure is quantitative or photographic before attempting to read it. Do NOT invoke for creating figures, plotting data, or general image description."
---

# figure-verifier — Figure Claim Verifier

The failure this skill prevents: a paper states a result in the abstract or results section, but the figure tells a different story — different magnitude, different trend, different units, or data that doesn't reach the claimed threshold. These discrepancies are easy to miss when reading quickly and impossible to catch without actually reading the figure data.

The core rule, parallel to citation verification: every numerical value you report from a figure must be read from the actual image you can see — not recalled from training data, not inferred from the paper's reputation, not estimated from the figure caption alone.

> **Maturity note:** The reading-precision of the image (VLM) path depends heavily on figure resolution and on the host model's vision capability. Reading a value off a dense plot is inherently approximate — prefer `APPROXIMATE` with a stated precision over `MATCH` unless the value is unambiguous. When a figure is too low-resolution or cluttered to read confidently, `UNREADABLE` is the correct, honest answer.

---

## What you need to run a check

**Required:**
- The figure image (user pastes it into the conversation, or you fetch it from an open-access source)
- The specific claim to check — either provided by the user or extracted from the abstract/results

**Optional but helpful:**
- The paper DOI (lets you fetch the abstract to find relevant claims automatically)
- The figure caption (helps identify what axes and conditions the figure represents)

If the user provides a DOI but no image, fetch the abstract to identify the specific claims, then tell the user which figure to provide and what you'll be checking against it.

---

## Verification Protocol

### Step 1 — Identify the claim

Extract the specific quantitative assertion being verified. Examples of well-formed claims:
- "IPMC actuators achieved 12% tip deflection at 3V applied voltage"
- "Young's modulus increased from 2.1 MPa to 8.4 MPa after crosslinking"
- "Actuation strain exceeded 100% at 150 MV/m"

If the user provides a DOI, fetch the abstract from CrossRef:
`https://api.crossref.org/works/{DOI}`

Extract every quantitative claim that references a figure. Ask the user which one(s) to check, or proceed with all of them if there are ≤3.

### Step 2 — Read the figure

Look at the provided image and systematically extract:

1. **Axis labels and units** — what physical quantity is on each axis? What are the units?
2. **Axis scale and range** — linear or log? What are the min/max values? Are there breaks?
3. **Data at the relevant condition** — what is the y-value at the x-value the claim references?
4. **Error bars or uncertainty** — if present, what is the spread?
5. **Which data series** — if multiple curves/bars, which one does the claim reference?

Be precise about what you can and cannot read. A figure at low resolution, with overlapping elements, or with unlabeled axes limits what can be reliably extracted. Say so explicitly rather than estimating.

### Step 3 — Compare

Map the figure reading to the claim:

| Figure reading | Claimed value | Verdict |
|---|---|---|
| Exact or within ~5% | Matches | MATCH |
| Within ~15% | Close but not exact | APPROXIMATE — note the difference |
| Substantially different | Disagrees | MISMATCH — state what you actually read |
| Cannot read reliably | — | UNREADABLE — state why |

Reasons for APPROXIMATE instead of MATCH: axis gridlines are coarse, data point falls between gridlines, error bars are large relative to the claimed value. When in doubt between MATCH and APPROXIMATE, use APPROXIMATE and explain.

---

## Output Format

```
FIGURE VERIFICATION
────────────────────────────────────────────────
Claim:    "[exact quote from paper text]"
Figure:   Fig. [N] — [brief description of what the figure shows]
Condition: [x-axis value or experimental condition being checked]

Figure reading:
  x-axis: [label] ([units]), range [min]–[max], [linear/log] scale
  y-axis: [label] ([units]), range [min]–[max], [linear/log] scale
  Read value at [condition]: [value] ± [uncertainty if readable]
  Data series: [which curve/bar/dataset]

Claimed value: [value from text]
Read value:    [value from figure]
Difference:    [absolute and % difference]

VERDICT: MATCH | APPROXIMATE (±X%) | MISMATCH | UNREADABLE
Notes: [what limits confidence, alternative readings, context]
────────────────────────────────────────────────
```

For multiple claims from the same paper, produce one card per claim then a summary:

```
SUMMARY
────────────────────────────────────────────────
Fig. 2, Claim 1 (strain at 3V): MATCH
Fig. 3, Claim 2 (modulus ratio): MISMATCH — figure shows 2.8×, paper claims 4.1×
Fig. 5, Claim 3 (response time): UNREADABLE — axis labels cut off
────────────────────────────────────────────────
```

---

## What to do with each verdict

**MATCH**: The figure supports the claim. State this clearly. The user can cite with confidence.

**APPROXIMATE**: The figure is consistent with the claim within the measurement uncertainty of the figure. Note the best-estimate reading and the precision limit (e.g., "±20 Hz based on gridline spacing"). The user should decide whether the difference matters for their purpose.

**MISMATCH**: The figure exists and is readable, but the data contradicts the claim. State exactly what you read and how it differs. Common causes: wrong data series (the value belongs to a different condition or actor), wrong units, genuine discrepancy. Do not speculate — just report.

**WRONG-FIGURE**: The claimed value may be numerically correct, but the figure cited does not contain it. This includes: (a) the figure is photographic/schematic with no quantitative axes, (b) the figure shows a different experimental condition than the claim specifies, (c) the data is in a different figure number. State which figure does contain the evidence if you can identify it. This is a common error in papers and worth flagging explicitly.

**UNREADABLE**: Something prevents reliable extraction — low resolution, cut-off axis labels, overlapping curves, unlabeled scales. State the specific obstacle. If a partial reading is possible (e.g., can read the range but not a precise point), provide it with a confidence qualifier.

---

## Common edge cases

**Log-scale axes**: When reading from a log-scale axis, values between gridlines are not linearly interpolated — they follow log spacing. State "log scale" explicitly and note if this affected precision.

**Inset figures**: Insets often use different scales than the main plot. Identify which scale applies to the data point being checked.

**Multiple y-axes**: Some figures have a left and right y-axis for different quantities. Confirm which axis the relevant data series uses before reading.

**Error bars larger than the claim**: If the error bars (±σ or ±SEM) on the data point are comparable in size to the claimed value itself, a precise match is physically impossible to verify from the figure. Report the center value and note the uncertainty.

**Normalized vs. absolute values**: Check whether the y-axis shows normalized values (relative to a baseline) or absolute values. Claims about "improvement" often refer to ratios — ensure you're comparing the right quantities.

**Figure panels (a, b, c, d)**: Specify exactly which panel you read from and confirm it matches the panel referenced in the paper text.

---

## Open-access figure fetching (when no image is provided)

If the paper is open access, attempt to locate the figure:

1. **arXiv**: `https://arxiv.org/abs/{arxiv_id}` — HTML version often has inline figures; PDF can be fetched at `https://arxiv.org/pdf/{arxiv_id}`
2. **PubMed Central**: `https://pmc.ncbi.nlm.nih.gov/articles/{PMCID}/` — figures often embedded in HTML
3. **Unpaywall**: `https://api.unpaywall.org/v2/{DOI}?email={user_email}` — check for OA PDF. Unpaywall requires a real email; ask the user for theirs rather than using a placeholder.

If no open-access version is available, tell the user which figure to provide and what claim you'll verify against it. Do not attempt to reconstruct or describe a figure you haven't seen.
