# WRP Registration API — Synopsis

Easy-to-read summary of [`openapi.json`](openapi.json) (version
0.4.0). The OpenAPI file is authoritative; this file only gives an
overview.

Base URL (placeholder): `https://registry.example.org/api/v1`

## Authentication

Requests send a Keycloak OIDC access token (lifetime ~10 minutes) as
`Authorization: Bearer <token>`. The following endpoints are public
(no token, rate-limited):

* `GET /wrp`, `GET /wrp/{identifier}`, `GET /wrp/check-intended-use`
* `POST /representatives`
* `POST /codes/validate`

On authenticated requests the token is introspected; deactivated
Representatives, and Representatives who are not members of the
Organization involved, get `403`.

## Endpoints

### Public Register (APTITUDE Common Register API)

| Method | Path | Purpose |
|---|---|---|
| GET | `/wrp` | Search/list published registration records |
| GET | `/wrp/{identifier}` | Get the published record with this WRP ID |
| GET | `/wrp/check-intended-use` | Check whether an intended use is registered for WRP `rpidentifier` |

* Responses are JWS-signed statements (`application/jwt`); send
  `Accept: application/json` to get the same payload unsigned.
* `GET /wrp` takes the Common Register API filters (`identifier`,
  `legalname`, `tradename`, `policy`, `entitlement`,
  `usesintermediary`, `isintermediary`, `intendeduseidentifier`,
  `claimpath`, `credentialmeta`, `credentialformat`,
  `providesattestation`) plus extensions `q` (free text),
  `organizationid` and `country`. Pagination uses `cursor`/`limit`;
  the next cursor is in `pagination.next_cursor` and the
  `X-Next-Cursor` header.
* Write methods of the Common Register API are not offered.

### Representatives

| Method | Path | Auth | Purpose |
|---|---|---|---|
| POST | `/representatives` | public | Register (email, first/last name, password); emails a validation code |
| POST | `/codes/validate` | public | Validate an emailed code: registration, email change, or invitation |
| GET | `/me` | token | Profile: Representative data, Organization, all Organization applications, controlled records |
| PATCH | `/me` | token | Edit name and/or email (email change is pending until its code is validated) |

### Organizations

| Method | Path | Purpose |
|---|---|---|
| POST | `/organizations` | Create an Organization (only if the caller has none); caller becomes a member |
| GET | `/organization` | Get the caller's Organization, including its members |
| PATCH | `/organization` | Edit Organization data, including its ID (no review; published records unaffected) |
| POST | `/organization/invitations` | Invite a registered Representative without an Organization, by email |
| GET | `/organization/records` | List published registration records controlled by the Organization |

### Applications

| Method | Path | Purpose |
|---|---|---|
| GET | `/organization/applications` | List applications (filters: `state`, `intent`; `limit`/`offset`) |
| POST | `/organization/applications` | Submit a `create`, `edit` or `revoke` application (starts `pending`) |
| GET | `/organization/applications/{applicationId}` | Get one application, with its state history |
| POST | `/organization/applications/{applicationId}/withdraw` | Withdraw a `pending` application |

## Main flows

1. **Registration:** `POST /representatives` → code emailed →
   `POST /codes/validate` → log in via Keycloak.
2. **Email change:** `PATCH /me` with `email` → `pendingEmail` set,
   code emailed to the new address → `POST /codes/validate`. The old
   address stays in use until then.
3. **Joining an Organization:** either `POST /organizations`, or
   another member calls `POST /organization/invitations` and the
   invitee validates the code (`409` if they already belong to an
   Organization).
4. **Publishing a WRP:** `POST /organization/applications` with
   `intent: create`, a `wrpId`, and `data` → Registrar accepts or
   rejects it offline → on acceptance the record appears in `/wrp`
   and `/organization/records`.

## Applications

Request body (`ApplicationRequest`, discriminated by `intent`):

| `intent` | Fields | Rules |
|---|---|---|
| `create` | `wrpId`, `data` | `wrpId` must equal one of `data.identifier[].identifier` (preferably the EUID) and must not be the WRP ID of a published record (`409`) |
| `edit` | `wrpId`, `data` (complete new data) | Record must be published and controlled by the Organization (`404`); `data.identifier` must still contain `wrpId` |
| `revoke` | `wrpId` | Record must be published and controlled by the Organization (`404`) |

Intended uses in `edit`: entries carrying an existing
`intendedUseIdentifier` are kept (possibly modified); entries without
one are new; existing ones left out are ended (they get `revokedAt`).

State machine (changes are emailed to all Organization members):

```
pending --+-> accepted  (Registrar)
          |-> rejected  (Registrar; rejectionReason is set)
          '-> withdrawn (Representative or Registrar)
```

Applications that are `accepted`, `rejected` or `withdrawn` cannot
change state again.

## Key data types

* **WalletRelyingPartySubmission:** registration data as submitted:
  the APTITUDE Register Data Schema `WalletRelyingParty` without the
  Registrar-provided `registryURI`, intended-use `createdAt`/`revokedAt`,
  and (for new uses) `intendedUseIdentifier`. Required: `identifier`,
  `country`, `providerType`, `policy`, `supportURI`,
  `srvDescription`, `isPSB`, `entitlement`, `supervisoryAuthority`,
  `isIntermediary`, and exactly one of `legalPerson`/`naturalPerson`.
  `intendedUse` is required unless `isIntermediary` is true;
  `providesAttestations` is required for PID/EAA providers.
* **PublishedWalletRelyingParty** (public `/wrp`): the snapshot taken
  when the last create/edit application was accepted, without
  `postalAddress`, plus `wrpId`, `organizationId` and
  `registrationCertificates`.
* **RegistrationRecord** (`/organization/records`, `/me`): `wrpId`,
  `organizationId`, full `data` (including `postalAddress`),
  `registrationCertificates`, `publishedAt`, `updatedAt`.
* **RegistrationCertificate:** WRPRC for one intended use
  (`intendedUseIdentifier`, `format` `jwt`|`cwt`, `value`, `issuedAt`,
  `status` `valid`|`revoked`, `revokedAt`). These are issued by the
  Provider of WRPRC and recorded by the Registrar, so they are
  read-only in this API.
* **Application:** `id`, `intent`, `state`, `wrpId`, `data`,
  `submittedBy`, `submittedByName`, `rejectionReason`, `createdAt`,
  `updatedAt`, `history[]` (`from`, `to`, `at`, `by`).
* **Organization:** `id` (globally unique, chosen by the
  Representative), `name`, optional `legalName`, `address`, `website`,
  `contactEmail`; `OrganizationDetails` adds `members[]`.

## Errors

* Own endpoints: RFC 9457 `application/problem+json`. Common codes:
  `400` invalid input or code, `401` bad/missing token, `403`
  deactivated or not a member, `404` not found, `409` conflict
  (duplicate email/Organization ID/WRP ID, already a member, not
  `pending`), `429` rate limited (`Retry-After`).
* Register endpoints: `{"message": "..."}` with `400`, `404`, `429`.

## Not in this API

Registrar actions are CLI tools that are not exposed over HTTP:
accepting, rejecting and withdrawing applications; revoking records;
viewing logs; activating and deactivating Representatives; inviting
Representatives to, and removing them from, Organizations; and
recording WRPRCs.
