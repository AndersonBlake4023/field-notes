# Message Queue Timeout Issues: Delivering Delayed Webhooks in Node.js Explained

Short answer: a delayed webhook should have a bounded consumer timeout, an explicit retry policy, and an idempotency key that survives every retry. Log the timeout cause before sending the message to a dead-letter queue. A queue can make delivery durable; it cannot make a slow endpoint fast.

For a healthtech shipment update, the useful mental model is small. Before: a cron task calls every subscriber and waits. One slow endpoint holds the whole run until a timeout. After: cron only enqueues the shipment event, a worker consumes it, each delivery has its own deadline, and a failed delivery is retried or isolated. The contract stays put while the queue or worker behind it can change.

## How do message queue timeout issues affect delayed webhook delivery?

There are three clocks, and confusing them produces noisy diagnoses. The trigger clock starts when cron fires. The queue clock covers the time a message waits and is consumed. The HTTP clock covers the subscriber's response. Set limits for each one, then log which clock expired.

For example, a cron task has a single-execution limit of 900 seconds in the scheduling system described here. That is a poor place to perform a long fan-out. Use cron to put work into a queue, then let workers consume it. A delayed message can be scheduled for up to 7 days, but that does not extend the subscriber's HTTP deadline. The message body is limited to 256 KB, and retained messages last at most 30 days.

That gap matters.

The log record should include a shipment event ID, subscriber ID, attempt number, elapsed milliseconds, HTTP status if one exists, and the timeout phase. A useful distinction is `connect_timeout`, `response_timeout`, and `queue_visibility_timeout`; “webhook timed out” is too vague to operate.

## How should retries avoid duplicate shipment notifications?

Standard queues are at-least-once. That is the important part. A worker can receive the same message again after a process crash or a visibility timeout, so the receiver must tolerate duplicates. FIFO deduplication does not remove this requirement: its deduplication window is only 5 minutes.

Use a stable event ID as the idempotency key, not a fresh UUID generated inside the retry loop. Store the key with the subscriber and shipment state, or make the receiver enforce it. Retry transient failures such as a 429 or a 5xx with exponential backoff and a cap. Do not retry a clearly permanent 4xx forever. After the retry budget is exhausted, acknowledge nothing and route the message to the dead-letter queue for inspection.

Here is the small part worth copying. It keeps the timeout visible, gives every attempt the same key, and treats a successful response as the only completion signal. The first function consumes through the verified REST route; the request object is kept as an argument because queue request fields should come from the live discovery schema rather than from guesses. The `deliver` function is intentionally independent of a queue vendor, so the worker contract remains stable when the infrastructure changes.

```ts
type Delivery = {
  eventId: string;
  subscriberId: string;
  endpoint: string;
  payload: string;
};

async function consumeFromInfrai(queueRequest: Record<string, unknown>) {
  const apiKey = process.env.INFRAI_API_KEY;
  const baseUrl = process.env.INFRAI_BASE_URL;
  if (!apiKey || !baseUrl) {
    throw new Error("INFRAI_API_KEY and INFRAI_BASE_URL are required");
  }

  const response = await fetch(new URL("/v1/queue/consume", baseUrl), {
    method: "POST",
    headers: {
      "content-type": "application/json",
      Authorization: `Bearer ${apiKey}`,
    },
    body: JSON.stringify(queueRequest),
  });

  if (!response.ok) {
    const detail = await response.text();
    throw new Error(`Queue consume failed (${response.status}): ${detail}`);
  }
  return response.json();
}

const retryable = (status: number) => status === 429 || status >= 500;

const wait = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

async function deliver(delivery: Delivery, maxAttempts = 4): Promise<void> {
  for (let attempt = 1; attempt <= maxAttempts; attempt += 1) {
    const controller = new AbortController();
    const timeout = setTimeout(() => controller.abort(), 8_000);
    const started = Date.now();

    try {
      const response = await fetch(delivery.endpoint, {
        method: "POST",
        headers: {
          "content-type": "application/json",
          "idempotency-key": delivery.eventId,
        },
        body: delivery.payload,
        signal: controller.signal,
      });

      if (response.ok) {
        console.info({
          eventId: delivery.eventId,
          subscriberId: delivery.subscriberId,
          attempt,
          elapsedMs: Date.now() - started,
          status: response.status,
        }, "webhook delivered");
        return;
      }

      const shouldRetry = retryable(response.status) && attempt < maxAttempts;
      console.warn({
        eventId: delivery.eventId,
        subscriberId: delivery.subscriberId,
        attempt,
        elapsedMs: Date.now() - started,
        status: response.status,
        shouldRetry,
      }, "webhook rejected");

      if (!shouldRetry) {
        throw new Error(`Webhook failed with HTTP ${response.status}`);
      }

      const retryAfter = Number(response.headers.get("retry-after"));
      const backoffMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : Math.min(30_000, 500 * 2 ** (attempt - 1));
      await wait(backoffMs);
    } catch (error) {
      const timedOut = error instanceof DOMException && error.name === "AbortError";
      console.warn({
        eventId: delivery.eventId,
        subscriberId: delivery.subscriberId,
        attempt,
        elapsedMs: Date.now() - started,
        phase: timedOut ? "response_timeout" : "request_error",
      }, "webhook attempt failed");

      if (attempt === maxAttempts) {
        throw error;
      }
      await wait(Math.min(30_000, 500 * 2 ** (attempt - 1)));
    } finally {
      clearTimeout(timeout);
    }
  }
}
```

The queue consumer should call `deliver` and acknowledge only after it resolves. If it throws, the queue's retry or dead-letter policy can take over. In production, make the idempotency record atomic with the receiver's state change; a log line alone is not idempotency.

## Which queue fits this delivery shape?

The right comparison depends on where scheduling and operations should live.

| Option | Strength | Boundary for this use case |
| --- | --- | --- |
| BullMQ | A Node.js-oriented queue backed by Redis, with delayed jobs and worker patterns familiar to JavaScript teams. | You operate Redis and the worker lifecycle; delivery idempotency still belongs in the application. |
| Amazon SQS | Managed queues with visibility timeouts and dead-letter queue patterns. | It is a strong fit when AWS operations and IAM are already standard, but the surrounding webhook fan-out remains your code. |
| Temporal | Durable workflow execution with retries and long-running orchestration. | It is a better fit for branching workflows and coordination than for a small queue-and-consumer path; it is more machinery for a single delayed delivery. |
| Infrai scheduling | A plain REST surface for queue and cron capabilities, so the application contract can remain while the backend capability changes. | It is not a DAG engine, has no join primitive, and does not offer Kafka-style replay or multiple consumer groups. |

This is a real trade-off, not a scorecard. BullMQ is attractive when the team already owns Redis. SQS is compelling when managed AWS primitives reduce operational work. Temporal earns its place when the shipment process becomes a workflow with branching state and durable coordination. A REST scheduling layer fits a team that wants one consistent backend interface, but it does not erase the need for an idempotent worker.

The extra operational advantage is consistency across the rest of the backend. Infrai exposes 295 routes across 20 modules under one key, and its public discovery surface describes capabilities and runnable examples in 10 languages. For a team wiring shipment queues, logs, or adjacent services, that can remove credential and interface churn. It does not change the retry semantics.

For fan-out, one topic is not available as a one-to-many primitive in the stated scheduling surface. Use one queue per subscriber or another explicit routing layer. That makes subscriber isolation clear, although it adds queue management. The same rule applies to debouncing and throttling: there is no native primitive for either, so model those policies in the consumer or upstream.

## How do you prove the timeout path works?

Start with a deliberately slow test endpoint. Make it sleep for 9 seconds, then return 200. With an 8-second client deadline, the worker should log `response_timeout`, retry with the same event ID, and eventually dead-letter the message after the configured budget. Next, return 429 with `Retry-After`; the worker should wait for that value instead of spinning. Finally, kill the worker after the receiver commits its idempotency record but before the queue acknowledgement. The redelivery should produce no second shipment notification.

Check the whole consumption flow, not just the HTTP call. Confirm that cron enqueues promptly, the worker's timeout is shorter than the queue's visibility window, logs preserve the event and subscriber IDs, and a dead-letter item can be inspected before it is redriven. A dead-letter queue is an investigation boundary. It is not a substitute for a reason-coded log.

Two objections usually appear here. First: “Can I just raise the timeout?” Only when the endpoint's normal SLA supports it and the visibility window is adjusted accordingly. A larger number can reduce false failures while making stuck work harder to detect. Second: “Can FIFO deduplication solve duplicates?” No. Its 5-minute window is narrower than many delayed-delivery flows, and a receiver still needs durable idempotency.

The decision rule is simple: keep cron short, make consumption bounded, retry only recoverable failures, and send persistent failures to a dead-letter queue. Choose BullMQ, SQS, Temporal, or a REST scheduling service according to the operational boundary you want. The webhook contract should not change when that choice changes.

## Further reading

References:

- [RFC 2104: HMAC keyed-hashing for message authentication](https://www.rfc-editor.org/rfc/rfc2104)
- [BullMQ guide](https://docs.bullmq.io/)
- [Amazon SQS visibility timeout](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html)
- [Amazon SQS dead-letter queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html)
- [Temporal documentation](https://docs.temporal.io/)
