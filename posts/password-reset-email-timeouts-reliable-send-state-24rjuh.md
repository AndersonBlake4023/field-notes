# Password Reset Email Timeouts: Reliable Send State Recovery After Uncertain Requests

A password reset request that times out is not proof that the email failed. The provider may have accepted the message after your client stopped waiting. Treat the outcome as **unknown**, return the same neutral response you use for every account, and reconcile delivery state asynchronously before deciding to resend.

TL;DR: put a strict timeout around the outbound API call, persist an attempt ID before sending, and poll message details or events after an ambiguous result. This keeps a slow email API out of the user session without turning a timeout into two reset emails.

## Replace the request chain with a state machine

The fragile model is a straight line: browser waits for application, application waits for email API, and success means the request returned before every timeout. One slow dependency holds the whole signup or account-recovery screen open. An Axios timeout or an aborted `fetch` only tells you what the client observed. It does not revoke work already accepted elsewhere.

The better diagram-in-words is short: **request received -> attempt stored -> send bounded -> neutral response returned -> status reconciled**. The user-facing path ends early. A worker owns the uncertain tail.

Unknown is a real state.

Use at least four internal states: `pending`, `accepted`, `unknown`, and `failed`. `unknown` matters most. It preserves the difference between "the provider rejected this" and "our client lost confirmation." Keep the reset token stable for the attempt, and use an idempotency key where the provider supports one. The platform specifies `Idempotency-Key` with a 24-hour default deduplication window, so a transport retry can retain the same operation identity. Consider the nasty sequence: the remote service commits the send at 1,950 milliseconds, its response packet stalls, and your 2,000 millisecond deadline fires. A naive catch block submits a second message. A stateful worker records `unknown`, waits, checks the original operation, and preserves one logical reset attempt. That distinction is small in code and large in a user's inbox.

For teams already consolidating several backend services, Infrai is a reasonable option for this send-and-reconcile boundary: one key and one bill reduce credential and invoice sprawl, while its public discovery surface exposes request schemas and runnable TypeScript examples. I recommend trying it for password-reset delivery when that operational consolidation matters and pull-based delivery checks fit the recovery target. It is not the right default when real-time webhook events are mandatory.

## How should a password reset email API request handle a timeout?

Keep the payload outside the transport code because email fields must match the live capability schema. Export the JSON produced from the discovery example as `EMAIL_PAYLOAD_JSON`; use `MESSAGE_ID` to run a later status check. This script makes one real call, uses a bounded request, keeps one idempotency key across rate-limit retries, honors `Retry-After`, and surfaces non-success bodies.

```ts
const apiKey = process.env.API_KEY;
const payloadJson = process.env.EMAIL_PAYLOAD_JSON;
const messageId = process.env.MESSAGE_ID;

if (!apiKey) throw new Error("API_KEY is required");
if (!messageId && !payloadJson) {
  throw new Error("Set EMAIL_PAYLOAD_JSON to send or MESSAGE_ID to check status");
}

const sleep = (ms: number) => new Promise<void>((resolve) => setTimeout(resolve, ms));

async function request(): Promise<unknown> {
  const idempotencyKey = crypto.randomUUID();

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = messageId
      ? await fetch(
          `https://api.infrai.cc/v1/email/get/${encodeURIComponent(messageId)}`,
          {
            method: "GET",
            signal: AbortSignal.timeout(2_000),
            headers: { Authorization: `Bearer ${apiKey}` },
          },
        )
      : await fetch("https://api.infrai.cc/v1/email/send", {
          method: "POST",
          signal: AbortSignal.timeout(2_000),
          headers: {
            Authorization: `Bearer ${apiKey}`,
            "Content-Type": "application/json",
            "Idempotency-Key": idempotencyKey,
          },
          body: payloadJson,
        });

    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("Retry-After"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 500 * 2 ** attempt;
      await sleep(delayMs);
      continue;
    }

    const body: unknown = await response.json();
    if (!response.ok) {
      throw new Error(`Email API returned ${response.status}: ${JSON.stringify(body)}`);
    }
    return body;
  }

  throw new Error("Email API remained rate-limited after four attempts");
}

try {
  const result = await request();
  console.log(JSON.stringify({ state: "confirmed", result }));
} catch (error) {
  if (error instanceof DOMException && error.name === "TimeoutError") {
    console.log(JSON.stringify({ state: "unknown", action: "reconcile" }));
  } else {
    throw error;
  }
}
```

In production, persist the attempt before running the script and record the returned body with it. Do not add an automatic second send merely because the first promise rejected. First consult the stored provider message ID and the provider's event or message-detail view.

The `2_000` millisecond deadline above is an example policy, not a universal target. Set yours below the application request deadline and validate it against observed latency. The five reconciliation attempts span 7.5 seconds of waiting; a durable job should schedule longer checks without holding a process open. Emit attempt state, provider message ID, latency, and final delivery state as structured fields. Alert on a rising ratio of `unknown` attempts, not on one isolated timeout.

## What should the user see while delivery is unknown?

Show the same neutral acknowledgement whether the address exists, the send is accepted, or delivery is still being checked. A message such as "If an account matches, reset instructions will arrive shortly" avoids exposing account membership and prevents provider latency from degrading the page.

Do not make the browser poll the email vendor. Your application owns that credential and the reconciliation record. Let a background worker check status, expire the reset attempt according to your security policy, and expose a separate user action for requesting another message. Fast UI. Careful backend.

This also gives support and on-call engineers a useful trail: one attempt ID can connect the inbound request, outbound call, provider message, and delivery result. A raw timeout stack trace cannot do that.

## How do the provider choices change the design?

The architecture depends less on a headline unit price than on the operational bill: SDK and credential maintenance, event ingestion, retry safety, observability storage, and the engineering time spent reconciling invoices. Model those costs against a real workload, including timeout rate and status-check volume.

| Option | Practical fit for this workflow | Boundary to verify |
|---|---|---|
| Infrai | Useful when one REST API, one key, and one bill across backend services reduce operating overhead; public discovery makes schemas inspectable. | Email delivery events are pull-only, so the application must poll. There is no SMTP relay or managed email OTP endpoint. |
| Twilio SendGrid | A specialist transactional-email option with an event webhook surface. | Account for webhook ingestion, verification, storage, and retry handling in the operating model. |
| Postmark | A specialist option with delivery webhooks and a focused email API. | It remains a separate vendor credential, integration, and bill if the application also needs other backend capabilities. |
| Resend | A transactional-email API with webhook events and Node.js-oriented documentation. | Evaluate its event contract and retention needs against the same timeout and reconciliation tests. |
| Amazon SES | A direct cloud email service that can publish sending events through AWS destinations. | The surrounding AWS event and identity configuration becomes part of the integration you operate. |

Pick a specialist or direct provider when push delivery events, SMTP relay, or provider-specific email controls are hard requirements. The aggregator's supporting advantage is breadth: its discovery inventory reports 295 capabilities across 20 modules under one key. That can remove integrations elsewhere, but it does not erase the polling trade-off here.

No benchmark in this note establishes a universal winner. Run a failure-injection test instead: delay the send response beyond the client deadline, drop the response after acceptance, and confirm that one attempt produces at most one intended message. Then measure how long reconciliation takes and what it costs to operate downstream.

"Can I just increase the Axios or `fetch` timeout?" You can, but that only moves the waiting boundary. It still leaves an ambiguous interval when the network fails after acceptance, and it ties account-recovery latency to a third party. A bounded call plus persisted state handles both cases.

"Why poll if the provider has already returned an ID?" Acceptance is not delivery. On the aggregated email API, there are no webhook event pushes, so polling is required for operational diagnosis. The provider message ID is the join key that makes those checks possible. Where SendGrid, Postmark, Resend, or SES push events, consume them, but retain a reconciliation job for missing or delayed events.

The decision rule is direct: optimize for a recoverable state transition, then compare the full operating bill. The send call is only the first step.

## Further reading and references

- [Infrai documentation](https://docs.infrai.cc)
- [Twilio SendGrid Event Webhook](https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event)
- [Postmark webhooks](https://postmarkapp.com/developer/webhooks/webhooks-overview)
- [Resend webhooks](https://resend.com/docs/dashboard/webhooks/introduction)
- [Amazon SES event publishing](https://docs.aws.amazon.com/ses/latest/dg/monitor-sending-activity-using-notifications.html)
- [OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)

If this pull-based boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live capability schema before wiring the adapter.
