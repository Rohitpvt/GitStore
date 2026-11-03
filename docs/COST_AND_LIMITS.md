# GitStore — Cost Analysis & Operational Limits Specification

**Document Version:** 1.7.0  
**Phase:** Phase 0.1.6 — Step-Up Proof Integrity, Firestore Rules & Deployment Reconciliation  
**Status:** IN_REVIEW — Awaiting Explicit User Approval  
**Budget Objective:** Target $0.00 / Month under Strict Free Quotas (Architectural Estimates; Spark Plan without Linked Billing)

---

## 1. Zero-Cost Architectural Philosophy & Quota Boundaries

GitStore is designed to operate within the published free tiers of **GitHub** and **Firebase**. However, in rigorous systems engineering, **free quotas are not unlimited capacity.** Free tiers enforce strict daily, monthly, and bandwidth ceilings.

This document details the exact consumption math, failure behaviors when quotas are exceeded, and the mechanisms preventing unexpected billing or catastrophic failure.

---

## 2. Platform 1: GitHub Limits & Quotas (Public Repository Tier)

| Service / Resource | Published GitHub Limit | GitStore Projected Usage (Estimates) | Failure Behavior When Quota Exceeded |
|---|---|---|---|
| **GitHub Pages Hosting** | 1 GB total site size | ~15 MB total static assets & JSON | Deployment fails; build rejected. |
| **GitHub Pages Bandwidth** | 100 GB / month (soft limit) | ~15 GB / month @ 100k pageviews | GitHub may throttle static file delivery or contact repository owner. |
| **Pages Custom Builds** | Soft limit of 10 builds/hr (waived via Actions) | 6 runs/hr via custom Actions workflow | Deployments queue or time out after 10 minutes. |
| **Actions Runner Minutes** | Free for public repos (fair use) | ~180 runs/day (~180 mins/day) | Subject to 20 concurrent job limits; runs queue during peak platform load. |
| **GitHub REST API Rate Limit** | 5,000 requests/hr (authenticated `GITHUB_TOKEN`) | ~30 requests/hr (~5 per batch run) | HTTP 403 `rate limit exceeded` until the hourly reset. |

### Critical GitHub Insights:
1. **Public Repository Actions:** GitHub provides standard Linux runner minutes for open-source public repositories. However, jobs can be delayed by 5–30 minutes during platform peak load, and job concurrency is capped at 20 jobs.
2. **Static Asset Serving:** GitHub Pages serves the static storefront (`index.html`, `explore.html`, `data/catalog.json`) directly via standard HTTP GET. Browsing consumes zero external cloud database bandwidth.

---

## 3. Platform 2: Firebase Spark Plan Limits & Hard-Stop Behavior

Firebase’s **Spark Plan** is a strict no-cost tier with no credit card requirement:

| Resource | Spark Free Quota | Projected GitStore Consumption (Estimates) | Hard-Stop Behavior When Exceeded |
|---|---|---|---|
| **Firestore Document Reads** | **50,000 reads / day** | ~400 – 1,200 reads / day | Returns `RESOURCE_EXHAUSTED` (HTTP 429) until midnight PST reset. |
| **Firestore Document Writes** | **20,000 writes / day** | ~50 – 150 writes / day | Returns `RESOURCE_EXHAUSTED` (HTTP 429) until midnight PST reset. |
| **Firestore Document Deletes**| **20,000 deletes / day**| ~5 – 20 deletes / day | Returns `RESOURCE_EXHAUSTED` (HTTP 429) until midnight PST reset. |
| **Stored Firestore Data** | **1 GiB total storage** | ~5 MB for 2,000 products | Further writes fail with storage quota errors. |
| **Firebase Authentication** | **50,000 MAUs / month** | ~500 – 2,000 active developers | Further user creations/logins fail until monthly reset. |
| **Firebase App Check (reCAPTCHA Enterprise)** | **10,000 assessments / month**| ~200 – 1,000 assessments / month | Returns HTTP 429; token acquisition fails; Firestore requests rejected. |

---

## 4. Why Storefront Visitors Do Not Exhaust Firestore Quotas

1. **Pre-Compiled Static JSON Catalogs:** Storefront visits never trigger Firestore document reads. All application discovery, search indexing, and category filtering consume static files (`data/catalog.json`) served by GitHub Pages CDN.
2. **Decoupled Binary Downloads:** Download buttons link directly to verified assets on `github.com/<owner>/<repo>/releases/download/...`. GitStore serves 0 binary bytes.
3. **Lazy-Loaded App Check:** Public visitors never initialize reCAPTCHA Enterprise. Assessments are consumed solely when developers or administrators log in or submit forms.
