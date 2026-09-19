# Splunk SOC Security Monitoring Dashboard

## Overview

This project demonstrates a basic Security Operations Center (SOC)
monitoring dashboard built using Splunk.

The project uses a real SOC dataset, `Realdata.csv`, to monitor:

- Login activity
- Privileged activity
- Host activity
- Unusual activity patterns

## Objectives

- Build a basic SOC monitoring dashboard
- Analyze authentication activity
- Monitor privileged events
- Identify unusual activity
- Create correlation/detection rules
- Demonstrate L1 SOC investigation workflow

## Tools Used

- Splunk
- SPL (Search Processing Language)
- Windows Security Event concepts
- SIEM
- SOC monitoring

## Dataset

The dataset contains:

- `_time`
- `host`
- `logins`
- `privileged_events`
- `users`
- `source_ips`

The supplied dataset contains 61 records and one host:
`DESKTOP-LUDJFJ7`.

## Dashboard Visualizations

### 1. Login Activity Over Time

Shows authentication/login activity across time.

### 2. Privileged Events Over Time

Shows privileged-event activity across time.

### 3. Host Activity

Shows total login and privileged activity for monitored hosts.

## Detection Rules

### Rule 1 — Unusual Login Activity Spike

Detects login activity significantly above the observed baseline.

### Rule 2 — Unusual Privileged Activity

Detects privileged activity significantly above the observed baseline.

## Project Structure

```text
├── README.md
├── dataset/
├── report/
├── screenshots/
└── spl/
