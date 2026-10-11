# Workflow AI Toolkit

Three small Python projects that work together: convert legacy XML workflows to Power Automate-style JSON, map actions with a schema-constrained LLM and score it, and expose the converter to AI agents over MCP.

## 1. `wfconvert` - XML to flow JSON conversion engine
- Parses an XML workflow with `xml.etree.ElementTree` and generates flow JSON (triggers, chained actions with `runAfter`, If/Else branches).
- Rule table maps action types (SetVariable, SendEmail, LogToHistory, Pause, Condition) to modern equivalents.
- `expressions.py` converts legacy tokens and comparisons (`{ItemProperty:Days} > 5`) into expressions like `@greater(triggerBody()?['Days'], 5)`.
- Anything without a rule is **not dropped**: it becomes a TODO placeholder and is listed in `report.manual_review`.

```bash
python -m wfconvert.converter samples/leave_approval.xml
```

> The input format is a simplified, Nintex-like XML written for this project, not the full real NWF schema. Extending it to real files means adding rules to `RULES`.

## 2. `llm_eval` - structured output + eval harness
- `schema.py`: Pydantic schema the model must follow (`ActionMapping`).
- `classifier.py`: sends the JSON schema in the prompt, validates the reply, retries once with the validation error, then falls back to `NONE / needs_review`.
- `evaluate.py`: scores schema validity on first try, accuracy, review rate and per-class accuracy against `golden.json`.

```bash
python -m llm_eval.evaluate --mock                 # offline keyword baseline
GEMINI_API_KEY=... python -m llm_eval.evaluate --gemini   # real model
ANTHROPIC_API_KEY=... python -m llm_eval.evaluate --claude  # real model (set CLAUDE_MODEL to choose)
```

The `--mock` run is a keyword **baseline**, not a model. Use it to compare against real model results. The Gemini adapter is not covered by automated tests (needs a key); set `GEMINI_MODEL` if the default name is unavailable.

## 3. `mcp_server` - MCP tools for agents
Exposes `convert_workflow` and `list_supported` over stdio.

```bash
python -m mcp_server.server
```

## 4. `webhook` - signed webhook receiver (FastAPI)
`POST /webhook/convert` accepts `{"event_id": ..., "xml": ...}`. The sender signs the raw body with HMAC-SHA256 using a shared secret and sends it in `X-Signature`; wrong or missing signatures get 401 (checked with a constant-time compare). Invalid payloads get 422, bad XML gets 400.

```bash
WEBHOOK_SECRET=change-me uvicorn webhook.app:app --reload
```

## 5. `azure_auth` - Microsoft Entra ID (Azure AD) token helper
`get_access_token()` uses `DefaultAzureCredential`, so it works with an `az login` session on your laptop, a service principal (`AZURE_TENANT_ID`, `AZURE_CLIENT_ID`, `AZURE_CLIENT_SECRET`) or a managed identity in Azure. `describe_token()` reads audience, tenant and expiry from the JWT (it does not verify the signature).

```bash
az login
python -m azure_auth.token
```

> The automated tests use a fake credential. Run it once against your own tenant to confirm the real sign-in works.

## Setup and tests
```bash
pip install -r requirements.txt
python -m pytest -q
```
