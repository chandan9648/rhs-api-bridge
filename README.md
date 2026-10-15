# RestoreHealth -- Multi-Vendor Integration Bridge

## Architecture and Implementation Guide

**Version:** 1.0\
**Date:** 13 October 2026\
**Scope:** Practo Partner API, Redcliffe Labs API, and future
healthcare/lab/consultation vendors

------------------------------------------------------------------------

## 1. Executive Summary

RestoreHealth should use **one patient experience and one internal
integration layer**, rather than creating a separate login page, profile
system, booking flow, or report screen for every vendor.

The recommended design is an **Integration Bridge** (also called an
adapter/provider architecture):

-   RestoreHealth owns the patient-facing login, profile, consent,
    bookings list, and app navigation.
-   A backend integration layer communicates with Practo, Redcliffe, and
    future providers.
-   Each provider has a small, isolated adapter that translates
    RestoreHealth's common requests into that provider's API format.
-   A provider registry selects the correct adapter.
-   Common models normalize provider-specific responses into
    RestoreHealth's internal format.
-   Vendor API keys and secrets remain on the backend, never in
    Flutter/web/mobile client code.
-   Existing RestoreHealth features should be preserved; introduce this
    structure incrementally rather than replacing unrelated modules.

**Important distinction:** a shared login does not mean all vendors use
the same credentials or identity system. RestoreHealth can provide one
patient sign-in, while the backend separately performs the
provider-specific authentication and account-linking required by each
vendor.

## 2. What the supplied documentation actually supports

### 2.1 Practo Partner API details supplied in the request

Base URL shown in the supplied Practo documentation:

`https://practo-partner-portal-main-5debb6d.zuplo.app`

The user also supplied a portal link using `zuplo.site`:

`https://practo-partner-portal-main-5debb6d.zuplo.site/routes/consult`

Use the API base URL confirmed by Practo for the environment and
endpoint being called. Do not silently interchange `.site` and `.app`;
the supplied API examples use `.zuplo.app`.

  -------------------------------------------------------------------------------------------------
  Endpoint                          Purpose                 Key details in supplied documentation
  --------------------------------- ----------------------- ---------------------------------------
  `POST /partner/auth`              Generate partner-level  Requires
                                    JWT                     `Authorization: Bearer YOUR_KEY_HERE`
                                                            and `x-api-key`

  `GET /user?phone_number=...`      Find user by phone      API key headers required; phone number
                                    number and obtain a     is required
                                    user JWT                

  `POST /user`                      Create/register a user  Body includes `id`, `name`,
                                    who can transact        `phone_number`; email, age, gender may
                                                            be required by partner configuration

  `PATCH /user/{partner_user_id}`   Update user profile     Updatable fields listed: name, email,
                                    fields                  age, gender

  `DELETE /user`                    Soft-delete user        Identifier is `id` or `phone_number`
  -------------------------------------------------------------------------------------------------

The documentation states Indian phone numbers should be sent in the
`+91XXXXXXXXXX` format. Gender values listed are `MALE`, `FEMALE`, and
`OTHER`. The `POST /user` example returns a `jwt_token`; the GET user
response also includes a `jwt_token`.

**Authentication caveat:** the supplied text is inconsistent in places.
It says `POST /user` requires an API key in `x-api-key` and says "not an
Authorization header", but its header list and cURL example also show
`Authorization`. Some endpoints describe partner-level JWT usage.
Confirm the exact required headers for each endpoint with Practo and
follow their live OpenAPI specification. Do not assume a single
authentication header pattern applies to every endpoint.

### 2.2 Redcliffe Labs details in the uploaded document

The uploaded `RedcliffeLabs API Documentation.docx` describes retrieving
digital test-result values for a booking ID:

`GET {{HOST}}/api/external/v2/get-digital-report/{{bookingId}}`

-   `bookingId` is an integer and is required.
-   A valid response is shown as an array of report records.
-   The response can include booking information, customer name,
    collection date, package/test details, test parameters, values,
    units, reference ranges, test method, and an `is_highlighted` flag.
-   The invalid/missing booking case shows an `errors` array with the
    message `Dont have a matching booking`.

The uploaded document does **not** establish the full Redcliffe
authentication method, booking-creation endpoints, cancellation flow,
slot search, pricing, or base `HOST` value. Those details must be
obtained from the vendor before implementing those operations. Do not
infer them from the digital-report endpoint.

## 3. Target architecture

``` text
Flutter RestoreHealth App
        |
        | RestoreHealth session token only
        v
RestoreHealth Backend / Firebase Cloud Functions
        |
        +--> Auth & Patient Identity Service
        |
        +--> Vendor Integration API
                 |
                 +--> Provider Registry
                 |       |
                 |       +--> Practo Adapter
                 |       |      +--> Practo Partner API
                 |       |
                 |       +--> Redcliffe Adapter
                 |       |      +--> Redcliffe API
                 |       |
                 |       +--> Future Vendor Adapter A
                 |       +--> Future Vendor Adapter B
                 |
                 +--> Normalization / Mapping Layer
                 +--> Vendor Credential & Token Manager
                 +--> Booking / Report Orchestration
                 +--> Audit Logs, Retry Rules, Monitoring
        |
        +--> Firebase (RestoreHealth-owned data)
                +--> patients
                +--> providerLinks
                +--> vendorBookings
                +--> labReports / report metadata
                +--> vendorConfig (non-secret metadata)
                +--> integrationAudit
```

This is a logical architecture, not a requirement to put every component
in one deployed function. Keep modules separated in code, even if they
initially deploy together.

## 4. Patient login: one screen or many?

### Recommended answer: one RestoreHealth login

1.  The patient signs in to RestoreHealth using the app's existing
    authentication flow (Firebase Auth if that is what the current app
    uses).
2.  The backend verifies the RestoreHealth session and identifies the
    internal patient.
3.  When the patient chooses a service/provider, the backend checks
    whether a provider-specific account link already exists.
4.  If no link exists, the backend uses the provider's documented
    registration or lookup flow, subject to consent and mandatory
    fields.
5.  The backend stores the mapping between the RestoreHealth patient and
    the provider's user/customer identifier.
6.  Provider-specific tokens are kept server-side when the vendor
    permits and the token is needed later. They are not exposed to the
    app unless the vendor's documented flow explicitly requires a client
    token.
7.  The app continues to use the same RestoreHealth UI and receives
    normalized data.

### Provider linking model

``` text
RestoreHealth patient ID
       |
       +-- providerLinks/practo
       |       providerUserId
       |       phoneNumber (if required)
       |       token reference / expiry metadata
       |       link status
       |
       +-- providerLinks/redcliffe
       |       vendorCustomerId (if provided)
       |       booking references
       |       link status
       |
       +-- providerLinks/futureVendor
```

Use a stable internal patient ID as the primary identity. Do not use
phone number as the only database key. Handle phone-number changes,
duplicate accounts, consent, account unlinking, and provider-specific
identifiers explicitly.

**Do not create a separate login screen for every company unless a
provider explicitly requires a separate user
authentication/consent/redirect flow.** If that happens, isolate it
inside the provider-specific flow; the rest of the app should remain
common.

## 5. Suggested project structure

The following is a target structure that can be introduced
incrementally. Adapt names to the existing RestoreHealth repository
instead of deleting or moving working features blindly.

``` text
restorehealth-app/
├── lib/
│   ├── core/
│   │   ├── auth/
│   │   ├── network/
│   │   ├── errors/
│   │   └── config/
│   ├── features/
│   │   ├── authentication/          # Existing RestoreHealth login
│   │   ├── patient_profile/
│   │   ├── lab_tests/
│   │   ├── consultations/
│   │   ├── bookings/
│   │   └── reports/
│   └── integrations/
│       ├── models/
│       │   ├── provider.dart
│       │   ├── unified_booking.dart
│       │   └── unified_lab_report.dart
│       ├── api/
│       │   └── integration_api_client.dart
│       └── presentation/
│           └── provider_selection_page.dart
│
└── functions/
    └── src/
        ├── index.ts
        ├── config/
        │   └── providers.ts
        ├── middleware/
        │   ├── require_auth.ts
        │   └── validate_request.ts
        ├── routes/
        │   ├── provider_routes.ts
        │   ├── patient_routes.ts
        │   ├── booking_routes.ts
        │   └── report_routes.ts
        ├── services/
        │   ├── provider_registry.ts
        │   ├── patient_identity_service.ts
        │   ├── booking_service.ts
        │   ├── report_service.ts
        │   └── token_manager.ts
        ├── integrations/
        │   ├── common/
        │   │   ├── provider_adapter.ts
        │   │   ├── provider_errors.ts
        │   │   └── normalizers.ts
        │   ├── practo/
        │   │   ├── practo_adapter.ts
        │   │   ├── practo_client.ts
        │   │   ├── practo_auth.ts
        │   │   └── practo_mappers.ts
        │   ├── redcliffe/
        │   │   ├── redcliffe_adapter.ts
        │   │   ├── redcliffe_client.ts
        │   │   └── redcliffe_mappers.ts
        │   └── future_vendor/
        │       └── ...
        ├── repositories/
        │   ├── provider_link_repository.ts
        │   ├── booking_repository.ts
        │   └── report_repository.ts
        ├── utils/
        │   ├── phone_number.ts
        │   ├── safe_logging.ts
        │   └── idempotency.ts
        └── tests/
            ├── practo_adapter.test.ts
            ├── redcliffe_adapter.test.ts
            └── provider_registry.test.ts
```

If the repository already has `functions/src/vendors/registry.ts`,
`types.ts`, `redcliffe.ts`, `gateway.ts`, or related files, reuse and
refactor those in place where practical. Do not create duplicate
registries or duplicate vendor clients. First inspect the existing
implementation and tests.

## 6. Provider adapter contract

Every provider should implement the same internal contract. Only add
operations that are genuinely needed and supported by the provider.
Unsupported operations should return a clear `NOT_SUPPORTED` result, not
pretend to succeed.

Illustrative TypeScript contract:

``` ts
export type ProviderId = "practo" | "redcliffe" | string;

export interface ProviderContext {
  requestId: string;
  internalPatientId: string;
}

export interface ProviderCapabilities {
  userLookup: boolean;
  userRegistration: boolean;
  profileUpdate: boolean;
  bookingCreate: boolean;
  bookingCancel: boolean;
  reportFetch: boolean;
  consultationBooking: boolean;
}

export interface ProviderAdapter {
  readonly id: ProviderId;
  readonly capabilities: ProviderCapabilities;

  healthCheck?(): Promise<{ ok: boolean }>;

  // Define methods only when their behaviour is confirmed for the provider.
  lookupPatient?(input: {
    phoneNumber: string;
  }, context: ProviderContext): Promise<unknown>;

  registerPatient?(input: {
    internalPatientId: string;
    name: string;
    phoneNumber: string;
    email?: string;
    age?: number;
    gender?: "MALE" | "FEMALE" | "OTHER";
  }, context: ProviderContext): Promise<unknown>;

  fetchReport?(input: {
    bookingId: string | number;
  }, context: ProviderContext): Promise<UnifiedLabReport>;
}
```

The `unknown` return types above are intentional placeholders until each
vendor's response is mapped and validated. In production, define
provider-specific DTOs and normalized return types rather than passing
raw responses through to the app.

### Provider registry

``` ts
import type { ProviderAdapter } from "./common/provider_adapter";
import { PractoAdapter } from "./practo/practo_adapter";
import { RedcliffeAdapter } from "./redcliffe/redcliffe_adapter";

const providers: Record<string, ProviderAdapter> = {
  practo: new PractoAdapter(),
  redcliffe: new RedcliffeAdapter(),
};

export function getProvider(providerId: string): ProviderAdapter {
  const provider = providers[providerId.toLowerCase()];
  if (!provider) {
    throw new Error("PROVIDER_NOT_CONFIGURED");
  }
  return provider;
}
```

Use dependency injection and tests if adapters require configuration. Do
not instantiate clients with secrets in source code. The registry should
contain code-level adapters; credentials should be injected from
server-side secret configuration.

## 7. Unified API exposed to the RestoreHealth app

The Flutter app should call RestoreHealth-owned endpoints, not vendor
endpoints directly.

Example internal API design:

  -----------------------------------------------------------------------------------
  RestoreHealth endpoint                          Purpose
  ----------------------------------------------- -----------------------------------
  `GET /api/providers`                            Return enabled providers and
                                                  supported services

  `GET /api/providers/:providerId/capabilities`   Return supported features

  `POST /api/patients/provider-link`              Start/link a patient to a provider

  `GET /api/patients/provider-links`              Show linked providers to the
                                                  authenticated patient

  `POST /api/bookings`                            Create a booking through a provider
                                                  that supports booking creation

  `GET /api/bookings`                             Get the patient's unified booking
                                                  list

  `GET /api/reports/:providerId/:bookingId`       Fetch/normalize a provider report

  `PATCH /api/patient/profile`                    Update RestoreHealth's patient
                                                  profile; propagate only when
                                                  configured and supported
  -----------------------------------------------------------------------------------

These are **proposed RestoreHealth endpoints**, not endpoints documented
by Practo or Redcliffe. Route names can be adapted to the existing
Firebase Functions API. Each endpoint must verify the RestoreHealth
user, validate ownership, validate input, and check provider
capabilities.

## 8. Practo adapter: documented operations

### 8.1 Partner token flow

The supplied Practo documentation describes `POST /partner/auth` to
generate a partner-level JWT. It lists both
`Authorization: Bearer YOUR_KEY_HERE` and `x-api-key`.

Expected high-level flow:

1.  Backend reads Practo API credentials from secret storage.
2.  Backend calls `POST /partner/auth` with the exact headers confirmed
    by Practo.
3.  Backend validates the status and response shape.
4.  Backend stores the token securely with expiry metadata if expiry is
    known; otherwise follow Practo's documented token lifecycle and
    refresh safely.
5.  Backend uses the appropriate token and any required user
    identifier/phone number for subsequent calls.

Do not assume a token's expiry duration when the documentation does not
state it. Do not log tokens.

### 8.2 User lookup

`GET /user?phone_number=+91XXXXXXXXXX`

-   Requires a phone number.
-   Supplied documentation lists `Authorization` and `x-api-key`.
-   Successful response includes user fields and `jwt_token`.
-   Handle `400`, `401`, and `404` separately.
-   Normalize phone numbers before calling the vendor.

### 8.3 User creation

`POST /user`

The supplied example includes:

``` json
{
  "id": "7864528c-4cb4-4414-aa40-c10a7343d781",
  "name": "John Doe",
  "email": "john@email.com",
  "phone_number": "+919876543210",
  "age": 30,
  "gender": "MALE"
}
```

Generate the partner user ID on the backend according to the agreed
identifier strategy. Send only fields collected with appropriate
consent. The documentation says mandatory fields can depend on the
partner's configuration. It also describes an existing user being
returned/updated, so make registration idempotent and test
duplicate-user behaviour before production.

### 8.4 Profile update and soft delete

-   `PATCH /user/{partner_user_id}` lists name, email, age, and gender
    as updatable fields.
-   `DELETE /user` supports `id` or `phone_number` and is described as a
    soft delete.
-   Do not automatically propagate RestoreHealth account deletion to a
    vendor until retention, consent, downstream booking/report
    obligations, and the vendor's deletion policy have been reviewed.

### 8.5 What is not established

The pasted Practo content includes a `/routes/consult` portal link, but
the supplied text does not provide the complete consultation API
contract (for example, doctor search, availability, appointment
creation, payment, cancellation, or webhooks). Do not implement those
based on the link name alone. Obtain the complete consult endpoint
schemas and auth rules first.

## 9. Redcliffe adapter: documented report flow

The uploaded Redcliffe document supports this report retrieval call:

``` http
GET {{HOST}}/api/external/v2/get-digital-report/{bookingId}
```

Implementation outline:

1.  Authenticate the RestoreHealth patient request.
2.  Verify that the booking belongs to the authenticated patient using
    RestoreHealth's stored booking/provider mapping. Do not accept an
    arbitrary booking ID without an ownership check.
3.  The backend calls Redcliffe using the configured `HOST` and the
    vendor's confirmed authentication headers.
4.  Validate the response; handle an array of report records and the
    documented `errors` response.
5.  Normalize the response into RestoreHealth's common report model.
6.  Return only the data needed by the authorized patient and record a
    safe audit event.

The document's example report includes: - Booking ID, customer name,
collection date, lead ID, client reference ID - Package/test name and
code - Test parameter name, value, unit - Male/female reference ranges -
Test method and highlighted flag - Additional fields such as barcode,
UHID, and timestamps

Lab reports are sensitive health information. Apply strict patient-level
access checks, secure transport, least-privilege access, retention
rules, and safe logging. Do not log complete report payloads in ordinary
application logs.

## 10. Common data models

### Provider

``` ts
export interface ProviderSummary {
  id: string;
  displayName: string;
  enabled: boolean;
  services: Array<"lab" | "consultation" | "diagnostics">;
  capabilities: Record<string, boolean>;
}
```

### Provider link

``` ts
export interface ProviderLink {
  internalPatientId: string;
  providerId: string;
  providerUserId?: string;
  status: "pending" | "linked" | "failed" | "unlinked";
  linkedAt?: string;
  lastVerifiedAt?: string;
}
```

Keep access tokens and API secrets out of client-readable provider-link
documents. If a token must be persisted, store it only in a server-only
location with access controls and encryption appropriate to the
platform.

### Unified lab report

``` ts
export interface UnifiedLabReport {
  providerId: string;
  bookingId: string;
  patientDisplayName?: string;
  collectionDate?: string;
  tests: Array<{
    testName: string;
    testCode?: string;
    parameters: Array<{
      name: string;
      value: string;
      unit?: string;
      referenceRangeMale?: string;
      referenceRangeFemale?: string;
      method?: string;
      isHighlighted?: boolean;
    }>;
  }>;
  fetchedAt: string;
}
```

This is a RestoreHealth-owned normalized model, not a verbatim vendor
schema. Preserve clinically relevant source values and avoid changing
the meaning of values, units, or reference ranges during mapping.

## 11. Configuration and secrets

Use environment-specific server-side secrets/configuration, for example:

``` text
PRACTO_BASE_URL
PRACTO_API_KEY
PRACTO_AUTHORIZATION_KEY
REDCLIFFE_BASE_URL
REDCLIFFE_API_KEY            # only if Redcliffe confirms this auth method
```

These names are illustrative; match the real credential names and
authentication scheme supplied by each company. Do not commit secret
values to Git, put them in Flutter assets, return them from an API, or
expose them through client-side environment variables. Use Firebase
Functions secrets or the hosting platform's secret manager. Keep
non-secret metadata (provider ID, display name, enabled services)
separate from credentials.

## 12. Error handling and resilience

Map vendor errors into a stable RestoreHealth error format:

``` json
{
  "error": {
    "code": "PROVIDER_AUTH_FAILED",
    "message": "The service could not authenticate with the provider.",
    "requestId": "server-generated-request-id",
    "retryable": false
  }
}
```

Suggested internal codes: - `INVALID_REQUEST` - `UNAUTHENTICATED` -
`FORBIDDEN` - `PROVIDER_NOT_CONFIGURED` - `PROVIDER_NOT_SUPPORTED` -
`PROVIDER_AUTH_FAILED` - `PROVIDER_TIMEOUT` - `PROVIDER_RATE_LIMITED` -
`PROVIDER_NOT_FOUND` - `PROVIDER_INVALID_RESPONSE` - `INTERNAL_ERROR`

Rules: - Do not return raw vendor stack traces or credentials. - Retry
only transient errors (timeouts, selected 5xx, rate-limit responses when
safe) with bounded exponential backoff and jitter. - Do not blindly
retry create-booking or create-user calls unless idempotency/duplicate
behaviour is understood. - Set request timeouts and payload-size
limits. - Add a correlation/request ID to logs and responses. - Avoid
storing duplicate bookings by using vendor booking IDs and idempotency
keys where supported.

## 13. Security and privacy checklist

-   [ ] Verify Firebase/RestoreHealth ID token on every protected
    backend request.
-   [ ] Check that the patient owns the booking, provider link, and
    report being requested.
-   [ ] Keep provider API keys and partner tokens server-side.
-   [ ] Validate and normalize Indian phone numbers.
-   [ ] Obtain user consent before sharing personal/health data with a
    provider.
-   [ ] Send only provider-required fields.
-   [ ] Restrict admin-only provider configuration operations.
-   [ ] Redact authorization headers, JWTs, API keys, phone numbers, and
    health data from logs.
-   [ ] Apply rate limits and request validation.
-   [ ] Audit account linking, booking changes, and report access
    without logging report contents.
-   [ ] Use test credentials and sandbox environments before production.
-   [ ] Confirm vendor data retention, deletion, and breach-notification
    obligations.

## 14. Adding a future provider

For each new company, the intended workflow should be:

1.  Collect the vendor's complete API documentation, base URLs, sandbox
    credentials, authentication, rate limits, webhook details, and
    support contact.
2.  Make a capability checklist: patient lookup/registration, service
    search, availability, booking, payment, cancellation, report
    retrieval, and webhooks.
3.  Add one adapter under `integrations/<vendor>/`.
4.  Implement the common adapter contract only for supported
    capabilities.
5.  Map the vendor's DTOs to RestoreHealth's common models.
6.  Register the adapter in `provider_registry.ts` and add non-secret
    provider metadata.
7.  Add secrets through the deployment secret manager.
8.  Add mocked unit tests, contract tests against the vendor sandbox,
    and integration tests.
9.  Add the provider to the provider list only when its required
    operations are configured and tested.
10. Test authentication failures, duplicate users/bookings, timeouts,
    invalid payloads, access control, and report ownership.
11. Deploy to staging, run end-to-end tests, then enable in production
    with monitoring and a rollback plan.

A provider that only supports reports should appear as a report-capable
provider; it must not automatically show booking or consultation
buttons.

## 15. Testing matrix

  -----------------------------------------------------------------------
  Scenario                            Expected result
  ----------------------------------- -----------------------------------
  Unknown provider ID                 Stable `PROVIDER_NOT_CONFIGURED`
                                      error

  Disabled provider                   Requests rejected safely

  Practo invalid API key              Safe auth error; secret not logged

  Practo user not found               Handle documented `404`;
                                      registration only if product flow
                                      and consent permit

  Practo duplicate user creation      Follow documented response; no
                                      duplicate internal link

  Malformed phone number              Reject before vendor call

  Redcliffe valid booking ID          Normalize report response

  Redcliffe missing/invalid booking   Handle documented `errors` response
  ID                                  

  Patient requests another patient's  `403`/not-found policy; no report
  report                              disclosure

  Provider timeout                    Bounded timeout and safe retry
                                      policy

  Provider lacks booking capability   Return `PROVIDER_NOT_SUPPORTED`

  Vendor response schema changes      Validation failure, alert, no
                                      unsafe data mapping

  Missing secret in deployment        Fail safely and alert; never fall
                                      back to a hardcoded key
  -----------------------------------------------------------------------

## 16. Recommended implementation phases

### Phase 1 --- Audit and foundation

-   Inspect existing RestoreHealth Flutter and Firebase Functions code.
-   Identify existing vendor registry/gateway/types and preserve working
    modules.
-   Confirm Practo and Redcliffe sandbox access, auth rules, and exact
    environments.
-   Define the provider contract and normalized models.

### Phase 2 --- Secure server-side clients

-   Build/reuse Practo client and partner/user auth flows.
-   Build Redcliffe client for the documented digital-report endpoint.
-   Add secrets, timeouts, validation, error mapping, and safe logs.

### Phase 3 --- Identity and provider linking

-   Keep the current RestoreHealth login.
-   Create server-side provider-link records.
-   Implement account lookup/linking with consent and duplicate
    handling.
-   Do not merge accounts based only on a matching name.

### Phase 4 --- Shared app screens

-   Reuse the existing patient profile, booking list, and report viewer
    where possible.
-   Show provider-specific services based on capabilities.
-   Keep vendor-specific differences inside adapter/mapping code and
    only the minimum necessary UI flow.

### Phase 5 --- Verification and rollout

-   Unit, contract, security, and end-to-end tests.
-   Test in sandbox/staging first.
-   Add monitoring and vendor-level enable/disable switches.
-   Release gradually and retain a rollback path.

## 17. Decisions that must be confirmed before production

1.  **Practo authentication:** exact required headers for each endpoint,
    whether partner JWT or API key is expected for each call, token
    expiry, and refresh behaviour.
2.  **Practo consult API:** complete endpoint schemas for
    provider/doctor search, availability, appointment booking,
    cancellation, payment, and callbacks if required.
3.  **Redcliffe configuration:** actual `HOST`, authentication method,
    booking ownership/reference strategy, and whether report access is
    available for every booking type.
4.  **Identity rules:** which patient fields are mandatory, whether
    vendor user IDs should be RestoreHealth UUIDs or vendor-issued IDs,
    and account-linking consent.
5.  **Operational rules:** rate limits, timeouts, retries, sandbox
    availability, support escalation, data retention, and deletion
    requirements.

Until these are confirmed, implement only the documented and testable
portions. In particular, the uploaded Redcliffe document supports
digital-report retrieval but does not, by itself, document the complete
booking workflow.

## 18. Final recommendation

Build **one RestoreHealth identity + one backend integration bridge +
one adapter per vendor + shared normalized models and UI**. This lets
future vendors be added mainly by implementing an adapter, mappings,
configuration, and tests---rather than duplicating login pages and
rewriting the whole app.

The bridge reduces repeated changes; it cannot eliminate all
vendor-specific work. Each new company will still have unique
authentication, data fields, booking rules, and capabilities that must
be handled in its adapter.
