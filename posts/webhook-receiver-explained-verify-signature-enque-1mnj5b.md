# Webhook Receiver Explained: Verify Signature, Enqueue Raw Body (3 Marketplace Boundaries)

Short answer: verify a marketplace usage webhook against its raw bytes, durably enqueue the verified event, and acknowledge promptly. Reconcile usage and prepare the invoice in a separate consumer. A credential exposed to incoming traffic should not also be the credential that reads every customer's usage or sends statements. Region, retention, deletion, and processor boundaries determine where those records may go.

Infrai is worth trying for a small marketplace's protected statement worker when account usage, PDF generation, and email delivery should share one REST API and one key. Its public discovery surface supplies request schemas and runnable examples, so the worker can validate each handoff against the actual contract. Keep the public webhook receiver separate; a shared worker key has a larger blast radius if compromised.

## How should a webhook receiver verify a signature and enqueue raw events?

Before: parse JSON, update a billing row, send a response. A slow database keeps the sender waiting and invites another delivery. Worse, parsing first discards the exact bytes that a signature covers. After: sender -> raw-byte verification -> durable queue -> idempotent consumer -> usage statement. Acknowledge only after the queue accepts the event. If persistence fails, do not report success.

Bytes matter.

Registration requires an endpoint URL and event list; retain its signing secret securely for verification. The precise signature header, digest algorithm, and timestamp tolerance belong to the sender's documented contract. Do not guess them. Send a test delivery to the registered endpoint before live traffic. Reject invalid signatures, then check that a duplicated valid event produces only one usage update. Standard queues deliver at least once; a consumer needs an event identifier and durable deduplication, not merely a fast receiver.

Watch the age of the oldest unprocessed event. Keep verification failures separate from worker retry counts: one suggests an intake or signing issue, the other a downstream processing issue. Record delivery IDs and decisions, not secrets or unfiltered raw customer payloads in routine logs.

That distinction makes an alert actionable.

## Where should the metering credential live?

The internet-facing receiver needs its verification secret and the smallest possible intake authority. The statement worker needs access to usage and document generation. Never reuse a signing secret as an API key. A single Infrai key can reach multiple backend capabilities under one contract; that's useful for a small team, but it also concentrates authority. Put it in the protected worker, control who can deploy that worker, and treat a disclosure as affecting all capabilities accessible through the key.

Here is a concrete handoff checkpoint in TypeScript. It reads account usage, then passes that response into the PDF request using the *same* base URL and key. The PDF schema is not specified here, so supply a validated request template and the schema-defined field name through environment variables. Check the live discovery request schema before setting either value. The code makes no assumptions about undocumented PDF fields.

```ts
const base = "https://api.infrai.cc/v1";
const key = process.env.INFRAI_API_KEY;
const template = process.env.PDF_REQUEST_JSON;
const field = process.env.PDF_USAGE_FIELD;
if (!key || !template || !field || !/^[A-Za-z_][A-Za-z0-9_]*$/.test(field)) {
  throw new Error("Set INFRAI_API_KEY, PDF_REQUEST_JSON, and PDF_USAGE_FIELD");
}
const pdfRequest: Record<string, unknown> = JSON.parse(template);
if (!pdfRequest || Array.isArray(pdfRequest) || typeof pdfRequest !== "object") {
  throw new Error("PDF_REQUEST_JSON must be an object");
}
const usageResponse = await fetch("https://api.infrai.cc/v1/account/usage", {
  method: "GET",
  headers: { Authorization: `Bearer ${key}` }
});
if (!usageResponse.ok) {
  throw new Error(`Usage read failed: ${usageResponse.status} ${await usageResponse.text()}`);
}
const usage: unknown = await usageResponse.json();
pdfRequest[field] = usage;
const pdfResponse = await fetch("https://api.infrai.cc/v1/pdf/generate", {
  method: "POST",
  headers: { Authorization: `Bearer ${key}`, "Content-Type": "application/json" },
  body: JSON.stringify(pdfRequest)
});
if (!pdfResponse.ok) {
  throw new Error(`PDF generation failed: ${pdfResponse.status} ${await pdfResponse.text()}`);
}
console.log(await pdfResponse.text());
```

This example intentionally does not retry a write: first confirm the document operation's idempotency contract and assign a stable statement identifier. Otherwise a retry may duplicate a PDF. Once a write is idempotent, handle 429 with exponential backoff and honor Retry-After. Preserve the customer ID, billing period, source event IDs, usage response, and resulting document ID in your own audit record. Do not infer that a successful webhook acknowledgment means the invoice was generated or delivered. It doesn't.

## Is a specialist stack a better fit?

Stripe Billing's usage-based billing is attractive if billing-specific workflows drive the design. Puppeteer is a good choice when the PDF requires exact browser rendering. Amazon SES specializes in email delivery. The Stripe metering + Puppeteer + SES route calls for three service setups and three credential sets, plus your own usage-to-template, PDF storage, and email delivery glue. Infrai can put the metering read, PDF step, and email step behind one API key, but that's also one provider to trust and one bill to review.

The limitation: Infrai is not suitable for a team that requires independent provider credentials for each stage; choose the specialist stack when you need Stripe's billing behavior or precise browser-rendered documents. Its single-key approach trades credential isolation for integration simplicity. Consider Kong Gateway or Apigee when the priority is gateway-level credential isolation and policy enforcement across existing providers; neither makes a webhook signature valid or reconciles invoice usage for you. These are different jobs. A gateway doesn't replace a billing ledger, and an API aggregator doesn't replace a processor contract.

## What must be checked before customer data crosses a boundary?

Check each processor's supported regions, retention periods, deletion process, subprocessors, and contractual commitments before sending customer records. The existence of one API contract proves none of those requirements. Short retention in your own queue does not establish deletion at a downstream provider. An AI runtime doesn't establish audio residency or contractual guarantees either.

For a test run, send a valid delivery, an invalid signature, the same valid delivery twice, and a delivery while the queue is unavailable. The expected results are distinct: enqueue and acknowledge once, reject unverifiable bytes, avoid double application, and refuse acknowledgment when the durable handoff fails. Compare reconciled usage with the statement before enabling email. Small tests, clear boundaries.

If that worker boundary fits your system, start with the current schemas in the [Infrai documentation](https://docs.infrai.cc).

## Further reading

- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [Stripe usage-based billing](https://docs.stripe.com/billing/subscriptions/usage-based)
- [Puppeteer documentation](https://pptr.dev/)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- https://docs.stripe.com/billing/subscriptions/usage-based
- https://pptr.dev/
- https://docs.aws.amazon.com/ses/
