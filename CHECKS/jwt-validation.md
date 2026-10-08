# JWT Validation

## What it is

JSON Web Tokens are a common stateless authentication mechanism. The token carries claims (identity, roles, expiry) signed by the server. Verifying that signature — and several other properties — is what makes the token trustworthy. Failing to validate any of them breaks the entire authentication model.

## How it happens

An implementation accepts a JWT and trusts its claims without fully verifying that the token was issued by the legitimate authority and remains valid. The most common failures are signature and algorithm handling errors.

## A realistic example

The algorithm confusion attack: an application configured to accept either RSA or HMAC signatures, where the HMAC verification key is the RSA public key (or a weak guessable secret). An attacker sets the `alg` field to `HS256` and signs the token with the RSA public key as the HMAC secret. Because the server uses the public key as the HMAC key, the forged token verifies.

## How to detect it

1. Decode a valid token and inspect the `alg` field and claims.
2. Test whether the server accepts a different `alg` than it signed with.
3. Test weak or well-known HMAC secrets.
4. Verify the token is rejected when expired, when the audience (`aud`) does not match, and when the signature is stripped or altered.

## How to fix it

- Restrict accepted algorithms to an explicit allow-list — for example, RS256 only, never mixing RSA and HMAC.
- Verify the signature using the correct key type for the stated algorithm.
- Validate `exp`, `iat`, `nbf`, `aud`, and `iss` on every request.
- Use a dedicated, sufficiently strong secret for HMAC, or the proper public/private key pair for asymmetric algorithms.
- Keep tokens short-lived and issue refresh tokens for renewal rather than long-lived access tokens.
