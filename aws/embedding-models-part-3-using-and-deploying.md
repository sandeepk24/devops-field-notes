# Embedding Models, Part 3 — Using and Deploying an Embedding Model

> Part 2 covered how to choose an embedding model and test whether it actually works well on your own content. This part is about the practical side: calling one of these models from code, running it as part of a real pipeline, and understanding where it can live — as a managed service you call over the internet, or as something you run and maintain yourself. Part 4 covers keeping it healthy once it's handling real traffic.

---

## 1. The simplest possible version: one function call

Underneath everything, using an embedding model comes down to one repeated action: you send it some text, and it sends back a list of numbers. Here's what that looks like using Amazon Bedrock, which is the easiest way to get started if you're already working in AWS — there's no server to set up, you just call an API.

```python
import boto3
import json

bedrock = boto3.client("bedrock-runtime", region_name="us-east-1")

def get_embedding(text):
    response = bedrock.invoke_model(
        modelId="amazon.titan-embed-text-v2:0",
        body=json.dumps({
            "inputText": text
        }),
    )
    result = json.loads(response["body"].read())
    return result["embedding"]

vector = get_embedding("The deployment pipeline failed at the build step")
print(len(vector))       # how many numbers came back
print(vector[:5])        # the first few, just to look at them
```

That's genuinely most of what you need to know to start experimenting. Run this on a handful of different sentences, and you'll get a feel for what's happening: similar sentences will produce similar-looking lists of numbers, and unrelated sentences will produce very different ones.

## 2. Comparing two pieces of text

On its own, one embedding isn't very useful — the numbers only mean something in comparison to other embeddings. The most common way to compare two embeddings is called **cosine similarity**: a way of measuring how similar in direction two lists of numbers are, which turns out to correspond well to how similar in meaning the original texts are.

You don't need to understand the math behind it to use it. Here's what it looks like in practice:

```python
import numpy as np

def compare(text_a, text_b):
    vec_a = np.array(get_embedding(text_a))
    vec_b = np.array(get_embedding(text_b))

    similarity = np.dot(vec_a, vec_b) / (np.linalg.norm(vec_a) * np.linalg.norm(vec_b))
    return similarity

print(compare(
    "The deployment pipeline failed at the build step",
    "The build stage of the CI pipeline broke"
))
# expect something close to 1 — these mean nearly the same thing

print(compare(
    "The deployment pipeline failed at the build step",
    "What time does the cafeteria close"
))
# expect something close to 0 — these are unrelated
```

The result is always a number between -1 and 1. Close to 1 means very similar meaning. Close to 0 means unrelated. Negative numbers, which are rare in practice for normal text, mean roughly opposite meaning. Trying this out on a few pairs of sentences — some obviously related, some obviously not — is a good way to build intuition before using this in a real system.

## 3. Handling a question that has a slightly different shape than a document

As mentioned in Part 2, some embedding models want to be told whether a piece of text is a short question or a longer document being searched, because treating them differently produces better results. Cohere's embedding models on Bedrock support this directly:

```python
def embed_as_query(text):
    response = bedrock.invoke_model(
        modelId="cohere.embed-english-v3",
        body=json.dumps({
            "texts": [text],
            "input_type": "search_query"
        }),
    )
    return json.loads(response["body"].read())["embeddings"][0]

def embed_as_document(text):
    response = bedrock.invoke_model(
        modelId="cohere.embed-english-v3",
        body=json.dumps({
            "texts": [text],
            "input_type": "search_document"
        }),
    )
    return json.loads(response["body"].read())["embeddings"][0]
```

The important habit to build here: when you're embedding something a person typed as a search question, use `embed_as_query`. When you're embedding something from your actual documents that will be searched *against*, use `embed_as_document`. Mixing these up — treating a document like a query, or the reverse — is an easy mistake to make and it quietly makes your search results worse, without giving you any error message to tell you something's wrong. It just underperforms, which is often harder to notice than an outright failure.

## 4. Processing many documents at once

In practice, you're rarely embedding just one sentence. You're usually processing an entire folder of documents, and each document might be broken into several smaller pieces first (this splitting step is usually called chunking, and it's covered in the Knowledge Bases guide). Here's a straightforward way to work through a batch:

```python
def embed_documents(chunks):
    """chunks: a list of text pieces, e.g. paragraphs pulled from your docs"""
    embeddings = []
    for chunk in chunks:
        vector = get_embedding(chunk)
        embeddings.append({"text": chunk, "vector": vector})
    return embeddings

my_chunks = [
    "To restart the payments service, run systemctl restart payments.",
    "Deployment logs are available in the #deploys channel.",
    "The staging environment refreshes automatically every night at 2am.",
]

results = embed_documents(my_chunks)
for r in results:
    print(r["text"][:40], "→", len(r["vector"]), "numbers")
```

A couple of things worth knowing before you run this on a large number of documents:

**It costs a small amount of money per piece of text**, since each call to the embedding model is billed. This usually isn't expensive, but it adds up with a genuinely large document set, so it's worth checking current Bedrock pricing before running this against, say, ten thousand documents.

**Sending requests one at a time, like the example above, is simple but slow.** Some embedding models let you send several pieces of text in a single request instead of one at a time (Cohere's model does this — you can pass a list under `"texts"`), which is noticeably faster for large batches. It's worth checking whether the model you're using supports this before writing a loop that makes thousands of individual calls.

**If you run this same process again later on documents that haven't changed, you're paying to redo work you already did.** A simple fix is to keep track of which pieces of text you've already embedded — for example, by saving a short fingerprint (a hash) of each piece of text alongside its embedding — and skip anything you've already processed. This one habit saves a meaningful amount of money once a pipeline is running regularly rather than as a one-time experiment.

## 5. Where to actually run this

There are three realistic options, and for the overwhelming majority of situations, the first one is the right choice.

**Calling a managed model through an API (like the Bedrock examples above).** You don't manage any servers. You send text, you get numbers back, and AWS handles all the infrastructure behind it. This is the least amount of setup, the least amount of ongoing maintenance, and it's usually the right starting point — and often the right ending point too. Most teams never need anything more than this.

**Running your own model on a managed training and hosting platform, like Amazon SageMaker.** This becomes relevant if you've gone through the process in Part 2, section 4 and trained your own version of a model. SageMaker gives you a place to host that custom model without managing the underlying servers yourself, though you're still responsible for the model itself and for keeping it updated.

**Running the model entirely yourself, on your own servers.** This is the most work by a significant margin — you're responsible for the compute, the scaling, keeping the software patched, everything. This is really only worth doing if you have a specific reason a managed option won't work: a strict requirement that nothing leave your own network, for instance, or an unusual technical need that managed services don't support. If you're not sure whether you need this option, you probably don't.

A reasonable way to think about this: start with the managed API. Only move down this list if you hit a wall the previous option genuinely can't get past — not because a more complicated setup sounds more impressive.

---

**Continue to [Part 4 — Keeping Embeddings Working in Production](./embedding-models-part-4-running-in-production.md):** what happens if you ever need to switch models, how to notice quietly declining search quality before someone complains about it, and a short checklist for anyone setting this up for the first time.
