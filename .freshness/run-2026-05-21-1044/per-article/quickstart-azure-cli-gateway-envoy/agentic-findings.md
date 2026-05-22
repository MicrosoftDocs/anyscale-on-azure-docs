# Agentic Findings — `quickstart-azure-cli-gateway-envoy.md`

Results from the *agent-only* scans the orchestrator runs without Content Mentor.

## Metadata (Phase 1)

| Field | Present | Value | Note |
|---|---|---|---|
| `title` | ✅ | `"Quickstart: Deploy Anyscale on Azure with Envoy Gateway"` | Has colon → correctly quoted |
| `description` | ✅ | 2 sentences, 33 words | Healthy length |
| `author` | ✅ | `kaysieyu` | |
| `ms.author` | ✅ | `kaysieyu` | |
| `ms.date` | ✅ | `04/29/2026` | 22 days old; **not** stale |
| `ms.service` | ✅ | `azure-kubernetes-service` | |
| `ms.topic` | ✅ | `quickstart` | |
| `ms.customer-intent` | ❌ | — | **Recommended add**: see CM `/suggestCustomerIntent` |
| `titleSuffix` | ❌ | — | Not strictly required for `-pr` repo; build pipeline may inject |

## Markdown hygiene (Phase 3 pre-scan)

| Check | Result |
|---|---|
| Trailing whitespace on lines | 0 |
| Hard line breaks (`  \n`) | none detected |
| "click here" / "click on" anti-pattern | none |
| Code fence count (matched pairs) | 46 fences — even count, balanced |
| Heading sentence-case | spot-checked OK (e.g. "Prerequisites and required tools", "Step 1: provision Azure resources") |
| Numbered-list-after-code-fence formatting | ⚠️ Possible issue near `max_retries: 1` block — closing fence may be mashed against the next list item. `/autoFixMarkdown` should resolve. |

## Link inventory (Phase 5 pre-scan)

| Type | Count |
|---|---|
| External `https://` | 8 |
| Internal `.md` | 9 |

External destinations (need HTTP probe — orchestrator will run during Phase 5 alongside `/validateLinks`):
- `portal.azure.com` (multiple)
- `kubernetes.io/docs/...`
- `kubernetes.io/releases/version-skew-policy/...`
- `helm.sh/docs/intro/install/`
- `docs.anyscale.com/reference/quickstart-cli`
- `www.anyscale.com/support`

Internal `.md` references to verify all exist in the repo:
- `Includes/anyscale-public-preview.md`
- `quickstart-azure-cli-ingress-nginx.md`
- `supported-regions.md`
- `support-model.md`
- `architecture.md`
- `networking.md`
- `identity-access.md`

(Confirmed all 7 targets exist in repo root via `git ls-files`.)

## Images (Phase 6 pre-scan)

7 `:::image` blocks, all under `media/quickstart/`. All have `alt-text=` populated:

| File | alt-text summary |
|---|---|
| `quickstart-anyscale-clouds-landing.png` | "Anyscale clouds page in the Azure portal showing a list of existing Anyscale cloud resources." |
| `quickstart-create-basics-filled.png` | "Basics tab with subscription, resource group, cloud name, region, and AKS cluster filled in." |
| `quickstart-create-infrastructure-settings.png` | "Infrastructure settings tab showing storage account..." |
| `quickstart-create-container-registry-settings.png` | "Container registry settings tab with ACR mode..." |
| `quickstart-create-support-plan.png` | "Support plan tab showing the support tier set to Enterprise Tier..." |
| `quickstart-create-tags.png` | "Tags tab showing name and value fields..." |
| `quickstart-review-submit-summary.png` | "Review + submit tab showing a summary..." |

All alt-text reads cleanly and describes content (not just filename echo). **No Phase 6 action required** unless `/suggestAltText` returns quality improvements.

## Security / SFI (Phase 9)

Regex scans:

| Pattern | Hits | Note |
|---|---|---|
| GUID (tenant/subscription) | 1 | `086bc555-6989-4362-ba30-fded273e432b` — this is the **Anyscale public service principal ID**, intentional and documented. No action. |
| JWT token shape | 0 | |
| `AKIA*` (AWS access key) | 0 | |
| Connection string fragments (`AccountKey=`, `SharedAccessKey=`) | 0 | |
| Internal hostnames (`*.redmond.corp.microsoft.com`, etc.) | 0 | |
| Bearer tokens | 0 | |
| `<your-...>` / `<my-...>` placeholders | several | Correctly used as user-replaceable values, not leaked secrets |

**No SFI findings.**

## Code accuracy (Phase 7 lightweight)

| Spot-check | Result |
|---|---|
| `az ad sp create --id <guid>` | Valid az CLI syntax |
| `az group create`, `az aks create` | Valid syntax |
| `helm install eg oci://docker.io/envoyproxy/gateway-helm --version v1.7.0` | Verify v1.7.0 is current GA — orchestrator will check helm registry during full mode |
| `Standard_D4s_v5` VM size | Valid, available in supported regions |
| Kubectl / Helm commands | Standard, no deprecation warnings spotted |
| YAML manifests (`envoyproxy.yaml`, `gatewayclass.yaml`, `gateway.yaml`) | Schema looks current for Gateway API v1 + EnvoyProxy v1alpha1 |

**Full `cm /validateAgainstCode` would deep-verify versions and command flags. Skipped in quick mode.**

## IA / Structure (Phase 10)

- Article is referenced in `toc.yml` at line 6.
- It pairs with `quickstart-azure-cli-ingress-nginx.md` via the `op_multi_selector` block — the partner article must be kept in lockstep on shared steps.
- No reorganization recommended.
