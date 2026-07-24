# Embedding Models, Part 4 — Keeping Embeddings Working in Production

> Part 3 covered how to call an embedding model from code and where to run it. This part covers something that catches a lot of people off guard: what happens after everything is set up and working. Embedding-based search doesn't usually fail loudly. It quietly gets a little worse over time, or breaks in ways that don't produce an error message. This part is about noticing that before someone else does.

---

## 1. The rule that's easy to break by accident: never mix embeddings from different models

Here's something that sounds obvious once it's stated clearly, but is genuinely easy to get wrong in practice.

Two lists of numbers only mean something when they came from the *same* embedding model, using the *same* settings. If you embed half your documents with one model, and the other half with a different model — even a very similar one — comparing those two sets of numbers to each other produces meaningless results. It won't crash. It won't warn you. It'll just quietly return bad matches, and because nothing looks broken on the surface, this can go unnoticed for a long time.

This matters in a few very ordinary situations:

- You started a project with one embedding model, and later decided to switch to a better one. If you switch just for new documents going forward, without redoing the old ones, you now have two incompatible sets of numbers sitting in the same search index.
- A model offers a setting for how many numbers to return (for example, Amazon's Titan model lets you choose 256, 512, or 1024). If some documents were embedded with one setting and others with a different one, they're not comparable either, even though they came from the same model.
- Someone updates a script, changes which model it calls, and doesn't realize that everything embedded before that change is now out of step with everything embedded after it.

The practical takeaway: if you ever change which embedding model you're using, or change a setting like the output size, you need to redo *every* document that was embedded with the old settings — not just the new ones going forward. There's no partial or gradual way to do this migration. It's genuinely an all-or-nothing switch.

A good way to protect yourself from this: keep a clear, written record — in your setup scripts, in a config file, somewhere durable — of exactly which model and which settings were used to build a given search index. "Which embedding model did we use for this" turns out to be a surprisingly common question during an incident, and not having a quick answer wastes real time while people are trying to fix something under pressure.

## 2. A safe way to switch models

Because switching models means starting over, it's worth having a plan that doesn't involve any downtime or risk to what's already working.

The simplest safe approach: build the new version entirely separately, test it thoroughly, and only then switch over.

1. Keep your current, working search running exactly as it is.
2. Build a brand new copy of your search index using the new model, from scratch, alongside the old one — don't touch the old one at all yet.
3. Run the test questions from Part 2 against both versions side by side, and compare the results honestly.
4. Only once you're confident the new version is actually better does traffic get pointed at it.
5. Keep the old version around for a little while after switching, in case something unexpected shows up once real usage hits the new version — it's much easier to switch back if the old one is still sitting there ready to go.

This is a bit more work than editing something in place, but it means you're never in a position where search is broken for everyone while you're in the middle of a risky change.

## 3. Noticing when things are quietly getting worse

Nothing about embedding-based search sends you an alert when it starts underperforming. Search results don't disappear — they're just less relevant than they used to be, and unless someone is actively comparing today's results to how things used to work, that kind of decline is easy to miss for weeks.

A few practical habits help catch this early rather than finding out from a frustrated coworker:

**Re-run your test questions regularly, not just once at the start.** The set of twenty to fifty real questions from Part 2 isn't a one-time exercise — it's worth keeping around and running again periodically, especially after any change to your documents, your model, or how documents get split into pieces. If the score on the same test questions drops over time, that's a real signal something changed, even if you can't immediately tell what.

**Pay attention when someone can't find something that should exist.** If a document clearly contains the answer to a question, but search isn't surfacing it, that's worth investigating rather than shrugging off as one-off bad luck. Sometimes it points to a real gap: the document might be poorly organized, split into chunks awkwardly, or written in a way that doesn't closely match how people naturally phrase the question.

**Notice when new kinds of content start showing up that didn't exist when you first set things up.** If your team launches a new product, adopts new tooling, or starts using new terminology, your existing setup was tested against a world that no longer fully matches what's actually being searched. This isn't a sign anything was done wrong originally — it's just a normal reason to periodically re-check that things still work well.

## 4. Common things that look like a model problem but usually aren't

When search results are disappointing, it's tempting to assume the embedding model itself is at fault and go looking for a better one. Often, though, the real issue is somewhere else, and swapping models won't fix it. Worth ruling these out first:

**The documents themselves are outdated or contradictory.** If two documents give different, conflicting answers to the same question, no embedding model can figure out which one is correct — that's not something embeddings are meant to solve.

**Documents are split into pieces in a way that cuts off important context.** If a piece of text starts mid-sentence, or is missing the heading that would have told a reader (or a search system) what it's actually about, even a very good embedding model has less to work with.

**The question and the answer use completely different words for the same thing**, and the embedding model genuinely doesn't have enough signal to bridge that gap — this does happen, especially with very unusual internal terminology, and it's the one case in this list where the model actually is a meaningful part of the problem, worth revisiting the choices from Part 2.

**Too many results are being returned, burying the right one under less relevant ones**, rather than too few good candidates existing in the first place. Sometimes the fix isn't a better model — it's simply asking for fewer, more tightly filtered results.

Working through this list before assuming "we need a different model" saves a lot of wasted effort chasing the wrong fix.

## 5. A short checklist for setting this up the first time

Pulling the last three parts together into something concrete:

- [ ] Picked a general-purpose embedding model to start with, rather than trying to evaluate everything up front
- [ ] Wrote down twenty or more real, realistic questions along with which document actually answers each one
- [ ] Ran those questions against the chosen model and checked whether the right answer showed up near the top, not just somewhere in the results
- [ ] Checked whether the model being used treats short questions differently from longer documents, if that distinction is available
- [ ] Wrote down, somewhere durable, exactly which model and which settings are being used
- [ ] Have a plan for how documents get re-embedded when they change, so search doesn't quietly go stale
- [ ] Know that switching models later means rebuilding the whole index, not a partial update — and have a rough plan for how that would work if it ever comes up
- [ ] Have a way to periodically re-run the test questions and notice if the score starts slipping

None of this needs to be built perfectly on day one. But having even a rough version of each of these in place from the start saves a lot of confusion later, when the person debugging a search problem might not be the person who originally set it up.

---

*A good way to build real confidence with all of this: take five documents you actually use at work, write ten honest questions about them, and walk through Parts 2 through 4 end to end on that small example before doing it for real. Small and real beats large and hypothetical every time.*
