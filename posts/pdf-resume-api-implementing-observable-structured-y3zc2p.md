# PDF Resume API: Implementing Observable Structured JSON for Applicant Tracking

Short answer: keep the applicant-tracking schema and its version in your service, then require any PDF extraction component to return values with page-level evidence. Trace each conversion, reject records that violate explicit quality rules, and store the original document hash beside the accepted JSON. This makes template ownership the deciding boundary: a parser may suggest data, but it does not get to redefine the hiring record.

| Template owner | Pick this when | Main benefit | Operational warning |
|---|---|---|---|
| Your ATS service | Fields drive workflow, policy, or reporting | Stable record contract and controlled migrations | You must maintain mappings and quality rules |
| Extraction component | You are exploring unknown document shapes | Fast schema discovery | Output can drift when extraction behavior changes |
| Shared versioned contract | Several internal teams publish and consume records | Coordinated evolution | Governance adds review and rollout work |

For a B2B SaaS applicant-tracking system, the first option is the dependable default. It also prepares the system for later server-side employment-contract signing: candidate facts enter a controlled record, while contract templates and signature audit events remain separate, owned artifacts. Do not let a resume parser silently become the authority for either.

## How should an API parse a PDF resume into structured JSON?

Pick service ownership when a field has consequences. If `workAuthorization` changes routing, or an employment date is inserted into a contract template, the application needs a reviewed definition, a schema version, and a migration path. The extraction boundary should return observations. Your service decides which observations become record values.

Pick extraction ownership for discovery. A recruiting operations team may need to inspect the variety of headings and layouts before committing to a durable schema. Preserve those exploratory results outside the canonical applicant record; promoting a field should be a deliberate schema change, not an accidental side effect of a new document layout.

A shared contract works when multiple internal services genuinely co-own the record. Make one repository or registry authoritative, assign reviewers, and deploy readers before writers when adding fields. This is slower. That friction is useful when a change can affect search, reporting, retention, or a downstream contract.

The wrong comparison is “Which API returns the most fields?” The useful comparison is “Who can change the meaning of a field, and how will we detect the change?”

Schema drift is quiet.

## Instrument the evidence boundary

Treat the pipeline as five observable transitions: bytes arrive; a document is identified; text and coordinates are extracted; candidate fields are proposed; the owned schema accepts or rejects them. Picture five boxes in a row. A trace connects the boxes. Metrics count outcomes at each edge. Logs explain a single rejected record without copying resume contents into the logging system.

The following TypeScript example keeps that boundary small. It uses a pluggable extractor, an owned versioned record, field-level evidence, a SHA-256 document identifier, and structured events. The included extractor is deterministic test data, so the example runs without pretending to parse a real PDF.

```ts
import { createHash, randomUUID } from "node:crypto";

type Evidence = {
  page: number;
  sourceText: string;
  confidence: number;
};

type ProposedField = {
  value: string;
  evidence: Evidence;
};

type Extraction = {
  name?: ProposedField;
  email?: ProposedField;
  skills: ProposedField[];
};

interface ResumeExtractor {
  extract(pdf: Uint8Array): Promise<Extraction>;
}

type ApplicantRecordV1 = {
  schemaVersion: "applicant-record/1";
  documentSha256: string;
  fullName: string;
  email: string;
  skills: string[];
  evidence: Record<string, Evidence[]>;
};

type ParseEvent = {
  traceId: string;
  stage: "received" | "extracted" | "validated" | "rejected";
  documentSha256: string;
  schemaVersion: ApplicantRecordV1["schemaVersion"];
  durationMs?: number;
  reasonCodes?: string[];
};

const emit = (event: ParseEvent): void => {
  process.stdout.write(`${JSON.stringify(event)}\n`);
};

const normalizeEmail = (value: string): string => value.trim().toLowerCase();

async function parseResume(
  pdf: Uint8Array,
  extractor: ResumeExtractor,
): Promise<ApplicantRecordV1> {
  const traceId = randomUUID();
  const documentSha256 = createHash("sha256").update(pdf).digest("hex");
  const schemaVersion = "applicant-record/1" as const;
  const startedAt = performance.now();

  emit({ traceId, stage: "received", documentSha256, schemaVersion });
  const proposed = await extractor.extract(pdf);
  emit({
    traceId,
    stage: "extracted",
    documentSha256,
    schemaVersion,
    durationMs: Math.round(performance.now() - startedAt),
  });

  const reasonCodes: string[] = [];
  if (!proposed.name?.value.trim()) reasonCodes.push("NAME_MISSING");
  if (!proposed.email?.value.includes("@")) reasonCodes.push("EMAIL_INVALID");
  if (proposed.name && proposed.name.evidence.page < 1) {
    reasonCodes.push("NAME_EVIDENCE_INVALID");
  }
  if (proposed.email && proposed.email.evidence.page < 1) {
    reasonCodes.push("EMAIL_EVIDENCE_INVALID");
  }

  if (reasonCodes.length > 0) {
    emit({ traceId, stage: "rejected", documentSha256, schemaVersion, reasonCodes });
    throw new Error(`Resume rejected: ${reasonCodes.join(",")}`);
  }

  const record: ApplicantRecordV1 = {
    schemaVersion,
    documentSha256,
    fullName: proposed.name!.value.trim(),
    email: normalizeEmail(proposed.email!.value),
    skills: proposed.skills.map((field) => field.value.trim()).filter(Boolean),
    evidence: {
      fullName: [proposed.name!.evidence],
      email: [proposed.email!.evidence],
      skills: proposed.skills.map((field) => field.evidence),
    },
  };

  emit({ traceId, stage: "validated", documentSha256, schemaVersion });
  return record;
}

const fixtureExtractor: ResumeExtractor = {
  async extract(): Promise<Extraction> {
    return {
      name: {
        value: "Avery Chen",
        evidence: { page: 1, sourceText: "Avery Chen", confidence: 0.99 },
      },
      email: {
        value: "AVERY@EXAMPLE.COM",
        evidence: { page: 1, sourceText: "AVERY@EXAMPLE.COM", confidence: 0.98 },
      },
      skills: [
        {
          value: "TypeScript",
          evidence: { page: 2, sourceText: "TypeScript", confidence: 0.96 },
        },
      ],
    };
  },
};

const fixturePdf = new TextEncoder().encode("fixture-not-a-real-pdf");
parseResume(fixturePdf, fixtureExtractor).then((record) => {
  process.stdout.write(`${JSON.stringify(record, null, 2)}\n`);
});
```

Notice what the event omits: names, email addresses, extracted text, and file bytes. The trace ID joins stages; the digest identifies the input; reason codes support aggregation. Evidence stays with the protected applicant record, where access and retention can match the resume. This separation is easy to miss. Logging the entire extraction result feels convenient during development and creates a second, loosely governed copy of applicant data.

Use confidence as an observation, not truth. The example records it but validates required fields with explicit rules. A threshold alone cannot tell you that an email belongs to a reference rather than the applicant, or that two columns were read in the wrong order.

## Test the failures that clean samples hide

Build a fixed evaluation corpus before comparing extraction approaches. Keep the documents access-controlled and record why each case exists. Start with at least 7 deliberately different fixtures: a 1-page text PDF, a scanned page, a 2-column resume, repeated headers, a rotated page, a document with no email address, and text containing characters outside ASCII. Consider the 2-column case closely. A visually obvious name can be extracted after a sidebar heading, the email can be paired with a reference, and every required key can still be present, so a shallow “valid JSON” check reports success. The field values, page evidence, and known expected record must be evaluated together. PDF is a document format with a broad specification; visual reading order is not the same thing as a guaranteed semantic resume structure. Version this corpus alongside the owned schema, because a fixture without its expected schema version becomes ambiguous after a migration.

For every fixture, assert both the accepted value and its evidence. A name without the correct page reference is not a complete success. Also assert that invalid output stops before persistence. This yields three crisp counters: accepted, rejected for expected reasons, and unexpected pipeline failure.

Track quality by schema version and extractor version. A deployment can leave request success unchanged while moving the wrong text into `fullName`; transport uptime will look healthy. Field acceptance rates, missing-evidence counts, and reason-code distributions expose that regression. Alert on a sustained change from an established baseline, after segmenting by relevant input class. A single universal threshold can hide a failure concentrated in scanned or multi-column documents.

Transport success is not parsing success.

The tradeoff is explicit: rich evidence costs storage and review time, but it shortens investigations and supports human correction. Retaining every intermediate artifact offers even more detail, yet it expands the sensitive-data footprint. Keep only artifacts tied to a stated debugging, audit, or legal need, and apply the same access controls and deletion lifecycle as the source resume.

## Connect hiring records to contract audit trails

Do not render an employment contract directly from the extractor response. First accept a versioned applicant record. Then create a separate contract-render request that names the contract template version and snapshots the approved values used for rendering. Signing events should reference the rendered document digest and the actor or system action that produced each transition.

This gives the B2B SaaS platform two clean chains: resume bytes to evidence-backed applicant JSON, and approved applicant JSON plus an owned template to a signed-contract record. The chains can share trace context while retaining different permissions and lifecycles. Template ownership remains visible. Good.

Before release, run malformed and oversized inputs through the same boundary, cap processing resources, define retry behavior for transient extraction failures, and make persistence idempotent on the document digest plus schema version. Keep failed records out of automated contract generation. A human review queue is a valid terminal state, not an exception to hide.

## Limits

This design does not prove that extracted claims are true; it shows where a value came from and how software transformed it. It also does not make every PDF machine-readable. Scanned, encrypted, malformed, or visually complex files may require a different extraction path or human review.

The decision rule stays concise: own the hiring schema and contract templates when they drive business actions; demand evidence from replaceable extraction components; observe semantic quality, not only request success. That boundary turns structured JSON into an auditable record instead of an opaque guess.

## Sources

- https://www.iso.org/standard/75839.html
- https://www.rfc-editor.org/rfc/rfc8259
- https://www.w3.org/TR/trace-context/
- https://opentelemetry.io/docs/specs/otel/trace/
