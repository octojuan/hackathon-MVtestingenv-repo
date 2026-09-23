# Project Brief: Integrated Multi-Variant Experimentation Pipeline

**(D) [TBD] | (A) [TBD] | (C) [TBD] | (E) [TBD]**

---

## Description

An autonomous, self-service experimentation pipeline that connects PDLC (clone/build/deploy, governance, approval) to IXP (traffic routing, metrics, statistical decision engine) via a single new integration point and a new decision-and-merge-back agent — so Intuit engineers get end-to-end, clone-to-merge multi-variant experimentation without hand-building routing, stats, or kill-switch infrastructure, and without bypassing governance.

---

## Problem

We believe Intuit product builders trying to run multi-variant (not just on/off flag) experiments struggle to do so reliably and quickly because no connective layer exists between PDLC's build/deploy/governance pipeline and IXP's experimentation capabilities — forcing teams to either build ad hoc tooling themselves, watch dashboards manually to decide and merge winners, or skip rigorous experimentation altogether.

---

## Why

- An internal gap-analysis doc lays out what a full multi-variant pipeline requires — variant build/deploy, traffic routing, instrumentation, a statistical decision engine (sample-size, anti-peeking), automated merge-back, and a kill switch. On closer look, **IXP already provides traffic routing, metrics, and the statistics engine (native Bayesian/CAR analysis)**, and **PDLC already provides the clone/build/deploy/governance layer** — the actual gap is the connective tissue between them: registering a PDLC-built service as a DevPortal asset IXP can route to, and an agent that watches the experiment, decides a winner, and pushes the merge back through PDLC's approval flow. The original doc warns: "auto-merging straight to main is how you get a great demo and a bad root-cause doc later," which is why the merge-back agent opens a PR through PDLC's existing Critic/Judge and human-approval checkpoint rather than auto-merging. *(Internal doc: "Multi-variant test environment")*
- A data scientist on the Intuit Consumer Support (ICS) team reported their team is "not onboard to IXP," forcing them to maintain their own ad hoc Databricks notebook for sample-size calculations — with open, unresolved questions about handling seasonal/skewed traffic for statistical balance. *(Slack, #ab-testing-and-causal-inference, 2026-08-18)*
- The tool fragmentation this pipeline gap sits inside is long-standing: as recently as September 2026, engineers in support channels were still asking which module to use for a WebX A/B test (the "Experiment" module tied to IXP vs. the separate "Multi-Variant Experience" module) — a confusion tracing back to the original FY20–21 LaunchDarkly-to-IXP consolidation. *(Slack, #dotcom-support, 2026-09-03; internal deck, "IXP Feature Flag Capability")*

---

## Success

| Metric | Baseline | Goal | Timeline |
|--------|----------|------|----------|
| Experiments that run clone-to-merge with no manual dashboard-watching or custom routing/stats tooling | N/A — new capability | 100% of participating hackathon-pilot teams launch a Blueprint-to-decision experiment through the shared PDLC + IXP pipeline, with the merge-back agent opening the PR and PDLC's governance gate the only manual step | [TBD — hackathon/pilot deadline] |

---

## Audience

**Who:** Product triad running product experiments who are either (a) not onboarded to IXP (e.g., ICS) or (b) onboarded but currently rely on manually watching an IXP experiment dashboard and hand-driving the merge once it's decided.
**Why now:** This gap has been independently identified twice from different angles (a platform gap-analysis and a live team's DIY workaround) — evidence is converging rather than isolated, and the confusion it sits inside has persisted 5+ years unresolved.

---

## What

- **DevPortal asset registration** — the one new integration point where PDLC's deployed variant is registered as an asset IXP can route traffic to (via Spectrum), so IXP's experiment has something to attach to
- **IXP-native experimentation** — Blueprint creation from a PRD, conversion to an experiment draft, QA/promotion, and traffic routing/assignment across variants, all handled by IXP rather than custom-built
- **IXP-native statistical decision engine** — sample-size calculation, guardrail metrics, and Bayesian/CAR analysis on IXP's dashboard, rather than a custom stats engine
- **New decision & merge-back agent** — watches the live experiment via IXP's API/MCP, applies a fixed win/kill rule (primary metric significant + guardrails intact), and either opens a PR to merge the winner (through PDLC's existing Critic/Judge and human-approval checkpoint — never auto-merged) or trips IXP's kill switch to route 100% back to control
- A safety net / kill switch to halt or roll back an experiment mid-flight, routed through IXP's existing promote/rollback controls

---

## How

A thin connective layer — one new DevPortal-asset-registration handoff plus a new decision-and-merge-back agent — plugged into PDLC's fixed core (clone/build/deploy, Critic/Judge, approval checkpoint, telemetry) on one side and IXP's flexible experimentation layer (Blueprint, routing, stats, dashboard) on the other. Validated via a hackathon prototype before committing to a full platform build; PDLC and IXP's own capabilities stay untouched.

---

## Impact Sizing

| Reach | Frequency | Severity |
|-------|-----------|----------|
| [TBD — needs: count of teams not onboarded to IXP, or onboarded teams currently hand-driving the watch-and-merge step] | Every experiment cycle for affected teams (ongoing, not one-time) | 7/10 — no direct outage evidence yet, but blocks rigorous decision-making and risks wrong product calls from underpowered experiments or delayed/mis-executed merges |

---

## When

**Target ship:** [TBD]
**Key milestone:** Hackathon prototype demonstrating the DevPortal-asset handoff and the decision-and-merge-back agent running one pilot team's experiment clone-to-merge through PDLC + IXP