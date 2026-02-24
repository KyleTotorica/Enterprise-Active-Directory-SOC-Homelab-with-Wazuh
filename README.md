Project Overview

This project is a hands-on Security Operations Center (SOC) home lab designed to simulate real-world monitoring, log ingestion, incident detection, and operational troubleshooting workflows.

The environment combines endpoint telemetry, centralized SIEM monitoring, and a multi-tier application architecture to mirror how production systems are monitored and supported. In addition to traditional security visibility, the lab focuses heavily on service reliability, dependency monitoring, controlled incident simulation, and operational automation.

The lab uses Wazuh as the SIEM platform, Windows and Linux endpoints for telemetry, and a Kali Linux machine to simulate external attacker behavior. The environment evolved into a production-style architecture that includes an internal web application and a dedicated database server, enabling realistic monitoring of service dependencies and outage scenarios.

Environment
Ubuntu Linux — Wazuh Manager, Indexer, and Dashboard
Windows Server — Domain Controller
Windows 10 — User Endpoint
Kali Linux — External attacker simulation
Ubuntu Server — Internal Application Server (Flask + Nginx)
Ubuntu Server — Database Server (PostgreSQL)

Telemetry Sources
Windows Event Logs
Sysmon (process and network telemetry)
Linux syslog
Authentication events (domain + local)
Nginx access and error logs
Application health logs
Watchdog automation logs
Deployment and triage automation logs

Objectives
Monitor authentication activity, endpoint behavior, and directory changes
Detect attacker techniques using real telemetry
Practice SOC-style triage and investigation workflows
Practice troubleshooting internal application outages and dependency failures
Implement automation to reduce downtime and accelerate recovery
Simulate production support responsibilities (deployments, monitoring, escalation)

Implementation Process
Wazuh SIEM Deployment
Installed and configured Wazuh Manager, Indexer, and Dashboard on Ubuntu
Enrolled agents on Windows Server, Windows 10, and Linux systems
Troubleshot agent enrollment issues including duplicate names and connectivity problems
Verified persistent agent connectivity and log ingestion 
![added agents](Images/WazuhDashboard.png)

Windows Endpoint Telemetry (Sysmon)
Installed Sysmon on Windows endpoints
Configured Sysmon to capture high-value telemetry such as:
  - Process creation (Event ID 1)
  - Network connections (Event ID 3)
Validated Sysmon locally using wevtutil before forwarding
Confirmed ingestion into Wazuh dashboard

Internal Application Architecture (Production Simulation)
Application Server (Blue/Green Deployment + Reverse Proxy)

Built an Ubuntu application server hosting a Flask web application
Implemented a strict /health endpoint that validates database connectivity
Deployed the application using blue/green systemd services:
  - internal-app-blue (port 5000)
  - internal-app-green (port 5001)
Configured Nginx reverse proxy to provide a stable front-door endpoint on port 80
Implemented Nginx upstream routing to switch production traffic between blue and green backends without client impact
Created a deployment/rollback script (switch-app.sh) that:
  - Performs health validation before switching traffic
  - Updates the active upstream
  - Reloads Nginx safely
  - Logs all deployment actions
Added a Wazuh agent to ingest Nginx logs for operational visibility

![internalappbrowser](Images/InternalAppRunning.png)

Nginx Logs
![nginxlogs](Images/deployments.log.png)

Database Tier Deployment (PostgreSQL)
Built a dedicated Ubuntu database VM to introduce a backend dependency
Installed and configured PostgreSQL as the application datastore
Enabled secure remote connections from the application server
Updated listen_addresses and pg_hba.conf to support authenticated internal access
Verified connectivity from the application server
Purpose: introduce realistic service dependency monitoring and failure scenarios.

Automation and Reliability Engineering
Service Recovery Automation (Watchdog)

Developed a watchdog script to monitor critical services (Nginx + application services)
Automatically restarts services if they become inactive
Scheduled execution via cron
Logged restart activity to /var/log/internal-app-watchdog.log
Configured Wazuh to ingest watchdog logs for visibility into service outages and recovery actions
Tested by intentionally stopping services and validating automatic recovery
![watchdogLog](Images/watchdogLogs.png)
  
Service Health Monitoring
Created a custom health monitoring script that calls the application /health endpoint
Scheduled checks via cron every minute
Logged:
  - Timestamp
  - HTTP status code
  - Overall service and dependency health (OK / FAIL)
Designed monitoring to detect degradation even when the application process is still running

Escalation Triage Automation
Built a triage bundle script that collects diagnostic data when health checks fail
Captures:
  - Service status snapshots
  - Active Nginx upstream configuration
  - Recent journal logs
  - Network connectivity checks
  - Stores bundles under /var/log/triage-bundles/ to simulate L2 escalation workflows
Added cooldown logic to prevent alert/triage noise

Current Attack Simulation
- Used Kali Linux to simulate sttacker activity
  - Port scans(Nmap)
  - Connection attempts(SMB, RDP, HTTP) with incorrect credentials to simulate a password spray      attack
- Generated detectable network activity through real TCP connections

Then confirmed that events from endpoints were properly being logged and aggregated at the Wazuh Dashboard
Here is an image from the dashboard looking at the security events for the Windows DC endpoint. You can see all of the different invalid login attempts that I simulated from the Kali Linux machine.
![dashboard view](Images/WazuhDashboardAuthentication.png)

Incident Simmulation - Dependency Failure(Database Outage)
- This icident was a controlled scenario that I ran just to ensure that my database and internal-app monitoring were working as I intended them to.
- Scenario: The Database VM has been powered off
- Observed Behavior:
    - The /health endpoint now returned HTTP 500 errors
    - My monitoring script began logging FAIL events every minute via cron job
    - Nginx 500 alerts triggered
- Resolution
  - Database VM turned back on
  - Healthmonitoring checks went back to returning OK
  - Monitoring confirmed the recovery

Next Planned Enhancements
- Add custom Wazuh rules to detect common attack vectors
- Noise reduction and alert tuning
- Integration of Suricata for network-based IDS detection
- MITRE ATT&CK mapping for detected activity
- Automation of common response techniques and workflows
