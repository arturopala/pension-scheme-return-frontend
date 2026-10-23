# Pension Scheme Return Frontend – External Dependencies

All outbound HTTP calls use the platform `HttpClientV2` (from `bootstrap-frontend-play-30`, enabled through `HttpClientV2Module`). The `HeaderCarrier` is built from the incoming request and session (`HeaderCarrierConverter.fromRequestAndSession`), so the session id, request id and the user's bearer token are forwarded to platform services.

Base URLs are resolved by `config.Service` from `microservice.services.<name>.{protocol,host,port}` (protocol defaults to `https`) and exposed on `FrontendAppConfig`. Paths are hard-coded in the connectors, except for the upscan initiate path, the upscan redirect templates and the email-callback template, which come from `urls.*` in `application.conf`. Most backend calls set a per-request timeout of `ifs.timeout` (50s).

## Overview

| # | Dependency | Type | Config key | Accessed via |
|---|---|---|---|---|
| 1 | `auth` | Platform auth | `microservice.services.auth` | `AuthConnector` (bootstrap), `IdentifierActionImpl` |
| 2 | `pension-administrator` | Backend (HTTP) | `microservice.services.pensionAdministrator` | `MinimalDetailsConnector`, `SessionDataCacheConnector` |
| 3 | `pensions-scheme` | Backend (HTTP) | `microservice.services.pensionsScheme` | `SchemeDetailsConnectorImpl` |
| 4 | `pension-scheme-return` | Backend (HTTP) | `microservice.services.pensionSchemeReturn` | `PSRConnector`, `FrontendAppConfig.eventReportingEmailCallback` |
| 5 | `upscan` (v2) | Platform file upload | `microservice.services.upscan`, `urls.upscan.*` | `UpscanConnector`, `UploadService`, `upscan.UploadCallbackController` |
| 6 | `address-lookup` | Platform | `microservice.services.address-lookup` | `AddressLookupConnector`, `AddressService` |
| 7 | `email` | Platform | `microservice.services.email`, `email.*` | `EmailConnector` |
| 8 | MongoDB | Datastore | `mongodb.*` | `SessionRepository`, `UploadRepository` |
| 9 | Datastream audit | Platform | bootstrap `auditing.*` | `AuditService` (`AuditConnector`) |
| 10 | Linked frontends (redirects only) | Browser redirects | `urls.*`, `microservice.services.contact-frontend` | `FrontendAppConfig.urls`, `DashboardUtils`, `OverviewController` |

---

## 1. Auth

- **Base URL**: `microservice.services.auth` (local `http://localhost:8500`).
- **Predicate**: `Enrolment("HMRC-PODS-ORG") or Enrolment("HMRC-PODSPP-ORG")`.
- **Retrievals**: `internalId`, `externalId`, `allEnrolments`. The PSA ID is read from identifier `PSAID` and the PSP ID from `PSPID`.
- **Outcomes**: see [Auth section](#auth-actions).

## 2. pension-administrator

- **Base URL**: `microservice.services.pensionAdministrator` (local `http://localhost:8205`).
- **Headers**: the forwarded `HeaderCarrier` (bearer token). Both endpoints are "-self" endpoints, so the backend resolves the user from the auth token.

| Operation | Method & path | Code | Notes / validation |
|---|---|---|---|
| Minimal PSA/PSP details | `GET /pension-administrator/get-minimal-details-self` | `MinimalDetailsConnector.fetch` | Header `loggedInAsPsa: true\|false`. Timeout `ifs.timeout`. Response read as `MinimalDetails` (`email`, `isPsaSuspended`, `organisationName?`, `individualDetails?`, `rlsFlag`, `deceasedFlag`). A 404 whose body contains `no match found` becomes `Left(DetailsNotFound)`. A 403 whose body contains `DELIMITED_PSAID` becomes `Left(DelimitedAdmin)`. Any other failure is logged and re-thrown. |
| Admin/practitioner choice | `GET /pension-administrator/journey-cache/session-data-self` | `SessionDataCacheConnector.fetch` | Response is `Option[SessionData]` (`administratorOrPractitioner`: `administrator`\|`practitioner`). A 404 becomes `None`. Called only for users holding **both** enrolments. |

**Usage**: `AllowAccessAction` calls minimal details on every scheme-scoped request. It supplies the user's email (for the confirmation email), the name used in audits and headers, and the flags used for access decisions.

⚠️ **Observation:** `SessionDataCacheConnector.fetch(cacheId)` takes an id (the caller passes `externalId`) but never sends it. The id is used only in a log message. The "-self" endpoint relies on the auth token instead.

## 3. pensions-scheme

- **Base URL**: `microservice.services.pensionsScheme` (local `http://localhost:8203`).
- **Timeout**: `ifs.timeout`. Responses are read as `Option[SchemeDetails]`, so a 404 becomes `None`.

| Operation | Method & path | Code | Notes / validation |
|---|---|---|---|
| Scheme details (as PSA), API 1444 | `GET /pensions-scheme/scheme/{srn}` | `SchemeDetailsConnectorImpl.details(psaId, schemeId)` | Headers `idNumber: {srn}`, `schemeIdType: srn`, `psaId: {psaId}` |
| Scheme details (as PSP), API 1444 | `GET /pensions-scheme/psp-scheme/{srn}` | `SchemeDetailsConnectorImpl.details(pspId, srn)` | Headers `pspId: {pspId}`, `srn: {srn}` |

`SchemeDetails` reads `schemeName`, `pstr`, `schemeStatus` (one of 8 known values; anything else is a `JsError`), `schemeType.name`, `pspDetails.authorisingPSAID?` and `establishers[]` (company, partnership or individual name). Failures are logged and re-thrown. A 403 `UpstreamErrorResponse` is turned into a redirect to `/unauthorised` by `AllowAccessAction`.

## 4. pension-scheme-return (backend)

- **Base URL**: `microservice.services.pensionSchemeReturn` (local `http://localhost:10700`).
- **Common headers**: the submit and get-return calls use `buildHeaders`, which sets `Content-Type: application/json`, `userName` (trimmed), `schemeName` (trimmed), `srn`, and `requestRole` (`PSA`\|`PSP`). The versions and overview calls send only `srn` and `requestRole`. All calls use timeout `ifs.timeout`.

| Operation | Method & path | Code | Notes / validation |
|---|---|---|---|
| Submit standard PSR (compile or final) | `POST /pension-scheme-return/psr/standard` | `PSRConnector.submitPsrDetails` | Body `PsrSubmission`. 204 → `Right(())`. Any other status → `Left("status: …, body: …")`, which the service turns into `PostPsrException`. |
| Submit pre-populated PSR (compile) | `POST /pension-scheme-return/psr/pre-populated` | `PSRConnector.submitPrePopulatedPsr` | Used when the session flag `prePopulationFlag=true` and the return is not being finally submitted. Same status handling as above. |
| Get standard PSR | `GET /pension-scheme-return/psr/standard/{pstr}?periodStartDate=&psrVersion=` or `?fbNumber=` | `PSRConnector.getStandardPsrDetails` | Throws `IllegalArgumentException` if neither combination is supplied. 200 → `PsrSubmission`, with `JsResultException` if the body doesn't parse. 404 → `None`. Any other status → `GetPsrException`. |
| Versions for one year | `GET /pension-scheme-return/psr/versions/{pstr}?startDate={date}` | `PSRConnector.getVersions` | 200 → `Seq[PsrVersionsResponse]`. 404 and 503 → empty. Any other status → `GetPsrException`. |
| Versions for several years | `GET /pension-scheme-return/psr/versions/years/{pstr}?startDates=…&startDates=…` | `PSRConnector.getVersionsForYears` | Same mapping as above. |
| Overview | `GET /pension-scheme-return/psr/overview/{pstr}?fromDate=&toDate=` | `PSRConnector.getOverview` | 200 → `Seq[OverviewResponse]`. Any other status (including 404) → `GetPsrException`. |
| Email delivery callback (inbound to backend, registered by this service) | `/pension-scheme-return/psr/email-response/{psaOrPsp}/{requestId}/{email}/{psaId}/{pstr}/{reportVersion}/{schemeName}/{taxYear}/{userName}` | `FrontendAppConfig.eventReportingEmailCallback`, `EmailConnector.callBackUrl` | Template `urls.emailCallback`. Email, PSA/PSP ID, PSTR, scheme name and user name are AES-encrypted with `queryParameter.encryption` and URL-encoded. |

**Usage notes**

- Callers: `PsrRetrievalService` (from the `DataRetrievalETMPAction` variants, `PrePopulationDataAction` and the Summary/ReturnsSubmitted controllers), `PsrSubmissionService` (on save in most section CYA pages and on declaration), `PsrVersionsService`, and `PsrOverviewService`.
- **Compile-on-change**: `PsrSubmissionService.submitPsrDetails` compares current answers with the `pure_` snapshot and only POSTs when they differ or when the return is being finally submitted.
- **Error mapping**: `GetPsrException` and `PostPsrException` (`HttpException` 417) are caught by `handlers.ErrorHandler.onServerError` and become a redirect to `/problem-with-service?continueUrl=…&answersSavedDisplayVersion=…`.
- **Audit**: after a successful POST the service sends `PensionSchemeReturnCompiled` (with the task-list status) or `PensionSchemeReturnSubmitted` (with the full payload) as an extended event.
- ⚠️ **Observation:** the `userName` header and the audit field `schemeAdministratorOrPractitionerName` are filled from `PsrBaseService.schemeAdministratorOrPractitionerName`. That returns the **first establisher's name** when the scheme has establishers, and only otherwise the logged-in user's name.
- ⚠️ **Observation:** `submitPsrDetails` returns `Future(None)` without calling the backend when the `pure_` snapshot is missing (for example after the 1h TTL expired for that document) or when the minimal submission, `CheckReturnDatesPage` or `WhichTaxYearPage` cannot be built. `PsaDeclarationController.onSubmit` and `PspDeclarationController.onSubmit` ignore that result. They then send the confirmation email, clear the answers and show the success page, even though **no submission was made**.
- ⚠️ **Observation:** `getVersions` and `getVersionsForYears` build their query strings by string concatenation (`?startDate=$startDate`), not with the `url` interpolator's encoded query parameters. `startDate` reaches `getVersions` from the URL path of `/:srn/summary/:startDate` without validation (see `validation.md`). The PSTR is likewise concatenated into the path before interpolation.
- ⚠️ **Observation:** `getOverview` treats 404 as an error (→ journey-recovery redirect), while the versions endpoints treat 404 as "no data". This is inconsistent behaviour for a scheme with no history.

## 5. upscan (file upload)

- **Base URL**: `microservice.services.upscan` (local `http://localhost:9570`). Initiate path: `urls.upscan.initiate` = `/upscan/v2/initiate`.
- **Max file size**: `microservice.services.upscan.maxFileSize` (MB, default `1`), sent as `maximumFileSize` in bytes.

| Operation | Method & path | Code | Notes |
|---|---|---|---|
| Initiate upload | `POST /upscan/v2/initiate` | `UpscanConnector.initiate` | Body: `callbackUrl` = `{pensionSchemeReturnFrontend base}/fileupload/upscan-callback`, `successRedirect` = `urls.upscan.success-endpoint` formatted with `(srn, "upload-member-details")`, `errorRedirect` = `urls.upscan.failure-endpoint`, `maximumFileSize`. The response gives the `reference`, form `href` and form `fields`. |
| Download scanned file | `GET {downloadUrl}` | `UpscanConnector.download` / `UploadService.stream` | `downloadUrl` comes from the callback stored in Mongo. The body is streamed into the CSV validator. A non-200 status is only audited and is not treated as an error. |
| Upload result callback (inbound) | `POST /fileupload/upscan-callback` | `upscan.UploadCallbackController.callback` | No authentication. Body is `READY` or `FAILED`, matched on `reference`. Updates the `upload` collection. |

Audit events: `PensionSchemeReturnFileUpload` (on upload failure or success seen), `PensionSchemeReturnFileUpscanDownloadCheck`, and `PensionSchemeReturnFileValidationCheck`.

## 6. address-lookup

- **Base URL**: `microservice.services.address-lookup` (local `http://localhost:9022`).

| Operation | Method & path | Code | Notes |
|---|---|---|---|
| Postcode lookup | `POST /lookup` | `AddressLookupConnector.lookup` | Body `{"postcode": …, "filter"?: …}`, `Content-Type: application/json`. The response is read as `List[ALFAddressResponse]` (`id`, `address.lines`, `town`, `postcode`, `country.{code,name}`). Results with no lines are dropped, and the rest are sorted by street, house number and flat (`AddressService`). The connector has no error recovery. |

Used by the land or property postcode lookup pages.

## 7. email

- **Base URL**: `microservice.services.email` (local `http://localhost:8300`).
- **Config**: `email.fileReturnTemplateId` (`pods_pension_scheme_return_submitted`), `email.force` (default `false`), and `queryParameter.encryption.key` for encrypting the callback parameters.

| Operation | Method & path | Code | Notes |
|---|---|---|---|
| Send email | `POST /hmrc/email` | `EmailConnector.sendEmail` | Body `SendEmailRequest{to:[email], templateId, parameters{schemeName, periodOfReturn, dateSubmitted, psaName}, force, eventUrl}`. 202 → `EmailSent`. Any other status, or a connection failure, → `EmailNotSent`; this is logged and never fails the request. |

Callers: `PsaDeclarationController.onSubmit` and `PspDeclarationController.onSubmit`, after the final submission.

- ⚠️ **Observation:** the key `queryParameter.encryption` is not defined in `conf/application.conf`, so it must come from environment config. Because `jsonCrypto` is a `lazy val` evaluated while building the callback URL, before the HTTP call and outside its `recoverWith`, a missing key fails the request with a 500 **after** the return has already been submitted.
- ⚠️ **Observation:** the full callback URL (with encrypted values) is logged at `INFO`.

## 8. Datastore (MongoDB)

`mongodb.uri` (local `mongodb://localhost:27017/pension-scheme-return-frontend`).

| Collection | Repository | Key | Expiry | Purpose |
|---|---|---|---|---|
| `user-answers` | `SessionRepository` | `_id` = `{internalId}{srn}` for the working copy, `pure_{internalId}{srn}` for the starting snapshot, `previous_{internalId}{srn}` for the previous submitted version | TTL index `lastUpdatedIdx` on `lastUpdated`, `mongodb.timeToLiveInSeconds` = 3600s. Every `get` refreshes `lastUpdated` on all three documents. | Journey answers (`data`) |
| `upload` | `UploadRepository` | `id` = `{internalId}&&{srn}` (unique), `reference` (unique) | TTL `lastUpdatedIdx`, `mongodb.upload.timeToLiveInSeconds` = 600s | Upscan status (`InProgress` / `Success(name, mimeType, downloadUrl, size)` / `Failed`) and the CSV validation result |

**Encryption**: `config.Module` binds `CryptoImpl` (AES-GCM via `SymmetricCryptoFactory.aesGcmCryptoFromConfig("mongodb.encryption")`) when `mongodb.encryption.enabled=true`; otherwise it binds a no-op `Crypto`. The encrypted fields are `user-answers.data`, `upload.status` and `upload.result` (crypto-json `Sensitive` wrappers). Keys and `lastUpdated` are always stored in plain text. Encryption is **disabled by default** in `application.conf`.

The locking check (`getBySrnAndIdNotEqual`) runs an unanchored, case-insensitive regex query on `_id` (`.*{srn}.*`). That query cannot use the `_id` index efficiently.

## Auth actions

| Action / guard | Requires | Retrieves / produces |
|---|---|---|
| `IdentifierActionImpl` | Active session. Enrolment `HMRC-PODS-ORG` (`PSAID`) or `HMRC-PODSPP-ORG` (`PSPID`). `internalId` and `externalId` present. When both enrolments are held, a role from `session-data-self`. | `AdministratorRequest` / `PractitionerRequest` (`userId`=internalId, `externalId`, `psaId`/`pspId`). No session → redirect to login (`urls.login` with `continue`). `AuthorisationException`, or a session with no PSA/PSP enrolment → redirect to MPS "you need to register". Dual enrolment with no stored role → "administrator or practitioner" page. |
| `AllowAccessAction(srn)` | Scheme details found for the user and SRN with status `Open`, `Wound-up` or `Deregistered`. Minimal details with `rlsFlag=false` and `deceasedFlag=false`. | `AllowedAccessRequest` (adds `SchemeDetails`, `MinimalDetails`, `srn`). Deceased → MPS "contact HMRC". RLS → update-contact-address page. Delimited PSA → "cannot access de-registered". Upstream 403 or anything else → `/unauthorised`. |
| `DataCheckLockingAction` | When `lockingEnabled`: no other user's answers whose `_id` contains the SRN | Throws `PsrLockedException` → redirect to `/:srn/psr-locked` |
| `DataRetrievalAction` / `DataRequiredAction` / `DataCreationAction` / `DataSavingAction` | – | Load, require, create or save `user-answers` |
| `DataRetrievalETMPAction` (`versionForYear`, `currentAndPreviousVersionForYear`, `fbNumber`) | – | Loads a return version from the backend, transforms it into `UserAnswers` (plus the previous version if any) and saves it |
| `PrePopulationDataAction` | `lastSubmittedPsrFbInPreviousYears` supplied | Fetches the last submitted return and builds pre-populated answers (when `prePopulationEnabled`) |
| `IdentifyAndRequireData` | Composes the above | – |

⚠️ **Observation:** `AllowAccessAction` does not look at `MinimalDetails.isPsaSuspended`. A suspended administrator is not blocked by this frontend.

⚠️ **Observation:** `FrontendAppConfig.urls.pensionPractitioner.updateContactDetails` is built from the key `urls.pension-administrator.updateContactDetails` (`/update-contact-address`) instead of `/update-contact-address-psp`. A PSP with the RLS flag is therefore sent to the wrong path on the practitioner frontend.

## Inbound callers and callbacks

| Inbound | Auth | Notes |
|---|---|---|
| Browser (all `/pension-scheme-return/**` pages) | Government gateway session plus the actions above | Entered from the MPS scheme dashboard (`/:srn/overview`) |
| `POST /fileupload/upscan-callback` | **None** (typical for upscan callbacks) | Registered with upscan on initiate. See `validation.md`. |
| Email delivery events | – | Registered with the email service but delivered to the **backend** (`pension-scheme-return`), not to this service |
| `GET /pension-scheme-return/test-only/:srn/print-user-answers` | Full auth stack | Only in `testOnlyDoNotUseInAppConf.routes` |

## Libraries

From `project/AppDependencies.scala` (Scala 3.7.1, Play 3.0):

| Library | Version | Used for |
|---|---|---|
| `bootstrap-frontend-play-30` | 10.7.1 | HttpClientV2, auth, audit, error handler, `RedirectUrl` |
| `hmrc-mongo-play-30` | 2.13.0 | `PlayMongoRepository`, TTL indexes |
| `crypto-json-play-30` | 8.4.0 | `Sensitive`/`JsonEncryption`, `SymmetricCryptoFactory` |
| `domain-play-30` | 13.0.0 | `uk.gov.hmrc.domain.Nino` (validated NINO type) |
| `play-frontend-hmrc-play-30` | 13.13.0 | GOV.UK / HMRC UI components |
| `play-conditional-form-mapping-play-30` | 3.5.0 | Conditional form mappings (yes/no + conditional input) |
| `tax-year` | 6.0.0 | Tax-year calculations |
| `pekko-connectors-csv` | 1.3.0 | Streaming CSV parsing of uploaded member files |
| `pekko-stream` and related | 1.7.0 | Streaming download and validation |
| `cats-core` | 2.13.0 | `Validated`, `EitherT`, traversals |
| `refined` | 0.11.4 | Bounded indexes (`Max3`, `Max50`, `Max300`, `Max5000`) |

## Local development

- `sm2 -start PSR_ALL` starts this frontend, the backend and the supporting services. The frontend runs on port **10701**.
- Local defaults in `application.conf`: auth 8500, pension-administrator 8205, pensions-scheme 8203, pension-scheme-return 10700, email 8300, address-lookup 9022, upscan 9570, contact-frontend 9250, Mongo 27017.
- Login uses the auth-login-stub (`http://localhost:9949/auth-login-stub/gg-sign-in`). The MPS dashboard is on 8204 and the SIPP frontend on 10703.
- Test-only routes: start with `-Dplay.http.router=testOnlyDoNotUseInAppConf.Routes`.
- `queryParameter.encryption.key` is not in `application.conf`. It must be supplied (e.g. by the service-manager profile) for the declaration email to work.
