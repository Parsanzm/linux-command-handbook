# curl — Edge Cases & Gotchas

> `curl` looks like a simple "fetch a URL" tool, but exit-code assumptions,
> redirect handling, and quoting rules routinely produce silently wrong
> results in scripts.

---

## Table of Contents

- [A 404 or 500 Response Is Still "Success" as Far as curl's Exit Code Is Concerned](#a-404-or-500-response-is-still-success-as-far-as-curls-exit-code-is-concerned)
- [-d Silently Switches the Request to POST](#-d-silently-switches-the-request-to-post)
- [curl Doesn't Follow Redirects Unless You Ask It To](#curl-doesnt-follow-redirects-unless-you-ask-it-to)
- [-k Disables Certificate Verification Entirely — Not Just for "This One Warning"](#-k-disables-certificate-verification-entirely--not-just-for-this-one-warning)
- [Unquoted URLs with Special Characters Get Mangled by the Shell, Not curl](#unquoted-urls-with-special-characters-get-mangled-by-the-shell-not-curl)
- [-O Fails Silently on URLs Without an Obvious Filename](#-o-fails-silently-on-urls-without-an-obvious-filename)
- [Piping curl Directly into bash Runs Whatever the Server Sends, No Review Possible](#piping-curl-directly-into-bash-runs-whatever-the-server-sends-no-review-possible)
- [A Redirect Can Silently Change the Request Method](#a-redirect-can-silently-change-the-request-method)
- [-s Also Suppresses Error Messages, Not Just the Progress Meter](#-s-also-suppresses-error-messages-not-just-the-progress-meter)
- [Cookies Aren't Persisted Between Separate curl Invocations by Default](#cookies-arent-persisted-between-separate-curl-invocations-by-default)
- [A Successful Connection Doesn't Mean a Successful Transfer If It's Interrupted Midway](#a-successful-connection-doesnt-mean-a-successful-transfer-if-its-interrupted-midway)

---

## A 404 or 500 Response Is Still "Success" as Far as curl's Exit Code Is Concerned

### The single most common curl scripting bug
```bash
curl -s https://api.example.com/does-not-exist -o response.json
echo $?
# 0     ← ⚠️ curl considers the HTTP TRANSACTION itself successful —
# it made the connection, sent the request, and received a complete
# response — even though that response was a 404 Not Found. curl does
# NOT treat HTTP-level error status codes as command failures by default.

if [ $? -eq 0 ]; then
  echo "Success!"   # ⚠️ THIS RUNS even for a 404 response, silently
                     # masking a real failure in a naive script
fi

# The fix: explicitly opt in to treating HTTP errors as failures
curl -sf https://api.example.com/does-not-exist -o response.json
echo $?
# 22    ← --fail (-f) makes curl return non-zero for 4xx/5xx responses
```

---

## -d Silently Switches the Request to POST

### An easy trap when trying to send data with a GET request
```bash
curl -d "search=widgets" https://api.example.com/search
# ⚠️ This sends a POST request, NOT a GET — -d/--data implicitly
# changes the HTTP method to POST unless -X explicitly overrides it,
# even if the endpoint was actually expecting a GET with query
# parameters instead.

# For a GET request WITH data as query parameters, don't use -d at all:
curl "https://api.example.com/search?search=widgets"

# If you genuinely need -d's data but with GET specifically:
curl -G -d "search=widgets" https://api.example.com/search
# -G tells curl to append -d's data as URL query parameters instead
# of putting it in a POST body
```

---

## curl Doesn't Follow Redirects Unless You Ask It To

### A very common "why is the response body wrong/empty" confusion
```bash
curl https://example.com/old-page
# <html><body>You are being redirected to <a href="/new-page">here</a></body></html>
# ⚠️ Without -L, curl does NOT follow a 3xx redirect automatically —
# it just prints whatever short response body the server sent
# alongside the redirect status, which is often nearly empty or just
# a pointer message, NOT the actual final page content someone
# expected to see.

curl -L https://example.com/old-page
# NOW curl follows the Location header (potentially through MULTIPLE
# chained redirects) to the final destination and shows THAT response instead
```

---

## -k Disables Certificate Verification Entirely — Not Just for "This One Warning"

### A security-relevant flag often reached for too casually
```bash
curl -k https://internal-service.example.com
# ⚠️ -k/--insecure doesn't selectively ignore one specific certificate
# problem (an expired cert, a hostname mismatch) — it disables TLS
# certificate verification COMPLETELY for that request, meaning curl
# will accept ANY certificate presented, including one from an
# attacker actively intercepting the connection. This defeats the
# entire point of TLS's identity verification for that request.

# For a genuinely self-signed internal cert, the better fix is trusting
# that SPECIFIC certificate/CA explicitly, rather than disabling
# verification globally for the request:
curl --cacert /path/to/internal-ca.pem https://internal-service.example.com
```

---

## Unquoted URLs with Special Characters Get Mangled by the Shell, Not curl

### The shell interprets the URL before curl ever sees it
```bash
curl https://api.example.com/search?q=cats&sort=asc
# [1] 12345     ← ⚠️ the shell interprets the UNQUOTED & as a
# background-job separator, not part of the URL — curl only ever
# receives "https://api.example.com/search?q=cats" as its argument,
# silently dropping "&sort=asc" entirely, and a completely separate
# background job "sort=asc" gets launched (and immediately fails, since
# that's not a real command).

# Always quote URLs containing query strings, especially with & or
# other shell-meaningful characters:
curl "https://api.example.com/search?q=cats&sort=asc"
```

---

## -O Fails Silently on URLs Without an Obvious Filename

### Not every URL maps cleanly to a local filename
```bash
curl -O https://api.example.com/download
# curl: Remote file name has no length!
# ⚠️ -O derives the local filename from the URL's OWN path — a URL
# ending in a trailing slash, or with no clear filename component at
# all (common for API endpoints rather than direct file links), gives
# curl nothing sensible to name the saved file, and it errors out
# rather than guessing.

# Specify an explicit output filename instead when the URL doesn't
# have an obvious one:
curl -o downloaded_file.zip https://api.example.com/download
```

---

## Piping curl Directly into bash Runs Whatever the Server Sends, No Review Possible

### A widely used but risk-carrying installation pattern
```bash
curl -fsSL https://example.com/install.sh | bash
# ⚠️ This executes whatever script content the server returns AT THE
# EXACT MOMENT of the request, with no opportunity to review it first
# — if the server is compromised, the connection is intercepted
# (mitigated but not fully eliminated by HTTPS depending on the full
# trust chain), or the script's content simply changes unexpectedly
# between when someone reviewed it and when it's actually run, there's
# no built-in safeguard catching a difference.

# A safer pattern for anything beyond a fully trusted, well-known source:
curl -fsSL https://example.com/install.sh -o install.sh
less install.sh        # actually review it first
bash install.sh
```

---

## A Redirect Can Silently Change the Request Method

### Not every redirect preserves your original method and data
```bash
curl -L -X POST -d "key=value" https://example.com/endpoint
# ⚠️ Depending on the specific HTTP status code involved in the
# redirect (a 301/302 historically causes some clients to switch a
# POST into a GET on the follow-up request, dropping the original
# body entirely; a 307/308 is specifically defined to PRESERVE the
# original method and body), the actual behavior after following a
# redirect chain can differ from what was originally sent — silently
# losing POST data partway through, if the server issuing the
# redirect used an older-style redirect code.

# Use -v to actually confirm what happened across the full redirect
# chain rather than assuming the original request was preserved:
curl -v -L -X POST -d "key=value" https://example.com/endpoint
```

---

## -s Also Suppresses Error Messages, Not Just the Progress Meter

### A quiet script can also become an UNHELPFULLY quiet script
```bash
curl -s https://unreachable-host.example.invalid
# (nothing printed at all — no progress meter, but ALSO no error
# message explaining WHY nothing happened)
echo $?
# 6     ← a real connection failure occurred, but -s suppressed the
# human-readable explanation entirely, leaving just a numeric exit code

# Combine with -S to keep error messages visible even in silent mode:
curl -sS https://unreachable-host.example.invalid
# curl: (6) Could not resolve host: unreachable-host.example.invalid
```

---

## Cookies Aren't Persisted Between Separate curl Invocations by Default

### Each curl call starts with a completely fresh, empty cookie jar unless told otherwise
```bash
curl -c cookies.txt https://example.com/login -d "user=alice&pass=secret"
curl https://example.com/dashboard
# ⚠️ This SECOND curl call has NO memory of the first one's session —
# each separate invocation of curl is its own independent process with
# no shared state, so any session cookie set by the login request is
# completely lost unless explicitly saved and re-sent.

# Explicitly save cookies with -c, then send them back with -b on
# every SUBSEQUENT request that needs the same session:
curl -c cookies.txt -d "user=alice&pass=secret" https://example.com/login
curl -b cookies.txt https://example.com/dashboard
```

---

## A Successful Connection Doesn't Mean a Successful Transfer If It's Interrupted Midway

### A partially downloaded file can look complete at a glance
```bash
curl -o largefile.iso https://example.com/largefile.iso
# ... network drops partway through ...
# ⚠️ Depending on exactly how/when the interruption happened, curl's
# exit code and the resulting partial file may not make it obvious
# that the download is INCOMPLETE — a truncated file sitting at the
# expected path can be mistaken for a successful, complete download
# without an explicit integrity check.

# Verify size/checksum explicitly rather than assuming file presence
# alone means success, especially for large or critical downloads:
curl -o largefile.iso https://example.com/largefile.iso
sha256sum largefile.iso
# compare against a known-good checksum published by the source
```

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`interview-questions.md`](interview-questions.md)
