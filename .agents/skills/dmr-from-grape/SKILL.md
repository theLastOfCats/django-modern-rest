---
name: dmr-from-grape
description: Port a Ruby/Rails service whose HTTP API is built with Grape to Python/Django/django-modern-rest. Use when re-implementing Grape resources, params blocks, grape-entity presenters, rescue_from handlers, rack-attack limits, devise_token_auth headers, or a rswag-covered API surface as Django apps with dmr controllers, typed DTOs, routers, and typed error handling.
---

# DMR from Grape

## Overview

Port a Ruby service — a Rails app whose HTTP layer is `Grape` — to `Django` + `django-modern-rest` (DMR).

This is a **re-implementation**, not a translation:
no Ruby code, no Ruby gems, and no Ruby runtime survive the port.
The deliverable is contract parity — paths, methods, status codes, headers, body shapes, error payloads,
auth/throttle/pagination/negotiation semantics, and data semantics — not code parity.

Because the domain layer is re-written in Python,
the whole service is in scope: persistence, jobs, mail, storage, and the test suite,
not just the transport layer.

Required docs (read before porting):
- DMR LLM docs: https://django-modern-rest.readthedocs.io/llms-full.txt
- DMR best practices: load the `$dmr` skill for this repository
- Grape README (§Routes, §Parameters, §Validation, §Exception Handling)
- grape-entity README
- grape-swagger README, rswag README
- Rails guides: Active Record, Active Record Migrations, Active Job, Action Mailer, Active Storage
- devise_token_auth / doorkeeper, rack-attack, rack-cors, kaminari / api-pagination / pagy READMEs as used by the project

Local references — read the one you need, when you need it:
- [references/grape-to-dmr-map.md](references/grape-to-dmr-map.md) —
  read when porting any HTTP surface: URL assembly, routes, params, entities, statuses, errors, filters, versioning,
  negotiation, pagination, throttling, CORS, wire format.
- [references/rails-to-django-map.md](references/rails-to-django-map.md) —
  read when porting persistence, domain logic, jobs, mail, storage, auth, config, idioms, or the test suite.
- [references/differential-parity.md](references/differential-parity.md) —
  read at the start (to freeze the contract) and at the end (corpus replay, fuzzing, cutover gate).

Check the project's `Gemfile.lock` first:
`grape`, `grape-entity`, `grape-swagger`, `rswag`, and the pagination/throttling gems pin the exact behavior described
here. If the pinned version differs from the current README, verify the specific behavior before claiming a mismatch.

## Read This First

- **Parity frontier.** Ruby dynamics are re-implemented, not translated:
  coercion, `blank?`/`present?`, ActiveRecord callbacks, metaprogramming, and `rescue_from` have no mechanical
  equivalent. Each is either reproduced explicitly or listed as approved drift.
- **The frozen contract is the sole authority** for what the API does. Inventories, memory, and the DSL reading are
  hypotheses; `port/contract/` plus the corpus are the evidence.
- **Three security surfaces must never silently change:**
  1. password-hash verifiability (existing bcrypt hashes must keep verifying),
  2. token rotation and TTL behavior,
  3. rate limits and CORS credentials semantics.
  Any deviation is a `security posture changes` report line, not an implementation detail.
- **Two verification tiers.** Tier 1 (ported pytest suite + `validate_responses`, per slice) is mandatory.
  Tier 2 (recorded-corpus differential replay against the booted Ruby service, plus fuzzing) is required whenever the
  Ruby service can be booted. Endpoints that cannot be replayed are reported `unverified`.

## Migration Policy

- Default mode: **contract parity** — paths, methods, status codes, headers, body shapes, error payloads, and
  auth/throttle/pagination/negotiation semantics preserved.
- **Phase gate:** no Python application code before `port/contract/` exists
  ([differential-parity.md §2](references/differential-parity.md#2-contract-capture)).
- If the user approves drift (for example DMR-native error payloads), stop forcing legacy behavior and record the
  deviation as an `approved drift` line with the affected endpoint and field.
- **No implicit drift.** Every intentional mismatch is listed; nothing is "close enough".
- Default DB strategy: one physical schema shared by both services (identical `db_table`/`db_column`) until cutover.
  Differential reads use the same snapshot; mutating replays use cloned/restored copies so the first service cannot
  change the second service's starting state. Any other strategy is recorded in the parity report.
- The legacy Ruby service is deleted only in the cutover commit
  ([differential-parity.md §7](references/differential-parity.md#7-cutover-procedure)).
- Never weaken security to simplify the port: hashers, token rotation, rate limits, and CORS credentials keep their
  semantics or become an explicit security-drift line.

## Workflow

### 1. Inventory the Ruby service

- Record the gem set and versions from `Gemfile`/`Gemfile.lock`.
- Locate the API: `app/api`, `lib/**`, `config/routes.rb` mounts, `Grape::API` subclasses, `rescue_from` handlers,
  entity classes, `helpers` blocks, `use`/middleware, `params do … end` blocks.
- Locate the domain: `app/models`, `db/schema.rb` (or `db/structure.sql`), `app/jobs`, `app/mailers`,
  `config/initializers` (`rack-attack`, `rack-cors`, `active_storage`), `config/application.rb`.
- Locate the tests: `spec/requests` (rswag specs, request specs), fixtures/factories, `spec/rails_helper.rb`.
- Useful greps: `Grape::API`, `mount `, `params do`, `requires`, `optional`, `route_param`, `rescue_from`, `error!`,
  `present `, `expose `, `before do`, `helpers do`, `throttle(`, `allow do`, `belongs_to`, `has_many`,
  `before_save`, `after_commit`, `after_create`, `perform_later`, `deliver_later`, `has_one_attached`, `enum `.
- Artifact: `port/inventory.md` — one row per endpoint group plus the domain/job/mail/storage/test inventories.

### 2. Freeze the contract (required gate)

- Capture the API document in this priority order: rswag → grape-swagger → hand-built
  ([differential-parity.md §2](references/differential-parity.md#2-contract-capture)).
- Store it under `port/contract/`. This is the input to the contract diff, the corpus, and the fuzz target.
- Artifact: `port/contract/` plus a note recording which source was used and which endpoint groups it does not cover.

### 3. Decide the port strategy (required gate)

Record all five decisions in `port/parity-report.md` before coding:

- **Parity mode**: contract parity (default) or explicitly approved drift, with the drift list started.
- **Error-payload strategy**: Grape-shaped error payloads or DMR-native errors — one of the two, app-wide.
- **Target layout**: extend an existing Django repo, or bootstrap a new one with `wemake-django-template`.
- **Serializer plugin**: one per app. `MsgspecSerializer` is the default; use `PydanticSerializer` for a controller
  whose DTOs need cross-field validators, arbitrary coercers (`coerce_with:`), or per-field serializers
  (`format_with`) — justified inline ([grape-to-dmr-map.md §3](references/grape-to-dmr-map.md#3-params-dsl-to-dto-constraints)).
- **DB strategy**: shared physical schema (default) or seeded fixtures.
- Also record the vertical slice order: one Grape resource (or mount) per slice.

### 4. Port persistence

- Port the resulting schema, not the migration files:
  [rails-to-django-map.md §1](references/rails-to-django-map.md#1-schema-source) –
  [§3](references/rails-to-django-map.md#3-migrations).
- Keep `db_table`/`db_column` literal in shared-schema mode; port associations, enums (read the ordinals from the
  original migration), scopes, validations, and counter caches.
- Gate: `manage.py makemigrations --check --dry-run` clean plus a column/index/nullability diff against `db/schema.rb`.

### 5. Port domain logic

- Services, not callbacks and not signals:
  [rails-to-django-map.md §4](references/rails-to-django-map.md#4-callbacks-and-domain-logic) –
  [§8](references/rails-to-django-map.md#8-storage).
- `after_commit` side effects become explicit `transaction.on_commit(...)` calls.
- Ruby metaprogramming is enumerated and re-written explicitly; never emulated.

### 6. Port auth and security

- [rails-to-django-map.md §9](references/rails-to-django-map.md#9-auth-and-credentials).
- Implement a Devise-bcrypt compatibility verifier/writer; stock Django does not recognize bare `$2a$…` digests,
  and stock `set_password()` writes a prefixed value Ruby cannot verify. Prove an existing and a Django-changed
  password through both services before proceeding.
- Reproduce the client-visible auth headers, token rotation, and TTLs
  ([grape-to-dmr-map.md §8](references/grape-to-dmr-map.md#8-filters-helpers-middleware-auth)).

### 7. Port throttling and CORS

- [grape-to-dmr-map.md §12](references/grape-to-dmr-map.md#12-throttling-and-cors).
- Preserve limits, periods, keys, and the 403-vs-429 distinction; wire a real `CACHES` backend for throttling.
- Port origins/methods/headers/expose/credentials/max-age explicitly; never widen CORS to make a test pass.

### 8. Port transport per slice

- One Grape resource → one DMR app module:
  [grape-to-dmr-map.md §1](references/grape-to-dmr-map.md#1-api-composition-and-url-assembly) –
  [§10](references/grape-to-dmr-map.md#10-negotiation).
- All errors go through the shared error module (`app/api/errors.py`): the exception → (status, payload) table, the
  project error model, and the 404/405/500 handlers
  ([grape-to-dmr-map.md §7](references/grape-to-dmr-map.md#7-errors)).
- Split the single Grape `params` block into `Path[...]`/`Query[...]`/`Body[...]`/`Headers[...]`/`Cookies[...]`.
- Gate per slice: ported pytest tests for that slice pass with `validate_responses` enabled.

### 9. Port pagination and envelopes

- [grape-to-dmr-map.md §11](references/grape-to-dmr-map.md#11-pagination).
- Keep param names, defaults, clamping, envelope key, and every pagination header.

### 10. Port the test suite

- [rails-to-django-map.md §12](references/rails-to-django-map.md#12-test-suite-port).
- Keep one test per behavior; add a negative-path test for every declared response.

### 11. Differential verification

- Only when the Ruby service is bootable:
  [differential-parity.md §4](references/differential-parity.md#4-corpus) –
  [§6](references/differential-parity.md#6-fuzzing).
- Otherwise record the reason and mark the affected rows `unverified`.

### 12. Cutover gate

- [differential-parity.md §7](references/differential-parity.md#7-cutover-procedure).
- Delete the Ruby service and the `port/` harness only in the cutover commit.

## Translation Rules

- Use DMR constructs; never port Grape/Rails internals as an implementation strategy.
- No Ruby files, gem names, or Ruby-style abstractions in the ported app.
- One serializer plugin per application.
- Errors are raised and rendered only through the shared error table — no endpoint builds an error response inline.
- Every non-default response is declared (`extra_responses=[ResponseSpec(...)]`).
- `validate_responses` is on outside production.
- Business logic lives in services called by controllers — never in DTOs or controllers.
- Security semantics (hashers, tokens, limits, CORS credentials) are preserved or listed as security drift.
- The wire format is a serializer/renderer concern, not ad-hoc dict post-processing.
- In shared-schema mode, table and column names stay literal until cutover.

## Reporting Format (required)

At each meaningful checkpoint and at completion, report in 4 sections:

1. `preserved behavior`
2. `approved drift`
3. `security posture changes`
4. `unresolved gaps`

`unresolved gaps` must name concrete endpoints/files and why they are unresolved.

Also maintain the endpoint matrix, updated every slice:

| Ruby source (path + verb) | Django target | Status | Notes |
|---|---|---|---|
| `app/api/users.rb` `GET /api/users` | `apps/users/views.py` `UserListController` | `ported` | envelope verified |
| `app/api/reports.rb` `POST /api/reports` | `apps/reports/views.py` `ReportCreateController` | `verified` | corpus replay clean |

Status is exactly one of `ported`, `verified`, `drifted`, `unverified`.

## Migration Pitfalls

1. Grape's 404 body is `["404 Not Found"]` with `X-Cascade: pass`. Its 405 exception message is
   `405 Not Allowed`, but the rendered body is formatter-specific (`{"error":"405 Not Allowed"}` under the JSON
   formatter); `Allow` and auto-OPTIONS behavior are separate contract fields. Capture them, then rebuild the exact
   body and headers with `build_404_handler`/error handlers when clients depend on them.
2. `error!` without a status uses `default_error_status`, else **500** — not 400.
3. Validation failures are 400 in both frameworks, but the payload differs (`{"error": "..."}` vs DMR's
   `{"detail": [{"msg": ...}]}`).
4. `""` means absence for every Grape type except `String`; DTO validation does not do that implicitly.
5. Query arrays use the Rack bracket form `?ids[]=1&ids[]=2`, and a comma string is a 400 (`... is invalid`).
   Match the client encoding with `__dmr_force_list__` / `__dmr_split_commas__`.
6. Path params outrank query/body params of the same name; `route_param` outranks a declared same-named param.
7. Undeclared params are ignored by Grape — do not enable extra-field rejection unless the contract shows clients
   rely on it.
8. `(.:format)` suffixes (`/users/1.json`) match in Grape and dots are excluded from captures; DMR has no such route.
9. `format :json` gives 404 for unknown extensions and 406 for `?format=xml`, but `Accept: application/xml` still
   returns JSON; DMR answers 406 for an unsatisfiable `Accept`.
10. An unsupported request body Content-Type is 415 in Grape and 400 in DMR.
11. POST → 201 in both; DELETE with a blank body → 204 (`return_type=None` is required in DMR); `body false` → 204.
12. Grape's `before` also runs for 405s and the auto-OPTIONS handler runs `before`+`after`; `after` is skipped on
    validation failure while `finally` always runs. Django middleware is not per-route — state which filter behaviors
    survive.
13. `rescue_from` is scope-local, first-registered-in-scope wins, must sit above its routes, and re-raises redispatch
    exactly once.
14. Uncaught exceptions (including `RecordNotFound` when the service never rescued it) become a generic 500 in Grape.
    Check the real behavior before "improving" it to 404.
15. ActiveSupport JSON emits 3-decimal UTC times (`...T12:00:00.000Z`), HTML-escapes `<`/`>`/`&`, and stringifies
    `BigDecimal` — DMR defaults differ. Byte-compare, then fix or drift-list.
16. ActiveRecord touches `updated_at`; Django `auto_now` does not fire on `.update()` or bulk updates.
17. ActiveRecord `enum` array form maps to ordinals from 0 — read the value from the original migration, not the
    symbol names.
18. `counter_cache` columns drift unless maintained with `F()` updates on the Django side.
19. devise_token_auth rotates tokens on every request by default and returns blank `access-token`/`expiry` inside the
    5 s batch window; dropping rotation silently lengthens sessions — a security drift that must be listed.
20. Devise stores raw bcrypt (`$2a$…`); Django verifies it only when a bcrypt hasher is in `PASSWORD_HASHERS`. Put
    `BCryptPasswordHasher` first to keep producing hashes both services can verify during dual-run.
21. rack-attack answers 403 for blocklists and 429 for throttles; `Retry-After` is opt-in. DMR emits `X-RateLimit-*`
    and `Retry-After` — new headers are drift.
22. rack-cors rejects `credentials: true` with `origins '*'`; port origins/methods/headers/expose/max_age (default
    7200) explicitly and keep that invariant.
23. Grape's single `params` block mixes sources — the split into `Body`/`Query`/`Path` is where ports break; verify it
    against the corpus, not against the DSL alone.
24. Grape normalizes trailing slashes and percent-encodes escapes (captures arrive decoded); Django `APPEND_SLASH`
    differs — pick one behavior and prove it in the differential run.

## Output Checklist

- Contract frozen under `port/contract/` and diffed against `build_schema(router)` — no unexplained rows.
- Every endpoint in the frozen contract is `ported`, `verified`, or explicitly `drifted`.
- Error payloads produced by the shared error table only; 404/405/500 shapes decided and tested.
- Request sources split into `Path`/`Query`/`Body`/`Headers`/`Cookies`, with array/blank semantics reproduced or
  drift-listed.
- Auth: existing and Django-changed passwords verified through both services; opaque-vs-JWT token format, rotation,
  refresh, revocation, and TTL behavior preserved or listed as security drift.
- Throttling limits, periods, keys, and the 403-vs-429 distinction preserved; cache backend configured.
- Pagination envelope, parameter names, defaults, clamping, and headers preserved.
- CORS origins/methods/headers/expose/credentials/max-age preserved; credentials plus wildcard rejected.
- Wire format byte-compared on at least one endpoint per body shape; deviations fixed or drift-listed.
- Test suite ported (one test per behavior) with a negative-path test for every declared response; `validate_responses`
  enabled.
- `port/` harness run and its output attached, or the reason it could not run recorded per endpoint group.
- Parity report emitted in the required 4-section format plus the endpoint matrix.
- Ruby service, gems, and the `port/` harness removed only in the cutover commit.
