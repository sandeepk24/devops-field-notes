# Uvicorn and Gunicorn, Explained Twice

Two explanations of the same thing, aimed at two very different audiences.

---


Imagine you run a lemonade stand, except it's not lemonade — it's answers. People walk up, ask you a question, and you give them an answer. That's basically what a website's "backend" does. Someone's phone sends a question ("show me my profile page"), and some code on a server sends back an answer.

Now here's the problem: what if 500 people show up to your lemonade stand at the exact same second?

If it's just you, one kid, running the stand — you can only serve one person at a time. Person 499 is going to be waiting a long, long time.

**This is where Gunicorn comes in.** Gunicorn is like the manager of the lemonade stand. Its whole job is: "I will not run the stand myself. Instead, I will hire a bunch of kids, station them at different tables, and make sure that when a big crowd shows up, there are enough kids working that nobody waits forever." If one kid gets tired and needs a break (crashes), the manager notices and quickly puts in a substitute, without shutting down the whole stand. Gunicorn doesn't actually know how to make lemonade — it just knows how to manage a crew of workers.

**Uvicorn is one of the kids working at the table** — and a very fast one. Some kids can only do one thing at a time: take an order, walk to get lemonade, pour it, hand it over, and *only then* take the next order. Uvicorn is the kid who's figured out a trick: while the lemonade is being poured (which takes a bit of time and doesn't need hands), they immediately turn around and take the next person's order. They juggle a bunch of orders at once instead of standing there waiting. This trick is called **async**, and it's incredibly useful when a lot of the job is *waiting* — waiting for a database to answer, waiting for another website to reply, waiting for a file to load.

So when you hear "we run our app with Gunicorn managing Uvicorn workers," here's the picture in your head: a manager (Gunicorn) hiring a whole team of the fast, juggling kind of worker (Uvicorn), so the lemonade stand can handle a giant crowd, and if one worker needs to step out, business doesn't stop.

---


### The one-sentence version

**Uvicorn is an ASGI server. Gunicorn is a process manager. In production, you very often run them together: Gunicorn supervises a pool of Uvicorn worker processes.** They solve two different problems and people conflate them constantly, so let's pull them apart properly.

### The problem both of these exist to solve

A Python web framework (FastAPI, Django, Flask, Starlette) is just application code — it defines routes, handlers, middleware. It does not know how to:

- listen on a TCP socket
- parse raw HTTP requests off the wire
- speak the HTTP protocol back to the client
- manage multiple OS processes so a crash doesn't kill your whole service
- restart workers gracefully during a deploy

That's the job of the **server**. Your framework and your server talk to each other through a standardized interface — this is where WSGI and ASGI come in.

### WSGI vs ASGI (this is the actual fork in the road)

```
Client → Server (Gunicorn/Uvicorn) → [ interface ] → Your app (FastAPI/Flask/Django)
                                          ↑
                                  WSGI or ASGI
```

- **WSGI** (Web Server Gateway Interface) — the older standard, synchronous only. One request occupies one worker for its entire lifetime. If the app is waiting on a database call, that worker is just... blocked, doing nothing else. Flask and classic Django are WSGI.
- **ASGI** (Asynchronous Server Gateway Interface) — the newer standard, supports both sync and async code, plus WebSockets and long-lived connections (things WSGI was never designed for). FastAPI and Starlette are ASGI-native. Django added ASGI support later.

This distinction is the whole reason Uvicorn exists separately from older servers like Gunicorn's default sync worker. **Gunicorn was born in the WSGI world.** It doesn't natively speak ASGI. Uvicorn was built specifically to be a fast ASGI server, using `uvloop` (a drop-in, much faster replacement for Python's default `asyncio` event loop, built on libuv — the same event loop library that powers Node.js) and `httptools` for the actual HTTP parsing.

### So what does each one actually do?

**Uvicorn — the ASGI server**
- Owns the event loop
- Parses HTTP/1.1 (and optionally HTTP/2 via `h11`/`h2`) off the socket
- Translates raw requests into ASGI-spec calls into your app
- Handles a single process's worth of concurrency very well — but by default, **Uvicorn alone runs as a single process**

That last point is the catch. A single Uvicorn process is excellent at juggling many concurrent I/O-bound requests within one process, thanks to the event loop. But it's still one process on one CPU core. If your workload has any CPU-bound stretches (JSON serialization of huge payloads, image processing, crypto), or if you just want to use more than one core, one process isn't enough.

Let me explain in more detail of what uvicorn actually does, plainly:

When someone's browser or app sends a request to your FastAPI app, that request has to travel over the internet as a bunch of raw text and bytes. Something has to sit there, actually receive that raw data off the network, figure out "oh, this is an HTTP request, here's the URL they're asking for, here's the data they sent," and then hand it to your FastAPI code in a format it understands. Once your FastAPI code decides what the answer is, something has to take that answer and translate it back into the raw format the network expects, and send it back.
That "something" is Uvicorn. It's the piece sitting between the raw internet and your actual Python code, doing the translation both ways.
The "fast juggler" part:
Uvicorn's specific trick is that it doesn't handle one request at a time and wait around. If a request is doing something that involves waiting — like your code asking a database for data and waiting for the answer — Uvicorn doesn't just sit idle during that wait. It goes and starts working on a different request that came in at the same time, then comes back to the first one once the database responds. It's constantly switching between many in-progress requests instead of doing them strictly one after another. That's why it can handle a lot of simultaneous traffic well, especially when a lot of your app's time is spent waiting on other things (databases, other APIs) rather than doing heavy computation.
The catch, in plain English:
All of that juggling happens inside a single worker — think of it as one person doing the juggling. That one person can juggle many things at once, but it's still just one person. If you want multiple people juggling in parallel (to use more than one CPU core, or to survive one of them getting stuck), you need more than just Uvicorn — which is where Gunicorn comes in to manage multiple of these Uvicorn "jugglers" running side by side.
That's really the whole idea: Uvicorn is the translator-and-juggler that talks to the internet and runs your code efficiently; it's just limited to being one process by itself.




**Gunicorn — the process manager**
- Doesn't handle requests itself — it's a **pre-fork worker model manager**
- Starts a master process, which forks N worker processes
- Each worker is a full, independent copy that can crash, be recycled, or be restarted without touching the others
- Handles graceful reloads (`SIGHUP`), graceful shutdowns (`SIGTERM`), and worker timeouts
- Ships with worker *classes* — by default, a synchronous WSGI worker

Since Gunicorn doesn't speak ASGI on its own, running a FastAPI app requires telling Gunicorn to use Uvicorn's worker class instead of its default. That's the marriage:

```bash
gunicorn myapp:app -k uvicorn.workers.UvicornWorker -w 4 --bind 0.0.0.0:8000
```

Read that as: *"Gunicorn, start 4 worker processes, but instead of your normal sync WSGI worker, each one should actually be a Uvicorn ASGI event loop."* You get Gunicorn's mature process supervision (multi-core, auto-restart, graceful reload) plus Uvicorn's async performance, in each of those processes.

### A diagram of the whole stack

```
                     ┌─────────────────────────────┐
                     │      Gunicorn (master)       │
                     │  - binds the socket          │
                     │  - forks & supervises workers│
                     │  - handles SIGTERM/SIGHUP    │
                     └───────────┬─────────────────┘
                                 │ forks
        ┌────────────────────────┼────────────────────────┐
        ▼                        ▼                        ▼
 ┌─────────────┐         ┌─────────────┐          ┌─────────────┐
 │ Uvicorn      │         │ Uvicorn      │          │ Uvicorn      │
 │ worker proc  │         │ worker proc  │          │ worker proc  │
 │ (event loop, │         │ (event loop, │          │ (event loop, │
 │  uvloop)     │         │  uvloop)     │          │  uvloop)     │
 │      │        │         │      │        │          │      │        │
 │      ▼        │         │      ▼        │          │      ▼        │
 │  FastAPI app │         │  FastAPI app │          │  FastAPI app │
 └─────────────┘         └─────────────┘          └─────────────┘
```

Each worker process handles many concurrent requests via its own event loop; Gunicorn handles the "many processes across many cores, and don't let a dead one take the service down" problem on top.

### Do you always need both?

No, and this trips people up. Three common setups:

| Setup | When it makes sense |
|---|---|
| **Uvicorn alone** (`uvicorn myapp:app --workers 4`) | Uvicorn actually grew its own `--workers` flag, which spins up multiple worker processes internally without Gunicorn at all. Perfectly fine for many deployments, especially containerized ones where an external orchestrator (ECS, Kubernetes) already handles process-level restarts and scaling. |
| **Gunicorn + Uvicorn workers** | You want Gunicorn's more battle-tested process management: graceful worker recycling, `max_requests` (restart a worker after N requests to fight memory leaks), pre-fork behavior, and mature signal handling. Common in traditional VM/bare-metal deployments or when a team already trusts Gunicorn operationally. |
| **Gunicorn alone (sync workers)** | Only makes sense for WSGI apps (Flask, classic Django) with no async code path. Not applicable to a FastAPI app. |

Given your ECS-heavy world: ECS/Fargate already gives you task-level supervision (health checks, auto-replacement of unhealthy tasks). That changes the calculus — you're less dependent on Gunicorn's process babysitting because the orchestrator is doing some of that job at a different layer. A lot of teams running FastAPI on ECS just use `uvicorn --workers N` directly in the container and let ECS handle task health. Others still prefer Gunicorn in front for the finer-grained in-container worker recycling (particularly `--max-requests` to guard against memory creep) independent of a full task restart, which is slower and pulls in the load balancer's deregistration/registration cycle.

### Sizing workers (the question everyone gets wrong)

The classic Gunicorn guidance is `(2 x $NUM_CORES) + 1` — but that formula is really tuned for CPU-bound, sync WSGI workloads. For async ASGI workers, the math changes because a single Uvicorn worker can hold open hundreds of concurrent I/O-bound connections without needing more processes. Overprovisioning workers here mostly just wastes memory (each worker is a full Python process with its own copy of your app loaded).

A more honest starting point for async workloads:
- Start with workers = number of CPU cores available to the container
- Load test and watch CPU utilization and p99 latency, not just request count
- If you're CPU-bound (heavy serialization, sync blocking calls sneaking into your async routes), scale workers up
- If you're I/O-bound (mostly waiting on downstream calls), your bottleneck is more likely downstream connection pool limits (DB connections, HTTP client pools) than worker count

### A production-shaped Gunicorn config

```python
# gunicorn.conf.py
import multiprocessing

bind = "0.0.0.0:8000"
workers = multiprocessing.cpu_count()
worker_class = "uvicorn.workers.UvicornWorker"

# Recycle workers periodically to fight memory leaks / stale connections
max_requests = 1000
max_requests_jitter = 50   # jitter avoids all workers restarting simultaneously

# Don't let a stuck worker hang forever
timeout = 30
graceful_timeout = 30

# Useful during rolling deploys behind an ALB
preload_app = True
```

```bash
gunicorn myapp:app -c gunicorn.conf.py
```

A couple of details worth internalizing:

- **`preload_app`** loads your application code once in the master before forking, so workers share memory pages (copy-on-write) instead of each re-importing everything — faster startup, lower memory. The tradeoff: any global state created at import time gets shared/forked, not freshly created per worker, which occasionally bites people using connection pools initialized at module import.
- **`max_requests_jitter`** exists specifically so you don't get a thundering-herd restart of every worker at once, which would briefly tank your capacity.
- **`graceful_timeout`** matters a lot behind an ALB during ECS deploys — it controls how long Gunicorn waits for in-flight requests to finish before force-killing a worker during a `SIGTERM`. Set it shorter than your ALB's deregistration delay isn't quite right either; you want it to comfortably cover your longest expected request duration.

### Common junior-engineer trap

Running blocking, synchronous code (a `requests.get()` call, a synchronous DB driver, `time.sleep`) inside an `async def` route handler. Because Uvicorn's whole performance model depends on the event loop never blocking, one blocking call inside an async handler stalls the *entire* event loop for that worker — every other "concurrent" request queued behind it stalls too. FastAPI's escape hatch is to define the route as a regular `def` (not `async def`) when it does blocking work; FastAPI then runs it in a thread pool instead of directly on the event loop. Knowing when to reach for that is one of the more common debugging rabbit holes people run into once they move from "it works locally" to "it's falling over under load."

### tl;dr for the on-call doc

- **Uvicorn** = the thing that actually speaks HTTP and runs your ASGI app's event loop
- **Gunicorn** = the thing that runs multiple copies of a worker process and keeps them alive
- Together: `gunicorn -k uvicorn.workers.UvicornWorker`
- On ECS/Fargate, `uvicorn --workers N` alone is a legitimate, simpler choice — let the orchestrator handle task-level health
- Never block the event loop in an `async def` handler
