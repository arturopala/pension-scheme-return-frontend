# Pension Scheme Return Frontend – Business Identifiers

This document lists every business and technical identifier the frontend handles. For each one it covers where the identifier comes from, where it is used, and which rules apply. Validation rules are compared across three sources:

- **Local**: binders, regexes, form mappings and normalisers in this repository.
- **Library**: domain wrapper types from dependencies that enforce a format on construction. Only `uk.gov.hmrc.domain.Nino` (`domain-play-30` 13.0.0) is used. Its constructor runs `require(Nino.isValid(…))`; this was confirmed with `javap`.
- **Contract**: formats published by upstream APIs. **This repository bundles no JSON or OpenAPI schemas**, so the contract column records only what can be inferred from upstream paths and models. Formal enforcement happens in the `pension-scheme-return` backend and ETMP.

Terminology: the scheme is identified by the **SRN** (Scheme Reference Number), which the code also calls `schemeId` or `Srn`, and by the **PSTR** (Pension Scheme Tax Reference). The code uses **fbNumber**, `reportFormBundleNumber` and "form bundle number" for the same thing. `psrVersion`, `fbVersion` and `reportVersion` are the return version number (zero-padded to 3 digits on the wire). `userId` in the code is the auth `internalId`.

## Summary table

| Identifier | Identifies | Format (contract) | Local validation | Enters via | Main uses |
|---|---|---|---|---|---|
| SRN | Registered pension scheme | `S` + 10 digits (inferred) | `^S[0-9]{10}$` binder | Path `/:srn/…` (untrusted), session `SRN` | Access check, cache keys, backend `srn` header, scheme-details path, redirects to MPS/SIPP |
| PSTR | Scheme for tax purposes | 8 digits + 2 letters (inferred) | None (trusted upstream) | Scheme-details response | Backend URL path segment, `reportDetails.pstr`, audit, encrypted in email callback |
| PSA ID | Scheme administrator | `A` + 7 digits | None for the auth value | Auth enrolment `HMRC-PODS-ORG/PSAID` (trusted) | `psaId` header to pensions-scheme, `submitterId`, audit, encrypted in email callback |
| PSP ID | Scheme practitioner | 8 digits (inferred) | None | Auth enrolment `HMRC-PODSPP-ORG/PSPID` (trusted) | `pspId` header, `submitterId`, audit, encrypted in email callback |
| Authorising PSA ID | PSA who authorised a PSP | `A` + 7 digits | `^(A[0-9]{7})$`, ≤ 8, must equal upstream value | Scheme-details `pspDetails.authorisingPSAID` (trusted) + PSP form input (untrusted) | Declaration check, `psrDeclaration.optAuthorisingPSAID` |
| internalId | Signed-in user (per credential) | Opaque | None | Auth retrieval (trusted) | Mongo keys, upload key, locking |
| externalId | Signed-in user | Opaque | None | Auth retrieval (trusted) | Logged only (`SessionDataCacheConnector`) |
| Form bundle number | One return version in ETMP | Numeric string (inferred) | None | Backend versions response (trusted) → query `fbNumber` / `lastSubmittedPsrFbInPreviousYears` (untrusted round-trip) | `GET …/psr/standard/{pstr}?fbNumber=`, session `fbNumber` (SIPP) |
| Return version | Version within a tax year | `000`–`999` | `toIntOption` for `version`; `Int` for `current`/`previous` | Backend data, path, query | `psrVersion` query, `fbVersion` in submission, email callback |
| Period start / tax year | Return period | ISO date | `LocalDate.parse` in some places only | Path `year`/`startDate`, query `taxYear` | `periodStartDate`, `startDate(s)`, `fromDate`/`toDate` queries |
| NINO | Member / individual counterparty | HMRC NINO | Domain regex (local + library) | Form input, CSV, upstream PSR | `memberDetails.personalDetails.nino`, `idNumber` |
| UTR | UK partnership counterparty | 10 digits | `^\d{5}\s*?\d{5}` | Form input, upstream PSR | `idNumber`, employer `value` |
| CRN | UK company counterparty | 8 alphanumerics | `^[A-Za-z0-9 ]*$` + 8 chars without spaces | Form input, upstream PSR | `idNumber`, `optCrnNumber`, employer `value` |
| Transfer scheme PSTR | Registered scheme transferred to or from | `^[0-9]{8}[A-Za-z]{2}$` | Same regex (whitespace removed) | Form input | `transferSchemeType.value` |
| QROPS reference | Overseas scheme | `^[Qq][0-9]{6}$` | Same | Form input | `transferSchemeType.value` |
| Upscan reference | One file upload | Opaque UUID | None | upscan initiate (trusted), callback body (untrusted) | `upload.reference` (unique index) |
| Upload key | User + scheme upload slot | `{internalId}&&{srn}` | SRN re-validated on read | Derived | `upload.id` (unique index) |
| User-answers id | User + scheme journey | `{internalId}{srn}` with optional `pure_`/`previous_` prefix | None | Derived | `user-answers._id` |
| Address lookup id | Address record | Opaque | None | Address-lookup response | Stored in `Address.id` |
| Request id | HTTP request (for the email callback) | Opaque | None | `hc.requestId`, else header `X-Session-ID` (untrusted) | Email callback path segment |

## Scheme identifiers (SRN, PSTR)

**SRN**
- **Issuer**: the pension scheme registry. Users arrive with it in the URL from the MPS dashboard.
- **Sources**: the route path on almost every page (untrusted, bound by `Binders.srnBinder`). The Play session key `SRN` is written by the overview page and re-validated with `Srn.fromSession`; an invalid value falls back to `""`.
- **Authorisation**: `AllowAccessAction` fetches scheme details for (PSA ID or PSP ID, SRN) from `pensions-scheme`. A missing record (404) or 403 → `/unauthorised`. The scheme status must be Open, Wound-up or Deregistered. The **path SRN is the one that gets authorised**, and the same `request.srn` is used downstream. Keep-alive and sign-out, however, use the **session** SRN, which is not authorised but only builds the user's own cache key.
- **Uses**: the `srn` header on every backend call; the path `/pensions-scheme/scheme/{srn}` or `/psp-scheme/{srn}` and the headers `idNumber`/`srn`; `user-answers._id` and `upload.id`; the locking regex; redirects to `manage-pension-schemes/pension-scheme-summary/{srn}` and SIPP `/{srn}/…`; upscan success and failure redirect URLs.
- **Mismatches**: none between binder and use.

**PSTR**
- **Issuer**: HMRC. Read from `SchemeDetails.pstr` (trusted) and never accepted from the user.
- **Uses**: URL path in `/psr/standard/{pstr}`, `/psr/versions/{pstr}`, `/psr/versions/years/{pstr}` and `/psr/overview/{pstr}`. Also `ReportDetails.pstr` in submissions, `schemeTaxReference` in all audit events, and the encrypted email-callback segment.
- ⚠️ **Observation:** the PSTR is concatenated into the URL string **before** the `url"…"` interpolator, so it is not path-encoded. The risk is low because the value is trusted, but it would break on unexpected characters.

## User and enrolment identifiers

| Identifier | Trusted source | Untrusted source | Uses | Validation |
|---|---|---|---|---|
| PSA ID | Enrolment `HMRC-PODS-ORG` identifier `PSAID` | – | `psaId` header (API 1444), `submitterId` when PSA, `psaOrPspId` in audits, email callback | None locally. `models.PensionSchemeId.PsaId` is a plain case class. |
| PSP ID | Enrolment `HMRC-PODSPP-ORG` identifier `PSPID` | – | `pspId` header, `submitterId` when PSP, audits, email callback | None locally |
| Role (PSA/PSP) | Derived from the enrolment; for dual enrolment, from `session-data-self` | – | `requestRole` header, `submittedBy`, `credentialRole` audit, email template | Enum read (`administrator`\|`practitioner`) |
| Authorising PSA ID | `SchemeDetails.authorisingPSAID` | PSP declaration form | `PsrDeclaration.optAuthorisingPSAID` (taken from scheme details, **not** from the form) | Form: `^(A[0-9]{7})$`, ≤ 8, `isEqual(authorisingPSAID)`. Skipped when the upstream value is absent. |
| internalId | `Retrievals.internalId` | – | `user-answers._id`, `upload.id`, locking exclusion | None |
| externalId | `Retrievals.externalId` | – | Only passed to `SessionDataCacheConnector.fetch` (logged, not sent) | None |

- All user identifiers come from auth retrievals. No PSA/PSP ID is read from headers, query or body for authorisation.
- The PSP-entered authorising PSA ID is used only as a confirmation check. The submitted value comes from the trusted upstream record.

## Return identifiers (form bundle number, version, period)

- **Issuer**: ETMP, via the backend versions and overview responses (`reportFormBundleNumber`, `reportVersion`, `startDate`).
- **Round-trip through the browser**: the overview and "returns submitted" pages embed `fbNumber`, `taxYear`, `version`, `reportType` and `lastSubmittedPsrFbInPreviousYears` into links. Those values come back as **untrusted** query or path parameters.
- **Authorisation**: every backend call is scoped by the PSTR taken from the authorised scheme details, together with the `srn` header. A tampered `fbNumber` can only point at another version that the backend resolves for **that PSTR**, assuming the backend enforces PSTR ↔ fbNumber consistency; this repository cannot confirm that.
- **Transformations**: `current`/`previous` are `Int` values formatted with `%03d`, so a negative or huge value is sent as-is (e.g. `-01`). The `fbVersion` of the loaded return drives fetching of the previous version (`fbVersion - 1`). `defaultFbVersion = "000"` is used before the first version exists.
- **Session**: SIPP hand-off writes `taxYear`, `version` and `fbNumber` into the Play session.

| Parameter | Where | Local rule | Sent upstream as |
|---|---|---|---|
| `fbNumber` | query `select-view-and-change`, `view-change/select-submitted-returns` | none | `fbNumber` query (encoded) |
| `lastSubmittedPsrFbInPreviousYears` | query `select-start` / `select-continue` | none | `fbNumber` query (encoded) for pre-population |
| `year` | path (view-only routes, `select-submitted-returns-to-view`) | none | `periodStartDate` query (encoded) |
| `taxYear` | query `select-start` / `select-continue` / `select-view-and-change` | `LocalDate.parse` on start only | `periodStartDate` (continue) |
| `startDate` | path `/:srn/summary/:startDate` | none | `startDate` query (**not encoded**, string concatenation) |
| `version` / `current` / `previous` | query / path | `toIntOption` / `Int` | `psrVersion` |

## Party and tax identifiers

| Identifier | Frontend key(s) (pages) | Upstream key (per flow) | Local rule | Contract rule |
|---|---|---|---|---|
| NINO (member) | `MemberDetailsNinoPage` / `NoNINOPage`, CSV column `Member National Insurance number` | `membersPayments.memberDetails[].personalDetails.nino` / `reasonNoNINO` | `Nino.isValid` after removing whitespace and upper-casing. No duplicates across members (form) or rows (CSV). | Library: `[[A-Z]&&[^DFIQUV]][[A-Z]&&[^DFIQUVO]] ?\d{2} ?\d{2} ?\d{2} ?[A-D]`, excluding prefixes BG, GB, NK, KN, TN, NT, ZZ. Backend: no schema bundled. |
| NINO (individual loan recipient, seller or buyer) | `IndividualRecipientNinoPage`, `IndividualSellerNiPage`, `SharesIndividualSellerNINumberPage`, `OtherAssetIndividualSellerNINumberPage`, `IndividualBuyerNinoNumberPage`, `AssetIndividualBuyerNiNumberPage` | Loans: `recipientIdentityType.idNumber`. Land, shares and other assets: `optPropertyAcquiredFrom.idNumber` (held and disposed). The no-ID reason goes to `reasonNoIdNumber`. | Same as the member NINO (no duplicate check) | Same |
| UTR | `PartnershipRecipientUtrPage` (by `IdentitySubject`), `PartnershipEmployerUtrPage`, `PartnershipBuyerUtrPage` (land, shares, other assets) | `idNumber`. Employer contributions: `employerType{employerType:"UKPartnership", value\|reason}`. | `^\d{5}\s*?\d{5}` on the upper-cased input; stored with spaces; `value` strips whitespace on write | Inferred: 10 digits |
| CRN | `CompanyRecipientCrnPage` (by `IdentitySubject`), `EmployerCompanyCrnPage`, `SharesCompanyCrnPage`, `CompanyBuyerCrnPage` (land, shares), `AssetCompanyBuyerCrnPage` | `idNumber`. Shares: `shareIdentification.optCrnNumber` / `optReasonNoCRN`. Employer: `employerType{employerType:"UKCompany", value\|reason}`. | `^[A-Za-z0-9 ]*$` and exactly 8 chars without spaces; spaces removed and upper-cased on store | Inferred: 8 alphanumerics |
| Transfer PSTR | `TransferringSchemeTypePage`, `ReceivingSchemeTypePage` (`registeredPS`) | `transferSchemeType{key:"registeredPS", value}` | `^([0-9]{8}[A-Za-z]{2})$` on the whitespace-stripped value | Written upper-cased without whitespace |
| QROPS ref | Same pages (`qualifyingRecognisedOverseasPS`) | `transferSchemeType{key:"qualifyingRecognisedOverseasPS", value}` | `^([Qq][0-9]{6})$` on the whitespace-stripped value | Written upper-cased |

**Normalisation algorithms**
- NINO: `s.filterNot(_.isWhitespace).toUpperCase`. The library regex also allows single spaces.
- UTR: `toUpperCase` only. Inner whitespace is kept in the session answers and removed by `Utr.value` when written to JSON.
- CRN: `toUpperCase.replaceAll(" ", "")`.

**Rule mismatches and constructor `require`s**
- `models.Utr` and `models.Crn` constructors `require` their regex, but the CRN **length** check exists only in the form. An upstream CRN of any length passes `Crn(id)` when the session answers are rebuilt from ETMP, and an empty string passes too.
- When answers are rebuilt from an upstream return, the transformers construct `Nino(id)`, `Utr(id)` and `Crn(id)` directly. An upstream value that breaks the local or library rule (e.g. a historical NINO with a now-disallowed prefix, or a UTR with a `K` suffix) fails the whole `Try`, and the user gets a **500** when opening the return. The failure is not limited to that one field.
- Shares transformer: `optReasonNoCRN.get` assumes a reason is present whenever the CRN is absent. Missing both also fails the transform.
- The UTR regex rejects `K`-suffixed 13-character UTRs and accepts only exactly 10 digits. That is consistent with partnership UTRs, but it is the same `Utr` type used everywhere.

## Technical identifiers

| Identifier | Format | Lifecycle | Notes |
|---|---|---|---|
| `user-answers._id` | `{internalId}{srn}`, `pure_{…}`, `previous_{…}` | Created on start, view-and-change or continue. TTL 1h, refreshed on each read. Cleared on sign-out (using the session SRN) and after submission. | No separator between internalId and SRN. Locking uses an unanchored regex `.*{srn}.*` across all users' documents. |
| `upload.id` (UploadKey) | `{internalId}&&{srn}` | Replaced on each upload initiate. TTL 10 min. | Reads validate the SRN part (`asSrn`) |
| `upload.reference` | upscan UUID | Set on initiate, matched on callback | Unique index. The callback does not check that the reference belongs to a pending upload; an unknown reference updates nothing and still returns 200. |
| Email `requestId` | `hc.requestId` or `X-Session-ID` header | Per submission | Put into the callback path **unencrypted** and not URL-encoded |
| Address `id` | Address-lookup UPRN-like id | Stored in answers | Not validated |

## Validation coverage matrix

| Identifier | Format checked locally? | Checked against the authenticated user? | Encoded when put in a URL? |
|---|---|---|---|
| SRN (path) | ✅ binder | ✅ scheme-details lookup with the PSA/PSP ID | ✅ (validated charset) |
| SRN (session, keep-alive/sign-out) | ✅ `Srn.fromSession` | ⚠️ not authorised, only builds the user's own cache key | n/a |
| PSTR | ❌ (trusted upstream) | ✅ from authorised scheme details | ⚠️ concatenated before interpolation |
| PSA ID / PSP ID | ❌ | ✅ from enrolment | n/a (headers) |
| Authorising PSA ID (form) | ✅ regex + equality | ⚠️ equality skipped when upstream is empty | n/a |
| Form bundle number | ❌ | ⚠️ only indirectly (backend scoped by PSTR) | ✅ query `Seq` |
| `year` / `taxYear` | ⚠️ parsed only in `onSelectStart` (500 on failure) | ⚠️ indirectly via PSTR | ✅ query `Seq` |
| `startDate` (summary) | ❌ | ⚠️ indirectly via PSTR | ❌ concatenated into the query string |
| `version` / `current` / `previous` | ⚠️ numeric only, no range | ⚠️ indirectly via PSTR | ✅ |
| Path indexes | ⚠️ range enforced by throwing (500) | n/a (scoped to the user's own answers) | n/a |
| NINO / UTR / CRN (forms) | ✅ | n/a | n/a (body only) |
| NINO / UTR / CRN (from upstream) | ⚠️ enforced by constructor `require` → 500 | n/a | n/a |
| Transfer PSTR / QROPS | ✅ | n/a | n/a |
| Upscan reference (callback) | ❌ | ❌ (unauthenticated by design) | n/a |
| Email `requestId` | ❌ | ❌ | ❌ |

## Recommendations

1. Encode the PSTR and `startDate` with the `url` interpolator (path segment and `Seq` query params), and validate `startDate`, `year` and `taxYear` as ISO dates at the route boundary.
2. Add format checks for `fbNumber` and version parameters (digits, positive range), and return 400 or 404 instead of 500 for out-of-range path indexes.
3. When rebuilding answers from ETMP, construct `Nino`/`Utr`/`Crn` defensively (e.g. fall back to raw text or flag the item) so that one legacy identifier cannot make a whole return impossible to open.
4. Align the CRN rules: enforce the 8-character length in `Crn.isValid`, or accept upstream values without the form-level length.
5. Encrypt or URL-encode the `requestId` in the email callback, and don't fall back to the client-supplied `X-Session-ID` header.
6. Anchor and narrow the locking query (e.g. store the SRN as its own indexed field) instead of using `.*{srn}.*` on `_id`.
7. Consider checking `MinimalDetails.isPsaSuspended` in `AllowAccessAction` if suspended administrators should not be able to file.
