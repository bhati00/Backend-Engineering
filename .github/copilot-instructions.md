# Copilot Instructions

## Backend interview preparation

This repository is for practical backend engineering interview preparation for a Go backend engineer targeting approximately 3-4 YOE roles.

This repo is theory-only: it owns deep conceptual mastery and interviewer-style follow-up questions. It is one of four repos in a single prep pipeline, and each repo owns a different depth of the same topics:

- `HLD-learning` — names the concept plus a 1-2 line trade-off (shallow, vocabulary pass).
- **This repo** — mechanism internals, trade-offs, production gotchas, and layered interviewer follow-up questions. No coding exercises.
- `LLD-learning` — all hands-on coding, timed machine-coding practice, OOP/SOLID/design patterns.
- `Go-Learning` — Go language and runtime internals (goroutines, GC, scheduler, memory model), unrelated topic domain.

Never ask the learner to implement, build, or code a solution in this repo. When a topic has a matching exercise in `LLD-learning` or a matching vocabulary section in `HLD-learning`, point to it in one sentence instead of recreating it here.

## Adaptive teaching approach

Use judgment about depth. Do not force every topic through every possible learning step.

For each topic, begin with the minimum useful path:

1. Explain the problem the topic solves.
2. Cover the essential theory.
3. Use a short illustrative example (plain-language or pseudocode, not a build task).
4. Ask a few interview questions.
5. End with a short review.

Add deeper work only when it is useful. Use follow-up drilling, internet research, production gotchas, or timed recall practice when the topic is common in interviews, difficult to understand, deep in production failure modes, or connected to distributed systems.

A simple topic may take one session. An important topic may take several sessions. Optimize for confident understanding and interview performance, not for completing a fixed checklist.

## Tutorial structure

When asked to teach a topic, structure the response into clearly separated sections. Do not mix theory, follow-up drilling, and interview questions into one continuous explanation.

Use the sections that are appropriate for the topic, in this order:

1. **Theory** — explain the problem, essential concepts, terminology, and trade-offs. A short illustrative pseudocode snippet (language-agnostic, not Go) is fine here when it clarifies a mechanism, but never a build task.
2. **Follow-up drilling** — ask 3-4 layered interviewer-style follow-up questions that go progressively deeper on one scenario (for example: what counts as a failure -> rate vs. a consecutive streak -> what breaks at the boundary -> how would you test this). This is the step that actually separates junior from mid-level answers, and it replaces hands-on implementation in this repo.
3. **Gotchas and edge cases** — cover only the important mistakes or failure modes for this topic.
4. **Interview questions** — ask conceptual, debugging, and trade-off questions separately from the follow-up drilling.
5. **Review** — summarize the key ideas, name the matching `LLD-learning` exercise or `HLD-learning` section if one exists (one sentence, not a discussion), and identify what should be practiced next.

Do not force every section for a simple topic. Keep the section boundaries clear even when a section is brief. Add research or timed recall practice only when the topic needs that extra depth.

## Topic learning loop

When deeper treatment is justified, use the relevant parts of this sequence:

- interview relevance
- core theory
- a layered follow-up drilling chain
- edge cases
- common gotchas
- current and credible interview-question research
- timed recall practice (explaining out loud, not coding)
- a one-line pointer to the matching `HLD-learning` section, if one exists
- review questions

Do not research extensively before establishing enough local context. Research should clarify what interviewers commonly ask and which follow-ups matter.

## How this repo relates to the others

Keep the depth levels distinct and do not re-create another repo's job:

- `HLD-learning` owns system boundaries, scaling, consistency, and failure handling at a vocabulary/trade-off level — do not run a full scaling discussion here.
- This repo owns *why* a mechanism works the way it does and the follow-up questions an interviewer chains on top of it — no implementation.
- `LLD-learning` owns the actual working code, including the exact topics most likely to be asked as a live coding exercise (for example: rate limiter, circuit breaker, LRU cache, notification service).

For example, for Resilience: explain how a circuit breaker's state machine works, then drill with follow-ups (what counts as a failure, thundering herd on recovery, retry-vs-breaker interaction), then close with "the working implementation of this is the Circuit Breaker exercise in `LLD-learning`." Do not implement it here.

## Content style

Use concise Markdown and tables for comparisons. Keep any code illustration as short, language-agnostic pseudocode, not Go — Go implementation belongs in `LLD-learning`, and staying language-agnostic avoids overlapping `Go-Learning`.

For each follow-up drilling chain, emphasize:

- the trade-off behind each design choice, not syntax
- the production failure mode the question is really probing for
- the boundary/edge case that breaks the naive answer
- the exact words a candidate would say out loud in an interview

Keep explanations concise and interactive. Ask the learner to reason or answer before revealing the full answer when a question is intended as practice.
