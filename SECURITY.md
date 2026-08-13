# Security Policy

QuantDinger is a **local-first, self-hosted quantitative trading system**.

Security is a core design principle of the project, but it is important to
understand the **responsibility boundaries** that come with self-hosted software.

This document explains what we support, how to report vulnerabilities,
and what to expect from the process.

---

## 🔒 Supported Versions

QuantDinger is under active development.

At this stage:
- The **`main` branch** is the only supported version for security updates.
- Older commits, forks, or modified deployments are **not actively supported**.

Users are strongly encouraged to stay up to date with the latest release
or commit when running QuantDinger in production environments.

---

## 🧠 Security Model & Scope

QuantDinger is designed to run **entirely under the user’s control**.

### In Scope
We consider the following areas in scope for security review:

- Source code vulnerabilities in this repository
- Authentication and authorization logic within QuantDinger
- Handling of API keys, secrets, and credentials by the application
- Strategy execution logic and isolation boundaries
- Default configuration security issues

### Out of Scope
The following are outside the scope of this security policy:

- Misconfigured user environments (OS, Docker, firewall, cloud host)
- Compromised user machines or infrastructure
- Third-party services, exchanges, or APIs
- Modified or unofficial builds of QuantDinger

---

## 📣 Reporting a Vulnerability

If you believe you have found a security vulnerability in QuantDinger,
we appreciate responsible disclosure.

### How to Report

Please **do not open a public GitHub issue** for security vulnerabilities.

Instead, report privately via email:

- **Email**: see the contact address listed in `README.md`
- **Subject**: `[Security] Brief description of the issue`

Please include:
- a clear description of the vulnerability
- steps to reproduce (if applicable)
- potential impact
- any suggested mitigations (optional)

---

## ⏱️ Response Expectations

We aim to:
- acknowledge reports within **72 hours**
- provide a preliminary assessment within **7 days**

Timelines may vary depending on the complexity and severity of the issue.

If a report is accepted, we will coordinate a fix and, when appropriate,
a responsible public disclosure.

---

## 🤝 Responsible Disclosure

We ask security researchers to:
- avoid exploiting vulnerabilities beyond proof of concept
- allow reasonable time for remediation before public disclosure
- act in good faith and with respect for users

We are happy to acknowledge responsible disclosures
in release notes or documentation, if desired.

---

## ⚠️ Disclaimer

QuantDinger is provided **as-is**, without warranty.

As a self-hosted system, users are responsible for:
- securing their own environments
- protecting API keys and credentials
- complying with applicable laws and regulations

---

Security is not a feature — it is a shared responsibility.

Thank you for helping keep QuantDinger safe.
