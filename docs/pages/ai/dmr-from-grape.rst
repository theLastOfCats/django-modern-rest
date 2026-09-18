Ruby on Rails with Grape migration
==================================

By example, you can use agent skill
`dmr-from-grape <https://github.com/wemake-services/django-modern-rest/tree/master/.agents/skills/dmr-from-grape>`_
when you have an existing Ruby service — a ``Ruby on Rails`` application
whose HTTP API is built with ``Grape`` — and you want to re-implement it
with ``Django`` and ``django-modern-rest``.

This is a **full-service port**, not a transport swap:
no Ruby code survives, so the domain logic, models, background jobs, mailers, storage,
and the test suite are all re-written in Python.
The database schema is kept compatible (same tables and columns) until cutover,
so both services can run against one database while the port is verified.

.. important::

  A full port re-implements domain logic, so the usual migration risks apply with more weight.
  Validate before production use:

  - the data model and its constraints against ``db/schema.rb``,
  - password-hash compatibility (``Devise`` stores bare bcrypt digests that stock Django
    neither identifies nor writes compatibly; the port needs a verified compatibility layer),
  - token rotation and TTL behavior,
  - rate limits and ``CORS`` credentials handling,
  - side effects: background jobs, mail delivery, webhooks, storage writes,
  - a recorded-corpus differential replay against the still-running Ruby service.

  Ruby metaprogramming and ``ActiveRecord`` callbacks have no mechanical translation:
  they are re-written explicitly, and anything that cannot be reproduced is reported
  as approved drift rather than silently dropped.

  The skill's reference maps cite the ``Grape``, ``grape-entity``, ``grape-swagger``,
  ``rswag``, ``rack-attack``, ``rack-cors``, pagination gem, ``Devise``/``doorkeeper``,
  and ``ActiveSupport`` behaviors it relies on.
  Check the project's ``Gemfile.lock`` pinned versions before assuming a specific behavior applies.


How to use in Codex
-------------------

1. Point Codex to the Ruby service: ``app/api``, ``lib``, ``db/schema.rb``, jobs, mailers,
   initializers, and ``spec/``.
2. Ask Codex to use the skill ``$dmr-from-grape``.
3. Freeze the contract first, then port one Grape resource (one vertical slice) at a time,
   running the ported tests after each slice.

You can use prompt like this:

.. code-block:: text

   $dmr-from-grape Port the Ruby service in `app/api/` (Rails + Grape) to Django with
   django-modern-rest. Freeze the contract first (rswag or grape-swagger) and put it in
   `port/contract/`. Keep the database schema identical to `db/schema.rb` so both services
   can share one database. Preserve paths, methods, status codes, headers, and error payload
   shapes, the auth headers and token rotation, rack-attack limits, and the pagination
   envelope. Port domain logic into services (no signals), port the RSpec request/rswag specs
   to pytest with DMRClient, and keep `validate_responses` on outside production.
   Run the recorded-corpus differential replay against the Ruby service before cutover,
   and report `preserved behavior`, `approved drift`, `security posture changes`, and
   `unresolved gaps`.

How to use in Claude Code
-------------------------

1. Add the marketplace and install the plugin:

.. code-block:: text

   /plugin marketplace add wemake-services/django-modern-rest
   /plugin install dmr-from-grape@django-modern-rest

2. Verify the plugin is installed and enabled:

.. code-block:: text

   /plugin

3. Invoke the skill:

.. code-block:: text

   /dmr-from-grape

4. Then describe the port scope and constraints in a normal prompt.


What is migrated
----------------

- ``Grape`` resources, ``mount``/``namespace`` composition, routes, and ``params`` blocks to
  DMR routers, :class:`~dmr.controller.Controller` handlers, and typed DTOs
  (``Path`` / ``Query`` / ``Body`` / ``Headers`` / ``Cookies``)
- ``Grape`` status/header behavior and ``rescue_from`` / ``error!`` handling to the DMR error chain
  (``@modify(error_handler=...)``, ``handle_error``, the global error handler)
- ``grape-entity`` presenters to return annotations, DTOs, and ``ResponseSpec`` declarations
- auth gems (``devise_token_auth``, ``doorkeeper``) to DMR auth classes, including bcrypt
  password-hash compatibility
- ``rack-attack`` limits and ``rack-cors`` configuration to DMR throttling and CORS settings
- pagination gems and their envelopes and headers to DMR DTOs, pagination helpers, and headers
- ``Ruby on Rails`` models, migrations, jobs, mailers, and storage to Django equivalents
  with a compatible schema
- ``RSpec`` request specs and ``rswag`` specs to pytest with ``DMRClient`` / ``DMRRequestFactory``
- the wire format (``ActiveSupport`` JSON conventions) as a serializer and renderer concern
- a parity report with ``preserved behavior``, ``approved drift``,
  ``security posture changes``, and ``unresolved gaps``, plus a per-endpoint status matrix
