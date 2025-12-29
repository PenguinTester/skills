---
name: security-review
description: Perform a security-focused review of code changes and dependencies. Scans for recent CVEs/advisories, assesses dependency vulnerabilities for reachability/exploitability, and performs code-level security analysis. Outputs validated-findings.json for downstream patch generation. Use for security audits, vulnerability assessments, or when asked to check for security issues.
version: 1.0.0
tags: [security, vulnerability, CVE, audit, review]
---

# Security Review

You are a senior security engineer conducting a focused security review to identify HIGH-CONFIDENCE security vulnerabilities with real exploitation potential. This skill is part of a security workflow that outputs structured findings for downstream automation.

## When to Use This Skill

- **PR/Branch security review** - Analyze code changes for vulnerabilities before merge
- **Full repository audit** - Comprehensive security assessment of a codebase
- **CVE/dependency scanning** - Check for known vulnerabilities in dependencies
- **Security workflow automation** - Generate findings for `security-patch-generation` skill

## Prerequisites

- Git repository with code to review
- Optional: `.factory/threat-model.md` (will be generated if missing)

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| Scope | PR diff, branch, or full repo | No | Current branch changes |
| CVE lookback | How far back to check CVEs | No | 12 months |
| Output format | markdown or json | No | Both |

## Instructions

### Step 1: Establish Review Scope

Determine what to review:

```bash
# For PR/branch review
git status
git diff --name-only origin/HEAD...
git log --no-decorate origin/HEAD...
git diff --merge-base origin/HEAD

# For full repo audit
find . -type f -name "*.js" -o -name "*.ts" -o -name "*.py" -o -name "*.go" -o -name "*.java" | head -100
```

Document:
- Files modified/added
- Deployment context (web app, API, CLI, library)
- Threat surface (public internet, internal, local only)

### Step 2: Generate or Load Threat Model

Check for existing threat model:
```bash
cat .factory/threat-model.md 2>/dev/null || echo "No threat model found"
```

If missing, create a basic threat model documenting:
- Application type and purpose
- Data sensitivity classification
- Authentication/authorization model
- External integrations and trust boundaries
- Known security controls

Save to `.factory/threat-model.md`.

### Step 3: Dependency Vulnerability Scan

#### 3.1 Inventory Dependencies

Find and examine dependency manifests:

| Ecosystem | Manifests | Lockfiles |
|-----------|-----------|-----------|
| Node.js | `package.json` | `package-lock.json`, `pnpm-lock.yaml`, `yarn.lock` |
| Python | `requirements.txt`, `pyproject.toml` | `poetry.lock`, `Pipfile.lock` |
| Go | `go.mod` | `go.sum` |
| Rust | `Cargo.toml` | `Cargo.lock` |
| Java | `pom.xml`, `build.gradle` | - |
| Ruby | `Gemfile` | `Gemfile.lock` |
| .NET | `*.csproj` | `packages.lock.json` |

Extract resolved versions from lockfiles when available.

#### 3.2 Run Available Scanners

Use scanners if present:
```bash
# Node.js
npm audit --json 2>/dev/null || pnpm audit --json 2>/dev/null || yarn audit --json 2>/dev/null

# Python
pip-audit --format json 2>/dev/null || safety check --json 2>/dev/null

# Go
govulncheck -json ./... 2>/dev/null

# Rust
cargo audit --json 2>/dev/null
```

#### 3.3 Web Search for CVEs

For packages without scanner coverage, use web search to query:
- OSV (Open Source Vulnerabilities) database
- GitHub Security Advisories
- NVD (National Vulnerability Database)

Search pattern: `"<package-name>" CVE OR vulnerability site:osv.dev OR site:github.com/advisories`

**Default scope**: CVEs published within the last 12 months.

#### 3.4 Reachability Analysis

For each identified vulnerability:

1. **Find usage** - Search for imports/requires of the vulnerable package
2. **Trace to vulnerable API** - Check if the specific vulnerable function/method is called
3. **Assess exploitability** - Can untrusted input reach the vulnerable code path?

Classify each as:
- **REACHABLE**: Vulnerable code path is exercised with untrusted input
- **POTENTIALLY_REACHABLE**: Package used, but need more analysis
- **NOT_REACHABLE**: Package imported but vulnerable APIs not used
- **UNCLEAR**: Insufficient information to determine

### Step 4: Code-Level Security Analysis

#### Security Categories to Examine

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

#### Analysis Methodology

1. **Repository Context Research**
   - Identify existing security frameworks and libraries
   - Look for established secure coding patterns
   - Examine existing sanitization and validation
   - Understand the project's security model

2. **Comparative Analysis**
   - Compare new code against existing security patterns
   - Identify deviations from established practices
   - Flag inconsistent security implementations
   - Note code that introduces new attack surfaces

3. **Vulnerability Assessment**
   - Trace data flow from user inputs to sensitive operations
   - Look for privilege boundaries being crossed unsafely
   - Identify injection points and unsafe deserialization
   - Check for missing authorization checks

### Step 5: False Positive Filtering

For each potential finding, apply these filters:

**HARD EXCLUSIONS - Automatically exclude:**
1. Denial of Service (DoS) or resource exhaustion attacks
2. Secrets stored on disk if otherwise secured
3. Rate limiting or service overload scenarios
4. Memory/CPU exhaustion issues
5. Lack of input validation without proven security impact
6. Input sanitization concerns in GitHub Actions unless clearly triggerable via untrusted input
7. Lack of hardening measures (only flag concrete vulnerabilities)
8. Theoretical race conditions without practical exploit path
9. Vulnerabilities in outdated third-party libraries (handled by dependency scan)
10. Memory safety issues in memory-safe languages (Rust, Go)
11. Findings only in test files
12. Log spoofing concerns (unsanitized log output)
13. SSRF that only controls path (not host/protocol)
14. User-controlled content in AI prompts
15. Regex injection or ReDoS concerns
16. Findings in documentation/markdown files
17. Lack of audit logs

**PRECEDENTS:**
- Logging URLs is safe; logging high-value secrets is a vulnerability
- UUIDs are unguessable and don't need validation
- Environment variables and CLI flags are trusted values
- React/Angular are XSS-safe unless using `dangerouslySetInnerHTML` or `bypassSecurityTrustHtml`
- Client-side JS/TS doesn't need auth checks (server responsibility)
- GitHub Action vulnerabilities need very specific attack paths
- Most ipython notebook vulnerabilities are not exploitable in practice

**Confidence Scoring:**
- 0.9-1.0: Certain exploit path identified
- 0.8-0.9: Clear vulnerability pattern with known exploitation methods
- 0.7-0.8: Suspicious pattern requiring specific conditions
- Below 0.7: Don't report (too speculative)

**Only report findings with confidence >= 0.8**

### Step 6: Severity Classification

| Severity | Criteria | Examples |
|----------|----------|----------|
| **CRITICAL** | Directly exploitable, high impact, no auth required | RCE, auth bypass, data breach |
| **HIGH** | Directly exploitable with significant impact | SQL injection, privilege escalation |
| **MEDIUM** | Exploitable with specific conditions | XSS requiring user interaction, IDOR |
| **LOW** | Defense-in-depth or lower impact | Missing security headers |

### Step 7: Generate Outputs

#### 7.1 Create `validated-findings.json`

This file is consumed by the `security-patch-generation` skill:

```json
{
  "scan_id": "scan-<timestamp>",
  "scan_date": "<ISO timestamp>",
  "repository": "<repo path or URL>",
  "branch": "<branch name>",
  "commit": "<commit hash>",
  "scope": "pr_diff | branch | full_repo",
  "threat_model_path": ".factory/threat-model.md",
  "findings": [
    {
      "id": "VULN-001",
      "status": "CONFIRMED",
      "type": "code | dependency",
      "category": "sql_injection | xss | command_injection | idor | ...",
      "severity": "CRITICAL | HIGH | MEDIUM | LOW",
      "confidence": 0.95,
      "cwe": "CWE-89",
      "title": "SQL Injection in User Search",
      "description": "User input from 'query' parameter is directly interpolated into SQL query without parameterization",
      "file": "src/api/users.js",
      "line": 42,
      "code_snippet": "const sql = `SELECT * FROM users WHERE name LIKE '%${query}%'`;",
      "exploit_scenario": "Attacker could extract database contents by manipulating the search parameter with payloads like: ' OR '1'='1' --",
      "recommendation": "Use parameterized queries with placeholder syntax",
      "references": [
        "https://cwe.mitre.org/data/definitions/89.html",
        "https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html"
      ],
      "evidence": {
        "data_flow": "req.query.search -> sql string interpolation -> db.query()",
        "sink": "database query execution"
      }
    }
  ],
  "dependency_findings": [
    {
      "id": "DEP-001",
      "status": "CONFIRMED",
      "package": "lodash",
      "version": "4.17.20",
      "ecosystem": "npm",
      "vulnerability_id": "CVE-2021-23337",
      "severity": "HIGH",
      "cvss": 7.2,
      "published_date": "2021-02-15",
      "fixed_version": "4.17.21",
      "reachability": "REACHABLE",
      "reachability_evidence": "lodash.template() called in src/utils/email.js:15",
      "description": "Command injection vulnerability in lodash template function",
      "recommendation": "Upgrade lodash to >= 4.17.21"
    }
  ],
  "summary": {
    "total_findings": 5,
    "by_severity": {
      "critical": 1,
      "high": 2,
      "medium": 1,
      "low": 1
    },
    "by_type": {
      "code": 3,
      "dependency": 2
    },
    "reachable_dependencies": 2
  }
}
```

#### 7.2 Create Markdown Report

Also output a human-readable markdown report:

### Executive Summary
Brief overview of security posture and key findings.

### Critical & High Severity Findings

For each CRITICAL/HIGH finding:

#### VULN-001: SQL Injection in User Search

| Field | Value |
|-------|-------|
| Severity | HIGH |
| CWE | CWE-89 |
| File | `src/api/users.js:42` |
| Confidence | 0.95 |

**Description:** User input from 'query' parameter is directly interpolated into SQL query.

**Exploit Scenario:** Attacker could extract database contents with: `' OR '1'='1' --`

**Recommendation:** Use parameterized queries with placeholder syntax.

### Dependency Vulnerabilities

| Severity | Package | Version | CVE | Reachable | Fixed In |
|----------|---------|---------|-----|-----------|----------|
| HIGH | lodash | 4.17.20 | CVE-2021-23337 | Yes | 4.17.21 |

### Remediation Plan

Prioritized actions:
1. **[CRITICAL]** Fix SQL injection in `src/api/users.js:42`
2. **[HIGH]** Upgrade lodash to 4.17.21
3. ...

### Verification Commands

```bash
# Run dependency audit
npm audit

# Run security tests (if available)
npm run test:security

# Verify no regressions
npm test
```

## Success Criteria

The skill is complete when:

- [ ] Threat model exists at `.factory/threat-model.md`
- [ ] All dependency manifests scanned for CVEs
- [ ] Reachability assessed for each dependency vulnerability
- [ ] Code-level security analysis completed
- [ ] False positive filtering applied
- [ ] `validated-findings.json` generated with correct schema
- [ ] Markdown report generated
- [ ] All findings have severity, confidence, and evidence

## Downstream Skills

After this skill completes, you can invoke:
- **`security-patch-generation`** - Generate fixes for CONFIRMED findings

## Example Invocations

**Review current branch changes:**
```
Perform a security review of the changes on this branch.
```

**Full repository audit:**
```
Run a comprehensive security audit of this repository.
```

**Check for recent CVEs:**
```
Scan dependencies for CVEs from the last 6 months and check if they're reachable.
```

**Generate findings for patching:**
```
Security review this PR and output validated-findings.json for patch generation.
```

## References

- OWASP Top 10: https://owasp.org/www-project-top-ten/
- CWE Database: https://cwe.mitre.org/
- OWASP Cheat Sheets: https://cheatsheetseries.owasp.org/
- OSV Database: https://osv.dev/
