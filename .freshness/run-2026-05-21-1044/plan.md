# Freshness Run — 2026-05-21 10:44 CDT

## Scope
- **Mode:** quick (dry-run)
- **Article:** `quickstart-azure-cli-gateway-envoy.md`
- **Repo:** `MicrosoftDocs/anyscale-on-azure-docs-pr` (private/PR repo — article not yet published)
- **Branch:** `wb-moved` (current); freshness work will go to `freshness/2026-05-21-envoy-quickstart`

## Article snapshot
| Field | Value |
|---|---|
| ms.date | 04/29/2026 (22 days old) |
| ms.topic | quickstart |
| ms.service | azure-kubernetes-service |
| author / ms.author | kaysieyu / kaysieyu |
| Lines | 427 |
| `:::image` blocks | 7 |
| Code fences | 46 |
| External `https://` links | 8 |
| Internal `.md` links | 9 |
| `<a>` "click here" patterns | 0 |
| Trailing whitespace | 0 |
| Recent committers | Douglas Strodtman, Tim Dwyer, mbender-ms |
| In TOC | Yes (`toc.yml:6`) |

## Pre-publish caveat (important)
This article lives in a `-pr` repo with a Public Preview include marker — **it's not live on learn.microsoft.com yet**. That means several freshness signals will be unavailable and the relevant phases are skipped or stubbed:

| Phase | Normally requires | This run |
|---|---|---|
| 0 — UUF customer feedback | Live article URL | **Skipped** (no public traffic) |
| 5 — Broken-link telemetry (ADX) | Live article URL | **Skipped** (no telemetry) |
| 8 — Engagement | Live article URL | **Skipped** |
| 11 — Insight-driven refresh | Batch / docset | N/A (single article) |

Phases that **do** apply pre-publish:
- 1 metadata, 2 style, 3 markdown hygiene, 4 SEO, 5 link *validation* (HTTP check), 6 images (alt-text), 7 code accuracy, 9 security/SFI, 10 IA, 13 PR, 14 report.

## Plan
1. Phase 0 — triage signals (this file).
2. Phase 1 — metadata validation: agentic check + propose suggestions via CM.
3. Phase 2 — style edits via CM (`/suggestQuickEdits`).
4. Phase 3 — Markdown auto-fix via CM (`/autoFixMarkdown`).
5. Phase 4 — SEO via CM (`/suggestSEO`), capture JSON deltas.
6. Phase 5 — agentic link probe + CM `/validateLinks`.
7. Phase 6 (light) — confirm alt-text exists on all 7 `:::image` blocks. Skip `/suggestAltText` unless gaps.
8. Phase 9 (light) — regex scan for tenant IDs / secrets.
9. Phase 13 — branch + commit groups; user runs `/createPR`.
10. Phase 14 — report + friction notes.

## Auto-apply vs propose-only
| Phase | Mode |
|---|---|
| ms.date update | **Skipped** (article only 22 days old, not stale) |
| autoFixMarkdown | Propose-only (run, review diff, accept manually) |
| Style edits | Propose-only |
| SEO edits | Propose-only |
| All others | Propose-only |

## Outputs
- `per-article/quickstart-azure-cli-gateway-envoy/report.md`
- `per-article/quickstart-azure-cli-gateway-envoy/cm-checklist.md`
- `per-article/quickstart-azure-cli-gateway-envoy/feedback.json` (stub)
- `per-article/quickstart-azure-cli-gateway-envoy/agentic-findings.md` (Phase 5/6/9 results)
- `dashboard.md` (single-row)
- `followups.json`
