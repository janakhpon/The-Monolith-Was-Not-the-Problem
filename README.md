# The Monolith Was Not the Problem

_A small team's journey from a simple Next.js app to a small fleet of background workers — and why we barely touched the original code._

_Corrected 3 October 2026: the claims on execution limits and retry safety were narrowed, and a caution was added. The note is at the end._

![Article cover - A Next.js monolith hitting a timeout, and the slow work moved to a message queue and background workers](./assets/monolith_to_workers.avif)

---

## Start Here: The Architecture Argument You Probably Heard

The common framing goes something like this:

> Monoliths are simple to start, but hard to scale. Microservices help with scale and team autonomy, but introduce operational complexity. The switch is not because monoliths are bad — it's because they don't fit large, fast-moving teams. But microservices are not a free win either. They just shift the pain somewhere else.

That framing is fair. But it misses something for small teams: the pain of microservices hits from day one. The pain of a monolith usually shows up much later, if it shows up at all.

When I started building this product, we were a small team with one clear goal — get something real in front of real users as quickly as possible. Not a prototype. A working product.

The architecture had to be simple enough to ship fast. That was the actual constraint.

---

## What We Built

The product was a B2B operations platform — dashboards, data integrations, document workflows, and coordination with external vendors.

The stack was:

- **Next.js** — frontend and backend in one framework
- **Vercel** — deployment, no infrastructure to manage
- **PostgreSQL** — single database
- **API routes** — for all backend logic

Everything in one repository. One codebase, one deployment, one place to look when something breaks.

For a long time — longer than I expected — this was completely fine. No containers. No orchestration. Just `git push` and the new version was live.

Looking back, starting this way was the right call. Not a clever architectural decision. Just an honest one about what a small team actually needs early on.

---

## When the Problems Started

The trouble crept in gradually as the product grew.

We integrated with a large external system that pushed data into ours via webhooks and periodic bulk syncs. A single bulk sync could contain thousands of records, each requiring multiple database writes. A full run could take several minutes.

We also added features that behaved differently from normal web requests — PDF document generation, image compression, scheduled data reconciliation.

Typical user requests looked like this:

```
GET /dashboard   → ~80ms
POST /settings   → ~200ms
```

Background jobs looked like this:

```
sync external records  → several minutes
fill and flatten docs  → subprocess calls, file I/O
compress uploaded file → download, process, re-upload
```

We were treating both the same way. That was the mistake.

---

## The Real Limit: Vercel Serverless Functions

This matters because we were running on Vercel, and the platform shapes what you can and cannot do.

Vercel's serverless functions are designed for fast, stateless requests. But they have hard limits for background work:

- **Bounded execution duration.** A function must complete within its configured
  duration. The limit depends on the plan, runtime and configuration. See
  [Vercel's function limits](https://vercel.com/docs/functions/limitations), checked
  3 October 2026. The five-minute limit in this story describes its historical setup.
- **Functions freeze after responding.** Once you send a response, the runtime is not guaranteed to stay alive. Any work scheduled after the response may not complete.
- **No persistent background processes.** There is no way to run a long-lived worker inside Vercel.

`waitUntil()` can extend work beyond the response but not beyond the function's
maximum duration ([Vercel API reference](https://vercel.com/docs/functions/functions-api-reference/vercel-functions-package#waituntil),
checked 3 October 2026). A reconciliation job needs an execution model suited to its
runtime and recovery requirements. In this story, the choice was a separate worker;
it is not the only available approach for every multi-minute job.

---

## What Actually Broke

The external system sent us events via webhooks. Our original handler received the payload, ran the full sync, and returned a response when done.

For small payloads, fine. For large reconciliation jobs, it ran past the 5-minute timeout. Vercel killed the function. We never returned a success response. So the external system retried. Which timed out again.

We had accidentally built a retry loop — the same job being sent over and over, never completing. A classic "works fine until it doesn't" situation.

The second problem was database connections. Serverless functions scale by spawning more instances. Each instance opened its own connection. Under load, we hit the connection limit and normal user requests — dashboards, settings — started failing too. Background work was starving the user-facing system.

Both problems had the same root cause: we were running work that needed time inside handlers that expected a fast response.

---

## Why We Did Not Rewrite Everything

When engineers hear "scaling problems," the instinct is to reach for microservices. Build proper services. Deploy containers. The whole thing.

We talked about it. Then we looked at the actual system.

Around ninety-five percent of it worked fine — auth, dashboards, reporting, all stable and fast. Real users depended on it daily. The pain lived in one specific place: long-running background tasks.

Rebuilding the whole system to fix five percent of it would have taken weeks, frozen features, and left us with something much more complex to operate. We pushed back against the instinct and asked a simpler question: what is the minimum change that fixes the actual problem?

---

## The Fix: Separate Ingestion From Processing

Split every background job into two stages:

1. **Receive** — fast, just acknowledge the event
2. **Process** — async, handled separately by a worker

A queue connects the two. The API handler enqueues the job and returns immediately. A worker running on a separate service does the actual work.

```
External system
      │
      ▼
Vercel API route  ← responds in ~50ms, just enqueues
      │
      ▼
Message queue
      │
      ▼
Background worker ← no timeout pressure, runs as long as needed
```

The external system gets its `200 OK` quickly. The retry loop disappears. Database connections stabilize because the unpredictable serverless concurrency is no longer holding them open.

---

## What the Code Looks Like

**Before** — the handler blocks until the full job finishes:

```typescript
// This blocks. If it takes over 5 minutes, Vercel kills it.
export const POST = async (req: Request) => {
  const event = await req.json();
  await processHeavyJob(event);
  return new Response("ok");
};
```

**After** — the handler enqueues and returns immediately:

```typescript
// Returns in ~50ms. The real work happens in a worker.
export const POST = async (req: Request) => {
  const event = await req.json();
  await queue.publish({ job: "heavy-sync", data: event, retries: 5 });
  return new Response("ok");
};
```

**The worker** is a separate service with no timeout constraints:

```typescript
queue.process("heavy-sync", async (job) => {
  await processHeavyJob(job.data);
  // throwing here signals the queue to retry
});
```

One thing worth saying: if the worker fails, let it throw. The queue uses that signal to schedule a retry. If you silently swallow exceptions and return success, failed jobs vanish with no trace.

---

## Protecting the Worker

When you extract a worker into its own service, its endpoint becomes publicly accessible. Anyone who finds the URL can trigger your workers directly, so you need to protect it.

The approach we used was signature verification. The queue signs each delivery with a shared secret. The worker verifies the signature before doing anything else.

```typescript
app.post("/process-job", verifyQueueSignature, async (req, res) => {
  await processHeavyJob(req.body);
  return res.json({ success: true });
});
```

Simple. If the signature is not valid, return 401 and stop.

---

## Connection Pooling

This one is easy to miss until it bites you.

In a serverless environment, each function invocation can open a new database connection. Under load, many instances spawn simultaneously, and each one grabs a connection. PostgreSQL has a connection limit. When you hit it, everything breaks — not just background jobs, but user-facing pages too.

The fix is explicit connection pooling with bounded limits:

```typescript
const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 50,
  idleTimeoutMillis: 30_000,
  connectionTimeoutMillis: 5_000,
  allowExitOnIdle: true,
});
```

The pool caps at 50 connections, closes idle ones after 30 seconds, and fails fast if a connection cannot be established rather than queuing requests indefinitely. In development we store the pool on `globalThis` so hot reloads do not leak new connections on every file change.

One more thing: background tasks that run inside the main application should not share the same pool as live API requests. A slow background job can hold a connection for several seconds, blocking a user request from getting one. For those cases, we give background tasks their own dedicated client:

```typescript
async function createOneOffDb() {
  const client = new Client({ connectionString: process.env.DATABASE_URL });
  await client.connect();
  return { db: wrapWithOrm(client), end: () => client.end() };
}

const { db, end } = await createOneOffDb();
try {
  await runBackgroundSync(db);
} finally {
  end();
}
```

Not glamorous. But it keeps user-facing requests responsive even when the background work is slow.

---

## Dead-Letter Queues

Retries handle transient failures — a network blip, a timeout during deployment, a temporary database hiccup. But some jobs fail permanently: the payload has bad data, the external system changed its format, there is a bug in the worker.

Without a plan for these, permanently failed jobs just disappear after exhausting their retries. You never know they existed.

We added a dead-letter queue. When a job exhausts all retries, instead of being discarded, it moves to a separate queue along with the error reason:

```typescript
worker.on("failed", async (job, err) => {
  const noMoreRetries = job.attemptsMade >= job.opts.maxAttempts;

  if (noMoreRetries) {
    await deadLetterQueue.add(job.name, {
      ...job.data,
      failureReason: err.message,
    });
  }
});
```

Jobs in the DLQ do not retry. They just sit there until you look at them. When you fix the bug, you can replay them. For a small team that cannot monitor everything around the clock, this is a practical safety net — a place where nothing is silently lost.

---

## Idempotency

When jobs retry, they will sometimes run the same job more than once. That is expected. What matters is that running a job twice produces the same result as running it once.

A success marker can skip a sequential replay after an earlier run completed.
This simplified example shows that optimization:

```typescript
async function processFile(fileKey: string) {
  const metadata = await storage.getMetadata(fileKey);

  if (metadata?.processed === "true") {
    return { skipped: true };
  }

  const result = await compress(fileKey);
  await storage.setMetadata(fileKey, { processed: "true" });

  return { result };
}
```

This check does not make the operation safe under every retry. Two workers can both
read an unset marker and both call `compress`. A crash after the side effect but
before `setMetadata` leaves the next run unable to tell that the work happened.

Correctness depends on the operation and storage contract. The side effect needs to
be safe to repeat, or coordinated with a durable identity and a recovery protocol.
An atomic claim can coordinate workers, but does not by itself close the crash
window between an external effect and recording completion. The snippet omits those
requirements and is not a complete idempotency implementation.

Design for this from the beginning. It is much harder to retrofit.

---

## The Workers We Built

We did not build one big "background service." We built small, focused services — each responsible for one type of work.

**Data sync worker** — handles events from the external system, both single-record updates from webhooks and full reconciliation runs.

**Document processing** — fills PDF templates with data, then flattens them. Used Python here because the reliable PDF manipulation libraries live in that ecosystem.

**File compression** — downloads files from cloud storage, compresses them, uploads them back. Written in Go for its efficiency on I/O-heavy work.

**Media optimization** — compresses uploaded images. Node.js, because the libraries we needed were already well-maintained there.

Each service does one job and is deployed independently. A failure in one does not affect the others.

---

## What the Full Picture Looks Like

The main Next.js application on Vercel barely changed. We just stopped asking it to run long jobs.

```
Next.js on Vercel
  UI, auth, CRUD, webhook ingestion
      │
      ▼
Queue Layer (push and pull)
      │
      ├── Data sync worker     (Node.js)
      ├── Document service     (Python)
      ├── File compressor      (Go)
      └── Media optimizer      (Node.js)

Each worker queue has a dead-letter queue alongside it.
```

We use two queue patterns — push (queue delivers to your endpoint) for event-driven work, and pull (worker polls the queue) for file and media jobs where we want more control over concurrency. Neither is better in general. They just fit different situations.

---

## Tradeoffs

What improved: the user-facing system became more stable. Background jobs no longer compete with dashboard requests for database connections. Long-running work runs without timeout pressure. Failed jobs retry, and permanently failed ones land in the DLQ instead of disappearing.

What got harder: debugging a failed job now means stitching together logs across multiple services. Schema migrations require coordination — the migration has to deploy before the code that depends on it. More services means more configuration, more pipelines, more things to watch.

These costs are real. They were justified because the workloads genuinely could not run inside Vercel. But they would not have been justified as upfront architecture before we knew that.

---

## For Small Teams

If you are a team of two to five people in early stages, do not start with distributed workers.

A Next.js monolith on Vercel gives you a lot: one codebase, simple deployments, fast development loops, easy debugging. The trade-off is real — you can hit the configured execution limit if background jobs get heavy. But that problem appears later, after you have real users and real usage patterns telling you what the system actually needs.

When you do hit it, extract only the part that is breaking. Leave everything else alone.

The cost of premature complexity is slower shipping, harder debugging, and time spent on infrastructure instead of product. For a small team, that is a significant cost.

Build the simple version. Let it run. Let the system tell you where it hurts.

---

## One Principle to Carry

> Fix the specific constraint. Not the whole system.

When something breaks, find the exact cause and fix that thing. Do not redesign what is working.

Our constraint was clear: certain work could not run within a 5-minute serverless function. So we moved that specific work out of Vercel. Everything else stayed the same. That discipline — resisting the urge to overengineer — matters more than any technical choice.

---

## Closing

The monolith was not a mistake. It was the right starting point.

It let us ship, validate ideas, and understand how the system actually behaved before making infrastructure commitments. The workers came later, as a response to real problems — not as upfront architecture.

The system we have now is not something we could have designed before running the product. It grew from what actually happened.

Start simple. Watch carefully. Change only what must change.

Everything else — keep it boring.

---

## Editorial correction — 2026-10-03

The original article described a five-minute limit in the deployment it discussed.
That should not be read as a universal Vercel limit today. Duration depends on the
plan, runtime and configuration. The general guidance above now links to the
current documentation; the historical deployment configuration was not rechecked.

The file-metadata example above also needs a narrower claim. It skips sequential
replays after a success marker exists. It does not prevent concurrent execution or
make a side effect atomic with writing that marker. No new production incident or
fix is established by this correction.

The queue examples illustrate background-work patterns. They should not be used to
identify the implementation behind a particular employer's webhook incident without
checking that system's own record.
