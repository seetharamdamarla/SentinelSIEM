# Attack Simulation Guide

This document provides step-by-step instructions on how to simulate attacks against the macOS Wazuh Agent endpoint. These exercises are designed to trigger the custom SOC alerts and validate SentinelSIEM's effectiveness.

> **WARNING:** Only run these steps in your authorized lab environment. Do not execute these commands on production systems without proper authorization.

---

## 1. Brute Force Login Attempts (MITRE T1110)
**Objective:** Trigger Rule `100001` (Level 10) by simulating an adversary attempting to guess an SSH password.

**Execution:**
Run the following bash loop from another machine in the network, or locally on the macOS endpoint pointing to `localhost`:
```bash
for i in {1..7}; do ssh invalid_user@localhost -p 22; done
```
*Expected Result:* The Wazuh Manager will interpret 6 or more failed logins within 120 seconds as a Brute Force attack. An alert titled **"Multiple SSH Failed Login Attempts (Brute Force Detection)"** will appear in the dashboard.

---

## 2. Unauthorized File Modification (FIM) (MITRE T1547)
**Objective:** Trigger Rule `100002` (Level 12) by modifying a highly sensitive configuration file tracked by `syscheck`.

**Execution:**
In the macOS terminal, simulate a modification to the `hosts` file (or another monitored file):
```bash
echo "# Simulation FIM modification" | sudo tee -a /etc/hosts
```
*Expected Result:* Wazuh's File Integrity Monitoring baseline checker will notice the hash change. An alert titled **"Unauthorized Modification of Critical System File: /etc/hosts"** will be generated.

---

## 3. Privilege Escalation Attempts (MITRE T1068)
**Objective:** Trigger Rule `100004` (Level 11) by simulating an attacker repeatedly failing `sudo` authentication while trying to gain root privileges.

**Execution:**
On macOS, first clear the sudo credential cache, then intentionally provide the wrong password three times in a row. Note: macOS uses `/var/root` (not `/root` which is Linux-only).
```bash
# Step 1: Clear the sudo cache so macOS prompts for password
sudo -k

# Step 2: Attempt sudo and enter the WRONG password 3 times
sudo ls /var/root
# Enter wrong password when prompted — repeat 3 times
```
*Expected Result:* An alert titled **"Privilege Escalation Attempt: Multiple failed sudo commands"** will trigger in the SIEM.

---

## 4. New User Account Creation (MITRE T1078 / T1136.001)
**Objective:** Trigger Rule `100003` (Level 8) by simulating the creation of a persistence account.

**Execution:**
On the macOS endpoint, create a generic test user using the `sysadminctl` CLI utility.
```bash
sudo sysadminctl -addUser rogue_admin -fullName "Rogue User" -password "Password123!"

# After testing, clean up the rogue account:
sudo sysadminctl -deleteUser rogue_admin
```
*Expected Result:* The `authd` logs on macOS will register the creation. The SIEM will parse this and trigger **"New macOS User Account Created"**.

---

## 5. Suspicious System Behavior (MITRE T1105)
**Objective:** Trigger Rule `100005` (Level 9) by simulating an adversary attempting to download a malicious payload via terminal utilities.

**Execution:**
Simulate pulling down a reconnaissance script using `curl` silently.
```bash
curl -sL https://raw.githubusercontent.com/EmpireProject/Empire/master/setup/install.sh > /tmp/malicious.sh
```
*Expected Result:* The bash history/command execution logs sent to Wazuh will flag the specific CLI arguments (`-sL`) associated with stealth downloads, triggering the **"Suspicious System Behavior"** alert.

---

## Conclusion
Once these simulations are complete, move over to the **Wazuh Dashboard** and follow the procedures outlined in the [SOC Workflow Guide](SOC_WORKFLOW.md) to practice triage and investigation.
