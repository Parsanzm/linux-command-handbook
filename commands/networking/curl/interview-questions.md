# curl — Interview Questions

> Covers beginner to senior level. Each question includes the answer.
> Questions marked 🔥 are commonly asked in real interviews.

---

## Table of Contents

- [Conceptual](#conceptual)
- [Requests and Methods](#requests-and-methods)
- [Exit Codes and Error Handling](#exit-codes-and-error-handling)
- [Internals](#internals)
- [curl vs Other Tools](#curl-vs-other-tools)
- [Scenario-Based](#scenario-based)

---

## Conceptual

**Q1. What does curl do?**
> Transfers data to or from a server specified by a URL, most commonly over HTTP/HTTPS but also FTP, SFTP, and several other protocols — used interchangeably for downloading files, testing APIs, and general network scripting.

---

**Q2 🔥 What library is curl built on, and why does that matter?**
> `libcurl` — the same underlying transfer library many other programs and programming-language HTTP clients also use internally. It matters because a protocol quirk or TLS behavior discovered in curl often reflects the same underlying library behavior anywhere else libcurl is embedded.

---

**Q3. What does curl print by default when run with no special flags?**
> Just the response body, to stdout — no headers, no status code, no extra formatting. This is deliberate, making curl's default output directly usable in a pipeline without needing to strip out extra noise first.

---

## Requests and Methods

**Q4 🔥 What HTTP method does curl use by default, and how do you change it?**
> GET by default. `-X METHOD` (e.g., `-X POST`, `-X DELETE`) explicitly sets a different method.

---

**Q5. What does the -d flag do, and what's a common mistake related to it?**
> `-d`/`--data` sends data in the request body. The common mistake: it implicitly switches the request method to POST unless `-X` explicitly overrides it — someone intending a GET request with query-string-like data can accidentally send a POST instead without realizing it.

---

**Q6 🔥 How would you send a GET request with query parameters using -d's data instead of a POST?**
> Add `-G` alongside `-d` — this tells curl to append the data as URL query parameters rather than putting it in a POST body, effectively converting what would otherwise become a POST back into a GET with the data attached to the URL.

---

## Exit Codes and Error Handling

**Q7 🔥 Does curl return a non-zero exit code for a 404 or 500 HTTP response by default?**
> No — curl considers the HTTP transaction itself successful as long as it made a connection and received a complete response, regardless of the status code the server returned. A 404 or 500 response still results in exit code 0 by default.

---

**Q8. How do you make curl treat HTTP error responses (4xx/5xx) as command failures?**
> The `-f`/`--fail` flag — with it, curl returns a non-zero exit code when the server responds with an HTTP error status, which is essential for scripts that need to detect application-level failures, not just connection-level ones.

---

**Q9 🔥 What's the difference between -s and -sS?**
> `-s` (silent) suppresses both the progress meter and error messages entirely. `-sS` keeps silent mode's lack of a progress meter but restores visibility of actual error messages — useful for scripts that want quiet successful output but still need to see what went wrong on failure.

---

## Internals

**Q10. Does curl follow HTTP redirects by default?**
> No — without `-L`/`--location`, curl simply prints whatever short response body accompanies a 3xx redirect status, rather than automatically fetching the URL in the redirect's `Location` header. `-L` must be explicitly added to follow redirects (potentially through a chain of several).

---

**Q11 🔥 What does -k/--insecure actually disable, and why is it a meaningful security consideration?**
> It disables TLS certificate verification completely for that request — curl will accept any certificate presented, including one from an actively intercepting attacker. It doesn't selectively ignore one specific certificate problem; it removes the entire verification step, defeating a core purpose of TLS for that connection.

---

**Q12. How would you see the full request and response headers exchanged during a curl request?**
> `-v`/`--verbose` shows the complete request line, request headers, response status line, and response headers as they're sent/received — useful for debugging exactly what's being sent or why a request isn't behaving as expected.

---

## curl vs Other Tools

**Q13 🔥 When might you reach for wget instead of curl?**
> For bulk or recursive downloading — mirroring an entire site or directory structure — which wget supports natively with simpler defaults, whereas curl is more focused on single-URL transfers and generally requires more manual scripting to replicate recursive fetching.

---

**Q14. What's a common tool paired with curl when working with JSON APIs, and why?**
> `jq` — curl retrieves the raw JSON response, and `jq` parses, filters, and pretty-prints it, since curl itself has no built-in JSON parsing or formatting capability.

---

## Scenario-Based

**Q15 🔥 A monitoring script checks a service's health using `curl https://api.example.com/health` and treats any exit code 0 as "healthy" — but the service has been silently returning 500 errors for hours. What's wrong with the script's check?**
> curl's exit code reflects whether the HTTP transaction itself completed, not whether the server's response indicated success — a 500 response still results in exit code 0 by default. The fix is adding `-f`/`--fail` (so curl itself returns non-zero on HTTP errors) or explicitly checking the returned status code via `-w "%{http_code}"` rather than relying on curl's exit code alone.

---

**Q16. A teammate runs `curl https://api.example.com/search?q=cats&sort=asc` from the command line and gets a `[1] 12345` job-control message along with an incomplete request. What happened?**
> The unquoted `&` in the URL was interpreted by the shell as a background-job separator rather than part of the URL — curl only received `https://api.example.com/search?q=cats` as its argument, and a separate (invalid) background job `sort=asc` was launched. Quoting the full URL (`curl "https://api.example.com/search?q=cats&sort=asc"`) prevents the shell from splitting it.

---

**Q17 🔥 Someone runs `curl -L -X POST -d "key=value" https://example.com/endpoint` through a redirect and finds the POST data was lost on the other end. What's the likely explanation?**
> Depending on the specific HTTP redirect status code involved, some redirects (traditionally 301/302 in many client implementations) cause the follow-up request to switch from POST to GET, dropping the original body — while 307/308 redirects are specifically defined to preserve the original method and body. Running the same request with `-v` reveals exactly which method/status codes were involved across the redirect chain.

---

**Q18. A script logs into a site with `curl -d "user=alice&pass=secret" https://example.com/login`, then immediately makes a second curl call to an authenticated page — but gets an unauthorized response. Why?**
> Each curl invocation is an independent process with no shared state between separate calls — any session cookie set by the login response is lost unless explicitly captured and resent. The fix is saving cookies from the login request with `-c cookies.txt` and sending them on the follow-up request with `-b cookies.txt`.

---

**Q19 🔥 What's a security consideration when using the common `curl ... | bash` installation pattern, and what's a safer alternative?**
> The script executes exactly whatever content the server returns at the moment of the request, with no opportunity to review it beforehand — if the source is compromised or the script's content changes unexpectedly, there's no safeguard catching the difference before execution. A safer alternative downloads the script to a file first (`curl -o install.sh ...`), allows a manual review, and only then runs it separately.

---

**Q20. You need to fetch a URL whose path doesn't end in an obvious filename, and `curl -O` fails with "Remote file name has no length!" What's happening, and what's the fix?**
> `-O` derives the local filename directly from the URL's own path — a URL with no clear filename component (a trailing slash, or an API-style endpoint rather than a direct file link) gives curl nothing sensible to name the output file, so it errors instead of guessing. Using `-o explicit_filename.ext` instead specifies the output filename directly, sidestepping the issue entirely.

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md)
