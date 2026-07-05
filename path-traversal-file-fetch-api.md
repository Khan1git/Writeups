# Path Traversal in File Fetch API Leading to Sensitive File Disclosure

**Severity:** High
**CWE:** CWE-22 (Improper Limitation of a Pathname to a Restricted Directory)
**CVSS 3.1 (estimated):** 7.5 (`AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N`)
**Status:** Fixed
**Type:** Web Application / API Security

> Note: Endpoint names, parameters, and identifiers below have been generalized/redacted from the original engagement to respect client confidentiality. The vulnerability class, root cause, and exploitation logic are reported accurately.

---

## Summary

While assessing a "data export via email" feature during a penetration testing engagement, I identified an internal file-fetch API used to retrieve user documents for PDF generation. The API accepted a user-controlled `url` parameter intended to reference files in a remote document storage system.

When the referenced file wasn't found remotely, the application silently fell back to reading from the **local filesystem** using the same unsanitized parameter. This fallback path performed no normalization or boundary checks, allowing an attacker to supply directory traversal sequences and read arbitrary files from the server's local disk — including system files, application configuration, and potentially secrets such as API keys and credentials.

---

## Affected Endpoint

```
GET /api/get-user/{id}/file?url=<file_reference>
```

This endpoint is not part of the primary user-facing UI — it's an internal API invoked as part of the export-generation workflow (export request → document fetch → PDF assembly → email delivery).

---

## Root Cause

The vulnerability stemmed from a combination of weaknesses rather than a single missing check:

1. **User-controlled input reached filesystem operations** — the `url` parameter was passed to a file read function with no validation.
2. **Unsafe fallback behavior** — when the remote storage lookup failed (e.g., file not found), the application fell back to resolving the same value as a local filesystem path.
3. **No path normalization or sanitization** — traversal sequences (`../`, `..\`) were not stripped, decoded, or rejected.
4. **No directory boundary enforcement** — there was no check that the resolved path stayed within an intended root directory.
5. **No allowlist for valid file references** — any string was accepted as a potential file identifier.

The fallback is what made this exploitable: an endpoint that looked like it only spoke to remote storage actually had a silent, unauthenticated-in-effect path to the local disk.

---

## Exploitation / Proof of Concept

**1.** Trigger the export feature normally through the application UI, and intercept the resulting request to the file-fetch API using a proxy (e.g., Burp Suite).

**2.** Identify the vulnerable request:

```http
GET /api/get-user/1234/file?url=documents/report.pdf HTTP/1.1
Host: target.example.com
Cookie: session=<valid_session>
```

**3.** Replace the `url` value with a traversal payload targeting a known system file:

```http
GET /api/get-user/1234/file?url=../../../../etc/passwd HTTP/1.1
Host: target.example.com
Cookie: session=<valid_session>
```

**4.** Forward the request and observe the response body contains the contents of `/etc/passwd`, confirming arbitrary local file read.

**5.** The same technique was used to confirm access to application configuration files and `.env`-style files containing environment variables — demonstrating potential exposure of credentials and API keys, though exact secret values were not exfiltrated during testing (redacted from this write-up regardless).

---

## Impact

An attacker able to reach this endpoint (which required only a valid session, not elevated privileges) could:

- Read arbitrary files accessible to the application's OS-level user
- Retrieve application configuration and environment files, potentially exposing:
  - Database credentials
  - Third-party API keys
  - Internal service URLs / secrets
- Use disclosed secrets as a foothold for further compromise (e.g., pivoting into connected databases or internal services)

This is classified as **High** severity due to the low complexity of exploitation, lack of required privileges beyond a standard session, and the potential for credential exposure leading to broader compromise.

---

## Recommendation

- **Remove the local filesystem fallback** for any user-controlled file reference. If remote lookup fails, return a generic "not found" response — do not fall back to a different, less-restricted resolution path.
- **Validate against an allowlist** of known file identifiers or storage keys rather than accepting arbitrary path-like strings.
- **Normalize and canonicalize paths** using secure, language-standard path handling functions, and verify the resolved absolute path is a descendant of the intended root directory before any file access.
- **Reject traversal patterns outright** (`../`, `..\`, URL-encoded and double-encoded variants) as a defense-in-depth measure, in addition to (not instead of) proper canonicalization.
- **Run the file-serving process with least privilege**, so that even a successful traversal has minimal blast radius.

---

## Timeline

| Date | Action |
|---|---|
| Day 0 | Vulnerability identified during engagement |
| Day 0 | Reported to client/vendor |
| Day X | Fix confirmed — local filesystem fallback removed, allowlist validation implemented |

---

## Disclaimer

This report describes a vulnerability identified and disclosed responsibly during an authorized security assessment. Identifying details have been generalized to protect the confidentiality of the client. This write-up is shared for educational and portfolio purposes only.
