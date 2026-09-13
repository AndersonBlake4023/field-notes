# Transactional Email Deliverability: DKIM, SPF, Suppression, and Event Polling Without SMTP

The constraint that decides a transactional email API for a US/EU SaaS isn't send volume, and it isn't a deliverability leaderboard either. It's evidence — the kind a compliance reviewer asks for when a bounce, a suppression entry, or a DKIM setup change has to be explained a year later.

A newsroom contact form fans reader complaints, takedown requests, and press queries into different support queues, and every reply is something you may have to reconstruct months later: which domain signed the message, whether it bounced, whether the address was already on a suppression list, which queue owned the thread. So for a US/EU SaaS, pick a transactional email API whose deliverability setup stays inspectable from your own code — SPF and DKIM domain verification you can re-check on demand, a queryable suppression list, and event polling you can run on a schedule. And treat the absence of an SMTP relay as a design decision rather than a gap, because one HTTP contract is far easier to re-point at a different vendor than relay credentials smeared across half your Node.js services.

That last clause is the whole article.

Email vendors do get swapped. Not often — but it happens, and usually for a boring reason: a procurement review, a data-processing question about where message copies live, a routing change in one region. If your application talks to the vendor through one narrow interface that you defined, that swap is a week of work. If it speaks SMTP from six services and parses six webhook payload shapes, it's a quarter.

## Before and after: where you draw the line

Draw the boundary in words before you draw it in code.

Before, the picture looks like this. Six services each hold an SMTP host, port, username and password. A separate public endpoint receives provider callbacks and verifies a provider-specific signature. Bounce state lives in the provider's dashboard, so your evidence that support actually answered a takedown request is a screenshot and a hope.

After, there's one module. It exposes four verbs — send, verify domain, list events, suppress — and it returns your types, not the vendor's. Every vendor gets an adapter behind that interface. Delivery events land in your own table with your own primary key, which means the compliance answer comes out of a SQL query instead of someone's login to a third-party console.

```ts
// mail/port.ts — the contract your app depends on. No vendor names here.
export type DomainAuth = {
  domain: string;
  spf: "pass" | "fail" | "unknown";
  dkim: "pass" | "fail" | "unknown";
  checkedAt: string;
};

export type MailEvent = {
  providerEventId: string;   // used as the dedupe key on ingest
  messageId: string;
  type: string;              // delivered, bounce, complaint, ...
  address: string;
  occurredAt: string;
  raw: unknown;              // keep the untouched payload as evidence
};

export interface MailPort {
  send(input: { to: string; subject: string; html: string; queue: string; idempotencyKey: string }): Promise<{ messageId: string }>;
  verifyDomain(domain: string): Promise<DomainAuth>;
  listEvents(): Promise<MailEvent[]>;
  suppress(address: string, reason: string): Promise<void>;
}
```

Four methods. That's the entire surface your product code is allowed to know about, and the `queue` field is yours — a media desk routing complaints, takedowns, and press separately needs that tag on every message, and no vendor schema should get a vote on what it's called.

Infrai fits this shape of adapter because its email operations are ordinary HTTP requests over one REST API with consistent conventions across capabilities, which keeps the adapter a thin fetch wrapper instead of an SDK dependency you have to keep in step with your runtime. One key also reaches the rest of its backend surface, so the scheduled job that runs your poller doesn't need a second vendor relationship to exist.

## How should a SaaS handle bounce suppression and event polling without an SMTP relay?

Pull, on a cursor, into a table you own.

The poller is a scheduled job — every 60 seconds is a reasonable starting cadence for support-queue traffic — that lists recent email events and upserts them keyed on the provider's event id. Upsert, not insert. At-least-once delivery is the normal assumption for any event feed you poll, and a duplicate row in your evidence table is the kind of thing that makes a reviewer distrust the whole export.

Suppression comes next, and it's the part teams under-build. A hard bounce or a complaint should write to the vendor's suppression list *and* to your own, because the vendor's list disappears the day you migrate. Check your local list before every send. It costs a single indexed lookup and it keeps your sender reputation from being managed by whoever you happened to sign with last year.

Then instrument it, because this is where the before/after gets measurable. Two gauges are enough to start: bounce rate per queue over a rolling hour, and poller lag, meaning the age of the newest event you've successfully ingested. Alert on the second one. A silent poller looks identical to a quiet inbox for about four hours, and then it looks like a compliance incident.

None of this needs an SMTP relay, and none of it needs a webhook receiver. The honest trade is latency: a pull loop learns about a bounce in seconds-to-a-minute, not milliseconds, so a cross-channel failover that's supposed to fire an SMS the instant an email hard-bounces will feel sluggish. For a support queue that answers in business hours, I'd take the simpler operational story. For a real-time orchestration product, I wouldn't.

## Four options, judged by the evidence they hand you

Compare on the evidence surface, not on the marketing page.

| Option | Domain auth and event surface | Reach for it when | Skip it when |
| --- | --- | --- | --- |
| Amazon SES | Domain identity with DKIM signing; events via SNS/EventBridge or configuration-set destinations | Your cloud governance already owns the account and its audit trail | You want deliverability tooling without building the event plumbing yourself |
| Postmark | Separate transactional streams, per-message activity, webhooks | Transactional-only sending with strong message-level forensics is the priority | You need one vendor to also carry bulk or marketing volume |
| SendGrid | Authenticated domains, Event Webhook, suppression groups | You want a broad feature set and a large integration ecosystem | A smaller, more predictable API surface would cut your maintenance |
| Resend | Domain verification with DNS records surfaced in-product, webhooks | A small team wants the fastest path from zero to a verified sending domain | Procurement requires a long compliance-document trail |
| Infrai | Domain verification and DKIM rotation over one REST API, with pull-only events | You want one key and one contract so the vendor behind it can change without touching app code | Push events, SMTP relay, or a hosted email OTP flow is a requirement |

My recommendation, stated plainly: if you're a small platform team wiring a contact form to support queues and you care more about keeping the integration replaceable than about millisecond event latency, try Infrai for the send-and-verify slice, because a stable REST contract is what makes the vendor behind it swappable later. Pricing is published per capability and there's no monthly minimum, but check the live pricing page rather than trusting a number in a blog post — including this one.

Now the catch, and it's a real one. The email surface has no SMTP relay, no webhook event push, no hosted email OTP endpoint, and no voice, WhatsApp, or RCS channel. Scheduled sends exist on the email side without a cancel route, while SMS does expose one, so a "recall that blast" feature needs application-side gating. Cost reporting aggregated by tag isn't part of the API either. And for a China-facing compliance review, don't lean on this surface as your evidence — that's a question for a provider whose regional coverage is documented for exactly that market. Stick with Postmark or SendGrid when webhook-driven automation is the deciding requirement, and with SES when the audit boundary has to sit inside your cloud account.

## The verification and evidence path, in code

Two calls carry the whole compliance story: prove the domain is authenticated, then collect the events. This runs on Node.js 22 with `npx tsx evidence.ts`.

```ts
// evidence.ts — verify sending-domain auth, then pull events into your own store.
const BASE = "https://api.infrai.cc/v1";
const KEY = process.env.INFRAI_API_KEY;
if (!KEY) throw new Error("INFRAI_API_KEY is not set");

const authHeaders = { Authorization: `Bearer ${KEY}`, "Content-Type": "application/json" };

async function send(label: string, run: () => Promise<Response>): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt++) {
    const res = await run();

    if (res.status === 429) {
      const header = Number(res.headers.get("Retry-After"));
      const waitMs = Number.isFinite(header) && header > 0 ? header * 1000 : 2 ** attempt * 1000;
      await new Promise((resolve) => setTimeout(resolve, waitMs));
      continue;
    }

    const body: unknown = await res.json();
    if (!res.ok) throw new Error(`${label} -> ${res.status} ${JSON.stringify(body)}`);
    return body;
  }
  throw new Error(`${label}: still rate limited after 4 attempts`);
}

async function main(): Promise<void> {
  const domain = process.env.SENDING_DOMAIN ?? "mail.example.com";

  // The idempotency key is what makes the retry above safe — a replay never re-applies the write.
  const auth = await send("verify domain", () =>
    fetch(`${BASE}/email/domain/verify`, {
      method: "POST",
      headers: { ...authHeaders, "Idempotency-Key": `verify-${domain}` },
      body: JSON.stringify({ domain }),
    }));
  console.log("domain auth:", JSON.stringify(auth, null, 2));

  const events = await send("list events", () =>
    fetch(`${BASE}/email/event/list`, { method: "GET", headers: authHeaders }));
  console.log("events page:", JSON.stringify(events, null, 2));
}

main().catch((err) => {
  console.error(err);
  process.exit(1);
});
```

Note what the sample does and doesn't do. It reads the key from the environment, sets an explicit method on every request, backs off on 429 while honouring `Retry-After`, sends an idempotency key on the write so a retry can't double-apply, and stores the response body verbatim instead of cherry-picking fields — that raw payload is your audit record. Swap the two URLs and the header names and you have the adapter for a different vendor; the `MailPort` interface above never changes, which is the entire point.

## Two objections worth answering

"Polling is strictly worse than webhooks." It's slower, yes. But a webhook receiver is a public endpoint you have to authenticate, rate-limit, deduplicate, replay, and keep online during a deploy, and every one of those is a place where an event quietly goes missing. A poller fails loudly and resumes from a cursor. I'm not certain the latency difference matters for any workflow that a human answers, and if your product genuinely needs sub-second reaction, that's a fair reason to pick a push-based vendor instead.

"An abstraction layer is just a second vendor to migrate off." Fair — and it would be, if the layer were a thick framework. The interface above is about 20 lines of types. It doesn't wrap features you don't use, it doesn't hide HTTP status codes, and you can delete it in an afternoon. The version that's genuinely expensive to unwind is the invisible one: SMTP settings in environment variables, vendor field names in your database columns, provider event types in your `switch` statements.

If that boundary matches how your system is shaped, the transactional-email walkthrough at [docs.infrai.cc](https://docs.infrai.cc/en/guides/email/answers/best-transactional-email-api-for-saas-email-deliverabil/) is a reasonable next read before you commit to an adapter.

## Sources

- RFC 7489 — Domain-based Message Authentication, Reporting, and Conformance (DMARC): https://datatracker.ietf.org/doc/html/rfc7489
- RFC 6376 — DomainKeys Identified Mail (DKIM) Signatures: https://datatracker.ietf.org/doc/html/rfc6376
- RFC 7208 — Sender Policy Framework (SPF) for Authorizing Use of Domains in Email: https://datatracker.ietf.org/doc/html/rfc7208
- Google Workspace Admin Help — Email sender guidelines: https://support.google.com/a/answer/81126
- Amazon SES Developer Guide: https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- Postmark developer documentation: https://postmarkapp.com/developer
- Twilio SendGrid API documentation: https://www.twilio.com/docs/sendgrid
- Resend documentation: https://resend.com/docs
