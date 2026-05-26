# Freshness Report — `quickstart-azure-cli-gateway-envoy.md`

**Run:** 2026-05-21 10:44 CDT — quick mode, dry-run
**Status:** ⏳ In progress (awaiting user-driven Content Mentor steps)

## Summary
- Article is healthy on autopilot checks (metadata, alt-text, SFI, mechanical Markdown).
- One recommendation already known: add `ms.customer-intent`.
- Style/SEO/link-validation depth depends on Content Mentor output — captured below as you run each command.

---

## Phase 0 — Triage ✅ done
See `feedback.json` (no UUF / ADO / GitHub issue signals — article is pre-publish) and `agentic-findings.md`.

## Phase 1 — Metadata
### Title
**Current:** `Quickstart: Deploy Anyscale on Azure with Envoy Gateway`
**CM /suggestTitle alternatives (paste top 3 here):**
1. _<paste>_
2. _<paste>_
3. _<paste>_
**Decision:** _<keep / swap to #N>_

### Description
**Current:** `Deploy your first Anyscale cloud on Azure Kubernetes Service using the Azure CLI and the Envoy Gateway controller. Configure your subscription, create an AKS cluster, and register through the Azure portal.`
**CM /suggestDescription alternatives:**
1. _<paste>_
2. _<paste>_
3. _<paste>_
**Decision:** _<keep / swap to #N>_

### Customer Intent (missing field)
**CM /suggestCustomerIntent output (paste here):**
> _<paste>_
**Recommended:** add as `ms.customer-intent` in frontmatter.

## Phase 2 — Style (Quick Edits)
**CM /suggestQuickEdits output — review and tag each:**

| # | Before | After | Decision |
|---|---|---|---|
| 1 | _<paste>_ | _<paste>_ | ✅ / ❌ |
| 2 | | | |
| 3 | | | |

## Phase 3 — Markdown Auto-Fix
**Pre-run `git diff --stat`:** _<paste>_
**Action taken:** kept staged / reverted (`git checkout`)

## Phase 4 — SEO
**File:** `seo-deltas.json` (paste raw output here)
**Score:** _<paste from CM output>_
**Orchestrator validation (filled in after `done`):** _<count of edits where originalText string-matches in file>_

## Phase 5 — Links
**CM /validateLinks summary:** _<paste counts: ok / broken / redirect>_
**Agent HTTP probe (orchestrator will fill in):** _<pending>_

## Phase 6 — Images
Skipped — agentic scan confirms all 7 images have alt-text. See `agentic-findings.md`.

## Phase 9 — Security
No SFI findings. See `agentic-findings.md`.

## Phase 13 — Commit / PR
- Branch (to be created): `freshness/2026-05-21-envoy-quickstart`
- Proposed commits (built after `done`):
  - `chore(freshness): add ms.customer-intent` _if accepted_
  - `style(freshness): markdown autofix via CM` _if accepted_
  - `docs(freshness): style + SEO edits per CM review` _accepted subset only_

## Friction notes (fill in as you go)
- _<what felt clunky?>_
- _<what worked smoothly?>_
- _<what should the skill auto-do but didn't?>_
