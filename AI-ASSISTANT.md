# ForgeEstimate Assist — AI estimating assistant

## 1. Architecture

```
 Browser (dist/ForgeEstimate.html — no keys, no provider URLs)
 ┌───────────────────────────────────────────────────────────────────────────┐
 │ ui/assist.js   Assist tray (docked under the bridge, Ctrl+J)              │
 │   │ build context ─► ai/context.js   read-only projection of Store.state  │
 │   │ POST /api/assist {message, context, history}                          │
 │   ▼                                                                       │
 │ ai/protocol.js  validate envelope (same schema as server: protocol.json)  │
 │ ai/tools.js     run read-only analyses with the Engine (explain, find,    │
 │                 quote status, revision diff, QA scan, what-if)            │
 │ ai/actions.js   resolve each proposed action against live data, price     │
 │                 the change set on a COPY with Engine.recalc, show diff    │
 │   │ estimator ticks actions + confirms assumptions ─► Approve             │
 │   ▼                                                                       │
 │ Store.commit('AI: …', st => Ops.*(…))  ← the same Ops functions the       │
 │   worksheet / pricing / dialogs use → one undo step, audit log, recalc,   │
 │   autosave, history. No AI-specific state is persisted.                   │
 └───────────────────────────────────────────────────────────────────────────┘
            │ HTTPS (optionally Bearer FORGE_AI_ACCESS_TOKEN)
 ┌──────────▼────────────────────────────────────────────────────────────────┐
 │ server/serve.py (stdlib)  or  server/lambda_handler.py (serverless)       │
 │ forge_ai/service.py  validate request size/shape → provider → validate    │
 │                      envelope (one repair retry) → JSON                   │
 │ forge_ai/prompt.py   system prompt (rules: never invent prices/rates…)    │
 │ forge_ai/providers/  anthropic.py · openai.py (any OpenAI-compatible      │
 │                      endpoint) · rules.py (deterministic, no model)       │
 └───────────────────────────────────────────────────────────────────────────┘
```

Design rules enforced in code, not just in the prompt:

* **The model picks, the engine computes.** The model returns structured actions/tool calls. Every number shown —
  cost impact, what-if result, explanation, QA finding — comes from `Engine` in the browser.
* **Proposals are validated twice** (server `protocol.py`, browser `protocol.js`, one shared `protocol.json`), then
  resolved against live data (`AIActions.resolve`): unknown items/assemblies/crews/resources/quotes, duplicate codes,
  out-of-range values and unpriced components are rejected with a reason shown to the user.
* **Provenance on every rate.** A component is either linked to a library resource (library rate wins and the model's
  rate is discarded), sourced from an existing estimate value/quote, or labelled `assumption`/`derived`. Labelled values
  must be ticked individually before Approve enables, and are stored with `sourceSnapshot.basis = AI_ASSUMPTION |
  AI_DERIVED`, the note, the estimator's name and date.
* **Nothing is applied without approval.** Preview runs on a clone. Apply checks an estimate fingerprint; if the
  estimate changed since the preview, the preview re-prices live and a stale apply is refused.
* **Rules vs. AI kept apart.** Deterministic validation (`Engine.issues`, calculation tests) is labelled
  “Validation rules · deterministic”. Heuristics and model observations are labelled with severity + confidence and
  never block export.

## 2. Live AI setup (backend required)

The HTML never talks to an AI provider. It calls `/api/assist` on the same origin (or the endpoint set in
**Assistant connection…**). Without a backend the tray still works for the built-in engine analyses.

Local / on-prem (Python 3.10+, no dependencies):

```
cd app
python3 build.py
ANTHROPIC_API_KEY=… python3 server/serve.py --host 0.0.0.0 --port 8787     # or OPENAI_API_KEY=…
# open http://<host>:8787/
```

| Variable | Purpose |
|---|---|
| `FORGE_AI_PROVIDER` | `anthropic`, `openai` or `rules`. Default: whichever key is present, else `rules`. |
| `ANTHROPIC_API_KEY` / `OPENAI_API_KEY` | Provider secret — server side only. |
| `FORGE_AI_MODEL` | Model id (default `claude-sonnet-4-5` / `gpt-4.1`). |
| `FORGE_AI_BASE_URL` | Alternate endpoint (Azure OpenAI, gateway, self-hosted OpenAI-compatible model). |
| `FORGE_AI_MAX_TOKENS` | Anthropic output cap (default 4096). |
| `FORGE_AI_ACCESS_TOKEN` | If set, clients must send `Authorization: Bearer <token>` (entered in Assistant connection…). |
| `FORGE_AI_ALLOWED_ORIGINS` | Comma list of origins allowed cross-origin (when the HTML is opened from a file or another host). |
| `FORGE_AI_RATE_PER_MIN` | Per-client request limit (default 30). |

Serverless: deploy `server/` with handler `lambda_handler.lambda_handler` (AWS Lambda + function URL / API Gateway;
same env vars, key in Secrets Manager/env). Point **Assistant connection…** at the function URL.

Production notes: put it behind HTTPS and your SSO/reverse proxy; the estimate context is sent to the provider, so
choose a provider/contract that meets your data-handling requirements. Adding a provider = one class in
`forge_ai/providers/` implementing `complete()` and one line in `PROVIDERS`.

## 3. What the assistant can read

Per request the browser sends a compact, read-only context (`ai/context.js`): project header and data policies,
markup settings, totals, every bid item (code, section, name, qty, unit, production rate/method, crew, days, unit
cost, direct, bid, quote link, notes, components with type/resource/qty/unit/rate/source), indirects, alternates,
quotes (vendor, amount, date, freshness, exclusions, linked item), resources (code, rate, unit, date), crews (hourly
cost), assemblies, deterministic issues, revisions (names/dates/totals), the last 15 audit entries, and the UI context
(current sheet, selected item). Engine tools additionally compute item explanations, revision diffs and what-ifs
locally.

## 4. What the assistant can modify (only after approval)

| Action | Uses |
|---|---|
| `add_item_from_assembly` (qty, section, name) | `Ops.assemblyItem` + `Ops.insertItem` — same as Insert Assembly |
| `add_item` (manual item with components, crew/production) | `Ops.manualItem` + `Ops.insertItem` — same as Add Scope |
| `update_item` (qty, unit, prod, method, crew, name, section, code) | `Ops.setItemField` — same as worksheet cells |
| `update_component` (qty / rate with override reason) | `Ops.setComponentField` |
| `adjust_rates` (% on matching component rates in this estimate; library untouched) | `Ops.setComponentField` overrides |
| `delete_item` | `Ops.deleteItem` |
| `set_markup` (overhead/contingency/profit/bond %) | `Ops.setPct` — same as Pricing |
| `add_indirect` / `update_indirect` | `Ops.addIndirect` / `Ops.editIndirect` — same as Indirects |
| `add_alternate` (add/deduct, amount from an item’s bid/direct or explicit) | `Ops.addAlternate` |
| `add_note` | `Ops.addNote` |

It cannot change company libraries (resources, crews, assemblies), quotes, revisions, project settings, other
projects, or delete anything outside the active estimate. All approved changes are one undoable commit tagged
`AI assistant` in the audit log.

## 5. Tests

* `tests/test_ai_server.py` — 18 unit tests: protocol parsing/coercion/limits/unknown ops/provenance, request
  validation, repair retry, provider errors, provider selection, rules provider on all 13 example prompts, no secret
  strings in the built HTML.
* `tests/ai_e2e.py` — Chromium against `serve.py --provider rules`: 25 browser-side validator/resolver unit tests plus
  the full workflow (open/dock, context awareness, every example prompt, approve/undo/redo/persist, assumption gating,
  stale preview, what-if → change set, review separation, offline behaviour, tablet layout, no console errors).
* Existing suites unchanged: `tests/qa.py` (41 steps), `tests/smoke.py`, `tests/parity.py` (baseline parity).
