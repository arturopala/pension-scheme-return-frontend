# Pension Scheme Return Frontend – Summary

## Purpose

This service is the web front end that lets registered pension scheme administrators (PSAs) and pension scheme practitioners (PSPs) complete, check and submit the yearly **Pension Scheme Return (PSR)** for a registered pension scheme to HMRC. It is part of the wider "Manage pension schemes" family: users arrive from the scheme dashboard, pick a tax year, and are taken through a task list covering the scheme's basic details, finances, members, member payments and the assets the scheme holds. It also lets users view, change and resubmit returns they have already submitted.

It is a user-facing service with a UI (GOV.UK design system pages). It holds no system of record of its own: in-progress answers live in a short-lived session cache, and every return is read from and written to the pension scheme return backend, which talks to HMRC's tax platform (ETMP). "SIPP" returns (self-invested personal pensions) are handled by a separate frontend; this service only routes users there.

## Main Capabilities

### 1. Return dashboard (overview)

- Lists, for the scheme, the tax years whose returns are outstanding (a notice to file was issued but nothing was started, or only a draft exists) and the returns already submitted in earlier years.
- Each row offers a single action: **Start**, **Continue** (a draft exists) or **View or change** (submitted).
- Uses the backend's return overview (for a fixed window of tax years) and the list of versions per year.
- Routes "SIPP" type returns to the separate SIPP frontend, passing the tax year, version and form bundle number in the session.

### 2. Starting a return and pre-population

- Starting a year creates a fresh set of answers for that user and scheme, records the selected tax year, and keeps a snapshot of the starting state, so that later changes can be detected.
- When pre-population is enabled and the scheme submitted a return for an earlier year, the last submitted return is fetched and carried forward as a starting point. Members, loans, land or property, shares, bonds and other assets are cleaned up so that only items still relevant are kept. The user is then asked to check the carried-forward information.
- An optional "locking" rule (switched off by default) stops a second user from opening a scheme's return while another user has answers in progress for it.

### 3. Completing the return (task list)

The return is made up of sections, each a sequence of question pages with "check your answers" summaries and add, change and remove flows:

- **Basic details**: confirm the return dates or give up to 3 accounting periods, scheme bank account, and active, deferred and pensioner member numbers.
- **Financial details**: cash and asset values at the start and end of the period, and total payments.
- **Members**: up to 300 scheme members, each with a name, date of birth and NINO or a reason for not having one. Members can be added by hand or by **uploading a CSV file** (see 4).
- **Member payments**: employer contributions, unallocated employer contributions, member contributions, transfers in and out, pension commencement lump sums, pension payments, and surrendered benefits.
- **Assets**: loans made, money borrowed, land or property (held, leased and disposed of), shares (held and disposed of), bonds, and other assets. Each asset category allows up to 5,000 entries and up to 50 disposals per item.
- For counterparties (loan recipients, sellers and buyers, employers) the user picks an identity type (individual, UK company, UK partnership or other) and gives the matching identifier (NINO, CRN or UTR) or a reason why there isn't one.
- UK addresses can be found by postcode lookup or entered by hand.
- Every change is saved to the session cache. When anything has changed against the starting snapshot, the return is also saved as a **draft ("compiled") version** in the backend.

### 4. Bulk member upload

- The user downloads a CSV template, uploads the completed file through the platform's file-upload (virus-scanning) service, and the file is then checked line by line.
- Validation covers mandatory names and date of birth (`dd/mm/yyyy`, not in the future, not before 1900), NINO format, NINOs repeated within the file, and a NINO or reason being given (exactly one). A file can hold at most 300 member rows.
- The user sees either a success summary, an error summary listing cell-level errors (e.g. `C3`), or a "too many errors / rows / missing information" page.

### 5. Declaration and submission

- When all sections are complete, an administrator agrees to a declaration. A practitioner must also enter the PSA ID of the administrator who authorised them, and it must match the one held for the scheme.
- On declaration the return is submitted as final. A confirmation email is sent to the user's registered email address, with a delivery-status callback going to the backend, and a submission audit event is raised.
- When the member count has risked a change of reporting obligation (over 99 members), there is a "bypass" route that submits only the basic details.

### 6. Viewing and amending submitted returns

- Users can list submitted versions for a year and open any of them read-only, comparing the current version with the previous one section by section.
- Choosing "view or change" loads the selected version from the backend into the session as the new starting point, and the user can then amend it and resubmit it as a new version.

### 7. Session management

- Keep-alive and sign-out pages extend or clear the user's cached answers for the current scheme. A timeout dialog warns before the session expires.

## Cross-cutting Concerns

- **Authentication and authorisation**: users must be signed in through the government gateway and hold either the PSA or the PSP enrolment. A user with both is asked (through the shared administrator/practitioner choice) which role they are acting in. Every scheme-specific page checks the user's link to the scheme (through the scheme-details service), that the scheme is in an allowed status (open, wound-up or deregistered), and the user's account flags (deceased or returned-letter flags send the user elsewhere, and a de-registered administrator is blocked).
- **Auditing**: events are raised when a return is started, compiled (with task-list status), or submitted (with the full payload), and when a member file is uploaded, downloaded and validated.
- **Validation and transformation**: all input is validated in forms (formats, character sets, lengths, numeric ranges and dates). The session answer model is converted to and from the backend's return model, which carries record versions so the backend can detect unchanged sections.
- **Data protection**: cached answers and upload results can be encrypted at rest (switched off by default). The email-callback URL carries the PSA/PSP ID, PSTR, email, scheme name and user name in encrypted form. Session data expires after one hour, upload records after ten minutes.
- **Feature toggles**: pre-population of a new return from the last submitted one (`prePopulationEnabled`, on by default); single-user locking per scheme (`lockingEnabled`, off by default); and encryption of cached data at rest (`mongodb.encryption.enabled`, off by default).

## Dependencies (generic)

| Kind | Role |
|---|---|
| Identity / auth platform | Sign-in, enrolments (PSA/PSP IDs), internal user id |
| Pension administrator service | Administrator/practitioner details (email, name, flags), role choice for dual-enrolled users |
| Pension scheme registry | Scheme name, PSTR, status, establishers, authorising PSA |
| Pension scheme return backend | Read, compile and submit returns; overview and version history; email-delivery callback |
| File upload (virus scanning) service | Hosted CSV upload, then download of the scanned file |
| Address lookup service | UK postcode-to-address lookup |
| Email service | Submission confirmation email |
| Document store (cache) | Session answers, snapshots, upload state |
| Audit platform | Business audit events |
| Related frontends | Scheme dashboard, administrator/practitioner registration, SIPP return journey, feedback |

## In One Sentence

A GOV.UK frontend where pension scheme administrators and practitioners complete, upload member data for, declare, submit and later amend their scheme's annual Pension Scheme Return. It keeps work in progress in a short-lived cache and stores every version through the pension scheme return backend.
