# Security Policy

`open-notebooklm` is an unofficial, community-maintained client for Google NotebookLM (Gemini Notebook). Because this project handles Google authentication material to interact with reverse-engineered endpoints, security, credential hygiene, and boundary isolation are top priorities.

---

## Supported Versions

Only the latest release track receives security updates. Earlier versions are deprecated and unsupported.

| Version Track | Supported Status | Notes |
| :--- | :--- | :--- |
| **0.8.x** | :white_check_mark: Supported | Active release line; receives vulnerability fixes and security patches. |
| **< 0.8.0** | :x: Unsupported | Legacy releases predating current storage and security architecture. |

---

## Reporting a Vulnerability

We take vulnerability reports seriously and appreciate responsible disclosure.

- **DO NOT** disclose security vulnerabilities via public GitHub issues, discussions, or pull requests.
- **Reporting Channel:**
  - Submit an advisory via **[GitHub Private Vulnerability Reporting](https://github.com/ishandutta2007/open-notebooklm/security/advisories/new)** (preferred).
  - Or email maintainers directly: `teng.lin@gmail.com` (and project maintainers listed in [`pyproject.toml`](file:///C:/Users/ishan/Documents/Projects/open-notebooklm3/pyproject.toml)).
- **What to Include in Your Report:**
  1. Detailed summary of the potential vulnerability.
  2. Proof-of-concept (PoC) script or step-by-step reproduction instructions.
  3. Scope of exposure (e.g. credential leakage, privilege escalation, unauthorized local or remote access).
  4. Any proposed remediations or patches if available.
- **Disclosure Policy:**
  - We aim to acknowledge reports within 48 hours.
  - We ask that you observe coordinated disclosure and maintain confidentiality until an official fix or advisory has been published.

---

## Threat Model & Security Boundaries

`open-notebooklm` acts as a client-side bridge between users/agents and Google's NotebookLM services. It operates across multiple transport modes and environments.

### Trust Boundaries

```text
[ AI Agent / CLI / User Script ]
               │
               ▼
   [ open-notebooklm Core API ]
        │                  │
        ▼ (Storage)        ▼ (Transport)
┌──────────────────┐  ┌────────────────────────────────────────────────────────┐
│ Profile Directory│  │ Web batchexecute (HTTPS) / Android Mobile gRPC (TLS)   │
│ - storage_state  │  └────────────────────────────────────────────────────────┘
│ - master_token   │                             │
│ - context/locks  │                             ▼
└──────────────────┘                 [ Google Cloud Infrastructure ]
```

1. **Local System Isolation:**
   - Credentials (`storage_state.json`, `master_token.json`) are account-equivalent. Any process with filesystem access to these files can impersonate the user against NotebookLM.
   - Posix file permissions default to restrictive modes (`0700` for directories, `0600` for credential files). On Windows, access is controlled via filesystem inherited ACLs.
2. **Network Communications:**
   - All outbound RPC traffic connects directly to official Google endpoints (`*.google.com`, `notebooklm.google.com`) over TLS.
   - No telemetry, analytics, or user secrets are dispatched to third-party servers.
3. **Agent & Subprocess Isolation:**
   - When executing external refresh commands (`NOTEBOOKLM_REFRESH_CMD`), sensitive NotebookLM environment variables are scrubbed to minimize ambient privilege escalation.
   - Chrome DevTools Protocol (`CDP`) attachments must strictly be loopback (`127.0.0.1` / `localhost`). Exposing CDP ports over public networks exposes full browser control and account theft risk.

---

## Credential Hygiene & Storage Architecture

### Local Storage Layout

Credentials are partitioned per profile under `~/.notebooklm/profiles/<profile>/` (configurable via `NOTEBOOKLM_HOME` and `NOTEBOOKLM_PROFILE`):

| File Path | Purpose / Secret Level | POSIX Mode | Best Practice |
| :--- | :--- | :--- | :--- |
| `profiles/<profile>/storage_state.json` | Active Google session cookies (`__Secure-1PSID`, `__Secure-1PSIDTS`, etc.) | `0600` (User RW only) | Treat as live password; never commit or leak in logs. |
| `profiles/<profile>/master_token.json` | Durable Google master authentication token | `0600` (User RW only) | Capable of minting fresh sessions indefinitely until revoked. |
| `profiles/<profile>/browser_profile/` | Playwright persistent browser context | `0700` (User RWX only) | Contains local browser cache, cookies, and session state. |
| `profiles/<profile>/context.json` | Active profile context, selected notebook, metadata | `0600` (User RW only) | Unsensitive runtime context state. |
| `config.json` | Global CLI preferences (active profile, display defaults) | Default | Non-sensitive configuration. |

### Defense-in-Depth Protections

- **Secret Redaction:** `CookieJar`, `MasterToken`, and connection credentials implement masked `__repr__` and `__str__` outputs to prevent accidental leakage in tracebacks, logs, and error envelopes.
- **Atomic Operations & Lock Files:** Writes are mediated by `filelock` and atomic file publication wrappers (`credential_io.py`) to eliminate race conditions and corrupted credential files during concurrent multi-agent access.
- **In-Memory Headless Auth:** CI/CD pipelines can supply raw session data via the `NOTEBOOKLM_AUTH_JSON` environment variable, avoiding writing secrets to disk.

---

## Best Practices for Operators & AI Agents

### 1. Environment & CI/CD Security
- **Never commit `.notebooklm/`:** Ensure `.notebooklm/` and `*.json` token stores are included in your `.gitignore`.
- **Use Dedicated Service Accounts:** Run automation and headless workflows using a dedicated Google Workspace or personal service account rather than a primary personal account.
- **Memory-Only CI Authentication:** Pass credentials into automated pipelines via GitHub Secrets / CI secret managers mapped to `NOTEBOOKLM_AUTH_JSON`.

### 2. FastMCP & Remote Server Deployments
- **Authenticate MCP Connections:** When deploying `open-notebooklm` as an SSE MCP server accessible over local networks or tunnels (Cloudflare, Tailscale), always enable bearer token verification or tunnel-level access control (OAuth/OIDC).
- **Secure CDP Endpoints:** Never bind remote browser debugging ports (`--remote-debugging-port`) to `0.0.0.0`. Restrict to loopback interfaces (`127.0.0.1`).

### 3. Compromise Response & Revocation
If credential exposure or token leakage occurs:
1. **Revoke Google Access:** Visit [Google Account Security: Third-party apps & services](https://myaccount.google.com/connections) and revoke access for the connected session or device.
2. **Purge Local Secrets:** Delete local credential caches:
   ```bash
   rm -rf ~/.notebooklm/profiles/<profile>/
   ```
3. **Change Password:** Changing the Google account password invalidates existing sessions. For master tokens, ensure corresponding device/app sessions are revoked.

---

## Dependency & Supply Chain Security

`open-notebooklm` keeps its core dependency footprint lean. Optional extensions are partitioned into targeted installation extras:

| Extra | Included Dependencies | Purpose | Security Notes |
| :--- | :--- | :--- | :--- |
| **base** | `httpx`, `click`, `rich`, `filelock` | Core CLI and async SDK | Minimal dependency tree, pinned upper bounds. |
| `browser` | `playwright` | Interactive browser login & automation | Downloads and manages browser binaries in isolated cache. |
| `cookies` | `rookie-cookies` | Local browser session extraction | Reads protected OS cookie stores (DPAPI/Keychain/SecretService). |
| `headless` | `gpsoauth` | Master-token session exchange | Pure Python; performs OAuth exchange without browser automation. |
| `android` | `grpcio`, `protobuf`, `gpsoauth` | Mobile gRPC transport | Authenticated via bearer tokens over TLS. |
| `mcp` | `fastmcp` | Model Context Protocol adapter | Exposes tools to AI agents; supports stdio and SSE transports. |
| `server` | `fastapi`, `uvicorn`, `python-multipart` | REST automation API | Local or containerized web service. |

### Automated Auditing

Dependencies are continuously checked against known vulnerability databases (OSV, PyPI Advisory DB) using `pip-audit`:

```bash
# Verify locked environment against known CVEs
uv export --frozen --extra browser --extra dev --extra markdown --format requirements-txt --no-emit-project \
  | uv run pip-audit --strict --require-hashes --disable-pip -r /dev/stdin
```

---

## Architectural & Protocol Disclaimers

- **Unofficial Integration:** `open-notebooklm` connects to Google's internal web (`batchexecute`) and mobile gRPC protocols. Google does not publish official SLAs, deprecation policies, or security guarantees for these reverse-engineered APIs.
- **Account Safeguards:** Rapid automated bursts, high-frequency requests, or anomalous geographic logins may trigger Google's automated rate limits, CAPTCHA challenges, or session invalidation.

---

## Additional Security Resources

- Detailed technical security model: [docs/security.md](file:///C:/Users/ishan/Documents/Projects/open-notebooklm3/docs/security.md)
- Authentication troubleshooting and session freshness: [docs/troubleshooting.md](file:///C:/Users/ishan/Documents/Projects/open-notebooklm3/docs/troubleshooting.md)
- Setup and installation methods: [docs/installation.md](file:///C:/Users/ishan/Documents/Projects/open-notebooklm3/docs/installation.md)
