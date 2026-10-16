# sqlmap Cookie Automator

`sqlmap_cookie_automator.py` replays an authenticated login request, captures the response cookies, and gives them to sqlmap through `--live-cookies`. It repeats the login at a configurable interval so a long-running authorized test can keep using fresh session cookies.

## How it works

1. The automator reads and replays the raw HTTP login request in `login.req`.
2. It captures cookies returned in the login response's `Set-Cookie` headers.
3. It writes the applicable cookies to a temporary live-cookie file with owner-only permissions.
4. It starts sqlmap with `--live-cookies`, which rereads that file before each request.
5. Every `--refresh-seconds`, the automator replays the login request and atomically replaces the cookie file. If re-login fails, it stops sqlmap rather than continuing with stale authentication.
6. When sqlmap exits or is interrupted, the automator removes the temporary cookie file.

The login request and sqlmap target request are separate: `login.req` authenticates; `target.req` is the request sqlmap tests.

## Requirements

- Python 3.10 or newer
- sqlmap installed and available as `sqlmap` on `PATH`, or specify its executable with `--sqlmap`
- A raw HTTP login request saved as `login.req`

## Prepare the login request

Capture the successful login POST from an intercepting proxy such as Burp or from browser developer tools. Save it as a raw HTTP request, including the request line, headers, a blank line, and the body. For example:

```http
POST /login HTTP/1.1
Host: app.example
Content-Type: application/x-www-form-urlencoded

username=YOUR_USERNAME&password=YOUR_PASSWORD
```

The example is illustrative; use the actual method, path, headers, and body required by your application. Keep `login.req` private because it may contain credentials, CSRF tokens, or other secrets. The automator replays this request for each refresh. It does not support chunked transfer-encoded login requests. A static one-time CSRF value may expire; such a flow needs a fresh-token step or an app-specific refresh process.

## Scenario 1: test a saved target request with `-r`

Save the authenticated request you want sqlmap to test as `target.req`, then run:

```bash
python3 sqlmap_cookie_automator.py \
  --login-request-file login.req \
  --refresh-seconds 180 \
  -- -r target.req --batch
```

The automator reads the target host and path from `target.req` to select applicable cookies. Raw request files often omit the URL scheme. The automator uses HTTPS by default; for an HTTP target, add `--target-scheme http` before `--`:

```bash
python3 sqlmap_cookie_automator.py \
  --login-request-file login.req \
  --refresh-seconds 180 \
  --target-scheme http \
  -- -r target.req --batch
```

## Scenario 2: test a URL with `-u`

Pass the URL after `--` so the automator can infer the target and pass the same options to sqlmap:

```bash
python3 sqlmap_cookie_automator.py \
  --login-request-file login.req \
  --refresh-seconds 180 \
  -- -u 'https://app.example/path/to/test' --batch
```

Include `http://` or `https://` in the URL. sqlmap-specific options such as `--batch`, `--dbs`, or `--tables` go after the `--` separator.

## Refresh interval and expired sessions

Set `--refresh-seconds` to less than the session lifetime, with enough margin for network delays. For a five-minute session, `180` seconds refreshes it about two minutes before expiry. sqlmap rereads the live-cookie file before each request, so requests after a successful refresh use the replacement cookie.

Refresh is time-based, not triggered by a `401` response. If the session expires before the next scheduled refresh, a request may still use the expired cookie. If the login replay fails, the automator stops sqlmap. Reduce the interval if the application's session lifetime is shorter or uncertain.

`--cookie-name SID` is optional. Use it when you want the automator to fail if a particular session cookie was not returned. Without it, the automator passes all cookies applicable to the inferred target host, path, and scheme.

If sqlmap is not on `PATH`, supply its executable with `--sqlmap`, for example:

```bash
python3 sqlmap_cookie_automator.py \
  --login-request-file login.req \
  --refresh-seconds 180 \
  --sqlmap /path/to/sqlmap \
  -- -r target.req --batch
```
