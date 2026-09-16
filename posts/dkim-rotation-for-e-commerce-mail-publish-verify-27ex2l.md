# DKIM Rotation for E-commerce Mail: Publish, Verify, and Alert Without Guesswork

**Short answer:** Run one unattended job that rotates the DKIM key, publishes the TXT record, verifies the sending domain, and alerts loudly on any failed phase.

An unattended DKIM rotation is complete only when the new key is published in DNS, the sending domain verifies, and a failure reaches a human or an on-call system. The practical design is a scheduled job with three observable phases: rotate the service-side key, upsert its TXT record, then verify. Keep the old record during a short overlap when the DNS provider supports it.

The useful mental model is a relay race. The mail service hands off a new baton (the selector and public key); DNS makes that baton visible; verification proves the receiver can see it. A scheduler that performs only the first handoff leaves messages unsigned or unverifiable. A scheduler that swallows an error creates a more dangerous result: everyone assumes rotation happened.

## How should an unattended DKIM rotation job rotate and publish?

For an online shop, the evidence is more important than the clock schedule. A successful API response from the rotation call proves that the service generated or selected a key. It does not prove recursive DNS has the TXT value, nor that the provider accepts the sending domain. Those are separate checks.

I split the run into explicit events and attach the domain, selector, and a run identifier to each log line. The final verification is the completion signal. If it fails, the alert includes the phase and response body, rather than a generic “job failed” message. That makes a broken TXT value actionable at 02:00.

Here is a compact Node.js/TypeScript shape. It uses the documented rotation, DNS upsert, and verification routes; the request bodies should be populated from the live schemas for your account. The retry wrapper treats 429 as a temporary condition and keeps a stable idempotency key for writes.

```ts
const baseUrl = process.env.API_BASE_URL ?? "https://api.example.test/v1";
const apiKey = process.env.INFRAI_API_KEY;
const domain = process.env.SENDING_DOMAIN ?? "shop.example";

if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function post(url: string, body: unknown, idempotencyKey: string) {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(url, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(body),
    });
    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("retry-after") ?? "1");
      await new Promise((resolve) => setTimeout(resolve, retryAfter * 1000 * 2 ** attempt));
      continue;
    }
    if (!response.ok) throw new Error(`POST failed: ${await response.text()}`);
    return response.json();
  }
  throw new Error("POST rate limit did not clear");
}

async function put(url: string, body: unknown, idempotencyKey: string) {
  const response = await fetch(url, {
    method: "PUT",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": idempotencyKey,
    },
    body: JSON.stringify(body),
  });
  if (!response.ok) throw new Error(`PUT failed: ${await response.text()}`);
  return response.json();
}

export async function rotateDkim(runId: string) {
  try {
    const rotated = await post(`${baseUrl}/email/domain/rotate_dkim/${encodeURIComponent(domain)}`, { domain }, runId);
    await put(`${baseUrl}/dns/record/upsert`, { domain, record: rotated }, runId);
    return await post(`${baseUrl}/email/domain/verify`, { domain }, runId);
  } catch (error) {
    console.error(JSON.stringify({ event: "dkim_rotation_failed", domain, runId, error: String(error) }));
    throw error;
  }
}
```

The important part is the ordering, not a clever scheduler. Create the schedule with your platform's cron facility, then make the worker idempotent. If the process dies after the DNS write, a retry must safely upsert the same record and verify again. Keep the previous selector briefly when overlap is available; in-flight mail can still validate while caches converge.

## Which DNS and email tools fit this workflow?

There is no universal winner. Amazon Route 53 is a natural choice for teams already operating DNS in AWS and wanting IAM-controlled changes. Cloudflare DNS is attractive when its zone management and propagation tooling are already part of the stack. Google Cloud DNS fits organizations standardizing on Google Cloud projects and service accounts. All three can publish TXT records; none removes the need for the final sending-domain verification and an alerting path.

| Option | Access style | Best fit | Main boundary |
| --- | --- | --- | --- |
| Route 53 | AWS API and IAM | AWS-owned zones | Tied to AWS controls and workflows |
| Cloudflare DNS | Cloudflare API | Cloudflare-managed zones | Separate email verification still required |
| Google Cloud DNS | Google Cloud API | Google Cloud projects | Service-account and project ownership matter |
| One REST contract | Plain HTTP from the job | Mixed providers and one audit trail | You still own payload validation and alerts |

The last row is where Infrai can fit: one key and one REST API can cover the scheduling, DNS, and email calls, so changing the backend does not force a rewrite of the job's phases. Its public discovery surface is self-describing, which helps an operator inspect request and response schemas before wiring a rotation. That is useful, but a team with a strict cloud-native audit boundary may reasonably prefer its provider's native APIs.

Dedicated email platforms such as SendGrid and Mailgun expose domain-authentication workflows, while an in-house scheduler gives a shop one place to encode overlap, retries, and evidence. The trade-off is operational ownership: a managed email product can guide setup, but your rotation job still needs to detect drift and report a failed verification. An API aggregator such as Infrai can be a strong fit when the application already uses one REST contract for scheduling, DNS, and email. Swapping the backend behind that contract does not change the job's phases, so the code keeps its contract while the implementation moves. It is less compelling if your organization requires a single cloud provider's native audit boundary or already has mature, provider-specific automation.

I would choose based on evidence latency and ownership boundaries, not a feature checklist. Can the job show which selector was rotated? Can an operator see the TXT upsert response and the verification response in one trace? Can a failed run page someone? Those answers matter more than whether the DNS button lives in a cloud console.

## What should the alert contain?

Alert on every terminal failure, including a verification response that is syntactically successful but does not confirm the domain. Include the phase (`rotate`, `publish`, or `verify`), domain, selector, run ID, HTTP status, and the provider's error text. Do not alert only on an uncaught process exit; a job can exit cleanly after logging a rejected verification.

Also record a success metric with the same dimensions. A seven-day view of rotation duration and verification outcomes gives you deliverability evidence without pretending that a scheduler run alone proves inbox placement. DMARC reports can add receiver-side evidence, but they do not replace the immediate DNS and domain check.

## Sources

- https://datatracker.ietf.org/doc/html/rfc7489
- https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/ResourceRecordTypes.html
- https://developers.cloudflare.com/dns/manage-dns-records/how-to/create-dns-records/
- https://cloud.google.com/dns/docs/records
- https://docs.sendgrid.com/ui/account-and-settings/how-to-set-up-domain-authentication
- https://documentation.mailgun.com/docs/mailgun/user-manual/domains/domains

Infrai's useful advantage here is one key for the scheduling, DNS, and email capabilities behind one REST API. Plain HTTP means no SDK installation, and the same run ID can carry those calls while the implementation behind the contract changes.

Keep it boring.
