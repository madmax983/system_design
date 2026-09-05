# Practice: Deliberate, Self-Directed Preparation

This section turns the [worked examples](../examples/) into a practice program. It is built on how adults actually learn (andragogy): you learn by *attempting* a problem, comparing your attempt against a model, reflecting on the gap, and re-attempting after a delay. Reading a solution feels like learning and is not.

## The Principles, and What They Mean Here

| Principle | What it means in practice |
|-----------|--------------------------|
| **Adults learn by doing, not by reading** | Every prompt card asks you to produce a design *before* you open the worked example. Timeboxed, written down, no peeking. |
| **Adults bring experience** | The reflection template asks you to connect each design to a system you have actually built or operated. Your war stories are your principal-level material. |
| **Adults need to know why** | Each prompt card names what the exercise is training (idempotency reasoning, migration mechanics, blast-radius thinking) so you practice with intent. |
| **Adults are problem-centered** | The prompts are problems, not topics. You do not "study caching"; you design a system where the cache is the hard part. |
| **Adults are self-directed** | You pick the prompt, the timebox, and the focus dimension. The scorecard tells you where you are; you decide what to work on. |
| **Feedback must be immediate and specific** | The scorecard is filled in right after the attempt, dimension by dimension, against concrete criteria. |
| **Retention needs spacing** | The schedule below re-attempts each prompt at increasing intervals. The second attempt is where the learning happens. |

## The Session Protocol

One session is 60-75 minutes. Do not skip steps 4 and 5; they are the ones that produce improvement.

```
1. PICK      (2 min)   Choose a prompt card. Read only the prompt and the "what this trains" line.
2. ATTEMPT   (45 min)  Timebox strictly. Write or speak your design following the interview
                       pacing in the card. Produce: requirements, dominant constraint, estimate,
                       architecture, 2-3 deep dives, failure modes, evolution, trade-offs.
                       Record yourself if you can; you will be surprised.
3. UNCOVER   (5 min)   Open the card's hints and follow-up questions. Answer the follow-ups
                       out loud before revealing the model answers.
4. COMPARE   (10 min)  Read the worked example. Fill in the scorecard, one line per dimension.
                       Be harsh. "I would have said that if asked" scores zero.
5. REFLECT   (10 min)  Fill in the reflection template. The key question: "what did I not
                       think to ask?" The gap between your questions and the model's questions
                       is the level gap.
6. LOG       (1 min)   One line in practice-log.md: date, prompt, score, the one thing to fix.
```

## The Spacing Schedule

Re-attempt each prompt on this schedule. On the re-attempt, do not review the worked example first. The goal is retrieval, not recognition.

| Attempt | When | Focus |
|---------|------|-------|
| 1st | Day 0 | Full attempt, cold |
| 2nd | Day 3 | Full attempt; compare against your own 1st-attempt notes before the worked example |
| 3rd | Day 10 | Attempt only the deep dives and failure modes, from memory, in 20 minutes |
| 4th | Day 30 | Full attempt, cold, with a different interviewer follow-up emphasis |

Seven worked prompts on this schedule is roughly 28 sessions over five weeks, at about five sessions per week. Interleave with the [prompt bank](prompt-bank.md) so that you are not memorizing solutions but practicing the method.

## Choosing Where to Focus

After three or four sessions, your scorecards will show a pattern. Common ones:

| Pattern in your scores | What to do |
|-----------------------|-----------|
| Architecture strong, requirements weak | Spend the first 7 minutes of every attempt *only* on framing. Do not draw a box until you have named the dominant constraint and one thing the prompt got wrong. |
| Deep dives strong, failure modes weak | For the next three sessions, do the failure-mode table *first*, before the architecture. It forces the design to be shaped by failure. |
| Everything strong, evolution weak | Practice the "you inherited v1" variant of each prompt: given the worked example's v1, design the migration to v3. |
| Strong in writing, weak out loud | Record every attempt. Listen back at 1.5x. Note every place you said "um, we could just" and replace it with a decision. |
| Strong on familiar domains, weak on unfamiliar | Prompt bank only, for a week. The method should transfer; if it does not, the method is not yet internalized. |

## Practicing With a Partner

The [interviewer guide](interviewer-guide.md) lets a peer run a mock from any prompt card without having studied the solution. Take turns; the interviewer role is itself excellent practice because it trains you to notice what a strong answer *omits*.

## Files

| File | Purpose |
|------|---------|
| [prompts/](prompts/) | One card per worked example: the prompt, what it trains, pacing, hidden hints, interviewer follow-ups with hidden answers, scoring checklist |
| [prompt-bank.md](prompt-bank.md) | Additional principal-level prompts without full solutions; hints hidden |
| [scorecard.md](scorecard.md) | The self-assessment rubric with level thresholds |
| [reflection-template.md](reflection-template.md) | Post-session journal |
| [interviewer-guide.md](interviewer-guide.md) | How a partner runs a mock from a card |
| [practice-log.md](practice-log.md) | Your running log; commit it |
