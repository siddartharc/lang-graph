# Local-First AI Dev Workstation (Windows 11 + Ollama + Claude Code)

A lightweight, local-first AI coding setup on Windows 11 built around **Ollama**, **Claude Code**, **LangGraph**, and **MCP**. Everything is tuned to run comfortably on an **8 GB RAM** machine with controlled agent autonomy—meaning the agent can inspect and plan freely, but always asks before making destructive or high-impact changes.

---

## Architecture Overview

```text
┌──────────────────────────────────────────────────────────┐
│                     Claude Code CLI                      │
│   Project Rules (CLAUDE.md) · Skills · Safe Permissions  │
└──────────────┬───────────────────────────┬───────────────┘
               │ (Anthropic API)           │ (HTTP MCP)
               ▼                           ▼
┌──────────────────────────────┐  ┌────────────────────────┐
│       Ollama (Local)         │  │     Context7 MCP       │
│  qwen3.5:4b-claude (Q4_K_M)  │  │  Up-to-date Library    │
│      16K Context Window      │  │     Documentation      │
└──────────────────────────────┘  └────────────────────────┘
```

### Stack Summary
- **Runtime:** Ollama (v0.14.0+ for Anthropic API compatibility)
- **Primary Model:** `qwen3.5:4b` with a custom 16K context Modelfile (`qwen3.5:4b-claude`)
- **Coding Agent:** Claude Code CLI (launched via an isolated PowerShell wrapper)
- **Docs Integration:** Context7 MCP *(note: remote service, so inference is local while doc lookups require internet)*
- **Python Environment:** Isolated `.venv` with LangChain and LangGraph

> **Hardware expectation:** A 4B model works well for Python scripts, explanations, and focused debugging on 8 GB RAM. Heavy multi-file refactors and deep system architecture still benefit from human guidance and smaller, scoped prompts.

---

## 1. Install & Configure Ollama

Open PowerShell and run the installer:

```powershell
irm [https://ollama.com/install.ps1](https://ollama.com/install.ps1) | iex
```

Add Ollama to your User `PATH` if it isn't picked up automatically:

```powershell
$ollamaPath = "$env:LOCALAPPDATA\Programs\Ollama"
$userPath = [Environment]::GetEnvironmentVariable("Path", "User")

if ($userPath -notlike "*$ollamaPath*") {
    [Environment]::SetEnvironmentVariable("Path", "$userPath;$ollamaPath", "User")
}

$env:Path += ";$ollamaPath"
```

Verify the installation (requires **Ollama v0.14.0+** for Claude Code compatibility):

```powershell
ollama --version
ollama list
ollama ps   # Shows active models and CPU/GPU memory usage
```

To upgrade an older version:

```powershell
winget upgrade --id Ollama.Ollama -e
# Or, if no upgrade is detected:
winget install --id Ollama.Ollama -e
```

*(Restart Ollama after upgrading.)*

### Pull Base Models

```powershell
ollama pull qwen3.5:4b
ollama pull llama3.2:1b
ollama pull llama3.2:latest
```

Quick sanity check (run `ollama ps` in a second terminal while active to inspect memory usage):

```powershell
ollama run qwen3.5:4b
```

---

## 2. Create a 16K-Context Custom Model

`qwen3.5:4b` advertises a 262K context window, which will quickly exhaust 8 GB of RAM. Create a dedicated model variant capped at 16,384 tokens:

```powershell
$ollamaConfigDir = "$env:USERPROFILE\ollama-config"
New-Item -ItemType Directory -Force $ollamaConfigDir | Out-Null
Set-Location $ollamaConfigDir

@'
FROM qwen3.5:4b
PARAMETER num_ctx 16384
'@ | Set-Content -Encoding utf8 .\Modelfile

ollama create qwen3.5:4b-claude -f .\Modelfile
ollama show qwen3.5:4b-claude
```

> If you hit out-of-memory (OOM) errors, lower `num_ctx` to `8192` and re-run `ollama create`.

---

## 3. Set Up the Python Project

Create the project directory, set up a virtual environment, and install the core dependencies:

```powershell
mkdir langchain-graph
cd langchain-graph

python -m venv .venv
.\.venv\Scripts\activate
```

Install packages:

```powershell
python -m pip install --upgrade pip
pip install langchain langchain-ollama langgraph python-dotenv
```

*(Use `deactivate` whenever you want to exit the virtual environment.)*

---

## 4. Install Claude Code & Local Launcher

Install the Claude Code CLI and add it to your User `PATH`:

```powershell
irm [https://claude.ai/install.ps1](https://claude.ai/install.ps1) | iex

$claudePath = "$env:USERPROFILE\.local\bin"
$userPath = [Environment]::GetEnvironmentVariable("Path", "User")

if ($userPath -notlike "*$claudePath*") {
    [Environment]::SetEnvironmentVariable("Path", "$userPath;$claudePath", "User")
}

$env:Path += ";$claudePath"
claude --version
```

You can test a quick connection directly through Ollama:

```powershell
ollama launch claude --model qwen3.5:4b
```

### Dedicated Local Launcher Script

Instead of setting `ANTHROPIC_BASE_URL` globally (which interferes if you ever use an actual Anthropic account), use a scoped PowerShell launcher that sets the variables for the session and restores them on exit.

```powershell
New-Item -ItemType Directory -Force .\scripts | Out-Null
```

Create `.\scripts\claude-local.ps1`:

```powershell
# Claude Code + Ollama local launcher
$ErrorActionPreference = "Stop"

# 1. Verify Ollama is running
try {
    $response = Invoke-RestMethod -Uri "http://localhost:11434/api/tags" -TimeoutSec 5
}
catch {
    Write-Host "Ollama is not responding. Start Ollama and try again." -ForegroundColor Red
    exit 1
}

# 2. Verify custom model exists
$model = $response.models |
    Where-Object { $_.name -like "qwen3.5:4b-claude*" } |
    Select-Object -First 1

if (-not $model) {
    Write-Host "Local model 'qwen3.5:4b-claude' not found." -ForegroundColor Red
    Write-Host "Run: ollama create qwen3.5:4b-claude -f `"$env:USERPROFILE\ollama-config\Modelfile`""
    exit 1
}

# 3. Preserve existing environment variables
$oldToken = $env:ANTHROPIC_AUTH_TOKEN
$oldKey = $env:ANTHROPIC_API_KEY
$oldBaseUrl = $env:ANTHROPIC_BASE_URL

try {
    $env:ANTHROPIC_AUTH_TOKEN = "ollama"
    $env:ANTHROPIC_API_KEY = ""
    $env:ANTHROPIC_BASE_URL = "http://localhost:11434"

    Write-Host "Starting Claude Code with local Ollama (qwen3.5:4b-claude)..." -ForegroundColor Green
    claude --model qwen3.5:4b-claude
}
finally {
    $env:ANTHROPIC_AUTH_TOKEN = $oldToken
    $env:ANTHROPIC_API_KEY = $oldKey
    $env:ANTHROPIC_BASE_URL = $oldBaseUrl
}
```

Launch with:

```powershell
.\scripts\claude-local.ps1
```

---

## 5. Permissions, Engineering Rules & `.gitignore`

### Safe Permissions (`.claude/settings.json`)

Keep `defaultMode` set to `default` (no `bypassPermissions` or unrestricted shell flags) and explicitly block access to secret files:

```powershell
New-Item -ItemType Directory -Force .\.claude | Out-Null
```

Create `.claude/settings.json`:

```json
{
  "$schema": "[https://json.schemastore.org/claude-code-settings.json](https://json.schemastore.org/claude-code-settings.json)",
  "permissions": {
    "defaultMode": "default",
    "deny": [
      "Read(./.env)",
      "Read(./.env.*)",
      "Read(./secrets/**)",
      "Read(./**/*.pem)",
      "Read(./**/*.key)"
    ]
  }
}
```

### Project Instructions (`CLAUDE.md`)

Create `CLAUDE.md` in the project root to define coding standards and approval boundaries:

```powershell
@'
# Project Engineering Instructions

## Role
Act as a senior Python software engineer and AI systems architect.

## Primary Technologies
- Python, FastAPI, Pytest
- LangChain, LangGraph, Ollama
- SQL and relational databases
- Git and GitHub

## Python Engineering Standards
- Follow PEP 8 and use type hints consistently.
- Prefer modular, maintainable code following SOLID principles.
- Avoid unnecessary abstractions.
- Handle errors explicitly; never hardcode credentials or API keys.

## AI & Agent Engineering
- Prefer LangGraph for stateful agent workflows; explain state transitions clearly.
- Use structured outputs where appropriate.
- Handle retries, timeouts, and failures explicitly; prevent infinite agent loops.
- Separate tools, prompts, nodes, and state definitions.

## System Design
When designing a system:
1. Clarify functional and non-functional requirements.
2. Estimate scale and traffic.
3. Define components, APIs, and data models.
4. Explain architectural trade-offs, scalability, and failure handling.

## Database Engineering
- Explain SQL queries before executing them and prefer parameterized queries.
- Never run destructive database operations without explicit approval.
- Consider indexing and query performance; never expose credentials.

## Debugging Workflow
1. Reproduce the issue and identify the root cause.
2. Explain the problem clearly.
3. Propose the smallest reliable fix.
4. Run relevant tests after approval and report actual results.

## Git & DevOps
- Inspect `git status` before modifying repository state.
- Never force push, overwrite uncommitted work, or touch production infrastructure without approval.

## Autonomy Boundaries
- Inspect and analyse freely within the project.
- Always ask before installing dependencies, deleting/overwriting files, modifying environment variables, writing to databases, committing/pushing, or deploying.
- Never bypass permission prompts.

## Communication
- Prefer practical Python examples and explain *why* a design choice fits.
- Never fabricate test results; admit uncertainty when context is missing.
'@ | Set-Content -Encoding utf8 .\CLAUDE.md
```

### `.gitignore`

```powershell
@'
# Environment & secrets
.env
.env.*
!.env.example
secrets/

# Python
__pycache__/
*.py[cod]
.venv/
venv/
.pytest_cache/
.mypy_cache/
.ruff_cache/

# Claude local config
.claude/settings.local.json

# Logs & IDEs
*.log
.vscode/
.idea/
'@ | Set-Content -Encoding utf8 .\.gitignore
```

---

## 6. MCP & CLI Integrations

Keep MCP servers minimal on 8 GB RAM—each connected server adds tool definitions to your context window. Start with **Context7** for up-to-date Python, LangChain, and LangGraph docs:

```powershell
claude mcp add --transport http context7 [https://mcp.context7.com/mcp](https://mcp.context7.com/mcp)
claude mcp list
```

Verify inside Claude Code using `/mcp`.

### Planned Integrations

| Area | Recommended Approach |
| :--- | :--- |
| Documentation | Context7 MCP *(configured)* |
| Git | Built-in Git CLI |
| GitHub | GitHub CLI (`gh`) |
| PostgreSQL | Read-only database MCP (dedicated read-only user) |
| SQLite | Local SQLite CLI/tooling |
| Browser Testing | Playwright MCP (add only when needed) |

Optional GitHub CLI setup:

```powershell
winget install --id GitHub.cli -e
gh auth login
gh auth status
```

---

## 7. Reusable Engineering Skills

Create three focused skills under `.claude/skills/`:

```powershell
New-Item -ItemType Directory -Force `
  .\.claude\skills\python-review, `
  .\.claude\skills\system-design, `
  .\.claude\skills\langgraph-review | Out-Null
```

<details>
<summary><strong>1. Python Code Review (<code>.claude/skills/python-review/SKILL.md</code>)</strong></summary>

```markdown
---
name: python-review
description: Review Python code for quality, performance, maintainability, and security.
---

# Python Code Review

Review the supplied code for:
1. PEP 8 compliance
2. Type safety and type hints
3. SOLID principles
4. Complexity and performance
5. Exception handling
6. Security vulnerabilities
7. Memory efficiency
8. Testability

Group findings by severity. For each issue, explain the problem, why it matters, and show the recommended fix. Do not modify files unless explicitly requested.
```

</details>

<details>
<summary><strong>2. System Design (<code>.claude/skills/system-design/SKILL.md</code>)</strong></summary>

```markdown
---
name: system-design
description: Design scalable backend and distributed systems.
---

# System Design

For every system design problem:
1. Gather functional and non-functional requirements.
2. Estimate scale.
3. Define APIs, database schemas, and core components.
4. Explain overall architecture, caching, queues, and async processing.
5. Discuss scalability, availability, fault tolerance, and bottlenecks.

Use Python examples where useful and highlight trade-offs rather than presenting a single universal design.
```

</details>

<details>
<summary><strong>3. LangGraph Review (<code>.claude/skills/langgraph-review/SKILL.md</code>)</strong></summary>

```markdown
---
name: langgraph-review
description: Review LangGraph agents, workflows, and stateful AI systems.
---

# LangGraph Review

Evaluate:
1. State schema design
2. Nodes and responsibilities
3. Conditional edges and graph transitions
4. Checkpointing and persistence
5. Tool integration, error handling, and retries
6. Human-in-the-loop checkpoints
7. Infinite-loop prevention and observability

Prefer modular graph architecture. Justify why LangGraph is needed over a simple chain or plain Python workflow, and avoid unnecessary agents.
```

</details>

---

## 8. Verification Checklist

Run these checks in a fresh PowerShell window from the project root:

```powershell
# 1. Check Ollama & API
ollama list
ollama ps
Invoke-RestMethod http://localhost:11434/api/tags

# 2. Test local inference
ollama run qwen3.5:4b-claude "Write a Python function to reverse a string."

# 3. Launch Claude Code
.\scripts\claude-local.ps1
```

Inside Claude Code, verify:
- `/status` — confirms the local Ollama backend and model.
- `/mcp` — confirms Context7 is connected.
- `/doctor` — checks for configuration warnings.

### Quick Agent Smoke Tests

1. **Read-only inspection:**
   > *"Inspect this repository. Explain its structure, Python entry point, dependencies, and any architectural issues. Do not modify any files."*
2. **Approval boundary check:**
   > *"Create a small Python calculator module with type hints, exception handling, and pytest tests. Show your proposed file structure first and wait for my approval before creating any files."*

---

## Daily Command Reference

| Task | Command |
| :--- | :--- |
| Start local Claude Code | `.\scripts\claude-local.ps1` |
| Activate Python venv | `.\.venv\Scripts\activate` |
| List local models | `ollama list` |
| Monitor RAM/VRAM usage | `ollama ps` |
| Check Claude setup | `claude doctor` |
| List MCP servers | `claude mcp list` |
| Resume last session | `claude --continue` |
| Plan before editing | `/plan` |
| Compact long context | `/compact` |
| Inspect permissions / MCPs | `/permissions` · `/mcp` |

### 8 GB RAM Tips
- Close memory-heavy apps (extra browser tabs, Electron apps) before running sessions.
- Keep only one Ollama model loaded at a time.
- Run `/compact` regularly during long coding sessions to stay within the 16K context budget.

---

## Next Steps
- Benchmark larger quantised coding models once CPU/GPU headroom is profiled.
- Add a read-only database MCP once the target database (PostgreSQL / SQLite) is finalised.
