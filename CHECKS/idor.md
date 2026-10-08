# Insecure Direct Object References (IDOR)

## What it is

IDOR is the broader class of vulnerability of which BOLA is the API-specific form. It occurs whenever an application exposes a direct reference to an internal object — a sequential ID, a filename, a database key — and the client can manipulate that reference to reach objects they should not access.

## How it happens

The application generates URLs or parameters containing predictable identifiers, such as incrementing integers, and trusts those identifiers as the sole access boundary. Because the identifier is predictable, an attacker can enumerate every object in the system.

## A realistic example

A loan application exposes documents at:

```
GET /document/download?file=statement_0042.pdf
```

Changing `0042` to `0043` downloads another customer's bank statement with no ownership verification.

## How to detect it

1. Log in and note every object reference the application exposes.
2. Modify identifiers by incrementing, decrementing, or substituting another user's values.
3. Check whether the application returns data or modifies state for the referenced object.
4. Test across all HTTP methods — IDOR affects reads (GET), writes (POST/PUT), and deletes (DELETE).

## How to fix it

- Enforce authorization on every direct reference, not just the most obvious ones. Access control must be applied consistently across the entire request lifecycle.
- Prefer indirect references stored server-side, such as a per-session mapping of opaque tokens to real objects.
- Use non-sequential identifiers where practical, while recognizing that this reduces but does not eliminate the risk.
- Log and alert on reference tampering to detect enumeration attempts early.
