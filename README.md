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
This is how disk monitoring should be done in **real production environments**.

---

## 🎯 Problem Statement

Disk exhaustion is one of the most common causes of production outages.

Common issues:

* Instances run out of disk silently
* Alerts are misconfigured or never tested
* Credentials are hardcoded (security risk)
* Cron jobs fail due to missing paths or permissions

The goal was to build:

* A **secure**
* **automated**
* **auditable**
* **cron-safe**
  solution using AWS best practices.

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

Listing SNS topics in `us-east-1` showed:

* ws-resource-creation-alerts
* bedrock-haiku-budget-alert
* cloudtrail-pb9-notifications
* spectra-send-notification

❌ The required topic **`disk-space-alerts` did not exist**

### ✅ Root Cause

The SNS topic was **never created in the account/region**.

This issue was **not related to**:

* IAM
* Email
* AWS CLI
* Permissions

SNS is **region-specific**, and topics must exist **before subscribing or publishing**.

---

## ✅ Correct Design Decision

### One Topic Per Concern (Best Practice)

| Use Case          | SNS Topic Name    |
| ----------------- | ----------------- |
| Disk Monitoring   | disk-space-alerts |
| CloudTrail Events | cloudtrail-*      |
| Budget Alerts     | *-budget-alert    |
| Infra Automation  | resource-*        |

This keeps alerts **clean, searchable, and maintainable**.

---

## 🟢 Part 1 — AWS One-Time Setup

### 1️⃣ Create SNS Topic

```bash
aws sns create-topic \
  --name disk-space-alerts \
  --region us-east-1
```

Expected output:

```json
{
  "TopicArn": "arn:aws:sns:us-east-1:891146181139:disk-space-alerts"
}
```

---

### 2️⃣ Subscribe Email

```bash
aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:891146181139:disk-space-alerts \
  --protocol email \
  --notification-endpoint your-email@example.com \
  --region us-east-1
```

📧 **Email confirmation is mandatory**
Unconfirmed subscriptions receive **zero alerts**.

---

### 3️⃣ Create IAM Policy (Least Privilege)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "sns:Publish",
      "Resource": "arn:aws:sns:us-east-1:891146181139:disk-space-alerts"
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
* Role name: `EC2-Disk-Monitor-Role`

---

### 5️⃣ Attach Role to EC2 Instance

EC2 → Instance → Actions → Security → Modify IAM Role
Attach: `EC2-Disk-Monitor-Role`

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
"Account": "891146181139"
```

If this fails → IAM role is not attached correctly.

---

### 8️⃣ Test SNS Access (Critical)

```bash
aws sns publish \
  --topic-arn arn:aws:sns:us-east-1:891146181139:disk-space-alerts \
  --message "IAM + SNS test" \
  --region us-east-1
```

✔ Email received → AWS side is correct
❌ AccessDenied → stop and fix IAM

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
TOPIC_ARN="arn:aws:sns:us-east-1:891146181139:disk-space-alerts"
AWS_REGION="us-east-1"

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

✔ Email received → restore threshold to `90`

---

## 🟢 Part 4 — Automation with Cron

### ⏱ Enable Cron

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

Expected logs:

```
DiskMonitor: CRITICAL: Disk usage at 92%
DiskMonitor: SUCCESS: Alert sent to SNS
```

---

## ✅ Final Checklist

✔ SNS topic exists
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

* Real IAM-based security
* Zero credential leakage
* Production-safe automation
* Proper AWS service boundaries
* Debugging through root cause analysis

This is **how disk monitoring should be built in production**.

---

## 🚀 Future Enhancements

* Slack / Teams alerts
* SMS notifications
* Multi-partition monitoring
* systemd timer instead of cron
* CloudWatch-native version

---

### ⭐ If this helped you

Consider starring the repository — it helps others find reliable DevOps references.
