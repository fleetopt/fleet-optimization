# Security Policy

## Overview

`fleet-optimization` processes sensitive operational data — including real-time vehicle telemetry, GPS location streams, driver personally identifiable information (PII), and route intelligence that may be commercially sensitive. We take the security of this system and the privacy of the individuals represented in its data seriously.

This document explains how to report vulnerabilities, what we consider in scope, and the commitments we make to researchers who engage with us in good faith.

---

## Supported Versions

We actively maintain security fixes for the following versions:

| Version | Supported          |
|---------|--------------------|
| `1.x` (latest) | ✅ Active support |
| `0.9.x` | ⚠️ Critical fixes only |
| `< 0.9`  | ❌ End of life — please upgrade |

If you are running an unsupported version, we strongly recommend upgrading before reporting an issue, as it may already be resolved.

---

## Reporting a Vulnerability

**Please do not open a public GitHub issue for security vulnerabilities.** Doing so exposes all users to risk before a fix is available.

### Preferred Channel

Report vulnerabilities privately via GitHub's built-in mechanism:

> **[Report a vulnerability](https://github.com/fleetopt/fleet-optimization/security/advisories/new)**

This opens a private advisory visible only to maintainers. It is the fastest path to triage.

### Alternative Contact

If you are unable to use GitHub's advisory system, open a private communication via the **[GitHub Security tab](https://github.com/fleetopt/fleet-optimization/security)** or contact a maintainer directly through their GitHub profile.

### What to Include

A useful report includes:

- A clear description of the vulnerability and its potential impact
- The affected component, endpoint, or data flow
- Steps to reproduce (proof-of-concept code or request/response pairs are helpful)
- Your assessment of exploitability (e.g., requires authenticated session, network position, etc.)
- Any suggested mitigations, if you have them

---

## Response Commitments

| Milestone | Target SLA |
|-----------|-----------|
| Acknowledgement of report | Within **48 hours** |
| Initial triage and severity assessment | Within **5 business days** |
| Status update (fix timeline or clarification) | Within **10 business days** |
| Patch release for critical/high severity | Within **30 days** where technically feasible |

We will keep you informed throughout the process and credit you in the release notes unless you prefer to remain anonymous.

---

## Scope

### In Scope

The following are considered valid targets for security research:

- **API layer** — authentication, authorization, input validation, rate limiting, and data exposure across all REST/gRPC endpoints
- **Route optimization engine** — manipulation of optimization inputs/outputs, injection of malicious constraints, denial-of-service via adversarial problem inputs
- **Telemetry ingestion pipeline** — spoofed GPS data acceptance, data injection, replay attacks on vehicle event streams
- **Driver and operator PII handling** — improper exposure, insufficient access controls, logging of sensitive fields
- **Authentication and session management** — token leakage, privilege escalation, insecure JWT handling, OAuth misconfiguration
- **Dependency vulnerabilities** — exploitable CVEs in direct or transitive dependencies that affect this project
- **Infrastructure-as-code and CI/CD** — misconfigured deployment pipelines, exposed secrets in workflow definitions, insecure defaults in Helm/Terraform configurations

### Out of Scope

The following are **not** eligible for this program:

- Vulnerabilities in third-party services we consume but do not control (e.g., mapping providers, cloud infrastructure)
- Issues requiring physical access to a vehicle or device
- Social engineering of maintainers or employees
- Denial-of-service attacks against production infrastructure
- Automated scanner output without demonstrated exploitability
- Theoretical vulnerabilities with no practical attack path

---

## Security Considerations for This Domain

Fleet optimization systems face a distinct threat model. Contributors, integrators, and operators should be aware of the following risks specific to this project:

### GPS and Telemetry Spoofing

The system ingests real-time location and sensor data from vehicles. Unauthenticated or weakly authenticated telemetry feeds are a primary attack surface. Always validate the source authenticity of incoming telemetry, enforce rate limits on ingestion endpoints, and treat GPS coordinates as untrusted input before normalization.

### Route Data as Sensitive Intelligence

Optimized route outputs may reveal commercial patterns — delivery frequencies, customer locations, and operational schedules — that are sensitive to operators. Access to route history and optimization results should be scoped to the minimum necessary audience and not exposed in aggregate APIs without appropriate access controls.

### Driver PII and Labor Compliance

Driver records may include location history, hours of service data, and performance metrics. This data is subject to privacy regulations (GDPR, CCPA, and sector-specific transport regulations in various jurisdictions). Ensure that data minimization, retention limits, and access logging are applied to all driver-related endpoints.

### Optimization Engine Adversarial Inputs

Solvers and heuristic engines can be vulnerable to resource exhaustion or incorrect outputs if given adversarially crafted inputs (e.g., degenerate constraint sets, extremely large problem instances). Validate and bound all inputs before they reach the solver layer.

### Supply Chain and Dependency Risk

This project relies on numerical optimization libraries, geospatial tooling, and routing graph dependencies. Monitor these for CVEs and pin dependencies to verified digests in production builds.

---

## Secure Development Practices

We expect contributors to follow these practices. PRs that introduce violations may be blocked:

- **No secrets in source** — API keys, credentials, and tokens must never appear in committed code, configuration files, or test fixtures. Use environment variables or a secrets manager.
- **Dependency hygiene** — new dependencies require justification. Keep the dependency surface minimal. Run `npm audit` / `pip-audit` / equivalent before merging.
- **Input validation** — all external inputs (API requests, telemetry payloads, file uploads) must be validated and sanitized before processing.
- **Principle of least privilege** — services, database users, and IAM roles should be scoped to only the permissions they require.
- **Avoid logging sensitive data** — do not log driver PII, vehicle credentials, or raw GPS payloads in application logs.
- **Signed commits** — maintainers are expected to sign commits with a verified GPG key. Contributors are encouraged to do the same.

---

## Known Security Controls

For transparency, the following controls are currently in place:

- API authentication via short-lived JWT tokens with audience and expiry validation
- Role-based access control (RBAC) on all multi-tenant data endpoints
- TLS 1.2+ enforced on all ingestion and API interfaces
- Dependency scanning via Dependabot with auto-merge for patch-level non-breaking updates
- Static analysis integrated into CI (SAST)
- Secrets scanning enabled on the repository

---

## Disclosure Policy

We follow a **coordinated disclosure** model:

1. Reporter submits a private report.
2. We triage, confirm, and develop a fix.
3. We coordinate a disclosure date with the reporter (typically 90 days from confirmation, or sooner if a fix is ready).
4. We publish a GitHub Security Advisory and release a patched version simultaneously.
5. Reporter is credited in the advisory and changelog unless they opt out.

We will not pursue legal action against researchers who act in good faith, stay within scope, and avoid accessing, modifying, or exfiltrating real user data.

---

## Hall of Fame

We publicly recognize researchers who responsibly disclose valid vulnerabilities. Acknowledgements are listed in [`ACKNOWLEDGEMENTS.md`](./ACKNOWLEDGEMENTS.md) and in the GitHub Security Advisories for each finding.

---

## Questions

For non-vulnerability security questions (e.g., compliance inquiries, data handling questions, integration security review), open a GitHub Discussion under the **Security** category or reach out directly via the **[GitHub Security tab](https://github.com/fleetopt/fleet-optimization/security)**.
