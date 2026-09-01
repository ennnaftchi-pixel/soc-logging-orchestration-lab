# MISP & Cortex — Threat Intelligence & Automated Analysis

**SOC Lab | Securinets FST Summer Camp 2026**

> A complete, production-ready deployment of MISP and Cortex integrated with Wazuh, TheHive, and secured via Tailscale VPN mesh.

---

## 📋 Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Key Accomplishments](#key-accomplishments)
4. [Quick Start](#quick-start)
5. [Detailed Configuration](#detailed-configuration)
6. [Integration with SOC Stack](#integration-with-soc-stack)
7. [Tailscale Networking](#tailscale-networking)
8. [Security Considerations](#security-considerations)
9. [Troubleshooting](#troubleshooting)
10. [References](#references)

---

## Overview

This module provides the **threat intelligence and automated analysis layer** for a complete Security Operations Center. It consists of two core components:

- **MISP** (Malware Information Sharing Platform) — centralizes and shares indicators of compromise (IOCs) from threat feeds and custom events
- **Cortex** — automatically enriches and analyzes observables using 275+ analyzers (VirusTotal, Shodan, AbuseIPDB, MISP lookup, etc.)

### What This Module Does

```
Threat Intelligence Feeds (CIRCL, Abuse.ch, etc.)
          ↓
        MISP
          ↓
    ┌─────┴─────┐
    ↓           ↓
 Wazuh      TheHive
 (SIEM)     (IR Platform)
    ↓           ↓
    └─────┬─────┘
          ↓
       Cortex (Analyzers)
          ↓
  Enriched Intelligence
```

---

## Architecture

### Deployment Structure

```
soc-logging-orchestration-lab/
├── src/
│   ├── misp/
│   │   ├── docker-compose.yml      # MISP stack definition
│   │   ├── docker-bake.hcl         # Docker build config
│   │   └── README.md               # MISP-specific docs
│   │
│   ├── cortex/
│   │   ├── docker-compose.yml      # Cortex + Elasticsearch
│   │   ├── cortex-compose.yml      # Alternative deployment
│   │   ├── application.conf        # Cortex configuration (template)
│   │   └── Cortex-Analyzers/       # 275 analyzers (submodule)
│   │
│   └── docker-compose.yml          # Master orchestration
│
├── docs/
│   ├── misp/                       # MISP deployment guides
│   └── cortex/                     # Cortex setup docs
│
├── assets/
│   └── screenshots/                # Visual documentation
│       ├── misp/
│       └── cortex/
│
└── .gitignore                      # Secrets protection
```

### Container Architecture

#### MISP Stack (5 containers)

| Container | Role | Port | Base Image |
|-----------|------|------|------------|
| `misp-core` | Main web app (Nginx + PHP-FPM) | 80, 443 | coolacid/misp-docker |
| `misp-db` | MariaDB 10.11 persistence | 3306 | mariadb:10.11 |
| `misp-redis` | Cache & job queue (Valkey 7.2) | 6379 | valkey:7.2 |
| `misp-modules` | Enrichment services | 6666 | misp/misp-modules |
| `misp-mail` | SMTP for notifications | 25 | mailhog:latest |

#### Cortex Stack (2 containers)

| Container | Role | Port | Base Image |
|-----------|------|------|------------|
| `cortex` | Analysis engine + UI | 9001 | thehiveproject/cortex:3.1.8 |
| `cortex-es` | Elasticsearch 7.17.9 | 9200 | docker.elastic.co/elasticsearch/elasticsearch:7.17.9 |

---

## Key Accomplishments

### Fully Operational Deployments

- **MISP 2.5.44** deployed with automatic database initialization, GPG key generation, and taxonomy synchronization (180 taxonomies, 424 object templates, MITRE galaxies)
- **Cortex 3.1.8** with 275 active analyzers and production-ready job queue
- Both platforms accessible via web UI with proper SSL/TLS certificates

### Threat Intelligence Integration

- **Public feeds configured**: CIRCL, Abuse.ch, MalwareBazaar, Feodo
- **Analyzer ecosystem**: VirusTotal (file/hash/IP/domain), Shodan (host reconnaissance), AbuseIPDB (abuse scoring)
- **Tested event lifecycle**: Event creation → publication → MISP API access → Cortex enrichment

### Cortex-MISP Bidirectional Integration

- MISP lookup analyzer configured in Cortex
- Cortex can directly query MISP events and attributes
- Analyzer results flow back into MISP for correlation

### Wazuh Integration (Ready)

- MISP API key generated and configured
- Wazuh module configuration parameters prepared
- IOC matching workflow validated (tested on Event ID 1662 with 4 sample indicators)

### TheHive Integration (Ready)

- Cortex organization (`SOC-Lab`) and orgAdmin account created
- API authentication validated
- Observable enrichment pipeline tested with Shodan analyzer

### Tailscale VPN Mesh Networking

- All containers accessible via Tailscale private IPs
- MagicDNS enabled for `.duckdns.com` resolution
- End-to-end encryption (WireGuard protocol)
- No port forwarding or public IP exposure required

### Security & Secrets Management

- `.gitignore` configured to exclude:
  - MISP `.env` files and runtime secrets
  - Cortex application.conf with secret keys
  - Docker build artifacts, backups, and SSL keys
- Secrets templated with environment variable references
- No credentials committed to Git repository

---

## Quick Start

### Prerequisites

- Docker Engine 29.6.2+
- Docker Compose v5.3.1+
- 8 GB RAM, 48 GB disk (minimum for MISP + Cortex)
- Ubuntu 22.04 LTS or Debian 12

### 1. Clone & Navigate

```bash
cd ~/soc-logging-orchestration-lab
git switch feature/misp-cortex
```

### 2. Configure Environment

**For MISP**, create `.env` in `src/misp/`:

```bash
cd src/misp/
cat > .env <<'EOF'
BASE_URL=https://192.168.1.207
ADMIN_EMAIL=admin@misp.local
ADMIN_ORG=SOC-Lab
ADMIN_PASSWORD=changeme-to-secure
MYSQL_PASSWORD=secure-db-password
MYSQL_ROOT_PASSWORD=secure-root-password
REDIS_PASSWORD=secure-redis-password
ENABLE_REDIS_EMPTY_PASSWORD=false
EOF
```

**For Cortex**, create `application.conf` in `src/cortex/`:

```bash
cd ../cortex/
cat > application.conf <<'EOF'
play.http.secret.key="$(head -c 32 /dev/urandom | base64)"
search.uri="http://cortex-es:9200"
job.directory="/tmp/cortex-jobs"
analyzer {
  urls = [
    "https://download.thehive-project.org/analyzers.json"
    "/opt/Cortex-Analyzers/analyzers"
  ]
}
EOF
```

### 3. Start Containers

```bash
# From soc-logging-orchestration-lab root
docker-compose -f src/misp/docker-compose.yml up -d
docker-compose -f src/cortex/docker-compose.yml up -d
```

### 4. Access Web UIs

| Service | URL | Credentials |
|---------|-----|-------------|
| MISP | `https://192.168.1.207` | admin / (from .env) |
| Cortex | `http://192.168.1.207:9001` | admin / changeme |

---

## Detailed Configuration

### MISP Configuration

#### Post-Installation Setup

1. **Change admin password** (Administration → User Management)
2. **Generate API key** (Administration → List Auth Keys)
3. **Activate threat feeds** (Sync Actions → Feeds):
   - CIRCL
   - Abuse.ch MalwareBazaar
   - Abuse.ch Feodo Tracker

#### Sample Test Event

**Event ID 1662** (Published 2026-07-25):

```
MD5 Hash:     44d88612fea8a8f36de82e1278abb02f
URL:          http://phishing-test-example.com/login
Domain:       malicious-test-domain.com
IP Address:   185.220.101.45
```

---

### Cortex Configuration

#### Production-Ready Analyzers

| Analyzer | Function | Status |
|----------|----------|--------|
| VirusTotal_GetReport_3.1 | File/hash/IP/domain reputation | ✅ Active |
| AbuseIPDB_2.0 | IP abuse scoring | ✅ Active |
| Shodan_Host_1.0 | Host reconnaissance | ✅ Active |

#### Organization Setup

- **Organization**: `SOC-Lab`
- **Admin Account**: `socadmin` (orgAdmin role)
- **Tested**: Shodan_Host on 8.8.8.8 returned success

---

## Integration with SOC Stack

### MISP ↔ Wazuh Integration

**Status**: Ready for activation

**Configuration on Wazuh**:

```xml
<integration>
    <name>misp</name>
    <hook_url>https://192.168.1.207</hook_url>
    <api_key>[GENERATED_MISP_KEY]</api_key>
    <alert_format>json</alert_format>
</integration>
```

---

### Cortex ↔ MISP Integration

**Status**: Operational ✅

**Configuration**:

```
MISP URL: http://192.168.1.207
MISP Key: [Generated API key]
SSL Verify: false
```

---

## Tailscale Networking

### Setup

#### 1. Install Tailscale

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

#### 2. Add Tailscale to Docker Compose

```yaml
services:
  tailscale-misp:
    image: tailscale/tailscale:latest
    container_name: tailscale-misp
    environment:
      - TS_AUTHKEY=${TAILSCALE_AUTHKEY}
      - TS_HOSTNAME=misp-soc
    volumes:
      - misp-ts:/var/lib/tailscale
    networks:
      - soc_net
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
```

#### 3. Access via Tailscale

```
MISP:   https://100.x.x.x  or https://misp-soc.duckdns.com
Cortex: http://100.x.x.x:9001
```

---

## Security Considerations

### Secrets Management

#### DO NOT commit:
- `.env` files with passwords
- API keys
- Private keys (TLS certificates)
- Database credentials

#### DO manage safely:
- Store in `.env` (added to `.gitignore`)
- Use environment variables in docker-compose.yml
- Rotate keys regularly
- Use strong passwords (20+ characters)

### Git Security

```bash
# Verify no secrets committed
git diff --cached | grep -i "password\|secret\|key" | wc -l
# Should return 0
```

---

## Troubleshooting

### MISP Won't Start

```bash
docker logs misp-core | grep -i error
# Check for: "MySQL is not up yet", "Database error"

# Wait for database
docker logs misp-db | grep "ready for connections"
```

### Cortex Analyzers Not Loading

```bash
docker logs cortex | grep -i "analyzer"

# Install missing Python dependencies
docker exec -it cortex pip3 install cortexutils requests shodan
docker restart cortex
```

### MISP ↔ Cortex Communication Failed

```bash
# Test from MISP container
docker exec misp-core curl -v http://cortex:9001/api/status

# Verify network
docker network inspect soc_net
```

---

## References

### Official Documentation

- **MISP**: https://misp.readthedocs.io/
- **Cortex**: https://cortex.readthedocs.io/
- **TheHive**: https://docs.strangebee.com/thehive/
- **Wazuh**: https://documentation.wazuh.com/

### Repositories

- **MISP Docker**: https://github.com/MISP/misp-docker
- **Cortex Analyzers**: https://github.com/TheHive-Project/Cortex-Analyzers
- **SOC Lab**: https://github.com/ennnaftchi-pixel/soc-logging-orchestration-lab

---

## Deployment Checklist

- [ ] `.env` files created (not in Git)
- [ ] `application.conf` with secret key (not in Git)
- [ ] Docker containers started
- [ ] MISP database initialized
- [ ] MISP web UI accessible
- [ ] Cortex web UI accessible
- [ ] Cortex analyzers loaded
- [ ] Cortex-MISP integration configured
- [ ] Wazuh module parameters ready
- [ ] TheHive parameters ready
- [ ] Tailscale devices visible
- [ ] No secrets in Git

---

## Author & Attribution

**Name**: Mohamed Nafti (Ennnaftchi)  
**Role**: MISP & Cortex Specialist  
**Project**: SOC Lab - Securinets FST Summer Camp 2026  
**Repository**: https://github.com/ennnaftchi-pixel/soc-logging-orchestration-lab  
**Branch**: `feature/misp-cortex`

---

**Last Updated**: August 31, 2026  
**Status**: Production Ready
