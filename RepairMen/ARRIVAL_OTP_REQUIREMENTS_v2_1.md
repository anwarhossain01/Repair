# Arrival OTP — confirmed requirement and implementation contract

18 October 2026 · Addendum to requirements v2.0 · Founder-requested security flow

This is the home-visit arrival check, separate from account sign-in OTP, quote approval, payment authentication and item handover codes. It helps bind an arriving assigned technician to the booked visit; it is not a guarantee of personal safety.

## Customer and technician flow

1. The customer reviews the assigned technician's name, organization and profile when the person arrives.
2. The customer opens the job's arrival check and shows or says the single-use arrival code at the door. Do not encourage sending it before arrival.
3. The assigned technician opens the matching job and enters the code on their own device.
4. The server checks the authenticated technician, active assignment, booked job, code, expiry, attempt limit and unverified arrival state in one transaction.
5. Both apps receive the same authoritative “Arrival verified” event and timestamp. The code is consumed. Diagnosis becomes available.
6. Repair work still requires approval of the exact quote. An OTP match never authorizes a price or payment.

## Numbered requirements

- **ARR-01 — Ownership.** Code is held by the customer for one job/appointment/assigned technician. The technician cannot query or view the expected code.
- **ARR-02 — Generation.** Cryptographically random six-digit code as a proposed usability/security baseline; no sequential codes. Proposed five-minute expiry starts when the customer reveals/requests the code at arrival, rather than at initial booking. Store a protected verifier using a server-held secret and strict access controls; do not expose the code in logs or analytics.
- **ARR-03 — Verification.** Server validates the technician's authenticated active assignment, job/appointment revision and allowed state. Use rate limiting per job, identity and device/network risk; avoid unlimited retry across sessions.
- **ARR-04 — Consumption.** Matching and state transition are atomic. Successful verification consumes the code, records assignment/version/actors/time, appends a job event and outbox event. Concurrent verification cannot succeed twice. Replayed successful command returns the same outcome without another transition.
- **ARR-05 — Failure.** Wrong code leaves arrival unverified and diagnosis locked. Proposed three attempts then temporary lock and support route. Do not reveal the correct code or whether another customer's code exists.
- **ARR-06 — Refresh.** Customer may request a fresh code with throttling. Old code becomes invalid immediately. Rescheduling, cancellation, assignment change or significant booking revision invalidates the current code.
- **ARR-07 — Offline behavior.** No offline match or locally simulated success. Show pending/offline and recover canonical server state. A lost notification cannot become a false successful check; both clients can poll/reconnect.
- **ARR-08 — Assisted exception.** Customer without a working phone/contact/app is directed to support. No technician self-bypass. Any permitted exception follows a reviewed identity/assignment check, customer confirmation, reason and audit evidence; it must be labelled as assisted verification, not falsely as an OTP match. Ordinary diagnosis remains blocked until the approved arrival-verification outcome exists.
- **ARR-09 — Notifications and permissions.** Both participants see the match. Shop dispatch sees verified/pending/failed status and time, never the secret. Support access is case-scoped. Reassigned technicians cannot verify using the previous worker's code.
- **ARR-10 — Data model.** Proposed `arrival_challenges` holds job_id, appointment_revision, assignment_id, customer_id, protected_verifier, expires_at, attempts, locked_until, consumed_at and status. `arrival_verifications` records challenge_id, verified_assignment, method, actor, verified_at and audit reference. Unique active-challenge and verified-visit guards are required. Final SQL follows schema review.

All timer/attempt values are proposed defaults, configurable after security review. A low-entropy six-digit code is protected by authentication, job/assignment binding, short validity and enforced attempt limits; hashing alone does not prevent brute-force attacks.

## Design coverage

- C24: customer sees the current arrival code and assigned professional.
- P14: technician enters it; diagnosis is not available yet.
- E01: customer success confirmation.
- E02: technician success and diagnosis unlock.
- E03: wrong code and remaining attempts.
- E04: expired/unavailable code and assisted route.
- E05: reassigned professional and old-code invalidation.
- E06: arriving person does not match; do not share code.
- E13: offline state; no offline OTP success.
- E20: refreshed customer code.
- E21: attempt-limit lock.
- E24: rescheduled appointment invalidates the earlier code.
- S10/O05/O11: shop status, incident handling and audit trail.

## Acceptance cases

Correct job and assigned worker + valid code succeeds once. Wrong worker, wrong job, expired code, consumed code, cancelled/rescheduled appointment and previous assignment all fail. Refresh invalidates the prior code. Repeated requests do not bypass attempt limits. Two concurrent correct submissions create one verified arrival. Lost push delivery recovers via canonical job state. Customer and technician cannot approve a repair price through this action. No plaintext code appears in logs, support exports or analytics.

## Prototype boundary

The clickable mock accepts the illustrative code 482716 on P14 and navigates to a success/error example. It has no real authentication, OTP backend, payment, messaging or geolocation integration. E20 shows a separate refreshed-code example. These are design demonstrations only.
