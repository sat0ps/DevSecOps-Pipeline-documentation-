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
4. **Configuration Drift:** Staging environment had updated API-Gateway config (header requirement enabled), but production gateway was not updated before service deployment
5. **Test Gap:** Integration tests passed because test environment mirrored staging, not production topology
6. **Deployment Sequence Error:** Service deployment proceeded without verifying dependent API-Gateway configuration version
7. **Code Review Miss:** PR review focused on code quality but didn't flag the breaking API contract change

### Contributing Factors
- **Lack of Contract Testing:** No automated check for API Gateway contract compatibility
- **Manual Config Management:** API-Gateway configuration not managed via Terraform (inconsistent across environments)
- **Insufficient Pre-Deploy Validation:** No production smoke test before full rollout

### Why Detection Was Fast
✅ **Alert fired within 2 minutes** - New Relic NRQL alert policy worked as designed  
✅ **Clear error logs** - NullPointerException with stack trace visible in kubectl logs  
✅ **Distributed tracing** - Quickly identified API-Gateway as failure point

---

## 🛠️ Immediate Fix

### Actions Taken
```bash
# 1. Rollback deployment to last known good version
kubectl rollout undo deployment/service-d -n production

# 2. Verify rollback completion
kubectl rollout status deployment/service-d -n production

# 3. Validate pod health
kubectl get pods -n production -l app=service-d

# 4. Monitor error rate recovery
# New Relic APM dashboard - confirmed drop from 22% → 1.2%

# 5. Execute smoke tests
curl -X POST https://api.internal.example.com/payments/authorize \
  -H "Authorization: Bearer $TEST_TOKEN" \
  -d '{"amount": 100, "currency": "USD"}'
# Result: 200 OK - payment processed successfully
```

### Why Rollback Was Chosen
- **No hotfix available** - Would require 15-20 minutes to code, test, and deploy
- **Customer impact ongoing** - Rollback faster than forward fix
- **Low risk** - v2.3.0 was stable in production for 2 weeks

---

## 🚫 Prevention Measures

### Implemented (Completed)

| Action | Owner | Completed | Verification |
|--------|-------|-----------|--------------|
| **1. Update CI/CD pipeline to include production-like gateway stub in integration tests** | DevOps Team | ✅ Nov 20 | Integration test now uses production gateway contract mock |
| **2. Add pre-deployment validation step: Confirm API-Gateway config version matches service contract** | Platform Team | ✅ Nov 18 | Automated check added to GitHub Actions workflow |
| **3. Migrate API-Gateway config to Terraform for consistency across environments** | Platform Team | ✅ Nov 19 | Gateway config now in `terraform/api-gateway/` repo |
| **4. Create NRQL alert for "missing header" pattern in Service-D logs** | SRE Team | ✅ Nov 17 | Alert fires if >10 "missing x-correlation-id" errors in 5 min |

### In Progress

| Action | Owner | Target Date | Status |
|--------|-------|-------------|--------|
| **5. Expand Trivy and SonarQube scans to flag breaking changes in API contracts (OpenAPI diff check)** | Security Team | Dec 1 | 🔄 Tool evaluation phase - Testing Spectral and Optic |
| **6. Implement canary deployment strategy for Service-D (5% → 50% → 100%)** | Platform Team | Dec 15 | 🔄 ArgoCD Rollouts configuration in review |

### Planned

| Action | Owner | Target Date | Status |
|--------|-------|-------------|--------|
| **7. Conduct dependency mapping workshop to document all service contracts** | Engineering Team | Dec 10 | 📅 Session scheduled |
| **8. Add contract testing framework (Pact or Spring Cloud Contract)** | Backend Team | Q1 2025 | 📅 Spike story created |

---

## 📖 Lessons Learned

### What Went Well ✅
1. **Fast Detection:** Alert fired 2 minutes after deployment - MTTD exceeded target
2. **Clear Diagnosis:** Distributed tracing immediately pinpointed API-Gateway as failure point
3. **Quick Decision:** Team confidently chose rollback over forward fix
4. **Effective Communication:** Incident Commander kept stakeholders updated via Slack
5. **Blameless Culture:** Team focused on process improvements, not individual fault

### What Could Be Improved ⚠️
1. **Test Environment Parity:** Staging diverged from production topology
2. **Pre-Deploy Validation:** No automated check for configuration drift
3. **Contract Visibility:** API dependencies not clearly documented
4. **Deployment Strategy:** All-at-once rollout increased blast radius
5. **Code Review Checklist:** No prompt to verify breaking API changes

### Systemic Issues Identified 🔍
1. **Configuration Management:** Manual config changes led to drift
2. **Integration Test Coverage:** Tests didn't validate production-like scenarios
3. **Deployment Pipeline Gaps:** No pre-production smoke tests against real endpoints

---

## 📈 Metrics & Outcomes

### Incident Metrics
- **MTTD (Mean Time to Detect):** 2 minutes ✅ (Target: <3 min)
- **MTTI (Mean Time to Investigate):** 9 minutes ✅ (Target: <15 min)
- **MTTR (Mean Time to Resolve):** 27 minutes ✅ (Target: <30 min)
- **Alert Precision:** 100% (true positive)

### Post-Incident Improvements (6-Week Follow-Up)
- **Deployment Error Rate:** Reduced by 35% (from 8 failures in Oct → 5 in Dec)
- **Configuration Drift Incidents:** Zero (Terraform eliminated manual changes)
- **Contract-Related Failures:** Zero (new validation steps caught 2 potential issues pre-deployment)
- **CI/CD Pipeline Quality:** +25% test coverage for integration scenarios

### Error Budget Impact
- **Consumed:** 12% of monthly budget in this incident
- **Recovery Plan:** Defer non-critical releases for 1 week to rebuild buffer
- **Current Status:** 38% remaining (within acceptable range)

---

## 🗣️ Stakeholder Communication

### During Incident
**14:35 UTC - Initial Notification (Slack #incidents):**
> **Incident Alert - Service-D P2**  
> Payment processing API experiencing elevated error rate (22%). Investigation underway. Payments failing with "try again" message. ETA for fix: 15-20 minutes.

**14:42 UTC - Resolution Update:**
> **Update:** Rollback completed. Error rate returned to normal (1.2%). Monitoring for stability. Will send full post-mortem within 48 hours.

### Post-Incident
**Nov 16 - Email to Engineering & Product Teams:**
> Subject: Post-Incident Summary - Service-D Deployment (INC-2024-042)
> 
> Team,
>
> Yesterday we experienced a P2 incident affecting Service-D payment processing. Full timeline and root cause analysis attached. Key takeaways:
> - Detection & resolution systems worked well (MTTR: 27 min)
> - Root cause: Configuration drift between environments
> - Prevention: API-Gateway now managed via Terraform
>
> Blameless retrospective scheduled for Nov 16 @ 2pm. All engineering teams invited.
>
> Questions? Reply here or join #incident-042 Slack channel.

---

## 🎓 Training & Knowledge Sharing

### Actions Taken
1. **Brown Bag Session (Nov 22):** "Contract Testing Best Practices" - 35 attendees
2. **Runbook Updated:** Added "Check Gateway Config" step to deployment runbook
3. **Wiki Documentation:** Created "Service Dependency Map" page with API contracts
4. **Onboarding Update:** New engineers now review this incident as case study

### Materials Created
- [Service Dependency Map (Confluence)](#)
- [Contract Testing Guide (GitHub Wiki)](#)
- [Updated Deployment Runbook](./01-runbook-deployment-alert.md)

---

## 🔗 Related Documentation

- [Service-D Architecture Diagram](#)
- [GitHub PR: Contract Validation Check](#)
- [Terraform Module: API Gateway Config](#)
- [New Relic Dashboard: Service-D Production](#)
- [Blameless Retrospective Notes (Nov 16)](#)

---

## 📋 Incident Review Checklist

- [x] Timeline documented with accurate timestamps
- [x] Root cause identified and validated
- [x] Immediate fix implemented and tested
- [x] Prevention actions assigned with owners
- [x] Stakeholders notified
- [x] Blameless retrospective conducted
- [x] Runbooks updated
- [x] Metrics captured (MTTD, MTTR)
- [x] Knowledge base articles created
- [x] Follow-up actions tracked in Jira

---

## 👥 Acknowledgments

**Thank you to:**
- **Satyam Priyam** - Fast diagnosis and decisive rollback execution
- **Mike Chen** - Backend expertise and smoke test validation
- **Jane Doe** - Effective incident command and stakeholder communication
- **Platform Team** - Rapid Terraform migration to prevent recurrence

---

**Report Author:** Satyam Priyam | DevOps Engineer  
**Reviewed By:** Jane Doe (Incident Commander), Engineering Leadership  
**Published:** November 16, 2024  
**Next Review:** Quarterly SRE Review (February 2025)

---

*This incident demonstrates the effectiveness of our observability stack (New Relic + Kubernetes monitoring) and the importance of environment parity in testing. Our 27-minute MTTR reflects strong operational practices, while prevention measures address systemic gaps to improve future resilience.*
```

---

## 🎯 **Now Let's Put It All on GitHub!**

### **Quick Method (Copy/Paste Each File):**

1. **Go to your repo:** `https://github.com/satyampriyam/devops-portfolio`

2. **Create `README.md` (main page):**
   - Click "Add file" → "Create new file"
   - Name: `README.md`
   - Paste the **File 1** content above
   - Click "Commit new file"

3. **Create samples folder with all files:**
   - Click "Add file" → "Create new file"
   - Name: `samples/01-runbook-deployment-alert.md`
   - Paste **File 2** content
   - Commit
   - Repeat for files 3, 4, 5

4. **Done!** Your portfolio is live at:
```
   https://github.com/satyampriyam/devops-portfolio
```

---

## 📧 **Update Your Resume**

Add this line to your resume under relevant experience:
```
📂 Portfolio: github.com/satyampriyam/devops-portfolio

- Designed DevSecOps pipeline integrating Kubernetes with GitHub Actions
- Reduced deployment errors by 30% through New Relic monitoring and NRQL alerts
- See detailed work samples: github.com/satyampriyam/devops-portfolio
