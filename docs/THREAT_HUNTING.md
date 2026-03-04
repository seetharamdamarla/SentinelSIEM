# Threat Hunting Documentation

While SIEM alerts trigger reactively based on known, bad behavior (Detection Engineering), **Threat Hunting** is the proactive methodology of searching through logs to find hidden anomalies that evaded automated rules.

This document contains KQL (Kibana Query Language) / OpenSearch queries that a SOC Analyst can execute within the Wazuh Dashboard (Discover tab) to hunt for adversaries in the SentinelSIEM environment.

---

## Scenario 1: Hunting for "Low and Slow" Password Spraying
Brute force rules (like Rule `100001`) often rely on a high frequency of failures (e.g., 6 failures in 2 minutes) to trigger. An advanced adversary might try 1 password every 5 minutes (Password Spraying) to evade detection.

**Query:**
```kql
rule.id: ("5716" OR "5710") AND agent.name: "macOS-endpoint"
```
**Hunting Methodology:**
1. Execute query over a broader time range: **Last 7 Days**.
2. Group the visualization by `srcip` and `data.srcuser`.
3. Look for a single source IP that has repeatedly attempted logins across multiple days at a slow interval.
4. **Action:** If found, add the IP to the Threat Intelligence blocklist.

---

## Scenario 2: Hunting for Suspicious Process Execution
Adversaries often rename malicious binaries to look legitimate (e.g., renaming `nmap` to `systemd-update`) or execute shell programs from unexpected directories like `/tmp`.

**Query:**
```kql
rule.groups: "ossec" AND data.command: *
```
*(Note: Requires command logging to be enabled on the macOS agent via `auditd` or history monitoring)*

**Hunting Methodology:**
1. Filter out known administrative commands.
2. Specifically search using RegEx or wildcard KQL for anomalous directories:
   ```kql
   data.command: (\/tmp\/* OR \/Library\/Caches\/*)
   ```
3. Look for executions of LOLBin (Living off the Land Binaries) commands like `curl`, `python3 -c`, or `base64`.
   ```kql
   data.command: ("*base64 -d*" OR "*curl* http*")
   ```

---

## Scenario 3: Hunting for Persistence Mechanisms
Once an attacker gains access, they will try to maintain it by installing backdoors or modifying startup scripts.

**Query:**
```kql
rule.groups: ("fim" OR "syscheck") AND (data.file: *LaunchDaemons* OR data.file: *LaunchAgents* OR data.file: *cron*)
```

**Hunting Methodology:**
1. macOS uses LaunchDaemons and LaunchAgents to run processes at boot. Modification of these paths is a primary persistence technique (MITRE T1543.004).
2. Look through the `data.file` logs for newly created `.plist` files.
3. Compare the modified file against known baseline images. If a new daemon calls an unknown python script, investigate immediately.

---

## Scenario 4: Investigating Abnormal Authentication Hours
In a corporate environment, users generally log in during business hours (e.g., 8 AM - 6 PM). Logins at 3 AM might indicate a compromised off-hours account.

**Query:**
```kql
rule.groups: ("authentication_success") AND agent.id: "001"
```
**Hunting Methodology:**
1. Set the time picker to **Last 30 Days**.
2. Create a Heatmap visualization based on the `Hour of the Day` vs `Day of the Week`.
3. Identify "hotspots" occurring during the weekend or middle of the night.
4. Investigate the associated users: Are they remote workers in a different time zone, or is it an attacker operating from another country?
