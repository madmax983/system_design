# Interviewer Guide

How to run a principal-level mock from any [prompt card](prompts/) for a partner. You do not need to have studied the worked example; the card gives you the probes. Your job is to *steer*, *probe*, and *notice omissions*, not to lecture.

## Before the Session

- Read the prompt card's prompt, "what this trains," and the follow-up questions (you may read the hidden answers; the candidate may not).
- Skim the worked example's section 12 (Level Signals) so you know what a senior, staff, and principal answer sounds like for this prompt.
- Have the [scorecard](scorecard.md) open.

## Running the Session (45 minutes)

### Opening (1 minute)
Read the prompt verbatim. Then say: "Take it from here. I'll interrupt if I need to." Do not add hints. Part of what you are assessing is whether they ask.

### During requirements (minutes 1-7)
Answer clarifying questions truthfully but minimally. If they ask a number you do not have, say "assume something reasonable and tell me what you assumed."

**Notice:** did they name a dominant constraint? Did they challenge any part of the prompt? Did they state non-goals? If they draw a box before minute 5, note it.

### During estimation (minutes 7-10)
Let them compute. Then ask: **"Which of those numbers is the design most sensitive to?"** A principal answer has one immediately.

### During high-level design (minutes 10-17)
Let them draw. Do not interrupt for small errors.

**Notice:** is every component justified by a requirement? Is there a critical-path walk-through?

### During deep dives (minutes 17-35)
This is where you steer. Use the card's probes. Pick the *hardest* sub-problem, especially if they are heading toward the most familiar one. Say: "I'd like to go deep on X."

Push with "and then what happens?" until they reach a mechanism or say "I don't know." Both are fine outcomes. Hand-waving is not.

**Notice:** do they state invariants? Do they reason about retries and idempotency without being asked? Do they name what they would measure?

### During failure and operations (minutes 35-42)
Ask, in this order, skipping any they already covered unprompted:
1. "What's the worst single failure, and what's its blast radius?"
2. "What's a failure that hits everything at once?" (correlated)
3. "In what order does this degrade under overload, and who decides?"
4. "How is this deployed, and how long does rollback take?"

### Wrap-up (minutes 42-45)
Ask: **"You've inherited a v1 of this that's much simpler. How do you get to what you drew without downtime?"** Then: **"What did you consider and reject, and why?"**

## Probing Technique

| If they say | You say |
|-------------|---------|
| "We'd handle that" | "How, specifically? What's the mechanism?" |
| "Best practice is..." | "Why is it the right call *here*?" |
| "It should be fine" | "What would make it not fine?" |
| "We could do X or Y" | "Which one? Decide." |
| "We'd add a cache / queue / replica" | "What does that cost, and what does it break?" |
| Something you know is wrong | Do not correct it. Ask "walk me through what happens when [the case that breaks it]." |
| Something excellent | Do not react. Note it. Keep the pressure even. |

## Scoring and Debrief (15 minutes after)

1. Both of you fill in the scorecard independently. Compare. Disagreements are the most useful part.
2. Read the worked example's section 12 together. Which level did the answer sound like, and why?
3. The interviewer names **the three questions the candidate did not ask** that the worked example asks. This is the single most valuable output of the session.
4. The candidate fills in the [reflection template](reflection-template.md).

## Interviewer Anti-Patterns

- **Helping.** A hint you give is a signal you cannot observe.
- **Correcting mid-flow.** Let a wrong path run until the probe exposes it; that is how real interviews go.
- **Going where they are strong.** Steer to the hard sub-problem, not the one they are enjoying.
- **Rushing failure modes.** They are the level discriminator; protect the time.
- **Being nice in the scorecard.** A generous score hides the thing they need to fix.
