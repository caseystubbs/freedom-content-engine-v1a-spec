# Freedom Content Engine
## V1A Implementation Specification

**Status:** Final-review draft — transport and PostgreSQL capability gates remain unproven; do not implement yet  
**Architecture source:** *Freedom Content Engine — Architecture Approved for V1A Implementation*  
**Scope:** V1A only  
**Date:** September 12, 2026  
**Implementation target:** Existing Freedom Hub application and its existing Railway PostgreSQL database

---

## 1. Purpose and implementation boundary

This document converts the approved V1A architecture into an implementation contract. It defines the database, transport, authentication, state machines, adapters, attribution path, UI behavior, tests, rollout, and rollback requirements.

This is a specification only. No production code, production migration, production feature flag, external draft, email send, or publication should be created from this document until Casey reviews and approves the specification.

### 1.1 V1A operating slice

```text
Campaign
  → Master Brief revision
  → Context Snapshot
  → Durable Work Job
  → Casey starts ChatGPT Work
  → Work returns one Substack output and one 7-Day Clicker email
  → Immutable output versions
  → Evidence validation
  → Casey approves exact versions
  → Hub Substack adapter / Hub ECC adapter
  → Existing Hub Substack queue / ECC operational draft
  → Casey manually publishes or sends
  → Tracked result
```

### 1.2 Explicitly excluded from V1A

V1A must not implement or activate:

- X distribution
- Telegram distribution
- LinkedIn distribution
- Substack Notes automation
- 30-Day Active email migration
- YouTube output automation
- Autonomous publishing
- Automatic email sending
- Automatic campaign strategy changes
- Full multi-platform metrics ingestion
- A generalized multi-platform publisher
- Complex multi-touch attribution
- AI-driven context or rules selection
- Automatic topic selection
- Automatic removal of Casey approval gates
- Separate channel-specific AI brains
- A second content database
- A second content service
- A replacement Substack writer
- A replacement Substack browser extension
- An OpenAI API content-generation path for V1A content

### 1.3 Open pre-implementation gates

These are intentionally open in this final-review draft:

- **ChatGPT Work transport capability:** not yet proven in Casey’s actual ChatGPT Work environment. No transport is authorized until the spike in Section 5.2 passes.
- **Railway PostgreSQL version:** not yet verified from the actual production database. The repository’s PostgreSQL 15 README statement is not evidence. The version query in Section 6.1 must be run and recorded before migration design is approved.
- **ECC database authority enforcement:** specified below, but not implemented or verified. The ECC-side trigger, deferred mapping constraint, unique exact-version index, and adapter database role are mandatory prerequisites to cutover.

Until all three gates are closed and Casey approves this document, production implementation remains paused.

---

## 2. Non-negotiable invariants

These are implementation requirements, not preferences.

1. ChatGPT Work may read the assigned context and propose content. It may not approve, publish, send, create ECC drafts, create Substack queue items, modify canonical business truth, or invoke delivery adapters.
2. Casey approves an exact immutable output version.
3. Only Freedom Hub adapters may hand an approved version to ECC or Substack.
4. Freedom Hub owns the existing Substack queue and receipt state; the browser extension is only the authenticated transfer client; Substack owns editor/draft state, final publication state, post ID, and post URL. ECC remains authoritative for email operational delivery state. Hub stores references, receipts, and reconciliation state.
5. Attribution identifiers must survive delivery and conversion. If a conversion cannot be legitimately linked, it remains explicitly unattributed.
6. For the V1A 7-Day Clicker workflow, the Hub context snapshot is authoritative for generation. ECC’s existing writer path and Knowledge Layer are reference-only or disabled for that workflow unless specific knowledge is explicitly imported into the Hub snapshot.
7. A failed, expired, or `needs_reconciliation` state fails closed: no email send and no publication.
8. A delivery action must reference the exact approved output version. Approval must never be inferred from the parent output or from a newer unapproved version.
9. All cross-system writes are authenticated, idempotent, auditable, retryable, and reconcilable. The Work authentication mechanism is selected only after the actual ChatGPT Work capability spike passes; this specification must not assume client-side HMAC signing.
10. The first production rollout is a dark launch followed by a manually controlled canary. No scheduler may autonomously create an audience-facing send or publication.

---

## 3. System ownership and actors

### 3.1 Ownership matrix

| System | Authoritative responsibility |
|---|---|
| Freedom Hub | Campaigns, brief revisions, context snapshots, Work jobs, output headers, immutable content versions, evidence manifests, approval events, adapters, the existing `substack_drafts` queue and receipt state, external references, reconciliation state |
| ChatGPT Work | Temporary research, reasoning, drafting, repurposing, and browser assistance. No durable business truth or delivery authority |
| ECC | Contact membership, operational segments, suppression, email drafts, scheduling, sends, delivery, opens, clicks, email operational metrics, email-attributed revenue |
| MemberPress / members site | Product availability, current product pricing, memberships, access, checkout behavior |
| Trade tracker and source systems | Original performance evidence, research data, source material |
| Browser extension | Authenticated transfer client that retrieves an existing Hub-queued draft and transfers it into the authenticated Substack editor. Browser/session state is never authoritative |
| Substack | Editor/draft state after transfer, final publication state, platform post ID, and post URL |
| Hub attribution read model | References and normalized observations imported from ECC and other owners. It is not a competing revenue-attribution source of truth |

### 3.2 Actors

#### Casey / Hub owner session

May:

- Create and edit campaigns and brief revisions
- Create or pause Work jobs
- Read all V1A context and output data
- Approve, reject, or request revision on exact output versions
- Request a delivery handoff for an approved version
- Retry a failed external reference after reconciliation review
- Mark a manual external outcome when the external system cannot provide a callback
- Change review-capacity settings
- Change workflow authority during the documented cutover or rollback procedure

May not bypass evidence validation or deliver an unapproved version through the Hub UI.

#### Work credential

May only:

- Claim a ready Work job
- Read the claimed job’s selected context snapshot
- Heartbeat the claim lease
- Submit a result for the claimed job
- Report a failure for the claimed job

The Work credential may not:

- Approve or reject content
- Create approval events
- Invoke ECC or Substack adapters
- Create ECC drafts
- Create Substack queue items
- Send or publish anything
- Read subscriber rows or raw recipient data
- Change offer, price, audience membership, proof approval, CTA, or other canonical business truth
- Read arbitrary Hub database records outside the claimed job

#### Hub adapter/service credentials

Adapters run only inside the Hub server process or through narrowly scoped service-to-service calls. They may:

- Create an ECC operational draft for an exact approved version
- Create or update an existing Substack queue draft for an exact approved version
- Read external delivery state needed for reconciliation

They may not approve editorial versions or modify canonical business truth.

### 3.3 Actor identifiers

Actor IDs are stable strings stored in audit records, but the stable ID alone is not an authentication claim:

```text
human:casey
work:<key_id>
adapter:ecc
adapter:substack
system:migration
system:reconciliation
```

`human:casey` is resolved server-side from the authenticated Hub owner session and is never accepted as request-supplied text. `work:<key_id>` may identify a Work credential, but every claim also records the distinct Work execution/session identity, claim ID, and lease ID. Secrets are never stored in actor rows or content records.

The actor record is bound to an authenticated identity through `content_actor_bindings`:

```sql
CREATE TABLE content_actor_bindings (
    id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    actor_id              TEXT NOT NULL REFERENCES content_actors(id),
    auth_provider         TEXT NOT NULL,
    authenticated_subject TEXT NOT NULL,
    credential_key_id     TEXT,
    active                BOOLEAN NOT NULL DEFAULT TRUE,
    created_at            TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    revoked_at            TIMESTAMPTZ,
    UNIQUE (auth_provider, authenticated_subject),
    CHECK (revoked_at IS NULL OR revoked_at >= created_at)
);
```

Human mutations derive `actor_id` from the active binding for the authenticated provider/subject. Work and adapter credentials derive the actor and credential key reference from the verified transport, not from a body field.

---

## 4. Existing ECC 7-Day Clicker writer path and V1A cutover

### 4.1 Current path being replaced for the migrated workflow

The existing ECC writing path includes the following routes and services:

1. The ECC `frontend/src/pages/Drafts.js` compose/generation UI.
2. `POST /api/drafts/generate` in `backend/src/routes/drafts.js`.
3. `aiService.generateDraftPair()` in `backend/src/services/ai.js`.
4. The AI service generates a main email and a triggered companion email.
5. The route inserts the triggered draft first and the main broadcast draft second into `email_drafts`, links them, marks them `agent_created=true`, and calls `persistDraftBodyUtms()`.
6. `POST /api/ai-campaigns`, its feedback/revision route, and its approval route in `backend/src/routes/ai-campaigns.js` can also generate and persist AI campaign drafts.
7. `POST /api/campaigns/:id/slots/:slotId/generate` in `backend/src/routes/campaigns.js` can generate slot drafts through the ECC AI service.
8. `jobs/ensure-daily-drafts.js` and `backend/src/services/daily-draft-guard.js` are an additional automated draft path. They are currently disabled and must not be re-enabled as part of V1A.

The operational audience token for the migrated workflow is:

```text
__clicked_last_7__
```

ECC evaluates it as:

```sql
subscriber.last_click_at > NOW() - INTERVAL '7 days'
```

The existing audience execution path also applies ECC’s normal active-contact, test-address, and core-member rules. Freedom Hub must not recreate recipient membership logic.

### 4.2 V1A authoritative workflow

The migrated workflow is exactly:

```text
workflow_key = email_7day_clicker
audience_token = __clicked_last_7__
delivery_system = ecc
authoritative_editorial_writer = freedom_hub
```

The `warm` operational segment is not automatically considered the V1A workflow. Only the reserved `__clicked_last_7__` audience is migrated unless Casey explicitly expands scope in a later specification.

### 4.3 Cutover control

The ECC application must have a server-side authority check for all routes that can create or AI-generate an `email_drafts` row for `__clicked_last_7__`.

When Hub authority is active, those routes must return:

```http
409 Conflict
```

```json
{
  "error": {
    "code": "CONTENT_ENGINE_AUTHORITATIVE",
    "message": "The 7-Day Clicker workflow is authored in Freedom Hub.",
    "workflow_key": "email_7day_clicker"
  }
}
```

The check must be applied before any AI call and before any draft insert or update. It must cover `/api/drafts/generate`, direct/manual `POST /api/drafts` creation, draft edit/update paths that can change `segment_target`, AI campaign approval/generation, campaign slot generation, and any future route that writes a draft for the reserved audience. The guard evaluates the normalized persisted audience target, not only a frontend selector.

Non-7-Day ECC workflows remain unchanged.

### 4.4 Existing ECC Knowledge Layer treatment

ECC’s existing brand voice, writer rules, transcripts, tracker facts, member/support themes, testimonials, and offer context are not automatically copied into the Hub Brain.

For V1A:

- The Hub brief explicitly selects the sources and rules needed for the job.
- Those selected items are copied or referenced into the immutable Hub context snapshot.
- ECC Knowledge Layer content is reference-only unless a specific item is imported into that snapshot.
- ECC’s AI writer must not independently regenerate or revise the migrated 7-Day Clicker copy.
- Existing ECC Knowledge Layer ingestion jobs may continue for non-migrated workflows.
- The V1A implementation must record which ECC item, if any, was imported into the snapshot.

### 4.5 Cutover procedure

1. Confirm ECC production API authentication and adapter credentials.
2. Apply Hub and ECC schema migrations without enabling generation or delivery.
3. Inventory existing ECC rows targeting `__clicked_last_7__`.
4. Resolve or explicitly quarantine existing unsent drafts, approved drafts, scheduled drafts, and in-flight campaigns. Do not delete them automatically.
5. Confirm no scheduled Hub-created or conflicting ECC-created 7-Day Clicker send is in flight.
6. Run a Hub-only shadow job. It may create Hub versions, but no ECC draft or Substack queue item.
7. Run the internal canary described in Section 22.3.
8. Verify the ECC-side authority mirror, database trigger, deferred mapping constraint, unique exact-version index, and adapter database role are active.
9. Set `email_7day_clicker` authority to `hub` and enable the ECC writer guard.
10. Create one V1A Work job in one-package-at-a-time mode.
11. Casey reviews and approves the exact versions.
12. Hub adapters create the ECC draft and existing Hub Substack queue draft.
13. Casey manually reviews the operational records, sends the email, and publishes the Substack article.

### 4.6 Rollback authority

Rollback authority is Casey only.

Rollback sequence:

1. Pause new Hub Work jobs and all Hub external handoffs.
2. Leave existing Hub and external records intact.
3. Mark any in-flight Hub external references `needs_reconciliation` unless the external system confirms a safe terminal state.
4. Confirm no Hub-created ECC draft is approved or scheduled for sending.
5. Set the ECC-side authority mirror and Hub authority back to `ecc` in a verified order.
6. Re-enable the existing ECC writer paths only after both authority records and the database guard confirm ECC authority.
7. Leave Hub-created drafts in review/quarantine; do not silently convert them into ECC-authoritative copy.
8. Resolve duplicate-risk records manually before any subsequent send.

Already sent emails and already published Substack articles are not rolled back automatically.

---

## 5. Work transport and security contract

### 5.1 V1A transport decision is conditional on a capability spike

The Work transport is intentionally **not selected yet**. Before any V1A implementation, Casey’s actual ChatGPT Work environment must prove one of these supported transports:

1. **Preferred:** a securely configured Freedom Hub action/plugin using an authentication mechanism the actual Work surface supports, such as an API-key/Bearer credential or supported OAuth connection. Work must never receive a reusable Hub database credential or a client-side signing secret. Hub authenticates the caller, resolves the Work actor, and generates the short-lived claim/lease token server-side after successful authentication.
2. **Fallback:** the already-proven private GitHub inbox pattern from `agent/substack_inbox.py`. GitHub is transport only; Freedom Hub remains the durable job/orchestration source of truth. Work uses its supported GitHub connection to submit structured claim/heartbeat/result/failure envelopes to a private inbox. Hub polls and processes those envelopes with advisory-locked receipts, payload hashes, authorized-identity checks, and idempotency.

Client-side HMAC signing is removed from V1A. Do not implement an HMAC scheme that assumes ChatGPT Work can receive a secret and calculate arbitrary request signatures.

Work never receives the ECC draft-write credential, Substack action credential, Hub owner session token, database credential, browser-extension credential, or any external-platform credential.

### 5.2 Required transport-capability spike

The spike is a mandatory pre-implementation gate and must run through Casey’s actual ChatGPT Work environment, not through an Adaptive simulation or a developer-only HTTP client. It must use an isolated/non-production Hub test job and must not touch ECC, Substack, production content, production drafts, production sends, or production publications.

The spike must prove, using the candidate transport:

1. Work authenticates to Freedom Hub using the actual supported connection mechanism.
2. Work retrieves one harmless test job containing no business content or recipient data.
3. Hub atomically claims that test job and returns a server-generated short-lived claim/lease token.
4. Work submits one harmless test result tied to the claim/session identity.
5. Work repeats the same result request or re-reads the result after a simulated response loss and receives/reuses the original receipt without creating a duplicate submission, output version, or job completion side effect.

The test fixture must be explicitly marked `transport_spike=true`, must be isolated from production content, and must use a result path that records the receipt but does not create editorial output versions or invoke any adapter. The fixture is deleted or expires through the test harness after evidence capture; no production content table is used for the proof unless the database is a separately isolated test database.

Required evidence:

```text
transport_candidate = hub_action_bearer | hub_action_oauth | github_inbox
work_environment_identifier = <non-secret-safe-label>
authenticated_provider = <provider>
authenticated_subject = <provider subject or safe account label>
work_session_id = <execution/session identifier>
claim_id = <uuid>
lease_id = <uuid>
test_job_id = <uuid>
request_ids = [<claim>, <result>, <replay>]
receipt_id = <id>
duplicate_output_versions_created = 0
ecc_or_substack_side_effects = 0
result = pass | fail
```

The spike is **not yet proven in this specification** because the actual ChatGPT Work environment is not accessible from the current Adaptive session. Production implementation remains blocked until Casey runs the spike and records the evidence above.

### 5.3 Supported action/plugin transport

If the spike proves a Hub action/plugin works in the actual Work environment:

- The action/plugin is the only primary Work transport for V1A.
- Authentication is handled by the supported API-key/Bearer or OAuth mechanism; the mechanism is documented with its provider, scopes, credential owner, and rotation procedure.
- Hub maps the authenticated provider subject to a `content_actor_bindings` row and rejects any request-supplied actor identity.
- Hub generates the claim token and lease ID server-side after the authenticated claim transaction succeeds.
- The transport supplies a Work execution/session identifier, but Hub records the authenticated subject and credential reference from the transport rather than trusting the body value.
- The action/plugin cannot call ECC or Substack adapters and cannot approve, send, or publish.

### 5.4 GitHub inbox fallback transport

If the action/plugin capability is unavailable or fails the spike, V1A uses the existing private GitHub inbox pattern as transport only:

- Work submits structured JSON envelopes to a dedicated private GitHub Issues inbox using its supported GitHub connection.
- Hub verifies the repository is private, verifies the external issue/comment author against the approved Work identity, rejects pull requests and unrecognized issue/comment shapes, and treats all body text as data rather than instructions.
- Claim, heartbeat, result, and failure operations carry request IDs and Work session identifiers in the structured envelope. The authenticated GitHub subject and external issue/comment IDs are captured by Hub and are not accepted from request text as proof of identity.
- Hub polls the inbox, processes each envelope inside a transaction, and records a durable transport receipt keyed by transport plus external locator and request ID plus payload hash.
- Hub posts the response receipt back to the issue/comment thread. Re-reading or replaying the same envelope returns the original receipt and cannot claim a second job or create a duplicate result.
- GitHub issues/comments are closed or marked complete only after the Hub receipt is durably committed. GitHub is never the job, approval, content, or delivery source of truth.
- The implementation reuses the security and idempotency lessons from `agent/substack_inbox.py`; it does not create a second content database or a second orchestration system.

### 5.5 Server-generated claim and lease tokens

Regardless of the selected transport, Hub generates an opaque short-lived claim token and distinct `claim_id`/`lease_id` values inside the atomic claim transaction. The token is returned only through the authenticated transport response, stored only as a hash, and is never derived from or signed by a Work-held secret. A replayed transport request cannot create a second active claim.

### 5.6 Work scopes

The Work transport credential/connection is restricted to:

```text
content:work:claim
content:work:read_assigned
content:work:heartbeat
content:work:submit
content:work:fail
```

It must not possess:

```text
content:approve
content:reject
content:deliver
content:send
content:publish
content:business_truth:write
content:subscriber:read
content:ecc:write
content:substack:write
```

The server must enforce scopes at the route layer and in the service layer. A hidden UI control is not an authorization boundary.

### 5.7 Maximum payload sizes

| Payload | Maximum |
|---|---:|
| Raw HTTP request body | 2 MiB |
| Context snapshot JSON | 1 MiB |
| One source snapshot in a context package | 256 KiB |
| One output body | 300 KiB |
| Complete result package | 1 MiB |
| Title | 250 UTF-8 characters |
| Subtitle | 500 UTF-8 characters |
| Subject line | 200 UTF-8 characters |
| Change summary | 2,000 UTF-8 characters |
| Safe failure message | 1,000 UTF-8 characters |

Oversized requests fail before database writes with `PAYLOAD_TOO_LARGE`.

### 5.8 Work must not receive raw recipient data

Work receives audience definition and aggregate context only:

```text
audience_token = __clicked_last_7__
audience_definition_version = <uuid/version>
aggregate_count = optional
```

No subscriber email, name, recipient row, IP, raw click log, or raw revenue customer record is included in the Work context.

---

## 6. PostgreSQL conventions

### 6.1 Database assumptions

- Minimum supported PostgreSQL version: 13. The actual Railway production major/minor version is an explicit pre-implementation gate and must not be inferred from repository documentation.
- Required pre-implementation verification query:

```sql
SELECT version(),
       current_setting('server_version_num') AS server_version_num,
       current_setting('server_version') AS server_version;

SELECT extname, extversion
FROM pg_extension
WHERE extname = 'pgcrypto';
```

- The verified Railway production version and `pgcrypto` extension version must be recorded in the implementation record before migrations are designed, reviewed, or approved. If the verified major is below 13 or `pgcrypto` is unavailable, implementation is blocked.
- Existing Freedom Hub Railway PostgreSQL service
- `pgcrypto` extension for UUID generation and cryptographic digests
- UTC storage using `timestamptz`
- America/New_York only for operating/reporting display and date-boundary logic
- UUID primary keys for content entities
- `jsonb` only for versioned snapshots, manifests, typed metadata, and external payloads
- No secrets in database rows
- No hard deletes for editorial history, approvals, submissions, or audit events

### 6.2 Migration ledger

Use one shared migration runner and one ledger table:

```sql
CREATE TABLE IF NOT EXISTS schema_migrations (
    version       TEXT PRIMARY KEY,
    checksum      TEXT NOT NULL,
    applied_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    applied_by    TEXT NOT NULL,
    duration_ms   INTEGER NOT NULL DEFAULT 0
);
```

The runner must acquire a single advisory lock before reading or applying migrations. Each migration runs in its own transaction. A failed migration records no applied row and blocks application startup in production.

No destructive down migrations are required. Rollback means a forward-fix migration or restoration to a verified backup according to Section 24.

### 6.3 Actor registry

```sql
CREATE TABLE content_actors (
    id              TEXT PRIMARY KEY,
    actor_type      TEXT NOT NULL CHECK (actor_type IN ('human','work','adapter','system')),
    display_name    TEXT NOT NULL,
    active          BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_content_actors_type_active
    ON content_actors(actor_type, active);
```

Seed actors are `human:casey`, `adapter:ecc`, `adapter:substack`, `system:migration`, and `system:reconciliation`. Work credential key references may be inserted or activated during connection rotation but contain no secret.

### 6.4 Channel definitions

V1A seeds only two channels.

```sql
CREATE TABLE content_channels (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    slug            TEXT NOT NULL UNIQUE CHECK (slug ~ '^[a-z0-9_]+$'),
    name            TEXT NOT NULL,
    delivery_system TEXT NOT NULL CHECK (delivery_system IN ('substack','ecc')),
    active          BOOLEAN NOT NULL DEFAULT TRUE,
    rules           JSONB NOT NULL DEFAULT '{}'::jsonb,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

Seed rows:

```text
substack_article  → Substack → substack
email_7day_clicker → 7-Day Clicker Email → ecc
```

Channel slugs are globally unique for all time. Inactive historical rows retain their slug permanently; V1A never reuses an inactive channel slug. The `UNIQUE` constraint on `slug` is the only channel-uniqueness rule; there is no redundant partial active-slug index.

No additional channel rows are needed for V1A.

### 6.5 Runtime settings and backpressure

```sql
CREATE TABLE content_runtime_settings (
    singleton_id             SMALLINT PRIMARY KEY CHECK (singleton_id = 1),
    max_open_review_packages INTEGER NOT NULL DEFAULT 1 CHECK (max_open_review_packages >= 1),
    one_package_at_a_time    BOOLEAN NOT NULL DEFAULT TRUE,
    generation_paused        BOOLEAN NOT NULL DEFAULT FALSE,
    pause_reason             TEXT,
    updated_by_actor_id      TEXT NOT NULL REFERENCES content_actors(id),
    updated_at               TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

Initial V1A values are `max_open_review_packages = 1` and `one_package_at_a_time = true`.

### 6.6 Campaigns

```sql
CREATE TABLE content_campaigns (
    id                           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    slug                         TEXT NOT NULL UNIQUE CHECK (slug ~ '^[a-z0-9-]+$'),
    name                         TEXT NOT NULL CHECK (length(btrim(name)) > 0),
    objective                    TEXT NOT NULL,
    status                       TEXT NOT NULL CHECK (status IN ('draft','active','paused','completed','archived')),
    start_at                     TIMESTAMPTZ,
    end_at                       TIMESTAMPTZ,
    primary_audience_snapshot_id UUID,
    primary_offer_snapshot_id    UUID,
    primary_cta_version_id       UUID,
    target_metrics               JSONB NOT NULL DEFAULT '{}'::jsonb,
    notes                        TEXT NOT NULL DEFAULT '',
    created_by_actor_id          TEXT NOT NULL REFERENCES content_actors(id),
    updated_by_actor_id          TEXT NOT NULL REFERENCES content_actors(id),
    created_at                   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at                   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CHECK (end_at IS NULL OR start_at IS NULL OR end_at > start_at)
);

CREATE INDEX idx_content_campaigns_status_dates
    ON content_campaigns(status, start_at, end_at);
```

The three nullable snapshot foreign keys are added after their referenced tables are created in migration `V1A-003`.

### 6.7 Business reference snapshots

Hub does not become the owner of these facts. These tables are immutable or append-only snapshots of owner-system data.

#### Audience-definition snapshots

```sql
CREATE TABLE content_audience_snapshots (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    audience_key      TEXT NOT NULL,
    version_number    INTEGER NOT NULL CHECK (version_number >= 1),
    audience_kind     TEXT NOT NULL CHECK (audience_kind IN ('strategic','operational')),
    name              TEXT NOT NULL,
    source_system     TEXT NOT NULL,
    external_key      TEXT NOT NULL,
    criteria          JSONB NOT NULL DEFAULT '{}'::jsonb,
    aggregate_count   INTEGER CHECK (aggregate_count IS NULL OR aggregate_count >= 0),
    source_updated_at TIMESTAMPTZ,
    valid_from        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    valid_through     TIMESTAMPTZ,
    snapshot_hash     CHAR(64) NOT NULL CHECK (snapshot_hash ~ '^[0-9a-f]{64}$'),
    created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (audience_key, version_number),
    CHECK (valid_through IS NULL OR valid_through > valid_from)
);

CREATE INDEX idx_content_audience_snapshots_external
    ON content_audience_snapshots(source_system, external_key, valid_from DESC);
```

The V1A operational row must reference ECC’s `__clicked_last_7__` token and its exact definition version. It must not contain recipient membership rows.

#### Offer snapshots

```sql
CREATE TABLE content_offer_snapshots (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    offer_key         TEXT NOT NULL,
    version_number    INTEGER NOT NULL CHECK (version_number >= 1),
    source_system     TEXT NOT NULL,
    external_product_id TEXT NOT NULL,
    name              TEXT NOT NULL,
    status            TEXT NOT NULL CHECK (status IN ('active','inactive','historical','unknown')),
    price             NUMERIC(12,2),
    currency          CHAR(3) NOT NULL DEFAULT 'USD',
    recurring_interval TEXT,
    canonical_url     TEXT NOT NULL CHECK (canonical_url ~ '^https://'),
    core_promise      TEXT NOT NULL DEFAULT '',
    benefits          JSONB NOT NULL DEFAULT '[]'::jsonb,
    inclusions        JSONB NOT NULL DEFAULT '[]'::jsonb,
    restrictions     JSONB NOT NULL DEFAULT '[]'::jsonb,
    source_updated_at TIMESTAMPTZ,
    valid_from        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    valid_through     TIMESTAMPTZ,
    snapshot_hash     CHAR(64) NOT NULL CHECK (snapshot_hash ~ '^[0-9a-f]{64}$'),
    created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (offer_key, version_number),
    CHECK (valid_through IS NULL OR valid_through > valid_from),
    CHECK (price IS NULL OR price >= 0)
);

CREATE INDEX idx_content_offer_snapshots_external
    ON content_offer_snapshots(source_system, external_product_id, valid_from DESC);
```

#### CTA versions

```sql
CREATE TABLE content_cta_versions (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cta_key           TEXT NOT NULL,
    version_number    INTEGER NOT NULL CHECK (version_number >= 1),
    offer_snapshot_id UUID REFERENCES content_offer_snapshots(id),
    name              TEXT NOT NULL,
    copy              TEXT NOT NULL,
    destination_url   TEXT NOT NULL CHECK (destination_url ~ '^https://'),
    utm_template      JSONB NOT NULL DEFAULT '{}'::jsonb,
    preferred_channels JSONB NOT NULL DEFAULT '[]'::jsonb,
    status            TEXT NOT NULL CHECK (status IN ('active','inactive','historical')),
    valid_from        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    valid_through     TIMESTAMPTZ,
    created_by_actor_id TEXT NOT NULL REFERENCES content_actors(id),
    created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (cta_key, version_number),
    CHECK (valid_through IS NULL OR valid_through > valid_from)
);

CREATE INDEX idx_content_cta_versions_active
    ON content_cta_versions(cta_key, status, valid_from DESC);

ALTER TABLE content_campaigns
    ADD CONSTRAINT fk_content_campaigns_audience_snapshot
    FOREIGN KEY (primary_audience_snapshot_id)
    REFERENCES content_audience_snapshots(id),
    ADD CONSTRAINT fk_content_campaigns_offer_snapshot
    FOREIGN KEY (primary_offer_snapshot_id)
    REFERENCES content_offer_snapshots(id),
    ADD CONSTRAINT fk_content_campaigns_cta_version
    FOREIGN KEY (primary_cta_version_id)
    REFERENCES content_cta_versions(id);
```

#### Proof records

```sql
CREATE TABLE content_proof_records (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    proof_key         TEXT NOT NULL,
    version_number    INTEGER NOT NULL CHECK (version_number >= 1),
    proof_type        TEXT NOT NULL,
    title             TEXT NOT NULL,
    statement         TEXT NOT NULL,
    supporting_data   JSONB NOT NULL DEFAULT '{}'::jsonb,
    source_system     TEXT NOT NULL,
    source_reference  TEXT NOT NULL,
    source_url        TEXT CHECK (source_url IS NULL OR source_url ~ '^https://'),
    approval_status   TEXT NOT NULL CHECK (approval_status IN ('pending','approved','revoked','expired')),
    approved_by_actor_id TEXT REFERENCES content_actors(id),
    approved_at       TIMESTAMPTZ,
    valid_from        TIMESTAMPTZ,
    valid_through     TIMESTAMPTZ,
    approved_channels JSONB NOT NULL DEFAULT '[]'::jsonb,
    restrictions      JSONB NOT NULL DEFAULT '[]'::jsonb,
    verified_at       TIMESTAMPTZ,
    content_hash      CHAR(64) NOT NULL CHECK (content_hash ~ '^[0-9a-f]{64}$'),
    created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (proof_key, version_number),
    CHECK (
        (approval_status = 'approved' AND approved_at IS NOT NULL AND approved_by_actor_id IS NOT NULL)
        OR (approval_status = 'pending' AND approved_at IS NULL AND approved_by_actor_id IS NULL)
        OR (approval_status IN ('revoked','expired') AND approved_at IS NOT NULL AND approved_by_actor_id IS NOT NULL)
    ),
    CHECK (valid_through IS NULL OR valid_from IS NULL OR valid_through > valid_from)
);

CREATE INDEX idx_content_proof_eligible
    ON content_proof_records(approval_status, proof_type, valid_from, valid_through);
```

Proof records are append-only. A revoked or expired proof retains its original approval metadata; a status transition is represented by a new proof version rather than mutating the prior approved row. A proof that was never approved may remain `pending` or be represented by a new non-approved version.

#### Source snapshots

```sql
CREATE TABLE content_source_snapshots (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_type       TEXT NOT NULL,
    title             TEXT NOT NULL,
    source_system     TEXT NOT NULL,
    source_reference  TEXT NOT NULL,
    source_url        TEXT CHECK (source_url IS NULL OR source_url ~ '^https://'),
    content_text      TEXT,
    metadata          JSONB NOT NULL DEFAULT '{}'::jsonb,
    source_date       TIMESTAMPTZ,
    retrieved_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    verified_at       TIMESTAMPTZ,
    content_hash      CHAR(64) NOT NULL CHECK (content_hash ~ '^[0-9a-f]{64}$'),
    is_untrusted_data  BOOLEAN NOT NULL DEFAULT TRUE,
    created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (source_system, source_reference, content_hash)
);

CREATE INDEX idx_content_source_snapshots_lookup
    ON content_source_snapshots(source_system, source_type, source_date DESC);
```

Source content is data, not instructions. Work prompts must label it as untrusted source material.

#### Rule versions

```sql
CREATE TABLE content_rule_versions (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    rule_key          TEXT NOT NULL,
    version_number    INTEGER NOT NULL CHECK (version_number >= 1),
    scope_type        TEXT NOT NULL CHECK (scope_type IN ('global','channel','audience','offer')),
    scope_id          TEXT,
    importance        TEXT NOT NULL CHECK (importance IN ('required','preferred')),
    priority          INTEGER NOT NULL DEFAULT 100,
    rule_text         TEXT NOT NULL,
    active            BOOLEAN NOT NULL DEFAULT TRUE,
    valid_from        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    valid_through     TIMESTAMPTZ,
    created_by_actor_id TEXT NOT NULL REFERENCES content_actors(id),
    created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (rule_key, version_number),
    CHECK (valid_through IS NULL OR valid_through > valid_from)
);

CREATE INDEX idx_content_rules_resolution
    ON content_rule_versions(scope_type, scope_id, active, importance, priority);
```

Rules are selected deterministically by scope, active interval, importance, and ascending priority. V1A does not build an AI rules engine.

### 6.8 Campaigns, briefs, revisions, and selections

```sql
CREATE TABLE content_briefs (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    campaign_id       UUID NOT NULL REFERENCES content_campaigns(id),
    internal_name     TEXT NOT NULL CHECK (length(btrim(internal_name)) > 0),
    status            TEXT NOT NULL CHECK (status IN ('idea','brief_ready','in_work','review','approved','complete','archived')),
    current_revision_id UUID,
    created_by_actor_id TEXT NOT NULL REFERENCES content_actors(id),
    updated_by_actor_id TEXT NOT NULL REFERENCES content_actors(id),
    created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at        TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_content_briefs_pipeline
    ON content_briefs(status, updated_at DESC);

CREATE TABLE content_brief_revisions (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    brief_id          UUID NOT NULL REFERENCES content_briefs(id),
    revision_number   INTEGER NOT NULL CHECK (revision_number >= 1),
    core_idea         TEXT NOT NULL,
    promise           TEXT NOT NULL,
    angle             TEXT NOT NULL,
    objective         TEXT NOT NULL,
    primary_audience_snapshot_id UUID NOT NULL REFERENCES content_audience_snapshots(id),
    primary_offer_snapshot_id    UUID REFERENCES content_offer_snapshots(id),
    primary_cta_version_id       UUID REFERENCES content_cta_versions(id),
    constraints       JSONB NOT NULL DEFAULT '{}'::jsonb,
    priority          INTEGER NOT NULL DEFAULT 100,
    due_at            TIMESTAMPTZ,
    created_by_actor_id TEXT NOT NULL REFERENCES content_actors(id),
    created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (brief_id, revision_number)
);

ALTER TABLE content_brief_revisions
    ADD CONSTRAINT uq_content_brief_revisions_id_brief
    UNIQUE (id, brief_id);

ALTER TABLE content_briefs
    ADD CONSTRAINT fk_content_briefs_current_revision
    FOREIGN KEY (current_revision_id, id)
    REFERENCES content_brief_revisions(id, brief_id);

CREATE INDEX idx_content_brief_revisions_brief
    ON content_brief_revisions(brief_id, revision_number DESC);

CREATE TABLE content_brief_requested_outputs (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    brief_revision_id UUID NOT NULL REFERENCES content_brief_revisions(id),
    channel_id        UUID NOT NULL REFERENCES content_channels(id),
    item_index        INTEGER NOT NULL CHECK (item_index >= 0),
    variant_key       TEXT NOT NULL DEFAULT 'default',
    output_spec       JSONB NOT NULL DEFAULT '{}'::jsonb,
    created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (brief_revision_id, channel_id, item_index, variant_key)
);

CREATE TABLE content_brief_source_refs (
    brief_revision_id UUID NOT NULL REFERENCES content_brief_revisions(id) ON DELETE CASCADE,
    source_snapshot_id UUID NOT NULL REFERENCES content_source_snapshots(id),
    position_index    INTEGER NOT NULL DEFAULT 0 CHECK (position_index >= 0),
    notes             TEXT NOT NULL DEFAULT '',
    PRIMARY KEY (brief_revision_id, source_snapshot_id)
);

CREATE TABLE content_brief_proof_refs (
    brief_revision_id UUID NOT NULL REFERENCES content_brief_revisions(id) ON DELETE CASCADE,
    proof_record_id   UUID NOT NULL REFERENCES content_proof_records(id),
    position_index    INTEGER NOT NULL DEFAULT 0 CHECK (position_index >= 0),
    notes             TEXT NOT NULL DEFAULT '',
    PRIMARY KEY (brief_revision_id, proof_record_id)
);

CREATE INDEX idx_content_brief_requested_outputs
    ON content_brief_requested_outputs(brief_revision_id, channel_id, item_index);
```

### 6.9 Context snapshots

```sql
CREATE TABLE content_context_snapshots (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    brief_id          UUID NOT NULL REFERENCES content_briefs(id),
    brief_revision_id UUID NOT NULL,
    snapshot_payload  JSONB NOT NULL,
    context_hash      CHAR(64) NOT NULL CHECK (context_hash ~ '^[0-9a-f]{64}$'),
    generated_at      TIMESTAMPTZ NOT NULL,
    created_by_actor_id TEXT NOT NULL REFERENCES content_actors(id),
    created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (brief_revision_id, context_hash)
);

ALTER TABLE content_context_snapshots
    ADD CONSTRAINT uq_content_context_snapshots_identity
    UNIQUE (id, brief_id, brief_revision_id),
    ADD CONSTRAINT fk_context_snapshot_revision_belongs_to_brief
    FOREIGN KEY (brief_revision_id, brief_id)
    REFERENCES content_brief_revisions(id, brief_id);

CREATE INDEX idx_content_context_snapshots_brief
    ON content_context_snapshots(brief_revision_id, created_at DESC);
```

The snapshot payload must include:

- Brief ID and revision
- Campaign ID
- Requested output definitions
- Offer snapshot IDs and values used
- Audience-definition snapshot IDs and values used
- CTA version IDs and canonical destinations
- Selected proof IDs and verification dates
- Selected source snapshot IDs and content hashes
- Applicable rule IDs and versions
- Channel-specific constraints
- Generation timestamp
- Operating timezone
- Context hash

### 6.10 Work jobs, transport receipts, attempts, and submissions

```sql
CREATE TABLE content_work_transport_receipts (
    id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    transport             TEXT NOT NULL CHECK (transport IN ('hub_action_bearer','hub_action_oauth','github_inbox')),
    operation             TEXT NOT NULL CHECK (operation IN ('claim','heartbeat','result','fail')),
    external_locator      TEXT NOT NULL,
    transport_request_id  UUID NOT NULL,
    payload_hash          CHAR(64) NOT NULL CHECK (payload_hash ~ '^[0-9a-f]{64}$'),
    authenticated_subject TEXT NOT NULL,
    credential_key_id     TEXT,
    work_session_id       TEXT NOT NULL,
    response_receipt      JSONB NOT NULL,
    created_at            TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (transport, transport_request_id),
    UNIQUE (transport, external_locator, operation)
);

CREATE INDEX idx_content_work_transport_receipts_locator
    ON content_work_transport_receipts(transport, external_locator, created_at DESC);

CREATE TABLE content_work_jobs (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    request_id        UUID NOT NULL UNIQUE,
    schema_version    INTEGER NOT NULL CHECK (schema_version = 1),
    brief_id          UUID NOT NULL REFERENCES content_briefs(id),
    brief_revision_id UUID NOT NULL,
    job_type          TEXT NOT NULL CHECK (job_type IN ('generate_content_package')),
    transport_spike   BOOLEAN NOT NULL DEFAULT FALSE,
    requested_outputs JSONB NOT NULL,
    context_snapshot_id UUID NOT NULL,
    context_hash      CHAR(64) NOT NULL CHECK (context_hash ~ '^[0-9a-f]{64}$'),
    status            TEXT NOT NULL CHECK (status IN ('ready','claimed','working','returned','failed','expired','needs_reconciliation','cancelled')),
    attempt_count     INTEGER NOT NULL DEFAULT 0 CHECK (attempt_count >= 0),
    max_attempts      INTEGER NOT NULL DEFAULT 2 CHECK (max_attempts BETWEEN 1 AND 5),
    claim_token_hash  CHAR(64),
    claimed_by_actor_id TEXT REFERENCES content_actors(id),
    authenticated_subject TEXT,
    credential_key_id  TEXT,
    work_session_id    TEXT,
    claim_id           UUID,
    lease_id           UUID,
    claimed_at        TIMESTAMPTZ,
    lease_expires_at  TIMESTAMPTZ,
    heartbeat_at      TIMESTAMPTZ,
    created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    expires_at        TIMESTAMPTZ NOT NULL,
    completed_at      TIMESTAMPTZ,
    failure_code      TEXT,
    safe_failure_message TEXT,
    result_receipt    JSONB,
    result_payload_hash CHAR(64) CHECK (result_payload_hash IS NULL OR result_payload_hash ~ '^[0-9a-f]{64}$'),
    result_received_at TIMESTAMPTZ,
    CHECK ((status IN ('claimed','working') AND claim_token_hash IS NOT NULL) OR status NOT IN ('claimed','working')),
    CHECK (lease_expires_at IS NULL OR claimed_at IS NOT NULL),
    CHECK (expires_at > created_at)
);

ALTER TABLE content_work_jobs
    ADD CONSTRAINT uq_content_work_jobs_id_request
    UNIQUE (id, request_id);

ALTER TABLE content_work_jobs
    ADD CONSTRAINT fk_work_job_revision_belongs_to_brief
    FOREIGN KEY (brief_revision_id, brief_id)
    REFERENCES content_brief_revisions(id, brief_id),
    ADD CONSTRAINT fk_work_job_context_belongs_to_brief_revision
    FOREIGN KEY (context_snapshot_id, brief_id, brief_revision_id)
    REFERENCES content_context_snapshots(id, brief_id, brief_revision_id);

CREATE INDEX idx_content_work_jobs_claimable
    ON content_work_jobs(status, created_at)
    WHERE status IN ('ready','claimed','working');

CREATE INDEX idx_content_work_jobs_expired_leases
    ON content_work_jobs(lease_expires_at)
    WHERE status IN ('claimed','working');

CREATE INDEX idx_content_work_jobs_brief
    ON content_work_jobs(brief_revision_id, created_at DESC);
```

```sql
CREATE TABLE content_work_job_attempts (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    work_job_id       UUID NOT NULL REFERENCES content_work_jobs(id) ON DELETE CASCADE,
    attempt_number    INTEGER NOT NULL CHECK (attempt_number >= 1),
    credential_key_id TEXT,
    authenticated_subject TEXT NOT NULL,
    work_session_id   TEXT NOT NULL,
    worker_instance_id TEXT NOT NULL,
    claim_id          UUID NOT NULL UNIQUE,
    lease_id          UUID NOT NULL,
    claim_token_hash  CHAR(64) NOT NULL,
    started_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    last_heartbeat_at TIMESTAMPTZ,
    finished_at       TIMESTAMPTZ,
    outcome           TEXT CHECK (outcome IN ('working','returned','failed','lease_expired','late','rejected')),
    failure_code      TEXT,
    result_payload_hash CHAR(64),
    UNIQUE (work_job_id, attempt_number)
);

CREATE INDEX idx_content_work_attempts_job
    ON content_work_job_attempts(work_job_id, attempt_number DESC);

CREATE TABLE content_work_submissions (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    work_job_id       UUID NOT NULL,
    request_id        UUID NOT NULL,
    attempt_number    INTEGER NOT NULL,
    payload_hash      CHAR(64) NOT NULL CHECK (payload_hash ~ '^[0-9a-f]{64}$'),
    payload           JSONB,
    outcome           TEXT NOT NULL CHECK (outcome IN ('accepted','duplicate','rejected_late','rejected_invalid','rejected_mismatch','quarantined')),
    received_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    safe_message      TEXT NOT NULL DEFAULT '',
    UNIQUE (work_job_id, payload_hash)
);

ALTER TABLE content_work_submissions
    ADD CONSTRAINT fk_work_submission_job_request
    FOREIGN KEY (work_job_id, request_id)
    REFERENCES content_work_jobs(id, request_id),
    ADD CONSTRAINT fk_work_submission_attempt
    FOREIGN KEY (work_job_id, attempt_number)
    REFERENCES content_work_job_attempts(work_job_id, attempt_number);

CREATE INDEX idx_content_work_submissions_request
    ON content_work_submissions(request_id, received_at DESC);
```

The transport receipt is the idempotency boundary for the selected Work transport. `content_work_jobs.request_id` identifies the durable job request; `content_work_transport_receipts.transport_request_id` identifies each transport operation. A reused transport request ID with a different payload hash is rejected; an exact replay returns the original receipt without a second claim, submission, version, or side effect.

### 6.11 Editorial outputs, versions, evidence, and approvals

```sql
CREATE TABLE content_outputs (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    brief_revision_id UUID NOT NULL,
    channel_id        UUID NOT NULL REFERENCES content_channels(id),
    item_index        INTEGER NOT NULL CHECK (item_index >= 0),
    variant_key       TEXT NOT NULL DEFAULT 'default',
    editorial_status  TEXT NOT NULL CHECK (editorial_status IN ('requested','drafting','review','approved','archived')),
    current_version_id UUID,
    approved_version_id UUID,
    created_by_actor_id TEXT NOT NULL REFERENCES content_actors(id),
    updated_by_actor_id TEXT NOT NULL REFERENCES content_actors(id),
    created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (brief_revision_id, channel_id, item_index, variant_key)
);

ALTER TABLE content_outputs
    ADD CONSTRAINT fk_content_outputs_revision
    FOREIGN KEY (brief_revision_id)
    REFERENCES content_brief_revisions(id);

CREATE INDEX idx_content_outputs_review
    ON content_outputs(editorial_status, updated_at DESC)
    WHERE editorial_status IN ('review','approved');

CREATE TABLE content_output_versions (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    output_id         UUID NOT NULL REFERENCES content_outputs(id),
    version_number    INTEGER NOT NULL CHECK (version_number >= 1),
    title             TEXT NOT NULL,
    subtitle          TEXT,
    subject_line      TEXT,
    body              TEXT NOT NULL,
    content_format    TEXT NOT NULL CHECK (content_format IN ('markdown','html','plain_text')),
    change_summary    TEXT NOT NULL DEFAULT '',
    evidence_manifest JSONB NOT NULL DEFAULT '[]'::jsonb,
    context_snapshot_id UUID NOT NULL REFERENCES content_context_snapshots(id),
    content_hash      CHAR(64) NOT NULL CHECK (content_hash ~ '^[0-9a-f]{64}$'),
    created_by_actor_id TEXT NOT NULL REFERENCES content_actors(id),
    created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (output_id, version_number),
    UNIQUE (output_id, content_hash),
    UNIQUE (id, output_id)
);

CREATE INDEX idx_content_output_versions_output
    ON content_output_versions(output_id, version_number DESC);

ALTER TABLE content_outputs
    ADD CONSTRAINT fk_content_outputs_current_version
    FOREIGN KEY (current_version_id, id) REFERENCES content_output_versions(id, output_id),
    ADD CONSTRAINT fk_content_outputs_approved_version
    FOREIGN KEY (approved_version_id, id) REFERENCES content_output_versions(id, output_id);

CREATE TABLE content_evidence_entries (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    output_version_id UUID NOT NULL REFERENCES content_output_versions(id) ON DELETE CASCADE,
    claim_hash        CHAR(64) NOT NULL CHECK (claim_hash ~ '^[0-9a-f]{64}$'),
    claim_text        TEXT NOT NULL,
    claim_type        TEXT NOT NULL CHECK (claim_type IN ('marketing_claim','trading_performance','testimonial','business_result','research_fact')),
    evidence_type     TEXT NOT NULL CHECK (evidence_type IN ('approved_proof','source_snapshot')),
    evidence_id       UUID NOT NULL,
    verified_at       TIMESTAMPTZ,
    notes             TEXT NOT NULL DEFAULT '',
    UNIQUE (output_version_id, claim_hash, evidence_id)
);

CREATE INDEX idx_content_evidence_version
    ON content_evidence_entries(output_version_id);

CREATE TABLE content_approval_events (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    output_id         UUID NOT NULL REFERENCES content_outputs(id),
    output_version_id UUID NOT NULL,
    decision           TEXT NOT NULL CHECK (decision IN ('approved','rejected','request_revision','revoked')),
    actor_id          TEXT NOT NULL REFERENCES content_actors(id),
    approved_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    notes             TEXT NOT NULL DEFAULT '',
    version_hash      CHAR(64) NOT NULL CHECK (version_hash ~ '^[0-9a-f]{64}$')
);

ALTER TABLE content_approval_events
    ADD CONSTRAINT fk_content_approval_version_same_output
    FOREIGN KEY (output_version_id, output_id)
    REFERENCES content_output_versions(id, output_id);

CREATE INDEX idx_content_approval_events_version
    ON content_approval_events(output_version_id, approved_at DESC);
```

The database must enforce append-only behavior for `content_output_versions`, `content_evidence_entries`, and `content_approval_events`. Corrections create new rows and never update editorial history.

### 6.12 External references and delivery attempts

```sql
CREATE TABLE content_external_references (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    output_id         UUID NOT NULL REFERENCES content_outputs(id),
    output_version_id UUID NOT NULL,
    system            TEXT NOT NULL CHECK (system IN ('ecc','substack')),
    external_type     TEXT NOT NULL CHECK (external_type IN ('ecc_operational_draft','substack_queue_draft')),
    external_id       TEXT,
    delivery_status   TEXT NOT NULL CHECK (delivery_status IN ('queued','running','transferred','published','sent','failed','cancelled','needs_reconciliation')),
    idempotency_key   TEXT NOT NULL,
    payload_hash      CHAR(64) NOT NULL CHECK (payload_hash ~ '^[0-9a-f]{64}$'),
    last_synced_at    TIMESTAMPTZ,
    next_retry_at     TIMESTAMPTZ,
    attempt_count     INTEGER NOT NULL DEFAULT 0 CHECK (attempt_count >= 0),
    last_error_code   TEXT,
    safe_last_error   TEXT,
    reconciliation_status TEXT NOT NULL CHECK (reconciliation_status IN ('not_needed','pending','resolved')) DEFAULT 'not_needed',
    created_by_actor_id TEXT NOT NULL REFERENCES content_actors(id),
    created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (system, external_type, idempotency_key),
    UNIQUE (system, external_type, external_id)
);

ALTER TABLE content_external_references
    ADD CONSTRAINT fk_content_external_version_same_output
    FOREIGN KEY (output_version_id, output_id)
    REFERENCES content_output_versions(id, output_id);

CREATE UNIQUE INDEX uq_content_external_one_active_ref
    ON content_external_references(output_version_id, system, external_type)
    WHERE delivery_status NOT IN ('cancelled','failed');

CREATE INDEX idx_content_external_references_reconcile
    ON content_external_references(delivery_status, next_retry_at)
    WHERE delivery_status IN ('failed','needs_reconciliation');

CREATE TABLE content_delivery_attempts (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    external_reference_id UUID NOT NULL REFERENCES content_external_references(id) ON DELETE CASCADE,
    attempt_number    INTEGER NOT NULL CHECK (attempt_number >= 1),
    action            TEXT NOT NULL CHECK (action IN ('create_draft','queue_substack','sync_status','retry')),
    request_id        UUID NOT NULL,
    started_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at      TIMESTAMPTZ,
    outcome           TEXT CHECK (outcome IN ('running','transferred','published','sent','failed','needs_reconciliation')),
    external_id       TEXT,
    response_metadata JSONB NOT NULL DEFAULT '{}'::jsonb,
    safe_error        TEXT,
    UNIQUE (external_reference_id, attempt_number),
    UNIQUE (external_reference_id, request_id)
);

CREATE INDEX idx_content_delivery_attempts_reference
    ON content_delivery_attempts(external_reference_id, attempt_number DESC);
```

`published` and `sent` remain distinct. Substack publication is not inferred from transfer. ECC delivery/send state is not inferred from draft creation.

### 6.13 Workflow authority

```sql
CREATE TABLE content_workflow_authority (
    workflow_key      TEXT PRIMARY KEY,
    authoritative_writer TEXT NOT NULL CHECK (authoritative_writer IN ('ecc','hub')),
    hub_generation_enabled BOOLEAN NOT NULL DEFAULT FALSE,
    ecc_generation_enabled BOOLEAN NOT NULL DEFAULT TRUE,
    changed_by_actor_id TEXT NOT NULL REFERENCES content_actors(id),
    changed_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    rollback_writer   TEXT NOT NULL CHECK (rollback_writer IN ('ecc','hub')),
    notes             TEXT NOT NULL DEFAULT ''
);
```

V1A migration seed:

```text
workflow_key = email_7day_clicker
authoritative_writer = ecc
hub_generation_enabled = false
ecc_generation_enabled = true
rollback_writer = ecc
```

The cutover procedure changes this row only through an owner-authenticated administrative operation with an audit event.

### 6.14 Tracking and attribution read model

```sql
CREATE TABLE content_tracking_links (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    output_version_id UUID NOT NULL REFERENCES content_output_versions(id),
    cta_version_id    UUID REFERENCES content_cta_versions(id),
    channel_id        UUID NOT NULL REFERENCES content_channels(id),
    destination_url   TEXT NOT NULL CHECK (destination_url ~ '^https://'),
    tracked_url       TEXT NOT NULL CHECK (tracked_url ~ '^https://'),
    utm_parameters    JSONB NOT NULL DEFAULT '{}'::jsonb,
    internal_parameters JSONB NOT NULL DEFAULT '{}'::jsonb,
    link_hash         CHAR(64) NOT NULL CHECK (link_hash ~ '^[0-9a-f]{64}$'),
    created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (output_version_id, channel_id, link_hash)
);

CREATE TABLE content_attribution_observations (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    output_version_id UUID REFERENCES content_output_versions(id),
    source_system     TEXT NOT NULL CHECK (source_system IN ('ecc','memberpress','substack','site')),
    external_event_id TEXT NOT NULL,
    event_type        TEXT NOT NULL CHECK (event_type IN ('click','order_form_visit','sale','revenue')),
    occurred_at       TIMESTAMPTZ NOT NULL,
    collected_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    amount            NUMERIC(12,2),
    currency          CHAR(3),
    linkage_status    TEXT NOT NULL CHECK (linkage_status IN ('confirmed','linked','unattributed')),
    confidence        TEXT NOT NULL CHECK (confidence IN ('high','medium','low','none')),
    attribution_window_days INTEGER NOT NULL DEFAULT 7 CHECK (attribution_window_days > 0),
    raw_reference     JSONB NOT NULL DEFAULT '{}'::jsonb,
    UNIQUE (source_system, external_event_id, event_type)
);

CREATE INDEX idx_content_attribution_output_time
    ON content_attribution_observations(output_version_id, occurred_at DESC);

CREATE INDEX idx_content_attribution_unattributed
    ON content_attribution_observations(linkage_status, occurred_at DESC);
```

This is an imported read model. ECC remains authoritative for email revenue attribution.

### 6.15 Audit events

```sql
CREATE TABLE content_audit_events (
    id                BIGSERIAL PRIMARY KEY,
    request_id        UUID,
    actor_id          TEXT NOT NULL REFERENCES content_actors(id),
    auth_provider     TEXT,
    authenticated_subject TEXT,
    credential_key_id TEXT,
    work_session_id   TEXT,
    claim_id          UUID,
    lease_id          UUID,
    event_type        TEXT NOT NULL,
    entity_type       TEXT NOT NULL,
    entity_id         TEXT NOT NULL,
    before_state      JSONB,
    after_state       JSONB,
    metadata          JSONB NOT NULL DEFAULT '{}'::jsonb,
    created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_content_audit_entity
    ON content_audit_events(entity_type, entity_id, created_at DESC);

CREATE INDEX idx_content_audit_request
    ON content_audit_events(request_id, created_at DESC);
```

Audit rows are append-only and must not include secrets, raw email addresses, raw subscriber rows, or full external response bodies when those contain sensitive data.

### 6.16 Database-enforced integrity and append-only history

The application must repeat these validations for safe error messages, but the database also enforces the cross-entity relationships that cannot be safely left to UI code.

```sql
CREATE OR REPLACE FUNCTION content_validate_output_version_context()
RETURNS trigger
LANGUAGE plpgsql
AS $$
DECLARE
    output_revision  UUID;
    context_revision UUID;
BEGIN
    SELECT brief_revision_id
      INTO output_revision
      FROM content_outputs
     WHERE id = NEW.output_id;

    SELECT brief_revision_id
      INTO context_revision
      FROM content_context_snapshots
     WHERE id = NEW.context_snapshot_id;

    IF output_revision IS NULL OR context_revision IS NULL
       OR output_revision <> context_revision THEN
        RAISE EXCEPTION 'Output version context does not belong to the output brief revision'
            USING ERRCODE = '23514';
    END IF;

    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_content_output_versions_context
    BEFORE INSERT
    ON content_output_versions
    FOR EACH ROW
    EXECUTE FUNCTION content_validate_output_version_context();

CREATE OR REPLACE FUNCTION content_validate_approval_hash()
RETURNS trigger
LANGUAGE plpgsql
AS $$
DECLARE
    expected_hash CHAR(64);
BEGIN
    SELECT content_hash
      INTO expected_hash
      FROM content_output_versions
     WHERE id = NEW.output_version_id
       AND output_id = NEW.output_id;

    IF expected_hash IS NULL OR NEW.version_hash <> expected_hash THEN
        RAISE EXCEPTION 'Approval event hash does not match the exact output version'
            USING ERRCODE = '23514';
    END IF;

    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_content_approval_events_hash
    BEFORE INSERT
    ON content_approval_events
    FOR EACH ROW
    EXECUTE FUNCTION content_validate_approval_hash();

CREATE OR REPLACE FUNCTION content_validate_external_reference_approval()
RETURNS trigger
LANGUAGE plpgsql
AS $$
DECLARE
    approved_version UUID;
BEGIN
    SELECT approved_version_id
      INTO approved_version
      FROM content_outputs
     WHERE id = NEW.output_id;

    IF approved_version IS NULL OR approved_version <> NEW.output_version_id THEN
        RAISE EXCEPTION 'External reference must target the exact approved output version'
            USING ERRCODE = '23514';
    END IF;

    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_content_external_references_approved
    BEFORE INSERT OR UPDATE OF output_id, output_version_id
    ON content_external_references
    FOR EACH ROW
    EXECUTE FUNCTION content_validate_external_reference_approval();

CREATE OR REPLACE FUNCTION content_validate_evidence_reference()
RETURNS trigger
LANGUAGE plpgsql
AS $$
BEGIN
    IF NEW.evidence_type = 'approved_proof' THEN
        IF NOT EXISTS (
            SELECT 1
              FROM content_proof_records
             WHERE id = NEW.evidence_id
               AND approval_status = 'approved'
        ) THEN
            RAISE EXCEPTION 'Evidence entry references a missing or unapproved proof record'
                USING ERRCODE = '23503';
        END IF;
    ELSIF NEW.evidence_type = 'source_snapshot' THEN
        IF NOT EXISTS (
            SELECT 1
              FROM content_source_snapshots
             WHERE id = NEW.evidence_id
        ) THEN
            RAISE EXCEPTION 'Evidence entry references a missing source snapshot'
                USING ERRCODE = '23503';
        END IF;
    END IF;

    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_content_evidence_entries_reference
    BEFORE INSERT
    ON content_evidence_entries
    FOR EACH ROW
    EXECUTE FUNCTION content_validate_evidence_reference();

CREATE OR REPLACE FUNCTION content_validate_tracking_link_channel()
RETURNS trigger
LANGUAGE plpgsql
AS $$
DECLARE
    output_channel UUID;
BEGIN
    SELECT o.channel_id
      INTO output_channel
      FROM content_output_versions v
      JOIN content_outputs o ON o.id = v.output_id
     WHERE v.id = NEW.output_version_id;

    IF output_channel IS NULL OR output_channel <> NEW.channel_id THEN
        RAISE EXCEPTION 'Tracking link channel does not match the output version channel'
            USING ERRCODE = '23514';
    END IF;

    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_content_tracking_links_channel
    BEFORE INSERT OR UPDATE OF output_version_id, channel_id
    ON content_tracking_links
    FOR EACH ROW
    EXECUTE FUNCTION content_validate_tracking_link_channel();

CREATE OR REPLACE FUNCTION content_reject_append_only_mutation()
RETURNS trigger
LANGUAGE plpgsql
AS $$
BEGIN
    RAISE EXCEPTION 'Append-only content history cannot be updated or deleted'
        USING ERRCODE = '55000';
END;
$$;

CREATE TRIGGER trg_content_brief_revisions_append_only
    BEFORE UPDATE OR DELETE ON content_brief_revisions
    FOR EACH ROW EXECUTE FUNCTION content_reject_append_only_mutation();

CREATE TRIGGER trg_content_context_snapshots_append_only
    BEFORE UPDATE OR DELETE ON content_context_snapshots
    FOR EACH ROW EXECUTE FUNCTION content_reject_append_only_mutation();

CREATE TRIGGER trg_content_output_versions_append_only
    BEFORE UPDATE OR DELETE ON content_output_versions
    FOR EACH ROW EXECUTE FUNCTION content_reject_append_only_mutation();

CREATE TRIGGER trg_content_evidence_entries_append_only
    BEFORE UPDATE OR DELETE ON content_evidence_entries
    FOR EACH ROW EXECUTE FUNCTION content_reject_append_only_mutation();

CREATE TRIGGER trg_content_proof_records_append_only
    BEFORE UPDATE OR DELETE ON content_proof_records
    FOR EACH ROW EXECUTE FUNCTION content_reject_append_only_mutation();

CREATE TRIGGER trg_content_approval_events_append_only
    BEFORE UPDATE OR DELETE ON content_approval_events
    FOR EACH ROW EXECUTE FUNCTION content_reject_append_only_mutation();

CREATE TRIGGER trg_content_work_submissions_append_only
    BEFORE UPDATE OR DELETE ON content_work_submissions
    FOR EACH ROW EXECUTE FUNCTION content_reject_append_only_mutation();

CREATE TRIGGER trg_content_work_transport_receipts_append_only
    BEFORE UPDATE OR DELETE ON content_work_transport_receipts
    FOR EACH ROW EXECUTE FUNCTION content_reject_append_only_mutation();

CREATE TRIGGER trg_content_audit_events_append_only
    BEFORE UPDATE OR DELETE ON content_audit_events
    FOR EACH ROW EXECUTE FUNCTION content_reject_append_only_mutation();
```

The migration test suite must verify each trigger with both a direct `UPDATE` and a direct `DELETE`, plus mismatched output/version, context/revision, evidence-type, approval-hash, tracking-channel, transport-request/payload, and job/request/attempt fixtures. Business-level checks such as proof validity intervals, channel restrictions, and evidence-manifest completeness remain service-layer validation.

The SQL in this section is grouped by integrity concern for readability; migrations must split it by dependency. Output/version, evidence, approval, and editorial-history triggers are created in `V1A-006`; the external-reference approval trigger is created only after `content_external_references` exists in `V1A-007`; tracking-link validation is created in `V1A-009`.

---

## 7. Numbered migration plan

Migrations are applied in this order, each in its own transaction under the migration advisory lock.

The ECC schema compatibility change is a separately versioned ECC migration and is not part of the Hub migration transaction. It must be applied and verified before `CONTENT_ENGINE_ECC_ADAPTER_ENABLED` can become true.

| Migration | Purpose |
|---|---|
| `V1A-001` | Verify PostgreSQL version/`pgcrypto`; create `schema_migrations`, `content_actors`, `content_actor_bindings`, `content_channels`, and `content_runtime_settings` |
| `V1A-002` | Create audience, offer, CTA, proof, source, and rule snapshot tables; create the proof-record append-only trigger |
| `V1A-003` | Create campaigns, briefs, revisions, requested outputs, source refs, proof refs, campaign snapshot foreign keys, and brief-revision append-only trigger |
| `V1A-004` | Create context snapshots, context/revision constraints, and context append-only trigger |
| `V1A-005` | Create Work jobs, transport receipts, attempts, submissions, claim indexes, job/request/attempt foreign keys, and transport-receipt/submission append-only triggers |
| `V1A-006` | Create output headers, immutable versions, evidence entries, approval events, output/context constraints, approval-hash validation, and editorial append-only triggers |
| `V1A-007` | Create external references, delivery attempts, output/version pairing constraints, exact-approved-version trigger, idempotency constraints, and reconcile indexes |
| `V1A-008` | Create workflow authority table and seed `email_7day_clicker` with ECC authority and Hub generation disabled |
| `V1A-009` | Create tracking links, tracking-channel validation, attribution observations, audit events, audit append-only trigger, and indexes |
| `V1A-010` | Seed only `substack_article` and `email_7day_clicker`; seed initial runtime settings; validate schema invariants |

### 7.1 Migration rules

- Never combine all migrations into one startup `CREATE TABLE IF NOT EXISTS` string.
- Never run a migration concurrently from two application replicas.
- Never drop or truncate existing Substack/ECC tables as part of V1A.
- Never backfill production content as authoritative without an explicit mapping and verification report.
- A failed migration blocks V1A startup but must not silently prevent the existing Hub agent or existing Substack queue from operating if the feature flag is disabled.
- V1A routes must not become active until every required migration is recorded as applied and verified.

### 7.2 Required database migration tests

1. Apply all migrations to a clean PostgreSQL 13 database.
2. Apply all migrations to a production-like database using the verified Railway production PostgreSQL major/minor version and extension set, containing current Hub audit, knowledge, monitor, memory, and Substack tables.
3. Re-run all migrations and verify no changes or duplicate rows.
4. Interrupt a migration transaction and verify no partial migration row is recorded.
5. Run two migration processes concurrently and verify one waits on the advisory lock.
6. Verify all foreign keys, partial indexes, unique constraints, and append-only triggers exist.
7. Verify the feature remains disabled if V1A migration verification fails.
8. Verify the recorded production version is at least PostgreSQL 13 and matches the database queried immediately before migration design.

---

## 8. Work job JSON contracts

All schemas below are `schema_version: 1`. Unknown fields are rejected on writes. Responses may add non-breaking metadata only under a versioned envelope.

### 8.1 Claim request

```json
{
  "schema_version": 1,
  "transport_request_id": "uuid",
  "requested_job_id": "optional-uuid",
  "work_session_id": "work-session-2026-09-12-01",
  "max_wait_seconds": 0
}
```

Rules:

- `requested_job_id` may be omitted to claim the oldest eligible ready job.
- `max_wait_seconds` is advisory and capped at 10.
- A claim request is authenticated by the transport proven in the capability spike. The request body does not contain a secret or a client-generated signature.
- A claim never returns raw subscriber data or unselected context.

### 8.2 Claim response

```json
{
  "schema_version": 1,
  "job": {
    "job_id": "uuid",
    "request_id": "uuid",
    "job_type": "generate_content_package",
    "brief_id": "uuid",
    "brief_revision": 3,
    "context_snapshot_id": "uuid",
    "context_hash": "64-hex",
    "requested_outputs": [
      {
        "output_key": "substack_article:0:default",
        "channel": "substack_article",
        "item_index": 0,
        "variant_key": "default",
        "output_spec": {}
      },
      {
        "output_key": "email_7day_clicker:0:default",
        "channel": "email_7day_clicker",
        "item_index": 0,
        "variant_key": "default",
        "output_spec": {}
      }
    ],
    "context_snapshot": {},
    "expires_at": "2026-09-12T16:00:00Z"
  },
  "claim": {
    "claim_token": "opaque-short-lived-token",
    "claim_id": "uuid",
    "lease_id": "uuid",
    "lease_expires_at": "2026-09-12T12:15:00Z",
    "attempt_number": 1,
    "heartbeat_interval_seconds": 120
  }
}
```

The claim token is returned only over the authenticated response and is never stored in plaintext. Hub stores its SHA-256 hash.

### 8.3 Heartbeat request

```json
{
  "schema_version": 1,
  "transport_request_id": "uuid",
  "job_id": "uuid",
  "request_id": "uuid",
  "claim_token": "opaque-short-lived-token",
  "work_session_id": "work-session-2026-09-12-01",
  "progress": "drafting both requested outputs"
}
```

The server verifies the token, active lease, request ID, worker instance, and hard `expires_at`. It extends the lease by 10 minutes, never beyond the job hard expiry. A heartbeat after lease expiry returns `409 LEASE_EXPIRED`.

### 8.4 Result submission

```json
{
  "schema_version": 1,
  "transport_request_id": "uuid",
  "job_id": "uuid",
  "request_id": "uuid",
  "claim_token": "opaque-short-lived-token",
  "attempt_number": 1,
  "work_session_id": "work-session-2026-09-12-01",
  "claim_id": "uuid",
  "lease_id": "uuid",
  "submitted_at": "2026-09-12T12:11:00Z",
  "outputs": [
    {
      "output_key": "substack_article:0:default",
      "channel": "substack_article",
      "item_index": 0,
      "variant_key": "default",
      "title": "...",
      "subtitle": "...",
      "subject_line": null,
      "body": "...",
      "content_format": "markdown",
      "change_summary": "Initial Work draft",
      "evidence_manifest": []
    },
    {
      "output_key": "email_7day_clicker:0:default",
      "channel": "email_7day_clicker",
      "item_index": 0,
      "variant_key": "default",
      "title": "...",
      "subtitle": null,
      "subject_line": "...",
      "body": "...",
      "content_format": "html",
      "change_summary": "Initial Work draft",
      "evidence_manifest": []
    }
  ],
  "result_metadata": {
    "work_run_label": "optional-safe-label"
  }
}
```

V1A result rules:

- Exactly the requested output keys must be present.
- Missing or extra output keys cause `PARTIAL_RESULT` or `EXTRA_OUTPUT` and create no output versions.
- The complete package is accepted atomically or rejected atomically.
- Work cannot submit only the Substack or only the email output.
- A duplicate transport request with the same payload hash returns the original receipt and creates no new versions.
- A duplicate transport request ID with a different payload hash returns `409 RESULT_HASH_MISMATCH`.
- A late result after hard expiry creates a quarantined submission record but no versions and no delivery reference.
- Work may not include an approval decision, delivery request, external ID, ECC draft ID, Substack draft ID, or recipient data.

### 8.5 Failure report

```json
{
  "schema_version": 1,
  "transport_request_id": "uuid",
  "job_id": "uuid",
  "request_id": "uuid",
  "claim_token": "opaque-short-lived-token",
  "attempt_number": 1,
  "failure_code": "SOURCE_UNAVAILABLE",
  "safe_failure_message": "The selected source could not be read.",
  "retryable": true
}
```

The failure message must not include secrets, raw source bodies, subscriber data, or external response bodies.

---

## 9. Atomic claim, leases, and interrupted-job recovery

### 9.1 Atomic claim behavior

Claiming a job occurs in one database transaction:

1. Verify the selected transport authentication, external identity, scope, work session, and transport request receipt.
2. Begin transaction.
3. Select the requested job or oldest eligible job using `FOR UPDATE SKIP LOCKED`.
4. Eligible states are `ready`, or `claimed/working` with expired lease and remaining attempts before hard expiry.
5. Increment `attempt_count`.
6. Generate cryptographically random `claim_id`, `lease_id`, and an opaque claim token.
7. Store only the claim-token hash; store the non-secret claim/lease/session identifiers and authenticated identity metadata.
8. Set `status = 'claimed'`, server-derived `claimed_by_actor_id`, `authenticated_subject`, `credential_key_id`, `work_session_id`, `claim_id`, `lease_id`, `claimed_at`, `heartbeat_at`, `lease_expires_at`, and the attempt row.
9. Commit.
10. Return the package and plaintext claim token.

No two successful claim responses may reference the same active attempt. A duplicate transport claim request returns the original receipt or the existing active lease and cannot claim another job.

### 9.2 Lease rules

- Initial lease: 10 minutes
- Heartbeat interval: every 2 minutes
- Heartbeat extension: 10 minutes from current request time
- Maximum hard job lifetime: 2 hours from `created_at`
- Maximum attempts: 2 by default
- A lease may not extend beyond `expires_at`
- A claim token is valid only for its attempt, job, request ID, and worker instance
- A new claim invalidates the prior claim token by replacing its hash

### 9.3 Recovery sweeper

A Hub-owned recovery operation runs at least every 5 minutes but is not an audience-facing publisher.

For `claimed` or `working` jobs with `lease_expires_at < NOW()`:

- If `attempt_count < max_attempts` and `expires_at > NOW()`, record `lease_expired` and return the job to `ready`.
- If the hard expiry has passed, set `status = 'expired'`.
- If a result submission was received but could not be committed atomically, set `status = 'needs_reconciliation'` and block all delivery.

The sweeper must never create output versions or external references.

### 9.4 Late-result behavior

| Job condition | Late result behavior |
|---|---|
| Active lease and matching token | Validate and process normally |
| Job already `returned`, same payload hash | Return original receipt, no-op |
| Job already `returned`, different payload hash | Reject `RESULT_HASH_MISMATCH` |
| Job `failed` but retryable and token still valid | Reject unless Casey explicitly requeues |
| Job `expired` | Store `rejected_late` submission, create no versions |
| Job `cancelled` | Store `rejected_late` submission, create no versions |
| Job `needs_reconciliation` | Quarantine submission and require Casey review |

### 9.5 Partial-result behavior

V1A package generation is all-or-nothing. A result containing one of the two requested outputs is rejected, stored as `quarantined`, and moves the job to `needs_reconciliation` if the server cannot prove the result was incomplete before any write.

No partial output is approved or handed to ECC/Substack.

---

## 10. State machines

### 10.1 Editorial output state machine

States:

```text
requested
   ↓ Work result creates first version
drafting
   ↓ version passes structural validation
review
   ├─ Casey approves exact current version → approved
   ├─ Casey requests revision → drafting
   └─ Casey archives → archived

approved
   ├─ new revision requested → drafting; prior approved_version_id remains valid
   └─ Casey archives → archived
```

Rules:

- `current_version_id` points to the newest version.
- `approved_version_id` points only to the exact version Casey approved.
- A new version never inherits approval.
- A parent may have `current_version_id = 5` and `approved_version_id = 4`.
- Only Casey can create approval events.
- Work can create a new version only through an active claimed job and never changes `approved_version_id`.
- An archived output cannot receive a new version without explicit owner reactivation.

### 10.2 Work job state machine

```text
ready
  → claimed
  → working
  → returned

working
  → failed
  → ready              (lease expired, attempts remain)
  → expired            (hard expiry)
  → needs_reconciliation
  → cancelled          (Casey only)

claimed
  → working            (first heartbeat or explicit work-start)
  → ready              (lease expires, attempts remain)
  → expired
  → cancelled
```

`returned` means the complete result was accepted, output versions were created transactionally, and a receipt exists. It does not mean approved, delivered, sent, or published.

### 10.3 External delivery state machine

```text
no_reference
  → queued
  → running
  → transferred
  → published       (after Substack confirmation or Casey records the Substack post ID/URL)
  → sent            (ECC only, after ECC authoritative state confirms send)

running
  → failed
  → needs_reconciliation

failed
  → running         (Casey retries after reviewing the error)

needs_reconciliation
  → running         (Casey resolves the external state and explicitly retries)
  → cancelled

queued/running/transferred
  → cancelled       (Casey only, before external terminal action)
```

A failed, expired, or `needs_reconciliation` state blocks delivery. `transferred` is a Hub queue/receipt state reported by the authenticated browser extension; it is not publication confirmation. Automatic retry may update an adapter transport attempt only when the exact approved version, idempotency key, and external state are known to be safe. Otherwise the record requires Casey reconciliation.

---

## 11. Evidence Gate and output validation

Every output version must contain an evidence manifest. Freedom Hub validates it before Casey can approve the version.

### 11.1 Evidence entry types

```json
{
  "claim_text": "string",
  "claim_type": "marketing_claim | trading_performance | testimonial | business_result | research_fact",
  "evidence_type": "approved_proof | source_snapshot",
  "evidence_id": "uuid",
  "verified_at": "timestamp-or-null",
  "notes": "string"
}
```

### 11.2 Validation rules

1. `marketing_claim`, `trading_performance`, `testimonial`, and `business_result` claims require an `approved_proof` record.
2. The proof must be `approval_status = approved`.
3. The proof must be within its validity interval.
4. The proof’s allowed channel list must include the current channel or explicitly allow all channels.
5. Proof restrictions must be evaluated against the brief, audience, offer, and channel.
6. `research_fact` claims may use a `source_snapshot` instead of approved marketing proof.
7. Source snapshot claims must cite the exact source snapshot ID and content hash.
8. Dollar and percentage strings are not automatically rejected; research figures are allowed when sourced.
9. Unsupported performance or marketing claims block approval.
10. Unresolved `INSERT`, `TODO`, or `ASSET REQUIREMENTS` placeholders block approval.
11. Public URLs must use HTTPS and match an approved CTA or an explicitly selected source destination.
12. Output bodies must not contain credentials, signed Work tokens, raw recipient emails, or identity-bridge parameters.
13. Email output must not contain `/go/` links before ECC’s delivery layer wraps links.
14. Substack output must contain the approved cover/asset requirements when the brief requests them.
15. Validation results are stored with machine-readable codes and a safe human-readable message.

### 11.3 Approval behavior

Approval is disabled if any required evidence or structural validation fails. Casey may request revision, which leaves the prior approved version untouched and creates a new drafting cycle.

---

## 12. ECC operational draft adapter

### 12.1 Adapter boundary

The Hub ECC adapter is the only V1A actor allowed to create an operational ECC draft from a Hub output.

Work never calls the adapter.

The adapter accepts only:

```text
output_id
output_version_id
approved_version_id
workflow_key = email_7day_clicker
```

It re-reads the output, verifies `output_version_id = approved_version_id`, verifies the approval event and content hash, validates the workflow authority, and only then creates the ECC draft.

### 12.2 ECC-side mapping table

The ECC database needs a small mapping table owned by ECC:

```sql
CREATE TABLE content_engine_draft_links (
    id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workflow_key          TEXT NOT NULL,
    hub_output_id         UUID NOT NULL,
    hub_output_version_id UUID NOT NULL,
    hub_content_hash      CHAR(64) NOT NULL,
    ecc_draft_id          UUID NOT NULL REFERENCES email_drafts(id),
    payload_hash          CHAR(64) NOT NULL,
    status                TEXT NOT NULL CHECK (status IN ('created','cancelled','sent','reconciled')),
    created_at            TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at            TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (workflow_key, hub_output_version_id),
    UNIQUE (ecc_draft_id)
);

CREATE INDEX idx_content_engine_draft_links_hub_output
    ON content_engine_draft_links(hub_output_id, hub_output_version_id);
```

This prevents duplicate ECC operational drafts for the same exact Hub output version even if the adapter request is retried.

### 12.2.1 ECC-side database authority enforcement

The Hub authority row cannot by itself protect the separate ECC database. ECC must maintain an authority mirror and enforce it in the database, in addition to the route-level `409 CONTENT_ENGINE_AUTHORITATIVE` guard:

```sql
CREATE TABLE content_engine_workflow_authority (
    workflow_key          TEXT PRIMARY KEY,
    authoritative_writer  TEXT NOT NULL CHECK (authoritative_writer IN ('ecc','hub')),
    changed_by            TEXT NOT NULL,
    changed_at            TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    active                BOOLEAN NOT NULL DEFAULT TRUE
);

ALTER TABLE email_drafts
    ADD COLUMN content_engine_origin TEXT NOT NULL DEFAULT 'ecc_legacy',
    ADD COLUMN content_engine_workflow TEXT,
    ADD COLUMN content_engine_output_id UUID,
    ADD COLUMN content_engine_output_version_id UUID,
    ADD COLUMN content_engine_content_hash CHAR(64);

CREATE UNIQUE INDEX uq_ecc_hub_output_version
    ON email_drafts(content_engine_workflow, content_engine_output_version_id)
    WHERE content_engine_origin = 'hub_v1a'
      AND content_engine_output_version_id IS NOT NULL;
```

After the ECC compatibility migration has converted `segment_target` to its actual production `text[]` representation, install the following trigger shape on `email_drafts`:

```sql
CREATE OR REPLACE FUNCTION content_engine_guard_ecc_draft()
RETURNS trigger
LANGUAGE plpgsql
AS $$
DECLARE
    hub_authoritative BOOLEAN;
BEGIN
    SELECT authoritative_writer = 'hub'
      INTO hub_authoritative
      FROM content_engine_workflow_authority
     WHERE workflow_key = 'email_7day_clicker'
       AND active;

    IF TG_OP = 'INSERT' THEN
        IF COALESCE(hub_authoritative, FALSE)
           AND '__clicked_last_7__' = ANY(COALESCE(NEW.segment_target, ARRAY[]::text[]))
           AND (current_user <> 'content_engine_adapter_role'
                OR NEW.content_engine_origin <> 'hub_v1a'
                OR NEW.content_engine_workflow <> 'email_7day_clicker'
                OR NEW.content_engine_output_version_id IS NULL
                OR NEW.content_engine_content_hash IS NULL) THEN
            RAISE EXCEPTION 'Hub is authoritative for the 7-Day Clicker workflow'
                USING ERRCODE = '42501';
        END IF;
    ELSIF OLD.content_engine_origin = 'hub_v1a' THEN
        IF NEW.subject IS DISTINCT FROM OLD.subject
           OR NEW.body IS DISTINCT FROM OLD.body
           OR NEW.segment_target IS DISTINCT FROM OLD.segment_target
           OR NEW.email_type IS DISTINCT FROM OLD.email_type
           OR NEW.send_mode IS DISTINCT FROM OLD.send_mode
           OR NEW.content_engine_origin IS DISTINCT FROM OLD.content_engine_origin
           OR NEW.content_engine_workflow IS DISTINCT FROM OLD.content_engine_workflow
           OR NEW.content_engine_output_id IS DISTINCT FROM OLD.content_engine_output_id
           OR NEW.content_engine_output_version_id IS DISTINCT FROM OLD.content_engine_output_version_id
           OR NEW.content_engine_content_hash IS DISTINCT FROM OLD.content_engine_content_hash THEN
            RAISE EXCEPTION 'Hub-originated editorial content is immutable in ECC'
                USING ERRCODE = '42501';
        END IF;
    ELSIF COALESCE(hub_authoritative, FALSE)
          AND '__clicked_last_7__' = ANY(COALESCE(NEW.segment_target, ARRAY[]::text[])) THEN
        RAISE EXCEPTION 'Legacy ECC routes cannot create the Hub-authoritative workflow'
            USING ERRCODE = '42501';
    END IF;

    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_content_engine_guard_ecc_draft
    BEFORE INSERT OR UPDATE ON email_drafts
    FOR EACH ROW
    EXECUTE FUNCTION content_engine_guard_ecc_draft();

CREATE OR REPLACE FUNCTION content_engine_require_ecc_mapping()
RETURNS trigger
LANGUAGE plpgsql
AS $$
BEGIN
    IF NOT EXISTS (
        SELECT 1
          FROM content_engine_draft_links l
         WHERE l.ecc_draft_id = NEW.id
           AND l.workflow_key = NEW.content_engine_workflow
           AND l.hub_output_version_id = NEW.content_engine_output_version_id
           AND l.hub_content_hash = NEW.content_engine_content_hash
    ) THEN
        RAISE EXCEPTION 'Hub-originated ECC draft lacks its exact-version mapping'
            USING ERRCODE = '23514';
    END IF;
    RETURN NEW;
END;
$$;

CREATE CONSTRAINT TRIGGER trg_content_engine_require_ecc_mapping
    AFTER INSERT OR UPDATE OF content_engine_origin,
                              content_engine_workflow,
                              content_engine_output_version_id,
                              content_engine_content_hash
    ON email_drafts
    DEFERRABLE INITIALLY DEFERRED
    FOR EACH ROW
    WHEN (NEW.content_engine_origin = 'hub_v1a')
    EXECUTE FUNCTION content_engine_require_ecc_mapping();
```

The trigger is the database enforcement mechanism; the following rules are its contract:

1. If `content_engine_workflow_authority.authoritative_writer = 'hub'` for `email_7day_clicker` and the normalized target contains `__clicked_last_7__`, a normal ECC application role may not insert the row.
2. The only permitted insert path is the separately authenticated Hub adapter database role. The row must set `content_engine_origin = 'hub_v1a'`, `content_engine_workflow = 'email_7day_clicker'`, `content_engine_output_id`, `content_engine_output_version_id`, and `content_engine_content_hash`.
3. A deferred constraint trigger verifies before commit that a matching ECC-owned `content_engine_draft_links` row exists for the same workflow, `ecc_draft_id`, Hub `output_version_id`, and content hash. The mapping row is inserted in the same transaction as the draft.
4. The unique index rejects a second Hub-originated draft for the same exact approved Hub `output_version_id`, even if a future ECC route bypasses the frontend guard.
5. While Hub authority is active, any update to a Hub-originated reserved-audience draft that changes `subject`, `body`, `segment_target`, `email_type`, `send_mode`, `content_engine_origin`, `content_engine_workflow`, `content_engine_output_id`, `content_engine_output_version_id`, or `content_engine_content_hash` is rejected. Operational status changes remain governed by the existing ECC send/schedule safeguards.
6. A reserved-audience insert or content mutation that violates these rules raises a database error that ECC routes translate to HTTP `409 CONTENT_ENGINE_AUTHORITATIVE`.

The adapter role cannot approve or send. Casey’s existing ECC operational UI may review and advance allowed operational state, but it cannot mutate the Hub-originated editorial content without creating a new Hub output version and repeating Casey’s approval gate.

The Hub cutover operation must verify that this ECC authority mirror, trigger, unique index, mapping table, and adapter database role are active before setting Hub authority to `hub`. If the ECC-side enforcement gate cannot be verified, cutover is blocked.

The ECC-side initial seed is `email_7day_clicker → authoritative_writer = ecc`. The Hub cutover changes the ECC mirror and Hub authority only through Casey-authorized, audited operations; rollback restores both to ECC before legacy writer routes are re-enabled.

### 12.3 ECC adapter request

The authenticated ECC adapter endpoint accepts. If Hub cannot use a private service call to the ECC process, the explicit ECC-side route is `POST /internal/v1/content-engine/drafts`; it is private/service-authenticated and is not exposed to Work or the owner browser:

```json
{
  "schema_version": 1,
  "workflow_key": "email_7day_clicker",
  "hub_output_id": "uuid",
  "hub_output_version_id": "uuid",
  "hub_content_hash": "64-hex",
  "subject": "...",
  "body_html": "...",
  "segment_target": ["__clicked_last_7__"],
  "email_type": "content_engine_7day",
  "send_mode": "broadcast",
  "tracking_context": {
    "campaign_id": "uuid",
    "brief_id": "uuid",
    "output_id": "uuid",
    "output_version_id": "uuid",
    "cta_version_ids": ["uuid"]
  },
  "idempotency_key": "content-engine:email_7day_clicker:<output-version-id>"
}
```

The endpoint must:

1. Authenticate the Hub adapter credential.
2. Verify the workflow is Hub-authoritative.
3. Verify the payload hash and idempotency key.
4. Verify no existing mapping conflicts with the exact output version.
5. Insert the ECC `email_drafts` row with `status = 'draft'`.
6. Set `agent_created = false` or the explicit non-AI value used by ECC for human/operational drafts.
7. Set `segment_target = ['__clicked_last_7__']`.
8. Set `send_mode = 'broadcast'`.
9. Set `email_type = 'content_engine_7day'`.
10. Store the mapping row in the same transaction as the draft insert.
11. Return `ecc_draft_id`, payload hash, and operational status.

The adapter must not approve or schedule the ECC draft. Casey must review and approve/send it through ECC’s existing operational UI and safeguards.

### 12.4 ECC schema compatibility gate

Before enabling the adapter, inspect the actual target ECC database schema in staging and production-like environments. Do not rely on the original bootstrap migration alone: the current ECC application code expects `email_drafts.segment_target` to behave as a PostgreSQL `text[]`, while older bootstrap definitions used scalar text and restrictive checks.

The ECC-side migration prerequisite must either verify or safely establish all of the following without losing existing draft data:

- `email_drafts.segment_target` accepts the existing array representation and the reserved `__clicked_last_7__` token.
- Existing segment values and queries continue to work for non-migrated workflows.
- `email_drafts.email_type` accepts `content_engine_7day`, or the adapter uses a documented existing value that is explicitly approved for this workflow.
- `email_drafts.id` remains UUID-compatible with `content_engine_draft_links.ecc_draft_id`.
- `email_drafts` contains the Hub-origin/workflow/output/version/hash metadata required by the ECC authority trigger, with legacy rows defaulting to `ecc_legacy` and no Hub output version.
- The new `content_engine_draft_links` table and its idempotency constraints are applied in the ECC database, not Hub’s database.

The adapter must fail closed with `MIGRATION_NOT_READY` until this schema gate passes. The compatibility migration and a populated-database rollback test belong in the ECC implementation plan.

### 12.5 Existing ECC writer cutover enforcement

The ECC authority guard must run in all existing code paths listed in Section 4.1 and the direct/manual draft-create and update paths named in Section 4.3. The guard is evaluated using the normalized `segment_target`, not only the frontend selector, so a direct API call cannot bypass it. The database trigger and deferred mapping constraint remain the final enforcement boundary if a future route is accidentally created without the application guard.

When `authoritative_writer = hub`:

- Existing AI generation routes reject the migrated audience.
- Existing manual AI campaign approval rejects the migrated audience.
- Existing slot generation rejects the migrated audience.
- Existing daily draft jobs remain disabled.
- Existing non-migrated audience workflows remain available.

---

## 13. ECC `/go/` link wrapping and tracking

### 13.1 Required tracking fields

Every V1A tracked CTA has both public UTM parameters and internal identifiers.

Public parameters:

```text
utm_source=email
utm_medium=broadcast
utm_campaign=ce_<campaign-slug>
utm_content=brief_<brief-short-id>_output_<output-short-id>_v<version>
utm_term=cta_<cta-key>_v<cta-version>
```

Internal parameters stored in Hub and the ECC mapping layer:

```text
fio_content_engine=1
fio_campaign_id=<uuid>
fio_brief_id=<uuid>
fio_output_id=<uuid>
fio_output_version_id=<uuid>
fio_cta_version_id=<uuid>
fio_workflow=email_7day_clicker
```

Internal identifiers contain UUIDs but no personal data.

### 13.2 ECC interaction order

The V1A email adapter supplies canonical HTTPS destination links with the deterministic UTM and `fio_*` parameters already present.

ECC then follows its existing order:

1. The adapter stores the operational ECC draft and mapping.
2. `applyDraftUtmsToBody()` may add missing legacy ECC UTM fields but must not overwrite existing V1A values.
3. At test-send or live-send time, `wrapLinksForTracking()` wraps canonical HTTP(S) links through `/go/:slug`.
4. `wrapLinksForTracking()` must continue skipping links already containing `/go/`.
5. The Hub adapter must therefore submit canonical destination links, never pre-wrapped `/go/` links.
6. `redirect_links.draft_id` maps each generated `/go/` link back to the ECC draft.
7. `click_events.draft_id` maps the click back to `content_engine_draft_links` and therefore to the exact Hub output version.

No V1A output may contain raw `?e=<email>` identity parameters. Existing ECC delivery behavior may add its current recipient-resolution mechanism at send time. The existing `/go/:slug` route must continue stripping raw email/identity parameters before redirecting and may create a signed identity token only when the existing identity bridge is enabled.

### 13.3 No double wrapping

The implementation must test that:

- Existing `/go/` links remain unchanged.
- Canonical V1A links are wrapped exactly once at ECC send/test time.
- UTM parameters survive the redirect destination.
- `redirect_links.draft_id` is the mapped ECC draft ID.
- Click events can join to the exact Hub output version.
- A failed link-wrap operation fails closed for the test/delivery action rather than silently removing attribution.

The current ECC fallback behavior that sends an unwrapped direct link when tracking creation fails must be overridden for V1A content. V1A delivery must fail or require Casey reconciliation if deterministic tracking cannot be created.

---

## 14. Substack external-reference adapter

### 14.1 Adapter boundary

The Hub Substack adapter is the only V1A actor allowed to create or replace a queued Substack draft from an approved output version.

Freedom Hub owns the existing `substack_drafts` queue, validation, queue status, and receipt state. V1A must reuse that existing Hub queue and receipt path; it must not create a second Substack queue abstraction. The local browser extension is the authenticated transfer client. Substack owns the editor/draft state after transfer, final publication state, post ID, and post URL. Browser/session state is never authoritative.

The adapter must not publish. Casey remains the final publisher.

### 14.2 Adapter flow

1. Casey approves the exact Substack output version.
2. Hub verifies the approval event, content hash, evidence status, and channel.
3. Hub computes idempotency key:

```text
content-engine:substack_article:<output-version-id>
```

4. Hub checks `content_external_references` for an existing non-cancelled mapping.
5. Hub calls the existing `agent/substack.py` queue storage/adapter path with title, subtitle, Markdown body, audience, and paywall configuration. This updates or creates the existing Hub-owned `substack_drafts` row; it does not create a second queue.
6. Existing Substack validation remains active, including unresolved-placeholder and image validation.
7. Hub stores the returned `substack_drafts.id` as `external_id`.
8. Hub stores the existing Hub queue receipt ID and exact approved-version mapping.
9. The browser extension retrieves the queued draft using its existing separate extension credential and transfers it into the authenticated Substack editor.
10. The extension may report the Hub queue/receipt state as `transferred`; this is not Substack publication confirmation.
11. Casey publishes manually in the authenticated Substack editor. `published` requires Substack confirmation or Casey’s explicit manual record of the Substack post ID and post URL.

### 14.3 Existing Substack compatibility

The existing private GPT Action and GitHub inbox remain supported for legacy articles. They must not be used as a way for Work to bypass V1A Hub approval. Those legacy transports still enter the same Hub-owned queue; they do not establish Substack as the queue owner.

V1A content must enter the existing queue only through the Hub adapter after exact-version approval.

Retries use the same idempotency key and must reuse or reconcile the existing Hub-queued draft rather than create a duplicate. Browser/session state is never used as the durable retry or publication source of truth.

### 14.4 Existing Substack credential boundary

The current Freedom Hub queue path is `agent/substack.py`, including `POST /api/v1/substack/drafts` and its internal `_store_substack_draft()` function. V1A may reuse that validated storage path, but it must introduce an explicit V1A adapter-authentication branch using `SUBSTACK_CONTENT_ENGINE_ADAPTER_KEY` (or an equivalent separately scoped credential). The legacy `SUBSTACK_ACTION_SECRET` used by the existing private GPT Action and the browser extension credential must remain separate and must never be given to Work.

The queue route must record whether the caller is the V1A adapter or the legacy action, and V1A calls must still require exact approved-version validation in Hub before invoking the queue path. The existing same-title replacement behavior may remain, but the Hub external-reference mapping and idempotency key are the authoritative duplicate/retry guard.

---

## 15. Attribution and conversion join

### 15.1 Attribution model

V1A uses:

- Last attributable content touch
- Seven-day lookback window
- UTC storage
- America/New_York reporting boundaries
- USD reporting currency
- Confirmed, linked, or unattributed revenue classifications
- Anonymous visitors remaining anonymous until legitimate linkage exists

Advanced multi-touch attribution is deferred.

### 15.2 Precedence rules

For an eligible conversion, use the first valid match in this order:

1. A valid signed identity-bridge touch that maps to the exact content-engine output version.
2. Explicit checkout attribution fields containing valid `fio_output_version_id` or deterministic V1A UTM fields within the seven-day window.
3. ECC’s canonical email click/revenue attribution where `click_events.draft_id` joins through `content_engine_draft_links` to an output version, with a human click within seven days before the completed conversion.
4. A legitimate visitor/session linkage to a V1A tracked URL where the conversion can be joined without guessing.
5. Otherwise, `unattributed`.

Do not use opens as an attribution touch. Do not infer attribution from the presence of a subscriber record alone. Do not treat a pending or incomplete payment as a sale.

When multiple eligible clicks exist, select the latest valid click before the conversion. If timestamps tie, select the smallest stable event ID for deterministic behavior.

ECC remains canonical for email revenue attribution. Hub imports the result and its confidence rather than recalculating a competing revenue number.

### 15.3 Attribution observation mapping

Hub imports only normalized observations and safe references:

```text
source_system
external_event_id
event_type
occurred_at
amount/currency where applicable
linkage_status
confidence
attribution_window_days
output_version_id when legitimately linked
```

No raw subscriber records are copied into Hub.

### 15.4 Required propagation tests

The controlled test suite must prove:

1. A Hub email output receives deterministic UTM and `fio_*` fields.
2. ECC draft persistence preserves those fields.
3. ECC `/go/` wrapping creates a redirect row linked to the ECC draft.
4. A click creates a `click_events` row linked to the ECC draft.
5. The click joins through the ECC mapping to the Hub output version.
6. The existing identity bridge, when enabled, carries only its signed token and not raw email into the destination URL.
7. A controlled checkout/conversion test can map the eligible conversion to the output version without a real production charge.
8. A conversion without legitimate linkage remains unattributed.
9. A pending transaction remains excluded from confirmed revenue.
10. Duplicate webhook/revenue records do not create duplicate Hub observations.

No real payment is submitted as part of this test.

---

## 16. HTTP/API surface

All owner routes require the existing Hub authenticated owner session and CSRF protection for mutations. All Work routes require the transport selected by the completed capability spike; they do not require client-side HMAC signing. Adapter routes are internal or separately authenticated service routes and are not exposed to Work.

### 16.1 Owner routes

```text
GET    /api/v1/content/today
GET    /api/v1/content/campaigns
POST   /api/v1/content/campaigns
GET    /api/v1/content/briefs/:brief_id
POST   /api/v1/content/briefs
POST   /api/v1/content/briefs/:brief_id/revisions
POST   /api/v1/content/briefs/:brief_id/work-jobs
GET    /api/v1/content/work-jobs/:job_id
POST   /api/v1/content/outputs/:output_id/approve
POST   /api/v1/content/outputs/:output_id/request-revision
POST   /api/v1/content/outputs/:output_id/reject
POST   /api/v1/content/outputs/:output_id/handoff
POST   /api/v1/content/external-references/:reference_id/retry
POST   /api/v1/content/external-references/:reference_id/reconcile
POST   /api/v1/content/runtime/pause
POST   /api/v1/content/runtime/resume
POST   /api/v1/content/workflow-authority/:workflow_key/cutover
POST   /api/v1/content/workflow-authority/:workflow_key/rollback
```

The approve, reject, request-revision, and handoff routes must all accept `output_version_id` and `version_hash`; they must reject an action based only on `output_id`. Handoff additionally accepts the target system (`ecc` or `substack`) and creates or advances the exact-version external-reference workflow only after durable approval.

### 16.2 Work routes

```text
POST   /api/v1/work/jobs/claim
POST   /api/v1/work/jobs/:job_id/heartbeat
POST   /api/v1/work/jobs/:job_id/result
POST   /api/v1/work/jobs/:job_id/fail
```

Work has no `GET all jobs` route. Claim returns the selected package. A Work read route may retrieve only a currently claimed job and must require the matching claim token.

### 16.3 Adapter routes

Preferred implementation is an internal Hub service call. If an HTTP boundary is required:

```text
POST   /internal/v1/adapters/ecc/drafts
POST   /internal/v1/adapters/substack/queue
POST   /internal/v1/adapters/ecc/status-sync
POST   /internal/v1/adapters/substack/status-sync
```

These Hub routes require adapter credentials and reject the Work transport credential/connection. They accept only approved exact-version references. If the ECC adapter is implemented as an ECC-side HTTP boundary, it additionally exposes only the private `POST /internal/v1/content-engine/drafts` route described in Section 12.3, authenticated with `ECC_CONTENT_ENGINE_DRAFT_WRITE_KEY`.

### 16.4 Error envelope

All new routes use:

```json
{
  "error": {
    "code": "MACHINE_READABLE_CODE",
    "message": "Safe human-readable message.",
    "request_id": "uuid",
    "retryable": false,
    "details": {}
  }
}
```

No error response contains secrets, raw prompts, raw recipient data, or unredacted external response bodies.

---

## 17. UI requirements

V1A builds only the UI required for the first loop. The existing Freedom Hub application remains the host. No second app or content service is created.

### 17.1 Today

Display:

- Current campaign
- Current brief and revision
- Ready-for-Work job count
- Work job status, lease, attempt, expiry, and last failure
- Review-capacity indicator: open package count, maximum, oldest review age
- Substack output state
- 7-Day Clicker email output state
- ECC/Substack integration freshness
- Stale/unavailable errors
- Generation pause state and reason

Actions:

- Open Brief
- Create Work Job
- Copy the Work initiation instruction
- Open Review
- Pause/resume generation as Casey
- Open external draft/queue reference after it exists

The UI must not show a Work button that directly approves, sends, publishes, or calls an adapter.

### 17.2 Brief Detail

Display:

- Campaign and brief identity
- Brief revision number
- Core idea, promise, angle, objective, constraints
- Selected audience-definition snapshot and freshness
- Selected offer snapshot and source system
- Selected CTA version and canonical destination
- Selected source snapshots and hashes
- Selected proof records and verification dates
- Applicable rules and priorities
- Requested outputs
- Context snapshot ID/hash
- Work job state and receipts
- External references and errors

Actions:

- Create a new revision
- Create one Work job when capacity allows
- Open exact output versions
- Request revision

### 17.3 Review

Display:

- Substack and email outputs side by side
- Exact version number and content hash
- Current versus approved version
- Evidence manifest and validation result
- Revision history
- Selected context snapshot
- CTA tracking preview
- External handoff state
- Safe errors and reconciliation status

Actions:

- Approve exact version
- Reject exact version
- Request revision
- Handoff approved version to Substack or ECC
- Open the external operational record

Approval requires Casey’s authenticated session, explicit version/hash confirmation, and a confirmation step. Handoff controls are disabled until approval is durable.

### 17.4 Minimal Pipeline

Show brief-centered cards in these columns:

```text
Brief Ready → Ready for Work → In Work → Review → Approved → External Handoff → Complete / Needs Reconciliation
```

Cards represent a brief revision/package, not individual platform posts. The card shows both requested outputs and their separate external states.

### 17.5 Fail-closed UI rules

- Failed, expired, and `needs_reconciliation` jobs show no send/publish action.
- Stale ECC or source data shows “stale as of [timestamp]” or “unavailable,” never zero.
- A non-approved output never shows an active delivery button.
- A current version newer than `approved_version_id` shows a clear stale-approval warning.
- A duplicate/existing external reference shows the existing record rather than a new action.

---

## 18. Review backpressure

### 18.1 Definition

An open review package is a distinct brief revision with at least one output in `review` and no terminal completion state.

V1A counts packages, not individual outputs. The default maximum is one package.

### 18.2 Rules

1. Creating a new Work job is blocked when open review packages are at or above `max_open_review_packages`.
2. One-package-at-a-time mode blocks all new generation while any package is in `in_work`, `review`, or `needs_reconciliation`.
3. A review older than 24 hours displays a warning.
4. A review older than 72 hours pauses generation until Casey resumes it.
5. Any `needs_reconciliation` external reference pauses the relevant workflow.
6. A failed Work job may be retried only by Casey and does not automatically generate more work.
7. Backpressure decisions are recorded in `content_audit_events`.

---

## 19. Error, retry, stale, and reconciliation behavior

### 19.1 Required error codes

```text
AUTH_INVALID
AUTH_SCOPE_DENIED
TRANSPORT_SPIKE_REQUIRED
TRANSPORT_UNSUPPORTED
TRANSPORT_REPLAY
WORK_SESSION_INVALID
PAYLOAD_TOO_LARGE
JOB_NOT_FOUND
JOB_NOT_CLAIMABLE
LEASE_EXPIRED
CLAIM_TOKEN_INVALID
LATE_RESULT
RESULT_HASH_MISMATCH
PARTIAL_RESULT
EXTRA_OUTPUT
CONTEXT_HASH_MISMATCH
EVIDENCE_INVALID
OUTPUT_VALIDATION_FAILED
BACKPRESSURE_ACTIVE
NOT_APPROVED
STALE_APPROVAL
WORKFLOW_NOT_AUTHORITATIVE
EXTERNAL_DUPLICATE
EXTERNAL_AUTH_FAILED
EXTERNAL_UNAVAILABLE
TRACKING_CREATION_FAILED
MIGRATION_NOT_READY
RECONCILIATION_REQUIRED
```

### 19.2 Retry rules

- Work lease expiry may return a job to `ready` within the hard expiry and attempt limit.
- Invalid result payloads are not automatically retried; Casey may create a new attempt.
- ECC/Substack adapter transport errors may be retried only with the same idempotency key and exact approved version.
- A timeout after an external request was sent becomes `needs_reconciliation` unless the external system can prove no write occurred.
- No failed/expired/reconciliation state may trigger a send or publication.
- Automatic background retries may reconcile status reads but may not create a new external draft without a known-safe idempotent operation.

### 19.3 Stale data

Every snapshot and external read includes freshness. The UI displays stale/unavailable status. Missing data is not converted to zero, “none,” or an empty audience.

---

## 20. Credential storage and rotation

### 20.1 Storage

All credentials remain in Railway environment variables or the existing platform secret mechanism. Never store them in:

- Postgres content rows
- Context snapshots
- Work result payloads
- GitHub issues
- Prompts
- Audit events
- Browser local storage

### 20.2 Required credential classes

```text
WORK_TRANSPORT_CREDENTIAL_OR_CONNECTION
WORK_TRANSPORT_PROVIDER
WORK_TRANSPORT_CREDENTIAL_KEY_ID
ECC_CONTENT_ENGINE_DRAFT_WRITE_KEY
SUBSTACK_CONTENT_ENGINE_ADAPTER_KEY
```

`WORK_TRANSPORT_CREDENTIAL_OR_CONNECTION` is a placeholder for the credential mechanism actually proven in Casey’s Work environment. It may be a platform-managed API-key/Bearer or OAuth connection for a Hub action/plugin, or a Hub-held GitHub polling credential for the GitHub inbox fallback. Work never receives the underlying secret. Existing Substack Action and extension credentials remain separate. The extension credential is never exposed to Work.

### 20.3 Rotation

1. Rotate the selected provider connection or Hub-held credential according to that provider’s supported procedure.
2. Verify the actual Work environment can authenticate using the replacement connection before retiring the prior one.
3. Keep the prior connection only for the documented in-flight lease/reconciliation window.
4. Revoke the prior connection.
5. Record provider, credential key reference, actor, time, and reason in an audit event without recording the secret.

---

## 21. Automated test specification

### 21.1 Schema and migration tests

- Clean database migration succeeds on the minimum supported PostgreSQL 13 version and the verified Railway production major/minor version.
- Populated production-like database migration succeeds against the verified production version and extension set.
- Re-running migrations is idempotent.
- Migration advisory lock serializes concurrent runners.
- Failed migration rolls back its transaction.
- Required indexes and constraints exist.
- Append-only triggers reject version/approval/audit updates and deletes.
- Proof records reject direct mutation/deletion and revoked/expired versions retain approval metadata.
- External references reject a version that is not the output's exact durable `approved_version_id`.
- Seed channels are exactly the two V1A channels.
- Initial authority is ECC with Hub generation disabled.

### 21.2 Authorization tests

- Unauthenticated owner mutation is rejected.
- An unproven or unsupported Work transport is rejected with `TRANSPORT_SPIKE_REQUIRED` or `TRANSPORT_UNSUPPORTED`.
- The selected Work transport cannot call owner routes, approve, reject, or call ECC/Substack adapter routes.
- Adapter key cannot approve or modify canonical business snapshots.
- A transport-authenticated identity that is not bound to an approved Work actor is rejected.
- A replayed transport request is rejected or returns its original receipt without repeating the write.
- A reused transport request ID with a changed payload is rejected.
- A duplicate claim request cannot claim a second job after a response timeout.
- Credential/connection rotation follows the proven provider mechanism and preserves in-flight lease safety.
- Human actor identity is derived from the authenticated Hub owner session, not request text.
- Work session, credential reference, claim ID, and lease ID are present in job attempts and audit events.
- Error responses do not leak credentials or raw recipient data.

### 21.3 Work job tests

- Two concurrent claims cannot claim the same job.
- Claim returns one plaintext token and stores only its hash.
- Wrong token cannot heartbeat or submit.
- Heartbeat extends the lease but not beyond hard expiry.
- Lease expiry returns a job to ready when attempts remain.
- Hard expiry produces `expired`.
- Interrupted result transaction produces `needs_reconciliation` rather than disappearing.
- Late result after expiry creates no output version.
- Same result replay returns the original receipt and creates no duplicate versions.
- The transport-capability spike passes authentication, harmless retrieval, atomic claim, harmless result, and receipt replay with zero output versions and zero external side effects.
- Different result with the same request ID is rejected.
- Partial result creates no output versions.
- Extra output creates no output versions.
- Context hash mismatch is rejected.
- Attempt count and attempt rows remain consistent.

### 21.4 Editorial/version tests

- Output versions are immutable.
- Version numbers are unique per output.
- Approval references the exact version and content hash.
- New version after approval does not change `approved_version_id`.
- A non-approved version cannot be delivered.
- A stale approval is visible and blocks handoff of the newer version.
- Approval events are append-only.
- Request revision leaves the previous approved version intact.
- Archived outputs cannot be edited without explicit reactivation.

### 21.5 Evidence tests

- Unsupported marketing claim blocks approval.
- Expired proof blocks approval.
- Revoked proof blocks approval.
- Channel-restricted proof cannot be used on an unauthorized channel.
- Sourced research facts are allowed with valid source snapshots.
- Missing source hash fails validation.
- Placeholder text fails validation.
- Unsafe or non-HTTPS CTA fails validation.
- Evidence manifest and normalized evidence rows remain consistent.

### 21.6 ECC adapter tests

- Exact approved version creates one ECC operational draft.
- Duplicate adapter request returns the existing ECC draft ID.
- Same Hub output version cannot create two ECC drafts.
- Non-approved version is rejected.
- Wrong workflow authority is rejected.
- `segment_target` is exactly `['__clicked_last_7__']`.
- Existing ECC AI-generation paths are blocked for the migrated audience after cutover.
- Non-migrated ECC generation still works.
- Existing scheduled/in-flight rows are not silently deleted.
- ECC draft mapping stores the exact Hub output version and content hash.
- ECC draft creation does not approve or schedule the draft.
- ECC schema compatibility rejects adapter activation when the array segment target, reserved token, email type, UUID, or mapping-table prerequisite is missing.
- Direct/manual ECC draft creation and update paths are blocked for the reserved audience after cutover.
- The ECC database trigger blocks a legacy-role insert for `__clicked_last_7__` while Hub authority is active.
- The ECC database trigger blocks mutation of Hub-originated editorial fields and origin/version mapping.
- The deferred constraint rejects a Hub-originated draft without a matching exact-version `content_engine_draft_links` row.
- The unique ECC exact-version index prevents duplicate Hub-originated drafts even when application route guards are bypassed.

### 21.7 Substack adapter tests

- Exact approved version creates one queued draft.
- Duplicate handoff reuses the existing queued draft.
- Existing title-replacement behavior remains intact where intended.
- Existing image and placeholder validation remains active.
- Extension transfer updates the external reference but does not publish.
- Non-approved version is rejected.
- Failed transfer is visible and retryable without duplicate queue items.
- Legacy Substack Action credentials and the V1A adapter credential cannot be used interchangeably.

### 21.8 Tracking and attribution tests

- Every V1A CTA receives deterministic UTM and internal fields.
- ECC persistence does not overwrite V1A fields.
- ECC `/go/` wrapping happens exactly once.
- Redirect destination retains UTMs and internal identifiers.
- Click joins from `/go/` to ECC draft mapping and Hub output version.
- Identity bridge linkage is accepted only when signature and time window are valid.
- Existing `revenue-attribution.js` remains the canonical ECC revenue path.
- Confirmed revenue excludes pending/incomplete payments.
- Last valid click within seven days wins.
- Opens do not create attribution.
- Anonymous conversions remain unattributed.
- Duplicate external events are deduplicated.

### 21.9 Backpressure and fail-closed tests

- New job creation pauses at `max_open_review_packages`.
- One-package-at-a-time mode blocks concurrent package generation.
- Old review warning appears at 24 hours.
- Generation pauses at 72 hours.
- Failed/expired/reconciliation states have no delivery action.
- External timeout does not automatically send or publish.
- Stale ECC data renders stale/unavailable, not zero.

### 21.10 UI tests

- Today renders job, review-capacity, and integration freshness state.
- Brief Detail shows context snapshot and selected evidence.
- Review shows current versus approved version.
- Approval requires exact version/hash.
- Pipeline separates review from external handoff.
- No Work user action can deliver content.
- No failed/reconciliation state shows a send/publish action.

---

## 22. Dark launch and canary procedure

### 22.1 Dark-launch flags

V1A must have independent flags:

```text
CONTENT_ENGINE_V1A_ENABLED=false
CONTENT_ENGINE_WORK_ENABLED=false
CONTENT_ENGINE_WORK_TRANSPORT=unproven
CONTENT_ENGINE_ECC_ADAPTER_ENABLED=false
CONTENT_ENGINE_SUBSTACK_ADAPTER_ENABLED=false
CONTENT_ENGINE_DELIVERY_ENABLED=false
CONTENT_ENGINE_EMAIL_7D_AUTHORITY=ecc
```

The exact configuration mechanism may use Railway environment variables plus the database workflow-authority row, but the effective state must be visible in Hub and audited. `CONTENT_ENGINE_WORK_TRANSPORT` may become `hub_action_bearer`, `hub_action_oauth`, or `github_inbox` only after the capability spike passes; `unproven` blocks Work routes.

### 22.2 Dark launch

1. Complete and record the actual ChatGPT Work transport-capability spike.
2. Apply and verify migrations against the verified PostgreSQL version.
3. Keep all adapter and delivery flags disabled.
4. Create a controlled internal campaign and brief.
5. Create a context snapshot and Work job.
6. Casey starts Work manually.
7. Work returns both outputs.
8. Hub validates and stores immutable versions.
9. Casey reviews but does not approve for external delivery yet.
10. Verify no ECC draft, Substack queue item, send, or publication was created.
11. Verify audit events, hashes, and Work receipts.

### 22.3 Internal adapter canary

After dark-launch validation:

1. Enable only the adapter dry-run mode.
2. Re-run exact-version validation and tracking preview.
3. Verify the ECC payload maps to `__clicked_last_7__` without creating a live draft, or create an explicitly Casey-only test draft if the adapter requires a real record.
4. Verify the Substack payload without creating a public publication.
5. Verify duplicate and retry behavior.
6. Keep `CONTENT_ENGINE_DELIVERY_ENABLED=false`.

### 22.4 First controlled production loop

1. Casey approves the architecture implementation and rollout checklist.
2. Casey authorizes workflow cutover.
3. Set ECC authority to Hub for `email_7day_clicker`.
4. Disable existing ECC generation for the reserved audience.
5. Enable Hub Work and adapter creation, but keep sending and publishing manual.
6. Run exactly one package.
7. Casey approves exact versions.
8. Hub creates one ECC draft and one Substack queue item.
9. Casey inspects both operational records.
10. Casey manually sends and publishes.
11. Run the attribution propagation test and verify results.
12. Keep one-package-at-a-time mode until Casey explicitly approves expansion.

---

## 23. Production rollout checklist

### Before migration

- [ ] This implementation specification is approved.
- [ ] The actual Railway PostgreSQL `version()`, `server_version_num`, `server_version`, and `pgcrypto` extension version are recorded; the verified major is at least 13.
- [ ] Migration tests pass against PostgreSQL 13 and the exact verified Railway production version.
- [ ] ECC production API authentication is deployed and verified.
- [ ] ECC read-only and draft-write credentials are scoped separately.
- [ ] ECC `email_drafts` schema compatibility is verified: array `segment_target`, reserved-token support, approved `email_type`, UUID draft IDs, and mapping-table constraints.
- [ ] The actual ChatGPT Work transport-capability spike has passed and its transport evidence is recorded.
- [ ] The selected Work transport credential/connection is configured and never exposed as a reusable secret to Work.
- [ ] Adapter credentials are configured and never exposed to Work.
- [ ] Current Hub and ECC database backups are verified restorable.
- [ ] Existing ECC 7-Day Clicker drafts and schedules are inventoried.
- [ ] Daily draft automation remains disabled.
- [ ] Current Substack queue and extension flow passes regression tests.

### Migration

- [ ] Apply V1A migrations under the advisory lock.
- [ ] Verify migration ledger and checksums.
- [ ] Run clean and populated database migration tests.
- [ ] Verify feature flags remain disabled.
- [ ] Verify the initial workflow authority is ECC.
- [ ] Verify the ECC-side authority mirror, trigger, deferred mapping constraint, unique exact-version index, and adapter database role are installed but inactive for Hub authority.
- [ ] Verify no existing operational records were deleted or modified unexpectedly.

### Dark launch

- [ ] Create one internal brief.
- [ ] Create context snapshot.
- [ ] Claim and complete one Work job.
- [ ] Verify two immutable output versions.
- [ ] Verify evidence validation.
- [ ] Verify no external draft or queue item exists.
- [ ] Verify no email send or publication occurred.

### Cutover

- [ ] Resolve or quarantine conflicting ECC 7-Day drafts.
- [ ] Enable ECC writer guard for the reserved audience.
- [ ] Verify the ECC database trigger blocks unauthorized reserved-audience inserts and content mutations.
- [ ] Set workflow authority to Hub.
- [ ] Verify old writer routes return `CONTENT_ENGINE_AUTHORITATIVE`.
- [ ] Enable one-package-at-a-time mode.
- [ ] Enable adapter creation but retain manual send/publish.

### First loop

- [ ] Run one Work job.
- [ ] Casey reviews exact versions.
- [ ] Casey approves exact hashes.
- [ ] Hub creates exactly one ECC draft.
- [ ] Hub creates exactly one Substack queue item.
- [ ] Casey sends/publishes manually.
- [ ] Verify delivery state remains external-system authoritative.
- [ ] Verify tracked links and attribution joins.
- [ ] Verify no unattributed conversion was incorrectly labeled attributed.

---

## 24. Rollback checklist

Rollback is fail-closed and Casey-controlled.

- [ ] Pause new Work job creation.
- [ ] Stop active Work claim/retry processing if safe.
- [ ] Disable ECC and Substack adapter handoffs.
- [ ] Keep `CONTENT_ENGINE_DELIVERY_ENABLED=false`.
- [ ] Mark uncertain external references `needs_reconciliation`.
- [ ] Confirm no Hub-created ECC draft is approved or scheduled.
- [ ] Confirm no pending Substack transfer is being retried blindly.
- [ ] Set both Hub authority and the ECC-side authority mirror back to ECC in a verified order.
- [ ] Re-enable ECC writer routes only after authority is confirmed.
- [ ] Leave Hub versions and audit records intact.
- [ ] Do not delete external drafts or queue items automatically.
- [ ] Reconcile every external reference before a subsequent send/publish.
- [ ] If schema rollback is required, use the verified backup or a forward-fix migration; do not drop V1A tables in production.
- [ ] Record the rollback reason, actor, timestamp, affected job IDs, and external references.

---

## 25. Implementation order after approval

The implementation should proceed in this order, with tests and review after each phase:

### Phase 0 — Security and migration foundation

- Shared migration runner and ledger
- Advisory lock
- Backup/restore verification
- Owner route authentication and CSRF validation
- Proven Work transport authentication, scope checks, transport receipts, and replay handling
- Adapter credential separation
- ECC endpoint authentication prerequisite

### Phase 1 — Core content model

- Actors and channels
- Campaigns
- Briefs and revisions
- Source/proof/offer/audience/CTA snapshots
- Rule resolution
- Context snapshots and hashes

### Phase 2 — Work contract

- Durable Work jobs
- Atomic claim
- Lease token and heartbeat
- Expiry/recovery
- Result validation
- Submission receipts and idempotency

### Phase 3 — Editorial integrity

- Output headers
- Immutable versions
- Evidence manifests
- Approval events
- Exact-version approval
- Review backpressure

### Phase 4 — Existing-system adapters

- Substack external-reference adapter
- ECC operational draft adapter
- ECC writer authority guard
- Idempotency and payload hashes
- Retry and reconciliation states

### Phase 5 — Tracking and feedback

- Deterministic CTA tracking
- ECC `/go/` interaction
- Seven-day last-touch attribution read model
- ECC aggregate reads
- Stale/unavailable states

### Phase 6 — Minimal operator UI

- Today
- Brief Detail
- Review
- Minimal Pipeline

### Phase 7 — Dark launch and acceptance

- Clean and populated migration tests
- Internal shadow job
- Adapter dry run
- One controlled manual production loop
- Attribution propagation proof
- Rollback rehearsal

Only after Phase 7 is reliable may Casey consider adding another channel in a separate specification.

---

## 26. Final implementation gate

Production implementation may begin only after the following are explicitly approved:

1. The actual ChatGPT Work transport-capability spike passes and the proven transport is documented as either the supported Hub action/plugin mechanism or the private GitHub inbox fallback.
2. Work authentication/connection and adapter credentials are separate and scoped; no client-side HMAC secret is assumed.
3. The exact ECC 7-Day Clicker writer paths and cutover guard are confirmed.
4. The Hub-only approved-version handoff rule is confirmed.
5. The exact ECC draft mapping and Substack external-reference design are confirmed.
6. The deterministic tracking fields and ECC `/go/` behavior are confirmed.
7. The controlled attribution propagation test plan is approved.
8. The migration runner, backup, dark-launch, canary, and rollback procedures are approved.
9. The complete automated test plan passes in a non-production environment.
10. The verified Railway PostgreSQL production version is recorded, is at least PostgreSQL 13, and the migration test matrix passes against that exact version.
11. Casey gives explicit approval to begin implementation.

Until then, this document is the implementation contract for review only.
