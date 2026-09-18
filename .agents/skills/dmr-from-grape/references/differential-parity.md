# Differential parity

Verification reference for a Grape → DMR port. Grounds every Ruby fact in the Grape/Rails/rswag sources and every
DMR construct in the repo's public API. Wire-format comparison rules live in
[grape-to-dmr-map.md §13](grape-to-dmr-map.md#13-wire-format); test-suite porting lives in
[rails-to-django-map.md §12](rails-to-django-map.md#12-test-suite-port).

Artifact tree produced by this reference (all paths relative to the repository root):

```text
port/
  contract/
    openapi.yaml          # frozen transport contract (OpenAPI 3.x or Swagger 2.0; dialect recorded in SOURCE.md)
    openapi.sha256        # checksum pinning the frozen document
    SOURCE.md             # capture command, gem versions, Ruby commit sha, capture date
    endpoints.md          # hand-built fallback contract, only when no doc generator exists
  corpus/
    <group>.jsonl         # recorded request/response pairs
  harness/
    compare_contract.py   # frozen contract vs build_schema(router)
    replay.py             # corpus replay + diff
    normalize.py          # normalization rules shared by replay.py
  parity-report.md        # per-endpoint matrix + drift list (reporting format: SKILL.md)
```

## 1. Tier 1: per-slice gate

Mandatory for every slice (one Grape resource group → one DMR app module; see `SKILL.md` workflow step 8). A slice is
"ported" only when all four conditions hold:

|#|Condition|How it is checked|
|---|---|---|
|1|Ported pytest suite green for the slice (`spec/requests` equivalent ported per [rails-to-django-map.md §12](rails-to-django-map.md#12-test-suite-port))|`uv run pytest tests/<slice>/ -x`|
|2|`DMR_SETTINGS['validate_responses']` enabled outside production|`DMR_SETTINGS = {'validate_responses': True}` in test/dev settings; production may set it `False` for speed|
|3|Every contract row for the slice satisfied (paths, methods, statuses, headers, body shapes, error payloads)|contract rows from `port/contract/`; diffed by `port/harness/compare_contract.py` (§3)|
|4|Drift list in `port/parity-report.md` updated (every non-parity is an explicit `approved drift` row)|report read at slice review|

Notes:

- Tier 1 is the **only** gate when tier 2 (differential replay) is unavailable.
- Tier 2 never replaces tier 1; it is additive.
- `validate_responses` may be relaxed per controller or per endpoint, and for statuses no controller can describe
  (for example `500` raised from anywhere), but NEVER disable the setting app-wide outside production.

## 2. Contract capture

Freeze the transport contract **before** writing Python. Capture in priority order; record the outcome in
`port/contract/SOURCE.md`.

|Priority|Source|Capture command|Frozen artifact|
|---|---|---|---|
|(a)|rswag (gems `rswag-specs`/`rswag-api`/`rswag-ui`)|`RSWAG_DRY_RUN=0 bundle exec rake rswag:specs:swaggerize` (alias `rake rswag`)|`openapi/v1/openapi.yaml` (`openapi: '3.0.1'`); older installs write `swagger/v1/swagger.yaml`|
|(b)|grape-swagger|`bundle exec rake oapi:fetch store=true`, validate with `bundle exec rake oapi:validate`|document served at the mount path, default `/swagger_doc` (Swagger **2.0**)|
|(c)|neither present|hand-built `port/contract/endpoints.md` from routes + specs|`port/contract/endpoints.md`|

Details per source:

- **(a) rswag** — specs live in `spec/requests` (plus `spec/api`, `spec/integration`); the DSL is
  `path`, verb blocks, `parameter name:, in:, type:/schema:, enum:`, `security [basic_auth: []]`, and
  `response '201' do schema/header/example/run_test! end`. `run_test!` executes and validates the request only when
  dry-run is disabled, so `RSWAG_DRY_RUN=0` is part of the capture command. Specs marked `document: false` are excluded
  from the document — check that no operation you must port is excluded before trusting the capture.
- **(b) grape-swagger** — installed through `add_swagger_documentation` on the mounted API class; the configured
  `mount_path` defaults to `/swagger_doc`, and the output is Swagger **2.0** (the README documents no OpenAPI-3
  switch). Rake tasks are registered with `GrapeSwagger::Rake::OapiTasks.new(::Api::Base)`; `rake oapi:fetch
  store=true` writes the fetched document to disk. Per-parameter metadata comes from `documentation:` keys
  (`param_type`, `type`, `collectionFormat(csv|ssv|tsv|pipes|multi)`, `hidden`, `default`, `example`,
  `additional_properties`), and entities are emitted under `definitions` with `is_array: true` and `using:`
  relations expanded to `allOf` + discriminator.
- **(c) hand-built** — write one row per Grape route: verb, assembled path (per
  [grape-to-dmr-map.md §1](grape-to-dmr-map.md#1-api-composition-and-url-assembly)), declared params with their
  source, statuses, response shape, and error payloads. Extract from the Grape `params` blocks, the `desc` blocks, and
  the request specs. When (c) applies, tier 2 (§5) becomes mandatory as soon as the Ruby service boots.

Freeze protocol:

```bash
mkdir -p port/contract
cp <captured-doc> port/contract/openapi.yaml
sha256sum port/contract/openapi.yaml > port/contract/openapi.sha256
```

Record in `port/contract/SOURCE.md`: source used (a/b/c), document dialect (`swagger: '2.0'` or `openapi: ...`), the
exact capture command, gem versions (`bundle exec ruby -e 'puts Gem.loaded_specs["rswag-specs"].version'` and
equivalents), the Ruby commit (`git rev-parse HEAD`), capture date, and confirmation that `RSWAG_DRY_RUN=0` was used
for rswag.

The frozen document is the sole authority for the port. `compare_contract.py` must detect Swagger 2.0 versus
OpenAPI 3.x and normalize their operation/parameter/request/response representations before comparison (§3).
The version string and document layout are accepted format differences; the represented contract is not.

## 3. Contract diff script

`port/harness/compare_contract.py` is a throwaway Python script that loads the frozen document, detects its dialect,
normalizes Swagger 2.0 and OpenAPI 3.x into one internal operation model, then compares it with
`dmr.openapi.build_schema(router)`.

For every operation it asserts:

|Check|OpenAPI 3 frozen side|Swagger 2 frozen side|DMR side|
|---|---|---|---|
|path + method present|`paths[path][method]`|`paths[path][method]`|operation keys of `build_schema(router)`|
|parameter names, sources, and required flags|path-level + operation-level `parameters[]`|path-level + operation-level `parameters[]` (`in: path/query/header/formData`)|fields and required flags in `Path[...]`/`Query[...]`/`Headers[...]`/`Cookies[...]`|
|request body required flag and schema|`requestBody.required` + `requestBody.content.*.schema`|the `in: body` parameter, or the set of `in: formData` parameters|`Body[...]` schema|
|request media types|`requestBody.content` keys|operation/global `consumes`|parser content types|
|every declared response status|`responses` keys|`responses` keys|`responses` keys|
|response media types per status|`responses.*.content` keys|operation/global `produces`|renderer content types|
|response headers per status|`responses.*.headers` keys|required/optional header metadata in `responses.*.headers`|`ResponseSpec.headers`|
|success and error body schemas per status|`responses.*.content.*.schema`|`responses.*.schema`|each response's normalized schema|

Schema comparison resolves local `$ref` values and compares observable constraints: `type`, `format`, required field
set, property names, array items, enum values, nullability, and `allOf`/`oneOf`/`anyOf` branches. Description text,
component names, and the OpenAPI version are documentation/layout differences, not body-shape checks. A missing schema
or header on either side is a difference; the script must never treat an absent Swagger-2-only/OpenAPI-3-only key as
an empty successful match.

Shape:

```bash
uv run python port/harness/compare_contract.py \
  --frozen port/contract/openapi.yaml \
  --router app.api.router
```

`--router` is an import path to the `dmr.routing.Router` instance; the script calls `django.setup()`, imports it, and
calls `build_schema(router)` (optionally with `DMR_SETTINGS['openapi_config']` applied through `OpenAPIConfig`).

Output: a markdown table with one row per difference (`operation`, `check`, `frozen value`, `DMR value`). Every row is
either **fixed** (code changed) or **drift-listed** (row copied into `port/parity-report.md` under `approved drift`).
"No unexplained rows" is a cutover precondition (§7). The script is `port/`-scoped scaffolding and is deleted with
`port/` at cutover.

## 4. Corpus

Record inbound Grape request/response pairs so the ported service can be replayed (§5). Sources, in priority order:

|Priority|Source|Extraction|
|---|---|---|
|1|passing rswag/request specs|instrument the Rack request boundary used by `run_test!` / the request example and record the actual method, URI, headers, and body it sent|
|2|synthetic contract fixtures|drive the booted Ruby service with deterministic requests derived from the frozen contract and record its responses|
|3|explicit inbound capture through a local proxy or sanitized access-log replay|proxy requests **into** the Ruby service; never treat VCR cassettes as inbound API traffic because they normally contain the service's outbound HTTP calls|

Freeze as JSON lines, one object per interaction, with exactly these keys:

```json
{"method":"GET","path":"/api/v1/users/1","query":"include=posts","headers":[["Accept","application/json"],["Authorization","Bearer ${TEST_ACCESS_TOKEN}"]],"body":"","body_encoding":"utf-8","ruby_status":200,"ruby_headers":[["Content-Type","application/json; charset=utf-8"],["Set-Cookie","client=abc"],["Set-Cookie","access-token=def"]],"ruby_body":"{\"id\":1}","ruby_body_encoding":"utf-8"}
```

Rules:

- `path` is the assembled path with no query string; `query` is the raw query string (ordering preserved).
- Header collections are ordered `[name, value]` pairs, never JSON objects. This preserves repeated fields such as
  `Set-Cookie`; the harness lowercases names for comparison but keeps every value as bytes and keeps duplicate order.
- `body` and `ruby_body` are raw bytes represented as UTF-8 text when valid; otherwise base64. The corresponding
  `*_encoding` key is exactly `utf-8` or `base64`.
- Request secrets are symbolic templates such as `${TEST_ACCESS_TOKEN}`. `replay.py` resolves them from an
  uncommitted environment/secret mapping immediately before sending each request. Never send the placeholder itself
  and never commit a live credential.
- Per-response credentials that are necessarily re-minted are normalized only after asserting their header/field
  presence, blank-vs-nonblank behavior, TTL bounds, rotation state, and ability to authenticate a follow-up request
  (§5). Request-secret injection and response-value normalization are separate operations.
- Corpora derived from real traffic may contain personal or confidential data even after credential removal. Keep
  them outside git (or generate a fully synthetic corpus); deletion in a later commit does not remove repository
  history.
- One file per endpoint group (`port/corpus/<group>.jsonl`) so §5 can replay a slice in isolation.

## 5. Replay harness

`port/harness/replay.py` resolves request-secret templates, sends each interaction to the Ruby and Django services from
equivalent initial state, and compares both live responses with the recorded Ruby response. Body comparison is on
**raw bytes**, per [grape-to-dmr-map.md §13](grape-to-dmr-map.md#13-wire-format): parsing JSON loses key order, number
formatting, time precision, and HTML escaping.

Preconditions:

- Read-only requests may use one shared database snapshot. Any request that mutates state must run once per service
  against independent clones of that snapshot, or restore the snapshot before the second request; otherwise the first
  service changes the result observed by the second.
- When the physical schema cannot be shared, seed both databases from one deterministic fixture set and record that
  choice in `port/parity-report.md`.
- Boot the Ruby service with `bundle exec puma` or `bin/rails s`; boot Django with `uv run python manage.py runserver`.
- Both services use production-like HTTP settings (no debug error pages, same formatter/renderer configuration),
  deterministic clocks where timestamps are emitted, and sandboxed equivalents of external side effects.
- The secret environment contains valid test-only credentials. It is never committed or copied into replay output.

Command:

```bash
uv run python port/harness/replay.py \
  --corpus port/corpus \
  --target http://127.0.0.1:8000 \
  --ruby http://127.0.0.1:3000 \
  --secrets-env /run/secrets/port-replay.env \
  --normalize port/harness/normalize.py \
  --report port/parity-report.md
```

Comparison, after the narrow allowlist below:

|Field|Rule|
|---|---|
|status|exact integer equality|
|headers|lowercase field names, preserve duplicate occurrences and their values, then compare value bytes exactly|
|body|exact byte equality; no general JSON parsing or canonical re-encoding|

`port/harness/normalize.py` is an endpoint-specific allowlist, not a bag of regular expressions:

|Allowed rule|Required guard|
|---|---|
|drop `Date`, `Server`, or an explicitly named request-id header|name every dropped header and record it as accepted framework drift|
|replace a re-minted response credential at an exact header occurrence or JSON pointer|first assert presence, blank-vs-nonblank behavior, format, TTL bounds, rotation state, and successful follow-up authentication|
|replace a timeout/duration at an exact JSON pointer|record why the field is nondeterministic and preserve the surrounding raw bytes|
|map ids between independently seeded databases|use an explicit one-to-one map derived from stable fixture keys; assert repeated ids and foreign-key relationships stay consistent|

Never normalize arbitrary integers, UUIDs, timestamps, or all values matching a pattern. In shared-snapshot mode ids
and foreign keys must match exactly. Freeze clocks or fixtures so timestamp values match while their original lexical
form remains byte-comparable; otherwise the precision/offset bug this harness is meant to detect disappears.

Never reorder a response collection inside normalization. If the legacy service truly leaves ordering unspecified,
keep the raw-byte mismatch, record ordering as approved drift for that endpoint, and optionally add a parsed multiset
comparison as a diagnostic. The parsed diagnostic never turns the raw comparison green.

Any diff that survives normalization is either **fixed** in the Django service or **drift-listed** with endpoint +
field + reason. There is no third option, and no diff may be silently normalized away by widening a rule.

## 6. Fuzzing

Optional, and only meaningful once §2 has produced a frozen contract.

```bash
uv add --group dev schemathesis
uv run schemathesis run port/contract/openapi.yaml --url http://127.0.0.1:8000
```

`schemathesis` is not bundled with `django-modern-rest`; add it as a dev dependency. Run it against the DMR service
using the **frozen** contract so generation covers the Ruby surface (this is how schema-shape assumptions the ported
code makes get falsified), and optionally once more against the DMR-built schema exposed by
`dmr.openapi.views.OpenAPIJsonView` to catch DMR-side spec self-inconsistencies. Findings are triaged exactly like
replay diffs: fix or drift-list with endpoint + field. Fuzzing never substitutes for §1 or §5 — a fuzz run that finds
nothing does not make an un-replayed slice verified.

## 7. Cutover procedure

Ordered; each step is a precondition for the next.

1. Ported pytest suites green on all slices (tier 1, §1).
2. Corpus replay clean (§5), or every remaining row accepted as drift by the user, recorded in
   `port/parity-report.md`.
3. `port/harness/compare_contract.py` reports no unexplained rows (§3).
4. `port/parity-report.md` complete: endpoint matrix + `preserved behavior` + `approved drift` +
   `security posture changes` + `unresolved gaps`.
5. Backup snapshot of the production database.
6. Freeze writes on the Ruby service (maintenance mode / read-only route) so the final replay compares a static dataset.
7. Final replay against the frozen dataset; any new diff blocks cutover.
8. Switch traffic to Django.
9. Delete the Ruby service and the whole `port/` tree in the cutover commit (contract, corpus, harness are scaffolding,
   not deliverables).

Rollback is the reverse of step 8 while the Ruby service still exists; step 9 is therefore the point of no return and
belongs in its own commit.

## 8. When the Ruby service cannot be booted

The differential tier is skipped — explicitly, not implicitly:

- Record, per endpoint group, the concrete reason it could not be booted (missing credentials, unbuildable native gem,
  unavailable external dependency, no database snapshot, service decommissioned).
- Mark every row in that group `unverified` in `port/parity-report.md` (status column of the endpoint matrix).
- Tier 1 (§1) remains the gate, and the report must state that those endpoints were never differentially verified.
- No silent downgrade: an `unverified` row is an explicit report line naming the endpoints and the reason, never an
  omission and never a row relabelled `verified`.
- If a hand-built contract (§2c) was the only capture source and the service cannot boot, the port's highest available
  evidence is tier 1; say so at the top of the report.

## Approved drift candidates

Framework-level differences that may become explicit `approved drift` rows in `port/parity-report.md`, never silent
normalization:

- explicitly named framework headers such as `Date`, `Server`, and a framework banner header.
- OpenAPI document version and component layout: Swagger 2.0 / OpenAPI 3.x frozen document vs DMR's OpenAPI 3.1
  output. The represented operations, parameters, response schemas, and headers still must match.
- per-request re-minted credential **values**, only after the presence, format, TTL, rotation, and follow-up-auth checks
  in §5 pass.
- explicitly named request-id / trace headers generated per request.
- collection ordering where the Ruby service guarantees no order. Keep the raw mismatch and report the semantic
  multiset comparison separately.
- duration/elapsed/timeout-derived fields named by exact response location.
- timestamp instants for rows independently created by each service. Their serialized timezone, precision, offset,
  escaping, and surrounding raw bytes remain contract checks.
