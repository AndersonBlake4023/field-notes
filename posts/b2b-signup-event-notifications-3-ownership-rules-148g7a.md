# B2B Signup Event Notifications: 3 Ownership Rules for Email and SMS Timeout Handling

For a B2B SaaS signup flow, keep verification-link semantics in the application repository, even if a content service or transport system renders the final email or SMS. Then record the exact template revision beside each send attempt. If submission times out, mark the outcome as unresolved and let a scheduled worker query delivery status; do not infer failure and generate a second link.

**The least complex useful design is application-owned link policy plus a transport-neutral evidence ledger.** It gives product engineers control of expiry and single-use behavior while leaving channel delivery to an adapter.

| Template owner | Pick this when | Evidence to retain | Cost of the choice |
|---|---|---|---|
| Application repository | Link rules and copy ship with one product | Commit-derived version, locale, rendered-content hash | Copy changes follow application release controls |
| Shared content service | Several applications publish one reviewed message | Published revision, variables, rendered-content hash | Signup now depends on another runtime system |
| Email or SMS transport | Channel operators must edit channel-specific copy | Remote template identifier, revision when exposed, variables | Content history crosses an external boundary |

This table is about ownership, not transport selection. Email and SMS can use different delivery systems while sharing one rule: a verification attempt must point to immutable content identity and independent delivery evidence.

## How should event notifications handle email and SMS timeouts?

Choose the application repository when the template is tightly coupled to account state. The link destination, token lifetime, and single-use rule are product behavior. Reviewing those changes beside the code makes the coupling visible, and a version such as `signup-verification-v7` survives later edits. A mutable label such as `current` does not.

That last detail bites.

Choose a shared content service when multiple applications need the same approved wording or localization workflow. The service must resolve an alias to a concrete published revision before a send is recorded. Store that revision and the rendered-content hash. Otherwise, an investigation weeks later may reproduce today's copy rather than the copy attached to the original attempt.

Choose transport-owned templates when email and SMS specialists need separate channel tooling and the transport-rendered artifact is the operational source of truth. Preserve the remote identifier, any documented revision, and the submitted variables. A Git reference alone cannot identify an artifact rendered elsewhere.

There is a clean decision test: **the owner must be able to reproduce the exact customer-facing message without consulting mutable state.** If no team can do that, the ownership model is incomplete.

No revision, no audit trail.

## Make content identity precede delivery status

Start the record before calling either transport. Assign one logical notification ID for the signup action, then create a distinct attempt for each actual submission. The notification holds the verification purpose and template identity. An attempt holds channel, submission evidence, transport message ID when one exists, and the next observation time.

That ordering matters after a timeout. The remote system may have accepted the request even though the caller never received the response. Treating the timeout as a definite rejection can create two messages, possibly carrying two links. Treating it as success is equally unsupported. The honest state is unresolved.

Short version: no receipt, no conclusion.

Picture the flow in words. The signup transaction writes an outbox item. A sender claims it, renders an immutable template revision, and creates an attempt. The channel adapter submits the message. Its response updates the attempt if it arrives. Separately, a scheduled reconciler leases due attempts and asks the relevant adapter for fresh evidence. The verification endpoint changes account state; transport delivery never does.

DKIM sits on a different boundary. RFC 6376 defines a domain-level email signing mechanism and protects signed content against modification within its stated model. It does not establish that a recipient read a message or completed signup. Likewise, an SMS delivery status is transport evidence, not proof that the account owner followed the link. Keep those facts apart in dashboards and support tools.

## Implement a revision-aware polling worker

The following TypeScript keeps template provenance in the same row the worker examines, but it does not let the worker edit content or decide account verification. Adapters translate documented channel-specific results into local evidence. Raw status remains available for diagnosis.

```ts
type Channel = "email" | "sms";
type DeliveryEvidence = "accepted" | "delivered" | "failed" | "unresolved";

type DueAttempt = {
  id: string;
  notificationId: string;
  channel: Channel;
  templateRevision: string;
  renderedContentHash: string;
  transportMessageId: string | null;
  evidence: DeliveryEvidence;
  observationCount: number;
};

type Observation = {
  evidence: Exclude<DeliveryEvidence, "unresolved">;
  rawStatus: string;
};

interface AttemptLedger {
  leaseDue(options: {
    limit: number;
    now: Date;
    leaseUntil: Date;
  }): Promise<DueAttempt[]>;
  recordObservation(
    attemptId: string,
    observation: Observation,
    observedAt: Date,
  ): Promise<void>;
  reschedule(
    attemptId: string,
    observationCount: number,
    nextObservationAt: Date,
  ): Promise<void>;
  releaseAfterError(attemptId: string, nextObservationAt: Date): Promise<void>;
}

interface StatusReader {
  read(messageId: string): Promise<Observation>;
}

const retryDelayMs = [30_000, 120_000, 600_000, 1_800_000] as const;

export async function observeDueAttempts(
  ledger: AttemptLedger,
  readers: Record<Channel, StatusReader>,
  now = new Date(),
): Promise<void> {
  const attempts = await ledger.leaseDue({
    limit: 100,
    now,
    leaseUntil: new Date(now.getTime() + 60_000),
  });

  for (const attempt of attempts) {
    const delayIndex = Math.min(
      attempt.observationCount,
      retryDelayMs.length - 1,
    );
    const nextObservationAt = new Date(
      now.getTime() + retryDelayMs[delayIndex],
    );

    if (!attempt.transportMessageId) {
      await ledger.reschedule(
        attempt.id,
        attempt.observationCount + 1,
        nextObservationAt,
      );
      continue;
    }

    try {
      const observation = await readers[attempt.channel].read(
        attempt.transportMessageId,
      );
      await ledger.recordObservation(attempt.id, observation, now);

      if (observation.evidence === "accepted") {
        await ledger.reschedule(
          attempt.id,
          attempt.observationCount + 1,
          nextObservationAt,
        );
      }
    } catch {
      await ledger.releaseAfterError(attempt.id, nextObservationAt);
    }
  }
}
```

The batch size of 100, 60-second lease, and four delay values are explicit policy inputs, not transport promises. Tune them against the verification window, oldest-due-attempt age, worker capacity, and documented rate limits. Add jitter in production so workers do not synchronize on the same second. Use an atomic database claim or equivalent lease; a cron expression schedules work but does not prevent two overlapping processes from claiming the same row.

The missing-message-ID branch is deliberately boring. It preserves uncertainty and schedules another check of local or transport evidence. In some integrations, status lookup is impossible without that identifier. Polling cannot invent it. A separately defined observation deadline should eventually close the work item while retaining `unresolved` as the historical result.

Notice what the catch block does not do. It does not write `failed`. It does not render a new template. It does not send again.

Retries deserve two identities. The logical notification ID groups work for one signup action; each physical submission gets its own attempt ID. If a transport documents an idempotency mechanism, an adapter may use it, but the application ledger still owns the decision to create another attempt. This separation also prevents a copy edit from silently changing the artifact during a retry: a replacement attempt either retains the original revision by policy or records a deliberate new one.

## Operate the ownership boundary

The most revealing dashboard is a staged count, read left to right: notification created, template revision resolved, submission accepted, delivery evidence observed, verification completed. Add a gauge for the oldest due observation and histograms for time from notification creation to acceptance and to verification. A stalled content service appears before submission. A transport ambiguity appears after rendering. A broken verification endpoint appears after delivery. One chart can locate three different owners.

Keep metric dimensions bounded: channel, local evidence category, template revision, and normalized error class are useful. Email addresses, phone numbers, raw error strings, transport message IDs, and notification IDs do not belong in metric labels. Put investigation details in access-controlled structured logs and the attempt ledger under an explicit retention policy.

Tests should follow the same boundaries. Render each supported locale with representative long values and assert the selected revision and content hash. Make a fake adapter time out during submission, then verify that the ledger remains unresolved and no automatic duplicate appears. Return `accepted` for two observations and `delivered` for the third. Finally, run two workers against one due attempt and prove that only one lease succeeds.

There is also a deployment choice. Repository-owned templates can be checked in the same release as token behavior. Service-owned templates need a publish-and-pin step before application rollout. Transport-owned templates need a promotion record that maps the reviewed artifact to the remote revision. These are different workflows, but all three can satisfy the same invariant: every attempt names reproducible content.

## Limits and a practical rule

Polling is delayed evidence and consumes read capacity. It only works when a transport exposes a status lookup and the application retained the required identifier. Push callbacks may shorten the evidence gap, but they do not remove the value of a ledger or periodic reconciliation.

The approach has a clear limitation: polling is not suitable when the channel offers no readable status, when the required message identifier was lost, or when the required evidence latency is shorter than the permitted polling interval. A documented push callback, where available, supplies evidence through a different mechanism; without either mechanism, the attempt remains unobservable and the duplicate-send decision belongs to an explicit product policy. Repository-owned templates carry a separate trade-off: they are a poor fit when non-developers must publish urgent copy independently of an application release. A revisioned content service or a transport-owned template can fit that workflow better, provided the ledger captures the published revision.

Be honest about the boundary.

Template provenance also cannot prove receipt, reading, or account ownership. Delivery evidence ends at the channel boundary. The single-use link and verification endpoint establish the business outcome.

For B2B signup, choose the template owner that can reproduce the artifact and govern link semantics with the fewest cross-team handoffs. Then make submission and observation separate operations. **Content identity answers what was sent; the attempt ledger answers what is known; account state answers what the user completed.**

## Further reading

- RFC 6376, DomainKeys Identified Mail (DKIM): https://datatracker.ietf.org/doc/html/rfc6376
- Twilio SMS documentation: https://www.twilio.com/docs/sms
