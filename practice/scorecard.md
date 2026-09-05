# Self-Assessment Scorecard

Score each dimension 0-3 immediately after an attempt, against the worked example and the [principal signals rubric](../vocabulary/principal-engineer-signals.md). Copy the table into your reflection or log.

## Scoring Scale

| Score | Meaning |
|-------|---------|
| **0** | Did not address it, or addressed it only when the follow-up forced it |
| **1** | Addressed it generically ("we'd add monitoring"; "we'd handle failures") |
| **2** | Addressed it specifically, with a mechanism and a trade-off |
| **3** | Addressed it specifically, *unprompted*, and connected it to the dominant constraint or to another dimension |

## The Dimensions

| # | Dimension | What a 3 looks like | Score |
|---|-----------|--------------------|-------|
| 1 | **Reframing** | Named the dominant requirement and one thing the prompt got wrong or left dangerously ambiguous, in the first 5 minutes | |
| 2 | **Non-goals** | Stated 2-3 explicit non-goals and why each is out | |
| 3 | **Derived constraints** | Turned the estimate into 2-3 sentences of the form "this number forces this decision" | |
| 4 | **Sensitivity** | Identified which number the design is most sensitive to and what changes if it is off by 10x | |
| 5 | **Critical path** | Walked the most important write (or read) path step by step, including where idempotency and durability are enforced | |
| 6 | **Deep dive selection** | Spent the deep-dive time on the *hardest* sub-problem, not the most familiar one | |
| 7 | **Deep dive depth** | Went to the level of mechanism: the key, the lock order, the exact condition, the exact failure case | |
| 8 | **Invariants** | Stated what must always be true and pointed to where each is enforced and how it is verified | |
| 9 | **Relaxed invariants** | Named where an invariant is deliberately relaxed and what compensates | |
| 10 | **Failure modes** | Covered component, dependency, *and correlated* failures with blast radius for each | |
| 11 | **Degradation order** | Stated what is sacrificed first, second, third, and who or what decides | |
| 12 | **Evolution** | Gave a v1 and the path to the end state under live traffic; named the one-way doors | |
| 13 | **Operations** | Deploy safety, rollback time, what pages, and whether each page is actionable | |
| 14 | **Cost** | Named the dominant cost driver and the knob that controls it | |
| 15 | **10x** | Named the first and second things that break | |
| 16 | **Alternatives** | Named 2-3 rejected alternatives with a *specific* losing reason each | |
| 17 | **Communication** | Checked in with the interviewer, adjusted to steering, stated decisions rather than options | |
| 18 | **Honesty** | Said "I'd want to measure X first" at least once, and named the experiment | |

**Total: ___ / 54**

## Level Thresholds

These are calibration guides, not guarantees. The distribution matters as much as the total: a 40 with zeros in dimensions 1, 9, and 12 reads as strong senior, not principal.

| Total | Reads as | The tell |
|-------|----------|----------|
| < 20 | Mid / early senior | Design is present; reasoning is not |
| 20-32 | Senior | Correct, complete, few trade-offs named unprompted |
| 33-42 | Staff | Trade-offs and failure modes are native; evolution and cost are thin |
| 43-50 | Principal | Reframing, invariants, evolution, and cost all present unprompted |
| > 50 | Exceptional | You are teaching the interviewer something |

## The Three Dimensions That Move the Level Most

If you can only work on three: **1 (Reframing)**, **9 (Relaxed invariants)**, **12 (Evolution)**. They are the least common in senior answers and the most reliably present in principal answers, because they require having *owned* a system rather than having built one.
