# Grape to DMR mapping

Read this when re-implementing a Grape HTTP surface as DMR routers, controllers, and typed DTOs.
Only the observable HTTP contract is preserved: no Ruby construct survives the port, and no DMR row below relaxes a
contract behavior without being listed as drift in the last section.

Verify the pinned Grape version first: the README documents `4.1.0`, stable is `4.0.1`; route/params behaviors below
are asserted only for the version in the legacy `Gemfile.lock`.

## 1. API composition and URL assembly

Grape assembles a route as `mount paths → prefix → :version (only with using: :path) → namespace → path`, plus the
optional format suffix. DMR composes the same tree out of routers:

| Ruby/Grape construct | DMR/Python construct | Parity note |
|---|---|---|
| `class Api::V1::Base < Grape::API` | `Router(prefix=..., urls=[...])` + one `Controller[Serializer]` subclass per resource | DMR has no API-class inheritance; composition is explicit routers and Django `include`. |
| `mount Api::V1::Users` | `dmr.routing.path('users/', UsersController.as_view())` inside a `Router(urls=[...])`, or `Router.include(child_router)` | One mount = one router; the child's own prefix must reproduce the mounted segment. |
| `mount Api::V1::Users => '/v1'` | `Router(prefix='v1/')` for the mounted subtree, or a parent route `dmr.routing.path('v1/', include(child.to_urlpatterns()))` | The assembled URL must stay byte-identical to the frozen contract. |
| `mount({Api::V1::Users => '/path'}, with: { ... })` / per-mount `mounted do` configuration | No analogue | Per-mount configuration does not exist in DMR: port each mount as its own router + controller set, configured at the class level, and record the rest as drift. |
| `namespace :users do ... end` | `Router(prefix='users/')` (or `dmr.routing.path('users/', ...)`) | `resource` / `resources` / `group` / `segment` are aliases of the same segment — all four map identically. |
| `scope do ... end` | Nothing to emit | `scope` adds no URL segment; keep it only as a Python grouping (module or comments). |
| `:version` path segment | Router prefix segment | See §9. |
| `(:segment)` path captures, `/:id` | Django path converters via `dmr.routing.path('users/<int:pk>/', ...)` | Capture names are the DTO field names; converted types must match the Grape `type:`/`requirements:`. |
| `(.:format)` suffix | No analogue | Every client that sends `/users/1.json` is a drift item; see §9. |
| Non-DMR endpoints reachable from the same tree | `dmr.routing.external_path(route, view, openapi=...)` | Use for legacy/Django-admin handlers that must keep their URL, with `openapi=None` to hide them from the spec. |

## 2. Routes and request sources

| Ruby/Grape construct | DMR/Python construct | Parity note |
|---|---|---|
| `get '/users' do ... end` (+ `post`/`put`/`patch`/`delete`/`head`) | One controller method per verb on one `Controller[Serializer]` subclass | No route blocks: a method name *is* the route registration. |
| `route :any, '*path'` | Not available | DMR routes one method per verb; a greedy catch-all is a plain Django view behind `dmr.routing.external_path`, or a `<path:rest>` converter endpoint. |
| Grape `query` verb (QUERY parsed like POST, default 200) | A `query()` controller method with `'query'` added to `Controller.allowed_http_methods` and `@modify(status_code=HTTPStatus.OK)` | DMR registers custom method names through `allowed_http_methods`, but status inference only knows standard `HTTPMethod` members; set 200 explicitly to retain component parsing, validation, and OpenAPI registration. |
| `route_param :id, type: Integer, requirements: {...}` | `dmr.routing.path('users/<int:pk>/', ...)` + `Path[int]` parameter | `route_param` is namespace + `requires`; keep name/type/requirements identical. |
| `requirements: { id: /[0-9]+/ }` | Route converter or `re_path`-based view | Unanchored/odd regexes are drift unless reproduced with an explicit pattern. |
| `anchor: false` (prefix matching of a declared path) | Not available | Django path matching is anchored — drift per affected route. |
| GET gains HEAD automatically | Declare an explicit `head` method | A successful HEAD must be annotated to return `None`: DMR rejects response bodies for 2xx HEAD. |
| All routes gain OPTIONS → `204` + `Allow: OPTIONS, GET, PUT` | Controller `meta` method (mapped to OPTIONS); a ready-made default answers `204` + `Allow` | DMR builds `Allow` from the controller's allowed methods; byte-compare the header value with the Ruby service. |
| Unknown method on a known path → `405` + `Allow` | DMR 405 through `format_error` (`{"detail": [{"msg": "Method ... is not allowed, allowed: ..."}]}`) | Body shape differs; rebuild it with `dmr.routing.build_404_handler`-style handlers or an error handler when clients depend on `'405 Not Allowed'`. |
| One `params do ... end` block mixing path, query, body, and header params | **Split by source** into `Path[...]`, `Query[...]`, `Body[...]`, `Headers[...]`, `Cookies[...]` | Use the frozen contract's swagger `in:`/`param_type` plus the HTTP method; this split is where ports break, so verify it against the corpus, not the DSL alone. |
| Route-string params outrank query/body params; `route_param` outranks a declared same-named param | Independent components | DMR binds each component separately: if the same name arrives in two sources, the Ruby precedence rule is drift unless the DTOs encode it. |
| Undeclared params are ignored (only `declared(...)` filters them) | DTO fields define what is read | Ignore-by-default matches; do not turn on extra-field rejection unless the contract shows clients rely on `400`. |
| `?ids[]=1&ids[]=2` (Rack bracket form) | `Query[...]` DTO + `__dmr_force_list__` listing the **wire key including brackets** (`'ids[]'`) and a field aliased to it (`Field(alias='ids[]')` / `msgspec.field(name='ids[]')`) | Django preserves the brackets in the parameter key, so the alias, not the Python field name, is what must be listed. |
| `?ids=1,2` → `400` (`... is invalid`) | `__dmr_force_list__` yields `['1,2']` and a `list[str]` field accepts it | Comma form silently becomes a one-element list — either reject it in a validator or list it as drift. |
| Form-encoded bodies / list headers with a comma syntax | `__dmr_split_commas__` on the `Body[...]`/`Headers[...]` model | Django-native parsing only (form bodies come from `request.POST`, gated by `DMR_SETTINGS['django_treat_as_post']`, default `PUT`/`PATCH`). |

## 3. Params DSL to DTO constraints

One Grape `params` block becomes one DTO per source. Use the app-wide serializer (`MsgspecSerializer`); any DTO row
marked **inexpressible** needs `PydanticSerializer` (`dmr.plugins.pydantic.PydanticSerializer`) on that controller,
because cross-field rules and arbitrary callables cannot be expressed by msgspec metadata alone.

| Ruby/Grape construct | DMR/Python construct | Parity note |
|---|---|---|
| `requires :name, type: String` | Mandatory field (`name: str`) | Missing → `400` in both. |
| `optional :name, type: X` | `name: X \| None = None` | Optional + `None` skips the other validators in Grape; in a DTO the field still validates when present. |
| `type: Integer` / `Float` | `int` / `float` + lax coercion | See §4. |
| `type: BigDecimal`, `Numeric` | `decimal.Decimal` | Serialized as a JSON string (§13). |
| `type: Date` / `DateTime` / `Time` | `datetime.date` / `datetime.datetime` | Wire format in §13; AR/Rails time storage in [rails-to-django-map.md §5](rails-to-django-map.md#5-time). |
| `type: Boolean` | `bool` | `"true"`/`"false"`/`1`/`0` coerce in lax mode; verify unusual spellings against the corpus. |
| `type: String` | `str` | Only `String` keeps `""` as a value (§4). |
| `type: Symbol` | `enum.StrEnum` member or `str` | Symbols reach JSON as strings. |
| `type: File`, `Rack::Multipart::UploadedFile` | `FileBody` / `FileBodyLike` (+ `FileMetadata`) with `MultiPartParser` | Keep the multipart field name identical. |
| `type: JSON` | `dict[str, Any]` or a typed DTO | `coerce_with:` is not allowed with `JSON` in Grape; do not add coercion on the DMR side either. |
| `type: Hash` | `dict[str, Any]` or nested DTO | Nested DTO preferred; the contract's key set decides. |
| `type: Array[Integer]`, `Array[JSON]` | `list[int]`, `list[SomeDTO]` (+ `__dmr_force_list__` when it is a query param) | Array-of-hashes from a nested `requires ... do ... end` block → `list[NestedDTO]`. |
| `types: [Integer, String]` | `int \| str` union | Order in the union does not change the accepted set. |
| `oneof:` | Union/DTO shape + a model-level rule on a `PydanticSerializer` controller | Cross-param presence rule — **inexpressible** in msgspec metadata. |
| `values:` with an Array | `enum`/`Literal[...]` | Rejected values produce a different message; only the status and the accepted set have to match. |
| `values:` with a Range | `Annotated[int, Field(ge=..., le=...)]` (pydantic) / `msgspec.Meta(ge=..., le=...)` | Bounds are inclusive in both. |
| `values:` with a Proc or a hash of message per value | Custom validator returning the same message | **Inexpressible** as a declarative constraint. |
| `default:` eager value | DTO field default | Identical. |
| `default:` lambda (lazy) | `Field(default_factory=...)` / `msgspec.field(default_factory=...)`, or computed in the endpoint | Per-request defaults that depend on other params belong in the service. |
| `except_values:` | `PydanticSerializer` validator | **Inexpressible** declaratively. |
| `allow_blank:` | Validator interacting with the `""` rule | Only meaningful together with §4's blank semantics. |
| `regexp:` | `msgspec.Meta(pattern=...)` / pydantic `Field(pattern=...)` | Keep the pattern text identical; anchoring rules differ between engines. |
| `same_as:` another param | `PydanticSerializer` model validator | **Inexpressible** — cross-field. |
| `length: { is:, min:, max: }` | `msgspec.Meta(min_length=..., max_length=...)` / pydantic `Field(min_length=..., max_length=...)` | `is:` → set both bounds. |
| `mutually_exclusive`, `exactly_one_of`, `at_least_one_of`, `all_or_none_of` | `PydanticSerializer` model validator | **Inexpressible** — cross-field; confirm the exact semantics against the pinned gem. |
| `coerce_with:` (requires `type:`, forbidden for `JSON`) | `BeforeValidator` on the field | **Inexpressible** for msgspec; port the callable and keep its failure message if clients read it. |
| Custom type with `.parse` | Same as `coerce_with:` | **Inexpressible** declaratively. |
| `as:` | Field alias (`Field(alias=...)` / `msgspec.field(name=...)`) | Wire name is the alias; the Python name is free. |
| `desc:` / `description:` | `msgspec.Meta(description=...)` / `Field(description=...)`, or `@modify(summary=..., description=...)` for the endpoint | Documentation only; must still match the frozen contract's text. |
| `documentation:` passthrough keys | DMR component metadata + `dmr.openapi.OpenAPIConfig` | Keys with no DMR equivalent (`example`, `additional_properties`, custom) are drift. |
| `fail_fast` | No analogue | DMR reports all failures at once — drift when a client relies on first-error-only payloads. |
| `given`, nested `optional`/`requires` blocks | Nested DTO; `given` conditions → model validator | Conditional requirements are **inexpressible** declaratively. |
| Namespace-level `params do ... end` | Shared base DTO imported by every controller in the router | Do not duplicate the DTO per controller. |
| Reusable `params :name` + `use :name` | A shared DTO class imported (or mixed in) by controllers | Same effect, no registry. |
| `declared(params, ...)` | The DTO itself | DMR hands the endpoint a typed model; `declared`-style filtering (and its `before`-filter unavailability) has no analogue — see §8. |

## 4. Coercion and blank semantics

Grape coerces with dry-types: `"1"` → `1`, and `""` means *absence* for every type except `String`; `""` for an array
is `[]`; a comma string for an array is invalid; `nil` elements inside arrays are invalid; `optional` + `nil` skips
validators. A failed coercion is a `400` with the message `is invalid`.

| Ruby/Grape behavior | DMR/Python behavior | Decision per field |
|---|---|---|
| `"1"` → `1`, `"true"` → `true` | msgspec lax conversion and pydantic both coerce `"1"` → `1` and `"true"` → `True` for request validation | Parity by default; no extra work. |
| `""` → `nil` (all types except `String`) | Both engines raise instead (msgspec: ``Expected `int \| null`, got `str` ``; pydantic: ``Input should be a valid integer, unable to parse string as an integer``) | Use a Pydantic `BeforeValidator`, or normalize in a custom parser/component boundary before msgspec conversion; msgspec's `deserialize_hook` is not called for failed built-in conversions. Otherwise list the field as drift. |
| `""` → `[]` for arrays | Both engines raise (msgspec: ``Expected `array`, got `str` ``; pydantic: ``Input should be a valid list``) | Use the same pre-validation boundary to map `''` → `[]`, or list the field as drift. |
| Comma string for an array → `400` | In a `Body[...]`/`Headers[...]` model both engines reject it (`str` is not an array); a force-listed query key yields one element (`['1,2']`), and `__dmr_split_commas__` makes it split instead | Grape rejects the comma form everywhere. Pick reject or split per field and record it — neither default matches, and `__dmr_split_commas__` turns the rejection into a successful split. |
| `nil` elements inside arrays → invalid | `list[str]` (or any non-optional item type) rejects `null` items | Parity by default; loosening the item type (`str \| None`) is drift. |
| `optional` + `nil` skips validators | A DTO field still validates when present | Only matters for fields with extra constraints; port the constraint into the validator. |
| Failure message `is invalid`, status `400` | DMR request-validation → `400` with `{"detail": [{"msg": ...}]}` | Status matches; message text differs — clients that parse messages are drift. |
| Query values are always strings | Django exposes strings; DMR converts per the DTO type | Use `__dmr_cast_null__` where the contract sends the literal `'null'` for a nullable query field. |

## 5. Entities and response shaping

| Ruby/Grape construct | DMR/Python construct | Parity note |
|---|---|---|
| `present obj, with: Entity` | Endpoint return annotation → DTO; `Controller.to_response(raw_data, ...)` builds the body | Response validation is on by default (`DMR_SETTINGS['validate_responses']`), so the DTO is the contract. |
| `present :key, value` / a hash returned from the handler | Return a dict or a DTO with that exact key | No presenter layer: the return value *is* the body. |
| `body` assignment | `return` value of the endpoint | — |
| `Grape::PrecompiledJson` | `Controller.to_response(...)` with a dedicated renderer/serializer for already-encoded bytes | Never return raw `HttpResponse`: it bypasses negotiation and header/cookie management. Model ordinary JSON as data; use a renderer whose `render()` preserves prebuilt bytes only for endpoints whose frozen contract requires them. A plain `str` remains JSON-encoded (quoted), matching Grape's `body "string"`. |
| `expose :name` | DTO field | — |
| `expose :name, as: :wire` | Field alias | Wire name must match the contract. |
| `expose :assoc, using: Entity` | Nested DTO type | Relations as `allOf`-style nesting have no DMR analogue; flatten or nest per contract. |
| `expose :x, proc: ->(obj) { ... }` | Computed field: `@computed_field`-style property or a service-built DTO | Encoding logic must live in the mapping function, not in the endpoint body. |
| `expose :x, if:/unless:` | `Optional[X] = None` + explicit omission logic, or conditional response types | Omission is a behavior: see the `expose_nil` row. |
| `expose :x, safe: true` | No DMR equivalent | Entity-only option; keep the emitted value identical and record the option as having no DMR counterpart. |
| `expose :x, merge: true` | Explicit flattening in the DTO mapping | No automatic merge. |
| `expose :x, override:` / `unexpose :x` | Re-declare or drop the field on the subclass/inheriting DTO | Python DTO inheritance covers this. |
| `expose :x, default: ...` | Field default | Must match the entity's rendered default, not the DB default. |
| `format_with` + `with_options` | pydantic `field_serializer` | `PydanticSerializer` for that DTO. |
| `expose :x, documentation: {...}` | Component metadata / `dmr.openapi.OpenAPIConfig` | Only doc-level; keep the text. |
| **`expose_nil:` defaults to `true`** | Model nil attributes as present-but-`null` (`X \| None`), never as omitted | This is the single most common entity drift: an omitted key where the Ruby service emits `"key": null` breaks clients. |
| `root 'users', 'user'` | A wrapper DTO with the exact root key | The wrapper key is part of the contract. |
| `present_collection true, :key` (default key `items`) | A wrapper DTO with that exact key | No gem imposes an envelope; the key comes from the entity root or the handler hash. |
| `only:` / `except:` nested hash | Separate DTOs per projection | Share fields via inheritance, not runtime filtering. |
| Runtime options (`version`, `collection`, `env`, `attr_path`) | Endpoint/service inputs | These are serializer-time context switches; DMR has no entity runtime options. |
| `Grape::Entity::DSL` | Plain DTO class | No DSL form. |

## 6. Status codes, headers, redirects

| Ruby/Grape construct | DMR/Python construct | Parity note |
|---|---|---|
| POST defaults to `201` | `infer_status_code(HTTPMethod.POST)` → `201` | Already the DMR default; do not restate it. |
| DELETE with a blank body defaults to `204` | `@modify(status_code=HTTPStatus.NO_CONTENT)` + `return_type=None` | DMR rejects bodies for `204`/`205`/`304`; annotate the endpoint `None`. |
| Any other method defaults to `200` | `infer_status_code` → `200` | — |
| `status 202` | `@modify(status_code=HTTPStatus.ACCEPTED)` | Declare it as a response spec too. |
| `body false` → `204` | `return None` + `status_code=204` | Same as the DELETE row. |
| `return_no_content` | `status_code=204` + `None` | — |
| `header 'X-Foo', 'v'` | `@modify(headers={'X-Foo': NewHeader()})` and set it in the response | `NewHeader` for headers DMR adds, `HeaderSpec` for headers the framework/Django already sets. |
| `headers(...)` for several headers | `ResponseSpec(return_type, status_code=..., headers={...})` in `@modify(extra_responses=[...])` | Every non-default response must be declared or response validation will reject it. |
| `redirect url, permanent: true` | `raise RedirectTo(url, status_code=HTTPStatus.MOVED_PERMANENTLY)` | `Location` is set for you; annotate the endpoint with a `ResponseSpec(None, status_code=301, headers={'Location': HeaderSpec()})`. |
| `redirect url, permanent: false` (302) | `raise RedirectTo(url)` (default `HTTPStatus.FOUND`) | Also `303` via `status_code=HTTPStatus.SEE_OTHER`; `APIError` refuses 3xx, so use `RedirectTo`. |
| `stream` / `sendfile` | `dmr.streaming.StreamingController` / `SSEController` / `JsonLinesController`, or `FileRenderer` + `FileBody` for files | Streaming changes negotiation and validation semantics; port per endpoint, not wholesale. |

## 7. Errors

| Ruby/Grape construct | DMR/Python construct | Parity note |
|---|---|---|
| `error!(msg, status = nil, additional_headers = nil, ...)` | `raise APIError(raw_data, status_code=..., headers=..., cookies=...)` | `raw_data` is serialized by the controller's serializer — pass `self.format_error(...)` for the DMR-shaped body. |
| `error!` without a status → `default_error_status`, else `500` | `status_code` is required | Not `400`: keep the resolved status explicit per call site. |
| A hash passed to `error!` is rendered as-is | `raw_data` accepts a dict — including a Grape-shaped one | Best parity tool for legacy error payloads. |
| Validation failure → `400`, grouped payload with `full_messages` | Request validation → `400` with `{"detail": [{"msg": ..., "loc": [...]}]}` | Status matches; payload shape differs — a Grape-shaped `ErrorModel` on a shared base controller fixes it. |
| `404` body `["404 Not Found"]` + `X-Cascade: pass` | `dmr.routing.build_404_handler(...)` with a custom `format_error=` | Rebuild both the body and the cascade header when clients depend on them. |
| `405` message `405 Not Allowed` + `Allow`; rendered body depends on the configured formatter (`{"error":"405 Not Allowed"}` under Grape's JSON formatter) | DMR 405 body via `format_error` + its own `Allow` | Capture and reproduce the formatter-specific body; the `Allow` set is built from the controller's methods. |
| Failsafe `500` for unhandled exceptions | `dmr.routing.build_500_handler(...)` | Both answer 500; the body shape must be rebuilt for parity. |
| Uncaught `RecordNotFound` becomes a generic `500` | Django exceptions propagate to the global error handler | Do **not** "improve" it to 404 unless the corpus proves the Ruby service returned 404. |
| `rescue_from :all` / `:grape_exceptions` / classes / `with:` / `rescue_subclasses:` / `backtrace:` | `@modify(error_handler=...)` → `Controller.handle_error` / `handle_async_error` → `DMR_SETTINGS['global_error_handler']` (`dmr.errors.global_error_handler`) | Three-level chain: endpoint, controller, global (`dmr.errors.SyncErrorHandler` / `AsyncErrorHandler`). |
| Handler scope-local, first-registered wins, must sit above its routes, one re-raise redispatch | One `SyncErrorHandler`/`AsyncErrorHandler` per controller, plus the global handler; no re-dispatch loop | Exception→(status, payload) mapping is a plain Python lookup, not route-scoped registration. |

**Parity recipe.** Create one module (`app/api/errors.py`) holding: (a) the exception → `(status, payload)` table ported
from every `rescue_from`/`error!` site, (b) the Grape-shaped error model, and (c) the 404/405/500 handlers. Attach it
through a shared base controller that overrides `format_error` and pins the Grape-shaped `ErrorModel` as its error
model, so no endpoint builds error responses inline. Unit-test the table directly.

## 8. Filters, helpers, middleware, auth

| Ruby/Grape construct | DMR/Python construct | Parity note |
|---|---|---|
| `before` / `before_validation` / `after_validation` / `after` / `finally`, inherited down and across mounts | Django `MIDDLEWARE` wrapped with `dmr.decorators.wrap_middleware` (keeps OpenAPI + validation), or an explicit call in a shared base controller | Django middleware is not per-route: state which filter behaviors survive, and port order-dependent ones into the base controller. |
| `error!` in a callback aborts the request | Raising an auth/API exception aborts the request | Same control flow; `APIError.raw_data` carries the payload. |
| `after` is skipped on validation failure; `finally` always runs | No `finally` hook | Model as middleware `try/finally` or an explicit call, and record the difference. |
| `405` runs only `before`; auto-OPTIONS runs `before` + `after` | DMR dispatches 405/OPTIONS before/outside endpoint methods | Per-filter parity must be proven in the differential run. |
| `helpers do ... end` | Plain Python functions/modules called from the controller | Helpers are not part of the transport layer; port them as services. |
| `use`, `insert_before`, `insert_after` | `MIDDLEWARE` list order (insert relative to Django entries) | Never reorder Django's own middleware to mimic Grape. |
| `Grape::Middleware::Auth::Base`, `auth :label`, `http_basic` | `SyncAuth` / `AsyncAuth` / `SyncOrAsyncAuth` subclasses (e.g. `dmr.security.http.HttpBasicSyncAuth`) with `security_schemes` + `security_requirement`, wired via the `auth` list or `DMR_SETTINGS['auth']` | The OpenAPI security scheme must match the legacy declaration. |
| `declared(params)` inside a helper/filter | Not available: DMR parses components for the endpoint, not for middleware | Filter-time param inspection is drift; move the check into the endpoint or a validator. |

## 9. Versioning and the format suffix

| Ruby/Grape construct | DMR/Python construct | Parity note |
|---|---|---|
| `version 'v1', using: :path` | The router prefix (`Router(prefix='v1/')`, or the mount path segment) | Exact target: the assembled URL is unchanged as long as the prefix reproduces `:version`'s position. |
| `version ..., using: :header, vendor: 'acme'` (Accept `application/vnd.acme-v1+json`) | No analogue | Implement a small resolver middleware (plain Django) writing the version into `request`/`path_info` **only** where the frozen contract shows clients use it; otherwise drift. |
| `version ..., using: :accept_version_header` | No analogue | Same resolver-middleware treatment. |
| `version ..., using: :param, parameter: 'apiver'` | No analogue | Same; a query param cannot select a DMR router. |
| `cascade: true` emits `X-Cascade: pass` | No analogue | Drift: DMR has no cascade chain. |
| `strict: true` → `406` for a wrong/unsupported version | Can be reproduced by the resolver middleware | Requires explicit code; not a DMR feature. |
| `(.:format)` suffix, e.g. `/users/1.json` | No analogue | Add an explicit optional suffix segment (a `re_path`-based wrapper view that strips it and delegates) **only** where the contract proves clients use it; otherwise drift with the concrete paths listed. |
| Dots excluded from captures | Django converters exclude nothing by default | An optional suffix wrapper must strip the suffix before capture matching. |

## 10. Negotiation

| Ruby/Grape construct | DMR/Python construct | Parity note |
|---|---|---|
| `format :json` | `DMR_SETTINGS['renderers'] = [JsonRenderer()]` (or per-controller `@modify(renderers=[...])`) | The list's first element is the default. |
| `default_format :txt` | Put the default renderer/parser first in the list | DMR's default is the first entry, not a named format. |
| `content_type :json, 'application/vnd.acme+json'` | Extra `JsonRenderer(content_type=...)` / `JsonParser(content_type=...)` instances | Declaring extra content types must not drop the original ones (Grape replaces the whole set). |
| `formatter :json, ->(obj, env) { ... }` | A custom renderer, or a serializer hook | `error_formatter` is the same problem for error bodies. |
| `parser :json, ->(body) { ... }` | `dmr.parsers.JsonParser` / `MultiPartParser` / `FormUrlEncodedParser`, or a custom parser | Per-endpoint via `@modify(parsers=[...])`. |
| Format resolution `extension → ?format= → format option → Accept → default_format → txt` | Renderer/parser selection driven by `Content-Type` (request) and `Accept` (response); `dmr.negotiation.RequestNegotiator` / `ResponseNegotiator` | DMR has no format resolution order — `?format=` and extensions are drift unless reproduced in middleware. |
| `format :json`: unknown extension → `404` | No analogue (DMR routes have no extension) | Drift, with the affected paths listed. |
| `format :json`: `?format=xml` → `406` | No analogue | `?format=` is an ignored query param in DMR unless a validator rejects it — choose deliberately and record it. |
| `format :json`: `Accept: application/xml` still returns JSON | DMR answers `406` for an unsatisfiable `Accept` (the 406 response is part of every controller's response spec) | Drift where clients send a non-JSON `Accept` and still expect JSON. |
| Unsupported request body `Content-Type` → `415` | DMR answers `400` for an unsupported request `Content-Type` | Status difference; drift-listed when strict parity is chosen. |
| Absent/unsupported `Content-Type` → `406` unless `default_format` | DMR falls back to the default (first) parser and parses | Different: a `406` becomes a success. Drift-listed per endpoint. |
| Body `Content-Type: application/x-www-form-urlencoded` / multipart | `FormUrlEncodedParser` / `MultiPartParser`; Django-native parsing is enabled for the methods in `DMR_SETTINGS['django_treat_as_post']` (`PUT`, `PATCH` by default) | Keep the declared media types per endpoint. |
| `formatter` / `error_formatter` are applied by Grape itself when it renders a response (Grape README §Format) | `DMR_SETTINGS['validate_negotiation']` (defaults to `validate_responses`) asserts the returned `Content-Type` matches the negotiated renderer | Enable it outside production so a renderer mismatch surfaces as a test failure instead of a byte diff in production. |

## 11. Pagination

No pagination gem imposes a response envelope: the root key comes from the grape-entity `root`/`present_collection`
or the handler's hash. Port the gem's *params, defaults, clamping, and headers*, plus the service's exact envelope.

| Ruby/Grape construct | DMR/Python construct | Parity note |
|---|---|---|
| kaminari `param_name :page`, `default_per_page 25`, pages start at 1 | `page: int = 1` / `per_page: int = 25` inside `Query[...]` | Same names and defaults. |
| grape-kaminari `page` (1) / `per_page` (10) / `offset` (0) | The same three fields with the same defaults | Under `Query[...]`, so they are validated like any other param. |
| api-pagination `page` / `per_page` (`per_page: 10` default in the Grape `desc`) | Same, declared in the Query DTO and documented via `desc:`/`description:` | — |
| pagy offset keys `page` / `limit` (default `limit: 20`) | `page` / `limit` fields with the same defaults | Keep the offset semantics (offset, not page) in the service. |
| will_paginate `page` / `per_page` | Same field names | — |
| `page(0) == page(1)` clamping, `max_per_page` | Reimplement in a validator before slicing | Confirm the clamping rules against the pinned gem version; the DTO is where the `400`/silent-clamp decision lives. |
| Slicing via the gem's model extension | `django.core.paginator.Paginator` in the service layer | Only the queryset slicing is ported; the envelope stays custom. |
| `X-Total`, `X-Total-Pages`, `X-Page`, `X-Per-Page`, `X-Next-Page`, `X-Prev-Page`, `X-Offset` (grape-kaminari) | Set them on the response: `Controller.to_response(..., headers={...})` or an `@modify(headers={'X-Total': NewHeader(), ...})` declaration | Every header in the frozen contract must be emitted with the same value. |
| api-pagination RFC 8288 `Link` (first/last/next/prev) + `Total` + `Per-Page` | Build the `Link` value in the service and declare/set it via `NewHeader`/`ResponseSpec(headers=...)` | `Link` is one header with a comma-joined value list. |
| The envelope (`{"users": [...], "meta": {...}}`) | A custom DTO returned by the endpoint | `dmr.pagination.Page` / `Paginated` only when the contract's envelope happens to match (`count`, `num_pages`, `per_page`, `page.number`, `page.object_list`) — usually it does not. |

## 12. Throttling and CORS

| Ruby/Grape construct | DMR/Python construct | Parity note |
|---|---|---|
| `throttle(name, limit:, period:, &block)` | `SyncThrottle(max_requests, Rate.second/minute/hour/day, cache_key=..., response_headers=...)` / `AsyncThrottle`, attached per controller (`throttling=[...]`) or globally via `DMR_SETTINGS['throttling']` | One DMR throttle per legacy rule; same limit and window. |
| Cache key from `Rack::Attack` (`req.ip`, `req.env['warden'].user.pk`, a custom block) | `dmr.throttling.cache_keys.RemoteAddr` / `UserPk` / `JwtToken` | A fully custom block that is not IP/user/JWT is drift. |
| `Rack::Attack.cache.store` (defaults to the Rails cache) | The throttle backend defaults to Django's cache framework (`CACHES`); a Redis-backed throttle backend is available in DMR | Configure a shared cache: a per-process cache makes the limit meaningless behind multiple workers. |
| Precedence: safelist → blocklist → throttle → track | No blocklist/safelist stage | Blocklists and tracks have no DMR analogue — drift for every blocklist rule. |
| Blocklist default response `403` | No analogue (throttles answer `429`) | Drift: a `403` becomes a `429` unless a custom error handler or a middleware reproduces it. |
| Throttle default response `429` | A throttled request answers `429` | Parity on the status. |
| `throttled_responder` custom body | An `error_handler` / shared base controller building the same body | Rebuild the payload; DMR's is `{"detail": [...]}`. |
| `Retry-After` opt-in via `throttled_response_retry_after_header = true` | Emit it through the throttle's response-header providers | Only when the legacy service opted in. |
| **No rate-limit headers by default** | DMR's throttling header providers emit `X-RateLimit-Limit`/`-Remaining`/`-Reset`, and `RateLimitIETFDraft` emits `RateLimit` and `RateLimit-Policy` | New response headers are drift: keep only the providers whose headers the contract already had, and list the additions explicitly. |
| `allow { origins ...; resource '*', headers:, methods:, expose:, credentials:, max_age:, if:, vary: }` | `django-cors-headers` settings: `CORS_ALLOWED_ORIGINS`, `CORS_ALLOW_HEADERS`, `CORS_ALLOW_METHODS`, `CORS_EXPOSE_HEADERS`, `CORS_ALLOW_CREDENTIALS`, `CORS_PREFLIGHT_MAX_AGE` | Translate each key literally; `if:`/`vary:` conditional behavior is drift unless reproduced in middleware. |
| `methods` defaults to `[:get]`; `:any` = get/head/post/put/patch/delete/options | `CORS_ALLOW_METHODS` with the exact expanded list | Do not leave the Django default. |
| `max_age:` default `7200` | `CORS_PREFLIGHT_MAX_AGE = 7200` | Same default; still set it explicitly. |
| `credentials: true` with `origins '*'` raises | `CORS_ALLOW_CREDENTIALS = True` with `CORS_ALLOWED_ORIGINS = ['*']` is equally invalid | Preserve the legacy combination exactly; never "fix" it by widening origins. |
| Emitted `access-control-allow-origin/methods/expose-headers/max-age/allow-credentials`, preflight `access-control-allow-headers`, `Vary: Origin` | django-cors-headers emits the same set from the settings above | Byte-compare preflight responses in the corpus. |

## 13. Wire format

ActiveSupport JSON is not the DMR default. Every row below is either reproduced deliberately or listed as drift; and
the harness must compare **raw bytes**, never parsed JSON (see [differential-parity.md §3](differential-parity.md#3-contract-diff-script)).

| Ruby/ActiveSupport behavior | DMR/Python behavior | Decision |
|---|---|---|
| `time_precision = 3` → `2026-09-18T12:00:00.000Z` (ms always 3 digits, offset preserved) | msgspec emits `2026-09-18T12:00:00Z` when ms are zero and `...00.123456Z` (6 digits) otherwise | Reproduce with a serializer hook/encoder that always writes exactly three fractional digits; otherwise it is a drift line per timestamp field. |
| `Date#as_json` → `%Y-%m-%d` | `datetime.date` → `"2026-09-18"` | Parity by default. |
| `BigDecimal#as_json` → string | `decimal.Decimal('1.50')` → `"1.50"` | Parity; never convert Decimals to float. |
| Non-finite `Float` → `null` | msgspec encodes `nan`/`inf` as `null` | Parity. |
| `Symbol#as_json` → name | `enum.StrEnum` member / `str` | Parity. |
| Hash keys → strings | JSON object keys are always strings | Parity. |
| HTML escaping on: `<`, `>`, `&` → `\u003c`, `\u003e`, `\u0026`, plus U+2028/2029 | msgspec and stdlib `json` do **not** escape them | Either add the escaping (`MsgspecSerializer.serialize_hook`, or a `JsonRenderer(json_module=...)` whose `dumps` escapes), or drift-list every affected string field. |
| Grape `:json` formatter uses the object's `to_json` when defined, else `MultiJson.dump` | The controller's serializer/renderer pair always encodes | Objects with a custom `to_json` need an explicit `serialize_hook` branch. |
| A `String` body double-encodes unless it is `Grape::PrecompiledJson` | `return 'text'` with `JsonRenderer` → `"text"` (quoted) | Parity for ordinary strings; where the legacy code returned precompiled JSON, apply the dedicated-renderer `Grape::PrecompiledJson` row above while still returning through `to_response`. |
| Timezone storage (AR stores UTC) | `USE_TZ = True` | See [rails-to-django-map.md §5](rails-to-django-map.md#5-time). |

**Serializer choice.** Keep the app-wide `MsgspecSerializer` and reproduce wire details with a serializer subclass
(`serialize_hook` for values, plus field-level knobs); move a DTO to `PydanticSerializer` only for
`field_serializer`-style per-field control. Renderer-level escaping is a `JsonRenderer` tweak, not a per-endpoint hack.

## Approved drift candidates

Structurally inexpressible in DMR; each must appear as an explicit `approved drift` line in `port/parity-report.md`
with the affected endpoints and request/response shapes — never dropped silently:

1. **Per-mount Grape configuration** (`mount(..., with: ...)`, mount-level lifecycle hooks): no DMR analogue; each
   mount becomes its own router/controller set.
2. **Non-path versioning** (`using: :header` (+ `vendor:`), `:accept_version_header`, `:param`): resolver middleware
   or drift. `cascade: true` and its `X-Cascade: pass` header have no analogue at all.
3. **The `(.:format)` route suffix**: no DMR route shape; only an explicit optional-suffix wrapper where the contract
   proves clients use it, otherwise drift with the concrete paths listed.
4. **Cross-field params rules** (`oneof`, `mutually_exclusive`, `exactly_one_of`, `at_least_one_of`,
   `all_or_none_of`, `same_as`, `given`): expressible only as model validators, so their failure payloads differ; drift
   wherever the client contract writes those payloads down.
5. **Arbitrary coercers** (`coerce_with:`, custom `.parse` types): ported as code, so a Ruby error message or an
   unusual accepted input cannot be reproduced byte-for-byte; drift per field.
6. **Negotiation status differences**: request `Content-Type` handling (`415` → `400`), absent/unsupported
   `Content-Type` (`406` → default parser), `Accept: application/xml` (`200` JSON → `406`), `?format=xml` (`406` → an
   ignored param), unknown extensions (`404` → no such route).
7. **New rate-limit response headers**: DMR throttling can emit `X-RateLimit-*`, `RateLimit`/`RateLimit-Policy`, and
   `Retry-After`; the legacy service emitted none by default, so every added header is drift.
8. **rack-attack blocklists/safelists/tracks and their `403`**: no DMR stage exists; a blocklisted request becomes a
   `429` (or needs custom code).
9. **`fail_fast`**: DMR aggregates validation failures.
10. **`declared(params)` outside the endpoint** (filters/helpers): DMR gives parsed components to the endpoint only.
11. **Wire-format details not reproduced** (ms precision, HTML escaping, U+2028/2029, custom `to_json` objects):
    drift per field wherever the harness finds a byte diff.
12. **Entity runtime options** (`version`, `collection`, `env`, `attr_path`) and `only:`/`except:` projectors:
    one DTO per projection instead.
