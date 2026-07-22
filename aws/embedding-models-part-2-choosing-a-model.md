# Embedding Models, Part 2 — Picking a Model and Checking It Actually Works

> Part 1 covered what an embedding is, how the vector space gets built during training, and the different families of embedding models. This part is about a much more practical question: with several models to choose from, how do you pick one, and how do you know afterward whether you picked well? Part 3 covers calling these models from code and getting them running. Part 4 covers keeping everything working once it's live.

---

## 1. Why "just pick the popular one" isn't enough

Say you're setting up search over your team's documentation, and you need to choose an embedding model. There's an obvious shortcut: search online for "best embedding model," find a leaderboard, pick whatever's ranked first.

That shortcut gets you most of the way there, but not all the way. Here's why.

The leaderboard everyone uses is called **MTEB**, short for Massive Text Embedding Benchmark. It's a public scoreboard where embedding models are tested on dozens of tasks — searching news articles, comparing sentence pairs, sorting documents into categories — mostly using general-purpose text: Wikipedia, news, common Q&A datasets.

Your documentation is not Wikipedia. It's full of your own words: internal tool names, error messages, abbreviations only your team uses, config file syntax. A model can be excellent at understanding news articles and still be mediocre at understanding a Kubernetes error log, simply because it never saw much text like that during training.

So the honest way to think about MTEB is: it's a good way to build a shortlist of three or four models worth trying. It's not a good way to make the final decision. The final decision needs to be based on how well a model does on your own material.

## 2. Building a small test you can trust

Here's a simple, low-effort way to test any embedding model against your own content, before you commit to it.

**Step one: collect real questions.** Write down twenty to fifty questions that someone would realistically ask about your documentation. The best source for these is real life — pull them from support tickets, from questions people asked in chat, from things you've had to look up yourself. Don't make them too neat; real questions are often short, vague, or phrased casually. "Why is the deploy stuck" is a more realistic test question than "What are the possible causes of a deployment pipeline halting mid-execution."

**Step two: for each question, note the correct answer.** Not the full answer text — just which document or which section of a document actually contains the answer. This is your answer key.

**Step three: for a candidate embedding model, run every question through it and see what comes back.** For each question, check: did the correct document show up in the results at all? And if it did, how far down the list was it?

That second check matters more than people expect. A model that returns the right answer as the very first result is far more useful in practice than one that eventually includes the right answer buried at position eight, even though both technically "found" it. If your system only shows someone the top three or four results, a right answer sitting at position eight might as well not exist.

**A simple way to score this in code:**

```python
def score_results(retrieved_lists, correct_answers, top_n=5):
    """
    retrieved_lists: for each question, the list of document IDs returned,
                      ranked from most to least relevant
    correct_answers: for each question, the ID of the document that
                      actually has the answer
    top_n: how many results you'd realistically show someone
    """
    found_in_top_n = 0
    for retrieved, correct in zip(retrieved_lists, correct_answers):
        if correct in retrieved[:top_n]:
            found_in_top_n += 1

    return found_in_top_n / len(correct_answers)

accuracy = score_results(my_retrieved_results, my_answer_key, top_n=5)
print(f"Correct answer appeared in the top 5 results: {accuracy:.0%} of the time")
```

Run this same test, with the same questions and the same top_n, against every model you're considering. Whichever model scores highest on *your* questions is the one to use — even if it's not the one sitting at the top of the public leaderboard. This has genuinely happened to teams more than once: the "second-best" model on paper turns out to be the best fit once tested on real, specific content.

## 3. A detail that trips people up: short questions vs. long answers

Here's something worth understanding before you evaluate anything, because it explains a lot of confusing results.

When someone searches your documentation, they usually type something short: "how do I roll back a deployment." The actual answer, sitting in your documentation, might be a three-paragraph section with several sentences of setup and context before it gets to the actual steps.

Some embedding models were trained mostly on pairs of text that are roughly the same length and shape — two similar sentences, two similar paragraphs. Feed one of these models a six-word question and a four-hundred-word answer, and it can genuinely struggle to recognize that they're related, even when the answer is perfect, purely because the two pieces of text look so different in size and shape.

This is a real, well-known limitation, and the good news is some models are specifically built to handle it. Cohere's embedding models, for example, let you tell the model directly: "this text is a search question" or "this text is a document being searched," and the model treats the two differently as a result. If your use case is mostly short questions against longer documents — which describes almost every internal search or help-desk-style tool — checking whether a model supports this distinction is worth doing before you commit to one.

## 4. When it's worth teaching a model your own vocabulary

Most of the time, an existing embedding model — used as-is, without any extra training — will do a solid job. But every so often, teams run into text that's genuinely hard for a general-purpose model to understand: a codebase full of unusual internal naming conventions, a support system dense with product-specific acronyms, error taxonomies unique to one company.

If you've tried a few different models on your own test questions (section 2) and none of them score well — not because your test questions are unclear, but because the model consistently misunderstands your specific vocabulary — that's when it can make sense to adjust a model rather than just pick between existing ones.

This process is usually called **fine-tuning**. In plain terms: you take a model that's already been trained on general text, and you continue training it a little further, this time only on examples from your own world — pairs of real questions and the documents that answer them. The model doesn't start from scratch; it just adjusts what it already knows to better fit your specific material.

This is genuinely more work than picking an existing model, and it comes with an ongoing cost: once you've trained your own version, you're the one responsible for hosting it, keeping it updated, and re-training it if your material changes significantly. For that reason, it's worth treating as a last resort rather than a starting point — only reach for it once testing (section 2) has clearly shown that existing models fall short, not as a first move because it sounds more sophisticated.

## 5. Making sense of the options without getting overwhelmed

If you're just getting oriented, here's a simple way to narrow things down instead of trying to evaluate everything at once:

1. **Start with one well-known, general-purpose model.** On Amazon Bedrock, Amazon's own Titan Text Embeddings model is a reasonable default — it's built to handle a wide range of everyday text reasonably well, and it's the least amount of setup to get started with.
2. **Build the small test described in section 2.** Even twenty questions is enough to tell you something useful.
3. **If the results are good enough, stop there.** There's no prize for using something more elaborate than you need.
4. **If results are consistently weak in a specific, identifiable way** — say, it keeps missing questions about error codes and specific version numbers — try a model built to handle short questions against long documents differently, as described in section 3.
5. **Only consider training your own version (section 4) if none of the above closes the gap**, and only after you've confirmed the problem is genuinely about vocabulary, not about something else entirely — like your documents being outdated, poorly organized, or split into chunks in a way that cuts sentences in half.

That last point is worth sitting with for a second: a lot of what looks like "the embedding model isn't working" is actually a problem somewhere else in the pipeline — the documents themselves, or how they were split into smaller pieces before being processed. Before assuming the model is the issue, it's worth ruling out the simpler explanations first.

---

**Continue to [Part 3 — Using and Deploying an Embedding Model](./embedding-models-part-3-using-and-deploying.md):** how to actually call one of these models from your own code, where to run it, and what a batch of documents going through the process looks like end to end.
