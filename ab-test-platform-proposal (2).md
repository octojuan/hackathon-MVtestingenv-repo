# Autonomous A/B Testing on PDLC + IXP
### Revised Hackathon Proposal

## What changed

The original proposal assumed we'd build traffic routing, metrics, and a statistics engine from scratch. We don't need to — **IXP already does all of that**, and **PDLC already does the clone/build/governance/approval layer**. The only genuinely new thing to build is the connective tissue between them: an agent that watches an IXP experiment, decides a winner, and pushes the merge back through PDLC's own approval flow.

This keeps the fixed/flexible split PDLC is built around: PDLC's fixed core (orchestration, artifact contracts, approval checkpoints, Critic/Judge quality pattern, telemetry) stays untouched. IXP is the flexible layer doing the domain-specific work (blueprints, routing, analytics, stats). The new decision-and-merge-back agent is a team-level customization that plugs into both without forking either.

## Hypothesis

If an agent can read an IXP experiment's live results through IXP's own APIs/MCP, apply a fixed win/kill rule (statistical significance on the primary metric + guardrails intact), and hand the outcome to PDLC as a normal PR through PDLC's existing approval checkpoint — and then, once that PR merges, automatically clean up the losing variant's code and the resolved flag — then we get end-to-end autonomous experimentation — clone to merge to **clean** — without building or maintaining a separate routing or stats stack, without bypassing governance, and without leaving flag debt behind.

We'll know we're right if: an experiment registered as a Blueprint in IXP can run to a decision with no manual dashboard-watching, the resulting PR lands in the target repo through PDLC's normal review gate with full traceability back to the originating PRD and Blueprint ID, and the losing branch/flag/dead code are removed via a second governed PR with no manual flag-debt cleanup required afterward.

**Main proposal (this revision):** PDLC + IXP integration where an agent automates the *post-decision cleanup* — removing the losing branch, merging the winning path to main, and deleting the flag. This fills a real, unaddressed gap: IXP determines winners and routes traffic, but nothing today automates what happens to the losing code and the flag once a decision is made. That gap is confirmed by inspecting IXP's own experiment/flag UI (Experiments, Feature Flags), which shows experiment lifecycle status (Draft/Running/Stopped) and results, but no "auto-cleanup" action tied to a stopped experiment.

## Architecture

![Architecture diagram](architecture_v3.png)

## What each system owns

**Important boundary: IXP does not clone or build code.** It's purely the experimentation layer — Blueprint, experiment config, traffic assignment, metrics, and stats. PDLC is what clones the branch and builds/deploys it as a running service. The two systems meet at exactly one integration point: **registering PDLC's deployed service as a DevPortal asset**, which is what IXP's experiment actually routes traffic to. Everything else in each system is otherwise independent.

### PDLC (fixed core — unchanged)
- Clones the branch, orchestrates the lifecycle, enforces artifact contracts and traceability
- Builds and deploys the variant as a reachable service
- Owns the human approval checkpoint and the Critic/Judge quality pattern on the merge PR
- Owns telemetry and framework governance

### The handoff: DevPortal asset registration
This is the one place the two systems talk to each other, and it's the piece that doesn't exist yet. Once PDLC has built and deployed the treatment as a running service, that service needs to be registered as a DevPortal asset so step 08 of the IXP flow (*"Attach the DevPortal asset and Spectrum... so assignments resolve"*) has something to point at. Without this step, IXP has an experiment with nowhere to route traffic. This registration call — likely a small script or an added stage in the PDLC pipeline — is worth scoping explicitly; it's small, but everything downstream depends on it.

### IXP (flexible layer — does the experimentation work, not the code)
- **Blueprint creation**: PM/agent uploads the PRD; IXP extracts hypothesis, audience, owners (via MCP: *"Use PRD to create a blueprint in e2e using business unit CG"*)
- **Convert to experiment draft**: auto-fills the experiment from the Blueprint
- **Configuration**: PD attaches the DevPortal asset (produced by the handoff above) and Spectrum, sets segmentation rules — this *is* our traffic routing, no custom router needed, but it only works once the asset exists
- **QA & promotion**: validate in e2e, collect sign-offs, promote to prod
- **Metrics & analysis**: DS defines primary/guardrail/exploratory metrics; IXP's dashboard runs the Bayesian read (AlphaBeta, now native), applies CAR by default, and reports confidence/credible intervals — no custom stats engine needed

### New: Decision & merge-back agent
This is the only new build. It:
1. **Watches** the experiment via IXP's API/MCP on a schedule (or on a webhook if IXP exposes one)
2. **Applies a win/kill rule**: primary metric significant *and* every guardrail intact → winner; any guardrail violated → kill switch, route 100% back to control via IXP
3. **On a win**, opens a PR against the PDLC-managed repo to merge the winning branch into main, tagged with the originating PRD, Blueprint ID, and experiment ID for traceability
4. **Does not auto-merge** — the PR still passes through PDLC's existing Critic/Judge pattern and human approval checkpoint, so governance is never bypassed
5. **On merge, cleans up flag debt** — once the merge PR lands, the agent opens a second, separate cleanup PR that deletes the losing variant's dead code path and removes the now-resolved flag/experiment key from source, then deletes the IXP experiment's flag/config via its API and archives the losing branch. This cleanup PR goes through the same Critic/Judge + human approval gate as the merge PR — it is never auto-merged either.

**Why this matters:** IXP determines a winner and can ramp traffic to it, but nothing in IXP or PDLC today removes the losing code path from the repo or decommissions the flag afterward. That's a universal problem with feature-flag platforms — "flag debt" (stale flags and dead branches nobody cleans up) accumulates because promoting a winner and cleaning up the loser are treated as separate, manual steps. This agent is the first thing that closes that loop automatically, without skipping governance.

## Hackathon-scope MVP

1. PDLC clones the branch, builds it, and deploys the treatment as a reachable service
2. **New piece**: register that deployed service as a DevPortal asset (the handoff)
3. PM drops a PRD into e2e IXP; a Blueprint is created via the IXP MCP prompt
4. Blueprint converts to an experiment draft; PD attaches the DevPortal asset from step 2 + Spectrum (traffic routing is IXP's job, not ours, but it needs that asset to exist first)
5. Promote to prod; IXP starts collecting metrics
6. **New piece**: a scheduled job (or Builder.io agent — see the companion prompt) polls the IXP experiment's dashboard/API, checks primary metric significance and guardrails
7. On a clear win, the agent opens a PR to the PDLC repo with the Blueprint ID and experiment ID in the description
8. PDLC's normal review/approval gate takes it from there
9. **New piece**: once the merge PR lands, the agent opens a second cleanup PR — removing the losing variant's dead code path and flag key, deleting the IXP flag/experiment config via API, and archiving the losing branch
10. PDLC's normal review/approval gate takes the cleanup PR from there too — same governance, no shortcuts

## What's now out of scope (moved to IXP)

- ~~Traffic routing / assignment service~~ — IXP (DevPortal + Spectrum), once the asset is registered
- ~~Metrics pipeline~~ — IXP dashboard, base metrics & metric groups
- ~~Statistical decision engine~~ — IXP's native Bayesian/CAR analysis
- ~~Kill switch infrastructure~~ — IXP promote/rollback controls

## What's newly in scope

- **DevPortal asset registration** — the connective step between PDLC's build output and IXP's experiment config; doesn't exist today and needs to be scoped as part of the PDLC pipeline or as a small standalone script
- **Decision & merge-back agent** — as before, watches IXP and opens the PR
- **Post-decision cleanup (flag debt)** — after the merge PR lands, the agent opens a cleanup PR removing the losing variant's dead code path and the resolved flag/experiment key, deletes the IXP flag/experiment config via API, and archives the losing branch — still gated through Critic/Judge + human approval, never auto-merged

## What's still out of scope (confirmed not duplicating IXP)

- Winner determination itself — IXP's stats engine decides significance; the agent only reads that result and acts on it
- Traffic ramping to the winner — handled by IXP's promote controls, not rebuilt here
- The cleanup step described above is genuinely new: IXP's own experiment UI has no "delete flag + remove dead code + merge" action once an experiment stops, so this does not duplicate existing IXP functionality.

## Open questions

- Does IXP expose a webhook or event stream for "experiment reached significance," or does the agent need to poll the dashboard/API on an interval?
- What's the minimum experiment duration/sample size IXP enforces before a "win" is trustworthy — does the agent need its own guard against early stopping, or does IXP already prevent peeking?
- Where does the PR target repo's branch protection sit relative to PDLC's approval checkpoint — do we need the agent to wait on a specific PDLC-recognized label/status before it's considered "in flight" for governance purposes?
- Confirm the two IXP service accounts (prod / e2e & qal) have access to whatever repo or Drive location holds the PRD, so Blueprint extraction doesn't silently return nothing.
- Does IXP expose a "delete flag/experiment config" API endpoint the agent can call directly, or does flag deletion require a manual step in the IXP UI even after this agent's cleanup PR merges?
- How do we safely identify "dead code path" for automated removal across different languages/frameworks — does the agent need language-specific static analysis, or should cleanup scope start with flag-key removal only and leave broader dead-code detection to a follow-on phase?
