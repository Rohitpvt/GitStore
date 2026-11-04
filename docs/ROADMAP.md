# GitStore — Strategic Implementation Roadmap

**Document Version:** 1.10.0  
**Phase:** Phase 0.1.10 — Fan-Out Ordering, Firestore Scalability & Recovery Error Integrity  
**Status:** IN_REVIEW — Awaiting Explicit User Approval  
**Single Source of Execution Truth:** [`PHASE_PLAN.md`](./PHASE_PLAN.md)

---

## 1. Roadmap Overview & Timeline

The GitStore development lifecycle is structured into strictly sequenced phases. Each phase establishes a verified, testable baseline with defined exit criteria before downstream phases begin.

```mermaid
gantt
    title GitStore Phased Execution Roadmap
    dateFormat  YYYY-MM-DD
    section Phase 0 & 0.1
    Phase 0: Architecture & Baseline Docs   :done, p0, 2026-10-30, 1d
    Phase 0.1: Architecture Hardening & Plan:done, p0_1, 2026-10-30, 1d
    Phase 0.1.3: Deployment & Admin Security :done, p0_1_3, 2026-10-30, 1d
    Phase 0.1.4: Evidence & Deployment Recovery :done, p0_1_4, 2026-10-30, 1d
    Phase 0.1.5: Security Rules, Step-Up & Recovery :done, p0_1_5, 2026-11-03, 1d
    Phase 0.1.6: Proof Integrity & Reconciliation :done, p0_1_6, 2026-11-03, 1d
    Phase 0.1.7: Replay Protection & Field Integrity :done, p0_1_7, 2026-11-03, 1d
    Phase 0.1.8: Node 24, Staging & 2-Tier Finalization :done, p0_1_8, 2026-11-04, 1d
    Phase 0.1.9: Permissions, SHA & Fan-Out Recovery :done, p0_1_9, 2026-11-04, 1d
    Phase 0.1.10: Fan-Out Ordering & Scalability :active, p0_1_10, 2026-11-04, 1d
    section Phase 1 & 2
    Phase 1: Frontend Core Foundation (Mock):p1, 2026-11-05, 7d
    Phase 2: Firebase Foundation & Auth     :p2, after p1, 6d
    section Phase 3 & 4
    Phase 3: Dev Dashboard & Submission     :p3, after p2, 6d
    Phase 4: GitHub Integration & Batch Eng :p4, after p3, 7d
    section Phase 5 & 6
    Phase 5: Admin Moderation & Publishing  :p5, after p4, 6d
    Phase 6: Security, Integ & QA (Launch Gate):p6, after p5, 5d
    section Phase 7 & 8
    Phase 7: Controlled Beta Launch         :p7, after p6, 5d
    Phase 8: Post-Launch Improvements       :p8, after p7, 10d
```

---

## 2. Master Phase Summary

| Phase | Title | Focus & Core Deliverables | Status |
|---|---|---|---|
| **Phase 0** | Architecture and Documentation | Core product requirements, dual-platform architecture, 13 baseline docs. | `APPROVED` ✅ |
| **Phase 0.1** | Architecture Hardening & Validation | Audited admin workflow, issue ownership framework, security rules, `PHASE_PLAN.md`. | `IN_REVIEW` ⏳ |
| **Phase 0.1.3**| Deployment Integrity, Admin Bootstrap & Workflow Security | Non-circular manifest, Pages status `succeed`, Option A single workflow, admin bootstrap, event security matrix, claim whitelisting, monotonic sequencing. | `COMPLETED` ✅ |
| **Phase 0.1.4**| Evidence Alignment, Deployment Recovery & Governance Consistency | Evidence classification matrix, deterministic post-commit SHA recovery, 4-case monotonic sequence matrix, generatedAt manifest, Pages deploy requirements, ID normalization. | `COMPLETED` ✅ |
| **Phase 0.1.5**| Final Security Rules, Administrator Step-Up Authentication & Deployment Recovery Audit | Profile separation, private user data lockout, mandatory fresh step-up reauthentication (Option A), 12-state recovery machine, unshallow full-history search, digest verification, Cases A-D recovery, split-job workflow with `@v7`/`@v5` actions, 20-row matrix. | `COMPLETED` ✅ |
| **Phase 0.1.6**| Step-Up Proof Integrity, Firestore Rules & Deployment Reconciliation | Action-bound step-up proof without raw tokens, 5-min creation gate vs 30-min queue TTL, exhaustive security rules schemas (`diff().affectedKeys()`), reconciliation query fix across non-terminal states, executable Case B recovery, official Pages deployment endpoint by commit SHA, Commits API pagination loop, recursive canonical catalog digest (`sha256:v1`), transactional revision updates, and pre-deploy 4-case guard. | `COMPLETED` ✅ |
| **Phase 0.1.7**| Replay Protection, Firestore Field Integrity & Deployment State Consistency | GitHub Pages dedicated status endpoint (`status === 'succeed'`), Case B real artifact handoff to `$GITHUB_OUTPUT`, submission client create allowlist without worker fields, `/consumed_stepup_nonces` atomic single-use reservation ledger, truthful client checksum classification of `actionHash`, canonical target doc validation, mixed-provider reauth test scenario, durable revision transactions with stale abort (`capturedRevision < currentPublishedRevision`), single shared generation ordering function, complete commit pagination loop without depth limits. | `COMPLETED` ✅ |
| **Phase 0.1.8**| Node 24 Workflow Compatibility, Artifact Provenance & Transactional Deployment Integrity | Node 24 Actions runner migration (`checkout@v5.0.0`, `setup-node@v7.1.0`, `upload-pages-artifact@v5.0.0`, `deploy-pages@v5.0.1`), authoritative Case B staging eliminating corrupting workflow shell copy, `$GITHUB_OUTPUT` output bridge with delimiter escaping, 2-tier finalization architecture (3-write state anchor reading all state before writes + batched submission writes in chunks of 400) respecting 500-write limit, action-specific moderation payload schemas in Security Rules and processor logic, canonical `requestedByUid` naming, commit recovery granular error classification (`CANDIDATE_NO_MANIFEST` vs `API_RATE_LIMIT`), Pages API `status === 'succeed'` with HTTP manifest verification. | `COMPLETED` ✅ |
| **Phase 0.1.9**| Workflow Validation, Recovery SHA Preservation & Durable Finalization Recovery | Purged unsupported `metadata: read` from workflow YAML, separated `commit_needed` from `deploy_needed` ensuring Case B preserves recovered commit SHA, durable 2-tier finalization with independent submission fan-out state (`submissionFanoutStatus`, cursor resumption, `expectedSubmissionIds`, 5-attempt fallback to `manual_review`), unified canonical catalog digest input contract (`computeCatalogDigest(catalogDataRoot)`), verified action release metadata (`upload-pages-artifact@v5.0.0` uses `actions/upload-artifact@v7.0.1`), technical justification for retaining checkout v5 over v6, and actual repository file inventory. | `COMPLETED` ✅ |
| **Phase 0.1.10**| Fan-Out Ordering, Firestore Scalability & Recovery Error Integrity | Enforced conditional monotonic submission fan-out (`publishedGenerationSequence`, status eligibility guard) across 9 concurrency scenarios, eliminated unbounded array by partitioning fan-out into bounded subcollections (`fanout_batches`, max 400 IDs, parent doc $O(1)$ size << 1 MiB), implemented strict commit inspection error classifier (`classifyCommitInspectionError`, quarantine corrupt manifests, zero-error `EMPTY_SEARCH`), enforced complete nested `productData` validation in Rules and worker, and upgraded to `actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1` (`v7.0.1`) verifying runner credential persistence. | `IN_REVIEW` ⏳ (Awaiting Gate) |
| **Phase 1** | Frontend Core Foundation | Responsive Vanilla HTML5/CSS3/ES6 static storefront using mock catalogs. | `NOT_STARTED` 🛑 |
| **Phase 2** | Firebase Foundation and Auth | GitHub OAuth, Passwordless Magic Links, App Check (reCAPTCHA Enterprise), security rules. | `NOT_STARTED` 🛑 |
| **Phase 3** | Dev Dashboard & Submission System | Multi-format submission wizard, state polling, async claim queue. | `NOT_STARTED` 🛑 |
| **Phase 4** | GitHub Integration & Batch Processing | Actions batch processor, issue parser, 15m lease mutex, static compilation. | `NOT_STARTED` 🛑 |
| **Phase 5** | Admin Moderation & Public Publishing | Decoupled moderation queue with action-bound proof, developer verification, Two-Phase catalog compilation. | `NOT_STARTED` 🛑 |
| **Phase 6** | Security, Integration & QA | Security rules testing (emulator), E2E testing, pre-launch Pages policy gate. | `NOT_STARTED` 🛑 |
| **Phase 7** | Controlled Beta Launch | Staging deployment, initial developer onboarding, production release. | `NOT_STARTED` 🛑 |
| **Phase 8** | Post-Launch Improvements | Multi-language catalog index, dark/light toggle, automated release changelogs. | `NOT_STARTED` 🛑 |
