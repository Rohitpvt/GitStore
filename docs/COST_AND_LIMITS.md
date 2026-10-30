# GitStore — Cost Analysis & Operational Limits Specification

**Document Version:** 1.1.0  
**Phase:** Phase 0.1 — Architecture Audit, Hardening & Final Validation  
**Status:** Approved Operational Cost Baseline (Hardened)  
**Budget Objective:** Target $0.00 / Month under Strict Free Quotas (No Unrealistic "Free Forever" Guarantees)

---

## 1. Zero-Cost Architectural Philosophy & Quota Boundaries

GitStore is designed to operate within the published free tiers of **GitHub** and **Firebase**. However, in rigorous systems engineering, **free quotas are not unlimited capacity.** Free tiers enforce strict daily, monthly, and bandwidth ceilings.

This document details the exact consumption math, failure behaviors when quotas are exceeded, and the mechanisms preventing unexpected billing or catastrophic failure.

---

## 2. Platform 1: GitHub Limits & Quotas (Public Repository Tier)

| Service / Resource | Published GitHub Limit | GitStore Projected Usage | Failure Behavior When Quota Exceeded |
|---|---|---|---|
| **GitHub Pages Hosting** | 1 GB total site size | ~15 MB total static assets & JSON | Deployment fails; build rejected. |
| **GitHub Pages Bandwidth** | 100 GB / month (soft limit) | ~15 GB / month @ 100k pageviews | GitHub may throttle CDN delivery or contact repository owners to require plan adjustment. |
| **Pages Custom Builds** | Soft limit of 10 builds/hr (waived via Actions) | 6 runs/hr via custom Actions workflow | Deployments queue or time out after 10 minutes. |
| **Actions Runner Minutes** | Free for public repos (fair use) | ~180 runs/day (~180 mins/day) | Subject to 20 concurrent job limits; runs queue during peak load. |
| **GitHub REST API Rate Limit** | 5,000 requests/hr (authenticated `GITHUB_TOKEN`) | ~30 requests/hr (~5 per batch run) | HTTP 403 `rate limit exceeded` until the hourly reset. |

### Critical GitHub Insights:
1. **Public Repository Actions:** GitHub provides standard Linux runner minutes for open-source public repositories. However, jobs can be delayed by 5–30 minutes during platform peak load, and job concurrency is capped at 20 jobs.
2. **Fastly CDN Caching:** GitHub Pages is backed by Fastly’s global edge network. Correct cache-control headers on static JSON files ensure repeat visitor traffic consumes near-zero egress bandwidth.

---

## 3. Platform 2: Firebase Spark Plan Limits & Hard-Stop Behavior

Firebase’s **Spark Plan** is a strict no-cost tier with no credit card requirement:

| Resource | Spark Free Quota | Projected GitStore Consumption | Hard-Stop Behavior When Exceeded |
|---|---|---|---|
| **Firestore Document Reads** | **50,000 reads / day** | ~400 – 1,200 reads / day | Returns `RESOURCE_EXHAUSTED` (HTTP 429) until midnight PST reset. |
| **Firestore Document Writes** | **20,000 writes / day** | ~50 – 150 writes / day | Returns `RESOURCE_EXHAUSTED` (HTTP 429) until midnight PST reset. |
| **Firestore Document Deletes**| **20,000 deletes / day**| ~5 – 20 deletes / day | Returns `RESOURCE_EXHAUSTED` (HTTP 429) until midnight PST reset. |
| **Stored Firestore Data** | **1 GiB total storage** | ~5 MB for 2,000 products | Further writes fail with storage quota errors. |
| **Firebase Authentication** | **50,000 MAUs / month** | ~500 – 2,000 active developers | Further user creations/logins fail until monthly reset. |
| **Firebase App Check (reCAPTCHA v3)** | **1,000,000 calls / month**| ~30,000 – 100,000 calls / month | Requests lack valid App Check tokens and fail validation. |

---

## 4. Why Storefront Visitors Do Not Exhaust Firestore Quotas

### The Static Cache Leverage
If 100,000 visitors per month queried Cloud Firestore directly for products, catalog browsing would consume millions of document reads, instantly triggering `RESOURCE_EXHAUSTED` within hours and breaking the site.

**GitStore isolates visitor traffic entirely from Firestore:**
- **Storefront Browsing:** Visitors download pre-compiled `data/catalog.json` and `data/products/*.json` served via GitHub Pages CDN. **Firestore read consumption: Exactly 0 reads.**
- **Batch Worker Operations:** The worker runs every 10 minutes (144 times/day). Querying 3 queued submissions and checking moderation requests consumes:
  $$\text{Worker Daily Reads} = 144 \text{ runs} \times (3 \text{ submissions} + 1 \text{ config} + 2 \text{ moderation}) \approx 864 \text{ reads/day}$$
- **Developer Submissions:** 50 new submissions a day generate ~150 writes and ~300 reads.
- **Total Estimated Utilization:** ~1,200 reads/day out of **50,000 available** (~2.4% capacity).

---

## 5. Failure Recovery & Contingency Triggers

### 5.1 What Happens if Spark Quota is Exceeded?
Because Spark has no credit card attached, **there are no surprise charges or automatic debt.** If a Denial of Service attack or viral submission wave exhausts the 50,000 daily read limit:
1. **Storefront Remains Online:** Visitors can continue browsing, searching, and downloading software because `data/catalog.json` is hosted on GitHub Pages CDN, completely independent of Firebase.
2. **Submissions Gracefully Pause:** Developers attempting to submit receive a clean error: *"GitStore submission processing is temporarily paused due to daily capacity limits. Please try again after 00:00 PST."*
3. **Batch Worker Pauses:** The Actions worker detects `RESOURCE_EXHAUSTED`, logs a warning, and exits cleanly without failing GitHub Actions.

### 5.2 Scaling Triggers
- **Catalog Sharding:** If `catalog.json` exceeds 1,000 products (~150 KB), the compiler splits the catalog into category shards (`data/catalog-tools.json`, `data/catalog-apps.json`) to minimize client download times.
- **Batch Frequency Adjustments:** If Actions runner queue times increase, the cron interval can be adjusted from `*/10 * * * *` to `*/20 * * * *` without code changes.

---

## 6. Strictly Prohibited Cost Triggers

To prevent inadvertent tier upgrades or surprise bills:
- ❌ **No Cloud Functions for Firebase:** Cloud Functions requires transitioning the Firebase project to the paid Blaze Plan.
- ❌ **No reCAPTCHA Enterprise on Spark:** reCAPTCHA Enterprise enforces a 10,000 assessment free cap, charging $1.00/1,000 assessments thereafter. GitStore strictly uses **reCAPTCHA v3** (1,000,000 calls/month free).
- ❌ **No Direct Binary Uploads to Firebase Storage:** Desktop binaries (`.exe`, `.dmg`, `.apk`) remain exclusively on GitHub Releases.
- ❌ **No SMS Authentication:** Phone number verification incurs per-SMS telecom costs and is strictly excluded.
