# SLO Design: Service-C Latency & Error Rate Thresholds

**Service:** Service-C (Customer Profile API)  
**Environment:** Production  
**Owner:** SRE Team  
**Last Review:** October 2025

---

## 📊 Service Context

**Service-C** is a customer-facing REST API serving user profile data with the following characteristics:

- **Traffic:** ~10,000 requests/minute peak
- **Dependencies:** Database-Y (PostgreSQL), Cache-Layer (Redis), API-Gateway
- **SLA Commitment:** 99.5% availability
- **Target p95 Latency:** <400ms
- **Critical User Journey:** Profile load on app homepage

---

## 🎯 SLO Definition

### Primary SLO
**Availability:** 99.5% of requests return successful response (2xx/3xx) within 400ms

**Error Budget:**
- Monthly allowance: 0.5% = ~216 minutes downtime
- Daily budget: ~7.2 minutes
- Budget burn rate alert: >10% consumed in 1 hour

---

## 📈 Metric Selection & Rationale

### 1. P95 Latency (Primary Performance Indicator)

**Why p95 over p99?**
- ✅ More stable for alerting (less noisy than p99)
- ✅ Captures tail performance affecting majority of users
- ✅ Balances sensitivity with actionability
- ❌ p50 misses outliers; p99 too volatile for automated alerts

**Data Source:** New Relic APM Transaction duration

### 2. Error Rate (Availability Indicator)

**Definition:** Percentage of requests returning 5xx status codes

**Why 5xx only?**
- 4xx errors are client-side (not SLO breach)
- 5xx indicates service failure
- Directly impacts availability calculation

**Data Source:** New Relic APM HTTP response codes

### 3. Pod Ready Count (Infrastructure Health)

**Purpose:** Early warning before user impact

**Threshold Logic:**
- Service requires minimum 3 pods for redundancy
- <3 pods = degraded capacity (warning)
- <2 pods = single point of failure risk (critical)

**Data Source:** New Relic Kubernetes Monitoring

---

## ⚠️ Alert Threshold Design

### Latency Alerts

| Severity | Threshold | Duration | Rationale |
|----------|-----------|----------|-----------|
| **Warning** | p95 >500ms | 10 minutes | Warns of degradation trends; buffer above SLO target (400ms); 10-min window reduces flapping from temporary spikes |
| **Critical** | p95 >800ms | 5 minutes | Breaches SLO budget significantly; risks cascading timeouts to upstream services; 5-min critical enables fast rollback decision |

**NRQL Query:**
```sql
FROM Transaction 
SELECT percentile(duration, 95) 
WHERE appName = 'service-c' 
  AND transactionType = 'Web'
```

**Alert Logic:**
- Warning: Result >0.5 for 10 consecutive minutes
- Critical: Result >0.8 for 5 consecutive minutes

---

### Error Rate Alerts

| Severity | Threshold | Duration | Rationale |
|----------|-----------|----------|-----------|
| **Warning** | >2% errors | 5 minutes | Normal baseline is <1%; 2% indicates anomaly (e.g., database connection issues); 5-min window filters transient errors |
| **Critical** | >5% errors | 3 minutes | Threatens SLO (burns 10x normal error budget); indicates systemic failure (database down, bad deployment); short duration triggers immediate rollback |

**NRQL Query:**
```sql
FROM Transaction 
SELECT percentage(count(*), WHERE httpResponseCode LIKE '5%') 
WHERE appName = 'service-c'
```

**Alert Logic:**
- Warning: Result >2 for 5 consecutive minutes
- Critical: Result >5 for 3 consecutive minutes

---

### Pod Health Alerts

| Severity | Threshold | Duration | Rationale |
|----------|-----------|----------|-----------|
| **Warning** | <3 pods ready | 2 minutes | Degraded capacity; investigate before user impact; 2-min window allows brief disruptions during deployments |
| **Critical** | <2 pods ready | 1 minute | High risk of outage; single pod failure causes downtime; 1-min critical catches rapid failures (e.g., bad config, OOMKill) |

**NRQL Query:**
```sql
FROM K8sPodSample 
SELECT uniqueCount(podName) 
WHERE deploymentName = 'service-c' 
  AND status = 'Running' 
  AND clusterName = 'prod-cluster-01'
```

**Alert Logic:**
- Warning: Result <3 for 2 consecutive minutes
- Critical: Result <2 for 1 consecutive minute

---

## 📊 Dashboard Design

### Panel 1: Latency Heatmap (24-hour view)
**Purpose:** Visualize latency distribution and identify patterns

**Metrics:**
- p50, p75, p95, p99 overlaid
- Color gradient: Green (<400ms) → Yellow (400-800ms) → Red (>800ms)

**Insight:** Spot daily traffic patterns, deployment impacts, gradual degradation

---

### Panel 2: Error Rate Timeline with Deployment Markers
**Purpose:** Correlate errors with releases

**Data Sources:**
- Error rate (line chart)
- Deployment events from GitHub Actions metadata (vertical markers)

**Insight:** Quickly identify if spike followed deployment

---

### Panel 3: Pod Health Grid
**Purpose:** Real-time infrastructure status

**Metrics:**
- Pod count (running vs desired)
- CPU usage per pod
- Memory usage per pod
- Restart count per pod

**Insight:** Detect resource exhaustion, crashloops, node failures

---

### Panel 4: Distributed Trace Waterfall (Sample Slow Requests)
**Purpose:** Drill into root cause of latency

**Shows:**
- Service-C → Database-Y query duration
- Service-C → Cache-Layer lookup duration
- Service-C → API-Gateway auth overhead

**Insight:** Identify bottleneck span (e.g., "90% of latency in database query")

---

### Panel 5: SLO Burn Rate
**Purpose:** Track error budget consumption

**Calculation:**
```
Burn Rate = (Current Error Rate) / (Error Budget Rate)
```

**Alert if:** Burn rate >10x for 1 hour (consumes 10% of monthly budget)

---

## 🧪 Threshold Validation Process

### Historical Analysis (Completed Oct 2025)

**Data Period:** Last 90 days of production traffic

| Threshold | Alert Frequency | False Positives | Outcome |
|-----------|-----------------|-----------------|---------|
| p95 >500ms (10 min) | 2-3x per week | 5% | ✅ Acceptable - early warning of real issues |
| p95 >800ms (5 min) | Only during incidents | 0% | ✅ High signal - always actionable |
| Error >2% (5 min) | 1-2x per week | 8% | ✅ Acceptable - caught DB connection pool issues |
| Error >5% (3 min) | Only during outages | 0% | ✅ High signal - immediate action needed |

### Tuning Notes
- **Initial Duration:** Critical latency alert was 10 minutes
- **Adjustment:** Reduced to 5 minutes after incident review showed 10-min delay prevented timely rollback
- **Result:** MTTR improved from 25 min → 18 min average

---

## 🔄 Review Cadence

### Quarterly SLO Review
- **Participants:** SRE, Engineering, Product teams
- **Agenda:**
  - Review SLO attainment (target: >99.5%)
  - Analyze error budget spending patterns
  - Adjust thresholds if false positive rate >10%
  - Update based on service changes (new dependencies, traffic growth)

### Ad-Hoc Review Triggers
- Major incident (P1) related to alerting
- Service architecture change
- Consistent false positives (>3 per week)

---

## 📚 Related Documentation

- [Service-C Architecture Diagram](link)
- [New Relic Dashboard: Service-C SLO](link)
- [Incident Runbook: Service-C](./01-runbook-deployment-alert.md)
- [Post-Incident Review Template](./04-post-incident-summary.md)

---

## 📈 Success Metrics

### Alerting Effectiveness (Q4 2024)
- **Alert Precision:** 94% (6% false positives)
- **MTTD:** 2.1 minutes average
- **MTTR:** 18 minutes average
- **SLO Attainment:** 99.72% (exceeded target)

### Improvements Since Implementation
- ✅ 30% reduction in alert noise
- ✅ 40% faster incident detection
- ✅ Zero customer-reported outages missed by monitoring

---

**Document Owner:** Satyam Priyam | SRE Team  
**Approved By:** Engineering Leadership  
**Next Review:** January 2026
