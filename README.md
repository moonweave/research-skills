# groundcheck

Verification skills for academic research. The shared rule is simple: claims must be grounded in fetched evidence, with uncertainty stated explicitly when evidence is missing.

## Install

```bash
# one skill
npx skills add moonweave/groundcheck@ref-verify -g

# full suite
npx skills add moonweave/groundcheck -g
```

Works with Claude Code, Cursor, Codex, and other `npx skills` compatible agents.

## Skills

| Skill | Use When | Status |
| --- | --- | --- |
| [`ref-verify`](ref-verify/SKILL.md) | Verify citations, DOIs, paper metadata, and whether a paper supports a specific claim. Topic claims can use abstracts; mechanism/implementation claims require full text and the checker gate. | stable |
| [`arxiv-monitor`](arxiv-monitor/SKILL.md) | Monitor new arXiv papers by query, author, or watchlist and produce a filtered digest. | stable |
| [`figure-verifier`](figure-verifier/SKILL.md) | Check whether a paper's text claims are actually supported by its figures. | beta |
| [`contradiction-finder`](contradiction-finder/SKILL.md) | Compare claims across papers and surface direct, quantitative, or scope conflicts. | stable |
| [`nrf-grant`](nrf-grant/SKILL.md) | Draft and review Korean NRF research-plan sections against NRF-style criteria. | stable |

## Source Of Truth

`ref-verify` has its own canonical repository:

- Canonical development repo: [`moonweave/ref-verify`](https://github.com/moonweave/ref-verify)
- Suite mirror path: [`groundcheck/ref-verify`](ref-verify/)

Develop `ref-verify` features in the standalone repo first, then mirror the same `SKILL.md`, `checker.py`, and tests into this suite before release. Groundcheck owns suite-level README, install routing, and cross-skill consistency.

## Principles

- Fetch live sources instead of relying on model memory.
- Quote or directly paraphrase the source text used for each content verdict.
- Separate unsupported, contradicted, partial, and unverifiable states.
- Keep each skill narrow enough that an agent can apply it reliably.
- Treat missing evidence as a result, not a gap to fill with inference.

## Limits

- These are agent skills, not a controlled benchmark suite.
- Figure reading is approximate and depends on image resolution and plot clarity.
- Many materials, polymer, and engineering papers do not expose abstracts or full text openly. In those cases, content verification should degrade to `UNVERIFIABLE` or `ABSTRACT-LEVEL ONLY`, not guessed support.

## Development Checks

For `ref-verify`:

```bash
cd ref-verify
python3 -m unittest tests/test_checker.py
```

Before changing a mirrored skill, confirm whether that skill has a standalone canonical repo. If it does, update the canonical repo first and mirror the result here.
