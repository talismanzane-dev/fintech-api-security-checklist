# Broken Object Level Authorization (BOLA)

| | |
| --- | --- |
| **OWASP API Security Top 10** | API1:2023 — Broken Object Level Authorization |
| **CWE** | CWE-639 (Authorization Bypass Through User-Controlled Key), CWE-285 (Improper Authorization) |
| **Typical severity** | High to Critical — direct exposure of other customers' accounts, balances and transactions |
| **Applies to** | Any endpoint that takes an object identifier from the client: path parameters, query strings, request bodies, headers |

---

## What the flaw is

BOLA occurs when an API accepts an identifier for a resource — an account, a card, a transfer, a loan application — and returns or modifies that resource **without checking that the authenticated caller is allowed to access it**.

Authentication answers *"who is calling?"*. Object-level authorization answers *"is this caller allowed to touch this specific object?"*. BOLA is what happens when the second question is never asked, or is asked only on some code paths.

It is the most common and most damaging API vulnerability class because:

- Every API that exposes resources by ID is a candidate.
- The bug is invisible in functional testing: the endpoint works perfectly for the legitimate owner.
- It is usually introduced by omission (a missing check), not by an obviously wrong line of code.
- Mobile and single-page apps hide object IDs from the UI, giving a false sense that clients "can't" request other IDs. Any client can.

In fintech, the objects are money-bearing: account balances, statements, beneficiaries, card details, payment instructions and KYC documents.

## Realistic example

A retail banking API exposes account details to its mobile app:

```http
GET /api/v1/accounts/40817810099910004312 HTTP/1.1
Authorization: Bearer <token for customer A>
```

The handler authenticates the token, then loads the account straight from the path parameter:

```python
# VULNERABLE
@app.get("/api/v1/accounts/{account_id}")
def get_account(account_id: str, user: User = Depends(authenticated_user)):
    account = db.accounts.get(account_id)          # no ownership check
    if account is None:
        raise HTTPException(404)
    return AccountView.from_model(account)
```

Customer A only ever sees their own account number in the app, but the API returns **any** account whose number is supplied. Because account numbers follow a predictable structure (branch code + sequence + check digit), a caller can enumerate them.

The same pattern appears in write operations, where the impact is worse:

```http
POST /api/v1/transfers HTTP/1.1
Authorization: Bearer <token for customer A>
Content-Type: application/json

{ "source_account_id": "<an account owned by customer B>", "destination": "...", "amount": "250.00" }
```

If the transfer service validates the amount and the destination but trusts `source_account_id`, customer A can move customer B's money.

Other common fintech variants:

- `GET /statements/{statement_id}/pdf` — statement documents served by ID.
- `DELETE /beneficiaries/{id}` — removing another customer's saved payee.
- `PATCH /cards/{card_id}` with `{"status": "unblocked"}` — changing another customer's card state.
- `GET /loan-applications/{id}/documents` — exposure of identity documents and income proofs.
- Batch endpoints such as `POST /transactions/export` with a list of account IDs where only the first ID is checked.

## How to detect it

> Test only systems you own or are explicitly authorized to assess, in a non-production environment where possible, using test accounts you control.

### Manual testing with two principals

The core technique is to compare what two legitimate users can reach:

1. Create two test customers, **User A** and **User B**, each with their own accounts, cards, beneficiaries and transactions.
2. Log in as User B and record the identifiers of B's objects (from responses, not from the UI).
3. Log in as User A and replay every request that references an object, substituting B's identifiers.
4. Expected result: `403 Forbidden` or `404 Not Found` for every substituted request. Any `2xx` response, or any state change on B's objects, is a finding.

Cover every place an identifier can travel:

| Location | Example |
| --- | --- |
| Path | `/accounts/{id}` |
| Query string | `/transactions?account_id=...` |
| JSON body | `{"source_account_id": "..."}` |
| Nested objects | `{"payment": {"debtor": {"account_id": "..."}}}` |
| Arrays | `{"account_ids": ["mine", "theirs"]}` |
| Headers | `X-Account-Id`, `X-Customer-Id` |
| GraphQL arguments | `account(id: "...")`, including nested resolvers |

Also test:

- **Every HTTP method** on the same resource. `GET` may be protected while `PUT`, `PATCH` or `DELETE` are not.
- **Older API versions** (`/v1/` vs `/v2/`) and internal or partner endpoints that share the same data layer.
- **Indirect references**: a transaction ID that belongs to B, used in a dispute or refund endpoint by A.
- **Mixed batches**: one owned ID followed by one foreign ID in the same request.

### Automated testing

- Add **authorization regression tests** to CI that run each object-bearing endpoint as a non-owner and assert a denial. These are cheap and catch regressions permanently.
- Use proxy tooling with two sessions (for example Burp Suite with the Autorize extension, or OWASP ZAP with an access-control context) to automatically replay traffic from one user with the other user's session.
- Generate a matrix from the OpenAPI specification: every operation × every parameter that looks like an identifier × {owner, other customer, unauthenticated}.

### Code review signals

- A repository or ORM call that loads by ID alone: `get(id)`, `findById(id)`, `SELECT ... WHERE id = ?` with no tenant or owner predicate.
- Authorization performed in the client, the API gateway, or a UI layer only.
- Ownership checked in one handler but not in sibling handlers for the same resource.
- Identifiers read from the request body when the authenticated principal already determines the correct value (for example `customer_id` in the body).

### Production monitoring

- Alert on a single principal receiving many `403`/`404` responses for distinct object IDs in a short window — a strong enumeration signal.
- Log the authenticated principal and the owner of every accessed object; a mismatch on a successful response should never happen and is worth an alert.

## How to fix it

### 1. Enforce ownership on the server for every object access

Scope every data access to the authenticated principal. The safest pattern is to make the owner part of the query itself, so a foreign object is simply not found:

```python
# FIXED — ownership is part of the lookup
@app.get("/api/v1/accounts/{account_id}")
def get_account(account_id: str, user: User = Depends(authenticated_user)):
    account = db.accounts.get_for_owner(account_id=account_id, owner_id=user.customer_id)
    if account is None:
        raise HTTPException(404)   # same response whether it doesn't exist or isn't yours
    return AccountView.from_model(account)
```

```sql
SELECT * FROM accounts
WHERE id = :account_id
  AND customer_id = :authenticated_customer_id;
```

Return the same status code for "does not exist" and "not yours" so the API cannot be used to confirm which identifiers are valid.

### 2. Centralize the check

Don't rely on every developer remembering a check in every handler. Put authorization in one place:

- A **policy layer** (`can(user, action, resource)`) called by a shared resource loader, or a policy engine such as OPA/Cedar.
- **Repository methods that require a principal**, so there is no unscoped `get(id)` available to handler code.
- **Row-level security** in the database (for example PostgreSQL RLS keyed on a session variable) as defense in depth.

```python
class AccountRepository:
    def get(self, account_id: str, *, principal: Principal) -> Account | None:
        """There is deliberately no method that loads an account without a principal."""
        return self._session.query(Account).filter_by(
            id=account_id, customer_id=principal.customer_id
        ).one_or_none()
```

### 3. Derive identifiers from the session where possible

If the caller can only ever act on their own data, don't accept the identifier at all:

- `GET /me/accounts` instead of `GET /customers/{customer_id}/accounts`.
- Take `customer_id` from the verified token, never from the request body.

### 4. Authorize every object in compound requests

For transfers, verify the **source** account belongs to the caller, and verify any referenced beneficiary belongs to the caller's saved payees. For batches, check **each** ID and reject the whole request if any fails.

### 5. Handle delegated and business access explicitly

Joint accounts, business users with roles, power-of-attorney and third-party providers (open banking) all need explicit, tested relationships: `user → role → account → permitted actions`. Model them in the policy layer rather than special-casing handlers.

### 6. Make identifiers non-enumerable — as defense in depth only

Random identifiers (UUIDv4, or opaque tokens) slow enumeration but **do not fix BOLA**. Identifiers leak through logs, URLs, referrers, shared statements and support tickets. See [`idor.md`](idor.md).

### 7. Prevent regressions

- Write a denial test for every new object-bearing endpoint as part of the definition of done.
- Include an authorization review item in pull-request templates.
- Re-run the two-user test matrix before every major release.

## Checklist

- [ ] Every endpoint that accepts an object identifier verifies, server-side, that the caller may access that object.
- [ ] Ownership is enforced in the data-access query or a central policy layer, not ad hoc in each handler.
- [ ] All HTTP methods and all API versions for a resource are covered.
- [ ] Identifiers in bodies, nested objects, arrays, headers and GraphQL arguments are authorized, not only path parameters.
- [ ] Transfer and payment endpoints authorize the source account and every referenced beneficiary.
- [ ] Batch endpoints authorize every element.
- [ ] "Not found" and "not authorized" return the same response.
- [ ] Customer and account identifiers that the session already determines are taken from the session, not the request.
- [ ] Automated non-owner denial tests exist for every object-bearing endpoint and run in CI.
- [ ] Monitoring alerts on enumeration patterns and on owner/principal mismatches.

## References

- [OWASP API Security Top 10 (2023) — API1: Broken Object Level Authorization](https://owasp.org/API-Security/editions/2023/en/0xa1-broken-object-level-authorization/)
- [OWASP Authorization Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html)
- [OWASP Web Security Testing Guide — Testing for Insecure Direct Object References](https://owasp.org/www-project-web-security-testing-guide/)
- [CWE-639: Authorization Bypass Through User-Controlled Key](https://cwe.mitre.org/data/definitions/639.html)
- Related checks: [`idor.md`](idor.md), [`authentication.md`](authentication.md)
