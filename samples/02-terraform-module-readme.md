# Terraform Module: New Relic Alert Policy

## 📋 Overview
This Terraform module provisions a New Relic NRQL alert policy with multiple conditions for monitoring application performance, error rates, and Kubernetes pod health.

### Features
- ✅ Creates reusable alert policy with custom conditions
- ✅ Supports flexible NRQL queries and thresholds
- ✅ Integrates with notification channels (Slack, PagerDuty, Email)
- ✅ Configurable incident aggregation preferences
- ✅ Production-ready with validation and outputs

---

## 🔧 Prerequisites

**Required:**
- Terraform >= 1.3.0
- New Relic account with Admin API key
- Provider: `newrelic/newrelic` >= 3.0

**Environment Variables:**
```bash
export NEW_RELIC_API_KEY="your-api-key"
export NEW_RELIC_ACCOUNT_ID="your-account-id"
export NEW_RELIC_REGION="US"  # or "EU"
```

---

## 🚀 Usage

### Basic Example
```hcl
module "service_b_alerts" {
  source = "./modules/newrelic-alert-policy"

  policy_name          = "service-b-production-alerts"
  incident_preference  = "PER_CONDITION"
  notification_channel = "slack-devops-alerts"

  conditions = [
    {
      name        = "High Error Rate"
      nrql_query  = "FROM Transaction SELECT percentage(count(*), WHERE error IS true) WHERE appName = 'service-b'"
      critical    = 5
      warning     = 3
      duration    = 5
    },
    {
      name        = "P95 Latency Breach"
      nrql_query  = "FROM Transaction SELECT percentile(duration, 95) WHERE appName = 'service-b'"
      critical    = 0.8
      warning     = 0.5
      duration    = 10
    }
  ]
}
```

### Advanced Example (Multiple Services)
```hcl
module "api_gateway_alerts" {
  source = "./modules/newrelic-alert-policy"

  policy_name          = "api-gateway-prod"
  incident_preference  = "PER_POLICY"
  notification_channel = "pagerduty-critical"

  conditions = [
    {
      name        = "Critical Error Rate"
      nrql_query  = "FROM Transaction SELECT percentage(count(*), WHERE httpResponseCode LIKE '5%') WHERE appName = 'api-gateway'"
      critical    = 10
      warning     = 5
      duration    = 3
    },
    {
      name        = "Pod Ready Count Low"
      nrql_query  = "FROM K8sPodSample SELECT uniqueCount(podName) WHERE deploymentName = 'api-gateway' AND status = 'Running'"
      critical    = 2
      warning     = 3
      duration    = 2
    },
    {
      name        = "Memory Usage High"
      nrql_query  = "FROM K8sContainerSample SELECT average(memoryUsedBytes/memoryLimitBytes)*100 WHERE deploymentName = 'api-gateway'"
      critical    = 90
      warning     = 80
      duration    = 10
    }
  ]
}
```

---

## 📊 Variables

| Variable               | Type          | Description                                           | Default         | Required |
|------------------------|---------------|-------------------------------------------------------|-----------------|----------|
| `policy_name`          | `string`      | Name of the New Relic alert policy                    | -               | Yes      |
| `incident_preference`  | `string`      | How to aggregate incidents: `PER_POLICY`, `PER_CONDITION`, `PER_CONDITION_AND_TARGET` | `PER_CONDITION` | No       |
| `notification_channel` | `string`      | Notification channel ID or name (Slack, PagerDuty, Email) | -           | Yes      |
| `conditions`           | `list(object)`| List of NRQL alert conditions (see structure below)   | `[]`            | Yes      |

### Condition Object Structure
```hcl
{
  name        = string  # Condition display name
  nrql_query  = string  # NRQL query string
  critical    = number  # Critical threshold value
  warning     = number  # Warning threshold value (optional)
  duration    = number  # Duration in minutes before alert fires
}
```

---

## 📤 Outputs

| Output            | Description                                    | Example                          |
|-------------------|------------------------------------------------|----------------------------------|
| `policy_id`       | New Relic alert policy ID                      | `"1234567"`                      |
| `condition_ids`   | Map of condition names to their IDs            | `{"High Error Rate": "89101112"}` |
| `policy_url`      | Direct link to policy in New Relic UI          | `"https://one.newrelic.com/..."` |

### Example Output
```bash
Outputs:

policy_id = "1234567"
condition_ids = {
  "Critical Error Rate" = "89101112"
  "P95 Latency Breach"  = "13141516"
  "Pod Ready Count Low" = "17181920"
}
policy_url = "https://one.newrelic.com/alerts/policy/1234567"
```

---

## 🏃 Deployment Steps

### 1. Initialize Terraform
```bash
terraform init
```

### 2. Validate Configuration
```bash
terraform validate
```

### 3. Plan Changes
```bash
terraform plan -var-file="prod.tfvars" -out=tfplan
```

### 4. Apply Configuration
```bash
terraform apply tfplan
```

### 5. Verify in New Relic UI
- Navigate to: **Alerts & AI → Alert Policies**
- Find your policy by name
- Verify conditions and notification channels

---

## ⚙️ Configuration Notes

### NRQL Query Best Practices
1. **Always filter by `appName` or `deploymentName`** to avoid cross-service noise
2. **Use appropriate time windows:**
   - Error rates: 3-5 minutes
   - Latency: 5-10 minutes
   - Resource usage: 10-15 minutes
3. **Test queries in New Relic UI** before adding to Terraform

### Threshold Tuning
- Start with **conservative thresholds** (higher values)
- Monitor alert frequency for 1-2 weeks
- Adjust based on false positive rate (<5% target)
- Document rationale in commit messages

### Notification Channels
- **Critical alerts:** PagerDuty (immediate page)
- **Warning alerts:** Slack (team notification)
- **Informational:** Email (async review)

---

## 🧪 Testing

### Validate Module Locally
```bash
# Dry-run validation
terraform plan -var-file="test.tfvars"

# Check NRQL syntax
# Copy query → New Relic UI → Query Builder → Validate
```

### Integration Test
1. Apply to non-prod environment first
2. Trigger test alert (e.g., scale pods to 0)
3. Verify notification received
4. Clean up test resources

---

## 🔄 Version History

| Version | Date       | Changes                                    |
|---------|------------|--------------------------------------------|
| 1.0.0   | Oct 2025   | Initial release with basic alert support   |
| 1.1.0   | Oct 2025   | Added Kubernetes monitoring conditions     |

---

## 📚 Additional Resources

- [New Relic NRQL Reference](https://docs.newrelic.com/docs/query-your-data/nrql-new-relic-query-language/)
- [Terraform New Relic Provider](https://registry.terraform.io/providers/newrelic/newrelic/latest/docs)
- [Alert Best Practices](https://docs.newrelic.com/docs/alerts-applied-intelligence/new-relic-alerts/learn-alerts/alerts-best-practices/)

---

**Module Author:** Satyam Priyam  
**Maintained By:** DevOps Team  
**License:** Internal Use Only
