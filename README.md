# EvoAgent PR Reviewer

EvoAgent is a production-oriented pull-request review service that analyzes unified diffs and returns structured findings, fixes, and test recommendations.

## Features

- GitHub `pull_request` webhooks for `opened`, `reopened`, and `synchronize`
- Three honestly reported modes: `rules-only`, `hybrid`, and `agentic`
- Four real LLM roles in agentic mode: Planner, Security, Correctness/Reliability, and Critic
- JSON APIs, Markdown reports, a web console, task dashboard, and Prometheus metrics
- SQLite for local use; PostgreSQL and Redis for production
- Persistent checkpoints, execution budgets, resumable tasks, and a bounded agent loop
- Tool schema validation, structured observations, context compression, and layered memory
- Tenant/repository isolation, login, RBAC, immutable audit logs, and signed skill manifests
- Feedback-driven prompt and declarative-skill evaluation, activation, and rollback
- Automatic-fix test gates, canary releases, shadow traffic, OpenTelemetry, and alerts

## Quick start

EvoAgent requires Python 3.11. Install dependencies and configure a local administrator in the same PowerShell window:

```powershell
python -m pip install -r requirements.txt

$bytes = New-Object byte[] 32
[Security.Cryptography.RandomNumberGenerator]::Create().GetBytes($bytes)
$env:EVOAGENT_AUTH_REQUIRED = 'true'
$env:EVOAGENT_AUTH_SECRET = [Convert]::ToBase64String($bytes)
$env:EVOAGENT_BOOTSTRAP_ADMIN_USERNAME = 'admin'
$env:EVOAGENT_BOOTSTRAP_ADMIN_PASSWORD = '<password-of-at-least-10-characters>'

python -m evoagent
```

Never use example placeholders as passwords or secrets. Environment variables apply only to the current PowerShell process and its children. Restart EvoAgent after changing configuration. The bootstrap administrator is created only if the username does not exist, so restarting never overwrites an existing password.

The service listens on `127.0.0.1:8080` by default. Open `http://127.0.0.1:8080/`, log in, and use the returned Bearer token for API calls:

```powershell
$session = Invoke-RestMethod -Method Post -Uri http://127.0.0.1:8080/v1/auth/login `
  -ContentType 'application/json' `
  -Body (@{username='admin'; password='<your-password>'} | ConvertTo-Json)
$headers = @{Authorization="Bearer $($session.access_token)"}

Invoke-RestMethod -Method Post -Uri http://127.0.0.1:8080/v1/reviews `
  -Headers $headers -ContentType 'application/json' `
  -Body (@{
    repository = 'demo/api'
    pull_request = 12
    mode = 'rules-only'
    diff = "diff --git a/app.py b/app.py`n--- a/app.py`n+++ b/app.py`n@@ -1 +1,2 @@`n+password = 'secret'`n+eval(user_input)"
  } | ConvertTo-Json)
```

Query a task, download its report, or run the tests:

```powershell
Invoke-RestMethod -Headers $headers http://127.0.0.1:8080/v1/tasks/<task-id>
Invoke-WebRequest -Headers $headers http://127.0.0.1:8080/v1/tasks/<task-id>/report
python -m unittest discover -s tests -v
```

## Model configuration

Requests may use `rules-only` (scanners), `hybrid` (scanners plus one LLM), or `agentic` (four independent LLM roles). The latter two explicitly fall back to `rules-only` when no model is configured.

Reports include actual model/tool calls, input/output tokens, cost, latency, failures, and the complete agent trace. Cost comes from the provider's `usage.cost` or `EVOAGENT_LLM_*_COST_PER_MILLION`; it remains zero when pricing is not configured. Pass an absolute `repository_root` to enable full-repository search, symbols/call relationships, tests, configuration and permission inspection, AST analysis, Git history, and installed static analyzers.

DeepSeek:

```powershell
$env:EVOAGENT_LLM_PROVIDER = 'deepseek'
$env:EVOAGENT_DEEPSEEK_API_KEY = '<deepseek-api-key>'
python -m evoagent
```

OpenRouter's rate-limited free DeepSeek route:

```powershell
$env:EVOAGENT_LLM_PROVIDER = 'openrouter-deepseek-free'
$env:EVOAGENT_OPENROUTER_API_KEY = '<openrouter-api-key>'
python -m evoagent
```

If that free model is removed, set `EVOAGENT_LLM_MODEL` to another `:free` model, or select provider `openrouter-free`. For any OpenAI Chat Completions-compatible endpoint:

```powershell
$env:EVOAGENT_LLM_PROVIDER = 'custom'
$env:EVOAGENT_LLM_BASE_URL = 'https://example.com/v1'
$env:EVOAGENT_LLM_API_KEY = '<token>'
$env:EVOAGENT_LLM_MODEL = '<model-name>'
```

Secrets are read from environment variables and must not be committed. EvoAgent reads root `.env` (and supports `evoagent/.env`) at startup; system variables take precedence. The root `.env` is ignored by Git.

## Evaluation and evolution

The service maintains validation and hidden regression sets. Caller-provided “regression scores” are never accepted as release evidence. EvoAgent replays current and candidate prompts on identical diffs, computes precision, recall, F1, severity accuracy, high-risk recall, clean-sample accuracy, and execution success, then applies minimum-improvement and hidden-set non-regression gates. Missing models or undersized datasets produce a `deferred` candidate.

All runs, versions, aggregate metrics, SHA-256 fingerprints, activation decisions, and rollback points are persisted. Hidden examples remain undisclosed. Add immutable, versioned cases through `POST /v1/evaluation/cases`; `split` accepts `train`, `validation`, or `holdout`. With a model configured, `POST /v1/evolution/auto` clusters failed traces and generates structured prompt, few-shot, routing, tool-policy, and budget candidates. Passing candidates become `shadow_ready` and cannot modify production Python code.

Run the reproducible controlled offline proof with:

```powershell
python scripts/run_prompt_evolution_proof.py
```

Output is written to `output/prompt-evolution-proof/`. This synthetic-controlled experiment demonstrates behavior change and hidden-set gating; it is not evidence of LLM weight improvement or production performance on public PRs.

Skill evolution uses a separate version chain and produces host-independent declarative artifacts, never executable Python. `POST /v1/skill-evolution/auto` builds candidates from unresolved tenant feedback. Missed findings should include `finding.rule_id`, `severity`, `path`, `line`, and preferably `finding.evidence`. Candidates activate only after validation improvement and protected-metric/holdout non-regression. Rejected or undersampled versions remain auditable but inactive.

Skill names must begin with `evolved-`. Rules support constrained literal matching only on added lines—never arbitrary code, regular expressions, or host permissions. Gates are configured with `EVOAGENT_EVAL_MIN_CASES`, `EVOAGENT_EVAL_MIN_HOLDOUT_CASES`, `EVOAGENT_EVAL_MAX_CASES`, `EVOAGENT_EVAL_MIN_IMPROVEMENT`, and `EVOAGENT_EVAL_MAX_METRIC_REGRESSION`.

## GitHub webhook

EvoAgent uses a repository webhook, a public tunnel, and an optional fine-grained PAT; no GitHub App is required.

```text
GitHub pull-request event
        |
        v
https://<public-domain>/webhooks/github
        | public tunnel
        v
http://127.0.0.1:8080/webhooks/github
        |
        v
EvoAgent creates an asynchronous review task
```

Configure the secret and optional PAT:

```powershell
$webhookBytes = New-Object byte[] 32
[Security.Cryptography.RandomNumberGenerator]::Create().GetBytes($webhookBytes)
$env:EVOAGENT_GITHUB_WEBHOOK_SECRET = [Convert]::ToBase64String($webhookBytes)
$env:EVOAGENT_GITHUB_TOKEN = '<fine-grained-PAT>'
$env:EVOAGENT_AUTO_POST_REVIEW = 'true' # Disabled by default
python -m evoagent
```

The webhook secret verifies GitHub's HMAC-SHA256 signature and must differ from `EVOAGENT_AUTH_SECRET`. Grant the PAT only to required repositories with minimum permissions:

- Private PR diffs: `Contents: Read`, `Pull requests: Read`
- Review comments: `Pull requests: Read and write`
- Fix branches: `Contents: Read and write`, `Pull requests: Read and write`

No PAT is needed for public repositories without comments or fixes. Create a public HTTPS tunnel:

```powershell
cloudflared tunnel --url http://127.0.0.1:8080
# Or: ngrok http 8080
```

Keep both processes running. Quick tunnels expose the console and API, so authentication must remain enabled. For long-running deployments, expose only `/webhooks/github` and optionally `/health` through a reverse proxy.

In the GitHub repository, open **Settings → Webhooks → Add webhook** and set:

- **Payload URL:** `https://<public-domain>/webhooks/github`
- **Content type:** `application/json`
- **Secret:** `EVOAGENT_GITHUB_WEBHOOK_SECRET`
- **SSL verification:** enabled
- **Events:** only **Pull requests**
- **Active:** enabled

Verify local and public health endpoints, then create, reopen, or update a PR:

```powershell
Invoke-RestMethod http://127.0.0.1:8080/health
Invoke-RestMethod https://<public-domain>/health
```

Recent Deliveries should show `202`, and the task should appear in the console. Results are posted to the PR only when `EVOAGENT_AUTO_POST_REVIEW=true`. Automatic fixes cover deterministic rules only and always use a new `evoagent/fix-pr-*` branch.

## Production mode

```powershell
Copy-Item .env.example .env
docker compose up --build
```

Compose starts PostgreSQL, Redis, and EvoAgent. Without those services, EvoAgent falls back to SQLite and an in-process thread queue for local use.

## API

| Method | Path | Description |
|---|---|---|
| `GET` | `/health` | Health check |
| `POST` | `/v1/auth/login` | Log in and receive a tenant-bound Bearer token |
| `POST` | `/v1/reviews` | Create a synchronous review |
| `POST` | `/v1/reviews?async=true` | Create an asynchronous review |
| `GET` | `/v1/tasks/{id}` | Get status, trace, and report |
| `GET` | `/v1/tasks/{id}/report` | Get the Markdown report |
| `GET/POST` | `/v1/tasks/{id}/feedback` | Read or submit feedback |
| `POST` | `/v1/tasks/{id}/fix` | Create an automatic-fix branch and commit |
| `POST` | `/v1/tasks/{id}/cancel` | Request cancellation |
| `POST` | `/v1/tasks/{id}/resume` | Resume from the latest checkpoint |
| `POST` | `/webhooks/github` | Receive GitHub PR webhooks |
| `POST` | `/v1/skills/reload` | Reload dynamic skills |
| `GET/POST` | `/v1/evaluation/cases` | Read or add evaluation cases |
| `GET/POST` | `/v1/evolution/status`, `/v1/evolution/auto` | Inspect gates or evolve prompts |
| `GET` | `/v1/evolution/runs` | Read persisted evaluation runs |
| `GET/POST` | `/v1/skill-evolution/*` | Inspect, propose, activate, or evolve skills |
| `GET` | `/metrics` | Prometheus metrics |
| `GET` | `/api/alerts` | Read tenant alerts |
| `GET` | `/api/audit` | Read tenant audit events |
| `GET` | `/api/queue/dead-letters` | Read dead-letter tasks |
| `POST` | `/v1/queue/dead-letters/replay` | Replay dead-letter tasks |
| `GET/POST` | `/api/deployments/llm-review`, `/v1/deployments/llm-review` | Configure canary/shadow rollout |

The default maximum `diff` is 1 MiB; tasks default to eight steps and 120 seconds. See `.env.example` for all options. Completed tasks accept `false_positive`, `missed_issue`, or `bad_fix` feedback. Include `finding.rule_id`, `path`, and `line` for missed issues whenever possible.

## Architecture

```text
HTTP / GitHub Webhook
        |
        v
 ReviewService -- TaskStore (SQLite / PostgreSQL)
        |
        v
 ReviewHarness (runtime / checkpoint / resume / budget / trace)
        |
        +-- DiffParser
        +-- Redis Streams / ACK / lease / retry / DLQ
        +-- ContextManager (token budget / iterative compression)
        +-- MemoryManager (working / episodic / semantic / expiry)
        +-- ModeRouter
              +-- rules-only: scanners -> gates
              +-- hybrid: scanner + one LLM -> gates
              +-- agentic: Planner + Security + Correctness/Reliability + Critic -> gates
```

The in-project `AgentRuntime` controls `PENDING → PLANNING → EXECUTING → REVIEWING → SUCCESS`. Only `hybrid` and `agentic` enter a model decision loop. Each role has its own prompt, context, tool allowlist, token/time budget, and trace. The tool layer handles repository search, symbols, tests, configuration, ASTs, Git, static analysis, and isolated execution. Gates enforce format, evidence, confidence, and release eligibility. Reports persist actual calls, token use, and cost.
