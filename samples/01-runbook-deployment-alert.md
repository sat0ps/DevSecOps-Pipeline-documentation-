# Runbook: Service-A Deployment Alert Response

**Alert Names:** `service-a-deployment-failure` | `service-a-high-error-rate`  
**Severity:** P1 (Critical) / P2 (High)  
**Owner:** DevOps/SRE Team  
**Last Updated:** October 2025

---

## 🎯 Objective
Quickly restore Service-A availability after a failed deployment or elevated error rate in the production Kubernetes cluster.

---

## ✅ Prerequisites
- Access to GitHub Actions logs
- kubectl access to `prod-cluster-01`
- New Relic APM dashboard access for Service-A
- PagerDuty/Slack notification received

---

## 📊 Alert Thresholds
- **Warning:** Error rate >2% for 5 minutes
- **Critical:** Error rate >5% for 3 minutes
- **Critical:** Pod ready count <2 for 1 minute

---

## 🔧 Response Steps

### Step 1: Confirm the Alert
- Open New Relic alert incident page
- Note: timestamp, error rate, affected pods
- Review NRQL query:
```sql
  FROM Transaction 
  SELECT percentage(count(*), WHERE error IS true) 
  WHERE appName = 'service-a'
```

### Step 2: Check Recent Deployments
- Navigate to **GitHub Actions** → `service-a` repository → latest workflow run
- Review build and test stages
- Check if Trivy or SonarQube scans failed
- Note: If security scans blocked deployment, investigate before proceeding

### Step 3: Inspect Kubernetes Pods
```bash
# Check pod status
kubectl get pods -n production -l app=service-a

# Describe failing pod
kubectl describe pod <failing-pod-name> -n production

# View recent logs
kubectl logs <failing-pod-name> -n production --tail=100

# Check for previous crashes
kubectl logs <failing-pod-name> -n production --previous
```

**Look for:**
- CrashLoopBackOff
- ImagePullBackOff
- OOMKilled
- Startup probe failures

### Step 4: Review New Relic Logs & Distributed Tracing
**Logs:**
- Open New Relic Logs dashboard
- Filter: `kubernetes.pod_name: service-a-*` AND `logtype: error`
- Time range: Last 15 minutes

**Distributed Tracing:**
- Check trace waterfall for failed requests
- Identify downstream dependencies (API-X, Database-Y)
- Look for latency spikes or connection errors

### Step 5: Remediation Decision Tree

#### If Deployment Config Issue:
```bash
# Rollback to last stable version
kubectl rollout undo deployment/service-a -n production

# Verify rollback
kubectl rollout status deployment/service-a -n production
```

#### If Resource Exhaustion:
```bash
# Scale up replicas
kubectl scale deployment/service-a -n production --replicas=5

# OR increase resource limits (edit deployment)
kubectl edit deployment/service-a -n production
# Update memory/CPU limits, then save
```

#### If Upstream Dependency Down:
- Coordinate with API-X or Database-Y team via Slack
- Check dependency health dashboards
- Consider enabling circuit breaker if configured
- Evaluate if traffic can be routed to backup service

#### If Security Scan Failure:
- Review Trivy vulnerability report in GitHub Actions
- Check SonarQube for critical code quality issues
- If non-critical: Override after team review
- If critical: Roll back and fix in new PR

### Step 6: Validate Recovery
```bash
# Check error rate in New Relic
# Should drop below 2% within 5 minutes

# Run smoke test
curl -v https://service-a.prod.example.com/health
# Expect: HTTP 200 OK

# Verify pod stability
kubectl get pods -n production -l app=service-a
# All pods should be Running with 0 restarts
```

**Monitor for 10 minutes:**
- New Relic APM: Response time, throughput, error rate
- NRQL alert should auto-resolve
- No new errors in pod logs

### Step 7: Document & Communicate
1. **Update incident ticket** (e.g., JIRA-1234)
   - Add timeline of actions taken
   - Note current service status
   - Tag relevant team members

2. **Notify stakeholders:**
   - Post update in #incidents Slack channel
   - Update status page if customer-facing

3. **Schedule post-incident review** (within 48 hours)
   - If root cause unclear
   - If incident recurred
   - If MTTR exceeded 30 minutes

---

## 🚨 Escalation Path

**If resolution fails after 15 minutes:**
1. Page on-call lead via PagerDuty
2. Escalate to platform engineering team
3. Consider activating major incident protocol

**Contacts:**
- On-call Lead: Check PagerDuty schedule
- Platform Team: #platform-engineering Slack
- Security Team: #security Slack (for CVE-related blocks)

---

## 📚 Related Documentation
- [Service-A Architecture Diagram](#)
- [New Relic Dashboard: Service-A Production](#)
- [GitHub Actions Workflow: service-a-ci-cd.yml](#)
- [Rollback Procedures](#)

---

## 📈 Success Metrics
- **MTTD (Mean Time to Detect):** <3 minutes
- **MTTR (Mean Time to Resolve):** <30 minutes
- **False Positive Rate:** <5%

---

**Author:** Satyam Priyam | DevOps Engineer  
**Version:** 1.0  
**Review Cycle:** Quarterly
