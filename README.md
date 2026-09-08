# Task 10: AWS Observability with CloudWatch

**Name:** Shubham Singh  
**Task:** AWS CloudWatch Observability for Twenty CRM  
**Date:** 08-09-2026  
**Region:** US East (N. Virginia) — us-east-1  
**Organization:** PearlThoughts DevOps Internship

---

## Table of Contents

1. [Introduction](#introduction)
2. [Architecture Overview](#architecture-overview)
3. [AWS Console Setup](#aws-console-setup)
4. [EC2 Instance Setup](#ec2-instance-setup)
5. [Twenty CRM Deployment](#twenty-crm-deployment)
6. [Default EC2 Metrics in CloudWatch](#default-ec2-metrics-in-cloudwatch)
7. [CloudWatch Agent Setup](#cloudwatch-agent-setup)
8. [Custom Metrics Verification](#custom-metrics-verification)
9. [CloudWatch Alarms](#cloudwatch-alarms)
10. [CloudWatch Dashboard](#cloudwatch-dashboard)
11. [Load Testing & Results](#load-testing--results)
12. [How CloudWatch Helps](#how-cloudwatch-helps)
13. [Issues Faced & Solutions](#issues-faced--solutions)
14. [Conclusion](#conclusion)

---

## Introduction

Amazon CloudWatch is AWS's native monitoring and observability service. It collects and tracks metrics, monitors log files, sets alarms, and automatically reacts to changes in AWS resources.

In this task, I implemented AWS Observability for the Twenty CRM application running on an EC2 instance using Amazon CloudWatch. The implementation includes:

- Deploying Twenty CRM on EC2 from the PearlThoughts repository
- Installing and configuring the CloudWatch Agent
- Collecting system-level metrics (CPU, Memory, Disk)
- Creating CloudWatch Alarms with SNS email notifications
- Building a CloudWatch Dashboard for real-time monitoring
- Load testing and verifying alarm triggers

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                    EC2 Instance (t3.small)               │
│                                                         │
│  ┌─────────────────┐    ┌──────────────────────────┐   │
│  │  Twenty CRM App  │    │   CloudWatch Agent        │   │
│  │  (Docker)        │    │                          │   │
│  │  Port: 2020      │    │  Collects every 60s:     │   │
│  └─────────────────┘    │  - CPU Usage             │   │
│                          │  - Memory Usage          │   │
│                          │  - Disk Usage            │   │
│                          └──────────┬───────────────┘   │
└─────────────────────────────────────┼───────────────────┘
                                      │ sends data
                                      ▼
                          ┌───────────────────────┐
                          │   AWS CloudWatch       │
                          │                       │
                          │  ┌─────────────────┐  │
                          │  │    Metrics       │  │
                          │  │  (CWAgent NS)    │  │
                          │  └────────┬────────┘  │
                          │           │            │
                          │  ┌────────▼────────┐  │
                          │  │    Alarms        │  │
                          │  │  CPU > 80%  🔴  │  │
                          │  │  Disk > 80% 🔴  │  │
                          │  └────────┬────────┘  │
                          │           │            │
                          │  ┌────────▼────────┐  │
                          │  │   Dashboard      │  │
                          │  │ (shubham-singh-  │  │
                          │  │    task-10)      │  │
                          │  └─────────────────┘  │
                          └───────────┬───────────┘
                                      │
                                      ▼
                          ┌───────────────────────┐
                          │   SNS → Email Alert   │
                          │  (portfolio-alerts)   │
                          └───────────────────────┘
```

---

## AWS Console Setup

After logging into the AWS Console, the services used for this task were EC2, CloudWatch, Simple Notification Service (SNS), and IAM — all visible in the recently visited panel.

![AWS Console Home](images/aws-console-home.png)

*AWS Console home showing recently visited services: EC2, CloudWatch, SNS, and IAM used during Task 10.*

---

## EC2 Instance Setup

### Instance Configuration

| Setting | Value |
|---|---|
| Instance Name | shubham-sing... (shubham-singh-task10) |
| AMI | Ubuntu LTS |
| Instance Type | t3.small |
| Region | us-east-1 (N. Virginia) |
| IAM Role | CloudWatchAgentEC2Role |
| Storage | 20 GB |

### Security Group Inbound Rules

| Port | Protocol | Purpose |
|---|---|---|
| 22 | TCP | SSH Access |
| 2020 | TCP | Twenty CRM Application |
| 80 | TCP | HTTP |

> **Note:** Only necessary ports are opened. Unused ports are kept closed as a security best practice to reduce the attack surface.

### IAM Role

> **Important:** As instructed by the mentor, the `CloudWatchAgentEC2Role` was already pre-created by the organization. This existing role was selected during EC2 launch — no new IAM role was created.

This role contains the `CloudWatchAgentServerPolicy` which allows:
- Writing metrics to CloudWatch
- Writing logs to CloudWatch Logs
- Reading SSM parameters for configuration

### EC2 Instance Termination

After completing the task, the EC2 instance was terminated as per the internship policy. All three team instances (tannu-task10, shubham-sing..., vasundara-tas...) are shown in terminated state.

![EC2 Instances Terminated](images/ec2-terminated.png)

*EC2 Instances dashboard showing all 3 task-10 instances (tannu-task10, shubham-sing..., vasundara-tas...) in terminated state after task completion — region us-east-1, instance type t3.small.*

---

## Twenty CRM Deployment

This project uses the PearlThoughts DevOps CRM repository — a Twenty CRM application scaffold that uses the twenty-sdk and runs a local Twenty server via Docker.

### Prerequisites Installation

```bash
# Step 1 — Install NVM (Node Version Manager)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash

# Step 2 — Load NVM into current terminal session
# (This is needed because NVM is a shell function, not a regular program.
#  Every new SSH session forgets it, so we reload it manually)
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"

# Step 3 — Install Node.js v24 (as required by .nvmrc in the project)
nvm install 24
node --version   # should show v24.x.x

# Step 4 — Install Yarn 4 (package manager used by this project)
corepack enable
corepack prepare yarn@4.13.0 --activate
yarn --version   # should show 4.13.0

# Step 5 — Install Docker
sudo apt update
sudo apt install docker.io -y
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker ubuntu
newgrp docker
docker --version
```

### Clone Repository & Deploy

```bash
# Clone the PearlThoughts repo
git clone https://github.com/PearlThoughts-Intern-DevOps/devops-crm-project.git
cd devops-crm-project

# Install all project dependencies
yarn install

# Start the Twenty CRM server via Docker
yarn twenty docker:start

# Verify it is running
yarn twenty docker:status
```

### Twenty CRM Running Successfully

The Twenty CRM application was successfully deployed and accessible, showing 599 companies seeded in the workspace.

![Twenty CRM Application](images/twenty-crm.png)

*Twenty CRM application running successfully — Companies view showing 599 seeded companies (Google, Microsoft, Meta, SLB, Cisco, Uber, Salesforce, etc.) accessible at `http://<EC2-IP>:2020`.*

### Deployment Status Output

```
Status:  running
URL:     http://localhost:2020
Version: v2.38.1
```

### Accessing Twenty CRM

| Field | Value |
|---|---|
| URL | http://\<EC2-PUBLIC-IP\>:2020 |
| Default Email | tim@apple.dev |
| Default Password | tim@apple.dev |

### What `yarn twenty docker:start` Actually Does

```
yarn twenty docker:start
        ↓
Pulls Docker image: twentycrm/twenty-app-dev:latest
        ↓
Starts containers:
  - Twenty CRM server (NestJS backend)
  - PostgreSQL database
  - Redis cache
        ↓
Runs database migrations
        ↓
Seeds initial workspace data
        ↓
App available on port 2020
```

---

## Default EC2 Metrics in CloudWatch

Before installing the CloudWatch Agent, AWS provides these default metrics automatically:

| Metric | Description | Available by Default |
|---|---|---|
| CPUUtilization | CPU usage at hypervisor level | ✅ Yes |
| NetworkIn | Bytes received | ✅ Yes |
| NetworkOut | Bytes sent | ✅ Yes |
| DiskReadOps | Disk read operations | ✅ Yes |
| DiskWriteOps | Disk write operations | ✅ Yes |
| StatusCheckFailed | Instance health check | ✅ Yes |
| Memory Usage % | RAM utilization | ❌ NOT available |
| Disk Used % | Disk utilization % | ❌ NOT available |

> **Key Insight:** Memory and Disk utilization percentage are NOT available by default because AWS only has visibility from outside the VM (hypervisor level). The CloudWatch Agent runs inside the OS and sends these internal metrics to CloudWatch.

---

## CloudWatch Agent Setup

### Step 1 — Download and Install Agent

```bash
# Download the CloudWatch Agent package
wget https://s3.amazonaws.com/amazoncloudwatch-agent/ubuntu/amd64/latest/amazon-cloudwatch-agent.deb

# Install it
sudo dpkg -i amazon-cloudwatch-agent.deb
```

### Step 2 — Create Configuration File

Instead of using the interactive wizard (which caused issues — see Issues section), the config file was created manually:

```bash
sudo nano /opt/aws/amazon-cloudwatch-agent/bin/config.json
```

Configuration used:

```json
{
  "agent": {
    "metrics_collection_interval": 60,
    "run_as_user": "cwagent"
  },
  "metrics": {
    "append_dimensions": {
      "InstanceId": "${aws:InstanceId}"
    },
    "metrics_collected": {
      "cpu": {
        "measurement": [
          "cpu_usage_user",
          "cpu_usage_idle",
          "cpu_usage_system"
        ],
        "metrics_collection_interval": 60,
        "totalcpu": true
      },
      "mem": {
        "measurement": [
          "mem_used_percent"
        ],
        "metrics_collection_interval": 60
      },
      "disk": {
        "measurement": [
          "used_percent"
        ],
        "metrics_collection_interval": 60,
        "resources": ["/"]
      }
    }
  }
}
```

What each section means:

| Config Section | Metric Collected | Interval |
|---|---|---|
| cpu | cpu_usage_user, cpu_usage_idle, cpu_usage_system | 60 seconds |
| mem | mem_used_percent | 60 seconds |
| disk | disk used_percent (root partition only) | 60 seconds |
| append_dimensions | Tags each metric with InstanceId | — |

### Step 3 — Start the CloudWatch Agent

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config \
  -m ec2 \
  -c file:/opt/aws/amazon-cloudwatch-agent/bin/config.json \
  -s
```

What each flag means:

| Flag | Meaning |
|---|---|
| -a fetch-config | Load the config file |
| -m ec2 | Running on EC2 (not on-premises) |
| -c file:... | Path to our config.json |
| -s | Start the agent after loading config |

### Step 4 — Verify Agent is Running

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a status
```

Expected output:

```json
{
  "status": "running",
  "starttime": "2026-09-08T05:00:00Z",
  "version": "1.300072.0"
}
```

---

## Custom Metrics Verification

After starting the agent and waiting 3-5 minutes:

`CloudWatch → Metrics → All Metrics → CWAgent`

| Dimension Group | Metrics Available |
|---|---|
| InstanceId, cpu | cpu_usage_user, cpu_usage_idle, cpu_usage_system |
| InstanceId | mem_used_percent |
| InstanceId, device, fstype, path | disk used_percent |

---

## CloudWatch Alarms

### Alarm 1 — High CPU Utilization

| Setting | Value |
|---|---|
| Alarm Name | shubham-singh-task-10 |
| Namespace | CWAgent |
| Metric | cpu_usage_user |
| Statistic | Average |
| Period | 1 minute |
| Threshold Type | Static |
| Condition | Greater than 80% |
| Datapoints to Alarm | 1 out of 1 |
| SNS Topic | portfolio-alerts |

**Result:** ✅ Alarm triggered — CPU reached **83.45%** → alarm went **IN ALARM** state 🔴

The graph below shows the `cpu_usage_user` metric spiking sharply to 83.45% at approximately 07:45 UTC, crossing the 80% threshold (red dashed line) and triggering the alarm.

![CPU Alarm Triggered](images/cpu-alarm-triggered.png)

*CloudWatch CPU alarm `shubham-singh-task-10` in **IN ALARM** state — `cpu_usage_user` spiked to 83.45%, exceeding the static threshold of 80% (1 datapoint within 1 minute). The alarm bar at the bottom shows the red IN ALARM band at the far right.*

---

### Alarm 2 — High Disk Utilization

| Setting | Value |
|---|---|
| Alarm Name | shubbham-singh-task-10-disk-usage |
| Namespace | CWAgent |
| Metric | disk_used_percent |
| Statistic | Average |
| Period | 5 minutes |
| Threshold Type | Static |
| Condition | Greater than 80% |
| Datapoints to Alarm | 1 out of 1 |
| SNS Topic | portfolio-alerts |

**Result:** ✅ Alarm stayed **OK** — Disk usage stayed well below 80% threshold.

The disk usage graph shows a steady reading around 42–61% — comfortably below the 80% alarm threshold (red line).

![Disk Alarm OK - View 1](images/disk-alarm-ok.png)

*CloudWatch disk alarm `shubbham-singh-task-10-disk-usage` — showing `disk_used_percent` metric in **OK** state. Disk usage ranged between ~42% and ~61%, staying well below the 80% threshold (red line). Total alarms: 11 in OK state, 0 in alarm.*

![Disk Alarm OK - View 2](images/disk-alarm-ok-2.png)

*Second view of the disk usage alarm in **OK** state — same alarm `shubbham-singh-task-10-disk-usage` with 10 alarms visible in the list panel, confirming stable disk utilization throughout the load test period.*

---

### Alarm States Explained

| State | Indicator | Meaning |
|---|---|---|
| OK | 🟢 Green | Metric is within acceptable threshold |
| IN ALARM | 🔴 Red | Metric exceeded threshold — alert sent! |
| Insufficient Data | ⚪ Grey | Not enough data points collected yet |

---

## CloudWatch Dashboard

**Dashboard Name:** `shubham-singh-task-10`

### Widgets Created

| Widget # | Metric | Namespace | Purpose |
|---|---|---|---|
| 1 | cpu_usage_idle, cpu_usage_system, cpu_usage_user | CWAgent | Monitor all CPU components |
| 2 | disk_used_percent | CWAgent | Monitor disk storage usage |
| 3 | mem_used_percent | CWAgent | Monitor RAM usage |
| 4 | cpu_usage_user + disk_used_percent + mem_used_percent | CWAgent | Combined overview |

The dashboard below shows all 4 widgets during the load test, with visible spikes across CPU, Disk, and Memory metrics at approximately 07:00–07:30 UTC.

![CloudWatch Dashboard](images/cloudwatch-dashboard.png)

*CloudWatch Dashboard `shubham-singh-task-10` showing all 4 metric widgets during load testing:*
- *Widget 1 (CPU): `cpu_usage_user` and `cpu_usage_idle` — spike visible, reaching ~99%*
- *Widget 2 (Disk): `disk_used_percent` — spike to ~42.37%*
- *Widget 3 (Memory): `mem_used_percent` — spike to ~66.04%*
- *Widget 4 (Combined): All three metrics overlaid — peak at ~78.37%*

### Dashboard Observations During Load Test

| Widget | Normal Value | Peak Value | Change |
|---|---|---|---|
| CPU (user) | ~0.17% | ~99.35% | +99% spike |
| Disk | ~42.02% | 42.37% | +0.35% |
| Memory | ~62.32% | 66.04% | +3.72% |
| Combined | ~0.33% | 78.37% | Multi-metric spike |

---

## Load Testing & Results

### Tools Used

```bash
# Install stress tool
sudo apt install stress -y

# Generate CPU load (2 cores, 5 minutes)
stress --cpu 2 --timeout 300 &

# Generate Memory load (500MB, 5 minutes)  
stress --vm 1 --vm-bytes 500M --timeout 300 &

# Generate HTTP traffic to Twenty CRM
for i in {1..50}; do
  curl -s http://localhost:2020 > /dev/null
  echo "Request $i done"
done
```

### Results Table

| Metric | Before Stress | During Stress | Threshold | Alarm State |
|---|---|---|---|---|
| cpu_usage_user | ~0.17% | 83.45% | 80% | 🔴 IN ALARM |
| mem_used_percent | ~62.32% | 66.04% | — | 🟢 OK |
| disk_used_percent | ~42.02% | 42.37% | 80% | 🟢 OK |

### Key Observations

- **CPU Alarm triggered** — stress pushed CPU to 83.45%, exceeding the 80% threshold ✅
- **Memory stayed OK** — 500MB stress was not enough to push t3.small past the threshold ✅
- **Disk stayed OK** — no significant disk write during the short test ✅
- **Dashboard updated live** — all 4 widgets showed spikes exactly when stress ran ✅

---

### Anomaly Detection (Explored but Not Used for Alarms)

During the task, CloudWatch Anomaly Detection was also explored as an alternative alarm type.

**What Anomaly Detection Does:**

Instead of a fixed threshold, Anomaly Detection uses machine learning to learn your metric's normal pattern over time and creates a dynamic band:

```
CPU %
100│
 80│·····upper band·····  ← moves based on learned pattern
 50│    (normal zone)
 20│·····lower band·····
   └────────────────────
   
Alarm triggers when metric goes OUTSIDE the band
```

**Why It Was Not Used for This Task:**

| Reason | Explanation |
|---|---|
| New instance | Anomaly Detection needs 2+ weeks of historical data to learn patterns |
| No baseline | A fresh EC2 instance has no usage history to build a model from |
| Unreliable results | With insufficient data, the band is inaccurate and causes false alarms |

> **Conclusion:** Static threshold alarms were used instead, as they work reliably from day one without requiring historical data. Anomaly Detection is better suited for production systems that have been running for weeks or months.

---

## How CloudWatch Helps

### 1. Monitoring
- Real-time visibility into EC2 instance health every 60 seconds
- Custom metrics via CloudWatch Agent (Memory %, Disk % — not in default metrics)
- Dashboard provides a single-pane view of all critical metrics
- Historical data — metrics stored and queryable for analysis
- Namespace organization — CWAgent metrics separate from default EC2 metrics

### 2. Alerting
- CloudWatch Alarms trigger automatically when thresholds are crossed
- SNS integration sends email notifications instantly
- State transitions — OK → ALARM → OK tracked with timestamps
- Multiple conditions — can combine alarms using composite alarms
- Prevents downtime — catch resource exhaustion before it crashes the app

### 3. Troubleshooting
- Metrics history — go back in time to see what happened during an incident
- Correlate metrics — compare CPU, Memory, Disk spikes at the same timestamp
- Root cause analysis — identify which metric spiked first
- Alarm history — see exactly when alarms triggered and for how long
- Cross-service visibility — correlate EC2 metrics with application behavior

---

## Issues Faced & Solutions

### Issue 1 — CloudWatch Agent Config Wizard Infinite Loop

**Problem:** The config wizard kept asking for log file paths in an infinite loop  
**Root Cause:** Pressing Enter without entering a path caused it to keep asking  
**Solution:** Used `Ctrl+C` to exit the wizard and manually created the `config.json` file instead — this approach is more reliable and gives full control over the configuration

### Issue 2 — CollectD Not Installed

**Problem:** Agent failed to start because CollectD was accidentally selected in the wizard  
**Error:** `CollectD metrics_aggregation_interval configuration error`  
**Solution:** Removed the `collectd` section from `config.json` and restarted the agent:

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config -m ec2 \
  -c file:/opt/aws/amazon-cloudwatch-agent/bin/config.json -s
```

### Issue 3 — EC2 Instance Auto-Terminated (Previous Session)

**Problem:** EC2 instance was automatically terminated after the 2-hour time limit before the task was complete  
**Impact:** Lost all progress — had to re-setup the entire environment on a new instance  
**Solution:** Communicated with the mentor, explained the situation, and was given access to relaunch a new instance

### Issue 4 — nvm Not Found After SSH Reconnect

**Problem:** After reconnecting via SSH, the `nvm` command was not found  
**Root Cause:** nvm is a shell function, not a regular installed program. It is loaded per terminal session and is lost when the SSH session closes  
**Solution:** Manually reload nvm each time a new session starts:

```bash
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
```

Explanation of the command:
- `export NVM_DIR="$HOME/.nvm"` → tells the shell where nvm is installed
- `[ -s "$NVM_DIR/nvm.sh" ]` → checks if the nvm script file exists and is not empty
- `&& \. "$NVM_DIR/nvm.sh"` → if it exists, loads (sources) it into the current session

---

## Conclusion

This task successfully demonstrated AWS CloudWatch observability for the Twenty CRM application deployed on EC2 within the PearlThoughts internship environment.

### Final Status Summary

| Task | Status | Details |
|---|---|---|
| EC2 Instance launched | ✅ Done | t3.small, Ubuntu LTS, us-east-1 |
| IAM Role attached | ✅ Done | CloudWatchAgentEC2Role (pre-created) |
| Twenty CRM deployed | ✅ Done | Via `yarn twenty docker:start` |
| App accessible | ✅ Done | http://EC2-IP:2020, 599 companies seeded |
| Default metrics explored | ✅ Done | EC2 namespace in CloudWatch |
| CloudWatch Agent installed | ✅ Done | v1.300072.0 |
| CPU metric collected | ✅ Done | cpu_usage_user |
| Memory metric collected | ✅ Done | mem_used_percent |
| Disk metric collected | ✅ Done | disk_used_percent |
| CPU Alarm created | ✅ Done | Static > 80% |
| Disk Alarm created | ✅ Done | Static > 80% |
| CPU Alarm triggered | ✅ Done | Peaked at 83.45% |
| Dashboard created | ✅ Done | 4 widgets — `shubham-singh-task-10` |
| Anomaly Detection explored | ✅ Done | Not used — insufficient data on new instance |
| Load testing performed | ✅ Done | stress tool + curl requests |
| EC2 terminated | ✅ Done | After task completion |

### Key Learnings

- **CloudWatch Agent is essential** — default EC2 metrics do not include Memory % or Disk %
- **IAM Role must be attached to EC2** — without it, the agent silently fails to send data
- **Manual config is better than wizard** — the wizard has too many prompts and can loop
- **Static threshold works from day one** — Anomaly Detection needs weeks of historical data
- **Dashboard gives real-time visibility** — critical for catching issues during load spikes
- **SNS + Alarms = automated alerting** — no need to manually watch metrics

---

*Documentation by Shubham Singh | Task 10 | PearlThoughts DevOps Internship | September 2026*
