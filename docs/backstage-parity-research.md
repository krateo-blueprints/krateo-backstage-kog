# Backstage → Krateo parity: research and implementation plan

Research date: **2026-10-08**. All claims below are tagged:

- **[V]** verified by reading source / running code / fetching the artefact in this session
- **[P]** partially verified (read one layer, inferred the next from the same code path)
- **[U]** **unverified** — stated as a hypothesis, must be checked before anyone acts on it

Versions read:
- `krateo-platformops/oasgen-provider` @ `eac184c` (main, 2026-10-06) — local clone `/Users/diegobraga/krateo/krateo-oasgen-provider`
- `krateo-platformops/rest-dynamic-controller` @ `217ccd0` — local clone `/Users/diegobraga/krateo/krateo-rest-dynamic-controller`
- `krateoplatformops/frontend` @ `7fb2263` (2026-10-05) — local clone `/Users/diegobraga/krateo/frontend`
- `snowplow` monorepo clone `/Users/diegobraga/krateo/snowplow-monorepo` @ `b651f97`
- `backstage/backstage` @ `pushed_at 2026-10-08`, `backstage/community-plugins` (live `gh api` reads)

---

## 1. Executive summary

### Scale of the Backstage surface **[V]**

| Measure | Count | Source |
|---|---|---|
| Packages under `backstage/backstage/plugins/` | **160** | `gh api repos/backstage/backstage/contents/plugins` |
| Distinct core plugin *families* (catalog, scaffolder, techdocs, search, kubernetes, permission, notifications, signals, auth, devtools, home, org, api-docs, events, user-settings, proxy, gateway, mcp-actions, catalog-graph, catalog-import) | **~20** | same, after collapsing `-backend` / `-node` / `-react` / `-common` / `-module-*` siblings |
| Workspaces in `backstage/community-plugins/workspaces/` | **111** | `gh api .../contents/workspaces` |
| Plugin packages inside those workspaces | **253** | enumerated per workspace |
| Significant third-party plugins outside both repos (PagerDuty's own org, RoadieHQ's monorepo, …) | **~20+** | `gh api repos/PagerDuty/backstage-plugin`, `repos/RoadieHQ/roadie-backstage-plugins` |

So the honest headline is **~130 distinct integrations** (20 core families + 111 community workspaces), not 253 and not 160 — the package counts double- and triple-count the same integration.

### How the parity split falls out

Of the **40 integrations** I analysed in any depth (section 2):

- **Widgets only, no controller: 14.** Pure read-only dashboards (Grafana, Tech Radar, Cost Insights, SonarQube, Lighthouse read-path, code-coverage, …) and everything Krateo already owns natively (Kubernetes, Tekton, Argo Workflows, Flux, Topology, Kiali — these are CRDs, so Krateo reads them with a RESTAction against the apiserver and needs no generated controller at all).
- **Controller + widgets: 13.** Anything whose UI has a write button or whose objects should be GitOps-managed (GitHub, Jira, PagerDuty, Keycloak, ServiceNow, Azure DevOps, Harbor/Quay/Nexus, Sentry, Vault, Octopus, …).
- **Controller only: 0.** Every plugin is, by definition, a UI.
- **Neither / not applicable: 13.** Backstage's own machinery whose Krateo counterpart is a different mechanism entirely, not a plugin port — catalog→CompositionDefinitions+CRs, scaffolder→Builders/Forms, permissions→k8s RBAC+`userAccessFilter`, search, techdocs, notifications→sse-proxy bell, auth→authn.

### Top 5 oasgen-provider gaps (ordered by how much parity they unblock)

| # | Gap | Blocks | Evidence |
|---|---|---|---|
| **1** | **Swagger / OpenAPI 2.0 documents are rejected outright.** `Parse()` calls `BuildV3Model()` only. | **ArgoCD, Grafana, Argo Workflows, Harbor** — four of the highest-value integrations, dead at the front door | **[V]** ran the real ArgoCD spec through the real parser: `failed to build V3 model: unable to build openapi document, supplied spec is a different version (oas2). Try 'BuildV2Model()'`. Source: `go/oasgen-provider/internal/tools/oas2jsonschema/libopenapi_adapter.go:41` |
| **2** | **No credential-exchange / token-minting auth.** `authentication` admits exactly `basic`, `bearer`, `apiKey`-in-header, each a static value read from a Secret. | **ArgoCD** (`POST /api/v1/session` → short-lived token), **Jenkins** (CSRF crumb), **Jira/Confluence OAuth2**, **ServiceNow OAuth2**, **Vault login**, any OAuth2 client-credentials backend | **[V]** `configuration_builder.go:183-230` — `authMethodKey` is a closed 3-case switch; `createSchemaForSecurityScheme` returns `unsupported security scheme` for oauth2/openIdConnect/apiKey-in-query/cookie. **[V]** ArgoCD's own session call read from `community-plugins/.../argocd-node/src/ArgoCDService.ts:131` |
| **3** | **Path-item-level `parameters` are invisible to generation.** The `PathItem` interface has no `GetParameters()`; only operation-level params are read. A required `{realm}`-style path param declared once per path never becomes a CRD spec field, so the kind cannot bind its own URL. | **Keycloak** (273 of 274 paths affected), PagerDuty (7 paths), and any vendor spec that factors params out | **[V]** three independent ways: (a) `interfaces.go` — `PathItem` exposes only `GetOperations()`; `libopenapi_adapter.go:136` reads `a.op.Parameters`; (b) ran a 3.1.0 fixture declaring `tenantId`(path) + `apiVersion`(query) at path-item level and `dryRun` at operation level → `GetParameters()` returned **only `dryRun`**; (c) `krateo-blueprints/krateo-keycloak-operator-kog/hack/gen_oas_assets.py:270` says in a code comment *"oasgen-provider only reads path parameters declared at the OPERATION level (verified on a live cluster: path-item-level `parameters` are ignored, so the param never lands in the generated CRD spec)"* |
| **4** | **No pre-call dependent read.** A value that must be fetched from the API before the main request cannot be produced. | ArgoCD sync (needs app revision), Jenkins build (needs crumb), GitHub `GitRef`, any "look up the id of the thing I named" | **[V]** already filed as `oasgen-provider#120` (open) and `rest-dynamic-controller#30` (open); `FieldResolver.Type` is `+kubebuilder:validation:Enum=secretRef` (`types.go:~262`) |
| **5** | **No fire-and-forget action verb.** `VerbsDescription.Action` is `Enum=create;update;get;delete;findby` (`types.go:358`). A non-CRUD action that returns 204 with no handle has no expressible shape. | **ArgoCD sync, Jenkins build trigger, GitHub `workflow_dispatch`, Argo Workflows resubmit, Octopus deploy, StackStorm execute** — i.e. the write button on half the CI/CD plugins | **[V]** enum read from source. **[V]** the current workaround, and its ceiling, is written down in `github-provider-kog/chart/templates/rd-workflow.yaml`: *"enable/disable are not an update — they are the create and the delete"*. That trick works because `enable` is idempotent and observable; a dispatch is neither |

A sixth that is nearly as expensive in practice: **no schema allowlist**. `excludedSpecFields` is a blocklist, so a 152-field vendor schema can only be trimmed by naming ~140 exclusions. The Keycloak KOG's response was to abandon the vendor document and hand-write a curated OAS per kind (`gen_oas_assets.py`, 328 lines) — *"the full RealmRepresentation has 152 fields and would blow up the generated CRD"* **[V]**.

### Top portal gaps

Krateo ships **44 widget CRDs / 50 widget modules** **[V]**. Five widget types that Backstage parity demands do not exist in any form: **Iframe/Embed** (Grafana's whole UI is an embedded dashboard), **LogViewer** (GitHub Actions, Jenkins, Kubernetes), **RadarChart** (Tech Radar's entire reason to exist), **Timeline** (ArgoCD deployment lifecycle, PagerDuty change events), **Heatmap** (code-coverage, xcmetrics). A sixth is a capability gap rather than a missing kind: `Table.widgetData` has `rowNavigateTo` but no per-row action, so "re-run this build" cannot be put on the row.

---

## 2. Plugin inventory

Legend — **Ctrl?** = needs an oasgen-generated controller. **Widgets?** = needs portal widgets. `n/a` = the Krateo counterpart is a different mechanism, not a port.

### 2a. Core / built-in (`backstage/backstage/plugins/`) **[V for existence, P for the parity verdict]**

| Plugin | What it does | Backend it talks to | OpenAPI spec | Ctrl? | Widgets? |
|---|---|---|---|---|---|
| `catalog` (+`-react`,`-node`,`-graph`,`-import`) | software catalog: entities, relations, ownership, entity pages | its own Postgres via catalog-backend | **YES** — `plugins/catalog-backend/src/schema/openapi.yaml` **[V]** | **n/a** — Krateo's catalog *is* the apiserver (CRs + CompositionDefinitions). A controller over Backstage's catalog would only matter for importing *from* Backstage | YES |
| `scaffolder` (+`-backend-module-*` ×14) | templated software creation, multi-step wizard, dry-run | its own DB + git providers | **YES** — `plugins/scaffolder-backend/src/schema/openapi.yaml` **[V]** | **n/a** — Krateo's counterpart is Builders + `Form` widget + git-provider/cdc | YES |
| `techdocs` | docs-like-code: build, store and render MkDocs sites per entity | techdocs-backend + object storage | not found in the `openapi.yaml` search **[V]** | n/a | YES (and needs a new widget — see §5) |
| `search` (+5 collator modules) | federated search over catalog/techdocs/stackoverflow | search-backend (pg / elasticsearch) | **YES** — `plugins/search-backend/src/schema/openapi.yaml` **[V]** | n/a | YES |
| `kubernetes` (+`-cluster`,`-react`,`-node`) | per-entity workload view, pod status, logs, events | real clusters via kubeconfig | k8s `/openapi/v3` | **n/a — Krateo IS this.** A RESTAction against the apiserver endpoint covers it | YES |
| `permission` (+`-backend-module-policy-allow-all`) | authorization decisions for plugin actions | permission-backend | not found **[V]** | n/a — k8s RBAC + snowplow `userAccessFilter` | partial |
| `notifications` (+ email/slack modules) | in-app notification inbox | notifications-backend | **YES** — `plugins/notifications-backend/src/schema/openapi.yaml` **[V]** | n/a — sse-proxy `/events`+`/notifications` + the portal bell | YES |
| `signals` | server→client websocket push | signals-backend | — | n/a — sse-proxy | n/a |
| `events` (+9 ingress modules: github, gitlab, bitbucket, kafka, aws-sqs, google-pubsub, azure…) | **inbound webhook/event bus** | varies | **YES** — `plugins/events-backend/src/schema/openapi.yaml` **[V]** | **see §4 item 8** — Krateo has no inbound-webhook ingress at all | n/a |
| `auth` (+20 provider modules) | OIDC/OAuth sign-in | IdPs | — | n/a — Krateo `authn` | n/a |
| `api-docs` | render OpenAPI/AsyncAPI/GraphQL specs for an entity | catalog entity annotations | — | no | YES |
| `home`, `org`, `user-settings`, `devtools`, `app-visualizer`, `proxy`, `gateway`, `mcp-actions` | shell / meta | — | — | no | partial |

### 2b. Widely-used integrations (`community-plugins` + external)

| Plugin | Backend | OpenAPI spec URL | Spec ver | Ctrl? | Widgets? |
|---|---|---|---|---|---|
| **argocd** | Argo CD API server | `https://raw.githubusercontent.com/argoproj/argo-cd/master/assets/swagger.json` **[V] HTTP 200, 369 KB** | **swagger 2.0 → REJECTED** **[V]** | YES (blocked: gaps 1,2,5) | YES |
| **github-actions** (workspace `github`) | GitHub REST | `https://raw.githubusercontent.com/github/rest-api-description/main/descriptions/api.github.com/api.github.com.json` **[V] 200, 13 MB** | openapi 3.0.3 | **partly exists** — `krateo-blueprints/github-provider-kog` already ships a `Workflow` kind **[V]**; runs/dispatch/logs missing | YES |
| **github-issues / -pull-requests-board / -deployments / -discussions** | GitHub REST (+ GraphQL for discussions) | as above; GraphQL has **no** OpenAPI | 3.0.3 / n/a | partly exists (`Issue`, `PullRequest`, `Deployment` kinds ship today **[V]**) | YES |
| **jenkins** | Jenkins Remote Access API | **NONE official.** Jenkins has no OpenAPI; a GSoC-2025 project goal exists to create one; the only artefact is community `swaggy-jenkins` **[V]** (`jenkins.io/projects/gsoc/2025/project-ideas/swagger-openapi-for-jenkins-rest-api`) | — | YES, but **the spec must be authored by us** | YES |
| **grafana** | Grafana HTTP API | `https://raw.githubusercontent.com/grafana/grafana/main/public/api-merged.json` **[V] 200, 695 KB** | **swagger 2.0 → REJECTED** **[V]** | partly (alerts/dashboards/folders) | YES — and the plugin's `DashboardViewer` is an **iframe**, so an Iframe widget is mandatory |
| **sonarqube** | SonarQube web API | v2 only: `https://next.sonarqube.com/sonarqube/api/v2/api-docs` **[V] 200, 476 KB, openapi 3.1.0**. The `api/measures`,`api/qualitygates` endpoints the plugin actually reads are **v1 `web_api`, which has no OpenAPI** **[U] — I did not diff the plugin's calls against the v2 spec** | 3.1.0 (partial coverage) | read-only → **no** | YES |
| **pagerduty** (PagerDuty's own org) | PagerDuty REST | `https://raw.githubusercontent.com/PagerDuty/api-schema/main/reference/REST/openapiv3.json` **[V] 200, 2.8 MB, openapi 3.0.2, `api_key` apiKey-in-header** | 3.0.2 | **YES** — the plugin ships `TriggerButton`/`TriggerDialog` **[V]**, i.e. it writes | YES |
| **jira** (RoadieHQ) | Jira Cloud platform REST v3 | `https://developer.atlassian.com/cloud/jira/platform/swagger-v3.v3.json` **[V] 200, 2.5 MB, openapi 3.0.1, securitySchemes = OAuth2 + basicAuth** | 3.0.1 | YES (create/transition issue) | YES |
| **tech-radar** | **none** — a static JSON/TS `TechRadarLoaderResponse` | n/a | n/a | **NO** | YES — needs a RadarChart widget that does not exist |
| **cost-insights** | **none standard** — you implement a `CostInsightsApi` against your own cloud-billing source | n/a | n/a | **NO** | YES |
| **lighthouse** | `spotify/lighthouse-audit-service` | **[U] — I did not locate an OpenAPI spec for lighthouse-audit-service** | — | YES for "run an audit" (`CreateAudit` **[V]**) | YES |
| **keycloak** | Keycloak Admin REST | `https://www.keycloak.org/docs-api/latest/rest-api/openapi.json` **[V] 200, 567 KB, openapi 3.0.3, securitySchemes = {} (none declared)** | 3.0.3 | **exists** — `krateo-blueprints/krateo-keycloak-operator-kog` **[V]** | YES |
| **sentry** | Sentry API | `https://raw.githubusercontent.com/getsentry/sentry-api-schema/main/openapi-derefed.json` **[V] 200, 4.3 MB, openapi 3.0.3** | 3.0.3 | YES (resolve/ignore issue) | YES |
| **opencost** | OpenCost | `https://raw.githubusercontent.com/opencost/opencost/develop/docs/swagger.json` **[V] 200, openapi 3.0.1** | 3.0.1 | NO (read-only) | YES |
| **quay / nexus-repository-manager / jfrog-artifactory** | registry APIs | Quay: `/api/v1/discovery` per-instance. Nexus: `/service/rest/swagger.json` per-instance. **[U] — I did not fetch either; both are instance-hosted, not published at a stable public URL** | **[U]** | YES (delete tag / promote) | YES |
| **harbor** (not a Backstage community plugin; listed because Krateo has a local clone) | Harbor | `https://raw.githubusercontent.com/goharbor/harbor/main/api/v2.0/swagger.yaml` **[V] 200, 327 KB, `swagger: '2.0'`** | **2.0 → REJECTED** **[V]** | YES | YES |
| **argo-workflows** | Argo Workflows server **or** the CRDs | `https://raw.githubusercontent.com/argoproj/argo-workflows/main/api/openapi-spec/swagger.json` **[V] 200, 823 KB, swagger 2.0** | 2.0 → rejected | **NO — read the CRDs** via a RESTAction on the apiserver | YES |
| **tekton** | Tekton CRDs (+ optional Results API) | Results: `https://raw.githubusercontent.com/tektoncd/results/main/docs/api/openapi.yaml` **[V] 200, openapi 3.0.0** | 3.0.0 | NO for pipelines (CRDs); optional for Results | YES |
| **flux / kiali / linkerd / topology / ocm / fairwinds-insights** | all CRD-backed or in-cluster | k8s `/openapi/v3` | native | **NO** | YES |
| **azure-devops** | Azure DevOps REST | **[U] — Microsoft publishes reference docs, not a single consolidated OpenAPI document I could fetch.** Krateo nonetheless ships `azuredevops-provider-kog` **[V]** (there is a `testdata/azuredevops-provider-kog/` fixture in oasgen) | **[U]** | exists | YES |
| **servicenow** | ServiceNow Table API | per-instance OpenAPI **[U] — not fetched** | **[U]** | YES | YES |
| **newrelic** | NerdGraph | **GraphQL — no OpenAPI, and oasgen cannot express GraphQL at all** **[P]** (oasgen's only input is an OAS document) | n/a | **NO (hard blocker)** | YES |
| **dynatrace** | Dynatrace API v2 | **[U] — the URL I tried 404'd; Dynatrace does publish OpenAPI per-tenant** | **[U]** | YES | YES |
| **vault** | HashiCorp Vault | per-instance `/v1/sys/internal/specs/openapi` **[U] — the repo-root URL I tried returned 404** | **[U]** | YES | YES |
| **snyk / blackduck / checkmarx / mend / fossa / codescene / code-climate** | SaaS scanners | Snyk: date-versioned OpenAPI, `https://api.snyk.io/rest/openapi` returns the **version list**, not a spec **[V]** — you must then fetch a dated version | varies | mostly read-only → NO | YES |
| **gocd / bitrise / octopus-deploy / cloudbuild / apache-airflow / n8n / stackstorm / tekton / cicd-statistics** | CI/CD APIs | Octopus: the `OctopusDeploy/OpenAPI` path I tried **404'd** **[V]**; others **[U]** | varies | YES (trigger) | YES |
| **adr / announcements / bazaar / badges / bookmarks / entity-feedback / explore / linguist / playlist / shortcuts / todo / wheel-of-names / feedback / manage** | mostly Backstage's own DB, or the catalog | their own backends | n/a | n/a | YES |
| **tech-insights** | tech-insights-backend (scorecards/checks) | its own DB | **[U]** | n/a — closest Krateo analogue is `compliance` / `access-review` | YES |
| **rbac** | Backstage permission backend | — | — | n/a — k8s RBAC | YES |
| **copilot / analytics / graphiql / graphql-voyager / apollo-explorer / mcp-chat / agent-forge** | assorted | varies | varies | NO | YES |

### What Krateo already has, so parity is cheaper than it looks **[V]**

`krateo-blueprints` already ships KOGs for: `github-provider-kog`, `krateo-keycloak-operator-kog`, `krateo-zendesk-kog`, `krateo-vmware-kog`, `krateo-openshift-kog`, `krateo-nutanix-operator-kog`, `krateo-arubacloud-provider-kog`, `krateo-oxide-operator-kog`, `hetzner-object-storage-kog`, `openstack-{nova,keystone,glance,neutron,ironic}-operator-kog`, `krateo-{5g-apn,mpls}-kog`, `krateo-hyperdx-provider`. **None of the top CI/CD-or-observability Backstage backends (ArgoCD, Jenkins, Grafana, SonarQube, Jira, PagerDuty) has a KOG today.**

---

## 3. Per-plugin detail (top 12)

### 3.1 ArgoCD — *the single hardest and highest-value target*

- **UI surface [V]** (`community-plugins/workspaces/argocd/plugins/argocd/src/components/`): `AppStatus`, `DeploymentLifeCycle`, `DeploymentSummary`, `Common`.
- **Endpoints the plugin calls [V]** (`argocd-node/src/ArgoCDService.ts`, `serviceUtils.ts`):
  - `POST {base}/api/v1/session` with `{username,password}` → `{token}` — **a login exchange**
  - `GET {base}/api/v1/applications` (+ `selector` query)
  - `GET {base}/api/v1/applications/{appName}`
  - `GET {base}/api/v1/applications/{appName}/revisions/{revisionID}/metadata`
  - instances are multi-tenant: `matchedInstance.token ?? await this.getToken(matchedInstance)` — a per-instance static token **or** a minted one.
- **Auth model**: bearer token, either supplied statically or minted via `/api/v1/session`.
- **Controller**: wanted for `Application`, `AppProject`, and the **sync** action. **Blocked by three gaps at once**: spec is swagger 2.0 (gap 1), the session exchange (gap 2), and `sync` being a non-CRUD action (gap 5).
- **Widgets**: Table (app list, health/sync columns) + Badge/Tag (Healthy/Degraded, Synced/OutOfSync) + Descriptions (app detail) + **Timeline (missing)** for `DeploymentLifeCycle` + FlowChart (resource tree) + Button (sync) + **LogViewer (missing)** if you want sync logs.

### 3.2 GitHub Actions

- **UI surface [V]**: `WorkflowRunsTable`, `WorkflowRunsCard`, `WorkflowRunDetails`, `WorkflowRunLogs`, `WorkflowRunStatus`, `Cards`.
- **Backend**: GitHub REST, **openapi 3.0.3, 816 paths / 1232 operations, 0 path-item-level parameters [V]** — i.e. GitHub's document is maximally oasgen-friendly.
- **Already native**: `github-provider-kog` ships 31 kinds including `Workflow` (create=PUT `.../enable`, delete=PUT `.../disable`, get with a `notFoundBody` jq predicate on `.state`) **[V]**.
- **Still missing for parity**: workflow **runs** (`GET /repos/{o}/{r}/actions/runs` — read-only, so a RESTAction suffices, no controller needed), run **logs** (`GET .../runs/{id}/logs` returns a **302 to a zip** — **[U]** I did not verify how RDC or snowplow handle a redirect-to-binary), and **`workflow_dispatch`** (`POST .../workflows/{id}/dispatches`, **204 with an empty body** → gap 5: fire-and-forget with no handle).
- **Widgets**: Table + Badge + **LogViewer (missing)** + Button (re-run) + **per-row action on Table (missing)**.

### 3.3 Jenkins

- **UI surface [V]**: `BuildsPage`, `BuildWithStepsPage`, `JobRunsTable`, `Cards`, plus a polling hook `useAsyncPolling.ts`.
- **Backend**: `/api/json` with `?tree=` projections. **No official OpenAPI exists** **[V]** — Jenkins has "no automated REST API documentation at all"; standardising it is a *GSoC 2025 project idea*, and the only artefact is the community `swaggy-jenkins` generator.
- **Consequence**: a Jenkins KOG requires Krateo to **author and maintain the OAS document itself**. That is already the established Krateo idiom — the Keycloak KOG does exactly this (`gen_oas_assets.py`) — but it should be a deliberate decision, not a surprise.
- **Auth**: HTTP basic with user + API token → **supported today**. But a POST also needs a **CSRF crumb** fetched from `/crumbIssuer/api/json` first → gap 4 (pre-call dependent read).
- **Widgets**: Table (`JobRunsTable`) + Badge + Steps (build stages) + **LogViewer (missing)** for console output.

### 3.4 Grafana

- **UI surface [V]**: `AlertsCard`, `DashboardsCard`, `DashboardViewer`.
- **Backend spec**: `public/api-merged.json`, **swagger 2.0 [V]** → rejected (gap 1).
- **The decisive frontend point**: `DashboardViewer` **embeds Grafana's own rendered dashboard**. No amount of chart widgets reproduces it. Parity requires an **Iframe/Embed widget, which Krateo does not have** **[V]**. Roadie ships a general-purpose `backstage-plugin-iframe` **[V]** for exactly this reason.
- **Controller**: only worth it for provisioning dashboards/folders/alert rules as CRs. Read panels need only RESTActions.
- **Widgets**: **Iframe (missing)** + Table (alerts) + Listy (dashboards) + Badge (alert state).

### 3.5 SonarQube

- **UI surface [V]**: `SonarQubeCard`, `SonarQubeTable`, `SonarQubeContentPage`, `SonarQubeRelatedEntitiesOverview`.
- **Spec**: v2 is **openapi 3.1.0** and oasgen **parses 3.1.0 fine** **[V]** (probed). But the measures/quality-gate endpoints the plugin reads are v1 `web_api`, outside the v2 document **[U] — not diffed**.
- **Verdict: widgets only.** The plugin never writes.
- **Widgets**: Card + Statistic (bugs / vulnerabilities / code smells / duplication) + **Progress with `type: circle`** for coverage — `Progress.widgetData` has `type`, `percent`, `steps`, `strokeColor`, `status` **[V]**, so this one is covered without new work + Badge (quality gate pass/fail) + Table.

### 3.6 PagerDuty

- **UI surface [V]**: `EntityPagerDutyCard`, `EntityPagerDutySmallCard`, `HomePagePagerDutyCard`, `Incident`, `Escalation`, `ChangeEvents`, **`TriggerButton` + `TriggerDialog`**, `Errors`.
- **Spec [V]**: openapi 3.0.2, 274 paths / 466 operations, `securitySchemes = {api_key: apiKey in header}` → **oasgen's `apiKey`-in-header support covers it exactly**, and the header default is auto-derived when a document declares a single apiKey scheme (`applyAPIKeyHeaderDefault`) **[V]**. Only **7 paths / 10 params** sit at path-item level, so gap 3 is a minor annoyance here, not a blocker.
- **Controller**: `Incident` (POST /incidents — returns the created incident, so this one is a *proper* create, not a fire-and-forget), `Service`, `EscalationPolicy`, `Schedule`. Acknowledge/resolve are `PUT /incidents/{id}` → expressible as `update`.
- **Verdict: this is the cheapest full-parity (controller + widgets) win on the list.**
- **Widgets**: Card + Listy (incidents, with `sseEndpoint` for live updates **[V]**) + Table (escalation policy / on-call) + Button + **Timeline (missing)** for `ChangeEvents` + Alert.

### 3.7 Jira (RoadieHQ)

- **Spec [V]**: openapi 3.0.1, 423 paths / 620 operations, 0 path-item params, `securitySchemes = {OAuth2: oauth2, basicAuth: http/basic}`.
- **Auth**: Jira Cloud's practical auth is **basic with email + API token**, which oasgen supports **[V]**. The declared `OAuth2` scheme is **skipped with a warning** (gap 2) — tolerable *only because* `basicAuth` is also declared. Note the subtle trap, which the source is explicit about: a document whose **only** scheme is oauth2 still yields a `<Kind>Configuration` CRD with **no `authentication` field at all**, and the user "finds out via 401s" — the generator's own comment calls it *"A CRD that lies is worse than no CRD"* **[V]** (`configuration_builder.go:112-125`).
- **Controller**: `Issue` (create/update/transition), `Project`, `Component`, `Version`.
- **Widgets**: Card (issue counts by type/status) + Table (issue list) + PieChart/BarChart (breakdown) + Form (create issue) + Badge.

### 3.8 Kubernetes (core)

- **Verdict: no controller, ever.** Krateo *is* the apiserver. A RESTAction with `endpointRef` → the `krateo-kube` Endpoint already reads cluster objects, and `userAccessFilter` gives per-user RBAC refiltering **[V]** (`snowplow/apis/templates/v1/core.go`).
- **Widgets**: Table + Descriptions + FlowChart (owner graph) + Badge + **LogViewer (missing)** for pod logs + Listy with `sseEndpoint` for events.
- **This is the one plugin where Krateo is structurally ahead of Backstage**, and it should be said that way in any parity pitch.

### 3.9 Tech Radar

- **Backend: none** **[V]** — the data is a static `TechRadarLoaderResponse` (JSON/TS).
- **UI surface [V]**: `Radar`, `RadarPlot`, `RadarGrid`, `RadarLegend`, `RadarBubble`, `RadarEntry`, `RadarDescription`, `RadarTimeline`, `RadarFooter`.
- **Verdict: widgets only, no controller.** But the one widget it needs — a **quadrant/ring radar plot** — does not exist and is not approximable by Bar/Line/Pie. This is the clearest "build one new widget, get a whole plugin" item on the list.
- Data source in Krateo: a ConfigMap read through a RESTAction, or a `TechRadar` CR.

### 3.10 Cost Insights

- **Backend: none standard** **[V]** — Backstage ships only an API *interface*; every adopter writes their own.
- **UI surface [V]**, and it is big: `CostOverviewCard`, `ProductInsightsCard`, `EntityCosts`, `CostGrowth`, `ProjectGrowthAlertCard`, `UnlabeledDataflowAlertCard`, `ActionItems`, `AlertInsights`, `BarChart`, `MetricSelect`/`PeriodSelect`/`ProjectSelect`/`CurrencySelect`, `CostInsightsTabs`.
- **Krateo already has the backend**: `finops-*` (operator-focus, operator-scraper, database-handler, prometheus-exporter) and `krateo-platformops/showback` **[V existence from the local clone list]**. So this is a **widgets-only** job over an existing Krateo backend, not an integration.
- **Widgets**: BarChart (**`stack` and `group` already exist [V]** → stacked cost-by-product works today) + LineChart + Statistic + Card + Filters (the four selects) + Tabs + Alert.

### 3.11 Lighthouse

- **UI surface [V]**: `AuditList`, `AuditView`, `CreateAudit`, `AuditStatusIcon`, `Cards`, `Intro`.
- **Backend**: `spotify/lighthouse-audit-service`. **[U] — no OpenAPI spec located.**
- **Controller**: `Audit` as a create-only resource — a genuinely good fit for the existing idiom (POST creates an audit, GET polls it, and `async` in `VerbsDescription` already handles "trigger then poll to terminal status" **[V]**: `AsyncConfig` with `operationRef` + `poll.statusPath` + `successValues`, including a non-blocking `mode: requeue`).
- **Widgets**: Table (audit list) + **Progress with `type: circle`** for the five 0-100 scores (covered) + Form (create audit) + Badge.

### 3.12 Catalog + Scaffolder (core) — the structural pair

Not a port. The honest framing for a parity document:

| Backstage | Krateo counterpart | Status |
|---|---|---|
| `catalog-info.yaml` + entity kinds | CRs + `CompositionDefinition` | exists, stronger (typed, validated, reconciled) |
| catalog relations / ownership graph | owner refs + `FlowChart` widget | exists |
| catalog processors/providers (ingest from GitHub/LDAP/AWS…) | **nothing equivalent** — Krateo has no "discover entities from an external source and keep them in sync" loop. The closest is a KOG `findby` over one resource kind | **gap, and it is a product gap, not an oasgen gap** |
| scaffolder templates + actions | Builders + `Form` widget + git-provider + cdc | exists |
| scaffolder dry-run / preview | `oasgen-render` service (closed issue #157 **[V]**) + `helm-render-service` | partially exists |
| techdocs | — | **gap** (needs a doc-site widget) |

---

## 4. oasgen-provider implementation plan

Ordered. Each item names the plugin that needs it and the evidence.

---

**1. Accept Swagger / OpenAPI 2.0 documents.**

- *Needed by*: **ArgoCD**, **Grafana**, **Argo Workflows**, **Harbor** — and, going by how common 2.0 still is in vendor specs, a long tail beyond them.
- *Why it cannot be expressed today*: `libOASParser.Parse` calls `d.BuildV3Model()` and returns `CodeModelBuildError` on anything else (`libopenapi_adapter.go:41`). **[V]** Running the real ArgoCD document through the real parser produces: `failed to build V3 model: unable to build openapi document, supplied spec is a different version (oas2). Try 'BuildV2Model()'`.
- *Shape*: libopenapi's own error names the remedy. The pinned fork is `krateoplatformops/libopenapi v0.21.8` (`go.mod:133`) **[V]**, which exposes `BuildV2Model()`. Two candidate designs, and the choice matters:
  - **(a) a second adapter** (`libOASv2DocumentAdapter`) behind the existing `OASDocument` interface. Keeps the "a generated CRD traces back to one vendor document, checksum-enforced" guarantee that `OASPath`'s doc comment is explicit about **[V]**. Costs a parallel adapter.
  - **(b) convert 2.0→3.0 on ingest.** Cheaper to write, but it **breaks the checksum guarantee** — `status.oasHash` would hash a document the vendor never published, and the "two documents that disagree are REFUSED at generation" logic (#108) would be reasoning about converted artefacts.
  - *Recommendation*: **(a)**. The repo has gone out of its way to keep vendor documents unmerged and unmodified; a converter contradicts that on purpose rather than by accident.
- *Watch out for*: 2.0 has no `components.securitySchemes` (it has `securityDefinitions`), no `requestBody` (it has `in: body` parameters), and `produces`/`consumes` at document level. The `GetRequestBody()` and `SecuritySchemes()` adapter methods both need a 2.0 story, and `AcceptedMIMETypes` (`DefaultGeneratorConfig`) interacts with document-level `produces`.

**2. Credential exchange: a `tokenExchange` auth method.**

- *Needed by*: **ArgoCD** (`POST /api/v1/session` → `{token}`), **Jenkins** (crumb), **Jira/Confluence/ServiceNow OAuth2**, **Vault login**, any OAuth2 client-credentials API.
- *Why it cannot be expressed today*: `authMethodKey` is a closed three-case switch — `http/basic`, `http/bearer`, `apiKey in header` — and everything else is returned as a *skipped* scheme (`configuration_builder.go:183-230`) **[V]**. The credential in a `<Kind>Configuration` is a **static** Secret value; there is no notion of a credential with a lifetime.
- *Shape*: a fourth `authentication.*` key whose config declares the exchange — endpoint, method, request body template (from the Secret's fields), a JSONPath to the token in the response, an optional TTL path, and the header to put it in. Reuse `JQProgram` for the body template so there is no new templating language. Cache the minted token in RDC keyed by `(Configuration UID, Secret resourceVersion)` and re-mint on 401.
- *Do this before item 1* if you only do one, because ArgoCD needs **both** and Jira/ServiceNow need only this.
- *Explicitly also fix the silent case*: a document whose only declared scheme is unsupported still generates a `<Kind>Configuration` with **no `authentication` property**. The source already calls this out as the bug it is **[V]**; closed issue **#49** ("Support `apiKey`-in-header security schemes **and never skip an auth scheme silently**") says schemes are now reported, but `skippedSecuritySchemes` is surfaced as a **warning**, and closed issue **#151** records that *"Schema-generation warnings are unreachable: DEBUG+3 and absent from RestDefinition status"* **[V]**. **[U] — I did not verify whether #151's fix made skipped schemes visible in `RestDefinition.status`.** Check that before assuming a user would ever see the warning.

**3. Read path-item-level `parameters`.**

- *Needed by*: **Keycloak** (**273 of 274 paths, 555 parameters** **[V]** measured on the official document), **PagerDuty** (7 paths / 10 params **[V]**), and, going by how often vendors factor a `{tenant}`/`{realm}`/`{subscription}` param out, most enterprise specs.
- *Why it cannot be expressed today*: `PathItem` (the interface) declares only `GetOperations()`; `libOASPathItemAdapter` implements only that; `libOASOperationAdapter.GetParameters()` reads `a.op.Parameters` **[V]**. A path-item parameter therefore never reaches `addParametersToSpec`, so a **required** path param never becomes a CRD spec field and the kind cannot bind its own URL.
- *Evidence, three ways*: source read; a live probe against a 3.1.0 fixture returning only the operation-level param; and the Keycloak KOG's code comment, which says it was *"verified on a live cluster"* **[V]**.
- *Shape*: add `GetParameters()` to the `PathItem` interface and merge path-item params **under** operation params per the OAS spec's own override rule (an operation-level parameter with the same `name`+`in` replaces the path-item one). Feed the merged set to `addParametersToSpec`.
- *Blast radius to think about*: `addParametersToSpec` already treats a cross-document same-name disagreement as **fatal** (#108). Newly-visible path-item params will add names to that merge, so a RestDefinition that generates cleanly today could start failing. **[P]** That is correct behaviour, but it is a behaviour change on upgrade and belongs in the release notes.
- *Same bug, second site*: `ConfigurationField.FromOpenAPI{Name, In}` resolves through the same adapter **[P]** — so a path-item param cannot be promoted into the Configuration CR either. Fix both or the fix is half a fix.

**4. A schema allowlist (`includedSpecFields`), as the counterpart to `excludedSpecFields`.**

- *Needed by*: **Keycloak** today; **Jira**, **Sentry**, **GitHub** (13 MB document) next.
- *Why it cannot be expressed today*: `excludedSpecFields` is a blocklist **[V]** (`types.go`). Keycloak's `RealmRepresentation` has **152 fields**, so expressing "I want 30 of them" means naming ~122 exclusions — and the field is **immutable** (`self == oldSelf`), so getting the list wrong once is unrecoverable without deleting the RestDefinition and its CRs.
- *Evidence*: the Keycloak KOG abandoned the vendor document entirely and hand-writes a curated OAS per kind, stating the reason in the module docstring: *"keeps a curated, SSO-relevant subset of the upstream *Representation schemas (the full RealmRepresentation has 152 fields and would blow up the generated CRD)"* **[V]**.
- *Shape*: `includedSpecFields []string`, mutually exclusive with `excludedSpecFields`, append-only rather than immutable (same reasoning as `additionalStatusFields`, whose immutability was already relaxed for exactly this class of upgrade pain **[V]**).
- *Why this ranks above the action verb*: it is the difference between "a KOG is a 30-line RestDefinition" and "a KOG is a 328-line Python generator plus a vendored OAS subset per kind". It changes the **cost per integration**, which is what a 130-plugin parity programme is bounded by.

**5. A non-CRUD `action` verb.**

- *Needed by*: **ArgoCD** `POST /api/v1/applications/{name}/sync`, **GitHub** `POST .../workflows/{id}/dispatches` (**204, empty body**), **Jenkins** `POST /job/{name}/build`, **Argo Workflows** resubmit/retry, **Octopus** deploy, **StackStorm** execute, **Lighthouse** run-audit.
- *Why it cannot be expressed today*: `Action` is `Enum=create;update;get;delete;findby` **[V]**. The established workaround is to map an action onto create/delete, and the GitHub KOG documents both the trick and its precondition: *"enable/disable are not an update — they are the create and the delete"* **[V]** (`rd-workflow.yaml`). That works only because `enable` is **idempotent and observable** — the kind's whole loop is a `notFoundBody` jq predicate on `.state`.
- *A dispatch is neither.* It is not idempotent (two POSTs = two runs) and the response carries no handle, so there is nothing to observe and `findby` has nothing to key on.
- *Shape — and this is the part worth designing rather than bolting on*: the reconciler is level-based by construction, so a truly fire-and-forget action has **no safe home inside a reconcile loop**. Two honest options:
  - **(a) An `action` verb with a mandatory correlation rule**: the author declares what, in some observable collection, identifies *this* invocation (e.g. for `workflow_dispatch`: a `findby` over `/actions/runs` keyed on a client-supplied input echoed in `.head_sha`/`inputs`). No correlation ⇒ refuse the RestDefinition. This keeps the level-based guarantee.
  - **(b) Declare it out of scope and route actions through `createApiRef`** (a snowplow RESTAction), accepting that the credential moves to a platform-level Endpoint — which `#120` already measured as *"a real change in the security model, not a detail"* **[V]**.
  - *Recommendation*: **(a)**, and refuse rather than guess. The repo's own history is a sequence of "silently guessed wrong" bugs (`#119` pagination creating duplicates, `apiLookup`'s `items[0]`, `#110` envelope inference) and the existing types all say some version of *"guessing here is not a smaller version of being right"* **[V]**.

**6. Pre-call dependent read (`#120`, open).**

- *Needed by*: GitHub `GitRef` (today's single consumer), **Jenkins** crumb, **ArgoCD** app-revision lookup, `rest-dynamic-controller#30`'s Keycloak alias→id.
- Backstage parity is the **second, third and fourth consumers** `#120` says it does not have. That is the one thing this research changes about that issue, and it is worth saying in the issue itself: the bar it declines to claim is now met.
- Shape is already designed in `#120` (sibling of `verbsDescription`, referenced from a `FieldMappingItem` resolver, **client-side matching with ambiguity a hard error**, a `when` guard, order-only chaining). Implement that, not a new design.

**7. Multi-segment path parameters (`#114`, closed NOT_PLANNED).**

- *Needed by*: GitHub `GET /repos/{o}/{r}/git/ref/{ref}` where `ref` = `heads/main`; **[U]** I did not survey how many other candidate APIs need this.
- `#114` was closed as not-planned, and `#117` (the re-filed version, **COMPLETED 2026-10-01** **[V]**) delivered request-direction jq on `inPath`/`inQuery` instead, which covers *transforming* a path param but not *un-escaping* it. The `url.PathEscape` guard is deliberate and load-bearing (a CR field would otherwise become a path-traversal primitive).
- Keep this low. An opt-in per-parameter `allowSegments: true` is the shape if it ever becomes necessary; the security argument in `#114` is correct and should not be re-litigated.

**8. Inbound webhooks / event ingress — *note the gap, do not put it in oasgen*.**

- *Needed by*: everything whose freshness matters — ArgoCD app health, Jenkins build completion, GitHub Actions run status, PagerDuty incident triggered. Backstage has a whole `events` plugin family with **9 ingress modules** (github, gitlab, bitbucket-cloud, bitbucket-server, gerrit, kafka, aws-sqs, google-pubsub, azure) and a published OpenAPI **[V]**.
- Krateo has **no inbound webhook ingress** **[V]** — `grep -rin "webhook|callback"` across oasgen's Go source returns only comments about *not* having admission webhooks.
- *Why not in oasgen*: oasgen generates **controllers**, which are pull-based by design, and the whole `async`/`poll` machinery exists to make pull work. A webhook receiver is a different component (an ingress that converts an HTTP POST into either a CR annotation poke or an sse-proxy event).
- *Action*: file this against the platform, not oasgen. Nearest existing homes: `sse-proxy`, `krateo-platformops/alert-provider`, `incident-controller`.

**9. Re-test the "inline array-of-object items are dropped" claim.**

- The Keycloak KOG works around it: *"Uses a `$ref`: oasgen-provider drops inline array-of-object items"* **[V] that the comment exists**.
- But the current source **does** convert `Items` (`libopenapi_adapter.go:346-356`: `s.Items.IsA()` → `convertLibopenapiSchemaWithVisited`), with an explicit warning only for the 3.1 **tuple** form (`items` as an array of schemas) **[V]**.
- **[U] — I did not run a probe for inline array-of-object.** Either the KOG comment is stale (likely, given the source) or the drop happens later in `prepareSchemaForCRD`. Resolve it with a 10-line probe before anyone writes another `$ref` workaround; and note the 3.1 tuple case is a *real* confirmed gap, logged only to `log.Print` (see #151 on warning reachability).

**10. GraphQL is out of scope, and should be said so.**

- **New Relic** (NerdGraph), **GitHub Discussions**, parts of Shopify-style APIs are GraphQL. oasgen's only input is an OAS document **[P]**, so these are permanently widgets-only (a RESTAction can POST a GraphQL query; a generated controller cannot be built from a GraphQL schema). Put it in the docs as a stated non-goal rather than letting each KOG author rediscover it.

### Already present — do not re-plan these

Worth stating because a parity plan that proposes them would look uninformed. All **[V]** from `types.go`:

- **Pagination** for `findby` — `continuationToken` *and* `pageNumber` (with short-page / `Link`-header / total-count last-page detection, a required `maxPages`, and page-exhaustion reported as its own outcome rather than not-found).
- **Async long-running operations** — `AsyncConfig` with `operationRef` (body or header, + jq), `poll` (`statusPath`, `successValues`, `failureValues`, interval/attempts/timeout, a `handleParam` override), `postGet`, and a non-blocking `mode: requeue`.
- **Per-tenant credentials** — a `<Kind>Configuration` CR per instance via `configurationRef`, with per-namespace auth-Secret RBAC auto-managed (`AuthSecretDigest`, `AuthSecretRBACNamespaces`).
- **Value transforms** — `fieldMapping` with `alias` pairs and `jq` (inline or `configmap://`/`http://` module ref), in request **and** response direction, including on `inPath`/`inQuery` since #117.
- **Document transforms** — `requestTransform` and `responseTransform` (whole-body gojq).
- **Response→status mapping** — `identifiers` (immutable) + `additionalStatusFields` (append-only) + `inResponse` field mapping with `defaultIfAbsent`.
- **Drift scoping** — `compareScope: fullSpec | identifiersAndStatus | updatable`.
- **Odd HTTP semantics** — `successCodes`, `tolerateCodes`, `notFoundCodes`, `notFoundBody` (jq predicate), static per-verb `headers` and `queries`.
- **Envelope responses** — `itemsPath`, refusing ambiguity rather than guessing.
- **Multi-document resources** — per-verb `oasPath`, with cross-document parameter disagreement fatal.
- **RESTAction delegation** — `observeApiRef` / `createApiRef` / `updateApiRef` / `deleteApiRef` with `notFoundExpr` / `upToDateExpr`, which is the existing escape hatch for multi-call sequences.
- **Secret injection** — `FieldResolver` / `secretRef` with same-namespace-only enforcement at the type level.
- **Read-only resources** — a kind with no create verb gets its identifiers materialised into the spec as selectors (#75).

---

## 5. Portal widget parity plan

### What exists **[V]**

**44 widget CRDs** in `krateoplatformops/frontend/helm/frontend-crds/templates/` (group `widgets.templates.krateo.io`), **50 widget modules** in `ui/src/widgets/`:

`Alert · Badge · BarChart · Breadcrumb · Button · ButtonGroup · Card · Checkbox · Col · DatePicker · Descriptions · Divider · Drawer · Filters · Flex · FlowChart · Form · Image · Input · InputNumber · Layout · LineChart · Listy · Markdown · Menu · Modal · PageHeader · Paragraph · PieChart · Progress · QRCode · Radio · RangePicker · Result · Row · Select · Slider · Statistic · Steps · Switch · Table · Tabs · Tag · Theme · Upload · YamlViewer`

(`Listy` is `List` renamed — the CRD kind `List` is reserved and the apiserver 404s on CREATE.)

Every widget CRD shares one envelope — `apiRef`, `resourcesRefs`, `resourcesRefsTemplate`, `widgetData`, `widgetDataTemplate`, `freshness`, `identityContext`, `keyExtras` **[V]** — with the per-widget shape inside `widgetData`.

Capabilities that matter for parity, read from the CRDs **[V]**:

| Widget | `widgetData` keys | Parity relevance |
|---|---|---|
| `Table` | `columns, dataSource, pagination, rowNavigateTo, prefix, size, bordered, fitContent, watch, allowedResources` | **`pagination` and `watch` exist. No per-row action** — only `rowNavigateTo` |
| `BarChart` | `data, xField, yField, colorField, group, **stack**, legend, title, height, watch` | **stacked + grouped already work** → Cost Insights needs no new chart |
| `Progress` | `percent, type, steps, strokeColor, status, showInfo, size, label, description` | `type` covers `circle`/`dashboard` → **SonarQube coverage rings and Lighthouse 0-100 scores are covered today** |
| `Listy` | `dataSource, itemTemplate, grid, itemLayout, pagination, maxItems, actions, header, footer, **sseEndpoint, sseTopic**, watch, hideWhenEmpty` | `sseEndpoint`/`sseTopic` + `actions` → **a live activity feed is buildable today**; the closest thing to a Timeline |
| `Form` | `schema, stringSchema, items, initialValues, submitActionId, submitActionSelector, draftActionId, reviewBeforeSubmit, propertiesToHide, buttonConfig, layout` | scaffolder parity |
| `Button` | `actions, clickActionId, label, type, variant, icon, danger, disabled, block, shape, size` | the write path for sync/trigger/re-run |
| `Card` | `items, title, extra, cover, footer, tags, legend, **live**, **widgetActions**, clickActionId, tooltip, icon, anchorId` | entity cards |
| `FlowChart` | `data, variant` | resource trees / catalog graph |
| `Statistic` | `value, title, prefix, suffix, precision, valueStyle, watch` | KPI tiles |
| `Markdown` | `markdown, allowCopy, allowDownload, downloadFileExtension` | ADR, announcements, partial TechDocs |

### What is missing

| # | Widget | Needed by | Why nothing existing covers it |
|---|---|---|---|
| **1** | **`Iframe` / `Embed`** | **Grafana** (`DashboardViewer` is an iframe), TechDocs, api-docs, any "embed the vendor's own UI" plugin. Roadie ships a generic `backstage-plugin-iframe` precisely for this **[V]** | Nothing renders external HTML. There is no sandboxed-embed widget at any fidelity. **This is the single highest-leverage missing widget** — it buys Grafana parity outright and degrades a dozen other plugins from "impossible" to "embedded" |
| **2** | **`LogViewer`** | **GitHub Actions** (`WorkflowRunLogs`), **Jenkins** (`BuildWithStepsPage`), **Kubernetes** (pod logs), ArgoCD | `Markdown`/`YamlViewer` are static single-shot renderers: no tailing, no ANSI colour, no search, no virtualised scroll for a 50k-line log, no step-grouped folding. `Listy`'s `sseEndpoint` is the right *transport* and the wrong *renderer* |
| **3** | **`RadarChart`** | **Tech Radar** — and only Tech Radar | Quadrant/ring/bubble plot with per-entry placement and a movement timeline (`RadarPlot`, `RadarGrid`, `RadarBubble`, `RadarTimeline` **[V]**). Bar/Line/Pie cannot approximate it. Cheap win: one widget = one whole plugin, **no controller needed at all** |
| **4** | **`Timeline`** | **ArgoCD** (`DeploymentLifeCycle`), **PagerDuty** (`ChangeEvents`), Tech Radar (`RadarTimeline`), announcements, catalog entity history | `Steps` is a fixed linear progress indicator (`current`, `items`, `orientation`) **[V]** — not an unbounded, timestamped, heterogeneous event stream with per-entry detail |
| **5** | **`Heatmap`** | code-coverage, xcmetrics, cicd-statistics, contribution calendars | no equivalent |
| **6** | **Per-row actions on `Table`** | **GitHub Actions** (re-run a run), **Jenkins** (abort a build), ArgoCD (sync one app), PagerDuty (ack one incident) | `Table.widgetData` has `rowNavigateTo` and no action slot **[V]**. `Listy` has `actions`, so the pattern exists in the codebase — this is an extension, not a new widget. **[U] — I read the CRD, not the React `Table` component; confirm there is no undocumented action path before filing** |
| **7** | **`CodeViewer`** (syntax-highlighted, multi-language, optional diff) | api-docs, scaffolder preview, TechDocs, catalog-import | `YamlViewer` is YAML-only; `Markdown` fenced blocks are not a viewer |
| **8** | **`TechDocs` site renderer** | TechDocs | A navigable multi-page doc site with a sidebar, search and asset handling. Far bigger than a widget — **[U]** it may be better served by `Iframe` over an existing MkDocs build than by a native widget. Decide before scoping |
| **9** | **`Gauge`** (distinct from `Progress`) | **[U]** — I believe `Progress` with `type: circle`/`dashboard` covers SonarQube and Lighthouse, so I am **not** claiming this is needed. Verify against a real Lighthouse score card before adding it |

### Per-plugin widget shopping lists

| Plugin | Existing widgets | Missing |
|---|---|---|
| **ArgoCD** | Table, Badge/Tag, Descriptions, FlowChart, Button, Card | **Timeline** (deployment lifecycle), LogViewer (optional) |
| **GitHub Actions** | Table, Badge, Card, Button, Steps | **LogViewer**, **Table row actions** |
| **Jenkins** | Table, Badge, Steps, Card | **LogViewer**, **Table row actions** |
| **Grafana** | Table (alerts), Listy (dashboards), Badge | **Iframe** — blocking |
| **SonarQube** | Card, Statistic, **Progress(circle)**, Badge, Table | none |
| **PagerDuty** | Card, Listy(+sse), Table, Button, Alert | **Timeline** (change events) |
| **Jira** | Card, Table, PieChart, BarChart, Form, Badge | none |
| **Kubernetes** | Table, Descriptions, FlowChart, Badge, Listy(+sse) | **LogViewer** |
| **Tech Radar** | Card, Filters | **RadarChart** — blocking; **Timeline** for movement |
| **Cost Insights** | **BarChart(stack/group)**, LineChart, Statistic, Card, Filters, Tabs, Alert | none |
| **Lighthouse** | Table, **Progress(circle)**, Form, Badge | none |
| **Catalog / entity pages** | Card, Descriptions, Tabs, FlowChart, Table, Tag | **CodeViewer** (raw entity), TechDocs link-out |
| **Scaffolder / Builders** | Form, Steps, Button, Result, Markdown | **LogViewer** (task log), **CodeViewer** (dry-run diff) |
| **Search** | Filters, Input, Listy, Table | none |
| **TechDocs** | Markdown | **TechDocs renderer** or **Iframe** |

**Net: 5 genuinely new widget kinds (Iframe, LogViewer, RadarChart, Timeline, Heatmap) + 1 extension (Table row actions) + 1 undecided (TechDocs renderer vs Iframe) unlock the frontend half of every plugin on this list.**

### Two portal-layer gaps that are not widgets

1. **No pagination primitive in `RESTAction`** **[V]**. `API` carries `name, path, verb, headers, payload, endpointRef, dependsOn{name,iterator}, filter, continueOnError, errorKey, exportJwt, resolve, userAccessFilter` **[V]** (`snowplow/apis/templates/v1/core.go`) — and `grep -rin "pagination|per_page|nextPage|cursor"` over `apis/` and the restactions resolver finds nothing but a `resourcesrefs.go` comment. So a read panel over GitHub Actions runs or Jira issues can only ever fetch what one hard-coded `?per_page=` returns; there is no `Link`-header follow and no cursor walk. `dependsOn.iterator` gives fan-out but not page-walking. **The controller side solved this (`Pagination`, with a required `maxPages`); the read side has not.** Worth filing against snowplow.
2. **`Endpoint` auth is static too** **[V]**. `endpoints_cache.go` reads `token`, `username`, `password`, `client-certificate-data`, `certificate-authority-data`. So the ArgoCD session exchange blocks the *read* path as well as the controller — gap 2 is not only an oasgen gap.

---

## 6. What I could NOT verify

Stated plainly, because a marked gap is worth more than a confident invention.

**Backstage side**

1. **Plugin popularity.** "Widely used" in section 2b is my judgement from repo prominence and workspace presence. I found **no primary telemetry source** for Backstage plugin adoption. Any prioritisation that leans on "most used" is leaning on me, not on data.
2. **OpenAPI specs I could not fetch or locate**: Octopus Deploy (the `OctopusDeploy/OpenAPI` path 404'd **[V]**), Vault (repo-root `openapi.json` 404'd; it is served per-instance at `/v1/sys/internal/specs/openapi`), Dynatrace (404'd; published per-tenant), Azure DevOps (reference docs, no single consolidated document found), ServiceNow (per-instance), Quay (`/api/v1/discovery`, per-instance), Nexus (`/service/rest/swagger.json`, per-instance), lighthouse-audit-service (**none found**), tech-insights.
3. **Snyk** returns a *list of dated API versions* at `https://api.snyk.io/rest/openapi`, not a spec **[V]**. I did not fetch a dated version, so I have not confirmed its shape.
4. **SonarQube coverage.** I confirmed the v2 document is openapi 3.1.0 **[V]** but did **not** diff the endpoints the Backstage plugin calls (`api/measures/component`, `api/qualitygates/project_status`) against it. My "the plugin's endpoints are v1 and outside the spec" statement is **inference, not verification**.
5. **GitHub Actions run logs** are a 302 to a zip archive. I did not verify how RDC or snowplow behave on a redirect-to-binary. Closed issue #132 (*"307 not followed because the request body is unreplayable"*) suggests redirect handling has history worth reading first.
6. I read **UI component directory listings**, not the components. "What panels a plugin presents" is derived from directory names (`WorkflowRunLogs`, `RadarPlot`, `TriggerButton`), which is strong evidence of *what* but not of *how much*.
7. I did not survey the ~70 community workspaces outside the top ~25. The inventory is deliberately deep-not-wide, per the brief.

**Krateo side**

8. **Whether `prepareSchemaForCRD` drops inline array-of-object items.** The Keycloak KOG asserts it; the adapter source contradicts it. **Unresolved** — see §4 item 9.
9. **Whether schema-generation warnings (including skipped security schemes) are visible in `RestDefinition.status`.** Issue #151 says they were unreachable; it is closed; I did not read the fix.
10. **Whether `Table`'s React component supports per-row actions** beyond what the CRD exposes. I read the CRD only.
11. **Whether `ConfigurationField.FromOpenAPI` misses path-item params.** Inferred from "one adapter, one `GetParameters()`" — not probed.
12. **Whether a per-instance `<Kind>Configuration` can hold a *minted* credential at all**, or whether RDC's auth path assumes an immutable Secret read. I read the generation side, not RDC's `auth.ToType` consumer.
13. **Whether `krateo-blueprints/portal-pages` exists.** It does **not** appear in the `krateo-blueprints` repo listing **[V]**; the nearest are `portal-builder`, `krateo-portal-chart`, `blueprint-builder`, and local clones `composable-portal-*` / `portal-composition-page`. I did not inspect those for existing Backstage-shaped pages, so **some of the widget work in §5 may already be prototyped there.** Check before scoping.
14. **The Swagger-2.0 fix cost.** I verified rejection and that the pinned libopenapi fork is `v0.21.8`; I did **not** verify that `BuildV2Model()` on that fork exposes everything the four adapter methods need (notably `securityDefinitions` and `in: body` parameters).
15. **Whether a 2.0→3.0 offline conversion is a viable short-term workaround** for ArgoCD/Grafana. Plausible, untested, and it breaks the OAS-checksum guarantee by construction — see §4 item 1.

---

## Appendix: reproducing the three load-bearing probes

All three were run against the real repo at `eac184c` and then reverted (`git status` clean).

1. **Swagger 2.0 rejection** — drop a `*_test.go` in `internal/tools/oas2jsonschema/` that reads the fetched ArgoCD `swagger.json` and calls `NewLibOASParser().Parse(b)`. Result: `parser error [ModelBuildError]: failed to build V3 model: unable to build openapi document, supplied spec is a different version (oas2). Try 'BuildV2Model()'`.
2. **OAS 3.1.0 acceptance + path-item parameter blindness** — same harness, with a fixture declaring `tenantId`(path) and `apiVersion`(query) at path-item level and `dryRun`(query) at operation level. Result: parse OK; `GetParameters()` → `[dryRun/query]` only.
3. **Path-item parameter frequency in real specs** — count paths carrying a `parameters` key: Keycloak **273/274 (555 params)**, PagerDuty **7/274 (10)**, GitHub **0/816**, Jira **0/423**, SonarQube **0/196**.
