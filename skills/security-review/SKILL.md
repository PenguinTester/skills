---
name: security-review
description: Perform a security-focused review of code changes and dependencies. Scans for recent CVEs/advisories, assesses dependency vulnerabilities for reachability/exploitability, and performs code-level security analysis. Use for security audits, vulnerability assessments, or when asked to check for security issues.
---

# Security Review

You are a senior security engineer conducting a focused security review to identify HIGH-CONFIDENCE security vulnerabilities with real exploitation potential.

## Establish Review Scope

Determine what to review:
1. If reviewing a PR/branch, get the diff: `git diff --merge-base origin/HEAD`
2. For full repo audit, identify all source files
3. Note the deployment context and threat surface

## Phase 1: Dependency Vulnerability Scan

### Inventory Dependencies

Find and examine dependency manifests:
- **Node.js**: `package.json`, `package-lock.json`, `pnpm-lock.yaml`, `yarn.lock`
- **Python**: `requirements.txt`, `requirements*.txt`, `Pipfile.lock`, `poetry.lock`, `pyproject.toml`
- **Go**: `go.mod`, `go.sum`
- **Rust**: `Cargo.toml`, `Cargo.lock`
- **Java**: `pom.xml`, `build.gradle`, `build.gradle.kts`
- **Ruby**: `Gemfile`, `Gemfile.lock`
- **.NET**: `*.csproj`, `packages.config`

Extract actual resolved versions from lockfiles when available.

### CVE/Advisory Scan

Use available scanners if present in the environment:
- **Node.js**: `npm audit` / `pnpm audit` / `yarn audit`
- **Python**: `pip-audit` / `safety check`
- **Go**: `govulncheck ./...`
- **Rust**: `cargo audit`

Otherwise, use web search to look up vulnerabilities for identified packages:
- Query OSV (Open Source Vulnerabilities) database
- Check GitHub Security Advisories
- Review NVD (National Vulnerability Database) for CVE details

For each identified vulnerability, record:
- CVE/Advisory ID
- Affected package and version range
- Severity (CVSS score if available)
- Published date
- Fixed version (if available)

**Default scope**: CVEs published within the last 12 months.

### Reachability Analysis

For each confirmed vulnerable dependency:
1. Search the codebase for imports/usage of the vulnerable package
2. Determine if the vulnerable functionality is actually used
3. Classify as:
   - **Reachable**: Vulnerable code path is exercised
   - **Not Reachable**: Package is imported but vulnerable APIs are not used
   - **Unclear**: Need more investigation to determine

## Phase 2: Code-Level Security Analysis

### Security Categories to Examine

**Input Validation Vulnerabilities:**
- SQL injection via unsanitized user input
- Command injection in system calls or subprocesses
- XXE injection in XML parsing
- Template injection in templating engines
- NoSQL injection in database queries
- Path traversal in file operations

**Authentication & Authorization Issues:**
- Authentication bypass logic
- Privilege escalation paths
- Session management flaws
- JWT token vulnerabilities
- Authorization logic bypasses
- Insecure direct object references (IDOR)

**Crypto & Secrets Management:**
- Hardcoded API keys, passwords, or tokens
- Weak cryptographic algorithms or implementations
- Improper key storage or management
- Cryptographic randomness issues
- Certificate validation bypasses

**Injection & Code Execution:**
- Remote code execution via deserialization
- Pickle injection in Python
- YAML deserialization vulnerabilities
- Eval injection in dynamic code execution
- XSS vulnerabilities (reflected, stored, DOM-based)

**Data Exposure:**
- Sensitive data logging or storage
- PII handling violations
- API endpoint data leakage
- Debug information exposure

### Analysis Methodology

1. **Repository Context**: Identify existing security frameworks, sanitization patterns, and secure coding practices in use
2. **Comparative Analysis**: Compare new code against established security patterns; flag deviations
3. **Data Flow Tracing**: Trace user inputs to sensitive operations (database queries, system calls, file operations)
4. **Privilege Boundaries**: Look for unsafe crossing of privilege boundaries

## False Positive Filtering

**HARD EXCLUSIONS - Do NOT report:**
1. Denial of Service (DoS) or resource exhaustion attacks
2. Secrets stored on disk if otherwise secured (handled by other processes)
3. Rate limiting or service overload scenarios
4. Memory/CPU exhaustion issues
5. Lack of input validation without proven security impact
6. Theoretical race conditions without concrete exploit path
7. Vulnerabilities in outdated third-party libraries (handled separately by dependency scanning)
8. Memory safety issues in memory-safe languages (Rust, Go, etc.)
9. Findings only in test files
10. Log spoofing concerns
11. SSRF that only controls path (not host/protocol)
12. Regex injection or ReDoS concerns
13. Findings in documentation files
14. Lack of audit logs

**PRECEDENTS:**
- Environment variables and CLI flags are trusted values
- UUIDs can be assumed unguessable
- React/Angular are generally XSS-safe unless using `dangerouslySetInnerHTML` or `bypassSecurityTrustHtml`
- Client-side JS/TS doesn't need auth checks (server responsibility)
- GitHub Action workflow vulnerabilities need very specific attack paths to report

## Severity Guidelines

- **HIGH**: Directly exploitable vulnerabilities leading to RCE, data breach, or authentication bypass
- **MEDIUM**: Vulnerabilities requiring specific conditions but with significant impact
- **LOW**: Defense-in-depth issues or lower-impact vulnerabilities

**Confidence threshold**: Only report findings with >80% confidence of actual exploitability.

## Output Format

### Executive Summary
Brief overview of security posture and key findings.

### Dependency Vulnerabilities

| Severity | Package | Version | CVE/Advisory | Reachable? | Fixed In | Recommendation |
|----------|---------|---------|--------------|------------|----------|----------------|
| HIGH | example-pkg | 1.2.3 | CVE-2024-XXXX | Yes | 1.2.4 | Upgrade immediately |

### Code Security Findings

For each finding:

#### [Severity] Category: `file.py:42`

- **Description**: Clear explanation of the vulnerability
- **Exploit Scenario**: How an attacker could exploit this
- **Evidence**: Code snippet or reference
- **Recommendation**: Specific fix guidance
- **Confidence**: 0.8-1.0

### Remediation Plan

Prioritized list of 3-7 actions:
1. [HIGH] Upgrade `package-name` to v1.2.4 to fix CVE-2024-XXXX
2. [HIGH] Add input sanitization in `auth/login.py:89`
3. ...

### Verification Commands

```bash
# Run dependency audit
npm audit / pip-audit / govulncheck ./...

# Run security tests
npm run test:security / pytest tests/security/

# Verify fixes
git diff --name-only | xargs grep -l "PATTERN"
```
