# fintech-api-security-checklist

`fintech-api-security-checklist` is a practical reference for engineering and security teams building or auditing fintech APIs. It catalogs the authorization and validation flaws that most commonly expose financial systems — and, for each one, explains how to detect it and how to fix it. It is written for defenders: security engineers, penetration testers conducting authorized assessments, and developers who want to ship APIs that don't leak other people's money.

Fintech APIs are the fastest-growing attack surface in financial services. They process payments, transfers, loan disbursements, and customer data, and they are exposed to more third-party integrations than almost any other class of software. A single authorization flaw can turn a legitimate account into a path to someone else's balance. This project organizes those flaws into a checklist that teams can apply before and after they deploy.

The checklist covers:

- **Broken Object Level Authorization (BOLA)** — where an API trusts a client-supplied object ID without verifying that the requester owns it. Includes detection techniques and server-side ownership enforcement patterns.
- **Insecure Direct Object References (IDOR)** — the broader class of predictable object identifiers and how to make resource access non-enumerable.
- **Race conditions** — how parallel or repeated requests can double-spend balances, bypass withdrawal limits, or process the same transaction twice. Includes idempotency keys, atomic operations, and locking strategies.
- **JWT validation** — proper signature verification, algorithm confusion attacks (none/RSA-to-HMAC), expiration and audience checks, and secret management.
- **Authentication and session controls** — password reset flows, OTP handling, rate limiting, and OAuth scope validation.

Each item follows the same structure: what the flaw is, a realistic example, how to test for it, and the concrete fix. The goal is to be actionable — a checklist a team can run through in a Sprint review or a penetration test report, not a theoretical essay.

Everything here is public knowledge, aggregated from OWASP's API Security Top 10 and standard security reference material. Nothing in this repository teaches exploitation of a specific live system. It exists to help teams close gaps before attackers find them.

This project is maintained as a companion to [llm-safety-eval](https://github.com/talismanzane-dev/llm-safety-eval), representing the API-security side of the author's independent research.

---

## Table of contents

- [Checks](#checks)
  - [Broken Object Level Authorization (BOLA)](CHECKS/bola.md)
  - [Insecure Direct Object References (IDOR)](CHECKS/idor.md)
  - [Race conditions](CHECKS/race-conditions.md)
  - [JWT validation](CHECKS/jwt-validation.md)
  - [Authentication and session controls](CHECKS/authentication.md)
- [Templates](#templates)
  - [Penetration test report template](TEMPLATES/pentest-report-template.md)
- [Repository structure](#repository-structure)
- [How to use this checklist](#how-to-use-this-checklist)
- [Responsible use](#responsible-use)
- [Contributing](#contributing)
- [License](#license)

## Checks

| Check | OWASP API Top 10 (2023) | Status |
| --- | --- | --- |
| [Broken Object Level Authorization (BOLA)](CHECKS/bola.md) | API1 | Available |
| [Insecure Direct Object References (IDOR)](CHECKS/idor.md) | API1, API3 | Available |
| [Race conditions](CHECKS/race-conditions.md) | API6 | Available |
| [JWT validation](CHECKS/jwt-validation.md) | API2 | Available |
| [Authentication and session controls](CHECKS/authentication.md) | API2, API4, API5 | Available |

Every check file uses the same four sections, followed by a checklist and references:

1. **What the flaw is**
2. **Realistic example**
3. **How to detect it**
4. **How to fix it**

## Templates

| Template | Purpose |
| --- | --- |
| [`pentest-report-template.md`](TEMPLATES/pentest-report-template.md) | Structure for reporting findings from an authorized assessment, mapped to the checks in this repository |

## Repository structure

```
fintech-api-security-checklist/
├── README.md
├── LICENSE
├── .gitignore
├── CHECKS/
│   ├── authentication.md
│   ├── bola.md
│   ├── idor.md
│   ├── jwt-validation.md
│   └── race-conditions.md
└── TEMPLATES/
    └── pentest-report-template.md
```

Further check files are added to `CHECKS/` as they are completed.

## How to use this checklist

There is nothing to install: every file is plain Markdown and renders on GitHub or in any Markdown viewer. To keep a local copy:

```bash
git clone https://github.com/talismanzane-dev/fintech-api-security-checklist.git
```

Suggested ways to use it:

- **Design and code review.** Before an endpoint ships, walk through the **Checklist** section at the end of each relevant check and confirm every item.
- **Sprint review.** Pick the checks that match the endpoints changed in the sprint and record which items were verified.
- **Authorized assessments.** Use the checks to plan coverage and the [report template](TEMPLATES/pentest-report-template.md) to document findings, impact and remediation.
- **Regression prevention.** Turn checklist items into automated tests in your own CI so fixed issues stay fixed.

Each checklist item is written as a Markdown task (`- [ ]`), so you can copy a section into an issue or pull request and tick items off.

## Responsible use

- Assess only systems you own or have **written authorization** to test, within the agreed scope and rules of engagement.
- Prefer non-production environments and test accounts you control. Never access, modify or retain real customer data.
- Report vulnerabilities in third-party services through the vendor's security contact or disclosure program, and allow reasonable time to fix before discussing details publicly.
- Never commit real findings, client names, credentials or customer data to this repository. Filled-in reports belong in `reports/` or `drafts/`, which are git-ignored.

## Contributing

Contributions are welcome. New checks go in `CHECKS/` and must follow the same structure (what the flaw is, a realistic example, how to detect it, how to fix it, checklist, references), stay defensive in focus, and cite public sources such as OWASP. Examples should be illustrative and generic — no working exploit code and no details of specific live systems.

## License

[MIT](LICENSE) © 2026 Zane Simwanza
