#### Description

The `github` command registers GitHub's authenticated-user check on a running `aux4/mock` server in one shot, so a test can stand up the boilerplate and then stub only the business endpoints it cares about. It is contributed to `aux4/mock`'s `mock:preset` profile by installing this package — the mock core ships no presets and knows nothing about GitHub.

GitHub authenticates the REST API with a token (`Authorization: Bearer <token>` / `token <token>`); there is no OAuth `/token` exchange endpoint to stub, so this preset covers the authenticated-user check itself. It registers two stubs by calling `aux4 mock stub`:

- **`GET /user`** → `200` realistic GitHub user (`login`, `id`, `name`, `email`, `html_url`, `public_repos`, …), gated on `Authorization: Bearer *`.
- **`GET /user`** → `401` bare fallback with GitHub's `Requires authentication` envelope.

The two stubs sit on the same path: a happy path gated on any bearer token and a fallback returning the real `401`. So a request with a bearer token gets the user; one without gets GitHub's error shape.

The `login` handle is **derived** from `--user` — lowercased with spaces removed (e.g. `--user "Sally"` → `login: sally`).

The preset is **additive** — it only calls `mock stub`, so it never clears existing stubs and layers cleanly with your own business stubs.

#### Usage

```bash
aux4 mock preset github [--port 7070] [--name <handle>] [--stateDir <dir>] \
  [--user <name>] [--email <email>] [--id <number>]
```

--port       Port of the running mock server, derives stateDir (default: 7070)
--name       Address the server by handle instead of port (precedence over --port)
--stateDir   Explicit state directory (highest precedence)
--user       Display name (name) in GET /user (default: Sally); login is derived from it
--email      Email in GET /user (default: sally@example.com)
--id         Numeric user id in GET /user (default: 1234567)

#### Example

```bash
aux4 mock start --port 8080
aux4 mock preset github --port 8080
```

```text
stub GET /user -> 200
stub GET /user -> 401
```

```bash
curl -s http://localhost:8080/api/user -H 'Authorization: Bearer x'
# {"avatar_url":"...","login":"sally","name":"Sally","id":1234567,"email":"sally@example.com",...}

curl -s http://localhost:8080/api/user
# {"message":"Requires authentication","documentation_url":"https://docs.github.com/rest"}
```
