# research-skills

AI agent skills for academic researchers. Install individually or all at once.

Each skill is a verified, tested tool that fills a gap not covered by existing research software.

---

## Install

```bash
# individual skill
npx skills add moonweave/research-skills@ref-verify -g

# all skills (once available)
npx skills add moonweave/research-skills -g
```

Works with **Claude Code, Cursor, Codex**, and any `npx skills` compatible agent.

---

## Skills

### ref-verify — Citation hallucination prevention
**Status: stable (v1.0.0)**

Prevents AI agents from citing papers with wrong DOIs, wrong authors, wrong year, or fabricated content. Every citation is verified live against CrossRef, Semantic Scholar, and PubMed. Every content claim is traced to a verbatim-fetched abstract — never recalled from training data.

```bash
npx skills add moonweave/research-skills@ref-verify -g
```

**Real catches during testing:**
- Abstract content described by AI that doesn't appear in the actual CrossRef record
- DOI resolving to a completely different paper (different authors, different year)
- "500% strain" in an abstract that referred to a measurement condition, not an actuation result

→ [README](ref-verify/SKILL.md) · [Standalone repo](https://github.com/moonweave/ref-verify)

---

### arxiv-monitor — Daily arXiv briefing
**Status: stable (v1.0.0)**

Scheduled monitoring of new arXiv papers by keyword and author watchlist. LLM-based relevance scoring (1-5 tiers). Routable output to Slack, Obsidian, Telegram, or file. Built for Claude Code Routines. Auto-detects low-arXiv-coverage domains and falls back to Semantic Scholar.

**Why build this:** Scholar Inbox handles passive email digests well. The gap is agent-native monitoring with custom filtering and routing that connects directly to your research workflow.

```bash
npx skills add moonweave/research-skills@arxiv-monitor -g
```

→ [SKILL.md](arxiv-monitor/SKILL.md)

---

### figure-verifier — Text-vs-figure claim verification
**Status: stable (v1.0.0)**

Reads numerical values from scientific figures, then cross-checks them against quantitative claims in the paper's text. Flags discrepancies, wrong figure pointers (value cited in the wrong figure number), and characterizes whether a figure is quantitative or photographic before reading.

**Why build this:** WebPlotDigitizer and PlotPick extract data from figures. No existing tool — anywhere — verifies whether a paper's stated results are actually supported by its own figures. This is the figure equivalent of what ref-verify does for citations.

```bash
npx skills add moonweave/research-skills@figure-verifier -g
```

→ [SKILL.md](figure-verifier/SKILL.md)

---

### contradiction-finder — Cross-paper conflict detection
**Status: stable (v1.0.0)**

Takes a set of papers (by DOI or search query), extracts atomic claims from abstracts, compares them pairwise, and surfaces conflicts (DIRECT / QUANTITATIVE / SCOPE) with verbatim evidence. Its core discipline: it does not infer a contradiction from a claim a paper's abstract doesn't actually make.

**Why build this:** Scite detects contradictions at the citation-graph level but only reactively (you supply the claim). Elicit shows divergent values but doesn't call them out as conflicts. No tool proactively maps contradictions from a paper set.

```bash
npx skills add moonweave/research-skills@contradiction-finder -g
```

→ [SKILL.md](contradiction-finder/SKILL.md)

---

### nrf-grant — NRF 연구계획서 작성
**Status: stable (v1.0.0)**

한국연구재단(NRF) 연구계획서 작성 전용 스킬. 필요성·연구내용·활용방안 섹션 구조, 우수성·필요성·실현가능성·활용방안 4대 심사기준 정렬, 한국어 학술 문체, 과제 유형별(신진/중견/선도연구센터/BK21) 특이사항 반영.

**Why build this:** NSF/NIH용 AI 도구는 포화 상태. NRF 형식을 지원하는 도구는 전 세계에 없음.

```bash
npx skills add moonweave/research-skills@nrf-grant -g
```

→ [SKILL.md](nrf-grant/SKILL.md)

---

## Design principles

All skills in this collection follow the same rules:

1. **Live verification over memory recall** — claims are grounded in fetched data, not training recall
2. **Explicit uncertainty** — if something can't be verified, it says so instead of guessing
3. **Narrow scope** — each skill does one thing well; no feature creep
4. **Evidence in output** — every verdict includes the source and evidence that produced it

---

## Related

- [moonweave/ref-verify](https://github.com/moonweave/ref-verify) — standalone repo for ref-verify (v1.0.0, stable)
- [moonweave/anneal-skill](https://github.com/moonweave/anneal-skill) — measure-first decision discipline
- [Moon-python/decide-skill](https://github.com/Moon-python/decide-skill) — decision automation for non-expert domains
