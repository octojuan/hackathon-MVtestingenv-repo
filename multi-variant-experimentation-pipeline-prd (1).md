# Multi-Variant Experimentation Pipeline PRD

PRD Lock-In Date: __________________

## Intake JIRA

*This project is associated with intake TBD (hackathon proposal — no intake ticket filed yet).*

*Source Brief: ../research/PRODUCT_BRIEF-multi-variant-testing-2026-09-14.md*

# **DACER**

Guidance for DACER is based on the product/feature ownership defined in the [product management rubric](https://docs.google.com/spreadsheets/d/1bRhZrzozCPx4fhvcOBvUtFYtsifFp1vDg_XUhKk9AWQ/edit?gid=30378849#gid=30378849).

| Role | Names of People | Status |
| :---- | :---- | :---- |
| **Driver** | PM (Primary Owner): Lissa Miller | Not started |
|  | PD: [Name] | Not started |
|  | PE: [Name] | Not started |
|  | XD: [Name] | Not started |
| **Approver** | PM - Lead: [Name] | Not started |
| **Contributor** | IXP platform team; ICS (Intuit Consumer Support) data science team | Not started |
| **Informed** | [Name] | Not started |
| **Escalation** | [Name] | Not started (Optional) |
| **Reviewers (optional)** | IXP platform team (SME on existing feature-flag/experimentation infrastructure) |  |

# **Background & Current State**

Intuit's experimentation infrastructure grew out of a consolidation of LaunchDarkly (feature flags) into IXP (Intuit Experimentation Platform) in FY20–21, driven partly by vendor cost savings (~$968K–$1.19M/year). That consolidation matured IXP as a feature-flagging and experimentation platform — IXP today natively provides Blueprint-based experiment setup, traffic routing/assignment (DevPortal asset + Spectrum), guardrail metrics, and a Bayesian/CAR statistical read. Separately, PDLC provides the fixed-core pipeline for cloning a branch, building/deploying a variant, and gating any merge to main behind a Critic/Judge review and human approval checkpoint.

What doesn't exist today is the connective tissue between the two: PDLC has no way to register a deployed variant as something IXP can route traffic to, and nothing watches an IXP experiment, decides a winner against a fixed rule, and pushes that decision back through PDLC's approval flow. Teams are left either hand-driving that watch-and-merge step, or — for teams not onboarded to IXP at all (e.g., Intuit Consumer Support/ICS) — building ad hoc, unsupported statistics tooling from scratch.

This PRD proposes a hackathon-validated prototype of that connective layer: a DevPortal asset registration handoff between PDLC and IXP, and a new decision-and-merge-back agent that closes the loop from experiment result to a governed PR — built on top of PDLC's framework, without forking or duplicating either system.

# **Problem Statements**

**Customer problem statement (engineer/data scientist running experiments):**

* **I am…** an Intuit engineer or data scientist trying to validate a product change with real customer traffic.
* **I am trying to…** run a rigorous multi-variant (more than two-arm) experiment and safely promote the winning variant, end-to-end, without manually watching a dashboard and hand-driving the merge.
* **But…** no connective layer exists between PDLC (which builds and deploys the variant) and IXP (which routes traffic and computes the statistical read) — so even though the routing and stats already exist in IXP, someone still has to register the deployed service as a DevPortal asset by hand, watch the experiment to completion, and manually open the merge PR.
* **Because…** IXP and PDLC matured independently, each solving its own half of the problem, with no agent or integration bridging the two.
* **Which makes me feel…** either blocked from testing rigorously (if not onboarded to IXP at all), or stuck doing manual, error-prone busywork to get from a decided experiment to a merged winner.

**Business problem statement:**

* **I am…** a product organization at Intuit that depends on experimentation to make good product decisions.
* **I am trying to…** ensure every team can validate changes with statistically sound, safely-run experiments that reach production without manual bottlenecks or governance being bypassed.
* **But…** experimentation rigor varies widely by team, some teams run experiments outside any shared, auditable platform, and even teams using IXP still rely on someone manually watching results and driving the merge.
* **Because…** there's no self-service, end-to-end pipeline connecting PDLC's build/deploy/governance layer to IXP's routing/stats layer.
* **Which makes me feel…** exposed to the risk of wrong product decisions, delayed rollouts, or ungoverned merges driven by manual gaps in the handoff.

# **Ideal State**

* **In a perfect world:** A PRD becomes a Blueprint, a Blueprint becomes a running IXP experiment routed to a PDLC-deployed variant, and once the experiment reaches a clear win, a PR merges the winner through PDLC's normal approval gate — with no manual dashboard-watching, no hand-built routing, and no hand-built stats, from clone to merge.
* **The biggest benefit to me is:** I trust my experiment results because IXP enforces statistical rigor and traffic routing by default, and I trust the outcome reaches production safely because PDLC's governance checkpoint is never bypassed.
* **Which makes me feel:** Confident that product decisions are backed by real evidence and that the path from decision to production is fast *and* safe.

# **Business Goals**

- Close the gap between PDLC's build/deploy/governance layer and IXP's routing/stats layer, so no team has to hand-build or hand-drive either half.
- Reduce the number of teams building unsupported, ad hoc experimentation tooling outside any shared platform (directly evidenced: ICS data science team's DIY Databricks notebook for sample sizing).
- Increase the rigor and safety of multi-variant experiments run across Intuit, reducing the risk of decisions driven by underpowered analysis or manually-driven, governance-bypassing merges.
- Validate, via a low-cost hackathon prototype, whether this integration is broad enough to justify a full platform investment before committing significant engineering resources.

# **Success Metrics**

| Metric | Description | Target |
| :---- | :---- | :---- |
| Experiments run clone-to-merge with no manual watch/merge step | Number of hackathon-pilot experiments where the decision-and-merge-back agent opens the PR with no human polling the IXP dashboard | 100% of participating pilot teams (target: at least 2 teams, including 1 not currently onboarded to IXP) |
| Time from experiment decision to merge PR opened | Time from IXP reaching statistical significance to the agent opening a PR against the PDLC-managed repo | Reduce to near-zero manual latency vs. current ad hoc, human-driven merge process (baseline to be measured during pilot) |
| Governance integrity | % of winning-variant merges that pass through PDLC's existing Critic/Judge + human approval checkpoint (i.e., none auto-merged) | 100% |

# **Key Features**

* ***DevPortal Asset Registration (the handoff):*** Registers a PDLC-built and deployed variant as a DevPortal asset, so IXP's experiment (via Spectrum) has something to route traffic to. This is the one new integration point between the two systems — PDLC does not otherwise clone or build inside IXP, and IXP does not otherwise clone or build code.
* ***IXP-Native Blueprint & Experiment Setup:*** PRD is used (via IXP's MCP prompt) to create a Blueprint, which converts to an experiment draft; PD attaches the DevPortal asset + Spectrum and sets segmentation — no custom routing built.
* ***IXP-Native Traffic Routing:*** IXP routes live traffic between control/treatment assets once QA and approvals promote the experiment to prod — no custom router built.
* ***IXP-Native Statistical Decision Engine:*** IXP's dashboard reports primary + guardrail metrics using native Bayesian/CAR analysis — no custom stats engine built.
* ***New: Decision & Merge-Back Agent:*** Watches the live IXP experiment via IXP's API/MCP (polling or webhook), applies a fixed win/kill rule (primary metric significant *and* every guardrail intact → winner; any guardrail violated → kill), and on a win opens a PR against the PDLC-managed repo — tagged with the originating PRD, Blueprint ID, and experiment ID — that still passes through PDLC's existing Critic/Judge and human approval checkpoint before merging to main. On a loss or guardrail failure, it triggers IXP's kill switch to route 100% of traffic back to control.

# **Use Cases & Functional Requirements**

| ID | Priority | Persona | Use Case | Description | Acceptance Criteria | Scope |
| ----- | ----- | ----- | ----- | ----- | ----- | ----- |
| 1.1 | P0 | PD / engineer configuring an experiment | As a PD, I want the PDLC-deployed variant registered as a DevPortal asset so that IXP's experiment has something to route traffic to. | This is the one integration point between PDLC and IXP; without it, IXP has an experiment config with nowhere to send traffic. | Once PDLC builds and deploys a variant as a reachable service, the system must register that service as a DevPortal asset (via a pipeline stage or standalone script) and surface the resulting asset ID so it can be attached to the IXP experiment draft. | DevPortal Asset Registration |
| 1.2 | P0 | Engineer running a multi-variant test | As an engineer, I want a PRD to become an IXP Blueprint and experiment draft so that I don't have to hand-configure variant routing per team. | The engineer needs Blueprint creation and conversion to an experiment draft handled by IXP's own MCP prompt and UI, not custom-built. | Given a PRD, the system must produce an IXP Blueprint (via IXP's MCP: "Use PRD to create a blueprint...") that converts to an experiment draft to which the DevPortal asset from 1.1 can be attached. | IXP-Native Blueprint & Experiment Setup |
| 1.3 | P0 | Engineer running a multi-variant test | As an engineer, I want the decision-and-merge-back agent to halt an in-flight experiment immediately when a guardrail fails so that I can stop customer exposure to a bad variant. | The agent needs to detect a guardrail violation and trigger IXP's existing kill-switch/rollback control, not a custom-built one. | On detecting any guardrail violation while watching the experiment, the agent must trigger IXP's promote/rollback control to route 100% of traffic back to control, without requiring a full code deploy. | New: Decision & Merge-Back Agent |
| 1.4 | P0 | Engineer running a multi-variant test | As an engineer, I want the winning variant merged only after IXP confirms significance and guardrails are intact, and only through PDLC's existing approval checkpoint, so that I avoid "auto-merging straight to main." | The agent must apply a fixed win/kill rule and open a PR rather than auto-merge. | On primary-metric significance with all guardrails intact, the agent must open a PR against the PDLC-managed repo, tagged with the originating PRD, Blueprint ID, and experiment ID, and that PR must pass through PDLC's existing Critic/Judge pattern and human approval checkpoint before it can merge to main. | New: Decision & Merge-Back Agent |
| 1.5 | P1 | PM or data scientist reviewing results | As a PM, I want to see experiment status and results on IXP's own dashboard so that I don't need custom reporting per experiment. | The PM needs visibility into in-flight and completed experiments via IXP's existing dashboard, not a new reporting surface. | IXP's dashboard must display, per experiment: current status, traffic split per variant, and the statistical read (significant / not yet / inconclusive), and the decision-and-merge-back agent must be able to read this same data via IXP's API/MCP. | IXP-Native Statistical Decision Engine |
| 1.6 | P2 | Data scientist not onboarded to IXP (e.g., ICS) | As a data scientist without IXP access, I want a path to this pipeline's benefits so that lack of IXP onboarding doesn't block me from rigorous experimentation. | This use case depends on IXP onboarding being resolved for the team; it is not solved by the agent itself. | The system must document the IXP onboarding path required before a team can use the DevPortal-asset handoff and decision-and-merge-back agent, and flag onboarding as a pilot-selection dependency rather than building a parallel non-IXP routing path. | Platform Independence (dependency, not new build) |

# **Non-Functional Requirements**

*Latency, volume, availability, performance constraints, scalability, observability, etc.*

- **Latency:** The decision-and-merge-back agent's polling interval (or webhook reaction time, if IXP exposes one) must be frequent enough that a decided experiment doesn't sit un-merged for an unreasonable window — target [TBD — needs: PE estimate] between IXP reaching significance and the PR opening.
- **Volume:** Must support at least the scale of a single pilot team's traffic during the hackathon validation (exact volume TBD per pilot team selected) — traffic volume itself is handled entirely by IXP, not this pipeline.
- **Availability:** The kill-switch trigger path (agent → IXP promote/rollback control) must remain available even if the merge-back/PR-opening path is degraded — treat guardrail-driven kill as a higher-availability-tier action than win-driven merge.
- **Observability:** Every DevPortal asset registration, win/kill decision, and PR opened by the agent must be logged with enough detail to trace back to the originating PRD, Blueprint ID, and experiment ID, given the evidenced problem of teams needing to "diff" experiment/flag state across environments.

## **Security Requirements**

No new PII/PCI data is introduced by this pipeline — it operates on existing traffic and assignment data already flowing through IXP, and existing merge/PR data already flowing through PDLC. Standard internal access controls apply to who can configure experiments, trigger the kill switch, and approve the agent's merge-back PR. The agent itself must never bypass PDLC's human approval checkpoint.

## **Legal Requirements**

None identified specific to this internal tooling initiative at this stage.

## **Internationalization Requirements**

Not applicable for the hackathon-scope pilot; revisit if a future full build extends to teams serving non-US audiences (e.g., TurboTax Canada, GBSG international regions).

# **Dependencies / Risks / Assumptions**

- **Dependency:** Pilot team selection — at least one IXP-onboarded team needed to validate the full DevPortal-asset-handoff and decision-and-merge-back agent flow; a non-onboarded team (e.g., ICS) surfaces the separate onboarding dependency (use case 1.6) rather than being solved by the agent itself.
- **Dependency:** Whether IXP exposes a webhook or event stream for "experiment reached significance," or the agent must poll the dashboard/API on an interval — affects latency NFR above and needs confirmation with the IXP platform team.
- **Dependency:** IXP's minimum experiment duration/sample-size enforcement before a "win" is trustworthy — need to confirm whether IXP already guards against early stopping/peeking, or whether the agent needs its own guard.
- **Dependency:** Where the PR target repo's branch protection sits relative to PDLC's approval checkpoint — may require the agent to wait on a specific PDLC-recognized label/status before a PR is considered "in flight" for governance purposes.
- **Dependency:** Confirm the two IXP service accounts (prod / e2e & qal) have access to whatever repo or Drive location holds the PRD, so Blueprint extraction doesn't silently return nothing.
- **Risk:** Evidence for the underlying problem currently rests on two sources (one internal gap-analysis doc, one Slack account from a single ICS data scientist) — Confidence is Moderate (80%), not Strong. If broader validation surfaces this is a narrow, one-team problem, scope should shrink accordingly.
- **Risk:** Building even a hackathon-scope prototype risks scope creep into "replace IXP" or "replace PDLC" territory — this PRD explicitly scopes to the connective handoff and merge-back agent only, not a full platform replacement (see MVP Scope below).
- **Assumption:** IXP platform team is a Contributor, not blocked from this effort proceeding independently as a hackathon prototype — final production ownership (standalone agent vs. absorbed into IXP or PDLC) is an open question, not a blocker to piloting.

# **Discovery Research and Problem Validation**

## **Market research (optional) learnings**

Not conducted for this initiative — internal tooling gap, not a competitive/market-facing feature.

## **VOC / Customer Feedback**

Internal "customer" here is the Intuit engineer/data scientist, not an external end user. Key evidence:

- A data scientist on Intuit Consumer Support (ICS) reported their team is "not onboard to IXP," maintaining an ad hoc Databricks notebook for sample-size calculations, with unresolved questions about statistical balance under skewed/seasonal traffic. *(Slack, #ab-testing-and-causal-inference, 2026-08-18)*
- Engineers in support channels continue to confuse the "Experiment" module (tied to IXP) vs. the separate "Multi-Variant Experience" (MVE) module for WebX A/B tests, as recently as September 2026. *(Slack, #dotcom-support, 2026-09-03)*
- Full findings and citations are in the upstream Insights Report: `../research/ab-testing-internal-problems-2026-09-14.md`

## **Data Analysis**

No quantitative usage-analytics pull was performed for this PRD (out of scope for a hackathon-stage proposal). The Insights Report's evidence is qualitative (Slack threads, internal docs) — reach and frequency are marked TBD in the Success Metrics and Impact Sizing sections of the source Brief pending a PD/data estimate.

## **Information audit (secondary research)**

- Internal doc: "Multi-variant test environment" (gap-analysis, owned by Lissa Miller) — frames the full pipeline requirement, refined by the revised proposal into a PDLC+IXP connective-layer scope rather than a from-scratch build.
- Internal doc: "Feature Flag Tech Debt" — background on IXP feature-flag governance issues (related but distinct problem space; not in scope here).
- Internal deck: "IXP Feature Flag Capability" (2019/2020) — historical context on the LaunchDarkly-to-IXP consolidation.
- Revised Hackathon Proposal: "Autonomous A/B Testing on PDLC + IXP" — defines the DevPortal-asset handoff and decision-and-merge-back agent this PRD specifies. `../ab-test-platform-proposal.md`
- Source Brief: `../research/PRODUCT_BRIEF-multi-variant-testing-2026-09-14.md`

# **Milestones and Releases**

## **MVP Scope**

Hackathon-scope prototype: (1) the DevPortal asset registration handoff from PDLC's build output to IXP, and (2) the decision-and-merge-back agent watching one pilot experiment and opening a PR on a clear win, validated against one pilot team's real use case. Explicitly **out of scope** for MVP: any custom traffic routing, statistics engine, or kill-switch infrastructure (all provided natively by IXP), a new reporting/dashboard surface (use IXP's own dashboard), and a non-IXP onboarding path for teams like ICS (tracked as a separate dependency, use case 1.6).

## **MILESTONE RELEASE SCHEDULE & SCOPE**

| Milestones | Target Date | Description |
| :---- | :---- | :---- |
| Milestone #1 | [TBD — hackathon date] | Working prototype: DevPortal asset registration handoff + decision-and-merge-back agent demoed against one pilot team's IXP experiment, ending in a governed PR |
| Milestone #2 | [TBD] | Pilot validation: run one real experiment end-to-end (clone → deploy → Blueprint → experiment → decision → merge PR) with an IXP-onboarded pilot team |
| Milestone #3 | [TBD] | Go/no-go decision on full platform investment, based on pilot results and broader team validation, including resolution of the ICS/non-onboarded-team dependency |

*Additional context: this schedule assumes the hackathon prototype (Milestone #1) is the immediate deliverable; Milestones #2–3 depend on hackathon outcome and are not committed dates.*

# **Open Questions and Decisions**

| Description | Type | Person Responsible | Status | Date to Close |
| :---- | :---- | :---- | :---- | :---- |
| Does IXP expose a webhook/event stream for "experiment reached significance," or must the agent poll? | Open Question | [Name] | Not started | Before Milestone #1 |
| Does IXP already guard against early stopping/peeking, or does the agent need its own guard before treating a result as a trustworthy "win"? | Open Question | [Name] | Not started | Before Milestone #1 |
| Where does the PR target repo's branch protection sit relative to PDLC's approval checkpoint — does the agent need to wait on a specific PDLC-recognized label/status? | Open Question | [Name] | Not started | Before Milestone #1 |
| How many teams are actually in a "not onboarded to IXP, building DIY stats tooling" position? | Open Question | [Name] | Not started | Before Milestone #3 go/no-go |
| Should the decision-and-merge-back agent live as a standalone tool or be absorbed into IXP or PDLC as a platform feature? | Open Question | [Name] | Not started | Before Milestone #3 go/no-go |
| What is the actual Reach (# of teams/engineers affected) for Impact Sizing? | Open Question | [Name] | Not started | Before full build commitment |

# **Appendix**

## **Related Artifacts**

| Artifact | Description | Links |
| ----- | ----- | ----- |
| Insights Report | Full `/research` synthesis: 11 findings, theme breakdown, trend analysis | `../research/ab-testing-internal-problems-2026-09-14.md` |
| Product Brief | 1-page upstream alignment brief for this initiative | `../research/PRODUCT_BRIEF-multi-variant-testing-2026-09-14.md` |
| Revised Hackathon Proposal | Defines the PDLC+IXP connective layer and decision-and-merge-back agent scope | `../ab-test-platform-proposal.md` |
| Architecture diagram | PDLC ↔ IXP integration and decision-and-merge-back agent flow | `../canary_architecture.png` |
| Internal doc: Multi-variant test environment | Original gap-analysis doc; scope since refined per the revised proposal | Google Drive (see Insights Report references) |
| Internal doc: Feature Flag Tech Debt | Related but distinct feature-flag governance problem space | Google Drive (see Insights Report references) |