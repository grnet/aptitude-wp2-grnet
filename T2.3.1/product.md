# Introduction

This document describes the services providing the Registration
Service and the Registrar roles for a Wallet-Relying Party (WRP)
according to the APTITUDE Onboarding Process.

# High-level requirements / Use case scenarios

The API serves the following use cases:

1. Registration of a human Representative of an Organization that
   participates in APTITUDE.

2. Login of a Representative in order to:
    - Create a new Organization if the Representative does not yet
      belong to an Organization.
    - Edit the Organization that the Representative belongs to. This
      needs no external review.
    - Invite another Representative (code via email) to join an
      existing Organization, to which the Representative belongs. The
      invited Representative must not belong to an Organization.
    - Edit their Representative data, which were entered during
      their registration.

3. A Representative that belongs to an Organization can apply to:

    - Publish a new registration record for a WRP. The information
      follows the APTITUDE Register Data Schema.
    - Revoke an existing published WRP registration record.
    - Edit the details of an existing published WRP registration
      record.

   The corresponding application for all subcases above is always
   created at state "pending". All applications are tied to the
   current Organization that the Representative represents. One
   Organization can apply via its Representatives for many WRPs.

4. A Representative can withdraw any of the pending applications of
   their Organization.

5. A Representative can query their profile:

    - Their data (email address, first/last name)
    - Their Organization
    - The state of all Organization applications
    - The list of all published registration records of WRPs that the
      Organization controls

6. Non-logged-in users can only read the published Register
   (registered WRPs). This conforms to the APTITUDE Common Register
   API.

There are CLI tools used by the Registrar that run in the deployment
and are not exposed to the public. These tools offer the following
functionality:

1. Transition an application from state "pending" to "accepted". No
   further state transitions are possible on this application. The
   transition effects depend on the intent of the application:

    - Registration record creation: A new WRP registration record is
      created in the Register. If another published registration
      record for this WRP ID exists, this operation fails. This marks
      the WRP as controlled by the application's Organization.

    - Registration record revocation: An existing WRP registration
      record is removed from the Register. The WRP must be one
      that the application's Organization controls. All its certificates
      must be revoked (see below).

    - Registration record edit: An existing WRP registration record is
      updated. The WRP must be one that the application's Organization
      controls. Its certificates must be updated (see below).

   The published WRP registration record data must be stable in the
   absence of registration record edits and must come from a data
   snapshot when the create/edit application is accepted.

   If the transition effects cannot be applied (e.g., another
   published registration record with the same WRP ID exists, or the
   WRP registration record to be revoked or edited no longer exists or
   is no longer controlled by the application's Organization), the
   transition is refused and the application remains "pending". The
   Registrar can then reject the application with a corresponding
   rejection reason.

   Registration certificates are issued and revoked by the Provider of
   WRPRCs/WRPACs (SIROS Trust Services), not by this backend:

    - A separate set of certificates is issued for each intended use
      of a published registration record, only for WRPs that are legal
      persons and where a WRPRC is required (CIR 2025/848, Article 8).
    - Certificates are issued after the create/edit application has
      been accepted, so that they carry the intended-use identifiers
      assigned by the Register.
    - After an accepted edit, new intended uses get new certificates,
      modified uses get new certificates and their previous ones are
      revoked, removed uses have their certificates revoked, and there
      are no certificate changes in uses that do not change. If any
      other WRP data change, all certificates are revoked and new ones
      are issued in their place.
    - When a registration record is removed (by an accepted revocation
      application or by tool 4 below), all its certificates are
      revoked.

   The Registrar records issued certificates, and their revocations,
   with tool 10 below. Recorded certificates are visible in the
   published WRP registration record data. Recording certificates is
   not a registration record edit: it does not change the reviewed
   data snapshot.

2. Transition an application from state "pending" to "rejected". No
   further state transitions are possible on this application. The
   transition is accompanied by a rejection reason, which will be
   included in email communications and in application state queries.

3. Transition an application from state "pending" to "withdrawn". No
   further state transitions are possible on this application.

4. Revoke an existing published WRP registration record. All related
   certificates must then be revoked (see tool 1).

5. View all logs (such as all application transitions).

6. Deactivate a Representative account to disable login. This
   communicates with the authentication backend. This must forcefully
   log out the Representative. The state of the applications of the
   Representative is left unchanged.

7. Activate a disabled Representative to permit login. This
   communicates with the authentication backend.

8. Invite a Representative to an existing Organization. Similarly to
   an invitation by the API, this sends an email code to verify the
   email address. This action cannot happen if the invited
   Representative belongs to an Organization.

9. Remove a Representative from an existing Organization. The
   Representative can be invited again to the organization by another
   Representative or by the corresponding CLI tool.

10. Record certificates for a published WRP registration record: add a
    certificate issued by the Provider of WRPRCs/WRPACs for one of its
    intended uses (identified by its intended-use identifier), or mark
    a recorded certificate as revoked.

# State transitions

The following diagram shows the WRP application state transitions:

```
pending --+-> accepted (by the Registrar)
          |
          |-> rejected (by the Registrar)
          |
          |-> withdrawn (by Representative or Registrar)
```

The states of the published WRP registration records follow:

```
create application -> published ---+--> removed
                          ^        |
                          |        |
                          \- edit -/
```

# Implementation properties

## Authentication

* Login functionality is provided by an authentication/OIDC
  service. Access tokens are short-lived.

* The Representative is registered at the authentication backend
  during registration.

* All API requests use the OIDC access token for authentication,
  except for the following endpoints:
  - the public endpoints of the Common Register API
  - the registration endpoint
  - the code validation endpoint, which validates: (a) registration
    email address, (b) changed email address, (c) invitation code

## Logging
* All application state transitions are logged.

* All published registration record transitions are logged: creation
  of new registration record, removal of registration record,
  registration record details edit.

* All actions related to Organizations (create/edit Organization, join
  Organization, invitation to Organization, removal from Organization)
  are logged. In particular, change of Organization IDs should log
  both the new and the old Organization IDs.

* All actions related to Representatives are logged: Representative
  registration, login (de)activation, Representative data edits.

* All certificate changes (such as certificate creation/revocation)
  are logged.

## Notifications

* All application transitions send an email to all Representatives of
  the Organization.

* Editing Organization data sends an email to all Representatives of
  the Organization.

* WRP registration record revocation via CLI notifies by email all
  Representatives of the Organization.

* Representative account activation/deactivation sends an email to all
  Representatives of the Organization.

* A Representative joining an Organization sends an email to all
  other Representatives of the Organization.

* Removing a Representative from an Organization sends an email to all
  Representatives of the Organization, including the removed
  Representative.

* For actions that are related to Organizations, emails are sent to a
  Representative only if the Representative is still a member of the
  Organization at the time that the email is to be sent. The only
  exception to this rule is the removal of a Representative from an
  Organization; in that case, the removed Representative also receives
  an email.

* When WRP certificates are created/revoked, notifications are sent to
  the Representatives of the Organization controlling the WRP.

## Identifiers

* Each Organization has a globally-unique ID, supplied by a
  Representative on Organization creation. The format of the
  identifier is out of the scope of this document. A Representative
  belongs to at most one Organization. A Representative cannot create
  or edit Organization data to have the same ID as an existing
  Organization.

* Each WRP has a globally-unique ID and is controlled by one
  Organization. The format of the identifier is out of the scope of
  this document. A Representative cannot create an application to
  create a WRP registration record with a WRP ID that is the same as
  one found in an existing published WRP registration record.

* The WRP ID is the value of one of the identifiers of the
  registration record (`identifier` in the APTITUDE Register Data
  Schema), preferably the EUID where available. The applicant states
  which identifier is the WRP ID when applying to publish a
  registration record. Edits cannot remove or change that identifier.
  The WRP ID is the identifier used to look up a registration record
  in the APTITUDE Common Register API (`{identifier}` and
  `rpidentifier`).

* The above identifiers are for display purposes and entity
  disambiguation and may or may not be used as object identifiers in
  the underlying storage.

* We use the APTITUDE Register Data Schema. If some part of it is
  unclear, we also consult Annex I in CIR 2025/848, TS5, TS6, CIR
  2026/1730, and ETSI TS 119 475.

## Security

* For API requests that use the access token, the token must be
  introspected and the database must be consulted to ensure that the
  Representative is not deactivated. If the API request involves an
  action that includes an Organization, the database must be consulted
  to ensure that the Representative is a member of the Organization.

* The name/email information are authoritative in the authentication
  service, not in the database. The two must be synchronized on
  Representative data edits and on every token introspection.

* There is a list of all Organizations participating in APTITUDE,
  available to the registration services. Only Organizations named in
  this list can be onboarded.

* Representative registration includes email validation.

* Only Organizations that participate in APTITUDE can have their
  applications accepted. This is part of the manual review when
  accepting an application via the CLI tool.

* The publicly accessible endpoints must be subject to rate limiting.

* Registration, invitation, and email-change codes are single-use,
  hard to guess, and expire after 2 days. Invitation codes are also
  bound to the invited Representative.

* When an invitation code is evaluated, it should be checked that the
  invited Representative does not belong to an Organization.

* When editing an email address, email validation must happen
  again. The old email address should stay in use until the new email
  address is validated. After validation, both the authentication
  service and the database must use the updated email address.

* Invitations are only allowed to registered Representatives. This
  permits discovery of who has registered, which is not a threat for
  the purposes of this document.

* During acceptance of WRP registration, reviewing must ensure that
  the Organization data is correct. The review should happen on a
  given set of data that is the same as the data published. In
  practice, this assumes a mechanism (such as checksums) to validate
  that the published data is the same as the one reviewed.

* WRP IDs can be reused by different Organizations as long as only one
  WRP with the given ID is published.

* Organization data is controlled by Representatives; no Registrar CLI
  support is available.

* Editing Organization data only affects internal bookkeeping. The
  contents of published WRP registration records are not affected.

* A previously used but currently unused Organization ID can be reused
  by another Organization. This cannot happen if there are published
  WRP registration records that mention the previously used
  Organization ID.

* Every time an Organization ID is created or edited, the uniqueness
  of the resulting ID is checked against all other Organization IDs
  and all Organization IDs included in published WRP registration
  records, except the Organization IDs used by the Organization's own
  published registration records. A failed check should prevent the
  creation/use of the new Organization ID.

* Every time a WRP ID is created, the uniqueness of the resulting ID
  is checked against all other published WRP IDs.

# Technical frameworks

* We use Keycloak as an authentication service. Access tokens live for
  ~10 minutes.

* We use PostgreSQL for database storage.

* It is out of scope how registration certificate issuance/revocation
  is performed (automated or manual).

* WRPRC/WRPAC issuance and revocation happens via the SIROS Trust
  Services.

# References

* CIR 2025/848: https://eur-lex.europa.eu/eli/reg_impl/2025/848/oj
* CIR 2026/1730: https://eur-lex.europa.eu/eli/reg_impl/2026/1730/oj/eng
* Specification of common formats and API for Relying Party Registration information (TS5): https://github.com/eu-digital-identity-wallet/eudi-doc-standards-and-technical-specifications/blob/main/docs/technical-specifications/ts5-common-formats-and-api-for-rp-registration-information.md
* Common Set of Relying Party Information to be Registered (TS6): https://github.com/eu-digital-identity-wallet/eudi-doc-standards-and-technical-specifications/blob/main/docs/technical-specifications/ts6-common-set-of-rp-information-to-be-registered.md
* APTITUDE Onboarding Process: https://aptitude-consortium.github.io/wp2-trust-specifications/v1.0/sections/onboarding-process/
* APTITUDE Common Register API: https://aptitude-consortium.github.io/wp2-trust-specifications/v1.0/api/register-api/
* APTITUDE Register Data Schema: https://aptitude-consortium.github.io/wp2-trust-specifications/v1.0/sections/trust-artifacts/#register-data-schema
* ETSI TS 119 475 V1.2.1, "Electronic Signatures and Trust Infrastructures (ESI); Relying party attributes supporting EUDI Wallet user's authorization decisions": https://www.etsi.org/deliver/etsi_ts/119400_119499/119475/01.02.01_60/ts_119475v010201p.pdf
* SIROS Trust Services: https://github.com/APTITUDE-Consortium/aptitude-wp2-deliverables/pull/79/changes
