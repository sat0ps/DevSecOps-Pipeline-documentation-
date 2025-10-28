# Post-Incident Summary: Service-D Deployment Rollback

**Incident ID:** INC-2024-042  
**Date:** November 15, 2024  
**Duration:** 14:22 UTC – 14:51 UTC (29 minutes)  
**Severity:** P2 (Service Degraded)  
**Affected Service:** Service-D (Internal Payment Processing API)  
**Incident Commander:** Jane Doe  
**Responders:** Satyam Priyam (DevOps), Mike Chen (Backend Engineering)

---

## 📊 Impact Summary

### User Impact
- **Failed Transactions:** ~1,200 payment authorization requests failed
- **User Experience:** Generic "Please try again" error message displayed
- **Customer Complaints:** 8 support tickets filed during incident window
- **Revenue Impact:** Estimated $15K in delayed transactions (all recovered post-incident)

### Technical Impact
- **Error Rate:** Spiked from baseline 0.8% → 22% peak
- **Latency:** p95 increased from 180ms → 450ms
- **Throughput:** Dropped 30% due to retries and failures

### SLO Impact
- **Monthly Error Budget:** Consumed 12% in 30 minutes
- **Remaining Budget:** 38% for remainder of November
- **SLO Status:** Still within 99.5% target (99.68% actual)

---

## ⏱️ Incident Timeline

| Time (UTC) | Event | Actor |
|------------|-------|-------|
| **14:22** | GitHub Actions completes deployment of Service-D v2.3.1 to production (commit `abc123def`) | CI/CD Pipeline |
| **14:24** | New Relic alert fires: `service-d-high-error-rate` (critical threshold: >5% for 3 min) | Automated Alert |
| **14:26** | On-call engineer (Satyam) acknowledges alert and begins investigation | Satyam Priyam |
| **14:26** | Confirmed error rate at 22% via New Relic APM dashboard | Satyam Priyam |
| **14:28** | kubectl logs reveal NullPointerException in payment validation logic | Satyam Priyam |
| **14:30** | Distributed trace analysis shows calls to API-Gateway fail due to missing `x-correlation-id` header | Satyam Priyam |
| **14:32** | Incident escalated to backend team; Jane Doe assigned as IC | Jane Doe |
| **14:35** | Decision made to rollback deployment (no immediate fix available) | Jane Doe + Team |
| **14:35** | Rollback executed: `kubectl rollout undo deployment/service-d -n production` | Satyam Priyam |
| **14:42** | Service-D v2.3.0 pods fully deployed; error rate drops to 1.2% | System |
| **14:45** | Smoke tests executed: payment flow validated end-to-end | Mike Chen |
| **14:48** | Monitoring confirmed stable; throughput recovered to normal | Satyam Priyam |
| **14:51** | New Relic alert auto-resolves; incident marked as resolved | Automated Alert |
| **14:55** | Post-incident communication sent to stakeholders | Jane Doe |

---

## 🔍 Root Cause Analysis

### What Happened
Service-D version 2.3.1 introduced a code refactor that added a mandatory `x-correlation-id` header requirement for all outbound calls to API-Gateway. 

### Why It Happened
1. **Configuration Drift:** Staging environment had updated API-Gateway config (header requirement enabled), but production gateway was not updated before service deployment
2. **Test Gap:** Integration tests passed because test environment mirrored staging, not production topology
3. **Deployment Sequence Error:** Service
