# ☁️ AWS EC2 Disk Space Monitoring & Alerting (Production-Ready)

## 📌 Overview

This project implements a **production-grade disk space monitoring system** for an **Ubuntu EC2 instance** using native AWS services.

When disk usage crosses a defined threshold, the system automatically sends an **email alert via Amazon SNS**, without storing any credentials on the server.

This repository documents the **complete journey**, including:

* Initial errors
* Root cause analysis
* Correct architecture
* Secure IAM setup
* Final working deployment

This is **not a demo script**.
This is how disk monitoring should be implemented in **real production environments**.

---

## 🎯 Problem Statement

Disk exhaustion is one of the most common causes of production outages.

Common issues:

* Instances run out of disk silently
* Alerts are misconfigured or never tested
* Credentials are hardcoded (security risk)
* Cron jobs fail due to missing paths or permissions

The goal was to build a solution that is:

* Secure
* Automated
* Auditable
* Cron-safe
* Production-ready

---

## 🧠 Final Architecture

```
Disk Usage Check (Bash Script)
        ↓
AWS CLI (on EC2)
        ↓
IAM Role (Temporary Credentials)
        ↓
SNS Topic
        ↓
Email Notification
```

If **any single component is missing**, the system fails.
This repository documents how to build **every piece correctly**.

---

## 🚨 Initial Issue & Root Cause Analysis

### ❌ Observed Error

```
Error: Topic does not exist
```

### 🔍 Investigation

SNS topics were listed in the target region, but the required topic for disk monitoring was **missing**.

### ✅ Root Cause

The SNS topic for disk alerts had **not been created** in the correct AWS account and region.

This issue was **not related to**:

* IAM
* Email subscriptions
* AWS CLI
* Permissions

SNS is **region-scoped**, and topics must exist **before subscribing or publishing**.

---

## ✅ Design Decision: One Topic per Concern

| Use Case          | SNS Topic Naming Example |
| ----------------- | ------------------------ |
| Disk Monitoring   | `disk-space-alerts`      |
| CloudTrail Events | `cloudtrail-*`           |
| Budget Alerts     | `*-budget-alerts`        |
| Infra Automation  | `resource-*`             |

This keeps alerts:

* Isolated
* Maintainable
* Easy to debug

---

## 🟢 Part 1 — AWS One-Time Setup

### 1️⃣ Create SNS Topic

```bash
aws sns create-topic \
  --name disk-space-alerts \
  --region <AWS_REGION>
```

Example output:

```json
{
  "TopicArn": "arn:aws:sns:<AWS_REGION>:<ACCOUNT_ID>:disk-space-alerts"
}
```

---

### 2️⃣ Subscribe Email to SNS

```bash
aws sns subscribe \
  --topic-arn arn:aws:sns:<AWS_REGION>:<ACCOUNT_ID>:disk-space-alerts \
  --protocol email \
  --notification-endpoint <YOUR_EMAIL> \
  --region <AWS_REGION>
```

📧 Email confirmation is **mandatory**.

---

### 3️⃣ Create IAM Policy (Least Privilege)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "sns:Publish",
      "Resource": "arn:aws:sns:<AWS_REGION>:<ACCOUNT_ID>:disk-space-alerts"
    }
  ]
}
```

Policy name:

```
AllowDiskSpaceAlertsSNS
```

---

### 4️⃣ Create IAM Role for EC2

* Trusted entity: **EC2**
* Attach policy: `AllowDiskSpaceAlertsSNS`
* Example role name: `EC2-Disk-Monitor-Role`

---

### 5️⃣ Attach IAM Role to EC2 Instance

EC2 → Instance → Actions → Security → Modify IAM Role
Attach the role created above.

✔ Takes effect immediately
✔ No reboot required

---

## 🟢 Part 2 — EC2 Server Setup (Ubuntu)

### 6️⃣ Install AWS CLI

```bash
sudo apt-get update
sudo apt-get install awscli -y
```

Verify:

```bash
aws --version
```

---

### 7️⃣ Verify IAM Role

```bash
aws sts get-caller-identity
```

Expected:

```json
{
  "Account": "<ACCOUNT_ID>"
}
```

---

### 8️⃣ Test SNS Access

```bash
aws sns publish \
  --topic-arn arn:aws:sns:<AWS_REGION>:<ACCOUNT_ID>:disk-space-alerts \
  --message "IAM + SNS test" \
  --region <AWS_REGION>
```

✔ Email received → AWS configuration is correct
❌ AccessDenied → IAM role or policy is incorrect

---

## 🟢 Part 3 — Production Script Deployment

### 9️⃣ Create Script

```bash
sudo nano /opt/disk_alert.sh
```

```bash
#!/bin/bash

THRESHOLD=90
PARTITION="/"
TOPIC_ARN="arn:aws:sns:<AWS_REGION>:<ACCOUNT_ID>:disk-space-alerts"
AWS_REGION="<AWS_REGION>"

PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
export PATH

CURRENT=$(df -h "$PARTITION" | awk 'NR==2 {print $5}' | tr -d '%')

if [ "$CURRENT" -ge "$THRESHOLD" ]; then
    logger -t DiskMonitor "CRITICAL: Disk usage at ${CURRENT}%"

    TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
      -H "X-aws-ec2-metadata-token-ttl-seconds: 60")

    if [ -n "$TOKEN" ]; then
        INSTANCE_ID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
          http://169.254.169.254/latest/meta-data/instance-id)
    else
        INSTANCE_ID=$(hostname)
    fi

    aws sns publish \
      --topic-arn "$TOPIC_ARN" \
      --subject "ALARM: Disk usage ${CURRENT}%" \
      --message "Disk usage on $INSTANCE_ID is ${CURRENT}%" \
      --region "$AWS_REGION"
fi
```

---

### 🔐 Secure Permissions

```bash
sudo chown root:root /opt/disk_alert.sh
sudo chmod 700 /opt/disk_alert.sh
```

---

### 🧪 Manual Test

Temporarily set:

```bash
THRESHOLD=1
```

Run:

```bash
sudo /opt/disk_alert.sh
```

✔ Alert received → restore threshold to `90`

---

## 🟢 Part 4 — Automation with Cron

```bash
sudo crontab -e
```

```bash
0 * * * * /opt/disk_alert.sh
```

Runs:

* Every hour
* As root
* Production-safe

---

## 🔍 Observability

```bash
sudo tail -f /var/log/syslog
```

Example logs:

```
DiskMonitor: CRITICAL: Disk usage at 92%
DiskMonitor: SUCCESS: Alert sent to SNS
```

---

## ✅ Final Checklist

✔ SNS topic created
✔ Email subscription confirmed
✔ IAM policy created
✔ IAM role attached to EC2
✔ AWS CLI installed
✔ Script secured
✔ Manual test passed
✔ Cron enabled

---

## 🏁 Conclusion

This project demonstrates:

* IAM-based authentication (no secrets stored)
* Least-privilege security
* Production-safe cron execution
* Clear root cause analysis and debugging
* Clean separation of AWS concerns

This is a **real-world DevOps implementation**, not a tutorial shortcut.

---

## 🚀 Future Enhancements

* Slack / Teams alerts
* SMS notifications
* Multi-partition monitoring
* systemd timers instead of cron
* CloudWatch-native implementation

---

⭐ If this repository helped you, consider starring it to help others find reliable DevOps references.
