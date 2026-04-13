# Autoreason — Self-Refinement with Borda Count Judges

**Source:** https://github.com/NousResearch/autoreason
**Authors:** SHL0MS / HERMES AGENT
**Paper:** `paper/autoreason.pdf`

Self-reflection loops fail for 3 structural reasons: **prompt bias**, **scope creep**, and **lack of restraint**.
Autoreason fixes all three using a tournament architecture where "do nothing" is always a first-class option.

---

## Core Method

```
Task Prompt → Incumbent A (unchanged)
                  ↓
        ┌─── Critic ───→ Critique (prompt bias elimination)
        ├─── Author B ──→ Adversarial revision
        └─── Synthesizer → Synthesis (AB)
                  ↓
          3-Judge Panel (fresh agents, Borda count)
                  ↓
         Winner → new A  (or converge if A wins 2×)
```

**Key insight:** Judges have NO shared context with the author — eliminates prompt bias.

**"Do nothing" is a first-class option:** If incumbent A wins, nothing changes. This prevents scope creep.

---

## Borda Count Voting

Each judge ranks all candidates. Points: last place=0, ..., first place=(N-1).
Sum across judges. Highest total wins.

```python
def aggregate_rankings(rankings, labels):
    """
    rankings: list of ranked lists, e.g. [['A','B','AB'], ['B','AB','A']]
    labels: ['A','B','AB']
    Returns: {label: borda_score}
    """
    n = len(labels)
    scores = {l: 0 for l in labels}
    for ranking in rankings:
        for rank, item in enumerate(ranking):
            scores[item] += (n - 1 - rank)  # first=2pts, second=1pt, third=0pts
    return scores
```

---

## When to Use Autoreason

Use when:
- Output quality matters more than speed (async/costly)
- Model tends to over-revise (scope creep)
- Model hallucinates flaws when critiquing (prompt bias)
- Simple single-pass is insufficient

Don't use when:
- Speed is critical (each pass = 3+ LLM calls + judge calls)
- Input is trivial / low-stakes
- Compute budget is tight

**Transition point:** At ~60% base accuracy, autoreason gains vanish — the generation-evaluation gap has closed. Below that threshold, autoreason helps significantly.

---

## Applying Autoreason to Intent Routing

The NL intent router currently uses static regex matching. Autoreason suggests a **confidence-gated tournament** approach:

```python
async def route_with_confidence(text: str, ctx: dict) -> dict:
    """Route intent with autoreason-style confidence check."""

    # 1. Run intent detector (fast, no LLM)
    primary_intent = detect_intent(text)
    confidence = get_confidence(text, primary_intent)

    # 2. Low confidence? Run the autoreason tournament
    if confidence < 0.7:
        candidates = {
            'market_analysis': await handle_market_analysis(text, ctx),
            'earnings_query':  await handle_earnings_query(text, ctx),
            'system_status':   await handle_system_status(text, ctx),
            'agent_status':   await handle_agent_status(text, ctx),
            'unknown':         await forward_to_zeroclaw(text, ctx),
        }

        # Judges score each (no context sharing)
       judges = await spawn_fresh_judges(3)
        rankings = await asyncio.gather(*[
            judge_rank(candidates, text) for judge in judges
        ])

        # Borda count
        scores = aggregate_rankings(rankings, list(candidates.keys()))
        winner = max(scores, key=scores.get)

        # "Do nothing" check: if primary won AND high confidence → use primary
        if scores[primary_intent] >= scores[winner]:
            return {'intent': primary_intent, 'response': candidates[primary_intent],
                    'method': 'regex', 'confidence': confidence}

        return {'intent': winner, 'response': candidates[winner],
                'method': 'autoreason', 'scores': scores}

    # 3. High confidence — fast path
    return {'intent': primary_intent,
            'response': await HANDLERS[primary_intent](text, ctx),
            'method': 'regex', 'confidence': confidence}
```

---

## Code References

| File | Purpose |
|------|---------|
| `experiments/v2/run_overnight.py` | Main experiment runner — `run_autoreason()`, `run_baseline()`, `run_5way_judge()` |
| `experiments/v2/compute_stats.py` | Bootstrap CIs, McNemar tests |
| `paper/autoreason.pdf` | Full paper with scaling curves and ablation results |

Key functions in `run_overnight.py`:
- `call_llm()` — async LLM calls with retry (max_retries=8)
- `aggregate_rankings()` — Borda count implementation
- `run_autoreason_pass()` — one iteration of the tournament
- `parse_ranking()` — extract rankings from judge text output
