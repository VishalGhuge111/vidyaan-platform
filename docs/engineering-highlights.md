# Engineering highlights

Short, simplified versions of code from the private repo. Names and business values are removed. Real code has more error handling and logging.

---

## 1. Webhook signature, checked on the raw body

Razorpay signs the exact bytes it sends. If you parse the JSON and stringify it again, key order or spacing can change and a good signature looks bad. So the route reads the body once as text, verifies, and only then parses.

```ts
import { createHmac, timingSafeEqual } from "node:crypto";

export function verifyWebhookSignature(
  rawBody: string,
  signature: string,
  secret: string,
): boolean {
  if (!secret || !signature) return false;
  const expected = createHmac("sha256", secret).update(rawBody).digest("hex");
  const a = Buffer.from(expected);
  const b = Buffer.from(signature);
  return a.length === b.length && timingSafeEqual(a, b);
}

// In the route: read once, verify, only then JSON.parse.
const rawBody = await request.text();
const sig = request.headers.get("x-razorpay-signature") ?? "";
if (!verifyWebhookSignature(rawBody, sig, env.WEBHOOK_SECRET)) {
  return NextResponse.json({ ok: false }, { status: 401 });
}
```

The length check comes first because `timingSafeEqual` throws on unequal lengths. An empty secret never matches.

---

## 2. Activation that can run twice and still grant once

The checkout callback and the webhook can both land for the same payment, at the same moment. A per payment advisory lock makes them queue. A marker row written in the same transaction as the grant makes the second one a no op.

```ts
export async function activatePayment(paymentId: string) {
  return db.transaction(async (tx) => {
    // Same payment queues here. Different payments never wait on each other.
    await tx.execute(sql`select pg_advisory_xact_lock(hashtext(${paymentId}))`);

    const row = await tx.query.payments.findFirst({ where: eq(payments.id, paymentId) });
    if (!row) throw new Error("payment not found");

    const done = await tx.query.auditLogs.findFirst({
      where: and(eq(auditLogs.action, "payment.activated"), eq(auditLogs.entityId, row.id)),
    });
    if (done) return { alreadyDone: true };

    await grantWhatWasBought(tx, row); // plan or batch membership

    await tx.insert(auditLogs).values({
      action: "payment.activated",
      entityType: "payments",
      entityId: row.id,
    });
    return { alreadyDone: false };
  });
}
```

---

## 3. A webhook that never tells Razorpay "ok" when it was not

An early version answered 2xx even when activation threw. The student paid and got nothing, and Razorpay never retried. Now the decision is a small function with its side effects passed in, so the failure path is unit tested without a database.

```ts
export async function settleCaptured(
  currentStatus: string,
  deps: {
    markPaid: () => Promise<boolean>; // created or failed -> paid, one atomic UPDATE
    activate: () => Promise<unknown>; // idempotent grant
    onPaid: () => Promise<void>;      // only for the call that won the transition
    onActivationError: (e: unknown) => void;
  },
): Promise<"activated" | "retry"> {
  const won = currentStatus === "paid" ? false : await deps.markPaid();
  try {
    await deps.activate();
  } catch (error) {
    deps.onActivationError(error);
    return "retry"; // route answers 500, Razorpay redelivers
  }
  if (won) await deps.onPaid();
  return "activated";
}
```

A redelivery skips the status update and goes straight to activation, which is safe to repeat.

---

## 4. The quiz answer, with the database clock as the judge

Grading happens on the server. The window check and the latency calculation sit inside the same insert, so there is no read then write gap for a late answer to slip through.

```ts
const isCorrect = gradeAnswer(question.type, correctOptionIds, selectedOptionIds);

const inserted = await db.execute(sql`
  insert into answers
    (college_id, quiz_push_id, user_id, session_id, selected_option_ids,
     is_correct, responded_at, latency_ms)
  select qp.college_id, qp.id, ${user.id}::uuid, qp.session_id,
         ${selectedArray}, ${isCorrect}, now(),
         round(extract(epoch from (now() - qp.pushed_at)) * 1000)::int
  from quiz_pushes qp
  where qp.id = ${quizPushId}::uuid and now() < qp.closes_at
  returning id
`);

// A unique index on (quiz_push_id, user_id) rejects the second answer.
// An empty result means the window closed a moment ago.
return NextResponse.json({ ok: true, accepted: inserted.length > 0 });
```

The type that goes to the browser has no `isCorrect` field at all, so the answer cannot leak by accident.

---

## 5. A rate limiter in Postgres that fails open

One atomic upsert per request. If the database is slow or the limiter errors, the request goes through. For a platform used in exams, a limiter that can block a student is worse than no limiter.

```ts
async increment(key: string, expiresAt: Date) {
  const rows = await db.execute<{ count: number }>(sql`
    INSERT INTO rate_limit_hits (key, count, expires_at)
    VALUES (${key}, 1, ${expiresAt.toISOString()})
    ON CONFLICT (key) DO UPDATE SET count = rate_limit_hits.count + 1
    RETURNING count
  `);
  return Number(rows[0]?.count);
}

export async function checkLimit(input: { key: string; limit: number; windowSeconds: number }) {
  const w = windowFor(input, Date.now());
  try {
    const count = await withTimeout(store.increment(w.storageKey, w.expiresAt), 1000);
    return { allowed: count <= input.limit, count, retryAfterSeconds: w.retryAfterSeconds };
  } catch {
    return { allowed: true, count: 0, retryAfterSeconds: w.retryAfterSeconds };
  }
}
```

It is applied to public routes only: contact form, email check, batch join attempts, CSV import. Never to login, test start, answers, submit, or payments.

---

## 6. The answer queue

College wifi drops. The queue keeps one request in flight per question, always sends the newest answer, mirrors to session storage, and retries on its own.

```ts
private async run(questionId: string): Promise<void> {
  if (this.inFlight.has(questionId)) return;
  this.inFlight.add(questionId);
  let failed = false;
  try {
    for (;;) {
      const sending = this.pending.get(questionId);
      if (!sending) break;
      let result: "saved" | "drop" | "retry";
      try { result = await this.send(sending); } catch { result = "retry"; }
      if (result === "retry") { failed = true; break; }
      // Forget it only if nothing newer arrived while it was in flight.
      if (this.pending.get(questionId) === sending) {
        this.pending.delete(questionId);
        this.write(); // session storage mirror, a reload loses nothing
        break;
      }
    }
  } finally {
    this.inFlight.delete(questionId);
    if (failed) this.scheduleRetry();
    this.notifyIfIdle();
  }
}
```

---

## 7. CSV formula injection

Student names are typed by students. A name that starts with `=` becomes a formula when a trainer opens the export in Excel. Every string cell in an export goes through this.

```ts
const FORMULA_START = /^[=+\-@\t\r]/;

export function neutraliseFormula(value: string): string {
  return FORMULA_START.test(value) ? `'${value}` : value;
}

function csvField(value: string | number | null | undefined): string {
  const s =
    value == null ? "" : typeof value === "string" ? neutraliseFormula(value) : String(value);
  return /["\r\n,]/.test(s) ? `"${s.replace(/"/g, '""')}"` : s;
}
```

Large exports are streamed in chunks of 500 rows, so memory stays flat however big the batch is.

---

## 8. The service worker denylist

This one came from a real bug. The registration finish page was blank for some users. The service worker was caching a page that depends on the session and serving an old copy. The fix is a small list that is checked before any cache rule, with a unit test.

```ts
export const NEVER_CACHE: RegExp[] = [
  /^\/api(\/|$)/,
  /^\/(login|register|logout|forgot-password|reset-password|set-password|redirect|join)(\/|$)/,
  /^\/auth(\/|$)/,
  /^\/[sta](\/|$)/, // all three role panels, including the live class
];

export function isDenied(pathname: string): boolean {
  return NEVER_CACHE.some((re) => re.test(pathname));
}
```

In the worker it is the first runtime rule and uses network only, so it wins over every cache strategy below it.
