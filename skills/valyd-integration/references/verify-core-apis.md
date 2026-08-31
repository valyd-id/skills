# Core APIs — server-to-server verification

Direct, synchronous checks. You build your own UI and call these from your backend.

**Base URL:** `https://idp.valyd.work` · **Auth:** `X-API-Key: <App API key>` on every request —
server-side only, never ship it to the browser.

> **Raw data vs proofs.** Without a Valyd user token these are **Non-account (Fresh)** checks: you
> did the capture, nothing is retained, and the response contains the **raw** extracted data
> (document `fields`, `dob`, portrait, OCR). Pass a `valyd_access_token` (or `valyd_id`) and the same
> endpoints run in **Account (Managed by Valyd)** mode — answering from the user's stored identity
> and returning **proofs only**, never raw KYC. See `verify-modes-and-account.md`.

## Response envelope

Every response carries a `check` object:

```json
{
  "success": true,
  "data": {
    "session_id": "ses_...",
    "status": "passed",
    "check": {
      "type": "id_verification",
      "status": "passed",
      "score": 0.97,
      "data": {},
      "error": null
    }
  },
  "error": null
}
```

`status` and `check.status` are `passed` | `failed` | `review`. The JSON blocks under each endpoint
below are the contents of **`check.data`** unless labelled otherwise.

## Idempotency

Every billable `POST /api/v2/*` accepts an **`Idempotency-Key`** header. Send a unique key per
logical operation; Valyd stores the first response and **replays it byte-for-byte** for repeats, so
a network retry can never double-charge or double-run a check.

```bash
curl -X POST https://idp.valyd.work/api/v2/liveness \
  -H "X-API-Key: $VALYD_API_KEY" \
  -H "Idempotency-Key: 5f2c...-your-unique-id" \
  -F "image=@./selfie.jpg"
```

- Keys are scoped **per project**, retained **24 hours**.
- A replayed response carries `Idempotency-Replayed: true`.
- Same key + **different body** → `422 idempotency_key_reused`.
- Key whose first request is still in flight → `409 idempotency_in_progress`; retry shortly.
- Only 2xx responses are stored; a failed call leaves the key free to retry.

## SDK setup

Image fields accept a file path via `readImage("./x.jpg")`, a `Buffer`, or a base64 / data-URL
string. Over plain HTTP, send images as base64 in the JSON field, or as a multipart file under the
same field name.

```bash
npm i @valyd/sdk
export VALYD_API_KEY="<your App API key from https://dev.valyd.work>"
```

```js
import { VerifyClient, readImage } from "@valyd/sdk";
const verify = new VerifyClient({ apiKey: process.env.VALYD_API_KEY });
```

---

## POST /api/v2/id-verification — ID verification

OCR + authenticity from a government ID.

**Fields:** `front_image` (image) **required**; `back_image` (image) when applicable.

```bash
curl -X POST https://idp.valyd.work/api/v2/id-verification \
  -H "X-API-Key: $VALYD_API_KEY" \
  -F "front_image=@./id_front.jpg" \
  -F "back_image=@./id_back.jpg"
```

```js
const { check } = await verify.standalone.idVerification({
  frontImage: readImage("./id_front.jpg"),
  backImage:  readImage("./id_back.jpg"),   // optional
});
console.log(check.data.fields.full_name, check.data.fields.document_number);
```

`check.type` is `"id_verification"`; `check.data`:

```json
{
  "fields": {
    "full_name": "Jane Doe",
    "fathers_name": "John Doe",
    "document_number": "X1234567",
    "date_of_birth": "1990-01-15",
    "date_of_issue": "2020-03-10",
    "date_of_expiry": "2030-03-10",
    "sex": "F",
    "issuing_state": "CA",
    "country": "US",
    "document_type": "driver_license"
  },
  "portrait": "<base64>",
  "dob": "1990-01-15",
  "authenticity": { "score": 0.96 }
}
```

---

## POST /api/v2/liveness — liveness

Passive liveness. **Passes when `live_score === 1`.**

**Fields:** `image` (image) **required** — a selfie.

```bash
curl -X POST https://idp.valyd.work/api/v2/liveness \
  -H "X-API-Key: $VALYD_API_KEY" -F "image=@./selfie.jpg"
```

```js
const { check } = await verify.standalone.liveness({ image: readImage("./selfie.jpg") });
```

`check.data`: `{ "live_score": 1, "result": "live" }`

`live_score`: `1` = live, `0` = spoof, `< 0` = no face detected.

---

## POST /api/v2/antispoof — anti-spoof (single image or burst)

Answers "is this a live human capture?" with a vendor-neutral `human_score` (0–100) and a pass/fail
verdict. A **burst** adds per-frame voting, motion analysis and same-person consistency — far
stronger than a single frame.

**Send one of:**
- `image` (image) — single still. Analysis-only; **`human_score` is capped at 85** (`assurance: "upload"`).
- `frames[]` (3–8 images) — chronological stills over ~2 s (`assurance: "burst"`).

```bash
curl -X POST https://idp.valyd.work/api/v2/antispoof \
  -H "X-API-Key: $VALYD_API_KEY" \
  -F "frames[]=@./f1.jpg" -F "frames[]=@./f2.jpg" -F "frames[]=@./f3.jpg" \
  -F "frames[]=@./f4.jpg" -F "frames[]=@./f5.jpg"
```

`check.type` is `"antispoof"`, `check.score` is the `human_score`. `check.data`:

```json
{
  "assurance": "burst",
  "frames_analyzed": 5,
  "frames_genuine": 5,
  "frames_spoof": 0,
  "motion": "natural",
  "face_consistency": "consistent",
  "human_score": 100
}
```

On failure `check.data.signal` is one of: `no_face`, `face_unreadable`, `spoof_detected`,
`low_confidence`, `duplicate_frames`, `static_capture`, `discontinuous_motion`, `different_faces`.

> For the **strongest** assurance, run anti-spoof through the **hosted flow**: Valyd captures a live
> camera burst and issues a random on-screen action (turn head, open mouth, nod, move closer). That
> path returns `assurance: "captured"`. An attacker can't know the action in advance, which is what
> makes photos, screen replays and pre-recorded clips fail.

---

## POST /api/v2/antispoof/identity — anti-spoof + identity

Runs the identical anti-spoof pipeline and, **only if it passes**, resolves the proven-live face
against the global Valyd face gallery, returning a **stable `valyd_` uuid**. The same face always
resolves to the same uuid across all your requests — duplicate-account / sybil detection. If
liveness fails, no identity lookup runs (and none is billed).

Same input as `/antispoof`. `check.data` adds:

```json
{ "human_score": 100, "identity": { "valyd_uuid": "valyd_f35fecf0...", "registered": "existing" } }
```

`registered` is `"new"` the first time a face is seen, `"existing"` afterwards.

---

## POST /api/v2/face-uniqueness — face uniqueness (dedup)

One face = one Valyd uuid. Enrols or matches a selfie against the global gallery. Accepts a single
`image`/`selfie` (single-frame liveness gate) or `frames[]` (full live pipeline gate).

```json
{ "valyd_uuid": "valyd_8f2...", "registered": "existing" }
```

**Unlink:** `DELETE /api/v2/face-uniqueness/{valyd_uuid}` removes a face from the gallery — e.g. to
clear test data.

---

## POST /api/v2/face-match — face match

Compare two images. Passes when similarity ≥ threshold (default ~0.95).

**Fields:** `image1` (image) **required** — reference, typically the ID portrait; `image2` (image)
**required** — the selfie.

```bash
curl -X POST https://idp.valyd.work/api/v2/face-match \
  -H "X-API-Key: $VALYD_API_KEY" \
  -F "image1=@./id_portrait.jpg" -F "image2=@./selfie.jpg"
```

```js
const { check } = await verify.standalone.faceMatch({
  idImage: readImage("./id_portrait.jpg"),
  selfie:  readImage("./selfie.jpg"),
});
```

`check.data`: `{ "similarity": 0.973, "threshold": 0.95 }`

In **Account** mode pass `{ valydAccessToken, selfie }` instead — the selfie is matched against the
user's stored face vector and you never ask for a reference image.

---

## POST /api/v2/age-verification — age verification

JSON body. Computes age from DOB and verifies the requested bands (no ZKP).

**Fields:** `dob` (string `YYYY-MM-DD`) **required**; `bands` (string[]) **required**.

```bash
curl -X POST https://idp.valyd.work/api/v2/age-verification \
  -H "X-API-Key: $VALYD_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "dob": "1995-06-01", "bands": ["is_18_plus","is_21_plus"] }'
```

```js
const { check } = await verify.standalone.ageVerification({
  dob: "1995-06-01",
  bands: ["is_18_plus", "is_21_plus"],
});
```

`check.data`:

```json
{
  "age": 30,
  "dob": "1995-06-01",
  "bands": {
    "is_18_plus": { "verified": true, "min_age": 18 },
    "is_21_plus": { "verified": true, "min_age": 21 }
  }
}
```

This is the cheapest call to smoke-test an API key with.

---

## POST /api/v2/credential-verification — professional license

Look up a license in the provider registry. **Registry lookups take 10–60 s — use a timeout of at
least 90 s.**

**Fields:**
- `first_name` **required** — required even when a provider's `required_fields` omits it; the
  registry always needs a name. Or supply `full_name` instead of first/last.
- `last_name` **required**
- `license_type` **required** — e.g. `'MD'`. Alias `provider_code`. Valyd resolves the board from
  state + type, so you do not need to pass a provider code.
- `license_state` **required** — 2-letter code. Alias `state`.
- `license_number` **required** — alias `license_no`.
- `npi` — optional NPI when applicable.

```bash
curl -X POST https://idp.valyd.work/api/v2/credential-verification \
  -H "X-API-Key: $VALYD_API_KEY" \
  -H "Content-Type: application/json" \
  --max-time 90 \
  -d '{
    "first_name": "Jane", "last_name": "Doe",
    "license_type": "MD", "license_state": "CA", "license_number": "A12345",
    "npi": "1234567890"
  }'
```

```js
const { check } = await verify.standalone.credentialVerification({
  firstName: "Jane", lastName: "Doe",
  providerCode: "MD", licenseState: "CA", licenseNumber: "A12345",
  npi: "1234567890",   // optional
});
```

`check.data`:

```json
{
  "match": true,
  "license": {
    "license_number": "A12345",
    "status": "active",
    "issued_at": "2015-01-01",
    "expires_at": "2027-01-01",
    "specialty": "Internal Medicine"
  }
}
```

Decline reasons seen on `check.error.code`: `license_not_found`, `license_expired`,
`license_inactive`, `name_mismatch`, `board_unavailable` (retry in a few minutes).

In **Account** mode the name comes from the connected account — do not pass one.

---

## POST /api/v2/kyc-credential — KYC + credential in one call

ID verification + liveness + face match + license lookup. **The license is matched against the name
OCR'd from the ID — never a client-supplied name** — so the holder cannot impersonate someone
else's license.

**Fields:** `front_image` **required**, `selfie` **required**, `back_image` optional,
`license_type` **required** (alias `provider_code`), `license_state` **required** (alias `state`),
`license_number` **required** (alias `license_no`), `npi` optional.

```bash
curl -X POST https://idp.valyd.work/api/v2/kyc-credential \
  -H "X-API-Key: $VALYD_API_KEY" \
  -F "front_image=@./id_front.jpg" -F "selfie=@./selfie.jpg" \
  -F "license_type=MD" -F "license_state=CA" -F "license_number=A12345"
```

```js
const result = await verify.standalone.kycCredential({
  frontImage: readImage("./id_front.jpg"),
  selfie:     readImage("./selfie.jpg"),
  providerCode: "MD", licenseState: "CA", licenseNumber: "A12345",
});
// result.status === "passed" only when EVERY check passes
// result.checks: [id_verification, liveness, face_match, credential]
// result.identity: { name, dob }  <- the name used for the license match
```

`data`:

```json
{
  "session_id": "ses_...",
  "status": "passed",
  "identity": { "name": "Jane Doe", "dob": "1990-01-15" },
  "checks": [
    { "type": "id_verification", "status": "passed", "data": {} },
    { "type": "liveness",        "status": "passed", "data": { "live_score": 1 } },
    { "type": "face_match",      "status": "passed", "data": { "similarity": 0.97 } },
    { "type": "credential",      "status": "passed", "data": { "match": true, "license": {} } }
  ]
}
```

---

## POST /api/v2/location — location / geofence

**A real GPS fix is always mandatory.** This step can never be skipped: a blocked permission or
missing coordinates is a hard `failed` — never a pass, never a review.

**Fields:** `latitude`, `longitude` (numbers) **required**; optional `accuracy` (metres),
`expected_latitude`, `expected_longitude`, `radius_m`.

What `status` means depends on what you sent:

| You send | `status` | `data.match` | Notes |
|---|---|---|---|
| expected lat/lng **+ `radius_m`** | **`passed` inside the radius, `failed` outside** | `true` / `false` | The status **is** the verdict. `error` reads e.g. "You are 119.3 km from the required location (must be within 200 m)." |
| expected lat/lng, **no** `radius_m` | `passed` | `null` | No threshold to judge against; reports `data.distance_m`, you decide. |
| no expected point | `passed` | absent | Capture-only: returns coordinates + accuracy. |

```bash
curl -X POST https://idp.valyd.work/api/v2/location \
  -H "X-API-Key: $VALYD_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "latitude": 37.3382, "longitude": -121.8863, "accuracy": 12,
    "expected_latitude": 37.3390, "expected_longitude": -121.8850,
    "radius_m": 200
  }'
```

```json
{
  "type": "location",
  "status": "passed",
  "score": 137.4,
  "data": {
    "latitude": 37.3382, "longitude": -121.8863, "accuracy": 12,
    "source": "gps", "captured_at": "2026-07-13T18:04:10+00:00",
    "expected_latitude": 37.339, "expected_longitude": -121.885,
    "distance_m": 137.4, "radius_m": 200, "match": true
  }
}
```

SDK helper: `verify.standalone.locationMatch({ latitude, longitude, accuracy, expectedLatitude, expectedLongitude, radiusM })`.

```text
Need a yes/no "is the user at the place?"  -> send expected lat/lng + radius_m, read `status`
Only want the distance                     -> send the expected point WITHOUT radius_m, read data.distance_m
Only want the coordinates                  -> send just latitude/longitude/accuracy
status failed, reason permission_denied     -> the user blocked location; ask them to allow it and retry
```

There is no separate `location-match` endpoint and no `evv_presence` bundle — `location` (feature
key `location`) is the single location check and does the geofence verdict itself. **Do not treat
location as report-only or always-passing.**

---

## Credential discovery

Build state and license-type pickers before calling `credential-verification` or `kyc-credential`. A
provider's `required_fields` tells you which license inputs to collect — but **always collect first
and last name even when it isn't listed**, because the registry lookup needs it. Read-only, cacheable.

### GET /api/v2/credential/states

```bash
curl https://idp.valyd.work/api/v2/credential/states -H "X-API-Key: $VALYD_API_KEY"
```

```js
const { states } = await verify.credentials.states();
```

```json
{ "states": [ { "state_name": "California", "state_code": "CA" } ] }
```

### GET /api/v2/credential/states/{state}/providers

`{state}` is a 2-letter code.

```bash
curl https://idp.valyd.work/api/v2/credential/states/CA/providers -H "X-API-Key: $VALYD_API_KEY"
```

```js
const { providers } = await verify.credentials.providers("CA");
```

```json
{
  "providers": [
    {
      "provider_code": "MD",
      "provider_display_name": "Medical Board of California",
      "credential_name": "Physician & Surgeon",
      "required_fields": ["license_number"]
    }
  ]
}
```

> The recipe page for license verification shows a different field set for these two responses
> (`state`/`name`/`provider_count`, `provider_id`/`name`/`license_types`). See
> `gotchas-and-doc-conflicts.md` — read the actual response rather than assuming either shape.

---

## Errors

Over HTTP: `{ success: false, error: { code, message } }` with a matching HTTP status.

- `401` — invalid or missing API key
- `400` — validation error (missing field, bad image, unknown provider)
- `404` — unknown state or provider
- `429` — rate limited
- `5xx` — upstream registry or internal error

In the SDK the same failures throw `ValydVerifyError` with `{ code, status, message }`.

```js
import { VerifyClient, ValydVerifyError, readImage } from "@valyd/sdk";

const verify = new VerifyClient({
  apiKey: process.env.VALYD_API_KEY,
  timeoutMs: 90_000,        // registry lookups can be slow
});

try {
  const { check } = await verify.standalone.credentialVerification({
    firstName: "Jane", lastName: "Doe",
    providerCode: "MD", licenseState: "CA", licenseNumber: "A12345",
  });
} catch (err) {
  if (err instanceof ValydVerifyError) console.error(err.status, err.code, err.message);
  else throw err;
}
```

### The three that actually happen

1. **HTTP 401 — invalid or missing API key.** `X-API-Key` absent, wrong, or revoked. Set it (or
   `apiKey` in the SDK client) from a valid App API key.
2. **Client timeout on credential / kyc-credential.** Registry lookups take 10–60 s and the default
   client timeout aborts first. Use `timeoutMs: 90_000` or `--max-time 90`.
3. **HTTP 400 / 404 — validation or unknown state/provider.** Missing required field, unreadable
   image, or a `license_state` / `license_type` not in the registry. Call the discovery endpoints
   first, and always include first/last name.
