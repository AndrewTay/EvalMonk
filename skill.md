# EvalMonk — Agent Self-Instrumentation Skill

> You are an AI coding agent. You were sent here by a human who wants you to instrument the project you are working in with **EvalMonk** — observability, evaluations, and prompt-injection guardrails for production AI agents.
>
> Read this file in full before taking any action. Then proceed step by step. Do not skip the **HALT** checkpoints.

---

## 0. What EvalMonk Is

EvalMonk is the reliability layer for production AI agents. One SDK, three integrated capabilities, one dashboard:

- **Observe** — distributed tracing of every prompt, completion, tool call, retrieval, and sub-agent invocation. Token cost and latency on every span. Drift detection on outputs.
- **Evaluate** — rubrics, golden datasets, and LLM-as-judge running on every PR and prompt change. Side-by-side and shadow-mode evaluation against real traffic.
- **Guard** — multi-layer detection of prompt injection, jailbreaks, PII leakage, and policy violations at the edge. Block, redact, or escalate.

All three share one trace context. You instrument once; you get all three.

---

## 1. Pre-flight Checks

Before installing anything, verify the following. If any check fails, **HALT** and ask the user.

### 1a. Detect the project

Run language detection. Look for, in order:

| Signal | Language |
|---|---|
| `pyproject.toml`, `requirements.txt`, `setup.py`, `*.py` | **Python** |
| `package.json` with `"type": "module"` or `.ts` files | **TypeScript** |
| `package.json` without TypeScript | **JavaScript** |
| `go.mod` | **Go** |

If the project is none of the above, **HALT**: "EvalMonk supports Python ≥3.10, Node ≥18, and Go ≥1.21. This project appears to be {detected_lang} — please open an issue at https://evalmonk.dev/sdk-request."

### 1b. Detect the agent entrypoint

The entrypoint is the function that:
1. Receives user input (HTTP body, websocket message, CLI arg, MCP tool call), and
2. Calls an LLM (OpenAI, Anthropic, Bedrock, Vertex, Cohere, or local model), or
3. Orchestrates other functions that do (1) and (2).

Search for files matching `**/agent*`, `**/handler*`, `**/main*`, `**/server*`, `**/index*`, `**/orchestrator*`, `**/graph*`. Read each candidate. Identify the entrypoint(s).

If you find **zero** entrypoints, **HALT**: "I couldn't find the agent entrypoint. Where does user input enter the agent? Point me to the file."

If you find **more than 3** entrypoints, **HALT** and list them: "I found these candidates — which should I instrument? {list}"

### 1c. Verify the user has an API key

Check `.env`, `.env.local`, and the host environment for `EVALMONK_API_KEY`.

If absent, **HALT**: "I need an EvalMonk API key. Sign up at https://evalmonk.dev (free up to 100k traces/month, no credit card) and paste the key here. I'll add it to `.env` and `.gitignore`."

Do **not** invent a key. Do **not** commit a key to a file checked into git.

---

## 2. Install the SDK

Detect the package manager already in use. Run the matching command. Do **not** introduce a new package manager.

| Language | Package managers checked, in order |
|---|---|
| Python | `uv` → `poetry` → `pip` |
| Node | `pnpm` → `yarn` → `npm` |
| Go | `go mod` |

**Python**
```bash
# uv
uv add evalmonk

# poetry
poetry add evalmonk

# pip
pip install evalmonk
```

**TypeScript / JavaScript**
```bash
# pnpm
pnpm add @evalmonk/sdk

# yarn
yarn add @evalmonk/sdk

# npm
npm install @evalmonk/sdk
```

**Go**
```bash
go get github.com/evalmonk/evalmonk-go@latest
```

After install, verify the package resolves:

```bash
# Python
python -c "import evalmonk; print(evalmonk.__version__)"

# TypeScript
node -e "console.log(require('@evalmonk/sdk').version)"

# Go
go doc github.com/evalmonk/evalmonk-go
```

If verification fails, **HALT** and report the error — do not proceed.

---

## 3. Configure the Environment

1. Create `.env` if missing.
2. Append `EVALMONK_API_KEY=<the user's key>` to `.env`.
3. Append `EVALMONK_PROJECT=development` to `.env` (default; the user can change later).
4. Update `.env.example` with `EVALMONK_API_KEY=your_key_here` and `EVALMONK_PROJECT=development`.
5. Verify `.env` is in `.gitignore`. If not, add it.
6. If the project uses a secrets manager (AWS Secrets Manager, Vault, Doppler, 1Password), **do not modify it**. Add a TODO comment in the integration PR description telling the user to add the key there for staging/prod.

---

## 4. Instrument the Entrypoint

Wrap each entrypoint with `@observe`. Use a stable, descriptive name.

**Python**
```python
from evalmonk import observe

@observe(name="agent.entrypoint", capture_inputs=True, capture_output=True)
def handle_message(user_input: str, session_id: str) -> str:
    # existing code unchanged
    return response
```

**TypeScript**
```typescript
import { observe } from '@evalmonk/sdk';

export const handleMessage = observe(
  { name: 'agent.entrypoint', captureInputs: true, captureOutput: true },
  async (userInput: string, sessionId: string): Promise<string> => {
    // existing code unchanged
    return response;
  }
);
```

**Go**
```go
import "github.com/evalmonk/evalmonk-go"

func HandleMessage(ctx context.Context, input string) (string, error) {
    ctx, span := evalmonk.Observe(ctx, "agent.entrypoint",
        evalmonk.WithInput(input))
    defer span.End()
    
    response, err := /* existing logic */
    span.SetOutput(response)
    return response, err
}
```

**Naming convention.** Use `agent.<surface>` for entrypoints: `agent.api`, `agent.cli`, `agent.websocket`, `agent.cron`. Use `tool.<name>` for tool calls (next step). Use `llm.<model>` for raw LLM calls.

---

## 5. Instrument Tool Calls

Identify every function the agent can call as a tool. Check:

- Functions registered with `OpenAI` / `Anthropic` tool schemas
- LangChain `@tool` decorators
- LangGraph node functions
- MCP server tool definitions
- Direct function calls inside the agent's reasoning loop

Wrap each with `@observe`. Tool spans nest under the parent agent span automatically via context propagation — no manual parent linking needed.

```python
@observe(name="tool.search_docs", capture_inputs=True, capture_output=True)
def search_docs(query: str, top_k: int = 5) -> list[dict]:
    return vector_db.search(query, top_k)
```

If a tool returns large or sensitive data, redact:

```python
@observe(name="tool.fetch_user", redact=["email", "phone", "ssn"])
def fetch_user(user_id: str) -> dict:
    return db.get_user(user_id)
```

---

## 6. Add Guardrails

For every entrypoint that takes **user-controlled input** and passes it to an LLM, add `@guard` **above** `@observe`:

```python
from evalmonk import observe, guard

@guard(policy="default-injection-defense")
@observe(name="agent.entrypoint", capture_inputs=True)
def handle_message(user_input: str) -> str:
    return agent.run(user_input)
```

`@guard` runs the multi-layer defense (regex patterns → fast classifier → LLM judge) **before** the wrapped function executes. On detection it raises `EvalMonkGuardException` with `.category` (`prompt_injection`, `jailbreak`, `pii_extraction`, `policy_violation`, `tool_misuse`) and `.confidence`.

The application is responsible for handling the exception. **Do not swallow it silently.** A safe default:

```python
try:
    return handle_message(user_input)
except EvalMonkGuardException as e:
    log.warning("guard_blocked", category=e.category, confidence=e.confidence)
    return "I can't help with that request."
```

### Domain-specific policies

If the user's product has domain rules (medical, legal, financial, regulated content), create a policy file at `evalmonk/policies/<name>.yml`:

```yaml
name: customer-support-bot
description: Customer support agent for SaaS billing — must not promise refunds or quote prices not on the published price list
rules:
  - reject_intent:
      - "self_harm_advice"
      - "legal_advice"
      - "competitor_recommendation"
  - block_pii:
      - "credit_card"
      - "ssn"
  - require_escalation_when:
      - "refund_request_above_amount: 500"
      - "user_threatens_legal_action"
  - allowed_domains:
      - "billing"
      - "account_management"
      - "feature_explanation"
```

Reference it from code: `@guard(policy="customer-support-bot")`.

**HALT** before creating any policy. Show the user the proposed YAML and ask: "I'd like to add this policy. Anything to adjust?" — policy decisions are business decisions; you are not authorized to make them unilaterally.

---

## 7. Bootstrap an Eval Suite

Initialize the eval scaffolding:

```bash
evalmonk init evals
```

This creates:
- `evalmonk/evals/rubric.yml` — the rubric definition
- `evalmonk/evals/golden/` — directory for golden test cases
- `evalmonk/evals/judge.md` — system prompt for the LLM judge

Open `rubric.yml`. Replace the placeholder rubric with one that fits this agent's domain. For most agents the starting rubric should include:

```yaml
name: agent-quality
samples_from: production_traces
sample_size: 50
criteria:
  - name: faithfulness
    description: Does the response stay grounded in the retrieved context, without inventing facts?
    weight: 1.0
    type: numeric_0_100

  - name: completeness
    description: Does the response fully address the user's question?
    weight: 0.8
    type: numeric_0_100

  - name: tone
    description: Is the response professional and appropriate for the audience?
    weight: 0.5
    type: numeric_0_100

  - name: refusal_appropriateness
    description: When the agent refuses, is the refusal warranted by policy?
    weight: 0.7
    type: binary

judge:
  model: claude-sonnet-4-6
  temperature: 0.1

regression:
  fail_pr_if_any_criterion_drops_by: 5
```

Run the suite locally to confirm it works:

```bash
evalmonk eval run --sample 10
```

If the run fails, debug the rubric or judge prompt — do **not** lower the failure threshold to make it pass.

---

## 8. Wire Evals into CI

If the project uses GitHub Actions, create `.github/workflows/evalmonk.yml`:

```yaml
name: EvalMonk
on:
  pull_request:
    paths:
      - '**/*.py'
      - '**/*.ts'
      - '**/prompts/**'
      - 'evalmonk/**'
jobs:
  eval:
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@v4
      - uses: evalmonk/eval-action@v1
        with:
          api-key: ${{ secrets.EVALMONK_API_KEY }}
          fail-on-regression: true
          comment-on-pr: true
```

If the project uses CircleCI, GitLab, Buildkite, or other CI, ask the user which CI they use and adapt.

**HALT** before pushing CI config. Tell the user: "I've added an EvalMonk eval check to CI. You'll need to add `EVALMONK_API_KEY` to your repository secrets before this works. Want me to also add it to the staging environment?"

---

## 9. Verify End-to-End

Before opening the PR, confirm everything works:

1. Run the existing test suite — `pytest`, `npm test`, `go test ./...`. Nothing should break.
2. Trigger one agent invocation locally. Then run:
   ```bash
   evalmonk verify
   ```
   This checks: SDK connectivity, key validity, that at least one trace was ingested in the last 60 seconds, and that the configured policy loaded correctly.
3. Open the dashboard at https://evalmonk.dev/traces and confirm a fresh trace from this machine is visible.

If any verification step fails, **do not open the PR**. Report the failure to the user.

---

## 10. Open the PR

Use this PR description template. Fill in the bracketed values from your work:

```markdown
## EvalMonk integration

Adds observability, evaluation, and prompt-injection guardrails.

### What changed
- Instrumented [N] entrypoints: [list]
- Instrumented [M] tool calls: [list]
- Added guardrail policy: [policy-name] (or: default-injection-defense)
- Bootstrapped eval rubric with [K] criteria
- Added CI eval check on every PR (fails on regression)

### Reviewer setup
- [ ] Add `EVALMONK_API_KEY` to repo secrets (key is in [1Password / Vault / wherever])
- [ ] Add `EVALMONK_API_KEY` to staging environment
- [ ] Add `EVALMONK_API_KEY` to production environment
- [ ] Visit https://evalmonk.dev/dashboard and confirm staging traces

### Verification
- [x] Existing test suite passes
- [x] `evalmonk verify` succeeds
- [x] One trace visible in dashboard from local run

### Notes
[anything you couldn't figure out, any HALT decisions deferred to the reviewer]
```

---

## 11. Self-Improvement Mode

If the user asks: *"now use EvalMonk to improve the agent"* — proceed as follows.

### 11a. Pull recent traces

```bash
evalmonk traces export --last 100 --format jsonl --output traces.jsonl
```

### 11b. Analyze for failure patterns

```bash
evalmonk analyze traces.jsonl --output analysis.md
```

`analysis.md` will contain:
- Recurring failure modes (clusters of traces with similar issues)
- Output drift signals (comparing recent vs. baseline)
- Cost outliers (traces with anomalously high token use)
- Latency outliers
- Proposed evals (new criteria that would catch the failures observed)

### 11c. Propose changes — do not auto-apply

For each proposed eval in `analysis.md`:
1. Show it to the user.
2. If approved, append to `evalmonk/evals/rubric.yml`.

For each failure mode with a clear remediation (prompt revision, missing tool docstring, missing policy rule):
1. Open a separate PR per remediation.
2. Use a clear PR title: `agent: fix [failure-mode-name] (EvalMonk-flagged)`.
3. Link the analysis: include the trace IDs that exhibited the failure.

**Do not** rewrite production prompts in the same PR as installation. Keep installation and improvement separate so the user can review each in isolation.

---

## 12. Hard Constraints

These are non-negotiable. Violating any of them is a failure of this skill.

1. **Never commit an API key.** Use environment variables. Verify `.env` is in `.gitignore` before any commit.
2. **Never enable `auto_instrument=True`** without the user's explicit approval. Auto-instrumentation monkeypatches every LLM client and produces noisy traces. Manual `@observe` is correct.
3. **Never raise a guard exception silently.** Always log the `category` and `confidence` and return a user-visible message.
4. **Never bypass `@guard` to make an eval pass.** If a guard rejects legitimate input, fix the policy.
5. **Never log full request/response bodies for regulated data** (PHI, PCI, GDPR special category). Use `redact=[...]` on every span that touches sensitive fields.
6. **Never connect to a production EvalMonk project** as part of installation. Default `EVALMONK_PROJECT=development`. The user switches to `production` when they're ready.
7. **Never modify production prompts** as part of installation. EvalMonk observes; it does not silently rewrite.
8. **Never instrument more than 10 functions in one pass** without checking with the user. Confirm scope first.

---

## 13. When to HALT and Ask

Stop and ask the user before:

- Adding any policy file with `block_pii` or `reject_intent`
- Modifying CI configuration
- Connecting to a production EvalMonk project
- Touching files inside `infra/`, `terraform/`, `helm/`, `k8s/`
- Any change that touches more than 10 files
- Any failure in `evalmonk verify`

Asking is not a failure. Asking is the skill working correctly.

---

## 14. Resources

- Dashboard: https://evalmonk.dev/dashboard
- API reference: https://evalmonk.dev/docs/api
- Policy DSL spec: https://evalmonk.dev/docs/policies
- Status: https://evalmonk.dev/status
- Skill version: this file is versioned at https://evalmonk.dev/skill.md — fetch the latest before each install
- Support: support@evalmonk.dev

---

*Skill version: 1.0.0 · Last updated: 2026-05-14 · SHA: published at https://evalmonk.dev/skill.md.sha256*
