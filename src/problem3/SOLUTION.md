# Problem 3: Stabilizing the Platform

## How I diagnosed it

Ran the stack locally exactly as instructed in the README:

```bash
docker compose up --build -d
```

All four containers started fine, but hitting the endpoints told a different story:

```bash
curl http://localhost:8080/            # -> 200 "Welcome to the platform"
curl http://localhost:8080/api/users   # -> 502 Bad Gateway
curl http://localhost:8080/status      # -> 404 Not Found
```

So the root `/` (served directly by nginx) worked, but everything going through the API was broken — matching the "API is unreliable and sometimes inaccessible" report. From there I compared the nginx config against the API source to trace the request path:

- `nginx/conf.d/default.conf` proxies `/api/` to `http://api:3001`
- `api/src/index.js` actually listens on `app.listen(3000, ...)`

That mismatch is the 502 — nginx forwards to a port nothing is listening on. The 404 on `/status` was simpler: the API defines a `/status` route, but nginx's config only has locations for `/` and `/api/`, so any other path (including `/status`) falls through to nginx's default 404.

## What problems I found

1. **Wrong proxy port** — nginx forwards `/api/*` to port `3001`; the API listens on `3000`. Causes every `/api/*` request to fail with `502 Bad Gateway`.
2. **Missing route for `/status`** — the API exposes it, but nginx never proxies it, so it 404s instead of reaching the API.

## Fixes applied

`nginx/conf.d/default.conf`:
- Changed `proxy_pass http://api:3001;` to `proxy_pass http://api:3000;` to match the port the API actually listens on.
- Added a `location /status { proxy_pass http://api:3000; }` block so the API's existing status endpoint is actually reachable through nginx instead of 404ing.

## Monitoring/alerts I'd add

- External uptime check hitting `/api/users` and `/status` through nginx (port 8080)
- Alert on nginx 502/504 rate — an instant signal that a proxied backend is unreachable.
- Alert on nginx 404 rate on `/api/*` and `/status` — catches routing/config drift like this one.

## How I'd prevent this in production

- A CI smoke test that runs `docker compose up --build` and curls every route through nginx (not directly against the API container) before merging — this exact bug would have failed that check immediately.
- Avoid hardcoding the same port in two separate files (nginx config and app code); define it once via a shared env var so nginx and the app can't drift out of sync.
