# WRP Registration Backend — Database Model

PostgreSQL database model for the backend described in
[`product.md`](product.md) and exposed by [`openapi.json`](openapi.json).
Keycloak holds credentials and is authoritative for Representative
names and email addresses; the database keeps a synchronized copy (see
*Synchronization with Keycloak*).

## Overview

```
organizations 1 ──── 0..n representatives          (membership: at most one Organization)
organizations 1 ──── 0..n applications ──── 1..n application_transitions
organizations 1 ──── 0..n registration_records     (control)
registration_records 1 ── 1..n record_versions     (immutable reviewed snapshots)
registration_records 1 ── 0..n intended_uses ── 0..n registration_certificates
applications 0..1 ─── 0..1 record_versions         (accepted create/edit → snapshot)
codes          (registration, email-change and invitation codes)
audit_log      (append-only log of all actions)
notifications  (outbox of emails to send)
```

Design choices:

* **Internal keys vs. business identifiers.** Every table uses a UUID
  primary key. The Organization ID and the WRP ID are business
  identifiers chosen by Representatives; they are ordinary columns,
  because both can be reused over time (Organization IDs after an
  Organization changes its ID, WRP IDs after a record is removed).
* **Registration data as JSONB.** The APTITUDE Register Data Schema
  is deeply nested and evolves with the specifications, so submitted
  and published registration data is stored as `jsonb`, validated by
  the backend against the schemas of `openapi.json`
  (`WalletRelyingPartySubmission`, `WalletRelyingParty`). The parts
  that the backend must enforce or query (WRP ID, intended-use
  identifiers, certificates) are kept in columns.
* **Immutable snapshots.** Published data is never updated in place.
  Each accepted create/edit application produces a new row in
  `record_versions`. A record points to its current version, which
  keeps published data stable between edits.
* **Checksums.** Applications and snapshots carry SHA-256 checksums of
  their canonical JSON (RFC 8785 JSON Canonicalization Scheme). The
  Registrar's accept tool requires the checksum of the data that was
  reviewed, and the snapshot is written only if it matches.

## Enumerated types

```sql
CREATE TYPE application_intent AS ENUM ('create', 'edit', 'revoke');
CREATE TYPE application_state  AS ENUM ('pending', 'accepted', 'rejected', 'withdrawn');
CREATE TYPE record_state       AS ENUM ('published', 'removed');
CREATE TYPE code_kind          AS ENUM ('registration', 'email_change', 'invitation');
CREATE TYPE actor_type         AS ENUM ('representative', 'registrar', 'system', 'anonymous');
CREATE TYPE certificate_format AS ENUM ('jwt', 'cwt');
CREATE TYPE certificate_status AS ENUM ('valid', 'revoked');
```

`citext` is used for email addresses:

```sql
CREATE EXTENSION IF NOT EXISTS citext;
```

## Tables

### organizations

```sql
CREATE TABLE organizations (
    id               uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id  text NOT NULL CHECK (organization_id <> ''),  -- business ID ("id" in the API)
    name             text NOT NULL CHECK (name <> ''),
    legal_name       text,
    street           text,
    city             text,
    postal_code      text,
    country          char(2) CHECK (country ~ '^[A-Z]{2}$'),
    website          text,
    contact_email    citext,
    created_at       timestamptz NOT NULL DEFAULT now(),
    updated_at       timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT organizations_organization_id_key UNIQUE (organization_id)
);
```

* Organizations are never deleted. Organization data is edited only by
  its Representatives (`PATCH /organization`); there is no Registrar
  tool for it.
* The `UNIQUE` constraint covers uniqueness among current Organization
  IDs. The additional rule against Organization IDs in published
  records is described under *Organization ID uniqueness*.

### representatives

```sql
CREATE TABLE representatives (
    id               uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    keycloak_id      uuid NOT NULL UNIQUE,          -- "sub" of the access token
    email            citext NOT NULL,               -- copy of the Keycloak value
    first_name       text NOT NULL,                 -- copy of the Keycloak value
    last_name        text NOT NULL,                 -- copy of the Keycloak value
    email_verified   boolean NOT NULL DEFAULT false, -- registration code validated
    active           boolean NOT NULL DEFAULT true,  -- false: deactivated by the Registrar
    organization     uuid REFERENCES organizations(id),  -- NULL: no Organization
    joined_at        timestamptz,                   -- when the current membership started
    created_at       timestamptz NOT NULL DEFAULT now(),
    updated_at       timestamptz NOT NULL DEFAULT now(),
    synced_at        timestamptz NOT NULL DEFAULT now(), -- last sync from token introspection
    CONSTRAINT representatives_email_key UNIQUE (email),
    CONSTRAINT representatives_membership_ck
        CHECK ((organization IS NULL) = (joined_at IS NULL))
);

CREATE INDEX representatives_organization_idx ON representatives (organization);
```

* Membership is a nullable foreign key, so a Representative belongs to
  **at most one** Organization by construction. Joining sets
  `organization`; removal by the Registrar clears it. Earlier
  memberships are kept in `audit_log`.
* `active` is checked on every authenticated request, together with
  the token introspection result. Deactivation also disables the user
  in Keycloak and ends their sessions there.
* The pending new email address of an email change is not stored
  here. It lives in the unused `email_change` code (see `codes`), and
  the API's `pendingEmail` is derived from that code.

### codes

Registration, email-change and invitation codes.

```sql
CREATE TABLE codes (
    id               uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    kind             code_kind NOT NULL,
    code_hash        bytea NOT NULL UNIQUE,         -- SHA-256 of the code; the code itself is not stored
    representative   uuid NOT NULL REFERENCES representatives(id),  -- registering / changing / invited
    new_email        citext,                        -- email_change only
    organization     uuid REFERENCES organizations(id),  -- invitation only
    created_by_type  actor_type NOT NULL,           -- invitations: representative or registrar
    created_by       uuid REFERENCES representatives(id),  -- inviting Representative, if any
    created_at       timestamptz NOT NULL DEFAULT now(),
    expires_at       timestamptz NOT NULL,          -- created_at + 2 days
    used_at          timestamptz,                   -- single use
    superseded_at    timestamptz,                   -- invalidated by a newer code
    CONSTRAINT codes_kind_fields_ck CHECK (
        (kind = 'registration' AND new_email IS NULL AND organization IS NULL) OR
        (kind = 'email_change' AND new_email IS NOT NULL AND organization IS NULL) OR
        (kind = 'invitation'   AND new_email IS NULL AND organization IS NOT NULL)
    )
);

CREATE INDEX codes_representative_idx ON codes (representative, kind)
    WHERE used_at IS NULL AND superseded_at IS NULL;
```

* Codes are generated with at least 128 bits of randomness, so an
  unsalted SHA-256 hash is enough. Lookup is by `code_hash`.
* A code is valid if `used_at IS NULL AND superseded_at IS NULL AND
  expires_at > now()`. Validation sets `used_at` in the same
  transaction as the code's effect, using
  `UPDATE ... WHERE used_at IS NULL ... RETURNING` so that a code
  cannot be used twice.
* Invitation codes are bound to `representative`. When one is
  validated, the transaction locks that Representative's row
  (`SELECT ... FOR UPDATE`) and checks that `organization IS NULL`
  before setting it.
* A new email change supersedes earlier unused `email_change` codes of
  the same Representative. At most one pending email exists per
  Representative:

  ```sql
  CREATE UNIQUE INDEX codes_one_pending_email_change
      ON codes (representative)
      WHERE kind = 'email_change' AND used_at IS NULL AND superseded_at IS NULL;
  ```

  Expired codes still match this partial index. They are superseded
  when a new email change is requested.
* On validating an email change, the new address must still be free
  (`representatives_email_key`). Keycloak is updated first, then the
  database, in that order.

### applications

```sql
CREATE TABLE applications (
    id                 uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    organization       uuid NOT NULL REFERENCES organizations(id),
    intent             application_intent NOT NULL,
    state              application_state NOT NULL DEFAULT 'pending',
    wrp_id             text NOT NULL CHECK (wrp_id <> ''),
    target_record      uuid,                        -- edit/revoke: the record concerned (FK added below)
    data               jsonb,                       -- create/edit: WalletRelyingPartySubmission
    data_sha256        bytea,                       -- SHA-256 of JCS(data)
    submitted_by       uuid NOT NULL REFERENCES representatives(id),
    submitted_by_name  text NOT NULL,               -- "first last" at submission time
    rejection_reason   text,
    created_at         timestamptz NOT NULL DEFAULT now(),
    updated_at         timestamptz NOT NULL DEFAULT now(),
    decided_at         timestamptz,                 -- time of the final transition
    CONSTRAINT applications_data_ck CHECK (
        (intent = 'revoke' AND data IS NULL AND data_sha256 IS NULL) OR
        (intent <> 'revoke' AND data IS NOT NULL AND data_sha256 IS NOT NULL)
    ),
    CONSTRAINT applications_target_ck CHECK (
        (intent = 'create') = (target_record IS NULL)
    ),
    CONSTRAINT applications_rejection_ck CHECK (
        (state = 'rejected') = (rejection_reason IS NOT NULL)
    ),
    CONSTRAINT applications_final_ck CHECK (
        (state = 'pending') = (decided_at IS NULL)
    )
);

CREATE INDEX applications_organization_idx ON applications (organization, created_at DESC);
CREATE INDEX applications_pending_idx ON applications (state) WHERE state = 'pending';
CREATE INDEX applications_wrp_id_idx ON applications (wrp_id);
```

* Applications belong to the Organization, not to the submitter.
  `submitted_by_name` keeps the name visible after the submitter
  leaves the Organization.
* `data` is the complete submitted data, unchanged after submission.
  For a `create` application, the backend checks on submission that
  `wrp_id` equals one of `data->'identifier'[*]->>'identifier'`. For
  an `edit` application, it checks that this identifier is still
  present.
* State changes happen only from `pending`, with
  `UPDATE applications SET state = $new ... WHERE id = $id AND state = 'pending'`.
  Zero updated rows means that the application is no longer pending.
  A trigger refuses any update of an application that is not
  `pending`, which makes the final states final:

  ```sql
  CREATE FUNCTION applications_freeze() RETURNS trigger AS $$
  BEGIN
      IF OLD.state <> 'pending' THEN
          RAISE EXCEPTION 'application % is in final state %', OLD.id, OLD.state;
      END IF;
      IF NEW.intent <> OLD.intent OR NEW.organization <> OLD.organization
         OR NEW.wrp_id <> OLD.wrp_id OR NEW.data IS DISTINCT FROM OLD.data THEN
          RAISE EXCEPTION 'application % content is immutable', OLD.id;
      END IF;
      RETURN NEW;
  END $$ LANGUAGE plpgsql;

  CREATE TRIGGER applications_freeze BEFORE UPDATE ON applications
      FOR EACH ROW EXECUTE FUNCTION applications_freeze();
  ```

### application_transitions

State history of applications. The API returns it as
`Application.history`.

```sql
CREATE TABLE application_transitions (
    id               bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    application      uuid NOT NULL REFERENCES applications(id),
    from_state       application_state,             -- NULL for the initial "pending"
    to_state         application_state NOT NULL,
    at               timestamptz NOT NULL DEFAULT now(),
    actor_type       actor_type NOT NULL CHECK (actor_type IN ('representative', 'registrar')),
    actor            uuid REFERENCES representatives(id),  -- set when actor_type = 'representative'
    registrar        text,                          -- operator name when actor_type = 'registrar'
    CONSTRAINT application_transitions_actor_ck CHECK (
        (actor_type = 'representative' AND actor IS NOT NULL AND registrar IS NULL) OR
        (actor_type = 'registrar' AND actor IS NULL AND registrar IS NOT NULL)
    )
);

CREATE INDEX application_transitions_application_idx ON application_transitions (application, at);
```

Every state change, including the creation at `pending`, inserts a row
here and a row in `audit_log`, in the same transaction.

### registration_records

One row per publication lifecycle of a WRP: created by an accepted
`create` application and ended by removal. If the same WRP ID is later
published again, that is a new row.

```sql
CREATE TABLE registration_records (
    id                 uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    wrp_id             text NOT NULL,
    organization       uuid NOT NULL REFERENCES organizations(id),  -- controlling Organization
    state              record_state NOT NULL DEFAULT 'published',
    current_version    uuid,                        -- FK added below (circular reference)
    published_at       timestamptz NOT NULL DEFAULT now(),
    updated_at         timestamptz NOT NULL DEFAULT now(),
    removed_at         timestamptz,
    removed_by_application uuid REFERENCES applications(id),  -- accepted revoke application
    removed_by_registrar   text,                    -- or direct revocation by the Registrar
    removal_reason     text,
    CONSTRAINT registration_records_removed_ck CHECK (
        (state = 'published' AND removed_at IS NULL
            AND removed_by_application IS NULL AND removed_by_registrar IS NULL) OR
        (state = 'removed' AND removed_at IS NOT NULL
            AND (removed_by_application IS NULL) <> (removed_by_registrar IS NULL))
    )
);

-- Only one published record per WRP ID.
CREATE UNIQUE INDEX registration_records_published_wrp_id
    ON registration_records (wrp_id) WHERE state = 'published';

CREATE INDEX registration_records_organization_idx
    ON registration_records (organization) WHERE state = 'published';

ALTER TABLE applications
    ADD CONSTRAINT applications_target_record_fk
    FOREIGN KEY (target_record) REFERENCES registration_records(id);
```

* `registration_records_published_wrp_id` enforces the WRP ID rule at
  acceptance: accepting a `create` application fails if another
  published record has the same WRP ID. Submission performs the same
  check in advance so that it can return `409`. Removed records do not
  block the WRP ID, so other Organizations can reuse it.
* `removed` is final. A trigger refuses updates of removed records,
  as for applications.
* `organization` is the Organization that controls the record. It
  does not change when that Organization edits its data, including
  its Organization ID.

### record_versions

Immutable snapshots of published data, one per accepted create/edit
application.

```sql
CREATE TABLE record_versions (
    id                 uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    record             uuid NOT NULL REFERENCES registration_records(id),
    version            integer NOT NULL CHECK (version > 0),
    application        uuid NOT NULL UNIQUE REFERENCES applications(id),  -- accepted create/edit
    organization_id    text NOT NULL,               -- Organization ID at acceptance (published "organizationId")
    data               jsonb NOT NULL,              -- full WalletRelyingParty, incl. Registrar-provided data
    data_sha256        bytea NOT NULL,              -- SHA-256 of JCS(data)
    reviewed_sha256    bytea NOT NULL,              -- checksum confirmed by the Registrar (see Review)
    organization_snapshot jsonb NOT NULL,           -- Organization data as reviewed
    base_sha256        bytea NOT NULL,              -- SHA-256 of JCS(data without intendedUse)
    accepted_by        text NOT NULL,               -- Registrar operator
    created_at         timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT record_versions_record_version_key UNIQUE (record, version)
);

ALTER TABLE registration_records
    ADD CONSTRAINT registration_records_current_version_fk
    FOREIGN KEY (current_version) REFERENCES record_versions(id)
    DEFERRABLE INITIALLY DEFERRED;

CREATE INDEX record_versions_organization_id_idx ON record_versions (organization_id);
CREATE INDEX record_versions_data_gin ON record_versions USING gin (data jsonb_path_ops);
```

* Rows are insert-only. A trigger refuses `UPDATE` and `DELETE`, and
  the application role has no such privileges.
* `data` is the application's `data` plus the Registrar-provided
  fields (`registryURI`, and the `intendedUseIdentifier`, `createdAt`
  and `revokedAt` of intended uses). Intended uses that an edit ended
  stay in `data` with their `revokedAt`.
* `registration_records.current_version` points to the newest version.
  The public Register serves `current_version.data` without
  `postalAddress`, and adds `wrpId`, `organizationId` (from
  `organization_id`) and `registrationCertificates`.
* `base_sha256` lets the Registrar's tools detect whether WRP data
  other than intended uses changed in an edit. If it did, all
  certificates are replaced (see *Registration certificates*).

### intended_uses

The intended uses of each record, with the identifiers assigned by
the Register. Used for certificate bookkeeping and for
`GET /wrp/check-intended-use`.

```sql
CREATE TABLE intended_uses (
    id                       uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    record                   uuid NOT NULL REFERENCES registration_records(id),
    intended_use_identifier  text NOT NULL UNIQUE,  -- Register-wide unique
    created_at               date NOT NULL,         -- "createdAt" (validity start)
    revoked_at               date,                  -- "revokedAt" (ended by an edit or removal)
    first_version            uuid NOT NULL REFERENCES record_versions(id),
    last_modified_version    uuid NOT NULL REFERENCES record_versions(id),
    content_sha256           bytea NOT NULL         -- SHA-256 of JCS(intended use without Registrar fields)
);

CREATE INDEX intended_uses_record_idx ON intended_uses (record) WHERE revoked_at IS NULL;
```

* The identifiers are never reused. They are generated by the backend
  when a create/edit application is accepted.
* On an accepted edit, each submitted intended use is matched by
  `intendedUseIdentifier`:
  * no identifier → a new row;
  * existing identifier with a different `content_sha256` → the row
    is modified (`last_modified_version` is updated);
  * existing row not submitted → `revoked_at` is set.

  These cases correspond to the certificate rules (new, modified and
  removed uses). An identifier that does not belong to the record
  makes the application invalid. Such applications are refused on
  submission.
* On record removal, `revoked_at` is set on all intended uses that are
  still valid.

### registration_certificates

WRPRCs issued by the Provider of WRPRC and recorded by the Registrar
(CLI tool 10).

```sql
CREATE TABLE registration_certificates (
    id                     uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    intended_use           uuid NOT NULL REFERENCES intended_uses(id),
    certificate_identifier text,
    format                 certificate_format NOT NULL,
    value                  text NOT NULL,           -- JWS compact or base64url CWT
    value_sha256           bytea NOT NULL UNIQUE,   -- prevents recording the same certificate twice
    issued_at              timestamptz NOT NULL,
    status                 certificate_status NOT NULL DEFAULT 'valid',
    revoked_at             timestamptz,
    recorded_at            timestamptz NOT NULL DEFAULT now(),
    recorded_by            text NOT NULL,           -- Registrar operator
    revocation_recorded_at timestamptz,
    revocation_recorded_by text,
    CONSTRAINT registration_certificates_revoked_ck CHECK (
        (status = 'valid' AND revoked_at IS NULL AND revocation_recorded_by IS NULL) OR
        (status = 'revoked' AND revoked_at IS NOT NULL AND revocation_recorded_by IS NOT NULL)
    )
);

CREATE INDEX registration_certificates_intended_use_idx ON registration_certificates (intended_use);
```

* Recording a certificate does not create a record version, because it
  does not change the reviewed data snapshot. Certificates are joined
  to the published record when it is served.
* Revoked certificates are kept and published with `status = 'revoked'`.
* The tools refuse to record a certificate for an intended use that
  is ended or whose record is removed, and for records of natural
  persons.
* After an accepted edit or a removal, the tools list the
  certificates that have to be revoked or replaced, using
  `intended_uses` and `base_sha256`. The Registrar records the changes
  once the Provider of WRPRC has made them.

### audit_log

Append-only log of all actions (CLI tool 5).

```sql
CREATE TABLE audit_log (
    id               bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    at               timestamptz NOT NULL DEFAULT now(),
    actor_type       actor_type NOT NULL,
    actor            uuid REFERENCES representatives(id),  -- acting Representative, if any
    registrar        text,                          -- acting Registrar operator, if any
    action           text NOT NULL,                 -- e.g. 'application.accepted'
    organization     uuid REFERENCES organizations(id),
    representative   uuid REFERENCES representatives(id),  -- Representative affected
    application      uuid REFERENCES applications(id),
    record           uuid REFERENCES registration_records(id),
    certificate      uuid REFERENCES registration_certificates(id),
    details          jsonb NOT NULL DEFAULT '{}'
);

CREATE INDEX audit_log_at_idx ON audit_log (at);
CREATE INDEX audit_log_organization_idx ON audit_log (organization, at);
CREATE INDEX audit_log_representative_idx ON audit_log (representative, at);
CREATE INDEX audit_log_application_idx ON audit_log (application);
CREATE INDEX audit_log_record_idx ON audit_log (record);
```

* Entries are written in the same transaction as the action they log.
  `UPDATE` and `DELETE` are refused by a trigger and are not granted
  to any role.
* `details` holds action-specific data. In particular,
  `organization.edited` records the changed fields with old and new
  values, including **both the old and the new Organization ID**.
  Because Organization IDs can change, every entry that concerns an
  Organization also stores the Organization ID that was current at
  that time (`details.organizationId`).
* Actions logged:

  | Area | Actions |
  |---|---|
  | Representatives | `representative.registered`, `representative.email_verified`, `representative.edited`, `representative.email_change_requested`, `representative.email_changed`, `representative.deactivated`, `representative.activated` |
  | Organizations | `organization.created`, `organization.edited`, `organization.invitation_sent`, `organization.joined`, `organization.member_removed` |
  | Applications | `application.submitted` (→ pending), `application.accepted`, `application.rejected`, `application.withdrawn`, `application.accept_refused` (effects not applicable; stays pending) |
  | Records | `record.published`, `record.edited`, `record.removed` |
  | Certificates | `certificate.recorded`, `certificate.revoked` |

### notifications

Outbox of emails. A row is inserted in the same transaction as the
event, and a separate worker sends the emails.

```sql
CREATE TABLE notifications (
    id                    bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    created_at            timestamptz NOT NULL DEFAULT now(),
    template              text NOT NULL,            -- e.g. 'application.rejected'
    recipient             uuid REFERENCES representatives(id),
    recipient_email       citext,                   -- explicit address (codes to a new email)
    organization          uuid REFERENCES organizations(id),  -- membership required at send time
    require_membership    boolean NOT NULL DEFAULT true,
    payload               jsonb NOT NULL DEFAULT '{}',
    audit_entry           bigint REFERENCES audit_log(id),
    sent_at               timestamptz,
    skipped_at            timestamptz,              -- recipient no longer a member
    attempts              integer NOT NULL DEFAULT 0,
    last_error            text,
    CONSTRAINT notifications_recipient_ck CHECK (recipient IS NOT NULL OR recipient_email IS NOT NULL)
);

CREATE INDEX notifications_unsent_idx ON notifications (created_at)
    WHERE sent_at IS NULL AND skipped_at IS NULL;
```

* One row per recipient. Recipients are the Organization's members at
  the time of the event. For a member joining, the joining member is
  excluded.
* Before sending, the worker checks that
  `representatives.organization = notifications.organization` still
  holds when `require_membership` is true. If it does not, the row is
  marked `skipped_at`. The removal notice to a removed Representative
  is inserted with `require_membership = false`, which implements the
  only exception to the membership rule.
* Code emails (registration, email change, invitation) have no
  Organization and are sent to `recipient_email` (the new address for
  email changes). The code itself is placed in `payload` and the
  payload is cleared once the email is sent, so that codes do not
  stay in the database.

## Rules spanning several tables

### Organization ID uniqueness

When an Organization ID is created or changed to `X` for Organization
`O`, the following must be true:

```sql
-- no other Organization uses X
NOT EXISTS (SELECT 1 FROM organizations
            WHERE organization_id = X AND id <> O)
-- no published record of another Organization mentions X
AND NOT EXISTS (SELECT 1 FROM registration_records r
                JOIN record_versions v ON v.id = r.current_version
                WHERE r.state = 'published'
                  AND v.organization_id = X
                  AND r.organization <> O)
```

The first condition is also enforced by
`organizations_organization_id_key`. The second involves two tables,
so it is enforced by the code, inside a transaction that first takes
`pg_advisory_xact_lock(hashtextextended('org-id:' || X, 0))`. Accepting
a `create` or `edit` application takes the same lock for the
Organization ID it writes into the snapshot, so a concurrent change of
Organization ID and an acceptance cannot both use `X`.

Organization IDs mentioned only by removed records, or by older
versions, do not block reuse. This follows `product.md`: the ID can be
reused unless *published* records mention it.

### Review and acceptance

1. The Registrar's tool shows the application and computes
   `reviewed_sha256 = SHA-256(JCS({wrpId, intent, data, organization}))`,
   where `organization` is the current Organization row as JSON. It
   displays the checksum together with the data.
2. The accept command is given the application ID and the checksum
   that was reviewed. In a single transaction it:
   * locks the application, the Organization and, for edit or revoke,
     the target record (`SELECT ... FOR UPDATE`), and takes the
     Organization ID advisory lock;
   * recomputes the checksum and aborts if it differs, for example if
     the Organization data changed after the review;
   * checks the effects: for `create`, that no published record has the
     WRP ID; for `edit`/`revoke`, that the target record is still
     published and controlled by the Organization. If a check fails,
     the transaction is rolled back, `application.accept_refused` is
     logged, and the application stays `pending`;
   * applies the effects (new record and version, new version, or
     removal), updates `intended_uses`, transitions the application
     to `accepted`, and writes the log entries and notifications.
3. The snapshot stores `reviewed_sha256` and `organization_snapshot`,
   so it can later be shown that the published data is the reviewed
   data.

### Removing a Representative from an Organization

This clears `representatives.organization` and `joined_at`, writes
`organization.member_removed`, and inserts notifications for the
remaining members plus the removed one. The Organization's
applications are not affected, including those submitted by the
removed Representative. Unused invitation codes for that Organization
created by the removed Representative stay valid. `product.md` does
not ask to invalidate them.

## Public Register queries

`GET /wrp` and `GET /wrp/{identifier}` read published records with
their current versions:

```sql
SELECT r.wrp_id, v.organization_id, v.data - 'postalAddress' AS data
FROM registration_records r
JOIN record_versions v ON v.id = r.current_version
WHERE r.state = 'published'
  AND ...filters...
ORDER BY r.wrp_id
LIMIT $limit;
```

* Most filters are JSONB containment queries on `v.data`, served by
  `record_versions_data_gin`. Examples:
  * `identifier`: `v.data @> jsonb_build_object('identifier', jsonb_build_array(jsonb_build_object('identifier', $1)))`
  * `entitlement`: `v.data @> jsonb_build_object('entitlement', jsonb_build_array($1))`
  * `isintermediary`: `v.data @> jsonb_build_object('isIntermediary', $1)`
* `organizationid` filters on `v.organization_id`, and `country` on
  `v.data->>'country'`.
* `q` (free text) uses an expression index:

  ```sql
  CREATE INDEX record_versions_fts ON record_versions
      USING gin (jsonb_to_tsvector('simple', data - 'postalAddress', '["string"]'));
  ```

* The cursor is the last `wrp_id` of the page, encoded opaquely
  (keyset pagination). This is possible because `wrp_id` is unique
  among published records.
* Registration certificates are added with a second query on
  `intended_uses` and `registration_certificates` for the records of
  the page.
* `GET /wrp/check-intended-use` resolves the published record by
  `wrp_id`, then checks the given parameters against the intended
  uses in `v.data` whose identifiers have `revoked_at IS NULL` in
  `intended_uses`.

## Synchronization with Keycloak

* Keycloak is authoritative for email, first name and last name. On
  every token introspection the backend compares the token's claims
  with `representatives` and updates the row (and `synced_at`) if they
  differ.
* Edits through the API (`PATCH /me`, email change validation) update
  Keycloak first and then the database. If the database update fails,
  the next introspection repairs it.
* `active` is stored in both. Deactivation sets `active = false` in
  the database first, which blocks the API at once, and then disables
  the Keycloak user and ends its sessions.

## Roles and privileges

| Role | Used by | Privileges |
|---|---|---|
| `wrp_api` | The API service | `SELECT` on all tables; `INSERT`/`UPDATE` on `representatives`, `organizations`, `codes`, `applications` (pending only, enforced by the trigger), `notifications`; `INSERT` on `application_transitions`, `audit_log`. No access to accepting applications or to certificates beyond `SELECT`. |
| `wrp_registrar` | Registrar CLI tools | As `wrp_api`, plus `INSERT`/`UPDATE` on `registration_records`, `intended_uses`, `registration_certificates`, and `INSERT` on `record_versions`. |
| `wrp_mailer` | Notification worker | `SELECT` on `notifications`, `representatives`; `UPDATE` on `notifications`. |

No role has `UPDATE` or `DELETE` on `audit_log`, `record_versions` or
`application_transitions`.

Rate limiting of the public endpoints is done outside the database,
for example in the reverse proxy.

## Mapping to the API

| API object | Source |
|---|---|
| `Representative` | `representatives`; `pendingEmail` from the unused, unexpired `email_change` code |
| `Organization` / `OrganizationDetails` | `organizations` (`id` = `organization_id`); `members` from `representatives` |
| `Application` | `applications`; `history` from `application_transitions` |
| `RegistrationRecord` | `registration_records` + current `record_versions` + certificates |
| `PublishedWalletRelyingParty` | as above, without `postalAddress` |
| `RegistrationCertificate` | `registration_certificates` + `intended_uses.intended_use_identifier` |
| `Profile` | combination of the above for the caller |
