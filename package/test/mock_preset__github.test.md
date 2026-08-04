# community/mock-preset-github

Proves that installing this package makes `aux4 mock preset github` available purely through
aux4's `global.aux4` profile merge — the `aux4/mock` core has zero knowledge of GitHub. The
preset stands up the authenticated-user check (bearer-gated `GET /user` + the real
`Requires authentication` 401 envelope) with one command, and derives the `login` handle
from `--user`.

The server is started inside the first test's `execute` block (not `beforeAll`, which the
harness would kill) and torn down in `afterAll` on a unique port. It is a detached process,
so it survives across the sibling `##` scenarios; each override scenario clears the stub
store first. All checks use plain `curl` with no external network.

```afterAll
aux4 mock stop --port 18786 2>/dev/null
pkill -f "18786" 2>/dev/null
true
```

## preset github via install alone

### should list github under mock presets

```execute
aux4 mock presets
```

```expect:partial
github
```

### should start the server and apply the github preset

```execute
aux4 mock start --port 18786
sleep 1
aux4 mock preset github --port 18786
```

```expect:partial
stub GET /user -> 200
```

### should return the GitHub user when a bearer token is present

```execute
curl -s -w "\n%{http_code}" http://localhost:18786/api/user -H 'Authorization: Bearer sally-token'
```

```expect
{"avatar_url":"https://avatars.githubusercontent.com/u/1234567?v=4","bio":null,"blog":"","company":null,"created_at":"2020-01-01T00:00:00Z","email":"sally@example.com","followers":5,"following":3,"html_url":"https://github.com/sally","id":1234567,"location":null,"login":"sally","name":"Sally","node_id":"MDQ6VXNlcjEyMzQ1Njc=","public_gists":2,"public_repos":10,"site_admin":false,"type":"User","updated_at":"2024-01-01T00:00:00Z"}
200
```

### should return a 401 Requires authentication envelope without a bearer token

```execute
curl -s -w "\n%{http_code}" http://localhost:18786/api/user
```

```expect
{"message":"Requires authentication","documentation_url":"https://docs.github.com/rest"}
401
```

## overrides: pin a known user

### should re-apply the github preset with overrides

```execute
aux4 mock reset --port 18786 --stubs true
aux4 mock preset github --port 18786 --user "Devon" --email devon@corp.io --id 7654321
```

```expect:partial
stub GET /user -> 200
```

### should return the overridden user (with derived login) with a bearer token

```execute
curl -s http://localhost:18786/api/user -H 'Authorization: Bearer devon-token'
```

```expect
{"avatar_url":"https://avatars.githubusercontent.com/u/7654321?v=4","bio":null,"blog":"","company":null,"created_at":"2020-01-01T00:00:00Z","email":"devon@corp.io","followers":5,"following":3,"html_url":"https://github.com/devon","id":7654321,"location":null,"login":"devon","name":"Devon","node_id":"MDQ6VXNlcjEyMzQ1Njc=","public_gists":2,"public_repos":10,"site_admin":false,"type":"User","updated_at":"2024-01-01T00:00:00Z"}
```

## layering: a preset coexists with a later business stub

### should apply the preset then a repos business stub

```execute
aux4 mock reset --port 18786 --stubs true
aux4 mock preset github --port 18786
aux4 mock stub --port 18786 --method GET --path /user/repos --status 200 --body '[{"id":1,"name":"hello","full_name":"sally/hello"}]'
```

```expect:partial
stub GET /user/repos -> 200
```

### should still serve the preset user endpoint

```execute
curl -s http://localhost:18786/api/user -H 'Authorization: Bearer sally-token'
```

```expect
{"avatar_url":"https://avatars.githubusercontent.com/u/1234567?v=4","bio":null,"blog":"","company":null,"created_at":"2020-01-01T00:00:00Z","email":"sally@example.com","followers":5,"following":3,"html_url":"https://github.com/sally","id":1234567,"location":null,"login":"sally","name":"Sally","node_id":"MDQ6VXNlcjEyMzQ1Njc=","public_gists":2,"public_repos":10,"site_admin":false,"type":"User","updated_at":"2024-01-01T00:00:00Z"}
```

### should serve the layered repos business endpoint

```execute
curl -s -w "\n%{http_code}" http://localhost:18786/api/user/repos -H 'Authorization: Bearer sally-token'
```

```expect
[{"id":1,"name":"hello","full_name":"sally/hello"}]
200
```
