# Race Conditions in Financial APIs

## What it is

A race condition is a flaw where the correctness of an operation depends on the timing or sequence of other events. In financial systems, this most often manifests when two requests read the same balance or state before either commits a change, allowing both to act on the stale value.

## How it happens

The classic pattern is check-then-act without an atomic boundary. The server reads an account balance, verifies it is sufficient, then performs the mutation — but if two requests interleave between the check and the act, both can pass the check and both can mutate state.

## A realistic example

A withdrawal endpoint:

1. Reads balance (R1,000)
2. Checks R1,000 >= withdrawal amount (R1,000)
3. Deducts and pays out

Two parallel withdrawal requests of R1,000 each can both read R1,000 before either deducts. Both pass the check. The account pays out R2,000 it did not have.

## How to detect it

1. Identify endpoints that mutate balances, limits, or state.
2. Send the same request in parallel — many HTTP clients and proxies support concurrent sessions or single-click parallel send.
3. Observe whether the outcome reflects the duplicate action (double payout, double spend, bypassed limit).
4. Also test sequential-but-rapid requests and request replays.

## How to fix it

- Use idempotency keys: require a client-generated key, and ensure a given key is processed exactly once.
- Perform the check and the mutation as a single atomic operation — for example, a conditional database update rather than a separate read then write.
- Apply appropriate locking or serialization for operations on the same resource.
- Enforce unique constraints on transaction records to prevent duplicate processing at the persistence layer.
