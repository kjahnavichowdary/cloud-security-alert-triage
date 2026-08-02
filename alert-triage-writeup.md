# Alert triage workflow — design write-up

## Problem statement

Security engineers in multi-cloud environments (AWS, Azure, GCP) receive a high volume of alerts from CSPM/CWPP tooling — misconfigurations, exposed resources, anomalous IAM activity. Alerts arrive with inconsistent context, so investigating one means pivoting across several tools: the cloud console for resource state, IAM for who can access it, log tooling for activity history, and a ticketing system to document the outcome. That fragmentation slows triage, contributes to alert fatigue, and increases the chance that a real threat sits un-investigated behind a stack of noise.

The workflow needs to answer three questions fast, in one place: **what happened, how bad is it, and what do I do about it** — without losing the audit trail that security work requires.

## User persona

**Priya, cloud security engineer** — works on a small security team responsible for a large multi-cloud estate (hundreds of accounts/projects). Triages 30–50 alerts a day across severities, mostly solo, with occasional escalation to a senior engineer or the resource-owning team. Comfortable in cloud consoles and CLI tools but wants to spend her attention on judgment calls, not re-navigation. Needs every action she takes to be defensible later — reasons, timestamps, and evidence matter as much as the fix itself.

## Proposed screens (see wireframe)

1. **Alert queue** — the day's alerts, filterable by severity, cloud, resource type, and status, with saved views for the filters engineers reuse constantly (e.g. "public-facing risk").
2. **Investigate** — an alert's full context in one view: what changed, a timeline, a config diff, and a **blast radius panel** (public exposure, sensitive-data tags, who has access) so she doesn't have to open IAM or the cloud console separately.
3. **Resolve** — the remediation decision: apply a suggested fix (with human approval), open a ticket with context pre-filled, or follow a manual checklist — always closing with an audit entry.

## Features and prioritization

| Priority | Feature | Why |
|---|---|---|
| P0 | Unified alert queue with severity/cloud/status filters | Baseline usability — nothing else matters if she can't find the right alert |
| P0 | Blast-radius panel (public exposure, sensitive data, access) | This is the actual bottleneck today — the reason engineers tab out to 3 other tools |
| P0 | Audit log on every action | Compliance requirement, not optional — must ship with v1, not bolted on later |
| P1 | Suggested-fix generation (patch/diff) | High leverage once blast radius is reliable, but needs the enrichment data first |
| P1 | Saved/shared filter views | Cheap to build, large daily time saving |
| P2 | One-click auto-remediation (no approval step) | Valuable but carries real risk — sequence after the suggest-only version earns trust |
| P2 | Cross-alert correlation ("related alerts") | Nice-to-have depth; not needed to resolve a single alert competently |

The sequencing logic: ship the parts that remove *investigation* friction (queue, blast radius, audit trail) before the parts that remove *remediation* friction (auto-fix), because a wrong or premature auto-fix is a worse outcome than a slow manual one.

## Success metrics

- **Median time-to-triage** (alert opened → decision made) — the direct target of this workflow.
- **% of alerts resolved without leaving the tool** — proxy for how well the blast-radius panel replaces console-hopping.
- **False-positive rate at close-out** — tracked from the required "reason" field; should trend down as blast-radius accuracy improves.
- **Suggested-fix acceptance rate** — leading indicator of whether engineers trust the auto-generated remediation, which gates whether P2 auto-remediation is worth building.

---

## Bonus: engineering scope (phases, dependencies, risks)

### Phases

1. **Alert ingestion & normalization** — unify alerts from different CSPM/CWPP sources into one schema (severity, resource, rule, timestamp). Foundational; everything else reads from this.
2. **Alert queue UI** — list, filter, saved views. Depends on (1).
3. **Context enrichment service** — computes blast radius: public exposure, sensitive-data tags, IAM access graph. Can be built in parallel with (2), but is the hard engineering problem in this project — it's effectively a lightweight access-graph analysis across multiple cloud providers.
4. **Investigate view** — timeline, config diff, blast-radius panel. Depends on (2) and (3); this screen is only as good as the enrichment data behind it.
5. **Remediation & audit engine** — suggested fixes, ticket creation, audit logging. Depends on (4); also depends on write-access integrations into customer cloud accounts and the ticketing system.
6. **Metrics & reporting** — MTTR, false-positive rate, fix acceptance rate. Depends on (5) for the underlying event data.

### Key dependencies

- The investigate view is **only as trustworthy as the enrichment service** — shipping it before blast-radius data is accurate risks engineers ignoring the panel entirely.
- Any remediation action needs **audit logging in place first**, not after — compliance requirement, not a follow-on feature.
- Auto-fix requires **write-access permissions into the customer's cloud environment**, which is a security/trust decision, not just an engineering task — likely needs its own review and staged rollout.

### Risks

- **Data quality across CSPM sources** — different tools score severity and describe resources differently; reconciling that is more work than it looks and can quietly delay everything downstream.
- **Enrichment accuracy** — computing a reliable blast radius (especially IAM reachability) across multi-cloud is non-trivial; a wrong "this isn't exposed" is worse than no answer at all.
- **Write-access trust** — granting the tool permission to modify customer cloud resources is a real security decision. Mitigate by shipping suggest-only remediation first and gating true one-click auto-fix behind a separate approval.
- **Performance at alert volume** — the queue needs to stay responsive with high alert counts and near-real-time updates; worth load-testing early rather than discovering it late.
- **Adoption** — engineers already have muscle memory in native cloud consoles; the tool has to be reliably faster, not just theoretically better, or it won't get used.

### Discussion items for the dev team

- Which CSPM/CWPP sources need to be supported at launch, and what's the ingestion mechanism (webhook, polling, pub/sub)?
- Is remediation suggest-only or auto-execute at v1 — and who owns that risk decision?
- How is multi-tenant access scoped (an engineer should only see alerts for accounts they're authorized on)?
- Real-time queue updates vs. polling — what's the latency budget?
- Audit log retention and export requirements for compliance.
