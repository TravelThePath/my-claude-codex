# Story Point Estimation

Fibonacci scale: **0, 1, 2, 3, 5, 8**. Points combine effort, time, complexity, and risk / uncertainty — they are not hours. **8 is the cap** — anything that feels larger must be split before it gets a score.

## Scale

| Pts | Effort | Time | Complexity | Risk | Example |
|-----|--------|------|------------|------|---------|
| 0 | None | None | None | None | Story container with no work beyond child tasks |
| 1 | Trivial | Minutes | None | None | Copy change, trivial config tweak |
| 2 | Small | Hours | Low | None | Simple logic change with clear requirements |
| 3 | Mild | ~1 day | Low | Low | Small feature or bug fix, limited scope |
| 5 | Moderate | Few days | Medium | Moderate | Touches multiple components or needs design decisions |
| 8 | Severe (cap) | ~1 week | Medium-High | Moderate-High | Large feature, refactor, or notable unknowns — the upper limit before splitting |

## How to estimate

1. Read **Context** and **Changes** — internalize scope and the named touchpoints.
2. Compare relatively — "more like a 3 or a 5?", not "how many hours".
3. Weigh the four dimensions: effort, time, complexity, risk / uncertainty.
4. Apply bias rules:
   - Low uncertainty + tedious work → bias **down**.
   - High uncertainty or cross-team coordination → bias **up**, even when the code change is small.
   - Prefer **5 over 8** unless risk or unknowns clearly justify the jump.
   - **Never beyond 8** — if the work feels larger, split it into smaller issues before estimating.

## Story-label issues

Parent containers whose work lives in child task issues. Each child is estimated individually, so the parent must not double-count:

- Default the parent to **0 or 1**.
- Higher values only when the parent itself carries coordination overhead beyond what the child tasks capture.

## Output format

```
**Story point: <N>** — <one sentence naming the dimension(s) that drove the score>.
```

Example: `**Story point: 2** — single-entity schema addition plus one filter parameter on a list endpoint; well-trodden path, low risk.`

If the issue lacks enough detail to estimate confidently, say so and name what's missing — do not guess.
