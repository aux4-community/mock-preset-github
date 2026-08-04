# community/mock-preset-github

A **service preset** for [`aux4/mock`](https://hub.aux4.io/package/aux4/mock) that stands up GitHub's authenticated-user check on a running mock server with a single command: the bearer-gated `GET /user` endpoint returning a realistic GitHub user, plus GitHub's real `Requires authentication` `401` envelope. Install it and `aux4 mock preset github` becomes available — the mock core stays preset-agnostic and knows nothing about GitHub.

This is a pure `.aux4` package (no binary, no bundle). It works entirely by contributing a `github` command into `aux4/mock`'s `mock:preset` profile via aux4's `global.aux4` profile merge.

## Installation

```bash
aux4 aux4 pkger install community/mock-preset-github
```

Installing this package pulls in `aux4/mock` as a dependency.

## Usage

```bash
aux4 mock start --port 8080
aux4 mock preset github --port 8080
```

```text
stub GET /user -> 200
stub GET /user -> 401
```

Confirm it is installed and visible to the mock:

```bash
aux4 mock presets
```

```text
Installed presets (apply with: aux4 mock preset <service> --port <port>):
  github
```

## What it stubs

| Method + Path | Status | Behavior |
|---------------|--------|----------|
| `GET /user` | `200` | Realistic GitHub user (`login`, `id`, `name`, `email`, `html_url`, `public_repos`, …) — **only** when `Authorization: Bearer <token>` is present |
| `GET /user` | `401` | GitHub `Requires authentication` envelope — fallback when no bearer token |

GitHub authenticates the REST API with a token (`Authorization: Bearer <token>` / `token <token>`); there is no OAuth `/token` exchange endpoint to stub, so this preset covers the authenticated-user check itself. `GET /user` is registered as **two** stubs on the same path: a happy path gated on `Authorization: Bearer *` (any token) and a bare fallback returning the `401`:

```bash
# with a bearer token → 200 GitHub user
curl -s http://localhost:8080/api/user -H 'Authorization: Bearer x'
# {"avatar_url":"...","...":"...","login":"sally","name":"Sally","id":1234567,...}

# without one → 401 GitHub error envelope
curl -s http://localhost:8080/api/user
# {"message":"Requires authentication","documentation_url":"https://docs.github.com/rest"}
```

The preset is **additive** — it only calls `aux4 mock stub`, so it never clears existing stubs. Layer the business endpoints your test exercises on top:

```bash
aux4 mock preset github --port 8080
aux4 mock stub --port 8080 --method GET \
  --path /user/repos \
  --status 200 --body '[{"id":1,"name":"hello","full_name":"sally/hello"}]'
```

## Overrides

All optional, with sane defaults, so a test can assert a known identity:

```bash
aux4 mock preset github --port 8080 \
  --user "Devon" --email devon@corp.io --id 7654321
```

- `--user` — display name (`name`) in `GET /user` (default `Sally`). The `login` handle is **derived** from it (lowercased, spaces removed) — e.g. `--user "Sally"` yields `login: sally`.
- `--email` — email in `GET /user` (default `sally@example.com`)
- `--id` — numeric user id in `GET /user` (default `1234567`)

Also accepts `--name` and `--stateDir` to address a server the same way every other `aux4/mock` command does (precedence: `--stateDir` > `--name` > `--port`).

## Base URLs and paths

The preset stubs GitHub's real path (`/user`). Point the code-under-test's base URL at `http://localhost:<port>/api` and let it append the real path — `aux4/mock` strips the `/api` mount prefix before matching.

## Using in tests

`aux4/mock` and this preset are plain CLIs, so they drop straight into an [`aux4/test`](https://hub.aux4.io/package/aux4/test) `.test.md`. In CI, declare both packages on the test job:

```yaml
- uses: aux4/action@v1
  with:
    command: test
    packages: aux4/mock,community/mock-preset-github
```

## This package as a template

`community/mock-preset-github` follows the same pattern as [`community/mock-preset-google`](https://hub.aux4.io/package/community/mock-preset-google), the reference implementation for writing your own `community/mock-preset-<service>` package: depend on `aux4/mock`, re-declare the `mock` profile with a `preset` routing command, and add a `<service>` command under `mock:preset` whose `execute` is a sequence of `aux4 mock stub` calls.
