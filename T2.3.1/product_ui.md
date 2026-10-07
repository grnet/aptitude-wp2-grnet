# WRP Registration Portal - Product Definition

## Purpose

The WRP Registration Portal lets organizations that participate in
APTITUDE register their Wallet-Relying Parties (WRPs) for the EUDI
Wallet, and lets anyone look up which WRPs are registered in the
APTITUDE Register.

A WRP is a party that relies on the EUDI Wallet: typically a service
that asks users to present data from their wallet (for example, a bank
checking identity, or a car rental checking a driving licence), but also
providers that issue attestations into wallets, and intermediaries
acting on behalf of other WRPs.

Within the APTITUDE Onboarding Process, the portal is the entry point
through which WRPs submit their registration data; the Registrar
reviews it and publishes it in the Register. The registration data
follows the APTITUDE Register Data Schema, which implements Annex I of
Commission Implementing Regulation (EU) 2025/848 (as amended by CIR
(EU) 2026/1730), and the EUDI technical specifications TS5 and TS6.

## Users

| User | Description | Uses the portal? |
|------|-------------|------------------|
| Visitor | Anyone, not signed in. | Yes: searches the public Register. |
| Representative | A person acting on behalf of an Organization. Signs in to manage the Organization and its WRPs. | Yes: main user. |
| Registrar | The authority that reviews applications and maintains the Register. | No: works with separate internal tools. Representatives see the Registrar's decisions in the portal and by email. |

## Key concepts

- **Representative**: a registered person with a validated email
  address. Belongs to **at most one** Organization at a time.
- **Organization**: the legal entity on whose behalf Representatives
  act. It has an Organization ID chosen by the Representative who
  creates it, plus descriptive data. Any member can edit it at any time,
  without review. An Organization can have many WRPs and many
  Representatives. Only Organizations that **participate in APTITUDE**
  can have their applications accepted.
- **Registration record**: the public record of a WRP in the Register.
  It is identified by its **WRP ID**, which is the value of one of the
  WRP's official identifiers (preferably its EUID) and cannot be changed
  after creation. It is controlled by one Organization.
- **Application**: a request, submitted by a Representative on behalf of
  their Organization, to **publish**, **edit** or **revoke** a
  registration record. Nothing changes in the public Register until the
  Registrar accepts the application. Applications belong to the
  Organization, not to the individual who submitted them, so every
  member sees and can act on all of them.
- **Registration certificate**: a certificate that lets wallets check
  one of the WRP's registered intended uses. Certificates are issued by
  a separate certificate provider after a registration record is
  published or edited (see *Registration certificates*).

## Application lifecycle

```
pending --+-> accepted  (by the Registrar)
          |-> rejected  (by the Registrar, always with a reason)
          |-> withdrawn (by any member of the Organization, or the Registrar)
```

- Every application starts as **pending**.
- **Accepted**, **rejected** and **withdrawn** are final. A rejected
  application cannot be fixed and resubmitted; the Organization submits
  a new application instead.
- The Registrar reviews the submitted data and the Organization data,
  and checks that the Organization participates in APTITUDE.
- Accepting has the following effect:
  - *publish*: a new registration record appears in the Register;
  - *edit*: the registration record is replaced with the new data;
  - *revoke*: the registration record is removed from the Register.
- If the effect is no longer possible (for example, someone else has
  meanwhile published the same WRP ID, or the record to edit was
  revoked), the application stays pending and the Registrar will
  normally reject it with an explanation.

Suggested UI labels: *Pending review*, *Accepted*, *Rejected*,
*Withdrawn*. The rejection reason must be shown prominently on rejected
applications.

## Registration record lifecycle

```
accepted publish application -> published ---+--> removed
                                    ^        |
                                    |        |
                                    \- edit -/
```

- A registration record becomes **published** when its publish
  application is accepted, and its data changes only when an edit
  application is accepted.
- It is **removed** when a revoke application is accepted, or when the
  Registrar revokes it directly. Removal also leads to the revocation of
  all its registration certificates.
- Removal is final. The same WRP ID can later be registered again (by
  any Organization) through a new publish application, which produces a
  new registration record.
- The published data is a snapshot taken at acceptance. **Editing
  Organization data later does not change published records.** The UI
  should say so on the Organization edit screen, and point users to an
  edit application if they want a published record updated.
- Registration certificates are added to a published record some time
  after acceptance, and may later be marked as revoked; this does not
  change the record's data.

## User journeys and screens

### 1. Public Register search (Visitor)

- Search box plus optional filters, for example: identifier (WRP ID or
  any registered identifier), legal name, trade name, entitlement,
  provided attestation type, intended-use identifier, intermediaries,
  country, and Organization ID.
- With no input, the page lists all published registration records.
- Results are paginated ("next page" navigation; the total number of
  results is not known in advance). Each result should show the legal
  name, the trade name and the WRP ID.
- Each result opens a detail page showing the full published record:
  the WRP, its service descriptions, its intended uses, and its
  registration certificates. The postal address is not shown publicly.
- A search with no matches returns an empty list; show "no WRPs match
  your search".
- Only published registration records are visible; removed records and
  applications are never shown to visitors.
- The search may be rate-limited; show a friendly "too many requests,
  try again shortly" message.

### 2. Registration (Visitor → Representative)

- Form: email address, first name, last name, password.
- The portal is meant for Representatives of Organizations that
  participate in APTITUDE; say so on the form.
- After submission, show "check your email": a validation code has been
  sent. The account can be used only after the code is validated.
- If the email is already registered, say so and offer sign-in.

### 3. Code validation

A single "enter/confirm your code" screen handles three kinds of emailed
code. The response tells which kind it was, so the UI can show the right
confirmation:

| Code kind | Outcome to show |
|-----------|-----------------|
| Registration | "Your email is validated. You can now sign in." |
| Email change | "Your new email address is now active." |
| Invitation | "You have joined Organization X." |

Rules: codes work **once** and **expire after 2 days**. An invalid,
expired or used code gets a single generic error. For an email change,
offer to submit the change again; a new invitation must come from a
member of the Organization. An invitation fails if the person has
meanwhile joined another Organization.

Emails will likely contain a link that pre-fills the code; the screen
should accept both a link and manual entry, and must not require being
signed in.

### 4. Sign-in

- Sign-in happens on a separate identity-provider page; the portal
  redirects there and back.
- Sessions are short-lived and renewed automatically; if renewal fails,
  send the user back to sign-in without losing unsaved form data where
  possible.
- A deactivated account cannot sign in, and an active session stops
  working immediately on deactivation. Show a clear "your account has
  been deactivated; contact the Registrar" message.

### 5. Dashboard / profile (Representative)

A single overview shows:

- the Representative's name and email (and a pending new email, if any);
- their Organization, or (if they have none) the two ways forward:
  **create an Organization** or **wait for an invitation**;
- all applications of the Organization with their states;
- all published registration records the Organization controls, with
  their registration certificates.

A Representative without an Organization can only edit their own data,
create an Organization, or accept an invitation.

### 6. Edit own data (Representative)

- First name, last name and email can be changed.
- A new email must be validated by code first. Until then the old email
  stays in use, and the UI should show the new one as "pending
  validation".
- Reject an email already used by another account.

### 7. Create an Organization (Representative without Organization)

- Form: Organization ID, name, and descriptive data (legal name,
  address, website, contact email).
- The Organization ID must be unique, including IDs still mentioned in
  published records of other Organizations. Show a clear "this
  Organization ID is not available" error.
- The creator becomes its first member.

### 8. Organization page (member)

- Shows the Organization data and its members, including whether each
  member's account is active.
- **Edit**: any member can change any field, including the
  Organization ID, with immediate effect and no review. All members are
  emailed about the change. Show a notice that published registration
  records are not affected.
- **Invite**: enter the email of a **registered** person who does not
  belong to any Organization. Distinguish the errors "no account with
  this email" and "this person already belongs to an Organization".
  Invitations expire after 2 days.
- Members cannot leave or remove others in the portal; removal is done
  by the Registrar. A removed Representative can be invited again.

### 9. Applications (member)

- List of all Organization applications, filterable by state and by
  type (publish / edit / revoke), with the WRP ID, submitter name and
  dates. The submitter name is kept even if that person has since left
  the Organization.
- Application detail: type, state, submitted data, rejection reason if
  rejected, and a history of state changes (when, and whether by a
  Representative or the Registrar).
- **Withdraw** button on pending applications only, with confirmation.

### 10. Register a new WRP (publish application)

- The WRP registration form (see *WRP data* below).
- The user enters the WRP's identifiers and chooses which one is the
  **WRP ID** (preferably the EUID). The WRP ID must not match any
  currently published record; show a specific error if it does.
- After submission, show the new application as *Pending review*.

### 11. Edit a published WRP (edit application)

- Start from one of the Organization's published registration records.
- Pre-fill the form with the current published data. The identifier
  used as WRP ID is shown but cannot be changed or removed.
- The application contains the complete new data, not just the changes.
- Intended uses are matched by their intended-use identifier: an
  intended use that keeps its identifier is kept (it may be modified);
  an intended use without an identifier is new; an existing intended
  use left out of the edit is ended (it gets an end date and stays
  visible as ended). The form must therefore carry the identifiers of
  existing intended uses along, and should make clear that deleting an
  intended use ends it.
- The published record stays as it is until the Registrar accepts the
  edit.
- An accepted edit leads to changes in the registration certificates
  (see *Registration certificates*); the review step can tell the user
  which certificates will be replaced.

### 12. Revoke a published WRP (revoke application)

- Start from one of the Organization's published registration records.
- Confirmation step explaining that the record remains public until the
  Registrar accepts the revocation, that removal is final, and that the
  record's registration certificates will be revoked.

## WRP data

The WRP registration form collects the information of the APTITUDE
Register Data Schema. Unlike some EUDI specifications, the schema has no
separate "service" level: the WRP's data, its service descriptions and
its intended uses form a single record.

```
WRP registration record
├── WRP data (identity, contact, policies, entitlements, ...)
├── Supervisory authority
├── Intermediaries used (zero or more)
├── Provided attestations (attestation providers only)
└── Intended uses (one or more, unless the WRP is an intermediary)
    └── Requested attestations (one or more)
        └── Requested attributes
```

A guided, step-by-step form with a review step before submission is
recommended; a single flat form would be very long.

### WRP data

- **Identity** (required): a legal/natural person switch.
  - Legal person: **legal name** (one or more, as in the official
    record), and, for public sector bodies responsible for authentic
    sources, the **legal basis** of their establishment (text and
    language).
  - Natural person: **given name** and **family name**; date and place
    of birth are optional.
- **Identifiers** (at least one): each with a type and a value. Types:
  EUID, LEI, EORI number, VAT number, tax identification number, excise
  number. One of them is chosen as the **WRP ID** (preferably the EUID,
  which should be included whenever the national business register
  provides one). VAT, EORI and tax numbers are usually prefixed with the
  two-letter country code (Greece uses **EL**, not GR); the UI can help
  with this.
- **Country** (required): two-letter country code, or **EU** for
  providers operating in the whole Union.
- **Postal address** (optional): one or more address lines. It is not
  shown in the public Register.
- **Email**, **phone** and **information web pages** (optional, each
  may have several values). Phone numbers are in international format
  (`+` and country code).
- **Trade name** (optional): the user-facing name of the WRP or its
  service, recognisable to wallet users.
- **Support web pages** (required, at least one): helpdesk/support URLs.
- **Service descriptions** (required): one or more descriptions, each
  given in one or more languages.
- **Policies** (required): links to the WRP's policies, each marked as
  privacy policy, terms and conditions, or privacy statement.
- **Public sector body** (required yes/no).
- **Entitlements** (required, multi-select):
  - Service provider
  - Qualified EAA provider
  - Non-qualified EAA provider
  - Public-sector EAA provider
  - PID provider
  - Qualified certificate for e-seal provider
  - Qualified certificate for e-signature provider
  - Remote qualified seal creation device provider
  - Remote qualified signature creation device provider
  - Non-qualified e-signature/e-seal creation provider

  plus optional sub-entitlements (for example, payment service provider
  roles). An attestation provider that also requests attestations
  during issuance selects both *Service provider* and its provider
  entitlement.
- **Provided attestations**: required for PID, qualified EAA,
  non-qualified EAA and public-sector EAA providers. Each with a format
  and the attestation type it issues.
- **Certificate chains** (optional, advanced): X.509 certificate chains
  of the WRP's services. Most users can skip this.
- The provider type is set automatically by the portal.

### Supervisory authority (required)

The competent data protection supervisory authority: its **country**
(required) and, typically, its name, identifiers, address, email, phone
and web pages. Users will often not know this; the UI should offer a
pick-list of national authorities if available.

### Intermediaries

- **Is an intermediary** (required yes/no): whether the WRP acts on
  behalf of other WRPs. An intermediary needs no intended uses.
- **Intermediaries used** (optional): the intermediaries the WRP relies
  on, each with its identifier(s), trade name and the address of the
  register where it is registered (possibly in another Member State). A
  WRP that uses intermediaries cannot itself be an intermediary.

### Intended uses (one or more, unless the WRP is an intermediary)

- **Purpose** (required): multi-language text describing why the data
  is requested.
- **Privacy policy** (required, one or more): link(s) to the privacy
  statement of the intended use (each marked as a privacy statement;
  the UI can set that automatically).
- **Requested attestations** (required, one or more), each with:
  - **format** (for example SD-JWT VC or ISO mdoc);
  - **attestation type** (for example a list of type values for SD-JWT
    VC, or a document type for mdoc; the field changes with the
    format);
  - **requested attributes** (optional): the path of each attribute
    within the attestation, optionally with the allowed values. Only
    the minimum data needed for the purpose may be requested; the UI
    may remind users of this.

Assigned by the Registrar, shown read-only once published:

- an **intended-use identifier**;
- a **valid-from date**, and an **end date** if the intended use is
  ended;
- the address of the Register (**registry URI**) on the record.

### UI considerations

- Multi-language fields (service descriptions, purposes) need a
  language selector per entry and add/remove controls.
- Repeatable groups occur at every level (names, identifiers, contacts,
  policies, entitlements, provided attestations, intermediaries,
  intended uses, privacy policies, requested attestations, requested
  attributes).
- Several fields are conditional (natural vs. legal person, legal basis
  for public sector bodies, provided attestations for attestation
  providers, intended uses for non-intermediaries, attestation type by
  format); hide irrelevant fields rather than leaving them empty.
- Edit applications contain the complete data, so the edit form starts
  from the published record and the review step should highlight what
  changed.

## Registration certificates

Registration certificates let wallets check a WRP's registered
intended uses. They are issued by a separate certificate provider, not
by the portal, and recorded in the Register by the Registrar.
Representatives do not request them separately.

- One certificate per intended use.
- Only for WRPs that are **legal persons**, and only where a certificate
  is required. Natural-person WRPs get no certificates; the UI should
  not suggest otherwise.
- Certificates are issued **after** an application is accepted, so a
  newly published or edited record may show no (or old) certificates
  for a while. Show this as "certificates pending" rather than as an
  error.
- **Publish accepted**: a certificate for each intended use.
- **Edit accepted**:
  - new intended uses get a certificate;
  - modified intended uses get a new certificate, and their previous
    one is revoked;
  - removed intended uses have their certificate revoked;
  - unchanged intended uses keep their certificate;
  - if any other WRP data changed, **all** certificates are revoked and
    replaced.
- **Record removed** (revoke application accepted, or revoked by the
  Registrar): all certificates are revoked.

Certificates are shown in the public Register and in the Organization's
view of its records, per intended use, with their status (valid or
revoked), issue date and, if revoked, revocation date. Revoked
certificates remain listed. Each certificate is encoded either as a JWT
or as a CWT; the UI should offer a download or copy action, and show
the intended use it belongs to. All members are emailed when
certificates are issued or revoked.

## Email notifications

Users receive emails for the events below. The UI can mention them
("all members will be notified") so users know what to expect:

| Event | Recipients |
|-------|------------|
| Any application state change (submitted, accepted, rejected with reason, withdrawn) | All members of the Organization |
| Organization data edited | All members |
| Published registration record revoked by the Registrar | All members |
| Registration certificates issued or revoked | All members |
| A member's account deactivated or reactivated | All members |
| A new member joins | All other members |
| A member is removed | All members, including the removed person |
| Registration, email change, invitation | The person concerned (the code) |

People who have left the Organization no longer receive its emails,
except the notice of their own removal.

## Permissions summary

| Action | Visitor | Representative without Organization | Member of an Organization |
|--------|:-------:|:-----------------------------------:|:-------------------------:|
| Search the public Register and view certificates | ✓ | ✓ | ✓ |
| Register / validate codes | ✓ | ✓ | ✓ |
| Edit own data | | ✓ | ✓ |
| Create an Organization | | ✓ | |
| Accept an invitation | | ✓ | |
| View / edit the Organization, invite others | | | ✓ |
| Submit publish / edit / revoke applications | | | ✓ |
| View the Organization's applications; withdraw pending ones | | | ✓ |

## Error situations the UI must handle

- Not signed in, or session expired → go to sign-in.
- Account deactivated → explain and stop.
- Not a member of an Organization (for example, removed while using the
  portal) → return to the dashboard, which offers create/join options.
- Organization ID or WRP ID not available → field-level error.
- WRP ID not among the entered identifiers → field-level error.
- Invitation target has no account, or already has an Organization →
  specific messages.
- Code invalid, expired or used → one generic message.
- Application no longer pending when withdrawing → refresh and show its
  current state.
- Registration record to edit or revoke no longer published or
  controlled → refresh the list of controlled records.
- Application rejected because the Organization does not participate in
  APTITUDE → show the rejection reason.
- Too many requests → ask the user to retry shortly.

## Out of scope for the portal

- Reviewing, accepting or rejecting applications, revoking records
  directly, deactivating accounts, removing members, and recording
  registration certificates: these are Registrar functions performed
  outside the portal.
- Issuing access certificates and registration certificates, and
  publishing trusted lists: these are handled by other APTITUDE
  services.
- Leaving an Organization or removing other members.
- Correcting Organization data on behalf of an Organization (only its
  members can do that).
