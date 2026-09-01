# Checks reference

Every check Valyd can run: what it verifies, what comes back, and where it's available.

**All of these run as checks inside a [workflow](workflows.md) session.** There are no per-check
public REST endpoints and no direct SDK methods for them — you compose them in the portal and run
them with `sessions.create()`.

| Check | Reusable Verification | Unique Human API |
| --- | --- | --- |
| `id_verification` | yes | — |
| `liveness` | yes | — |
| `antispoof` | yes | yes |
| `antispoof/identity` | yes | yes |
| `face-uniqueness` | yes | yes |
| `face_match` | yes | — |
| `age` | yes | — |
| `credential` | yes | — |
| `location` | yes | — |

---

## `id_verification` — ID verification

Verifies a government ID: OCR of the document fields plus an authenticity score. The user provides
the front (and optionally back) image. You get a **proof** that the document was verified; the raw
extracted fields stay encrypted on the account.

Use it whenever you need to know who a document says someone is.

In a decision from a connected session the check reduces to `{ status, id_verified }` — you do not
receive the raw fields.

---

## `liveness` — passive liveness

Passive liveness on a single selfie: is this a real, live capture? Use it as the cheap first gate
before a face match.

---

## `antispoof` — anti-spoof

A stronger "is this a live human capture?" answer with a vendor-neutral **`human_score` (0–100)**.

- A single image scores at most **85** (`assurance: "upload"`).
- A **3–8 frame burst** over ~2 s adds motion and same-person consistency analysis.
- A workflow session captures a live camera burst with a **random on-screen action** — the
  strongest assurance, `assurance: "captured"`. An attacker can't know the action in advance, which
  is what makes photos, screen replays and pre-recorded clips fail.

```json
{
  "assurance": "captured",
  "frames_analyzed": 5,
  "frames_genuine": 5,
  "frames_spoof": 0,
  "motion": "natural",
  "face_consistency": "consistent",
  "human_score": 100
}
```

On failure the check data carries a `signal`: `no_face`, `face_unreadable`, `spoof_detected`,
`low_confidence`, `duplicate_frames`, `static_capture`, `discontinuous_motion`, `different_faces`.

Use it when presentation attacks are a real threat.

---

## `antispoof/identity` — anti-spoof + identity

Runs the identical anti-spoof pipeline and, **only if it passes**, resolves the proven-live face to
a stable `valyd_` uuid from the global face gallery. If liveness fails, no identity lookup runs.

```json
{ "human_score": 100, "identity": { "valyd_uuid": "valyd_f35fecf0…", "registered": "existing" } }
```

The same face resolves to the same uuid whenever the gallery match clears its similarity threshold.
Use it for duplicate-account / sybil detection with liveness assurance built in.

---

## `face-uniqueness` — uniqueness

One face = one Valyd uuid. Enrols or matches the captured face against the global registry and
returns the stable `valyd_uuid` plus whether it was newly registered.

```json
{ "valyd_uuid": "valyd_8f2…", "registered": "existing" }
```

`registered` is `"new"` the first time a face is seen, `"existing"` afterwards. Store the
`valyd_uuid` against your own user record; a later session resolving to an already-seen uuid is the
same human, and your policy decides what happens next.

Pair it with anti-spoof in the same workflow so only a **proven-live** face is matched against the
registry.

**Deleting a face (GDPR):** unlink it by its `valyd_` uuid from the project's Verification settings
in the Developer Portal, or with `verify.faceUniquenessUnlink(valydUuid)`. The face is unlinked
from your project, and fully deleted once no remaining project — and no Valyd account — still knows
it.

---

## `face_match` — 1:1 face match

Compares two face images, typically the ID portrait against a fresh selfie. Returns `similarity`
and the pass `threshold` (default ~0.95).

```json
{ "similarity": 0.973, "threshold": 0.95 }
```

Use it to bind a live person to a verified document — or, on a connected session, to re-prove
identity before a sensitive action, matched against the account's stored face vector.

---

## `age` — age bands

Computes age bands from a verified date of birth. With a `valydAccessToken` the bands are computed
from the account's KYC-verified DOB and returned as a proof — **you never touch the raw DOB**.

Bands: `is_16_plus`, `is_18_plus`, `is_21_plus`, `is_30_plus`, `is_65_plus`.

> The response field `bands.*.verified` is a **deprecated alias** as of 2026-08-19. Read
> **`satisfied`** — same value, honest name.

---

## `credential` — professional license

Looks up a professional license in the provider registry and matches it to a name. Returns `match`
and the registry's `license` record.

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

**In a workflow session the name comes from the verified ID — never client-supplied** — so a caller
cannot substitute an arbitrary name to claim someone else's license.

**Registry lookups take 10–60 seconds.** Failure reasons on `check.error.code`:
`license_not_found`, `license_expired`, `license_inactive`, `name_mismatch`, `board_unavailable`
(retry in a few minutes).

### Discovery helpers

Build state and license-type pickers before the session:

```js
const { states }    = await verify.credentials.states();
const { providers } = await verify.credentials.providers("CA");
const types         = await verify.credentials.types("CA");   // whole catalog, per-state, or per-provider
```

`providers()` returns each provider's `required_fields`. Always collect first and last name even
when `required_fields` omits it — the registry always needs a name.

---

## KYC + credential — combined

ID verification + liveness + face match + license lookup in one workflow session. The license is
matched against the name OCR'd from the ID. You get a per-check breakdown plus the verified
`identity` proof.

`status` is `"passed"` only when **every** check passes.

---

## `location` — geolocation / geofence

Records and validates a geolocation fix for a session. Used by workflows like EVV to prove where a
check happened.

**A real GPS fix is mandatory** — a blocked permission or missing coordinates is a hard `failed`,
never a pass and never a review.

| What the workflow has | `status` | `data.match` |
| --- | --- | --- |
| an expected point **+ radius** | `passed` inside, `failed` outside | `true` / `false` |
| an expected point, **no radius** | `passed` | `null` — only `distance_m` is reported; you decide |
| no expected point | `passed` | absent — capture-only coordinates |

```json
{
  "latitude": 37.3382, "longitude": -121.8863, "accuracy": 12,
  "source": "gps", "captured_at": "2026-07-13T18:04:10+00:00",
  "distance_m": 137.4, "radius_m": 200, "match": true
}
```

Where a radius is configured, **the status is the verdict**. Never treat location as report-only.

---

## Biometrics

**Irreversible vectors, never images.** Valyd does not store or return face images. Enrollment
converts a selfie into a one-way biometric vector (template); every later face match compares
vectors. The photos submitted to a check are processed **transiently** and are not retrievable from
a Valyd account. The template is never exposed through any API.

The ID `portrait` in a KYC result is extracted from the document submitted in that request — not a
stored account photo.

## Related

- [`workflows.md`](workflows.md) — bundling these into a workflow
- [`unique-human-api.md`](unique-human-api.md) — the three checks available without a login
- [`statuses.md`](statuses.md) — per-check status values
