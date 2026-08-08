# curl — Practical Examples

> Real-world patterns for downloading files, testing APIs, and scripting
> against web services.

---

## Table of Contents

- [Basic Usage](#basic-usage)
- [Downloading Files](#downloading-files)
- [Testing APIs](#testing-apis)
- [Authentication](#authentication)
- [Debugging a Request](#debugging-a-request)
- [Combining curl with Other Tools](#combining-curl-with-other-tools)
- [Real-World Recipes](#real-world-recipes)

---

## Basic Usage

```bash
# Simple GET request, body printed to stdout
curl https://example.com

# Follow redirects to their final destination
curl -L https://example.com/old-page

# Show response headers along with the body
curl -i https://example.com

# Headers only, no body (HEAD request)
curl -I https://example.com
```

---

## Downloading Files

```bash
# Save to a specific filename
curl -o archive.zip https://example.com/download/archive.zip

# Save using the URL's own filename
curl -O https://example.com/download/archive.zip

# Download quietly, no progress meter
curl -s -O https://example.com/download/archive.zip

# Resume a partially downloaded file
curl -C - -O https://example.com/download/large-file.iso

# Download multiple files in one command
curl -O https://example.com/file1.zip -O https://example.com/file2.zip
```

---

## Testing APIs

```bash
# GET a JSON API endpoint, pretty-printed via jq
curl -s https://api.example.com/users | jq .

# POST JSON data
curl -X POST -H "Content-Type: application/json" \
  -d '{"name": "alice"}' \
  https://api.example.com/users

# Check just the HTTP status code
curl -s -o /dev/null -w "%{http_code}\n" https://api.example.com/health

# Send a PATCH request
curl -X PATCH -H "Content-Type: application/json" \
  -d '{"status": "active"}' \
  https://api.example.com/users/123

# Include timing information
curl -w "\nTime: %{time_total}s\n" -o /dev/null -s https://api.example.com/health
```

---

## Authentication

```bash
# HTTP Basic auth
curl -u username:password https://api.example.com/secure

# Bearer token
curl -H "Authorization: Bearer $TOKEN" https://api.example.com/data

# API key as a header
curl -H "X-API-Key: $API_KEY" https://api.example.com/data

# API key as a query parameter
curl "https://api.example.com/data?api_key=$API_KEY"
```

---

## Debugging a Request

```bash
# Full request/response trace
curl -v https://api.example.com/users

# Even more detail, including TLS handshake
curl -vv https://api.example.com/users

# Just the response headers, without the body
curl -sD - -o /dev/null https://api.example.com/users

# Test with a specific TLS version, for compatibility debugging
curl --tlsv1.2 https://api.example.com

# Skip certificate verification (only for known, trusted test environments)
curl -k https://self-signed-internal-service.local
```

---

## Combining curl with Other Tools

```bash
# Pretty-print a JSON response
curl -s https://api.example.com/users | jq .

# Extract just one field from a JSON response
curl -s https://api.example.com/users/1 | jq -r '.email'

# Check if a service is up, in a script
if curl -sf https://api.example.com/health > /dev/null; then
  echo "Service is healthy"
else
  echo "Service check failed"
fi

# Loop over a list of URLs, checking each one's status
for url in $(cat urls.txt); do
  code=$(curl -s -o /dev/null -w "%{http_code}" "$url")
  echo "$url -> $code"
done
```

---

## Real-World Recipes

```bash
# --- Simple Uptime/Health Check Script ---
STATUS=$(curl -s -o /dev/null -w "%{http_code}" https://myapp.example.com/health)
if [ "$STATUS" != "200" ]; then
  echo "ALERT: myapp health check returned $STATUS"
fi

# --- Download a File Inside a Dockerfile-Style Script ---
curl -fsSL https://example.com/install.sh | bash
# -f fail on HTTP errors, -s silent, -S show errors anyway, -L follow redirects

# --- Full API Workflow: Authenticate, Then Use the Token ---
TOKEN=$(curl -s -X POST -d "user=alice&pass=secret" https://api.example.com/login | jq -r '.token')
curl -H "Authorization: Bearer $TOKEN" https://api.example.com/profile

# --- Measure Response Time Breakdown for a Slow Endpoint ---
curl -o /dev/null -s -w "DNS: %{time_namelookup}s Connect: %{time_connect}s TTFB: %{time_starttransfer}s Total: %{time_total}s\n" \
  https://api.example.com/slow-endpoint

# --- Post a File's Contents as the Request Body ---
curl -X POST -H "Content-Type: application/json" -d @payload.json https://api.example.com/import

# --- Retry a Flaky Request a Few Times Before Giving Up ---
curl --retry 3 --retry-delay 2 -f https://api.example.com/data
```

---

> See also: [`README.md`](README.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
