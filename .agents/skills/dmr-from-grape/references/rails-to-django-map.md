# Rails to Django mapping

Use this map when the Ruby runtime and domain layer (models, migrations, callbacks, jobs, mailers, storage, auth gems,
test suite) is re-implemented in Python. Rails facts are cited from the Rails guides / gem READMEs; every DMR construct
named here is the one listed in [`grape-to-dmr-map.md`](grape-to-dmr-map.md). Nothing in this map is preserved by
translation — each row is a re-implementation with a stated parity cost.

## 1. Schema source

| Ruby/Rails construct | Django/DMR construct | Parity note |
|---|---|---|
| `config.active_record.schema_format :ruby` (default) → `db/schema.rb` (Rails guides, Active Record Migrations) | authoritative list of table/column names, types, defaults, nullability, indexes | read it as data, never as code: the schema is the contract the shared-database port must reproduce |
| `config.active_record.schema_format :sql` → `db/structure.sql` (Rails guides, Active Record Migrations) | same authority, plus index and default expression detail | prefer it when present; `db/schema.rb` can hide expression defaults |
| `create_table "books"` | `class Book(models.Model)` with `class Meta: db_table = "books"` | in shared-schema mode `db_table` and every `db_column` stay literal — both services read one database until cutover |
| `t.timestamps` (`null: false`, precision 6) (Rails guides, Active Record Migrations) | `created_at = DateTimeField(auto_now_add=True)` / `updated_at = DateTimeField(auto_now=True)` | `null=False` is Django's default; Django stores microsecond precision without a precision option |
| `t.datetime` / `t.date` / `t.decimal` / `t.json` / `t.string` | `DateTimeField` / `DateField` / `DecimalField` / `JSONField` / `CharField(max_length=…)` | copy `limit`, `precision`, `scale`, `default`, `null` from the schema file, not from intuition |
| `add_index` / `add_reference` | `Meta.indexes`, `db_index=True`, `ForeignKey(db_column="…_id")` | index names are included in the diff even when they do not affect query results |

**Warning — `auto_now`.** AR writes `updated_at` on every save; Django's `auto_now=True` does *not* fire for
`QuerySet.update()`, `bulk_update`, or raw SQL. Every code path ported from a Ruby write must either go through
`save()` or set the timestamp explicitly, otherwise rows silently keep a stale `updated_at`.

Bootstrap recipe (run once per app, keep the script under `port/`):

- `python manage.py inspectdb --database default <table>` generates Django field declarations straight from the shared
  database. Use it to cross-check every column the port declared from `db/schema.rb` — a name or type that differs
  between the two is a port bug, not a preference.
- Column diff: a throwaway script that prints, per table,
  `connection.introspection.get_table_description(cursor, table)` (name, type, `null_ok`, default) plus
  `connection.introspection.get_constraints(cursor, table)` (indexes, unique, primary key), and diffs that against the
  `create_table` / `add_index` blocks of `db/schema.rb`.

## 2. Models

| Ruby/Rails construct | Django/DMR construct | Parity note |
|---|---|---|
| `belongs_to :author` (Rails guides, Active Record Associations) | `author = ForeignKey(Author, on_delete=…, db_column="author_id")` | preserve `db_column`; the association is an explicit JOIN in both stacks |
| `belongs_to :author, optional: true` | `ForeignKey(…, null=True)` | the Ruby `optional` flag only removes the presence validation; read the schema for the real nullability |
| `has_many :books` | reverse accessor (`book_set` or `related_name`) | naming is visible in DTO output only if the Ruby code serialised the association; otherwise keep it internal |
| `has_many :through` — the join table needs an `id` (Rails guides, Active Record Associations) | `ManyToManyField(through=…)` plus an explicit through model | use the explicit `through` model whenever the join row carries columns or its own `id`, so the Django mapping matches the existing table |
| polymorphic association (`*_type` + `*_id`) | `GenericForeignKey` **or** an explicit FK pair | choose once and record it: `GenericForeignKey` cannot be joined or indexed like a real FK |
| `scope :published, -> { … }` | custom `Manager` / `QuerySet` method | keep the scope's SQL semantics, including default ordering |
| `validates :title, presence: true, uniqueness: true, length: { maximum: 200 }` | DTO field/model constraints (`msgspec.Meta`, Pydantic `Field`/validators) for request input; `UniqueConstraint` / `CheckConstraint` in `Meta.constraints` for storage | `dmr.validate` validates responses, not input; request validation and integrity constraints are different layers, and Django must preserve both |
| counter cache `books_count`, declared on `belongs_to` (Rails guides, Active Record Associations) | `F("books_count") + 1` / `- 1` updates in the owning service + a backfill migration | AR maintained it automatically; Django drifts unless every create/destroy path updates it |
| AR `enum` array form (ordinals from 0) or string-backed column (Rails guides, Active Record Query Interface) | `IntegerChoices` / `TextChoices` matching **the original migration's column** | read the stored value from the migration, not from the symbol names; Rails 7.2 deprecated the keyword form, so do not trust the model file alone |
| STI: `type` column + `type_condition` (Rails guides, Single Table Inheritance) | one model with explicit dispatch, or multi-table inheritance | STI filters every query by `type` implicitly; Django does not — state the choice and the changed query semantics, and audit every query that used to be implicitly scoped |

`Cannot be carried over:` implicit STI scoping (`type_condition` on every read) — replaced by an explicit dispatch
helper or a manager filter that every call site must use; association extensions and `has_many … do … end` blocks;
dynamic finders (`find_by_*` generated by `method_missing`) — each becomes a named manager method; the `optional: true`
validation semantics on a non-null column (Django has no equivalent of "validation absent, DB required").

## 3. Migrations

Port the **resulting schema**, not the migration files: existing rows were shaped by the Ruby migrations, and the
shared database is already at that shape.

| Ruby/Rails construct | Django/DMR construct | Parity note |
|---|---|---|
| `db/migrate/*.rb` (Rails guides, Active Record Migrations) | not ported — models + one initial migration are derived from the schema snapshot | replaying the Ruby history into Django produces a schema keyed to Django's own idioms |
| `ActiveRecord::Migration` + `schema_migrations` versioning | Django's migrations app + `django_migrations` | the two version tables coexist in shared-schema mode and must not be conflated |
| `add_column` / `change_column` / `remove_column` history | collapsed into the final model definitions | intermediate states are not reproduced |
| `add_foreign_key` / `add_index` / `add_reference` | `ForeignKey`, `Meta.indexes`, `db_index=True` | port the *names* too, when the index name is referenced in the schema file |

Verification recipes (all mandatory per slice):

- `python manage.py makemigrations --check --dry-run` exits `0` — models and migrations agree and nothing is unrecorded.
- `python manage.py sqlmigrate <app> 0001` against the `create_table` blocks of `db/schema.rb`: same column names,
  same nullability, same defaults, same indexes.
- `python manage.py inspectdb` against the ported models: no column exists on one side only.
- Data migrations are not ported: if the Ruby side backfilled a column, run the ported backfill command once against
  the shared database and record it in `port/parity-report.md`.

## 4. Callbacks and domain logic

| Ruby/Rails construct | Django/DMR construct | Parity note |
|---|---|---|
| `before_save` / `before_create` / `before_update` | the first statements of the service function that owns the write | the hook fired wherever `save()` was called; a service makes the order visible at the call site |
| `after_save` / `after_create` | statements after `.save()` in that same service | |
| `after_commit(on: :create)` | `transaction.on_commit(...)` registered inside the surrounding `transaction.atomic()` block | after-commit side effects (mail, jobs, webhooks) must not run before the row is durable; Django's `TestCase.captureOnCommitCallbacks` makes this testable |
| `validate` / `before_validation` | DTO field/model validator for request-shaped input, explicit service check otherwise | the failure must still produce the Grape-shaped 400 payload ([`grape-to-dmr-map.md` §7](grape-to-dmr-map.md#7-errors)); `dmr.validate` is a response-spec decorator and is not used here |
| `dependent: :destroy` | `ForeignKey(on_delete=CASCADE)` or an explicit service deletion | `CASCADE` runs in the database: Ruby callbacks that ran on each dependent destroy do not |
| `dependent: :nullify` | `on_delete=SET_NULL` + `null=True` | |
| `touch: true` | explicit timestamp update in the service | see the `auto_now` warning in §1 |
| concern `included do … end` DSL | a module of plain functions called explicitly | no inclusion-time code, no `included`/`class_methods` hook |
| `method_missing` / `define_method` / monkey patches / `alias_method` chains | explicit named functions, one per dynamic behaviour | enumerate every occurrence during inventory; each is a rewrite with its own test |

Business logic lives in services that controllers call — never in the DTO/DMR layer, and never in a model hook.
**NEVER use Django signals (`pre_save` / `post_save` / `m2m_changed`) as a stand-in for AR callbacks:** they reproduce
the implicit-firing problem the port is trying to remove and put logic outside the service that owns the transaction.

`Cannot be carried over:` implicit callback firing (a callback ran on every `save()` from anywhere); the callback
ordering rules the Ruby runtime enforced; `concern` inclusion-time behaviour; gem monkey patches — the patched
behaviour must be re-implemented explicitly or dropped with a drift line.

## 5. Time

| Ruby/Rails construct | Django/DMR construct | Parity note |
|---|---|---|
| AR persists timestamps in UTC (Rails guides, Active Record Migrations) | `USE_TZ = True`; aware datetimes, stored UTC | leave `USE_TZ` on; a naive-datetime port reintroduces drift and breaks the wire comparison |
| `Time.parse` (lax) | `datetime.fromisoformat` (strict ISO-8601) | laxer Ruby input has no equivalent: either restrict the accepted grammar and drift-list the affected endpoints, or add an explicit parser for the exact formats the corpus shows |
| `Time` / `DateTime` / `Date` classes | `datetime.datetime` (aware) / `datetime.date` | `Date` values reach the wire as `%Y-%m-%d` |
| Ruby-side formatting of times for the API | serializer/encoder concern — see [`grape-to-dmr-map.md` §13](grape-to-dmr-map.md#13-wire-format) | never format timestamps in services or controllers; the byte-level format is decided once, in the serializer |

`Cannot be carried over:` `Time.parse` laxness (Ruby ignored trailing junk and accepted many non-ISO strings);
per-request `Time.zone` switching; AR's implicit `updated_at` writes (see §1).

## 6. Background jobs

| Ruby/Rails construct | Django/DMR construct | Parity note |
|---|---|---|
| ActiveJob `queue_adapter = :async` (default) (Rails guides, Active Job) | do **not** reproduce it as in-process work — use the project's real queue | the `:async` adapter runs the job in-process and loses it on restart; reproducing that would multiply data loss |
| sidekiq / resque / delayed_job / solid_queue adapters (Rails guides, Active Job) | Celery / django-tasks / RQ, chosen per project and recorded | one queue only; the choice goes into `port/parity-report.md` |
| `class SomeJob < ApplicationJob; def perform(arg) … end` | a task function taking a typed, JSON-serialisable payload | pass ids and scalars, never model instances; the payload type is the job's new contract |
| `perform_later` | the queue's enqueue call | the ordering between commit and execution is adapter-dependent in Ruby — make it explicit with `transaction.on_commit` whenever the job reads the row it was enqueued for |
| `retry_on` / `rescue_from` | the queue's retry policy plus explicit `try/except` in the task | port the same exception classes and retry counts |
| `discard_on` | drop without retry for the same exception types | a job that Ruby discarded must not become a poison message in Python |
| idempotency | an explicit guard in the task (idempotency key or conditional update) | queue adapters differ; do not rely on the adapter for exactly-once |

`Cannot be carried over:` `:async` in-process semantics; the job argument marshalling the Ruby adapter performed;
whichever delivery guarantee the Ruby adapter happened to provide.

## 7. Mail

| Ruby/Rails construct | Django/DMR construct | Parity note |
|---|---|---|
| ActionMailer delivery `:smtp` (default) (Rails guides, Action Mailer) | `EMAIL_BACKEND` = SMTP backend | keep the same host/port/credentials behaviour and the same sender addresses |
| delivery `:test`, collecting into `ActionMailer::Base.deliveries` (Rails guides, Action Mailer) | `locmem` backend + `mail.outbox` in test settings | the ported tests must assert on the same messages, including subject and recipients |
| `deliver_now` | `send_mail(...)` / `EmailMultiAlternatives(...).send()` | |
| `deliver_later` | enqueue the mail as a task (§6) | delivery timing changes: state it and make sure the queue runs in every environment that previously delivered |
| mailer action + ERB views | Django templates rendered in a mail service, text and HTML alternatives | keep the layout structure and both parts; only HTML mail breaks text-only clients |
| mailer `default from:` / subject strings | settings + literal subjects | some clients and tests assert the exact subject and sender — do not rewrite them |

`Cannot be carried over:` ActiveJob-mediated delivery timing; ActionMailer's implicit view lookup (Django needs an
explicit template per message part).

## 8. Storage

| Ruby/Rails construct | Django/DMR construct | Parity note |
|---|---|---|
| `config.active_storage.service` selecting a service (Rails guides, Active Storage) | the `STORAGES["default"]` backend plus `FileField` / `ImageField` | one storage backend for the app; attachments referenced by URL must resolve to the same object |
| `Disk` service | `FileSystemStorage` with the same root and the same key layout | the *key* (path) is the contract when the API returns attachment URLs |
| `S3` service | the `storages` S3 backend with the same bucket and prefix | keep region/CDN host so returned URLs keep their shape |
| `GCS` service | the `storages` Google Cloud backend with the same bucket and prefix | same |
| `Mirror` service | no single Django setting: dual-write in the storage service, or record the drift | a mirror is two writes with a read-preference rule; Django must be told explicitly |
| attachment URLs returned by the API | backend `url()` result compared in the differential harness | a changed host, prefix, or signing style is a contract break, not a cosmetic one |

`Cannot be carried over:` the `Mirror` service's read-preference behaviour; any Ruby-side filename sanitisation and
key generation that the URL contract depends on unless it is re-implemented identically.

## 9. Auth and credentials

**This whole section is a security surface: it must never silently change.** Port the header names, rotation
behaviour, TTLs, token storage/verification, and password verification exactly, then verify them against real
credentials from the shared database.

devise_token_auth (devise_token_auth README):

| Ruby/Rails construct | Django/DMR construct | Parity note |
|---|---|---|
| headers `access-token`, `client`, `expiry`, `uid`, `token-type` | a `dmr.security.SyncAuth`/`AsyncAuth` subclass reading the same header names | clients send these verbatim; renaming them is a breaking change |
| Base64-JSON `Authorization: Bearer` form | the same auth class accepting the bearer form | both forms are in the wild and must be accepted |
| rotation on every request (`change_headers_on_each_request = true` by default) | rotated credentials issued by the compatibility auth layer and returned on the response | dropping rotation silently lengthens every session — list it as a security drift if you do |
| 5 s batch window returning blank `access-token` / `expiry` | reproduce the window and the blank values | clients that cache headers behave differently if the blanks disappear |
| `tokens` JSON column on the user row | a compatibility auth implementation reading and updating that column in shared-schema mode | DMR's stock token auth uses a different token model and hash scheme; it cannot validate in-flight devise_token_auth tokens without an adapter |
| bcrypt cost 12 (1 in test), optional pepper | a raw-Devise-bcrypt compatibility verifier/writer (see below) | stock Django password-hasher selection does not recognize a bare `$2a$…` digest |
| `expiry` header | TTL on the issued token | never lengthen it silently |

doorkeeper (doorkeeper README):

| Ruby/Rails construct | Django/DMR construct | Parity note |
|---|---|---|
| opaque `Authorization: Bearer <token>` access token | a `SyncAuth`/`AsyncAuth` compatibility class backed by the existing `oauth_access_tokens` rows, or a dedicated OAuth provider preserving the same store | plain Doorkeeper tokens are not JWTs; `HeaderJWTSyncAuth` rejects them |
| `expires_in` default 7200 | the same TTL unless the app configured otherwise — read the initializer | an assumption about TTL is a security drift |
| refresh tokens | a compatibility refresh endpoint operating on the existing Doorkeeper token rows | refresh must keep working for tokens issued before cutover, or the cutover invalidates every session |
| revocation and introspection | compatibility endpoints backed by the durable legacy token store | revocation must survive a process restart and introspection response fields must remain stable |
| token / refresh / introspect response bodies | existing auth app or custom controllers matching the frozen contract | DMR's stock token controllers are usable only when their request, response, token, and revocation formats are proven compatible |

doorkeeper-jwt (doorkeeper-jwt README):

| Ruby/Rails construct | Django/DMR construct | Parity note |
|---|---|---|
| JWT `Authorization: Bearer <token>` | `dmr.security.jwt.auth.header.HeaderJWTSyncAuth` | use JWT auth only after the pinned Gemfile confirms `doorkeeper-jwt` produced the access tokens |
| claims `iss`, `iat`, `aud`, `jti`, `sub`, `exp` and `signing_method` | `HeaderJWTSyncAuth` with the same issuer, audience, claim names, key, and algorithm | verify a real unexpired legacy token; other services see the claims and signature, not the framework |
| refresh, revocation, and introspection | compatibility controllers first; DMR JWT controllers only after a format-level parity proof | accepting an access JWT does not prove that legacy refresh tokens, revocation rows, or endpoint payloads are compatible |

Password hashes (devise README):

| Ruby/Rails construct | Django/DMR construct | Parity note |
|---|---|---|
| Devise stores bare bcrypt digests (`$2a$…`) | a compatibility verifier that detects the bare digest and invokes `BCryptPasswordHasher.verify()` with an in-memory `bcrypt$` prefix, or calls `bcrypt.checkpw()` directly | adding `BCryptPasswordHasher` to `PASSWORD_HASHERS` is insufficient: Django derives the hasher name from the stored prefix, so `check_password()` rejects a bare digest |
| bcrypt cost 12 (1 in test) | set the cost on the compatibility writer explicitly | keep the Ruby side's cost so the two services agree on newly generated hashes |
| newly stored hashes during dual-run | write the same bare bcrypt representation Devise expects | stock Django `set_password()` stores `bcrypt$$2b$…`; writing that value into the shared column closes the Ruby rollback path |
| optional pepper | include the exact Devise pepper step in the compatibility verifier/writer | omitting it rejects every peppered digest; never silently disable the pepper |
| password change endpoint | route verification and writes through the compatibility layer | both the Ruby and Django services must accept a password changed by either side during dual-run |

Acceptance for this section requires two real-credential probes before the auth slice is called ported:

1. an existing digest copied from the shared users table authenticates through the **actual Django login/auth path**;
2. a password changed through Django can authenticate through both Django and the still-running Ruby service.

## 10. Config and middleware

| Ruby/Rails construct | Django/DMR construct | Parity note |
|---|---|---|
| `config/application.rb` settings | Django settings module | classify every setting as ported / irrelevant / dropped, with the dropped list written into the parity report |
| `config/initializers/*.rb` | settings plus `AppConfig.ready()` | keep `ready()` free of writes and network calls |
| `config.middleware` (`use`, `insert_before`, `insert_after`) | `MIDDLEWARE` list | order is the contract; each entry is ported, replaced by a DMR mechanism ([`grape-to-dmr-map.md` §8](grape-to-dmr-map.md#8-filters-helpers-middleware-auth)), or dropped with a reason |
| rack-attack initializer (rack-attack README) | `dmr.throttling.SyncThrottle` / `Rate` + `DMR_SETTINGS["throttling"]` + the cache entry it needs | see [`grape-to-dmr-map.md` §12](grape-to-dmr-map.md#12-throttling-and-cors) |
| rack-cors initializer (rack-cors README) | `django-cors-headers` settings | see [`grape-to-dmr-map.md` §12](grape-to-dmr-map.md#12-throttling-and-cors) |
| `config.force_ssl = true` | `SECURE_SSL_REDIRECT = True` (plus the HSTS settings the app set) | a redirect is observable by clients; compare the status and `Location` in the harness |
| `config.time_zone` | `TIME_ZONE` | affects rendered times and offline jobs |
| `config.filter_parameters` | an exception-reporter filter subclass with `sensitive_post_parameters` | different mechanism, same intent: the same parameter list must not reach logs or error pages |
| Rails error-page/middleware behaviour for unhandled exceptions | `DEBUG = False` + `handler500` | the 500 body is part of the frozen contract if any client parses it ([`grape-to-dmr-map.md` §7](grape-to-dmr-map.md#7-errors)) |

`Cannot be carried over:` Rails initializer load-order effects; `to_prepare`-style reload hooks; any middleware whose
work was per-route (Django middleware is not route-scoped — such entries move into the shared base controller instead).

## 11. Ruby to Python idioms

| Ruby/Rails construct | Django/DMR construct | Parity note |
|---|---|---|
| `nil` / `nil?` | `None` / `is None` | never use bare truthiness as the port of `nil?` |
| `blank?` / `present?` | pydantic `BeforeValidator` that strips and rejects empty values, or an explicit service check | Ruby's `blank?` is true for `""`, whitespace-only strings, `[]`, `{}`, and `nil`; a typed DTO field accepts `" "` — decide per field: normalise, or drift-list |
| `HashWithIndifferentAccess` (the Grape params class default) | typed DTOs | string/symbol key duality disappears; `declared(...)`/`stringify` gymnastics have no target — read typed attributes instead |
| symbols as protocol values (`:active`) | `str` enum members / `TextChoices` | the wire form is the string name; keep the exact spelling |
| `true` / `false` truthiness (Ruby: only `nil` and `false` are falsy) | Python truthiness (`0`, `""`, `[]` are falsy) | translate `if x` as `if x is not None` where the Ruby value could be `0` or `""` |
| `7 / 2 == 3` (integer division) | `/` is true division; use `//` | a silent port changes stored values; audit every division in money/count code |
| `String#to_i` (`"12abc".to_i == 12`) | `int()` raises `ValueError` | decide per call site: strict parse (and drift-list previously-accepted junk) or explicit extraction |
| `Time.parse` | `datetime.fromisoformat` | see §5 |
| `attr_accessor` / dynamic attributes | explicit dataclass/DTO fields | no method-missing attributes on serialised objects |
| `include Enumerable` / `each` chains | list comprehensions / `QuerySet` iteration | keep laziness: a Ruby lazy chain ported to eager lists can change query counts, which the differential harness may surface as latency, not as a diff |

`Cannot be carried over:` `blank?` semantics on typed fields (explicit per-field decision only); `HashWithIndifferentAccess`
duality; `Time.parse` laxness; Ruby truthiness.

## 12. Test suite port

| Ruby/Rails construct | Django/DMR construct | Parity note |
|---|---|---|
| rspec-rails request specs in `spec/requests`, running the full stack through Rack::Test with the default host `www.example.com` (rspec-rails README) | pytest + `dmr.test.client.DMRClient` / `dmr.test.DMRRequestFactory` | same requests, same assertions; Django's test client uses `testserver` as host — any test asserting an absolute URL (mailer links, `Location`) needs the host made explicit and drift-listed |
| `get path, params: {…}, headers: {…}` | `client.get(path, data=…, headers=…)` on a `DMRClient` | keep the header casing and the query encoding of the original request (bracket-form arrays) |
| rswag specs in `spec/requests` (rswag README) | the contract source: parametrized pytest cases asserting status + body schema | rswag already encodes parameters and response schemas — it is the input to [`differential-parity.md` §2](differential-parity.md#2-contract-capture), not just a test |
| factory_bot `build` / `create` / `attributes_for` / `build_stubbed`, sequences, traits, transient and association attributes (factory_bot README) | factory_boy factories with the same names and defaults | `build_stubbed` has no analogue: substitute an unsaved instance and state what changes |
| `use_transactional_fixtures = true` (generated `rails_helper`) | `pytest-django` `django_db` (transactional by default per test) | one test per behaviour; no shared mutable state between tests |
| DatabaseCleaner `:transaction` / `:truncation` — `:truncation` requires `use_transactional_fixtures = false` (database_cleaner README) | `--reuse-db` for speed, `django_db(transaction=True)` where truncation semantics are needed | only tests that genuinely need commit visibility get `transaction=True` (it also matches the `on_commit` tests in §4) |
| RSpec `let` / `before` blocks | pytest fixtures | keep the fixture scope no wider than the Ruby `let` scope |
| per-endpoint happy-path specs | one test per behaviour plus a negative-path test for every declared response | every status in the frozen contract (400/401/403/404/405/429/500) gets at least one test |
| test-settings response validation | keep `DMR_SETTINGS["validate_responses"]` enabled in test settings | response validation is a tier-1 gate element ([`differential-parity.md` §1](differential-parity.md#1-tier-1-per-slice-gate)); never disable it to make a slice green |

`Cannot be carried over:` Rack::Test's `www.example.com` default host assumptions in specs that build absolute URLs;
`build_stubbed` semantics; DatabaseCleaner's out-of-band truncation between examples (replaced by pytest-django's
database management).

## Cannot be carried over

These are structurally inexpressible in Django/DMR and must each appear as an explicit `approved drift` line in
`port/parity-report.md` — never silently dropped:

- **Ruby metaprogramming** — `method_missing`, `define_method`, monkey patches, `alias_method` chains, and `concern`
  inclusion-time DSL are rewritten explicitly; every occurrence is enumerated in `port/inventory.md` with its
  replacement or its drift line.
- **ActiveRecord callbacks as implicit hooks** — a callback fired on every `save()` from any call site; the port has
  services called deliberately, so any behaviour that depended on a write happening outside the service must be
  re-attached at that call site (Django signals are not an acceptable substitute).
- **`blank?` / `present?` semantics on typed fields** — the empty-string/whitespace/empty-collection family has no
  single Django expression; each field gets an explicit normalisation rule or a drift line.
- **`Time.parse` laxness** — non-ISO input and trailing junk were accepted; `datetime.fromisoformat` is strict.
- **`HashWithIndifferentAccess` string/symbol duality** — typed DTOs have one key space; code that switched between
  string and symbol access is rewritten, and any client-visible consequence is drift-listed.
- **STI query semantics** — `type_condition` filtered every query automatically; Django requires an explicit dispatch
  helper or manager filter, and every query that used to be implicitly scoped must be audited.
- **AR `enum` ordinal coupling** — the stored integer ordinals come from the original migration and must be matched
  exactly; a Django `IntegerChoices` written from the symbol names in the model file can silently renumber them.
- **Counter-cache automation** — AR maintained `*_count` columns; Django drifts unless every write path updates them
  with `F()` expressions, plus a one-off backfill.
- **Storage `Mirror` service** — dual-write with a read preference has no single Django setting.
- **Rails initializer / middleware load-order behaviour** — only the observable HTTP effects survive; per-route
  middleware must move into the shared base controller.
