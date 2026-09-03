# Contributing to open-notebooklm

Thank you for contributing to **open-notebooklm** (`notebooklm-py` on PyPI)! We welcome bug fixes, documentation improvements, feature additions, and performance enhancements.

This guide covers everything you need to set up your environment, follow coding and architecture conventions, run the multi-tiered test suite, and submit pull requests that pass all merge gates.

---

## 📑 Table of Contents

- [🚀 Quick Start (Contributor Environment)](#-quick-start-contributor-environment)
  - [Canonical Install (`uv`)](#canonical-install-uv)
  - [Fallback Install (`pip`)](#fallback-install-pip)
- [🏛️ Architecture & Core Principles](#️-architecture--core-principles)
  - [Module Layout](#module-layout)
  - [Layer Responsibilities & Boundary Rules](#layer-responsibilities--boundary-rules)
  - [Architectural Decision Records (ADRs)](#architectural-decision-records-adrs)
- [🧪 Testing Guidelines & Test Tiers](#-testing-guidelines--test-tiers)
  - [Test Tiers](#test-tiers)
  - [Running Tests](#running-tests)
  - [Fast Feedback Loop (Skipping `repo_lint`)](#fast-feedback-loop-skipping-repo_lint)
  - [Testing with Optional Adapters (`mcp`, `server`, `android`)](#testing-with-optional-adapters-mcp-server-android)
  - [VCR Cassettes & Fixture Management](#vcr-cassettes--fixture-management)
  - [Coverage Expectations](#coverage-expectations)
- [🛡️ Architecture Invariant Gates (`tests/_guardrails/`)](#️-architecture-invariant-gates-tests_guardrails)
  - [Key Enforced Invariants](#key-enforced-invariants)
  - [Updating Regenerable Baselines](#updating-regenerable-baselines)
- [💅 Code Quality & Standards](#-code-quality--standards)
  - [Formatting & Linting (Ruff)](#formatting--linting-ruff)
  - [Type Checking (Mypy)](#type-checking-mypy)
  - [Pre-Commit Hooks](#pre-commit-hooks)
  - [Pre-Commit & CI Checklist](#pre-commit--ci-checklist)
- [📦 Dependencies & Version Caps](#-dependencies--version-caps)
- [📬 Pull Request Process & Quality Expectations](#-pull-request-process--quality-expectations)
  - [PR Submission Checklist](#pr-submission-checklist)
  - [Review Process & Automated Checks](#review-process--automated-checks)
  - [Commit Message Conventions](#commit-message-conventions)
- [🤖 Guidelines for AI Coding Agents](#-guidelines-for-ai-coding-agents)
- [📚 Documentation Index](#-documentation-index)

---

## 🚀 Quick Start (Contributor Environment)

### Canonical Install (`uv`)

The project uses [`uv`](https://docs.astral.sh/uv/) to manage locked dependencies via `uv.lock`. This guarantees deterministic development and test environments across platforms.

```bash
uv sync --frozen --extra browser --extra dev --extra markdown
source .venv/bin/activate
uv run playwright install chromium
pre-commit install
```

> **Why these flags and extras?**
> - `--frozen` enforces the exact versions pinned in `uv.lock` and fails fast if dependencies drift.
> - The `browser` extra is required for the local test suite because unit tests import and mock `playwright.sync_api`.
> - The `dev` extra supplies pytest, ruff, mypy, pre-commit, and vcrpy.
> - The `markdown` extra supplies `markdownify` for source text transformations.

### Fallback Install (`pip`)

If `uv` is unavailable on your system, you can use standard virtual environments with `pip`:

```bash
python -m venv .venv && source .venv/bin/activate
pip install -e ".[all]"   # [all] = browser + dev + markdown + mcp + server (no cookies; see installation.md)
playwright install chromium
pre-commit install
```

> 📘 For advanced setups, OS-specific dependencies, headless server options, and platform troubleshooting, consult [docs/installation.md](docs/installation.md).

---

## 🏛️ Architecture & Core Principles

### Module Layout

The codebase follows a strict separation of concerns:

```
src/notebooklm/
├── __init__.py           # Public exports (NotebookLMClient, types, error taxonomy)
├── client.py             # NotebookLMClient composition root
├── auth.py               # Public authentication facade
├── types.py              # Dataclasses and high-level type models
├── _app/                 # Transport-neutral business logic shared by CLI, MCP, and REST
├── _runtime/             # Lifecycle supervision, loop affinity, call leases
├── _web/                 # Google batchexecute RPC protocol and HTTP transport
│   ├── transport/        # Kernel, middleware, auth refresh, and session coordination
│   ├── wire/             # Codecs, encoders/decoders, positional mapping
│   └── *.py              # Feature implementations (notebooks, sources, artifacts, notes)
├── _android/             # Android bearer-gRPC / protobuf transport and codecs
├── rpc/                  # Raw batchexecute RPC facade and RPCMethod constants
├── cli/                  # Command-line interface Click commands and CLI services
├── mcp/                  # FastMCP server adapter (optional mcp extra)
└── server/               # FastAPI REST adapter (optional server extra)
```

### Layer Responsibilities & Boundary Rules

1. **Underscore Prefixes for Internal Modules**: Implementation modules use `_` prefixes (e.g. `_notebooks.py`, `_sources.py`, `_app/`, `_runtime/`). Public consumers import from `notebooklm` or `notebooklm.types`.
2. **Strict CLI Boundaries**: The `cli/` module must import only public surfaces (`notebooklm`, `notebooklm.types`, `notebooklm.auth`) and designated `_app/` workflows. Direct imports from `notebooklm._*` or `notebooklm.rpc.*` are rejected by boundary tests (`test_cli_boundary.py`).
3. **No Facade Reach-In**: Private service classes and feature adapters must not reach upward into facade modules or instantiate sibling facades directly. Dependencies must be passed as narrow protocols or constructor arguments.
4. **No Raw Positional Indexing**: Batchexecute responses are positional JSON. Raw chained indexing (e.g., `payload[0][1][3]`) is prohibited outside sanctioned codecs in `_web/wire/` and `_web/rows/`. All wire mappings are verified against protobuf tags (`test_wire_contract.py`).
5. **Constructor Injection over Monkeypatching**: Avoid dynamic monkeypatching on `NotebookLMClient` or private modules. Inject dependencies and test fakes through constructors or factories in `tests/_fixtures/` ([ADR-0007](docs/adr/0007-test-monkeypatch-policy.md)).

### Architectural Decision Records (ADRs)

Any contribution altering the architectural shape of the codebase must include or amend an Architectural Decision Record in `docs/adr/`. See [docs/adr/README.md](docs/adr/README.md) for conventions and index.

---

## 🧪 Testing Guidelines & Test Tiers

### Test Tiers

The test suite is partitioned into three distinct tiers based on network and authentication requirements:

| Tier | Location | Scope | Network | Auth Required |
|---|---|---|---|---|
| **Unit** | `tests/unit/` | Pure logic, data models, Click commands, encoder/decoders, mock-driven (`pytest-httpx`) API flows | None (Mocked) | None |
| **Integration** | `tests/integration/` | End-to-end RPC encoder/decoder roundtrips using recorded VCR cassettes (`tests/cassettes/web/`) | None (Replayed) | None |
| **E2E** | `tests/e2e/` | Live round-trips against Google NotebookLM services | Real Network | Yes (`notebooklm login`) |

### Running Tests

```bash
uv run pytest tests/unit
uv run pytest tests/integration
uv run pytest tests/e2e -m e2e        # requires auth
```

### Fast Feedback Loop (Skipping `repo_lint`)

A subset of unit tests are repo-wide audit / release-gate checks (cassette shape lint, public-surface scans, CI-script audits, doc-sync guards) that scan many files and add ~30–45s to the local `tests/unit tests/integration` loop. They're marked `@pytest.mark.repo_lint` so you can opt out while iterating:

```bash
# Fast feedback loop — drops repo_lint audits (~40s savings).
uv run pytest tests/unit tests/integration -m "not repo_lint"

# Run only the repo_lint audits (what you'd typically skip above).
uv run pytest tests/unit tests/integration -m "repo_lint"
```

Run the full suite (including `repo_lint`) before pushing. You can also run `make gates` to execute the AST/path/contract guards.

### Testing with Optional Adapters (`mcp`, `server`, `android`)

Tests for optional adapters use `importorskip`:
- **MCP Tests** (`tests/unit/mcp/`, `tests/integration/mcp_vcr/`) require `--extra mcp`.
- **REST Server Tests** (`tests/server/`) require `--extra server`.
- **Android Backend Tests** require `--extra android`.

To test all adapters locally:
```bash
uv sync --frozen --extra browser --extra dev --extra markdown --extra mcp --extra server --extra impersonate
uv run pytest -n auto --dist loadgroup
```

### VCR Cassettes & Fixture Management

- Cassettes for `batchexecute` Web RPC live in `tests/cassettes/web/`.
- Cassettes automatically sanitize and scrub sensitive tokens, email addresses, and session cookies via `tests/vcr_config.py`.
- New cassettes are recorded with:
  ```bash
  NOTEBOOKLM_VCR_RECORD=1 uv run pytest tests/integration/test_vcr_*.py -v
  ```
  *(Requires an authenticated session and setting `NOTEBOOKLM_GENERATION_NOTEBOOK_ID`)*.

### Coverage Expectations

- The repository maintains a **90% code coverage threshold** (`--cov-fail-under=90`).
- Ensure all new public methods, CLI subcommands, and branch conditions are covered with unit or integration tests.

---

## 🛡️ Architecture Invariant Gates (`tests/_guardrails/`)

### Key Enforced Invariants

Custom pytest guardrails in `tests/_guardrails/` enforce architectural invariants:
- **`test_cli_boundary.py`**: Ensures the CLI does not import internal modules or raw RPC methods.
- **`test_no_raw_positional_rpc_indexing.py`**: Disallows unprotected positional array accesses.
- **`test_rpc_method_ids_only_in_types.py`**: Enforces that all obfuscated method IDs reside solely in `src/notebooklm/rpc/types.py`.
- **`test_wire_contract.py`**: Validates JSON wire indices against Android protobuf tag mappings.
- **`test_no_forbidden_monkeypatches.py`**: Enforces constructor injection and prevents string-target monkeypatching.
- **`test_module_size_ratchet.py`**: Prevents runaway module sizes via a shrink-only LOC ceiling.

### Updating Regenerable Baselines

When intentionally modifying public API surfaces (e.g. exports in `notebooklm.types.__all__` or CLI options), committed baseline files will trigger test failures in `test_baseline_matches_committed_file`.

Regenerate baselines in the same PR:
```bash
python scripts/regen_baselines.py
git diff tests/fixtures/baselines tests/fixtures/cli_contract_baseline.json
```

---

## 💅 Code Quality & Standards

### Formatting & Linting (Ruff)

This project uses **Ruff** for linting and code formatting:
- **Target Version**: Python 3.10+
- **Line Length**: 100 characters
- **Quotes**: Double quotes (`"`)
- **Indentation**: 4 spaces

```bash
# Check for lint issues
ruff check .

# Auto-fix lint issues
ruff check --fix .

# Check formatting
ruff format --check .

# Apply formatting
ruff format .
```

### Type Checking (Mypy)

The codebase is fully typed:
```bash
uv run mypy src/notebooklm --ignore-missing-imports
```

### Pre-Commit Hooks

Pre-commit runs formatting, linting, and credential checks before code is committed:
```bash
pre-commit install                              # one-time, after the canonical install
pre-commit run --all-files                      # manual run on the whole tree (matches the CI lint gate)
```

> **Caveat:** if `pre-commit install` errors with `Cowardly refusing to install hooks with core.hooksPath set`, your git is configured to use a custom hooks directory (common with Husky / nx / shared dev configs). Workaround: `git config --unset core.hooksPath` then re-run `pre-commit install`, or run `pre-commit run --all-files` manually before each commit.

### Pre-Commit & CI Checklist

Before committing or opening a PR, run the full verification one-liner:

```bash
uv run ruff format --check . && \
    uv run ruff check . && \
    uv run mypy src/notebooklm --ignore-missing-imports && \
    uv run pytest --cov=src/notebooklm --cov-report=term-missing --cov-fail-under=90
```

---

## 📦 Dependencies & Version Caps

Every runtime and `[project.optional-dependencies]` entry in `pyproject.toml` must have an upper bound — typically `<currentmajor + 1` (or `<currentminor + 1` for pre-1.0 packages like `httpx`). The bound protects downstream installs from a breaking new release that lands before we have time to test it.

When you bump a cap (e.g. moving `pytest>=8.0,<10` to `pytest>=8.0,<11`):

1. Run `uv lock --refresh` and `uv sync --frozen --extra browser --extra dev --extra markdown` locally.
2. Run the full pre-commit one-liner above.
3. Mention the upgrade rationale in the PR description.

The `dependency-audit` workflow runs `pip-audit --strict --require-hashes` against the locked env on every push to `main` and nightly as a hard gate.

---

## 📬 Pull Request Process & Quality Expectations

### PR Submission Checklist

1. **Branching**: Create a clean feature branch from `main` (e.g., `feat/my-feature` or `fix/issue-description`).
2. **Reference an Issue**: PRs should link to an existing issue (`Closes #123` or `Fixes #123`) or clearly describe the problem being solved.
3. **No Duplicates**: Check existing open PRs before submitting.
4. **AI-Assisted Contributions**: Welcome, but the submitter must review, understand, and thoroughly test the code locally before submitting.
5. **Local Verification**: Ensure all tests pass, linting and formatting are clean, type checking succeeds, and code coverage stays at or above 90%.

### Review Process & Automated Checks

- Every PR triggers continuous integration workflows across supported Python versions (3.10–3.14) and operating systems (Ubuntu, macOS, Windows).
- Address all review comments and automated feedback. Unresolved threads block merge.
- Ensure the PR reaches a clean merge state (`mergeStateStatus: CLEAN`).

### Commit Message Conventions

Follow the existing conventional commit style:
- `feat(cli): add --format option to export command`
- `fix(rpc): update artifact polling response decoder`
- `refactor(auth): simplify cookie persistence adapter`
- `docs(readme): expand MCP quickstart examples`
- `test(unit): add coverage for mind map generation errors`

---

## 🤖 Guidelines for AI Coding Agents

All AI agents (Claude, Gemini, Codex, Antigravity, etc.) must follow these rules when working in this repository:

1. **No Root Rule**: Never create `.md` files in the repository root unless explicitly instructed by the user.
2. **Modify, Don't Fork**: Edit existing files; never create `FILE_v2.md`, `FILE_REFERENCE.md`, or duplicate copies.
3. **Scratchpad Protocol**: All analysis, investigation logs, and intermediate work go in `docs/scratch/` with date prefix: `YYYY-MM-DD-<context>.md`.
4. **Consolidation First**: Before creating new docs, search for existing related docs and update them instead.
5. **Protected Sections**: Never modify content between `<!-- PROTECTED: ... -->` and `<!-- END PROTECTED -->` or `# PROTECTED` markers unless explicitly instructed by the user.
6. **Prefer `--json` Output**: Pass `--json` and explicit notebook IDs instead of relying on interactive session state.

---

## 📚 Documentation Index

- **[README.md](README.md)**: Main project overview, features, and quickstart
- **[docs/installation.md](docs/installation.md)**: Canonical install guide (personas, extras, platform notes)
- **[docs/development.md](docs/development.md)**: Architecture, testing, and VCR cassette practices
- **[docs/architecture.md](docs/architecture.md)**: System design, layers, and protocol maps
- **[docs/cli-reference.md](docs/cli-reference.md)**: Complete reference for all CLI commands
- **[docs/cli-exit-codes.md](docs/cli-exit-codes.md)**: CLI exit-code convention (binding contract for scripts/CI)
- **[docs/python-api.md](docs/python-api.md)**: Async client reference, error types, and classes
- **[docs/mcp-guide.md](docs/mcp-guide.md)**: MCP server setup, tools, and troubleshooting
- **[docs/troubleshooting.md](docs/troubleshooting.md)**: Common issues and solutions
- **[docs/adr/](docs/adr/)**: Architectural Decision Records (ADRs)
- **[SECURITY.md](SECURITY.md)**: Security and vulnerability reporting policy
