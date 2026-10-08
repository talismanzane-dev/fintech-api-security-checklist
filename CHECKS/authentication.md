# Authentication and Session Controls

## What it is

Authentication establishes who is calling the API; session controls keep that identity bound across requests and prevent it from being hijacked, guessed, or abused. Weaknesses here undermine every downstream authorization check, since authorization can only be as strong as the authentication it relies on.

## How it happens

Flaws arise in account creation, password reset flows, OTP handling, and session token management — anywhere the application lets an attacker take over an identity or reuse a credential.

## A realistic example

A password reset endpoint that does not rate-limit OTP entry allows an attacker to brute-force a six-digit code. Because the endpoint never locks after repeated failures, the attacker enumerates the full space until the correct code is accepted and resets the target account.

## How to detect it

1. Test whether OTP or reset endpoints enforce rate limits, attempt caps, and expiry.
2. Test whether reset links are single-use and expire promptly.
3. Verify session tokens are invalidated on logout and password change.
4. Check for predictable or reusable session identifiers.
5. Confirm OAuth scopes are validated server-side and not just trusted from the client.

## How to fix it

- Rate-limit and lock OTP and reset attempts; expire codes quickly.
- Bind reset tokens to the exact account and ensure single-use semantics.
- Invalidate sessions and tokens on password change, logout, and suspicious activity.
- Use cryptographically secure random session identifiers and store only server-side references.
- Validate OAuth scopes on every protected route, independent of what the client claims.
