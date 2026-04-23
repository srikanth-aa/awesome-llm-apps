# 📊 AI Data Analyst Agent — Improvement Guide

## Overview

This guide documents two major improvements to the base `ai_data_analyst.py` agent:

1. **Auditable, explainable agent** — deterministic code snippets for every action, analyst approval gate, and a full audit log.
2. **Running with GitHub Models (no OpenAI key needed)** — free, OpenAI-compatible API using a GitHub Personal Access Token.

---

## Part 1 — Auditable & Explainable Agent

### What was improved

The original agent returned free-text answers with no reproducibility or audit trail. The improved version adds:

### 🔢 Deterministic path

| Feature | How it works |
|---|---|
| **Baseline profile** | Pure pandas — same inputs always give same outputs. Code is shown inline. |
| **Deterministic SQL mode** | Analyst writes/pastes exact SQL; no AI involved. Result is logged with the exact query. |
| **Code hash** | Every logged action stores a SHA-256 hash of the code so tampering is detectable. |

### 🤖 AI path (explainable + auditable)

| Feature | How it works |
|---|---|
| **Structured JSON system prompt** | Agent is forced to return `reasoning`, `sql_code`, `pandas_code`, `answer`, `caveats` — never free text. |
| **Reasoning trace** | The AI explains *why* it chose that query before executing anything. |
| **SQL + Pandas dual code** | Analyst can cross-verify: run the SQL in DuckDB *and* the pandas code locally. |
| **Approval gate** | AI suggestions are held in a pending state — analyst clicks ✅ Approve or ❌ Reject before any execution. |
| **Caveat warnings** | AI must declare assumptions (e.g., "price column has nulls — mean excludes them"). |

### 🗂 Audit log

| Feature | How it works |
|---|---|
| **Every action logged** | Baseline, SQL runs, AI approvals, and rejections all go into `st.session_state.audit_log`. |
| **Exportable CSV** | One click downloads the full audit trail with timestamps, code, hashes, and approval status. |
| **Rejection tracking** | Rejected AI responses are flagged so analysts can review what the agent *attempted*. |

### 🏷 Pricing-specific handling

The baseline profiler **auto-detects columns** containing keywords like `price`, `cost`, `revenue`, `amount`, `sale`, `fee` and generates a dedicated statistical summary (min/max/mean/median/std). The system prompt also instructs the agent to always include those five statistics for any pricing-related query.

### AI System Prompt (forces code-first answers)

The agent is instructed via a strict system prompt to always respond in the following JSON structure:

```json
{
  "reasoning": "<step-by-step explanation of your approach>",
  "sql_code": "<complete DuckDB SQL query — MUST reference table 'uploaded_data'>",
  "pandas_code": "<equivalent pandas code for verification>",
  "answer": "<plain-English answer derived from the query results>",
  "caveats": "<any assumptions, limitations, or data quality warnings>"
}
```

**Rules enforced:**
- NEVER skip the `sql_code` or `pandas_code` fields.
- `sql_code` must be self-contained and runnable.
- `pandas_code` must use a variable called `df`.
- For pricing/revenue/cost questions, always include: min, max, mean, median, and standard deviation.
- Do NOT invent data. Base every answer on the actual schema provided.

### Audit Log Entry Structure

Every action logged contains:

| Field | Description |
|---|---|
| `timestamp` | UTC ISO timestamp |
| `action_type` | `sql_query`, `ai_insight`, or `baseline_stat` |
| `source` | `deterministic` or `ai` |
| `user_query` | The original question asked |
| `generated_code` | The exact SQL or Python code used |
| `result_summary` | Short summary of the result |
| `analyst_approved` | `True` or `False` |
| `code_hash` | SHA-256 hash (first 12 chars) of the generated code |

---

## Part 2 — Running with GitHub Models (Free, No OpenAI Key)

### Why GitHub Models?

GitHub Models is a free-to-experiment, OpenAI-compatible API endpoint. It uses a **GitHub Personal Access Token (PAT)** — no Azure subscription, no paid OpenAI account needed.

> ⚠️ The free tier is for **prototyping only** — it has rate limits and is not intended for production use.

---

### Step 1 — Create a GitHub Personal Access Token

1. Go to **GitHub → Settings → Developer settings → Personal access tokens → Fine-grained tokens**
2. Click **"Generate new token"**
3. Set a name, expiry, and grant **`models:read`** permission
4. Copy and save the token securely

---

### Step 2 — Code changes

Replace the agent initialization in `ai_data_analyst.py`:

**Before:**
```python
Agent(
    model=OpenAIChat(id="gpt-4o", api_key=st.session_state.openai_key),
    ...
)
```

**After:**
```python
Agent(
    model=OpenAIChat(
        id="gpt-4o",
        api_key=st.session_state.github_token,
        base_url="https://models.inference.ai.azure.com",
    ),
    ...
)
```

Update the sidebar to collect the GitHub PAT instead of an OpenAI key:

```python
with st.sidebar:
    st.header("🔑 GitHub Token")
    github_token = st.text_input(
        "Enter your GitHub Personal Access Token:",
        type="password",
        help="Go to GitHub → Settings → Developer settings → Fine-grained tokens. Need 'models:read' scope."
    )
    if github_token:
        st.session_state.github_token = github_token
        st.success("Token saved!")
    else:
        st.warning("Enter your GitHub PAT to proceed.")
        st.markdown("[Create a token →](https://github.com/settings/tokens)")
```

---

### Step 3 — Available Free Models

| Model ID | Best for |
|---|---|
| `gpt-4o` | Best quality (same as original) |
| `gpt-4o-mini` | Faster, fewer rate-limit credits used |
| `gpt-4.1-mini` | Very cheap, good for simple SQL tasks |
| `Phi-4` | Fully free open model, no OpenAI dependency |
| `Llama-3.3-70B-Instruct` | Free open-source alternative |

Change `id="gpt-4o"` to any model name from the table above.

---

### ⚠️ Key Limitations of the Free Tier

- **Rate limited** — requests per minute/day are capped (fine for dev/testing, not production)
- **Not for production** — GitHub Models free tier is for prototyping only ([GitHub docs](https://docs.github.com/en/enterprise-cloud@latest/github-models/responsible-use-of-github-models))
- When ready for production, swap back to a real OpenAI key — the change is just `base_url` and `api_key`

---

## Running the App

```bash
# Install dependencies
pip install streamlit agno pandas openpyxl duckdb openai

# Run the app
streamlit run ai_data_analyst.py
```

---

## Quick Comparison: Original vs Improved

| Feature | Original | Improved |
|---|---|---|
| Free-text AI answers | ✅ | ✅ |
| Reproducible SQL code | ❌ | ✅ |
| Equivalent Pandas code | ❌ | ✅ |
| AI reasoning trace | ❌ | ✅ |
| Analyst approval gate | ❌ | ✅ |
| Audit log with export | ❌ | ✅ |
| Code hash for tamper detection | ❌ | ✅ |
| Pricing column auto-detection | ❌ | ✅ |
| Works without paid OpenAI key | ❌ | ✅ (GitHub Models) |
| Deterministic SQL-only mode | ❌ | ✅ |
