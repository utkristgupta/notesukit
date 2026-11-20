# 1. The one-paragraph pitch (say this first in the interview)

"The utility has a legacy internal web portal, Urja Meter Ops, with no API, people were copy-pasting from the browser. I built a clean FastAPI service that sits in front of it: it logs into the portal, reverse-engineers its internal JSON endpoints (including an HMAC-signed bulk export), normalizes the messy string-typed data into typed Pydantic models, repairs known data-quality problems while recording what it repaired, caches the result in memory, and exposes a documented REST API with filtering, geospatial search, a reconstructed network hierarchy, and consumption analytics. It's layered (client → service → API), fully tested offline, and self-documents via OpenAPI/Swagger."

---

# 2. The domain (understand this or the data makes no sense)

Electricity distribution is organized as a **hierarchy** from the utility down to the meter. Your 7 levels, broad → narrow:

`zone → circle → division → subdivision → substation → feeder → dt → (meter)`

- **Substation**: steps high voltage down; feeds multiple feeders.
- **Feeder**: a line leaving the substation.
- **DT (Distribution Transformer)**: the final transformer on a pole/pad that steps down to household voltage. It has a **kVA capacity** rating (e.g. 100 kVA). Multiple meters hang off one DT.
- **Meter**: the smart meter at a premises.

Electrical quantities you expose:
- **kWh (active energy)**: "real" energy that does useful work.
- **kVAh (apparent energy)**: total energy delivered including the reactive part.
- **Power factor = kWh / kVAh** (between 0 and 1). PF near 1 is efficient; low PF means a lot of reactive power. That's why your code clamps PF to ≤ 1.0, physically it can't exceed 1.
- **Cumulative ("odometer") registers**: the meter reports a *running total* that only goes up, like a car odometer. To get usage in a 30-minute interval you **subtract consecutive readings** (a delta). This is the single most important data insight in the project.
- **voltR**: voltage on the R phase (Indian 3-phase system has R/Y/B phases).

The portal gives **7 days of readings at 30-minute intervals** = ~336–337 points per meter.

---

# 3. How the portal actually works (the reverse-engineering story — this is what they said is most valuable)

The portal is a **SvelteKit** app using **Better Auth**. Everything you know came from poking at it and reading its compiled JS.

**Authentication**
- `POST /login`, form-encoded (`application/x-www-form-urlencoded`), with `email` + `password`.
- **Critical quirk**: SvelteKit enforces CSRF protection, so the request *must* include an `Origin` header matching the site (and you send `Referer` too). Without `Origin`, login is rejected even with correct credentials. That's a non-obvious thing you discovered.
- Success sets a cookie `__Secure-better-auth.session_token` with `Max-Age=3600` (1 hour). Every other endpoint needs that cookie.

**The signed bulk export (the hard part)**
- `GET /portal/export?page=1` returns all 403 meters at once — but returns an error unless the request is **HMAC-signed**.
- You found this by reading a compiled SvelteKit chunk (`4.2Bgc2kUI.js`). It revealed:
  - `GET /portal/keys` returns `{"data":{"signingSecret": "..."}}`.
  - The signing algorithm: build a **canonical string** = `method`, `path`, `query`, `timestamp` joined by **newlines** (`\n`), compute **HMAC-SHA256** keyed by the signing secret, hex-encode lowercase, and send two headers: `x-signature` and `x-timestamp`.
  - **The subtle trap**: the timestamp in the `x-timestamp` header must be *byte-for-byte* the same value you folded into the signed string. If you regenerate `now()` twice you get a mismatch and a rejected signature.

**Other endpoints**
- `GET /portal/dts?page=n` — the 40 distribution transformers, 20/page, 2 pages. **This is the only place `capacityKva` exists** — the meter export doesn't have it, so you must join on DT code.
- `GET /portal/meters/{id}/energy` — the 7-day, 30-min readings, all values as strings, sometimes `""` or `"—"`.
- `GET /portal/meters/{id}/geo` — coordinates as strings (note: the bulk export already has geo as floats, so this is a fallback).
- `GET /meters/{id}/__data.json` — SvelteKit's `devalue` format; you investigated it, found two encodings (`legacy` vs `v2`), and then abandoned it because the bulk export gave everything uniformly. (That's your "mistake" in the reflection — honest and good.)

---

# 4. Architecture: why it's layered

```
app/api       (HTTP: routes, dependencies, FastAPI wiring)    ← knows about HTTP
   │  depends on
app/services  (MeterService: business logic, caching, healing) ← knows nothing about HTTP
   │  depends on
app/client    (UrjaPortalClient: talks to the portal)          ← knows nothing about our API
   │
app/models    (Pydantic schemas, shared by all layers)
app/config    (settings, shared by all layers)
```

**Dependency direction points inward/downward.** The HTTP layer depends on the service; the service depends on the client. The client has no idea the service exists; the service has no idea FastAPI exists. Why this matters (say this):
- **Testability**: you can test the service with a fake client, no network. You can test routes by overriding one dependency.
- **Swappability**: if the portal's protocol changes, only `app/client` changes. If you add GraphQL instead of REST, only `app/api` changes.
- **Single responsibility**: each layer has exactly one reason to change.

---

# 5. Layer-by-layer deep dive

## 5.1 `app/config.py` — configuration

- Uses **pydantic-settings** `BaseSettings`. Each field has a working default, so the app runs with **zero setup** (no `.env` needed) — a deliberate "out-of-the-box" choice.
- Reads from environment variables / `.env` (case-insensitive), overriding defaults. `env_prefix=""` means the env var name equals the field name uppercased (`urja_base_url` ← `URJA_BASE_URL`).
- `get_settings()` is wrapped in **`@lru_cache`** → a single `Settings` instance per process (a cheap singleton), and it doubles as a FastAPI dependency.
- `base_url` property strips a trailing slash so URL joins are safe.
- Key tunables: `urja_session_ttl_seconds=3000` (refresh before the portal's 3600s cookie expiry), `max_retries=3`, `urja_cache_ttl_seconds=300` (catalog), `urja_energy_cache_ttl_seconds=60` (per-meter consumption), `signing_secret_ttl_seconds=300`.

**Likely question — "Why refresh at 3000s when the cookie lives 3600s?"** To avoid a race where the cookie expires *mid-request*. You proactively renew with a safety margin rather than waiting for a failure.

## 5.2 `app/models/schemas.py` — the typed contract

Pydantic v2 models that are the **public contract** of your API and deliberately *diverge* from the portal's shapes:
- `GeoCoordinates(lat, lng)` — floats, not strings.
- `HierarchyNodeRef(code, name, raw_name)` — `code`/`name` optional (can be `None` where the portal sent `""`); `raw_name` preserves the original string when you overwrite a conflicting name (audit trail).
- `MeterHierarchy` — the 7 named levels.
- `MeterRecord` — the normalized meter, including `dt_capacity_kva` (joined in) and crucially `data_warnings: List[str]`.
- `TransformerSummary` — DT plus computed `meter_count` and `status_breakdown`.
- `ConsumptionInterval` — one 30-min reading with both the cumulative registers *and* the derived interval deltas + power factor.
- `ConsumptionSummary` — aggregates.
- `MeterConsumptionResponse`, `PaginatedMetersResponse`, and the **self-referential** `HierarchyTreeNode` (has `children: List[HierarchyTreeNode]`, needs `model_rebuild()` to resolve the forward reference).
- `MeterBuild` enum (`legacy`/`v2`).

**Why Pydantic?** Automatic validation, parsing, and — because FastAPI reads these — automatic OpenAPI schema generation and response serialization. One source of truth for types, docs, and validation.

## 5.3 `app/client/portal_client.py` — `UrjaPortalClient` (the portal-facing layer)

This is the most interview-rich file. Walk through its responsibilities:

**(a) One persistent `httpx.AsyncClient`** with a shared cookie jar, created once. `follow_redirects=False` **on purpose**: the portal bounces unauthenticated requests with a 3xx redirect to `/login`. If you auto-followed, you'd silently get the login page HTML and mysterious failures. By *not* following, you can **detect** the redirect and treat it as an auth signal.

**(b) `ensure_authenticated(force=False)` with double-checked locking.** This is a concurrency showpiece:
```
if not force and session is fresh: return          # fast path, no lock
async with self._auth_lock:                         # only contended path locks
    if not force and session is fresh: return       # re-check inside lock
    await self._login()
```
Why: imagine 50 concurrent requests arrive and the session just expired. Without the lock, all 50 would log in simultaneously (a **login stampede**) — wasteful and possibly rate-limited. The `asyncio.Lock` serializes them; the first one logs in, and the other 49 re-check inside the lock, see a fresh session, and return immediately. The **outer** check avoids taking the lock at all on the hot path (performance); the **inner** re-check guarantees correctness.

**(c) `compute_hmac_signature(...)` — a `@staticmethod`** (pure function, no instance state, so it's trivially unit-testable):
```python
ts = int(time.time()) if timestamp is None else int(timestamp)
canonical = "\n".join([method.upper(), path, query, str(ts)])
digest = hmac.new(secret.encode(), canonical.encode(), hashlib.sha256).hexdigest()
return {"x-timestamp": str(ts), "x-signature": digest}
```
Note it accepts an optional `timestamp` so tests can pin it and assert determinism. It uppercases the method so `get`/`GET` produce the same signature. The returned `x-timestamp` is the *exact* string used in the digest — that's the trap you avoided.

**Interview question — "What is HMAC and why not just SHA256(secret + message)?"** HMAC is a keyed hash that proves both *integrity* and *authenticity* (you hold the secret). Plain `hash(secret+message)` is vulnerable to **length-extension attacks**; HMAC's nested construction (inner/outer padding) defends against that. Here the signature + timestamp also give a basic **replay/anti-tamper** guard: the server can reject stale timestamps and verify the request wasn't altered.

**(d) `_request_json(...)` — the central request state machine.** Every portal call goes through this. Its logic, in order:
1. `ensure_authenticated()` first.
2. Loop up to `max_retries` (3) attempts:
   - On `httpx.RequestError` (network/DNS/timeout) → exponential backoff, retry.
   - On **401/403 or redirect-to-`/login`** → force re-auth **once** (a `reauthed` flag prevents infinite re-auth loops), then retry; if it happens again after re-auth, raise.
   - On **404** → return `None` (meaning "not found", distinct from an error).
   - On **5xx** → backoff, retry (transient server problem).
   - On other **4xx** → raise `PortalUpstreamError` (our bug / unexpected; don't retry).
   - On 2xx → parse JSON (and raise if the body isn't valid JSON).
3. If the loop exhausts → raise `PortalUpstreamError`.

Backoff is `0.5 * 2**attempt` → **0.5s, 1s, 2s**. `PortalUpstreamError` carries `status_code=502` because when the *upstream* fails, the correct thing to tell *our* caller is 502 Bad Gateway (it's not the client's fault, and it's not a 500 in our code).

**(e) `get_signing_secret()`** — fetches `/portal/keys`, caches it for 300s, and is invalidated on every fresh login (a new session might pair with a rotated secret — a careful edge case).

**(f) Fetchers** — `fetch_bulk_meters_export()` (signs each page, loops while `len(collected) < total`), `fetch_all_transformers()` (paginates `/portal/dts`), `fetch_meter_energy()`, `fetch_meter_geo()`. These return **raw dicts** — the client doesn't normalize; that's the service's job. Clean separation.

## 5.4 `app/services/meter_service.py` — `MeterService` (the brain)

**(a) In-memory catalog with TTL + double-checked lock + stale fallback.**
- Holds `_meters_by_id` and `_transformers_by_code` dicts, plus `_catalog_loaded_at` and `_stale`.
- `_ensure_catalog()` uses the same double-checked `asyncio.Lock` pattern as auth: if fresh (within `urja_cache_ttl_seconds=300`), return; otherwise lock, re-check, refresh.
- **Stale-while-error**: if a refresh throws `PortalUpstreamError` but you *already* have a catalog, you log a warning, set `_stale=True`, and **keep serving the old data** instead of failing. For a read-mostly dataset, slightly stale beats a hard outage. Only if you have *no* data at all do you propagate the error.

**(b) Two-pass normalization & healing (`_build_normalized_index`) — the centerpiece.**

*First*, index the DTs from `/portal/dts`: `dt_canonical_names`, `dt_capacity`, `dt_feeder`.

*Pass 1* — build, for each of the 7 levels, two maps: `code_to_name` and `name_to_code`. You **seed the `dt` level first with the canonical names from `/portal/dts`**, then walk every meter and fill in mappings using `setdefault` (so the **first/seeded value wins** and is never overwritten). This is how canonical DT names beat conflicting meter-embedded names.

*Pass 2* — for each meter, heal every node via `_heal_node`, then assemble a `MeterRecord`, join `dt_capacity_kva` by DT code, parse geo, coerce `build` to the enum (unknown → `legacy` + warning), and tally the transformer accumulators (`meter_count`, status buckets).

`_heal_node` logic:
- **code `""`, name present** → look up `name_to_code[level][name]`; if found, fill the code + warn ("inferred missing code"). (Trap A, missing code)
- **name `""`, code present** → look up `code_to_name[level][code]`; fill the name + warn. (Trap A, missing name)
- **both present but name ≠ canonical** → overwrite with canonical, stash the original in `raw_name`, warn ("normalized name"). (Trap B)
- **both empty** → warn, leave `None`.

Then transformer summaries are built for **all 40 DTs** (union of DTs seen in the catalogue and in meters), with `status_breakdown` always showing the three buckets `Installed/Faulty/Decommissioned` even when zero.

**Why two passes?** You can't heal a meter whose circle code is blank until you've *seen* another meter that has both that circle's code and name. So pass 1 learns the full vocabulary; pass 2 applies it. A single pass would miss repairs for records encountered before their "teacher" record.

**(c) The four data traps (know the exact numbers — 25 total warned meters):**
- **Trap A — blank fields**: 14 meters miss a `code`, 8 miss a `name` (22 meters). Healed by cross-referencing the maps. Example IDs: missing code `J100011`, `J100097` (both C-06); missing name `J100218` (C-01).
- **Trap B — conflicting DT-007 name**: 3 meters (`J100400/401/402`, the `GN*` serials) call DT-007 "Old Malviya Nagar Xfmr" while `/portal/dts` calls it "Sanganer DT 7". You keep the catalogue name, preserve the original in `raw_name`. (22 + 3 = **25** meters with warnings.)
- **Trap C — non-tree overlap**: the first 400 meters assign levels via **modular arithmetic** over moduli that don't divide evenly (3/6/10/14/18/28/40). So a division code appears under multiple circles, and feeder F-001 appears under two substations — the "hierarchy" is a **DAG, not a strict tree**. Your insight: each **DT** still has exactly one unique lineage, so DT code is a safe join key and the tree is deterministic if built per-meter.
- **Trap D — string/blank numerics**: everything numeric is a string, with `""`/`"—"` sprinkled in; `_parse_number` maps a null-token set to `None`.

**(d) Querying / filtering / spatial (`list_meters`).**
- A local `matches(rec)` closure applies all filters (case-insensitive): `q` substring over `meter_id`+`serial_no`, attribute equality (`status`, `make`, `phase_type`, `install_type`, `build`), per-level hierarchy code filters, and `has_warnings`.
- **Spatial search**: if `near_lat`/`near_lng` given, compute **Haversine** distance to each meter with geo, drop those beyond `radius_km`, and **sort nearest-first**. Meters without coordinates are excluded when a proximity point is supplied.
- Then paginate with `offset:offset+limit`, and compute `page = offset//limit + 1`.

**Haversine** = great-circle distance on a sphere from lat/lng. You use Earth radius 6371.0088 km. Be ready to explain it's an approximation (Earth isn't a perfect sphere; for city scale the error is tiny). The scale caveat: this is an **O(n) linear scan** per query — fine for 403 meters, but past ~50k–100k you'd move to PostGIS with a spatial (GiST) index.

**(e) Hierarchy tree (`get_hierarchy_tree`).**
- `physical` mode = `substation→feeder→dt` (18 roots); `full` = all 7 levels (3 zone roots). Both sum to 403.
- Build a nested dict keyed by `code or name or "__unknown__"`, incrementing `meter_count` at **every** node along each meter's path (so counts are **inclusive** of descendants), then "freeze" that mutable dict into immutable `HierarchyTreeNode` pydantic models, sorted by key. Building with plain dicts first and converting at the end is a clean, fast way to assemble a recursive structure.

**(f) Consumption (`get_meter_consumption` + `_normalize_consumption`) — the delta math.**
- Per-meter TTL cache (60s), guarded by its own lock.
- Parse each row: timestamp `"%d/%m/%Y %H:%M"` → **IST-aware** datetime (emitted as ISO-8601 with `+05:30`); `kwh`/`kvah`/`voltR` via `_parse_number`.
- Sort chronologically (unparsable timestamps sink to the end deterministically).
- Compute deltas: `interval_kwh = round(max(0.0, cur - prev), 4)`. The `max(0, …)` guards against a register **reset/rollover** producing a negative "usage".
- **The elegant bit**: `prev_kwh`/`prev_kvah` advance **only when the current reading is present**. So one blank reading in the middle doesn't corrupt its neighbors — the next valid reading differences against the last *valid* one.
- `power_factor = round(min(1.0, interval_kwh/interval_kvah), 4)` only when `interval_kvah > 0` (avoid divide-by-zero; clamp to physical max 1.0).
- Summary: counts, window start/end, totals, average/peak interval kWh (+ its timestamp), min/max/avg voltage.
- Optional `start_time`/`end_time` window: deltas are computed over the **full** series first, *then* filtered — so the first in-window interval still shows a correct delta relative to the reading just before the window. Naive bounds are assumed IST so comparisons with tz-aware timestamps don't throw.

## 5.5 `app/api/routes.py` + `deps.py` — the HTTP layer
- An `APIRouter(prefix="/v1")` with the endpoints. Each route declares typed query params with `Query(...)` (FastAPI validates and documents them: `radius_km` has `ge=0`, `limit` `ge=1, le=1000`, `mode` is a `Literal["physical","full"]`).
- Routes are thin: they parse/validate input, call the service, and map "not found" to `HTTPException(404)`. Business logic lives in the service, not here.
- `deps.get_meter_service(request)` pulls the singleton service off `app.state`. Using a **dependency** (rather than a global import) is what lets tests swap in a fake via `app.dependency_overrides`.

## 5.6 `app/main.py` — app assembly
- **Lifespan** context manager: on startup, create the `MeterService`, store it on `app.state`, and **warm the catalog** (so the first user request is fast). If warm-up fails (`PortalUpstreamError`), log and continue — the app still boots and loads lazily. On shutdown, `aclose()` the client.
- A global **exception handler** for `PortalUpstreamError` → clean JSON `{"error":"upstream_error","detail":...}` with status 502. One place translates upstream failures into a consistent response.
- `GET /health` returns `status`, `catalog_cached_meters`, `catalog_stale` — a real readiness probe, not a trivial `{"ok":true}`.
- FastAPI auto-serves **Swagger at `/docs`** and **ReDoc at `/redoc`** from the OpenAPI schema it derives from your routes + Pydantic models.

---

# 6. Cross-cutting themes (interviewers love these)

**Concurrency / async.** FastAPI runs on an async event loop. `httpx.AsyncClient` lets many requests share connections without threads. The two `asyncio.Lock`s (auth, catalog) + double-checked locking prevent stampedes. Note: `asyncio.Lock` protects against interleaving between `await` points on a **single event loop** — it is *not* cross-process; that's exactly why your "multiple workers need Redis" criticism is valid.

**Caching strategy — two tiers, different TTLs.** Catalog (slow to build, rarely changes) at 300s; per-meter consumption (changes more, cheap-ish) at 60s. TTL + explicit `/v1/catalog/refresh` + stale-on-error. The "how do you decide stale?" answer: time-based TTL, plus a stale flag surfaced to clients so they *know* when they're reading old data.

**Error handling philosophy.** Distinguish (a) not found → 404/`None`, (b) transient → retry with backoff, (c) auth lost → re-auth once, (d) upstream broken → 502 with a clean body. Never leak a stack trace; never retry a non-retryable 4xx.

**Idempotency / read-only.** The whole service is read-only over the portal (you explicitly skipped writes), so retries are always safe.

---

# 7. Testing strategy (`tests/test_service.py`)
- A **`FakeClient`** implements the same async methods as `UrjaPortalClient` but returns canned dicts — so the entire suite runs **offline**, no network, no credentials.
- Unit tests: HMAC determinism + format; the three healing cases (missing code, missing name, DT-007 conflict); consumption deltas, PF, PF-clamp, and blank-reading handling.
- Endpoint tests use FastAPI's `TestClient` with `app.dependency_overrides[get_meter_service]` → the fake-backed service. Deliberately **not** using the `with TestClient(app)` form, so the real lifespan (which hits the live portal) never runs during tests.
- 15 tests, all green. This demonstrates the payoff of the layered design: you can test behavior without the portal.

---

# 8. Design decisions & trade-offs (say the decision, the alternative, and why)

| Decision | Alternative | Why you chose it |
|---|---|---|
| Direct HTTP + reproduce HMAC | Headless browser (Playwright) | Lightweight, fast, deterministic; no browser flakiness. Browser would survive UI changes but is huge overkill. |
| In-memory TTL index | Pass-through proxy | Catalog is tiny (403) and expensive to assemble; caching enables filtering/spatial/tree that a proxy can't. |
| Lenient healing + `data_warnings` | Strict rejection | Operators need the data even when imperfect; nothing hidden, everything auditable. |
| Lock-protected shared session | Login per request | Avoids stampede, respects the portal, reuses the cookie. |
| Stale-while-error | Fail on refresh error | Read-mostly data; availability > perfect freshness. |
| 502 for upstream errors | 500 | It's the *upstream's* fault, not a bug in our code; 502 is semantically correct. |

---

# 9. Honest weaknesses (have these ready — they *will* ask)

1. **Single-process, in-memory state.** Multiple Uvicorn workers → each has its own cache, its own login, its own refresh timer (inconsistent, wasteful). Fix: shared session + cache in **Redis**.
2. **Cold-cache latency on consumption.** First request for a meter pays the full upstream round-trip; a burst of distinct meters = a burst of upstream calls. Fix: background pre-fetch + durable store.
3. **7-day data horizon.** The portal only keeps a week; no history. Fix: a background collector writing intervals into **PostgreSQL/TimescaleDB**.
4. **Linear scans** for filter/spatial — fine at 403, not at 100k. Fix: Postgres + PostGIS indexes.
5. **No auth on *your* API** — anyone who can reach it gets the data. In production you'd add API keys / OAuth.
6. **Signing secret in memory only**, re-fetched — fine, but you depend on `/portal/keys` staying available.

---

# 10. Rapid-fire Q&A you should be able to nail

- **"Why async?"** The service is I/O-bound (waiting on the portal). Async lets one process handle many concurrent requests efficiently without a thread per request.
- **"What's double-checked locking and why?"** Check cheaply without a lock (fast path); if it looks stale, take the lock and check again (correctness), so only one task does the expensive work. Prevents stampedes.
- **"Why `follow_redirects=False`?"** So a 3xx to `/login` is detectable as "session dead" instead of silently fetching an HTML login page.
- **"Why newline-joined canonical string for HMAC?"** It's an unambiguous, order-fixed serialization so client and server hash *exactly* the same bytes; newlines are delimiters that can't appear in method/path/query/timestamp.
- **"How do you get interval usage from cumulative registers?"** Subtract consecutive readings; clamp negatives (resets) to 0; skip blanks by only advancing the "previous valid" pointer when a value exists.
- **"Why clamp PF to 1.0?"** Power factor is kWh/kVAh and is physically ≤ 1; values above 1 indicate noisy/rounding data, so clamp.
- **"Why is the hierarchy not a tree?"** Levels are assigned by modular arithmetic with non-dividing moduli (Trap C), so codes repeat under different parents — it's a DAG. DT lineage is still unique, so you build the tree per-meter and aggregate.
- **"How does the fake client enable testing?"** It satisfies the same interface the service depends on, so you inject it and run the real business logic with zero network.
- **"Where would this break at scale and what changes?"** ~50k–100k meters: move catalog to Postgres+PostGIS, cache/session to Redis, consumption to TimescaleDB with background ingestion, add indexes; the layering means only `app/client`/a new repository layer changes, not the API.

---

If it'd help, I can (a) turn this into a `STUDY_NOTES.md` you can keep open while you revise, (b) quiz you with mock interview questions and critique your answers, or (c) go even deeper on any one area (HMAC internals, asyncio event loop, Pydantic v2, or the healing algorithm). Which do you want?

---

# 11. Preparing for the follow-up conversation ("could a teammate understand this and pick it up?")

This line tells you the *format* of the interview, and it's different from a technical quiz. Let me decode it and give you a concrete game plan.

## What this conversation actually is

It's a **walkthrough / pairing-style chat**, not a whiteboard algorithm test. They want to see:
1. **Can you narrate your own thinking?** (problem-solving story, not memorized facts)
2. **Is the code something a teammate could inherit?** That exact phrase — *"could a teammate understand this and pick it up?"* — is the lens they'll judge by. So they'll probe: is it readable, is the reasoning documented, where would a newcomer get stuck.

Translation: they care more about **how you reason and communicate** than whether you recall a line of code. Your `STUDY_NOTES.md` already makes you strong on substance; now prepare the *delivery*.

## Concrete things to prepare

**1. A 2-minute spoken narrative of the journey** (they'll likely open with "walk us through it")
Tell it as a story, not a feature list:
- "No API existed, so first I explored the portal as a logged-in user and watched the network tab."
- "Login failed until I realized SvelteKit needed the `Origin` header for CSRF."
- "The bulk export rejected me, I read the compiled JS chunk and found the HMAC signing scheme."
- "Then I noticed the data was dirty, 25 meters with issues, so I built healing that repairs but records every change."
- "I wrapped it all in a layered FastAPI service with caching and tests."

Practice saying that out loud 3-4 times until it's natural. The reverse-engineering arc is your strongest material because the assignment literally said problem-solving is what they weigh most.

**2. Be able to do a live demo** (have the server runnable in 30 seconds)
- Know the two start commands cold (`.\venv\Scripts\python.exe -m uvicorn app.main:app --port 8000`).
- Open `/docs`, hit `/v1/meters/J100400` to show the DT-007 healing live, and `/v1/meters?has_warnings=true` to show the 25. A live demo of the healing is far more convincing than describing it.

**3. The "teammate pickup" answer** — have a crisp response to *"if a new engineer joined, how would they get started?"*
- "They'd read the `README` first (run instructions + examples), then `PROTOCOL.md` to understand the weird portal, then the code top-down: `main` → `routes` → `service` → `client`."
- "Each layer has one job, so they only need to understand the layer they're changing."
- "The tests document expected behavior and run offline, so they can refactor safely and get instant feedback."
- "Every non-obvious decision has a comment explaining *why*, and healing decisions are logged in `data_warnings` so the data is self-explaining."

**4. Pick 1-2 code spots you're proud of and can explain deeply**
- The **double-checked locking** in `ensure_authenticated` (prevents login stampede).
- The **two-pass healing** (why you can't do it in one pass).
Having a "favorite" bit shows ownership.

**5. Know where you'd be challenged and own it** (don't get defensive)
- Single-process in-memory state → Redis at scale.
- Cold-cache consumption latency.
- No auth on your own API.
Saying these *before* they do signals maturity. Your `REFLECTION.md` already has them — reread it the morning of.

## Likely opening questions and your angle

| They'll ask | Your angle |
|---|---|
| "Walk us through what you built." | The 2-min narrative above. |
| "What was the hardest part?" | HMAC discovery from the JS chunk + the byte-for-byte timestamp trap. |
| "Why did you structure it this way?" | Layering → testability + swappability + one-reason-to-change. |
| "What would you do differently / at scale?" | The weaknesses list — Redis, TimescaleDB, PostGIS. |
| "How would a teammate extend it, say add a new endpoint?" | "Add a method to the service, a thin route in `routes.py`, a test with the fake client, done, OpenAPI updates itself." |
| "Why heal instead of reject bad data?" | Operators need the data; heal + record in `data_warnings` = usable *and* auditable. |
| "Any bugs or things you're unsure about?" | Honest answer > pretending it's perfect. Mention cold-cache tail latency. |

## Behavioral tips for a walkthrough
- **Share your screen with the repo open.** Navigate the actual files; don't recite from memory.
- When unsure, **think out loud** — they're evaluating reasoning, so "I'd check X, because Y" beats silence or bluffing.
- If they suggest an improvement, **engage with it** ("good point, that'd help because…") rather than defending. Collaboration signal.
- Tie answers back to the **teammate lens**: "I named it that way so a new reader wouldn't have to guess."
- It's fine to say **"I don't know, here's how I'd find out."** That's a learning-ability signal, which they explicitly rank above speed.

## The night before / morning of
- Reread `STUDY_NOTES.md`, `REFLECTION.md`, and `PROTOCOL.md` once each.
- Start the server once to confirm it still runs and the portal session still works.
- Have the repo open in your editor and the browser `/docs` bookmarked.
