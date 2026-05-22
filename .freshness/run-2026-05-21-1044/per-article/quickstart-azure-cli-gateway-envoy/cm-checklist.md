# Content Mentor Checklist — `quickstart-azure-cli-gateway-envoy.md`

> Dry-run, quick mode, pre-publish. Article not yet live, so feedback/telemetry phases are skipped.

## Setup
1. In VS Code, open the file:
   ```
   code C:\github\anyscale-on-azure-docs-pr\quickstart-azure-cli-gateway-envoy.md
   ```
2. Open Copilot Chat (Ctrl+Alt+I or the Copilot icon).
3. Confirm `@content-mentor` is available — type `@content-mentor /help` and check it responds.

## Phase 1 — Metadata
4. `@content-mentor /suggestTitle`
   - **Current:** `Quickstart: Deploy Anyscale on Azure with Envoy Gateway`
   - Review the 10 alternatives. Note any you'd swap to. **Do not Accept yet** (propose-only).
   - Paste the top 3 you'd consider into `report.md` under "Phase 1 — Title".
5. `@content-mentor /suggestDescription`
   - **Current:** `Deploy your first Anyscale cloud on Azure Kubernetes Service using the Azure CLI and the Envoy Gateway controller. Configure your subscription, create an AKS cluster, and register through the Azure portal.`
   - Same treatment — top 3 alternates into `report.md`.
6. `@content-mentor /suggestCustomerIntent`
   - Article has no `ms.customer-intent` field yet. Capture the suggested statement and add it as a proposal in `report.md` under "Phase 1 — Customer Intent".

## Phase 2 — Style
7. `@content-mentor /suggestQuickEdits`
   - Review every suggestion. For each one you'd take, **don't Accept yet** — just note "✅ accept" or "❌ skip" in `report.md` under "Phase 2 — Quick Edits".

## Phase 3 — Markdown Hygiene
8. `@content-mentor /autoFixMarkdown`
   - This one **mutates the file**. Tree is clean — safe to run.
   - After it completes, run `git diff --stat` in the terminal and screenshot/paste the result into `report.md` under "Phase 3 — autoFixMarkdown diff".
   - If diff looks correct, leave changes staged but **don't commit** — the orchestrator will commit them in a labeled group later. If diff looks wrong, `git checkout -- quickstart-azure-cli-gateway-envoy.md` to revert.

## Phase 4 — SEO
9. `@content-mentor /suggestSEO`
   - Output is structured JSON. Copy the full JSON array.
   - Save it to: `C:\github\anyscale-on-azure-docs-pr\.freshness\run-2026-05-21-1044\per-article\quickstart-azure-cli-gateway-envoy\seo-deltas.json`
   - The orchestrator will validate every `originalText` string-matches and build an accept/reject worklist.

## Phase 5 — Links
10. `@content-mentor /validateLinks`
    - Walks through every link interactively. Capture the summary (counts: ok / broken / redirected) into `report.md` under "Phase 5 — Link Validation".

## Phase 6 — Images (lightweight)
11. Skip `/suggestAltText` for now. Orchestrator confirmed all 7 `:::image` blocks already have `alt-text=`. If you want a deeper pass on alt-text *quality*, run `/suggestAltText` on any image that looks underwhelming.

## When done
12. Reply in Clawpilot:
    ```
    done envoy-quickstart
    ```
    The orchestrator will then read `seo-deltas.json`, finalize `report.md`, commit, and propose the PR command.

## Bail-out
At any point, reply `abort` and I'll discard the run folder and `git checkout -- .` to restore. Your committed `wb-moved` state is preserved.
