<div align="center">

# 🛡️ Sentinel Core v2.2.1

**Real-Time Security Enforcement Gate for Development Teams**

[![License](https://img.shields.io/badge/license-Commercial-red.svg)](TERMS_OF_USE.md)
[![Python](https://img.shields.io/badge/python-3.10%2B-blue.svg)](https://python.org)
[![Platform](https://img.shields.io/badge/platform-Linux%20%7C%20macOS%20%7C%20Windows-lightgrey.svg)]()
[![Status](https://img.shields.io/badge/status-Production%20Ready-brightgreen.svg)]()
[![Version](https://img.shields.io/badge/version-2.3-blue.svg)]()

*ALLOW or BLOCK — no ambiguity, no silent bypass.*

**[🌐 Website](https://datawizual.github.io) · [📧 Get License](mailto:eldorzufarov66@gmail.com)**

---

### Installation Demo
![Sentinel Core Installation](demo_1.gif)

### Security Gate in Action
![Sentinel Core Block](demo_2.gif)

</div>

---

## What Is Sentinel Core?

Sentinel Core is an enterprise-grade security enforcement system that **intercepts every git commit** before it reaches your codebase. It runs an embedded [Auditor Core](https://datawizual.github.io) engine with AI-powered analysis via Gemini 2.5 Flash — verifying threats and eliminating false positives in real time.

Every violation is **blocked immediately** and creates an alert in your admin repository. Developers have no way to silently bypass enforcement.

---

## What Sentinel Protects

| Category | Details |
|---|---|
| 🔑 **Secrets** | Passwords, API keys, tokens hardcoded in source files |
| ⚙️ **CI/CD Configurations** | GitHub Actions, GitLab CI, Jenkinsfile |
| 🏗️ **Infrastructure** | Kubernetes, Terraform, Docker misconfigurations |
| 🐍 **Python Source Code** | Injection, insecure cryptography, SQL injection |
| 📦 **Supply Chain** | Unpinned dependencies, unsafe base images |
| 🔗 **Attack Chains** | Multi-step exploit paths combining multiple findings |
| 📋 **Compliance** | Findings mapped to SOC 2 TSC, CIS Controls v8, ISO/IEC 27001:2022 |

---

## How It Works

```
Developer runs: git commit
        ↓
Sentinel intercepts via pre-commit hook
        ↓
Auditor Core scans changed files (10 detection engines)
        ↓
Chain Analyzer correlates findings into attack paths
        ↓
Gemini AI verifies threats (chain-aware context)
        ↓
Intelligence Engine applies taint analysis and reachability scoring
        ↓
ALLOW → commit proceeds normally
BLOCK → commit rejected + alert fired to admin GitHub Issues
```

---

## Chain Analysis — Attack Path Detection

Sentinel Core v2.2.1 introduces **Chain Analysis**: a deterministic engine that identifies multi-step attack paths by correlating findings that, in combination, create a materially higher risk than any single finding in isolation.

### How chains work

The Chain Analyzer runs after all detectors complete and before AI advisory. It evaluates findings against configurable rules, each defining a trigger condition and a consequence class. When a trigger finding and a consequence finding co-exist within the same codebase scope, a chain is formed.

Each chain:

- Receives a unique identifier (`CHAIN_XXXX`) visible across the JSON report
- **Escalates the severity** of every finding in the chain to match the chain risk — a `LOW` severity finding that feeds a `CRITICAL` injection path is reported as `CRITICAL`
- Is passed to the AI advisor as an explicit chain context block, improving the accuracy of AI verdicts
- Appears in a dedicated `attack_paths` section in the JSON report

### Example

A hardcoded API key (`SECRET_HIGH_ENTROPY`, initially `LOW`) and a `SAST_COMMAND_INJECTION` finding in the same module trigger the `secret_to_command_injection` rule. Both findings are escalated to `CRITICAL` and grouped under a shared chain ID. The AI advisor receives the full chain context, not just the individual findings.

### JSON output

Each finding in a chain includes a `chain` block:

```json
{
  "rule_id": "SECRET_HIGH_ENTROPY",
  "severity": "CRITICAL",
  "chain": {
    "id": "CHAIN_0001",
    "risk": "CRITICAL",
    "rule": "secret_to_command_injection",
    "original_severity": "LOW",
    "partner_finding_id": "<uuid-of-SAST_COMMAND_INJECTION>"
  }
}
```

The report root also contains an `attack_paths` section aggregating all chains for SIEM integration:

```json
"attack_paths": [
  {
    "chain_id": "CHAIN_0001",
    "chain_rule": "secret_to_command_injection",
    "chain_risk": "CRITICAL",
    "step_count": 2,
    "steps": [
      { "rule_id": "SECRET_HIGH_ENTROPY", "file_path": "src/config.py", "line": 42, ... },
      { "rule_id": "SAST_COMMAND_INJECTION", "file_path": "src/config.py", "line": 87, ... }
    ]
  }
]
```

### Chain configuration

Chain behaviour is controlled in `audit-config.yml`:

```yaml
chaining:
  enabled: true
  max_chain_length: 5        # Maximum findings in a single chain
  max_line_distance: 200     # Maximum line distance between chained findings
  require_same_scope: true   # Restrict chaining to the same directory scope
  max_scope_depth: 2         # Directory depth tolerance for scope check
  rules:
    - name: "secret_to_command_injection"
      triggers: ["SECRET_HIGH_ENTROPY", "GITLEAKS_", "SECRET_AWS"]
      then: ["SAST_COMMAND_INJECTION", "BANDIT_B602", "BANDIT_B603"]
      resulting_risk: "CRITICAL"
```

> **Note:** Chain rules match against `rule_id` and `description` fields only — not arbitrary metadata. Use rule_id prefixes in triggers and then for precise, false-positive-free matching.

---

## 🤖 AI Operation Modes — External and Fully Local

Auditor Core supports two AI operation modes:

* **External LLM mode** (default)
* **Local LLM mode** (fully offline)

This ensures flexibility for enterprise, regulated, or air-gapped environments.

### Design Principle

AI in Auditor Core is **advisory and validation-only**.
The deterministic scanning and Chain Analyzer always run first and produce a complete structural result independently of AI availability.

If AI is disabled or unavailable:

* All findings are still generated.
* Chain analysis still executes.
* SPI and Gate logic remain deterministic and reproducible.
* Reports are produced without AI commentary.

**AI enhances reasoning quality — it does not control enforcement.**

### Mode 1 — External LLM (Default)

External mode integrates with supported providers:

* Google Gemini (primary)
* Groq (automatic fallback)

Configuration example:

```yaml
ai:
  enabled: true
  mode: "external"
  external:
    provider: "google"
    model: "gemini-2.5-flash"
```

If Gemini quota is exceeded or unavailable, Auditor Core automatically switches to Groq without interrupting the scan.

External mode is suitable for:

* Standard enterprise environments
* Teams requiring large-context reasoning
* Organizations without air-gap restrictions

### Mode 2 — Local LLM (Fully Offline Operation)

Auditor Core can run entirely offline using a local model adapter.

```yaml
ai:
  enabled: true
  mode: "local"
  local:
    model_path: "/models/your-local-model.gguf"
    backend: "llama.cpp"
```

In local mode:

* No outbound network calls occur.
* No code or findings leave the machine.
* All AI validation runs inside the local environment.

This mode is designed for:

* Air-gapped deployments
* Government and defense environments
* High-sensitivity codebases
* Organizations with strict data residency policies

### Graceful Degradation

AI availability does not affect:

* Detection results
* Chain formation
* SPI calculation
* Gate override logic
* Compliance mapping

If AI fails, times out, or is disabled:

* Findings remain marked as UNVERIFIED
* Chain severity remains deterministic
* Reports include a note indicating AI advisory was skipped

**No scan is blocked due to AI failure.**

### Deterministic‑First Architecture Reminder

Auditor Core follows a strict execution order:

1. Deterministic detectors
2. Chain Analyzer
3. SPI computation
4. AI advisory layer (optional enrichment)

This guarantees:

* Reproducibility under audit scrutiny
* Stable results across runs
* No probabilistic finding generation
* Compliance-safe operation even without AI

### Security & Data Handling

Auditor Core does not transmit source code unless external AI mode is explicitly enabled.

* In local mode, all processing occurs on the host machine.
* AI receives structured finding context, not entire repositories.
* AI never generates new findings — it only validates existing deterministic results.
* This dual-mode architecture ensures that AI is a controlled reasoning amplifier — not a detection engine and not a runtime dependency.

---

## Installation

### Step 1 — Get Your Machine ID

Each machine requires a hardware-bound License Key. Run on every machine to be protected:

```bash
python3 get_id.py
```

Output:
```
==================================================
  Sentinel Core — Machine ID
==================================================
  Machine ID: 81CE1239487E2EA172FF41BC4DD13BED
==================================================
  Send this ID to: eldorzufarov66@gmail.com
```

Send the Machine ID to **eldorzufarov66@gmail.com** — you will receive a License Key unique to that machine.

> ⚠️ A key issued for one machine will not work on another.

### Step 2 — Prepare GitHub Tokens

Create two Personal Access Tokens at `GitHub → Settings → Developer settings → Fine-grained tokens`:

**SENTINEL_INSTALL_TOKEN** — installs Sentinel from the private repository
```
Repository: sentinel-core
Permissions: Contents → Read-only
```

**SENTINEL_ALERT_TOKEN** — posts enforcement alerts as GitHub Issues
```
Repository: your admin repository
Permissions: Issues → Read and write
```

### Step 3 — Install via start.sh

Copy `start.sh` into the root of the project you want to protect:

```bash
bash start.sh
```

The script handles everything automatically:
- Accepts Terms of Use
- Installs Sentinel from the private GitHub repository
- Configures `.env` with your credentials
- Initializes hardware-bound license
- Creates `sentinel.yaml` enforcement policy
- Sets up GitHub Actions workflow
- Installs pre-commit hook

Successful output:
```
✅ License verified for Machine ID: 81CE1239487E2EA172FF41BC4DD13BED
✅ sentinel.yaml initialized
✅ GitHub Workflow initialized
✅ Pre-commit hook installed
------------------------------------------------------------
✅ SENTINEL CORE DEPLOYED SUCCESSFULLY
------------------------------------------------------------
```

### Step 4 — Verify

Test the enforcement gate immediately after installation:

```bash
echo 'password = "admin123"' > test_vuln.py
git add test_vuln.py
git commit -m "test"
```

Expected result:
```
🔍 Sentinel is verifying commit security...
❌ Found 1 security violations!
- [CRITICAL] SEC-001: Hardcoded Password in test_vuln.py at line 1.
🚀 Remote Alert Sent Successfully!
❌ Terminating: 1 CRITICAL threats found.
```

Clean up:
```bash
rm test_vuln.py
```

---

## Policy Configuration

All enforcement rules are defined in `sentinel.yaml`:

```yaml
severity:
  SEC-001: BLOCK        # Hardcoded secrets
  SUPPLY-001: BLOCK     # Supply chain integrity
  INFRA-K8S-001: BLOCK  # Kubernetes misconfigurations
  CICD-001: BLOCK       # CI/CD security issues

overrides: []
ignore:
  - venv/*
  - node_modules/*
```

To temporarily allow a violation with documented justification:

```yaml
overrides:
  - rule_id: SUPPLY-001
    justification: "Legacy base image — reviewed by security team"
```

To exclude intentionally-vulnerable projects or test fixtures from scanning, add them to `audit-config.yml`:

```yaml
scanner:
  exclude_patterns:
    - "DVWA-master/*"     # intentionally vulnerable training app
    - "tests/fixtures/*"  # scanner test fixtures
```

---

## Enforcement Alerts

Every blocked commit automatically creates a GitHub Issue in your admin repository:

```
🔴 BLOCK — SEC-001: Hardcoded Password
Environment: 💻 Local Development
Machine: worker-pc-01
Triggered by: developer_username
Timestamp: 2026-03-13 14:22:01
```

When a chain is detected, the alert includes an **Attack Path Analysis** section:

```
🔗 Attack Path Analysis:

CHAIN_0001 | Rule: secret_to_command_injection | Risk: CRITICAL
  Step 1: SECRET_HIGH_ENTROPY at src/config.py:42  (LOW → CRITICAL)
  → Step 2: SAST_COMMAND_INJECTION at src/config.py:87  (MEDIUM → CRITICAL)
```

Administrators have full visibility. Developers have no access to the enforcement dashboard.

---

## Project Structure After Installation

```
your-project/
├── start.sh                    ← provisioning script
├── audit-config.yml            ← Auditor Core configuration (incl. chain rules)
├── sentinel.yaml               ← enforcement policy
├── .env                        ← credentials (never commit)
├── .github/
│   └── workflows/
│       └── sentinel.yml        ← CI/CD pipeline protection
├── reports/
│   ├── report_*.pdf            ← Executive Summary (SOC 2 / cyber insurance ready)
│   ├── report_*.html           ← Interactive Enterprise Posture Report
│   └── report_*.json           ← Machine-readable findings with chain & compliance mapping
└── venv/                       ← Python virtual environment
```

---

## Requirements

| Component | Version |
|---|---|
| Python | 3.10+ |
| Git | any |
| OS | Linux / macOS / Windows |
| reportlab | Required (PDF report generation) |
| Gemini API | Optional (AI analysis) |
| Groq API | Optional (Gemini fallback) |

---

## What's New in v2.2.1

Sentinel Core v2.2.1 delivers the Chain Analysis engine from Auditor Core v2.2.1 together with a set of precision fixes identified during production validation:

* **Chain Analysis** — deterministic multi-step attack path detection. Findings that individually appear low-risk are automatically correlated and severity-escalated when they form a viable exploit chain. Chains appear in the JSON report as a dedicated `attack_paths` section and in GitHub Issue alerts as a readable attack path narrative.

* **Chain-aware AI advisory** — AI receives the full chain context when evaluating chained findings, not just the individual finding, improving verdict accuracy for correlated risks.

* **Chain-aware gate** — the enforcement gate now fires on chain-escalated CRITICAL/HIGH findings even when no AI verdict is available, closing a gap where chained findings without AI coverage could silently pass.

* **Precise file exclusion** — replaced `rglob()` + `fnmatch` with `os.walk()` directory pruning. The previous implementation failed to exclude `venv/lib/python3.12/site-packages/` because `fnmatch("venv/lib/.../file.py", "venv/*")` returns `False`. Directory pruning stops traversal at the excluded root, reducing scan scope from thousands of venv files to only project files.

* **Scope-safe chaining** — the chain scope check now uses project-relative paths instead of absolute resolved paths. Previously, any two files inside `site-packages/` were within scope of each other (depth ≤ 2 in absolute terms), generating chains across unrelated third-party packages.

* **AI filter post-chain** — `_filter_findings_for_ai` now reads the final post-chain severity rather than the raw detector severity stored in metadata. Previously, findings escalated to CRITICAL by the chain engine were still filtered out as LOW, leaving critical chains without AI coverage.

* **Chain partner IDs** — `partner_finding_id` in the JSON chain block is now correctly populated for all chain members. The previous implementation only wrote this field for two-finding chains via the `_apply_chain()` helper; multi-finding chains always produced an empty string.

* **Tightened chain rules** — chaining rules now match on `rule_id` prefixes rather than generic description keywords. The previous `sqli_to_auth_bypass` rule matched `BANDIT_B105` (hardcoded password string) on both trigger and consequence sides because the word "password" appears in the finding description, creating nonsensical self-chains.

---

## FAQ

**Can a developer bypass Sentinel?**
A developer can run `git commit --no-verify` locally. However, the CI/CD pipeline will catch the push, block it, and alert the administrator with the developer's identity and machine ID.

**Does Sentinel slow down commits?**
Basic scanning takes 2–5 seconds. With Gemini AI enabled — 15–30 seconds depending on project size.

**What if Gemini API quota is exceeded?**
Sentinel automatically switches to Groq (llama-3.3-70b-versatile) as fallback. Zero interruption to enforcement.

**How do I update Sentinel?**
Run `start.sh` again. It updates the package while preserving your existing `.env` configuration.

**Is developer activity logged?**
Yes. Every blocked commit is recorded as a GitHub Issue with machine identity, username, timestamp, and violation details — creating an immutable audit trail.

**How is Sentinel different from Auditor Core?**
Sentinel is a real-time gate — it intercepts every commit automatically. [Auditor Core v2.2.1](https://datawizual.github.io) is a deep on-demand audit engine for comprehensive posture reports with PDF executive summaries, SOC 2 / CIS / ISO 27001 compliance mapping, and Evidence Appendix. Sentinel uses Auditor Core internally as its scanning engine.

**What is a chain and how does it affect the gate decision?**
A chain is a group of two or more findings that together form a complete attack path — for example, a hardcoded credential that feeds a command injection sink. When a chain is detected, all participating findings are escalated to the chain's resulting risk level (typically CRITICAL), and the gate blocks the commit regardless of the individual finding severities. Chain escalation cannot be suppressed via `sentinel.yaml::overrides`.

**My project includes intentionally-vulnerable code for testing. How do I exclude it?**
Add the directory to `exclude_patterns` in `audit-config.yml`:
```yaml
scanner:
  exclude_patterns:
    - "DVWA-master/*"
    - "vulnerable-fixtures/*"
```

---

## Support

📧 **eldorzufarov66@gmail.com**

Please include: Machine ID · OS version · Python version · description of the issue.

---

<div align="center">

© 2026 DataWizual Security Labs. All rights reserved.

Use governed by [TERMS_OF_USE.md](TERMS_OF_USE.md)

</div>
