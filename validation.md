# Pension Scheme Return Frontend – Data Validation

Validation is **form-centric**. Everything a user types goes through Play `Form` mappings (`app/forms/**`), with format regexes, character whitelists, length limits, numeric ranges, date windows and duplicate checks. When a form fails, the page is re-rendered with **400 Bad Request** and GOV.UK error messages. Path and query parameters get much less checking: only the SRN has a strict binder. Indexes are range-checked late (refined types) and throw, and several string parameters (`year`, `taxYear`, `startDate`, `fbNumber`, `version`) are passed on unchecked. Upstream responses are parsed with Play JSON `Reads`. Upstream failures become redirects to the "problem with service" page, or generic 500 error pages. No JSON schemas are bundled and outbound payloads are not checked against a schema; the pension-scheme-return backend validates them.

## Validation layers at a glance

| # | Layer | Mechanism | Where | On failure |
|---|---|---|---|---|
| 1 | Route path binder (SRN) | `PathBindable[Srn]`, regex `^S[0-9]{10}$` | `config.Binders.srnBinder`, `SchemeId.Srn.apply` | 400 (bootstrap client-error page, "Invalid scheme reference number") |
| 2 | Route path binder (identity subject) | `PathBindable[IdentitySubject]`, unknown → `Unknown` | `config.Binders.identitySubjectBinder` | No binding error. The controllers checked redirect `Unknown` to `/unauthorised`. |
| 3 | Typed route params | Play `Int`/`Boolean` binders | `conf/*.routes` | 400 for non-numeric values |
| 4 | Index range | `Int` → `Int Refined OneTo[N]` via `given Conversion` (`IntUtils.refined`) | `utils.IntUtils`, controllers | Throws `UnrefinableIntException` → **500** |
| 5 | Auth / access | Enrolments, scheme association, scheme status, account flags | `IdentifierActionImpl`, `AllowAccessAction` | Redirects (login, register, unauthorised, contact HMRC, …) |
| 6 | Form binding | Play `Form` + `Mappings`/`Constraints`/`Formatters` | `app/forms/**`, controllers' `form(...)` | 400 with the page re-rendered and errors shown |
| 7 | CSV member upload | Header lookup + per-row form validation | `MemberDetailsUploadValidator` | Stored `UploadErrors` / `UploadFormatError` / `UploadMaxRowsError` → error pages (200) |
| 8 | Upload size | upscan `maximumFileSize` (1 MB) | `UpscanConnector.initiate` | upscan redirects back with `errorCode=EntityTooLarge` → form error |
| 9 | Upscan callback body | `Reads[CallbackBody]` keyed on `fileStatus` | `models.CallbackBody`, `UploadCallbackController` | 400 (`withJsonBody`) |
| 10 | Upstream response parsing | Play JSON `Reads`, `execute[Option[T]]` | Connectors | `JsResultException` / `UpstreamErrorResponse` → 500, or redirects (see the error map) |
| 11 | Upstream → session transform | Domain constructors with `require` (`Nino`, `Utr`, `Crn`) inside `Try` | `transformations/*Transformer.transformFromEtmp` | `Future.failed` → 500 |
| 12 | Redirect URL safety | `RedirectUrl` + `OnlyRelative` | `JourneyRecoveryController` | Unsafe `continueUrl` is ignored |

## Route and path parameters

| Parameter | Rule | Code |
|---|---|---|
| `srn` | Must match `^S[0-9]{10}$` | `Binders.srnBinder` → `Srn.apply` |
| `subject` (`IdentitySubject`) | Case-insensitive match on `loan-recipient`, `land-or-property-seller`, `shares-seller`, `asset-seller`; anything else becomes `Unknown` | `IdentitySubject.withNameWithDefault` |
| `index`, `secondaryIndex`, `disposalIndex`, `landOrPropertyIndex`, `shareIndex`, `bondIndex`, `assetIndex`, `memberIndex`, `transferIndex` | Bound as `Int`, then implicitly refined to `1..3` / `1..5` / `1..50` / `1..300` / `1..5000` depending on the page | `IntUtils.given toRefinedN` → `refined` throws `UnrefinableIntException` |
| `page` | `Int`, no range check at binding | Pagination helpers |
| `mode` | Fixed in the routes file (`NormalMode` / `CheckMode` / `ViewOnlyMode`) | – |
| `year` (view-only routes) | Free `String`. Sent as `periodStartDate` (encoded query param) to the backend. | `DataRetrievalETMPAction` |
| `current`, `previous` | `Int`, formatted as `%03d` for `psrVersion`. No range check. `previous == 0` selects the single-version path. | `IdentifyAndRequireData.apply(srn, mode, year, current, previous)` |
| `startDate` (`/:srn/summary/:startDate`) | Free `String`. **Concatenated** into the backend query string. | `SummaryController.onPageLoad` → `PSRConnector.getVersions` |
| `taxYear` (query on `select-start` / `select-continue` / `select-view-and-change`) | Free `String`. `onSelectStart` does `LocalDate.parse(taxYear)` (throws → 500). Elsewhere it is passed on or put in the session. | `OverviewController` |
| `version` (query, `select-start` / `select-continue`) | `String`. For a standard return, `toIntOption`; `None` → redirect `/unauthorised` (logged). | `IdentifyAndRequireData.apply(srn, year, current)` |
| `fbNumber`, `lastSubmittedPsrFbInPreviousYears` (query) | Free `String`, no format check. Sent as `fbNumber` (encoded query param). | `OverviewController`, `ReturnsSubmittedController`, `PrePopulationDataAction` |
| `reportType` (query) | `String`. `"SIPP"` → SIPP redirect; anything else is treated as standard. | `OverviewController` |
| `continueUrl` (query) | `RedirectUrl`, accepted only if relative | `JourneyRecoveryController` |
| `uploadFailed`, `isUkAddress`, `manualOrUpload` | Fixed in routes or typed `Boolean` | – |

⚠️ **Observation:** an out-of-range index such as `/accounting-period/7` (max 3) or `/loan/0` does not give a 404 or 400. The refinement runs inside the action body after the async auth chain, so `UnrefinableIntException` becomes a failed `Future`. `ErrorHandler.onServerError` then logs it as an internal server error and returns the generic **500** page.

⚠️ **Observation:** `onSelectStart` with a malformed `taxYear` (e.g. `?taxYear=2024`) throws `DateTimeParseException` → 500.

## Request headers

| Endpoint | Headers | Validation |
|---|---|---|
| All pages | Session cookie, bearer token (through auth) | Auth retrievals only. No custom inbound headers are required. |
| Upscan callback | `Content-Type: application/json` | Enforced by `parse.json` (wrong content type → 415/400) |
| Upload page | Query `errorCode`, `errorMessage` (from the upscan redirect) | Only `EntityTooLarge` and `InvalidArgument`/`'file' field not found` are recognised. Anything else is ignored. |
| Declaration | `X-Session-ID` (fallback only) | Used as the email `requestId` when `hc.requestId` is missing. Not validated. |

## Request bodies (forms)

Every form posts `application/x-www-form-urlencoded` and is bound through a form provider. On failure the controller returns `BadRequest(view(formWithErrors, …))`, with error message keys from `conf/messages.en`.

### What *is* checked

| Kind | Rule | Code |
|---|---|---|
| Required | `text(requiredKey)` trims and rejects empty values | `Mappings.text` |
| Free text (input) | Regex `^[a-zA-Z0-9\-’`'" \t,.@/&()]+$`, max 35 chars | `InputFormErrors.input`, `Constants.textAreaRegex`, `maxInputLength` |
| Free text (textarea) | Same regex, max 160 chars | `TextFormProvider.textArea`, `maxTextAreaLength` |
| Yes/No with reason textarea | Regex `^[a-zA-Z0-9\-'" \t\r\n,.@/]+$` (a different whitelist) | `YesNoPageFormProvider.textAreaRegex` |
| Person names | `^[a-zA-Z\-' ]+$`, max 35 (name + DOB page). Recipient names `^[a-zA-Z0-9 \-'".@/]+$`. | `NameDOBFormProvider`, `RecipientDetailsFormProvider`, `TextFormProvider.name` |
| Security description | `^[a-zA-Z0-9\-’`'" \t,.@#/&()]+$` | `models.Security` |
| Money | Decimal, at most 2 d.p., ranges such as `0.01..999,999,999.99` or `-999,999,999.99..999,999,999.99` depending on the page | `MoneyFormProvider`, `Formatters`, `Constants.max*Value` |
| Percentages | `-999.99..999.99`. Borrowing `0..100`. | `Constants.minPercentage`/`maxPercentage`, `borrowM*Percentage` |
| Integers | e.g. members `0..99,999,999`, shares `1..9,999,999`, loan period `1..999` months | `IntFormProvider`, `Constants` |
| Dates | Valid day, month and year. Page-specific windows (e.g. within the tax year, not before 1900, not in the future). Accounting periods must not overlap and must fall within the tax year. | `DatePageFormProvider`, `DateRangeFormProvider`, `DateFormErrors.failIf*` |
| Identifiers | NINO (domain regex), UTR, CRN, PSA ID, PSTR, QROPS. See `identifiers.md`. | `Mappings.nino`/`utr`/`crn`/`ninoNoDuplicates`/`validatedPsaId`, `inputRegexPSTR`/`inputRegexQROPS` |
| Duplicate NINO | Not already used by another member | `Mappings.ninoNoDuplicates` |
| Postcode | Chars `^[a-zA-Z0-9 ]+$`, format `^[A-Z]{1,2}[0-9][0-9A-Z]?\s?[0-9][A-Z]{2}\|BFPO\s?[0-9]{1,3}$` (after removing whitespace and upper-casing) | `Mappings.postCode`, `Constants.postcode*Regex` |
| Country | Must be in `location-autocomplete-canonical-list.json` | `Mappings.select` |
| Conditional yes/no | Detail required only when the matching answer is selected | `play-conditional-form-mapping`, `ConditionalYesNo` |
| PSP declaration | PSA ID `^(A[0-9]{7})$`, ≤ 8 chars, equal to the scheme's `authorisingPSAID` (when present) | `TextFormProvider.psaId`, `Constraints.isEqual` |

### What is *not* checked

- Cross-section consistency (e.g. member counts against the number of members listed) is not enforced at form level. It shows up only in task-list status and the "member count over 99" bypass logic.
- When the scheme has no `authorisingPSAID`, the PSP declaration accepts **any** well-formed PSA ID (`isEqual` returns `Valid` if the expected value is empty).
- The CRN regex `^[A-Za-z0-9 ]*$` would accept an empty string. Required-ness comes from `text(requiredKey)`, and length (exactly 8 without spaces) is a separate check.
- Land registry title numbers, bond and asset names, and other counterparty identifiers are free text (whitelist + length only).

### Normalisation and derivation

| Field | Rule | Code |
|---|---|---|
| NINO | Whitespace removed and upper-cased before validation and storage | `Mappings.nino`, `ninoNoDuplicates` |
| UTR | Upper-cased. Stored as entered, including inner spaces; `Utr.value` (used for JSON writes) strips whitespace. | `Mappings.utr`, `models.Utr` |
| CRN | Upper-cased, spaces removed before storage | `Mappings.crn` |
| Postcode | Whitespace removed, upper-cased | `Mappings.postCode` |
| Transfer scheme PSTR / QROPS | Regex checked on the whitespace-stripped value (case-insensitive letters). Written upstream upper-cased without whitespace. | `Mappings.input(key, …)`, `PensionSchemeType.writes` |
| "Other" transfer scheme description | CR/LF replaced with spaces on write | `PensionSchemeType.writes` |
| Establisher name | `first [middle ]last` | `Establisher.reads` |

## CSV member upload

Done by `MemberDetailsUploadValidator.validateCSV`, called from `CheckMemberDetailsFileController.onSubmit` once upscan reports `READY`.

| Rule | Detail |
|---|---|
| Framing | Pekko `CsvParsing.lineScanner()`. Row 1 is the header. Data validation skips the first two rows, the header and the template's example row (`drop(2)`), so the first validated record is CSV row 3. The first column is dropped from every data row. |
| Header lookup | Case-insensitive exact match against `UploadKeys` labels (`First name of scheme member (mandatory)`, …) |
| Missing mandatory column | `UploadFormatError` (stops the stream) |
| NINO and reason | Exactly one must be present. Both or neither → `UploadFormatError`. |
| Names | `NameDOBFormProvider` rules (regex, max 35) |
| Date of birth | Split on `/` into exactly 3 parts, otherwise a format error. Must be ≤ threshold and ≥ 1900-01-01. The threshold is `LocalDate.now()`, because the caller passes `None`. |
| NINO | Domain NINO regex. Must not repeat an earlier row in the same file. |
| No-NINO reason | Textarea regex, max 160 |
| Row limit | More than 300 data rows → `UploadMaxRowsError` |
| Empty / unreadable file | `NoSuchElementException` → `UploadFormatError` |
| Error location | `ValidationError.fromCell(column letter, row + 2)`, e.g. `B3` |
| MIME type / extension | Not checked server-side. The `.csv` accept attribute is client-side only. |
| Download status | Not checked. A non-200 body is still parsed; it is only audited as `Failed`. |

⚠️ **Observation:** `validateNino` maps the error type with `case "memberDetailsNino.error.duplicate"`, but the form is built with the message key `memberDetailsNino.upload.error.duplicate`. Duplicate NINOs are therefore classified as `ValidationErrorType.NinoFormat`, never `DuplicateNino`. The message shown is still correct.

⚠️ **Observation:** the date-of-birth "not in the future" check uses today's date instead of the end of the return period. `getTaxDates` exists but is not used by this caller.

## External validation services

| Service | Request | Decision | Fail-open / closed |
|---|---|---|---|
| Address lookup | `POST /lookup {postcode, filter?}` | The address list is shown for the user to pick from. An empty result is handled by the page. | **Closed**: any upstream error propagates (no recovery) → 500 |
| upscan | Initiate + scan | `READY` → file can be validated. `FAILED` → back to the upload page with an error. | **Closed**: an initiate failure → 500 |
| Scheme details / minimal details | Per request | Status, association and flags decide access | **Closed**: errors other than the mapped 403/404 → 500 |

## Outbound payload validation

- No JSON Schema files exist in this repository, and nothing is checked against a schema before sending. `PsrSubmission` is built by the transformers from already-validated answers and serialised with Play `Writes`.
- Upstream rejections (any non-204) are surfaced as `Left("status: …, body: …")` → `PostPsrException` → redirect to journey recovery. The upstream response body is included in the exception message and logged by `ErrorHandler.logError`. That body may contain payload details, and **no PII masking** is applied.
- The `userName` and `schemeName` headers are trimmed.

## Upstream response validation

| Response | Validation | Failure result |
|---|---|---|
| Scheme details | `SchemeDetails.reads` (status enum, establisher kinds) | `JsResultException` → 500. 404 → `None` → `/unauthorised`. 403 → `/unauthorised`. Other → 500. |
| Minimal details | `Json.reads[MinimalDetails]` | 404 "no match found" / 403 "DELIMITED_PSAID" → mapped `Left`. Other → 500. |
| Session data (role) | `SessionData.reads` (`administrator`\|`practitioner`) | 404 → `None` → role choice page. Bad value → 500. |
| Get standard PSR | `validate[PsrSubmission]` | `JsError` → `JsResultException` → 500. Non-200/404 → `GetPsrException` → journey recovery redirect. |
| PSR → session transform | Domain types (`Nino`, `Utr`, `Crn`) constructed from upstream strings; `require` failures are captured in `Try` | `Future.failed` → 500 |
| Versions / versions-for-years | `validate[Seq[…]]` | `JsError` → 500. 404/503 → empty. Other → `GetPsrException`. |
| Overview | `validate[Seq[OverviewResponse]]` | `JsError` → 500. Any non-200 → `GetPsrException`. |
| Submit | Status only (204) | `PostPsrException` |
| Address lookup | `List[ALFAddressResponse]` | Parse or HTTP error → 500 |
| Upscan initiate | `PreparedUpload` | Error → 500 |
| Email | Status only (202) | `EmailNotSent`, logged, the request continues |

⚠️ **Observation:** `OverviewController.outstandingData` calls `overviewResponse.psrReportType.get`. An overview entry with no `psrReportType` therefore throws `NoSuchElementException` → 500.

## Inbound callbacks

`POST /fileupload/upscan-callback` (unauthenticated):

- The body must have `fileStatus` = `READY` (needs `reference`, `downloadUrl` (a parsable URL), and `uploadDetails{uploadTimestamp, checksum, fileMimeType, fileName, size}`) or `FAILED` (needs `reference` and `failureDetails{failureReason, message}`). Anything else → 400.
- The `reference` is not checked against a pending upload before the update. `findOneAndUpdate(upsert = false)` simply matches nothing for an unknown reference, and the endpoint still returns 200.
- `downloadUrl` is not restricted to the upscan or object-store host. The server later issues `GET {downloadUrl}`. This is mitigated by the reference being an unguessable upscan id.

## Cache and opaque stores

- `user-answers.data` is an opaque JSON object, written only through typed `Page` setters. It is read back with `Reads.optionNoError`, so values that fail to parse are treated as **absent**, never as errors.
- `upload` keys are read with `uploadKeyReads`, which requires `{userId}&&{srn}` with a valid SRN.
- Session values (`SRN`, `prePopulationFlag`, `taxYear`, `version`, `fbNumber`) come from the Play signed session. `isPrePopulation` uses `.toBoolean`, which would throw on a non-boolean value. That value can't be forged because the session is signed.

## Error mapping summary

| Source | Exception / value | HTTP to caller |
|---|---|---|
| SRN binder | `Left("Invalid scheme reference number")` | 400 |
| Non-numeric `Int` path or query | Play binder error | 400 |
| Index out of range | `UnrefinableIntException` | 500 |
| `taxYear` not ISO date (`onSelectStart`) | `DateTimeParseException` | 500 |
| `version` not numeric (continue) | `errorAction` | 303 → `/unauthorised` |
| Form errors | `Form.fold(hasErrors)` | 400 (re-rendered page) |
| No session answers | `DataRequiredAction` | 303 → `/problem-with-service` |
| Missing prerequisite answer | `UsingAnswer` / `getOrRecoverJourney` | 303 → `/problem-with-service` |
| Lock held by another user | `PsrLockedException` | 303 → `/:srn/psr-locked` |
| Backend GET failure | `GetPsrException` | 303 → `/problem-with-service?continueUrl=…` |
| Backend POST rejection | `PostPsrException` | 303 → `/problem-with-service?…&answersSavedDisplayVersion=2` |
| Upstream JSON parse error | `JsResultException` | 500 |
| Scheme details 403/404, bad status | `AllowAccessAction` | 303 → `/unauthorised` (or a specific MPS page) |
| No active session | `NoActiveSession` | 303 → login |
| Upscan callback bad body | `withJsonBody` | 400 |
| Any other exception | `FrontendErrorHandler` | 500 |

## Recommendations

1. Validate path and query strings at the boundary. Give `year`, `taxYear` and `startDate` an ISO-date binder, `fbNumber` a numeric pattern, and `current`/`previous` positive ranges. Return 400 or 404 for out-of-range indexes instead of letting `UnrefinableIntException` surface as a 500.
2. Build every backend query string with the `url` interpolator's `Seq` parameters (as `getStandardPsrDetails` already does). This applies to `getVersions` and `getVersionsForYears`.
3. Fix the duplicate-NINO error-type mapping in `MemberDetailsUploadValidator`, and pass the return-period end date as the DOB threshold.
4. Check the upscan download status before parsing, and restrict `downloadUrl` to the expected upscan host(s).
5. Mask or truncate upstream response bodies before putting them into `PostPsrException` and `GetPsrException` messages that get logged.
6. Replace `psrReportType.get` in the overview with a safe default.
7. Decide whether a PSP declaration should be allowed when `authorisingPSAID` is missing. At present any PSA ID passes.
