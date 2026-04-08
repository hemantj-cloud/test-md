# Dragonfly Credit & Rate Limit Test Cases

This document outlines the test cases required to verify the Dragonfly-based credit checking and rate limiting implementation in Nitrostudio Gateway.

## Overview
The Dragonfly implementation provides an atomic, high-performance validation layer for:
- Organization-wide dollar credits and overage.
- Token-based free credits.
- Member-specific personal credit limits.
- NitroChat API key validation (status, expiration, and key-specific limits).
- Per-minute rate limiting (RPM).

---

## 1. Organization Dollar Credits

| Case ID | Scenario | Pre-conditions | Action | Expected Result |
|:---:|:---|:---|:---|:---|
| **ORG-01** | Sufficient Credits | `creditsTotal: 1000`, `creditsUsed: 0` | Request cost: 100 | **Allowed.** `creditsUsed` becomes 100. |
| **ORG-02** | Credits Exhausted (No Overage) | `creditsTotal: 1000`, `creditsUsed: 1000`, `overageEnabled: 0` | Request cost: 1 | **Refused (NCCreditOrgExhausted).** |
| **ORG-03** | Overage Allowed | `creditsTotal: 1000`, `creditsUsed: 1000`, `overageEnabled: 1`, `overageLimit: 500`, `overageUsed: 0` | Request cost: 100 | **Allowed (Overage).** `overageUsed` becomes 100. |
| **ORG-04** | Overage Limit Reached | `creditsTotal: 1000`, `creditsUsed: 1000`, `overageEnabled: 1`, `overageLimit: 500`, `overageUsed: 500` | Request cost: 1 | **Refused.** |

---

## 2. Member Personal Limits

| Case ID | Scenario | Pre-conditions | Action | Expected Result |
|:---:|:---|:---|:---|:---|
| **MEM-01** | Within Personal Limit | `creditLimit: 500`, `creditsUsed: 400` | Request cost: 50 | **Allowed.** Member `creditsUsed` becomes 450. |
| **MEM-02** | Exceed Personal Limit | `creditLimit: 500`, `creditsUsed: 460` | Request cost: 50 | **Refused (NCCreditKeyExhausted).** |
| **MEM-03** | No Personal Limit | `creditLimit: -1` | Request cost: 1000 | **Allowed** (assuming Org has credits). |

---

## 3. NitroChat API Keys

| Case ID | Scenario | Pre-conditions | Action | Expected Result |
|:---:|:---|:---|:---|:---|
| **NC-01** | Valid Active Key | `status: "active"`, `expiresAt: 0` (no expiry) | Check Credits | **Allowed.** |
| **NC-02** | Revoked Key | `status: "revoked"` | Check Credits | **Refused (NCCreditInvalid).** |
| **NC-03** | Expired Key | `expiresAt: <Current Time - 1h>` | Check Credits | **Refused (NCCreditExpired).** |
| **NC-04** | Key-Specific Limit | `creditsLimit: 100`, `creditsUsed: 95` | Request cost: 10 | **Refused (NCCreditKeyExhausted).** |
| **NC-05** | Parent Org Exhausted | Key is valid but `ncOrg` has 0 credits | Request cost: 1 | **Refused (NCCreditOrgExhausted).** |

---

## 4. Rate Limiting (RPM)

| Case ID | Scenario | Pre-conditions | Action | Expected Result |
|:---:|:---|:---|:---|:---|
| **RL-01** | Under Limit | `RPM Limit: 60` | 59 requests in 1 minute | **All Allowed.** |
| **RL-02** | Burst Limit | `RPM Limit: 60` | Request #61 in same minute | **Refused (Rate Limited).** |
| **RL-03** | Window Reset | `RPM Limit: 60` | Wait for next minute | **Allowed Again.** |

---

## 5. Cache Lifecycle & Fallback

| Case ID | Scenario | Pre-conditions | Action | Expected Result |
|:---:|:---|:---|:---|:---|
| **LC-01** | Cache Miss (Org) | Org key missing in Dragonfly | API Request | **Fallback to Mongo.** Triggers async background seed to Dragonfly. |
| **LC-02** | Cache Miss (Member) | Member key missing but Org exists | API Request | **Fallback to Mongo.** Triggers async background seed for member. |
| **LC-03** | TTL Expiration | Keys set with 7-day TTL | Wait for 7 days | Keys are purged; next request triggers **LC-01**. |

---

## Verification Strategy

### 1. Manual Testing via Python Script
Use the `/tmp/test_nitrochat_df.py` to simulate Redis commands:
- Populate `gway:nc:org:{orgID}` and `gway:nc:key:{apiKey}` hashes.
- Use `EVAL` to run the `nitroChatCreditScript` Lua logic.
- Verify return codes match `NitroChatCreditCheckResult` constants.

### 2. Integration Testing
- Restart Gateway with Dragonfly enabled.
- Send requests using valid and invalid API keys.
- Monitor Redis via `MONITOR` or `HGETALL` to verify atomic increments.
