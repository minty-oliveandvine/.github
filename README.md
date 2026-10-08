# minty-oliveandvine/.github

The shared CI for every Minty repo. Each repo's own `.github/workflows/ci.yml` is about a dozen
lines that call one of the workflows here, so fixing CI for all of them is one change in one
place.

Part 3 step 1 of [Minty's modernisation plan](https://github.com/minty-oliveandvine/Minty/blob/development/docs/modernisation/modernisation_plan.md).

## Why this repo is public

A public repository cannot call a reusable workflow that lives in a private one, and
`minty-web`, `minty-payment-request-web` and `minty-onboarding-web` are public. A private
`.github` could therefore only serve half the estate. Nothing but workflow YAML lives here;
every secret stays in the repo that calls it, named explicitly (never `secrets: inherit`).

`minty-oliveandvine` is a **user account, not an organisation**. So there are no org-level
secrets to inherit (the one shared `SECRET_KEY` comes from `minty-infra`) and no workflow
templates. Reusable workflows work the same either way.

## Callers pin a tag, never `@main`

```yaml
uses: minty-oliveandvine/.github/.github/workflows/next-web.yml@v1
```

`@main` would deploy a change to this repo straight into eight others with no run in between.
It is also the plan's own no-floating-pins rule for shared code.

## The workflows

| Workflow | Who calls it | What it runs |
|---|---|---|
| `python-api.yml` | minty-subscription-api, minty-payment-request-api, minty-onboarding-api | ruff; pytest on SQLite; pytest again on a Postgres built from Minty's `01_schema_rebased.sql` |
| `flask-app.yml` | Minty | ruff; the whole pytest suite on Postgres under `-n auto` (Minty has no SQLite mode) |
| `next-web.yml` | minty-web, minty-payment-request-web, minty-onboarding-web, daily-minty-landing-page | `npm ci`; typecheck; lint; unit tests; `next build`; Playwright only where the config starts its own server |
| `stack-e2e.yml` | Minty (`e2e.yml`) | the whole seven-service stack, the schema from `01_schema_rebased.sql`, the seed, then one repo's browser specs |
| `actionlint.yml` | this repo | lints the four above |

### `python-api.yml`

| Input | Required | Notes |
|---|---|---|
| `python-version` | yes | `3.13` for minty-subscription-api, `3.11` for the other two |
| `schema-name-test` | no | a pytest path re-run against `?schema=pettycash_alt`; empty skips it |

| Secret | Notes |
|---|---|
| `MINTY_READ_TOKEN` | fine-grained PAT, read on the **contents** of `minty-oliveandvine/Minty` only |

The Postgres pass is the one that catches a `shared_models` mirror drifting from the real
schema. It needs Minty checked out beside the caller, because all three repos' `conftest.py`
load `Minty/tests/pg_harness.py` by path.

### `flask-app.yml`

| Input | Required | Notes |
|---|---|---|
| `python-version` | no | `3.11` (Minty pins `>=3.11,<3.12`) |

No secret and no cross-repo checkout: Minty is the repo that holds the harness and the schema
file.

**ruff runs here too**, since 2026-10-07. It had never been run repo-wide and reported 96
findings; Minty now carries a `[tool.ruff]` section whose `per-file-ignores` say where the default
rules are wrong about this repo (the model hub's registering imports, Alembic's generated headers,
the tests' pre-import environment setup, the generator scripts' style) and the rest were fixed.

### `next-web.yml`

| Input | Default | Notes |
|---|---|---|
| `node-version` | `24` | no repo pins `engines`, so this is where Node is chosen - 24 is what every Vercel project builds with |
| `typecheck-script` | `typecheck` | the landing page calls it `type-check` |
| `lint` | `true` | `false` only where lint debt would make a repo permanently red — a hole to close, not a setting |
| `unit-tests` | `true` | `false` for the landing page, which has none |
| `extra-scripts` | — | further npm scripts, one per line (`check:routes`) |
| `build-env` | — | `KEY=VALUE` lines `next build` needs; a line that is not `KEY=VALUE` fails the job |
| `e2e` | `false` | `true` only for the landing page |

### `stack-e2e.yml`

| Input | Default | Notes |
|---|---|---|
| `suite` | required | `Minty`, `minty-web` or `minty-payment-request-web` |
| `ref` | `development` | the branch of the other six repos; Minty uses the caller's ref |
| `spec` | — | an optional Playwright filter |

| Secret | Notes |
|---|---|
| `STACK_READ_TOKEN` | fine-grained PAT, read on contents of the four private repos |

**No production secret is needed.** `SECRET_KEY` is generated per run (it only has to be the
same for every service), `S3_URL` falls back to compose's dummy, Stripe and SMTP stay unset, and
`E2E_XERO` stays off — so no run reaches a real Xero org or a real mailbox.

`minty-onboarding-web` is refused with a reason: its `walk`/`resume` specs need a disposable
entity in `onboarding` status that `scripts/e2e_seed.py` does not create yet.

## Where a check is switched off, and why

One, declared in the caller rather than skipped silently:

| Repo | Off | Why |
|---|---|---|
| daily-minty-landing-page | `unit-tests` | it has none, and no `test` script |

The `lint` input exists for a repo whose lint debt would otherwise make CI permanently red - which
matters because Render deploys on **checks pass**, so a repo that can never go green cannot deploy.
Nothing passes it today: `minty-payment-request-web`'s 20 errors were fixed on 2026-10-07 rather
than switched off.

## Adding a repo

Copy the nearest existing caller, change the inputs, add the secret it needs under
**Settings > Secrets and variables > Actions**. If it is a new kind of repo, add a workflow here
rather than hand-writing a job in the repo — that is the drift this repo exists to prevent.
