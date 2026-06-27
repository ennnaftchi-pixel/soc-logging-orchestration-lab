# Component Architecture Notes

A brief technical guide to how data flows and routes across the different services within the lab architecture.

## Telemetry Flow Engine
1.  **Endpoints (Target Hosts):** Run background logging agents (like Wazuh agents) to monitor file integrity, system logs (`auth.log`), and access events.
2.  **Aggregation (Wazuh SIEM):** Receives the encrypted data streams on defined listening ports, processes the raw logs against rule sets, and highlights security alerts.
3.  **Forwarding (Webhooks/APIs):** Alerts are formatted into JSON data payloads and pushed to management components via REST API endpoints.

## Intelligence & Incident Management
* **MISP:** Populated with threat data (IP addresses, malicious domains) accessed via an API token.
* **TheHive & Cortex:** When an alert is received, TheHive opens a case tracker while Cortex executes scripts to cross-check observables against MISP data via automated REST calls.
