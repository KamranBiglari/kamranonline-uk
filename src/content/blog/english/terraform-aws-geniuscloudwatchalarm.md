---
title: "YAML-Driven CloudWatch Alarms with Terraform: Meet GeniusCloudWatchAlarm"
meta_title: "Terraform Module: Manage AWS CloudWatch Alarms from YAML — GeniusCloudWatchAlarm"
description: "Stop writing repetitive aws_cloudwatch_metric_alarm blocks. This Terraform module lets you define all your CloudWatch alarms in a single YAML file — with templating, environment filtering, and log-based metrics built in."
date: 2026-02-17T11:00:00Z
image: "/images/blog/terraform-aws-geniuscloudwatchalarm/banner.png"
categories: ["AWS", "Terraform", "DevOps", "Monitoring"]
author: "Kamran Biglari"
tags: ["terraform", "aws", "cloudwatch", "alarms", "monitoring", "yaml", "sns", "devops", "observability"]
draft: false
---

## The Repetition Problem with CloudWatch Alarms

CloudWatch alarms are essential. But writing them in Terraform gets tedious fast. Every alarm is its own resource block — same structure, different values — and once you have dozens of them across multiple environments, the configuration sprawl becomes a real maintenance burden.

The [terraform-aws-geniuscloudwatchalarm](https://github.com/KamranBiglari/terraform-aws-geniuscloudwatchalarm) module takes a different approach: define all your alarms in a single YAML file and let Terraform do the heavy lifting. The result is a much more readable, maintainable monitoring configuration.

---

## How It Works

Under the hood, the module uses Terraform's `templatefile()` and `yamldecode()` functions to read your YAML, inject dynamic values, and then pass each alarm definition to the [`terraform-aws-modules/cloudwatch`](https://registry.terraform.io/modules/terraform-aws-modules/cloudwatch/aws/latest) submodules — one for metric alarms, one for log-based metric filters.

```
cloudwatch.yaml (with template vars)
         ↓ templatefile() + yamldecode()
    GeniusCloudWatchAlarm module
         ↓
  ┌──────────────────────────────┐
  │  metric-alarm (per entry)   │
  │  log-metric-filter (custom) │
  └──────────────────────────────┘
         ↓
  CloudWatch Alarms + SNS
```

Key behaviours:
- **Environment filtering** — each alarm can list which environments it applies to. If the `current_environment` variable isn't in the list, the alarm is skipped.
- **Templating** — pass a `template_data` map and reference values anywhere in your YAML with standard Terraform template syntax (`${variable}`).
- **Loop expansion** — the `loop` variable lets you multiply a single alarm definition across multiple service instances (e.g. multiple Redis cluster nodes).
- **Log-based metrics** — define a CloudWatch Logs filter pattern inline and the module creates the log metric filter before wiring up the alarm.

---

## Prerequisites

- Terraform `>= 1.0.11`
- AWS Provider `>= 4.8.0`

---

## Module Usage

```hcl
module "cloudwatch-monitor" {
  source  = "KamranBiglari/geniuscloudwatchalarm/aws"
  version = "~> 1.0"

  input               = "${path.module}/cloudwatch.yaml"
  alarm_name_prefix   = "my-app"
  current_environment = "prod"

  template_data = {
    rediscluster        = { node1 = "my-redis-001", node2 = "my-redis-002" }
    websocketmonitoring = module.log_group
  }

  alarm_actions = {
    default = {
      alarm = {
        critical = ["arn:aws:sns:eu-west-1:123456789012:pagerduty-critical"]
        warning  = ["arn:aws:sns:eu-west-1:123456789012:slack-warnings"]
      }
      ok = {
        critical = ["arn:aws:sns:eu-west-1:123456789012:pagerduty-critical"]
        warning  = ["arn:aws:sns:eu-west-1:123456789012:slack-warnings"]
      }
    }
  }
}
```

The `alarm_actions` map decouples alarm severity levels (`critical`, `warning`) from SNS ARNs, so you can wire them up once at the Terraform level and reference them by name in YAML.

---

## The YAML Configuration

This is where the real power lies. Here's a real example from the repo covering three different alarm types:

### Standard Metric Alarm — ElastiCache Connections

```yaml
Cloudwatch:
  - Name: redis-connections
    Status: 1
    Description: Exceeded 250 average ElastiCache current connections.
    Namespace: AWS/ElastiCache
    Metrics: CurrConnections
    Query:
      - label: "expression redis cluster"
        expression: MAX(METRICS())
        return_data: true
%{ for nodeK, nodeV in rediscluster ~}
      - label: "metric redis cluster ${nodeV}"
        metric:
          namespace: AWS/ElastiCache
          metric_name: CurrConnections
          period: 60
          stat: Average
          dimensions:
            CacheClusterId: ${nodeV}
%{ endfor ~}
    EvaluationPeriods: 2
    Period: 60
    Operator: GreaterThanOrEqualToThreshold
    Statistic: Average
    Threshold: 250
    TreatMissingData: breaching
    AlarmActions:
      OK:
        default:
          prod: critical
      ALARM:
        default:
          dev: critical
    Environment:
      - dev
      - prod
```

Notice the `%{ for ... }` loop — when `template_data` contains a `rediscluster` map with multiple nodes, this generates a separate metric query for each node and uses `MAX(METRICS())` as the aggregate expression. One alarm definition covers an entire Redis cluster.

### Simple Threshold Alarm — Memory Usage

```yaml
  - Name: redis-memory
    Status: 1
    Description: Average ElastiCache database memory over 30%.
    Namespace: AWS/ElastiCache
    Metrics: DatabaseMemoryUsagePercentage
    EvaluationPeriods: 2
    Period: 60
    Operator: GreaterThanOrEqualToThreshold
    Statistic: Average
    Threshold: 30
    TreatMissingData: breaching
    AlertLevel: warn
    AlarmActions:
      OK:
        default:
          dev: critical
      ALARM:
        default:
          dev: critical
    Environment:
      - dev
      - prod
```

### Log-Based Metric Alarm — Custom Application Metric

```yaml
  - Name: websocket-subscriptions
    Status: 1
    Description: Check websocket subscriptions.
    Namespace: cryptofeed/websocketmonitoring
    Metrics: SubscribeCount
    CustomMetrics:
      Patten: '{$.level = "INFO" && $.message = "Subscribed*"}'
      LogGroupName: ${websocketmonitoring.log_group_name}
    EvaluationPeriods: 2
    Period: 60
    Operator: LessThanOrEqualToThreshold
    Statistic: Sum
    Threshold: 30
    TreatMissingData: breaching
    AlertLevel: critical
    AlarmActions:
      OK:
        default:
          dev: critical
      ALARM:
        default:
          dev: critical
    Environment:
      - dev
      - prod
```

The `CustomMetrics` block tells the module to create a CloudWatch Logs metric filter using the JSON filter pattern, publish counts to the custom namespace, and then alarm on that metric — all from one YAML entry.

---

## Key YAML Fields Reference

| Field | Description |
|---|---|
| `Name` | Alarm name (prefixed with `alarm_name_prefix`) |
| `Status` | `1` = enabled, `0` = disabled |
| `Namespace` | AWS or custom CloudWatch namespace |
| `Metrics` | CloudWatch metric name |
| `CustomMetrics` | Log filter pattern + log group for log-based metrics |
| `Query` | Metric math expressions (supports `MAX`, `SUM`, etc.) |
| `EvaluationPeriods` | How many periods to evaluate before alarming |
| `Period` | Evaluation window in seconds |
| `Operator` | `GreaterThanOrEqualToThreshold`, `LessThanOrEqualToThreshold`, etc. |
| `Statistic` | `Average`, `Sum`, `Max`, `Min`, `SampleCount` |
| `Threshold` | Numeric trigger value |
| `TreatMissingData` | `breaching`, `notBreaching`, `ignore`, `missing` |
| `AlertLevel` | `warn` or `critical` — maps to SNS topics in `alarm_actions` |
| `AlarmActions` | Per-environment SNS routing for ALARM and OK states |
| `Environment` | List of environments this alarm applies to |

---

## Environment Filtering in Practice

The `Environment` field is optional. If omitted, the alarm deploys to every environment. If present, it acts as an allowlist against the `current_environment` variable passed to the module.

This means a single `cloudwatch.yaml` can serve both `dev` and `prod` deployments:

```hcl
# dev deployment
module "monitoring_dev" {
  source              = "KamranBiglari/geniuscloudwatchalarm/aws"
  version             = "~> 1.0"
  input               = "${path.module}/cloudwatch.yaml"
  alarm_name_prefix   = "myapp-dev"
  current_environment = "dev"
  # ...
}

# prod deployment
module "monitoring_prod" {
  source              = "KamranBiglari/geniuscloudwatchalarm/aws"
  version             = "~> 1.0"
  input               = "${path.module}/cloudwatch.yaml"
  alarm_name_prefix   = "myapp-prod"
  current_environment = "prod"
  # ...
}
```

Both modules read the same YAML — each only creates the alarms tagged for its environment.

---

## Why YAML Over HCL?

A few practical reasons:

- **Readability** — YAML is easier for non-Terraform engineers (SREs, developers) to read and edit without touching HCL.
- **Bulk changes** — updating a threshold or adding an alarm across environments is a one-line YAML change rather than hunting through multiple `.tf` files.
- **Templating** — Terraform's `templatefile()` gives you loops and conditionals inside YAML, so one definition can expand to many alarms dynamically.
- **Git diff clarity** — alarm configuration changes are isolated in one file, making code reviews straightforward.

---

## Source Code

The module is open source (Apache 2.0) and available on both GitHub and the Terraform Registry:

- **GitHub**: [github.com/KamranBiglari/terraform-aws-geniuscloudwatchalarm](https://github.com/KamranBiglari/terraform-aws-geniuscloudwatchalarm)
- **Terraform Registry**: `KamranBiglari/geniuscloudwatchalarm/aws`

Issues and PRs are welcome — particularly around adding more alarm types or extending the YAML schema.

---

## Wrapping Up

If your team is managing more than a handful of CloudWatch alarms, centralising them in YAML pays off quickly. The combination of environment filtering, template-driven dynamic alarms, and log-based metric support covers most real-world monitoring needs — without the boilerplate.

Give it a try on a non-critical environment first, check that the alarms appear correctly in the CloudWatch console, then roll it out to prod.
