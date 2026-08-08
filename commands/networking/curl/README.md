# curl — The Complete Reference

> **Transfer data to or from a server, over almost any protocol**
> The universal command-line HTTP client, API testing tool, and file
> downloader — present on nearly every system, script, and CI pipeline.

---

## Table of Contents

- [What is curl?](#what-is-curl)
- [Where does curl live?](#where-does-curl-live)
- [How curl works internally](#how-curl-works-internally)
- [Syntax](#syntax)
- [Understanding the Output](#understanding-the-output)
- [Making Requests — GET, POST, and Beyond](#making-requests--get-post-and-beyond)
- [Headers, Data, and Authentication](#headers-data-and-authentication)
- [All Key Options](#all-key-options)
- [Following Redirects and Handling Errors](#following-redirects-and-handling-errors)
- [curl vs wget vs httpie](#curl-vs-wget-vs-httpie)
- [Related Commands](#related-commands)

---

## What is curl?

`curl` ("Client URL") transfers data to or from a server using a URL — most commonly HTTP/HTTPS, but also FTP, SFTP, SCP, and several other protocols through the same unified interface. It's used interchangeably as a file downloader, an API testing tool, and a general scripting building block for anything involving a network request.

```bash
curl https://example.com
# <!doctype html>
# <html>...
```

**Why curl is so central to scripting and automation:** it's scriptable, composable with other Unix tools via pipes, and available virtually everywhere — making it the default way to test an API endpoint, download a file inside a Dockerfile, or check whether a service is responding, all without needing a browser or a dedicated HTTP client library.

---

## Where does curl live?

```
/usr/bin/curl
```

```bash
which curl
curl --version
# curl 8.5.0 (x86_64-pc-linux-gnu) libcurl/8.5.0 OpenSSL/3.0.13 zlib/1.3
# Release-Date: 2023-12-06
# Protocols: dict file ftp ftps gopher gophers http https imap ...
```

Built on **libcurl**, the underlying transfer library that many other programs and programming-language HTTP libraries also use internally. Present by default on virtually every Linux distribution, macOS, and (since Windows 10) modern Windows as well.

---

## How curl works internally

For an HTTP(S) request, `curl` broadly:

1. Resolves the hostname in the URL via DNS.
2. Opens a TCP connection to the resolved address (and, for HTTPS, performs a TLS handshake — negotiating encryption and verifying the server's certificate).
3. Sends an HTTP request line, headers, and (if applicable) a request body.
4. Reads the response status line, headers, and body back from the server.
5. By default, prints the response **body** to stdout; headers and connection details are hidden unless explicitly requested.

```bash
# See the full exchange — request AND response headers — with -v
curl -v https://example.com
# > GET / HTTP/1.1
# > Host: example.com
# > User-Agent: curl/8.5.0
# > Accept: */*
# >
# < HTTP/1.1 200 OK
# < Content-Type: text/html
# ...
```

Being built on libcurl means curl's protocol support, TLS behavior, and connection handling are shared with a huge ecosystem of other software — a TLS/certificate quirk discovered in curl often reflects the exact same underlying library behavior anywhere else libcurl is embedded.

---

## Syntax

```bash
curl [OPTIONS] URL
```

With no options, `curl` performs a `GET` request and prints the response body to stdout — nothing is saved to disk unless explicitly told to.

---

## Understanding the Output

```bash
curl https://example.com
```

By default, only the **response body** is printed — no progress meter (when output is redirected/piped), no headers, no status code shown explicitly. This is deliberate: curl is designed to be pipeline-friendly, so its default stdout output is exactly the data a script would want to consume, without extra noise mixed in.

```bash
# Interactively (terminal, not piped), curl shows a progress meter on stderr:
curl -o file.zip https://example.com/file.zip
# ######################################################## 100.0%

# Suppress the progress meter explicitly
curl -s -o file.zip https://example.com/file.zip
```

---

## Making Requests — GET, POST, and Beyond

```bash
# GET (the default)
curl https://api.example.com/users

# POST with a form-encoded body
curl -X POST -d "name=alice&role=admin" https://api.example.com/users

# POST with a JSON body
curl -X POST -H "Content-Type: application/json" \
  -d '{"name": "alice", "role": "admin"}' \
  https://api.example.com/users

# PUT, DELETE, and any other method
curl -X PUT -d '{"role": "member"}' https://api.example.com/users/123
curl -X DELETE https://api.example.com/users/123
```

Note: `-d`/`--data` **implicitly switches the request method to POST** unless `-X` explicitly overrides it — this is a common source of confusion covered further in [edge-cases.md](edge-cases.md).

---

## Headers, Data, and Authentication

```bash
# Add a custom header
curl -H "Authorization: Bearer TOKEN_HERE" https://api.example.com/data

# Add multiple headers
curl -H "Accept: application/json" -H "X-Custom: value" https://api.example.com

# Basic authentication
curl -u username:password https://api.example.com/secure

# Send data from a file instead of inline
curl -X POST -d @payload.json -H "Content-Type: application/json" https://api.example.com/users

# Upload a file as multipart form data
curl -F "file=@photo.jpg" -F "description=vacation photo" https://api.example.com/upload
```

---

## All Key Options

| Option | Long form | Description |
|---|---|---|
| `-X METHOD` | `--request` | Explicitly set the HTTP method |
| `-H HEADER` | `--header` | Add a request header (repeatable) |
| `-d DATA` | `--data` | Send data in the request body (implies POST) |
| `-F` | `--form` | Send multipart/form-data (for file uploads) |
| `-o FILE` | `--output` | Write the response body to a file instead of stdout |
| `-O` | `--remote-name` | Save using the URL's own filename |
| `-i` | `--include` | Include response headers in the output |
| `-I` | `--head` | Send a HEAD request (headers only, no body) |
| `-s` | `--silent` | Suppress the progress meter and error messages |
| `-S` | `--show-error` | Show errors even with `-s` |
| `-v` | `--verbose` | Show the full request/response exchange |
| `-L` | `--location` | Follow HTTP redirects |
| `-u USER:PASS` | `--user` | HTTP Basic authentication |
| `-k` | `--insecure` | Skip TLS certificate verification (use with caution) |
| `-w FORMAT` | `--write-out` | Print custom info after the transfer (status code, timing, etc.) |
| `--fail` (`-f`) | `--fail` | Return a non-zero exit code on HTTP error responses (4xx/5xx) |
| `-c FILE` / `-b FILE` | `--cookie-jar` / `--cookie` | Save/send cookies |
| `-A STRING` | `--user-agent` | Set a custom User-Agent header |
| `--max-time N` | | Abort the transfer after N seconds total |

---

## Following Redirects and Handling Errors

```bash
curl https://example.com/old-page
# By default, curl does NOT follow a 3xx redirect — it just prints the
# redirect response itself (a short HTML body pointing elsewhere)

curl -L https://example.com/old-page
# -L makes curl follow the redirect chain automatically to its final destination

# curl's exit code is 0 even for a 404 or 500 response BY DEFAULT —
# as far as curl is concerned, the HTTP transaction itself succeeded,
# even though the server reported an application-level error
curl https://example.com/does-not-exist
echo $?
# 0     ← the request/response exchange completed fine; curl doesn't
#          treat a 404 as a "failure" unless told to

curl --fail https://example.com/does-not-exist
echo $?
# 22    ← --fail makes curl return a non-zero exit code for 4xx/5xx
#          responses specifically, which is essential for scripts that
#          need to detect HTTP-level failures, not just connection failures
```

---

## curl vs wget vs httpie

| Tool | Best for | Key difference from curl |
|---|---|---|
| `curl` | API testing, scripting, broad protocol support | The most flexible and widely available; terser, more cryptic flag syntax |
| `wget` | Bulk/recursive downloading, mirroring a site | Simpler defaults for straightforward downloads; built-in recursive fetching curl lacks natively |
| `httpie` | Human-friendly interactive API exploration | Colorized, more readable output and simpler syntax by default; not always preinstalled, less universally available than curl |

```bash
curl -O https://example.com/file.zip     # download, keep original filename
wget https://example.com/file.zip         # same result, wget's default behavior
http GET https://api.example.com/users    # httpie, more readable by default
```

---

## Related Commands

| Command | Relation |
|---|---|
| `wget` | Alternative downloader, stronger at recursive/bulk fetching |
| `httpie` | Friendlier, more human-readable alternative for interactive API work |
| `jq` | Commonly piped after curl to parse/filter JSON API responses |
| `openssl s_client` | Lower-level TLS/certificate inspection, useful when curl reports a cert error |
| `nc` (netcat) | Raw socket-level testing, useful for non-HTTP debugging curl can't do |
| `ping` / `traceroute` | Network-level diagnostics, useful when curl fails to even connect |

---

> See also: [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
