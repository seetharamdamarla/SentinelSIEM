# SentinelSIEM Architecture

This document outlines the architecture of the SentinelSIEM environment, which consists of a centralized SIEM infrastructure and an endpoint agent continuously forwarding security events.

## Logical Architecture Diagram

```mermaid
graph TD
    classDef manager fill:#00a2ed,stroke:#0079b3,stroke-width:2px,color:#fff;
    classDef agent fill:#34a853,stroke:#2b8c45,stroke-width:2px,color:#fff;
    classDef threat fill:#ea4335,stroke:#b33529,stroke-width:2px,color:#fff;
    classDef user fill:#fbbc05,stroke:#d39e00,stroke-width:2px,color:#111;

    %% Components
    subgraph "Endpoints (Monitored Network)"
        macOS["macOS Workstation\n(Wazuh Agent)"]:::agent
    end

    subgraph "SIEM Backend (Ubuntu Server VM)"
        WazuhManager["Wazuh Manager\n(Rule Engine & Decoders)"]:::manager
        WazuhIndexer["Wazuh Indexer\n(OpenSearch Engine)"]:::manager
        WazuhDashboard["Wazuh Dashboard\n(Kibana Interface)"]:::manager
    end

    subgraph "External Feeds"
        ThreatIntel["Threat Intelligence Feeds\n(AlienVault OTX, Abuse.ch)"]:::threat
    end
    
    SOCAnalyst["SOC Analyst"]:::user

    %% Data Flow
    macOS -- "Secure TLS Connection\n(Port 1514)" --> WazuhManager
    macOS -- "Authentication Logs (authd)\nFile Integrity (syscheck)\nSyslog" --> WazuhManager
    WazuhManager -- "Log Enrichment" --> ThreatIntel
    ThreatIntel -- "IOC Data" --> WazuhManager
    WazuhManager -- "Indexed Jason Documents" --> WazuhIndexer
    WazuhIndexer -- "Data Queries" --> WazuhDashboard
    SOCAnalyst -- "HTTPS Dashboard Access\n(Port 443)" --> WazuhDashboard
```

## Component Details

### 1. The SIEM Backend (Wazuh Manager on Ubuntu)
The SOC "Brain". It receives events from the agent, decodes them, and evaluates them against custom rules and built-in rulesets.
- **Wazuh Indexer:** A highly scalable, full-text search and analytics engine. This component stores the log data as JSON documents.
- **Wazuh Dashboard:** The web user interface where the SOC Analyst performs threat hunting, alert triage, and visualizes security events.
- **Threat Intelligence Integrations:** Using Wazuh's CDB lists and native Virustotal/AlienVault integrations, extracted IPs and file hashes are compared against known malicious lists.

### 2. The Endpoint (macOS Wazuh Agent)
The Wazuh agent runs as a background service on the macOS machine. It periodically collects and forwards:
- **Authentication Logs:** Failed SSH attempts, local graphical logins, and `su` commands.
- **File Integrity Monitoring (FIM):** The `syscheck` module creates cryptographic baselines of critical files (`/etc/passwd`, `/etc/shadow`, `/etc/hosts`).
- **Command Executions:** Capturing bash history or auditd equivalents to monitor what software is executed.
