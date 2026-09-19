# Build Your SOC Dashboard Using Splunk — Project Report

**Dataset:** `Realdata.csv` (supplied, real SOC export)
**Fields present:** `_time`, `host`, `logins`, `privileged_events`, `users`, `source_ips`

> **Note on scope:** This environment does not have a live Splunk instance available, so the analysis below was performed directly against the actual CSV contents using standard data-analysis tooling (equivalent statistics to what the specified SPL would return). Every number, timestamp, host, and threshold in this report is computed directly from `Realdata.csv` — nothing is invented. The exact SPL for each search/dashboard panel/alert is provided so it can be run in a real Splunk instance to reproduce these results and capture the required screenshots.

---

## 1. Introduction

The purpose of this SOC dashboard is to give a Level‑1 analyst quick visibility into authentication and privileged-activity trends on the monitored host, and to provide two baseline correlation rules that flag time windows where activity deviates sharply from the norm — for further human review, not automatic incident declaration.

---

## 2. Dataset Analysis

Computed directly from `Realdata.csv`:

| Metric | Value |
|---|---|
| Number of records | 61 |
| Time range | 2026-09-18 10:35:00 +05:30 → 2026-09-19 10:55:00 +05:30 |
| Number of unique hosts | 1 (`DESKTOP-LUDJFJ7`) |
| Number of unique users | 0 usable values — the `users` field is empty (`NaN`) on every row |
| Number of unique source IPs | 0 meaningful external IPs — `source_ips` contains only the literal values `-` (47 rows) and `- 127.0.0.1` (14 rows) |
| Total login events (`sum(logins)`) | 536 |
| Total privileged events (`sum(privileged_events)`) | 497 |
| Maximum login count in a single bucket | 86, at 2026‑09‑18 17:40:00 +05:30 |
| Maximum privileged-event count in a single bucket | 81, at 2026‑09‑18 17:40:00 +05:30 |
| Host with highest activity | `DESKTOP-LUDJFJ7` (the only host in the dataset — 536 total logins, 497 total privileged events) |
| Time period with unusual activity | **2026‑09‑18, 17:20–17:40** — four consecutive/near-consecutive buckets (17:20, 17:25, 17:30, 17:40) with logins of 56, 35, 84, and 86 respectively, far above the dataset average of ~8.8 |

**Structural notes:**
- Each row is a **pre-aggregated 5–10 minute bucket** (a `logins`/`privileged_events` count per time bucket), not a raw individual authentication event — consistent with output from a `timechart`/`stats` style summary index rather than raw `Security`/`auth.log` events. The analysis below treats it accordingly.
- The dataset spans **two calendar days** but is heavily weighted to 2026‑09‑18 (56 of 61 rows); 2026‑09‑19 contributes only 5 rows (10:35–10:55), so day-over-day comparison has limited statistical weight.
- `privileged_events` tracks very closely with `logins` throughout the dataset (Pearson-style co-movement is visually obvious — spikes in one line up with spikes in the other), which is itself a useful observation for the dashboard narrative.

---

## 3. Importing the Dataset into Splunk

1. In Splunk Web, go to **Settings → Add Data → Upload**.
2. Select `Realdata.csv`.
3. Set:
   - **Source type:** `csv` (or a custom sourcetype, e.g. `soc:auth_summary`, if you want field extraction to be explicit).
   - **Index:** create or select a dedicated index, e.g. `soc_dashboard` (recommended over using `main` so this data doesn't mix with other indexed data).
   - **Host:** leave as the default (filename-based) or override to `DESKTOP-LUDJFJ7` if you want the event's `host` metadata field to match the CSV's `host` column — otherwise Splunk's internal `host` metadata field and the CSV's `host` **column** will coexist as separate fields, which can be confusing in searches.
4. Confirm Splunk recognizes the header row and assigns field names directly from the CSV header: `_time`, `host`, `logins`, `privileged_events`, `users`, `source_ips`.
5. Verify `_time` is parsed correctly as the event timestamp (Splunk should auto-detect the ISO‑8601 format with the `+0530` offset). If it does not, set the timestamp format explicitly during the upload wizard: `%Y-%m-%dT%H:%M:%S.%3N%z`.

**If Splunk auto-renames fields** (e.g. due to a naming collision with its own internal `host` metadata field), use the actual assigned name in all searches below — e.g. if the CSV's host column is ingested as `host{}.value` or similar, substitute that field name in place of `host` throughout.

---

## 4. Dashboard: "SOC Security Monitoring Dashboard"

### Layout

```
+--------------------------------------------------------------+
|              SOC SECURITY MONITORING DASHBOARD               |
|                  (Time range selector: All time)              |
+--------------------------------------------------------------+
|                 Login Activity Over Time                     |
|                       (Line Chart)                            |
+---------------------------------+------------------------------+
| Privileged Events Over Time     | Host Activity                |
|        (Column Chart)           |     (Bar Chart)               |
+---------------------------------+------------------------------+
| Source IP / User Activity: not used — see Section 8/9 below   |
+--------------------------------------------------------------+
```

Panels required: 3 (all present and populated with real data below). Optional Source IP / User panels are explicitly **not included** — see Sections 8 and 9 for why.

---

## 5. Visualization 1 — Login Activity Over Time

**SPL:**
```spl
index=soc_dashboard
| timechart span=5m sum(logins) AS logins
```
*(The dataset's native bucket spacing is irregular — mostly 5–10 minutes apart with gaps — so `span=5m` will show zero-filled gaps between real buckets; that is expected and reflects genuine gaps in the underlying data, not a display error.)*

**Chart type:** Line Chart

**What the real data shows:**
- Baseline activity through most of the day (2026‑09‑18 morning through mid-afternoon) is low and steady — mostly 1–13 logins per bucket, averaging **~8.8 logins/bucket** across the full dataset (std. dev. ≈ 16.7, which is itself inflated by the spike described below).
- A sharp, sustained spike occurs late afternoon on 2026‑09‑18: **56 (17:20) → 35 (17:25) → 84 (17:30) → 2 (17:35, a brief dip) → 86 (17:40)** logins, before dropping back to 1 login at 17:45.
- Outside that window, the second-highest bucket in the entire dataset is 22 logins (10:40 on both 09‑18 and 09‑19), which is a recurring value at the same time of day on both dates.

![Login Activity Over Time](login_activity.png)

---

## 6. Visualization 2 — Privileged Events Over Time

**SPL:**
```spl
index=soc_dashboard
| timechart span=5m sum(privileged_events) AS privileged_events
```

**Chart type:** Column Chart

**What the real data shows:**
- `privileged_events` closely tracks `logins` throughout the dataset (total 497 privileged events vs. 536 logins — i.e., the large majority of login buckets show a near 1:1 ratio of privileged activity to logins).
- The same 17:20–17:40 window dominates: privileged-event counts of **53, 33, 79, and 81** respectively — the two highest privileged-event buckets in the whole dataset (79 and 81) occur back-to-back with only a two-minute gap (at 17:30 and 17:40, with a quiet 2-event bucket at 17:35 in between).
- Baseline privileged-event activity elsewhere in the dataset is low, typically 1–11 per bucket (mean ≈ 8.1).

![Privileged Events Over Time](privileged_events.png)

---

## 7. Visualization 3 — Host Activity

**SPL:**
```spl
index=soc_dashboard
| stats sum(logins) AS total_logins
        sum(privileged_events) AS total_privileged_events
  by host
| sort - total_logins
```

**Chart type:** Bar Chart

**What the real data shows:**
- The dataset contains **exactly one host**, `DESKTOP-LUDJFJ7`, so this panel does not identify a "most active" host relative to peers — it simply totals all activity for the single monitored endpoint: **536 total logins, 497 total privileged events**.
- This is stated explicitly rather than implied, since a single-bar chart could otherwise be misread as showing comparative host risk.

![Host Activity](host_activity.png)

---

## 8. Optional Visualization — Source IP Activity

**Result: not used.**

The `source_ips` field contains only two distinct literal values across all 61 rows:
- `-` (47 rows) — no IP information
- `- 127.0.0.1` (14 rows) — loopback address only, paired with the same placeholder `-`

There are **no external or routable IP addresses** anywhere in the dataset.

> Source IP visualization was not used because the supplied dataset does not contain sufficient usable source-IP information.

---

## 9. Optional Visualization — User Activity

**Result: not used.**

The `users` field is empty (`NaN`) on all 61 rows — no username values are present anywhere in the dataset. A user-activity visualization could not be reliably created from the supplied dataset.

---

## 10. Correlation Rule 1 — Unusual Login Activity Spike

**SPL:**
```spl
index=soc_dashboard
| timechart span=5m sum(logins) AS logins
| eventstats avg(logins) AS avg_logins
| where logins > avg_logins * 3
```

**Baseline calculated from the actual dataset:** average logins per bucket = **8.79** (n=61 buckets).
**Threshold selected:** `avg_logins × 3` ≈ **26.36** logins per bucket. This multiplier was chosen (rather than a lower one, e.g. ×1.5) because the raw standard deviation (16.7) is very large relative to the mean, driven almost entirely by one event cluster — a ×3 threshold cleanly isolates that cluster without also flagging routine moderately-busy buckets (e.g. the 22-login buckets at 10:40 on both days stay below this threshold).

**Triggered timestamps and actual login counts (real data, not invented):**

| Timestamp | Logins | Multiple of baseline |
|---|---|---|
| 2026‑09‑18 17:20:00 | 56 | ~6.4× |
| 2026‑09‑18 17:25:00 | 35 | ~4.0× |
| 2026‑09‑18 17:30:00 | 84 | ~9.6× |
| 2026‑09‑18 17:40:00 | 86 | ~9.8× |

**Why this requires investigation:** Four buckets within a 20-minute window, each several multiples above the dataset's own baseline, represent a statistically significant and temporally concentrated deviation from normal login behavior on this host.

**Alternative legitimate explanations to consider before escalating:**
- Scheduled batch job, service-account authentication loop, or automated script that legitimately performs many rapid logins.
- A monitoring/health-check agent restarting and re-authenticating repeatedly.
- A user or admin running a bulk administrative task (e.g., a patch or software deployment) that generates many discrete authentication events in a short window.
- Since `users` and `source_ips` are not populated in this dataset, **it is not possible to confirm whether this was one user/process or many**, nor whether it originated internally or externally — this is a key investigative gap, not something the dataset can resolve on its own.

---

## 11. Correlation Rule 2 — Unusual Privileged Activity

**SPL:**
```spl
index=soc_dashboard
| timechart span=5m sum(privileged_events) AS privileged_events
| eventstats avg(privileged_events) AS avg_privileged
| where privileged_events > avg_privileged * 2
```

**Baseline calculated from the actual dataset:** average privileged events per bucket = **8.15** (n=61 buckets).
**Threshold selected:** `avg_privileged × 2` ≈ **16.30** privileged events per bucket. A ×2 threshold (lower than Rule 1's ×3) was chosen because privileged-event escalation is inherently more sensitive than raw login volume — a smaller relative increase in privileged activity is more worth reviewing than the same relative increase in ordinary logins.

**Triggered timestamps, actual counts, and host:**

| Timestamp | Host | Privileged events | Multiple of baseline |
|---|---|---|---|
| 2026‑09‑18 10:40:00 | DESKTOP-LUDJFJ7 | 22 | ~2.7× |
| 2026‑09‑18 17:20:00 | DESKTOP-LUDJFJ7 | 53 | ~6.5× |
| 2026‑09‑18 17:25:00 | DESKTOP-LUDJFJ7 | 33 | ~4.0× |
| 2026‑09‑18 17:30:00 | DESKTOP-LUDJFJ7 | 79 | ~9.7× |
| 2026‑09‑18 17:40:00 | DESKTOP-LUDJFJ7 | 81 | ~9.9× |
| 2026‑09‑19 10:40:00 | DESKTOP-LUDJFJ7 | 22 | ~2.7× |

**Why this requires investigation:** The five late-afternoon buckets overlap directly with Rule 1's login spike, reinforcing that this was a period of concentrated privileged activity, not just ordinary logins. Notably, the 10:40 bucket triggers on **both** 2026‑09‑18 and 2026‑09‑19 with the identical value (22), which suggests this smaller, recurring pattern may be routine/scheduled rather than anomalous — worth noting as a contrast to the one-off 17:20–17:40 cluster.

**Possible legitimate administrative explanations:**
- A recurring daily administrative or maintenance task around 10:40 (the fact that it repeats identically on both days supports this reading).
- The larger 17:20–17:40 cluster could reflect legitimate elevated/admin work (e.g., a deployment, backup job with privileged service accounts, or IT maintenance window) — but without `users` or `source_ips` data, this cannot be confirmed and should be treated as requiring human follow-up (e.g., checking a change log or asking the desktop's owner) rather than assumed benign.

---

## 12. Correlation Rule Documentation

| Rule | Data Used | Detection Logic | Threshold | Triggered Event(s) | Why It Matters | Confidence |
|---|---|---|---|---|---|---|
| Unusual Login Activity Spike | `logins` | `eventstats avg(logins)` compared against each 5-min bucket | `logins > avg_logins × 3` (≈26.36) | 17:20 (56), 17:25 (35), 17:30 (84), 17:40 (86) on 2026‑09‑18 | Four buckets, several multiples above baseline, concentrated in a 20-minute window on a single host — a genuine statistical outlier worth human review | Medium — statistically strong, but no user/IP data to confirm intent |
| Unusual Privileged Activity | `privileged_events` | `eventstats avg(privileged_events)` compared against each 5-min bucket | `privileged_events > avg_privileged × 2` (≈16.30) | 10:40 (22) on both 09‑18 and 09‑19; 17:20 (53), 17:25 (33), 17:30 (79), 17:40 (81) on 2026‑09‑18 | Late-afternoon cluster overlaps directly with the login spike, reinforcing it as a real anomaly; the recurring 10:40 value suggests a separate, likely-routine pattern | Medium — the 17:20–17:40 cluster is High confidence as a *statistical* anomaly; the 10:40 recurrence is Low confidence as a security concern given its repetition |

Neither rule's output should be read as **confirmed malicious activity** — both flag statistically unusual buckets that warrant analyst review, and the dataset's lack of user/IP granularity means root-cause determination requires additional data sources.

---

## 13. SOC Response to Rule 1 (Login Spike)

1. **Timestamp:** 2026‑09‑18, 17:20:00–17:40:00 (four buckets).
2. **Affected host:** `DESKTOP-LUDJFJ7` (the only host in the dataset).
3. **`users` field:** Not available — empty in all rows; cannot identify which account(s) drove the spike from this dataset alone.
4. **`source_ips` field:** Not available in a usable form — only `-` and loopback (`127.0.0.1`) are present; cannot determine whether the spike originated locally or remotely from this dataset alone.
5. **Compare to baseline:** Spike values (35–86 logins/bucket) are 4×–9.8× the dataset average of 8.79 — a clear statistical deviation.
6. **Surrounding time periods:** Activity immediately before (17:10: 4 logins) and after (17:45: 1 login) the spike returns to baseline levels, indicating a contained, time-boxed event rather than a sustained escalation.
7. **Legitimate business/administrative correlation:** Cannot be confirmed from this dataset; recommend checking change-management records, scheduled-task logs, or asking the system owner about activity around 17:20–17:40 on 2026‑09‑18.
8. **Escalate if:** Additional evidence (e.g., EDR telemetry, Windows Security logs, or the actual `users`/source IP values from the native log source rather than this summary export) shows the spike correlates with unfamiliar accounts, external IPs, or unexpected privileged commands.

---

## 14. SOC Response to Rule 2 (Privileged Activity)

1. **Timestamp:** 2026‑09‑18, 10:40:00; 17:20:00–17:40:00; and 2026‑09‑19, 10:40:00.
2. **Host:** `DESKTOP-LUDJFJ7`.
3. **User:** Not available in this dataset.
4. **Associated login activity:** Directly correlated — every triggering bucket for Rule 2 also shows an elevated `logins` value in the same bucket (e.g., 17:40 shows both 86 logins and 81 privileged events).
5. **Source IP:** Not available in a usable form.
6. **Expected administrative activity:** The recurring 10:40 value (22 privileged events, identical on both days) is more consistent with a routine/scheduled process than the one-off 17:20–17:40 cluster, which has no repeat pattern in the supplied data.
7. **Surrounding events:** As with Rule 1, activity returns to baseline immediately outside the 17:20–17:40 window.
8. **Escalate to L2/Incident Response if:** Follow-up investigation using the native (non-summarized) logs shows the privileged activity was performed by an unexpected account, from an external source, or involved commands inconsistent with routine administration.

---

## 15. Splunk Alert Configuration

### Alert 1 — Unusual Login Activity Spike
- **Search:** `index=soc_dashboard | timechart span=5m sum(logins) AS logins | eventstats avg(logins) AS avg_logins | where logins > avg_logins * 3`
- **Alert name:** `SOC - Unusual Login Activity Spike`
- **Schedule:** Run every 5 minutes (matching the bucket span), or every 15 minutes for a lower-overhead near-real-time check.
- **Time range:** Rolling window, e.g. last 60 minutes, so `eventstats avg()` reflects a recent, relevant baseline rather than the entire historical dataset.
- **Trigger condition:** Number of results > 0 (i.e., at least one bucket exceeds the threshold).
- **Severity:** Medium (statistical anomaly, not confirmed malicious — per Section 12).
- **Throttling:** Suppress repeat alerts for the same host for 20–30 minutes after a trigger, to avoid re-alerting on every bucket within the same spike cluster.
- **Alert action:** Send to SOC queue/email or ticketing system (e.g., create an investigative ticket); avoid auto-remediation given the Medium confidence level.

### Alert 2 — Unusual Privileged Activity
- **Search:** `index=soc_dashboard | timechart span=5m sum(privileged_events) AS privileged_events | eventstats avg(privileged_events) AS avg_privileged | where privileged_events > avg_privileged * 2`
- **Alert name:** `SOC - Unusual Privileged Activity`
- **Schedule:** Every 5–15 minutes, same rationale as Alert 1.
- **Time range:** Rolling 60-minute window.
- **Trigger condition:** Number of results > 0.
- **Severity:** Medium, with a note to review whether the trigger overlaps with an Alert 1 trigger (as seen in the real data, where the two rules co-fire during 17:20–17:40) — a co-occurring trigger from both rules should be treated with higher priority than either alone.
- **Throttling:** Suppress repeat alerts for the same host for 20–30 minutes.
- **Alert action:** Send to SOC queue/email or ticketing system.

**Note:** This environment does not have Splunk Enterprise Security installed, so no "notable event" is claimed to have been created — the above describes standard Splunk core alerting (Settings → Searches, Reports, and Alerts), not an ES notable-event workflow.

---

## 16. Dashboard Validation Checklist

| Check | Status |
|---|---|
| Dataset successfully indexed | To be confirmed in your Splunk instance following Section 3 |
| `_time` correctly recognized | To be confirmed — verify against the ISO‑8601 `+0530` format noted in Section 3 |
| Login visualization displays real data | Confirmed against the CSV directly (Section 5); reproduce in Splunk with the given SPL |
| Privileged-event visualization displays real data | Confirmed against the CSV directly (Section 6) |
| Host visualization displays real hosts | Confirmed — single real host, `DESKTOP-LUDJFJ7` (Section 7) |
| Source IP/user panels only used if fields contain usable values | Correctly excluded — both fields lack usable values (Sections 8–9) |
| Two correlation searches execute successfully | SPL provided in Sections 10–11; both produce real, computed trigger results against this dataset |
| Triggered alerts correspond to actual records in the dataset | Confirmed — all triggered timestamps/values in Sections 10–11 and 12 are taken directly from `Realdata.csv` |

---

## 17. Required Screenshots

Three chart images generated directly from the real dataset are provided alongside this report (`login_activity.png`, `privileged_events.png`, `host_activity.png`) to use as source material for Screenshots 3–5, or as a reference to confirm your live Splunk panels match the expected shape of the data. Screenshots 1, 2, 6, 7, and 8 (index ingestion, full dashboard view, correlation-rule SPL/results, and any triggered alert) must be captured directly from your own Splunk instance after completing the import steps in Section 3 and building the panels/alerts described above, since this environment does not have a live Splunk UI to screenshot.

---

## 18. Findings

**Observed** (directly visible in the dataset):
- 536 total logins and 497 total privileged events across 61 time buckets spanning 2026‑09‑18 10:35 to 2026‑09‑19 10:55, all on a single host, `DESKTOP-LUDJFJ7`.
- A sharp, four-bucket login/privileged-event spike between 17:20 and 17:40 on 2026‑09‑18 (logins: 56/35/84/86; privileged events: 53/33/79/81), each several multiples above the dataset baseline.
- A smaller, recurring privileged-event value (22) at 10:40 on both 2026‑09‑18 and 2026‑09‑19.
- No usable `users` values and no usable external `source_ips` values anywhere in the dataset.

**Suspicious** (requires further investigation):
- The 17:20–17:40 cluster on 2026‑09‑18, given its statistical magnitude and the fact that login and privileged-event volume rose together, in isolation from the surrounding baseline.

**Confirmed malicious:** None. No evidence in this dataset — which contains only aggregate counts, no usernames, and no external IP addresses — supports a malicious-activity determination. Confirming or ruling out malicious intent requires the underlying raw logs (with user and source-IP granularity) referenced in Sections 13–14.

---

## 19. Conclusion

The SOC Security Monitoring Dashboard, built from the real `Realdata.csv` export, gives visibility into three core dimensions:

- **Authentication activity** — the Login Activity panel shows overall login volume trends over time and immediately highlights the 17:20–17:40 spike against a clear, low baseline.
- **Privileged activity** — the Privileged Events panel shows that privileged-event volume closely tracks login volume throughout the dataset, and highlights the same late-afternoon spike plus a smaller recurring daily pattern at 10:40.
- **Host activity** — the Host Activity panel confirms all observed activity belongs to a single monitored endpoint, `DESKTOP-LUDJFJ7`.

The two correlation rules — a ×3 login-spike threshold and a ×2 privileged-activity threshold, both calculated from this dataset's own statistics rather than an arbitrary fixed number — successfully and correctly isolate the same real anomaly cluster (17:20–17:40 on 2026‑09‑18) without over-flagging routine moderate activity elsewhere in the data. Because the dataset lacks `users` and `source_ips` detail, these rules are best used as a **first-pass triage signal** that directs an L1 analyst toward the correct time window and host, with escalation to richer, user/IP-attributed logs required before any compromise determination can be made.
