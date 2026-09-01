# MISP — Malware Information Sharing Platform

**Part of**: SOC Lab - Securinets FST Summer Camp 2026  
**Status**: Fully Operational (v2.5.44)

---

## Overview

MISP is the **threat intelligence and IOC management backbone** of the SOC.

### Key Features

- **Centralized IOC repository** — Store and organize indicators of compromise
- **Threat feed integration** — CIRCL, Abuse.ch, MalwareBazaar, Feodo, etc.
- **Structured data model** — Events, attributes, objects, taxonomies, galaxies
- **Export capabilities** — STIX 2.0, OpenIOC, Yara, Sigma, CSV
- **API-first design** — REST API for integration with Wazuh, TheHive, Cortex
- **Multi-organization sharing** — Role-based access control (RBAC)

---

## Architecture

### Deployment Model

```
src/misp/
├── docker-compose.yml        # Complete stack definition
├── docker-bake.hcl          # Docker buildx configuration
└── README.md                # This file

Docker Compose creates:
├── misp-core        (Nginx + PHP-FPM) — Web UI + REST API
├── misp-db          (MariaDB 10.11)   — Persistent storage
├── misp-redis       (Valkey 7.2)      — Cache & job queue
├── misp-modules     (Enrichment)      — Whois, VirusTotal, PassiveDNS, etc.
└── misp-mail        (Mailhog)         — SMTP for notifications
```

### Ports & Services

| Service | Port | Protocol | Purpose |
|---------|------|----------|---------|
| MISP Web UI | 80, 443 | HTTP/HTTPS | Web interface |
| MySQL | 3306 | TCP | Database (internal) |
| Redis | 6379 | TCP | Cache (internal) |
| MISP Modules | 6666 | TCP | Enrichment services |
| Mailhog UI | 1025 | SMTP | Email notifications |

---

## Installation

### Prerequisites

- Docker 29.6.2+
- Docker Compose v5.3.1+
- 4 GB RAM minimum (8 GB recommended)
- 20 GB disk space minimum

### Setup Steps

```bash
cd ~/soc-logging-orchestration-lab/src/misp

# Create .env file
cat > .env <<'EOF'
BASE_URL=https://192.168.1.207
ADMIN_EMAIL=admin@misp.local
ADMIN_ORG=SOC-Lab
ADMIN_PASSWORD=changeme-to-secure-password
MYSQL_USER=misp
MYSQL_PASSWORD=secure-db-password
MYSQL_ROOT_PASSWORD=secure-root-password
REDIS_PASSWORD=secure-redis-password
ENABLE_REDIS_EMPTY_PASSWORD=false
GPG_PASSPHRASE=secure-gpg-passphrase
EOF

chmod 600 .env

# Start services
docker-compose up -d

# Monitor initialization (5-10 minutes)
docker logs -f misp-core
# Wait for: "MySQL is up - executing command"

# Verify deployment
docker ps | grep misp
```

---

## Initial Configuration

### 1. Change Admin Password

1. Go to `https://192.168.1.207`
2. Login with default credentials
3. **Administration** → **User Management**
4. Change password to something strong (20+ characters)

### 2. Generate API Key

1. **Administration** → **List Auth Keys**
2. Click **+ Add auth key**
3. Set Role: Org Admin
4. Copy the generated key (save securely)

### 3. Activate Threat Feeds

1. **Sync Actions** → **Feeds**
2. Enable recommended feeds:

| Feed | Provider | Recommendation |
|------|----------|-----------------|
| CIRCL Hash Lookup | CIRCL | ✅ Enable |
| CIRCL Passive DNS | CIRCL | ✅ Enable |
| Abuse.ch MalwareBazaar | Abuse.ch | ✅ Enable |
| Abuse.ch Feodo Tracker | Abuse.ch | ✅ Enable |

---

## Data Model

### Events

A collection of related indicators (threat, campaign, incident).

**Example**: "Emotet Campaign Q3 2026"
- Event ID: 1662
- Date: 2026-07-25
- Status: Published
- Threat Level: High

### Attributes

Individual indicators within an event.

**Types**: md5, sha-256, domain, ip-dst, url, email-src, vulnerability, comment

### Objects

Complex structures combining multiple attributes.

**Example**: File object with filename, hash, and ssdeep

### Taxonomies

Machine-readable classification systems (180 built-in taxonomies).

**Examples**: tlp (Traffic Light Protocol), pap, mitre-attack-pattern, malware-classification

---

## Tested Event

**Event ID 1662** (Published 2026-07-25):

| Attribute | Type | Value |
|-----------|------|-------|
| 1 | md5 | `44d88612fea8a8f36de82e1278abb02f` |
| 2 | url | `http://phishing-test-example.com/login` |
| 3 | domain | `malicious-test-domain.com` |
| 4 | ip-dst | `185.220.101.45` |

**Validation**:
- Event creation
- Attribute addition
- Event publishing
- API accessibility
- Observable matching for Cortex

---

## API Reference

### Authentication

```bash
Authorization: Bearer YOUR_API_KEY
```

### Get Event by ID

```bash
curl -H "Authorization: Bearer KEY" \
  https://192.168.1.207/api/events/1662 | jq .
```

### Search Attributes

```bash
curl -H "Authorization: Bearer KEY" \
  "https://192.168.1.207/api/attributes/search?value=44d88612fea8a8f36de82e1278abb02f"
```

### Create Event

```bash
curl -X POST -H "Authorization: Bearer KEY" \
  -d '{
    "Event": {
      "info": "Phishing Campaign 2026",
      "threat_level_id": 3,
      "analysis": 0,
      "distribution": 1
    }
  }' https://192.168.1.207/api/events
```

### Publish Event

```bash
curl -X POST -H "Authorization: Bearer KEY" \
  https://192.168.1.207/api/events/1662/publish
```

---

## Integration Points

### Wazuh Integration

**Purpose**: IOC correlation in Wazuh SIEM

**Configuration (Wazuh side)**:

```xml
<integration>
    <name>misp</name>
    <hook_url>https://192.168.1.207</hook_url>
    <api_key>[YOUR_MISP_API_KEY]</api_key>
    <alert_format>json</alert_format>
</integration>
```

**Workflow**:
```
Log Entry → Extract Observable → Query MISP API → 
Found in Event 1662 → Escalate to Critical → Create TheHive Case
```

### TheHive Integration

**Purpose**: Import MISP events for case analysis

**Configuration (TheHive side)**:

```json
{
  "misp": {
    "enabled": true,
    "url": "https://192.168.1.207",
    "apiKey": "[YOUR_MISP_API_KEY]",
    "verifyCert": false
  }
}
```

### Cortex Integration

**Purpose**: Historical IOC lookup analyzer

**Analyzer**: `MISP_Search_3.0`

**Workflow**:
```
Observable (IP) → Cortex MISP_Search → Query MISP API → 
Found in Event 1662 → Return Result → Score: Critical
```

---

## Threat Feeds

### Automatic Feed Sync

MISP syncs feeds automatically every 6 hours.

**To manually sync**:

```bash
docker exec misp-core php /var/www/html/app/Console/cake CakePHP.console \
  Shell.TestShell.feed
```

### Popular Feeds

| Feed | Type | Update Frequency |
|------|------|-----------------|
| CIRCL Hash Lookup | Hash | Daily |
| Abuse.ch MalwareBazaar | Malware | Real-time |
| Feodo Tracker | Botnet | Real-time |
| URLhaus | URLs | Real-time |
| YARA Rules | YARA | Weekly |

### Adding Custom Feeds

1. **Sync Actions** → **Feeds** → **+ Add Feed**
2. Configure URL, format, distribution
3. Enable and test

---

## Maintenance

### Database Backup

```bash
# Backup
docker exec misp-db mysqldump -u misp -p${MYSQL_PASSWORD} misp > backup.sql

# Restore
docker exec -i misp-db mysql -u misp -p${MYSQL_PASSWORD} misp < backup.sql
```

### Log Monitoring

```bash
# Follow MISP logs
docker logs -f misp-core

# Follow database logs
docker logs -f misp-db

# Monitor Redis cache
docker exec misp-redis redis-cli MONITOR
```

### Performance Optimization

```bash
# Database optimization
docker exec -it misp-db mysql -u misp -p${MYSQL_PASSWORD} misp

> OPTIMIZE TABLE events;
> OPTIMIZE TABLE attributes;
> ANALYZE TABLE events;
```

---

## Troubleshooting

### MISP Won't Start

```bash
docker logs misp-core | grep -i error

# Wait for database
docker logs misp-db | grep "ready for connections"

# Restart
docker-compose restart misp-db misp-core
```

### "Unable to connect to database"

```bash
# Check MariaDB status
docker-compose ps misp-db

# Restart database first
docker-compose restart misp-db

# Then restart MISP
docker-compose restart misp-core
```

### API Key Not Working (401 Unauthorized)

```bash
# List all API keys
docker exec misp-core php /var/www/html/app/Console/cake user list_authkeys

# Generate new key via UI or API
curl -s -H "Authorization: Bearer OLD_KEY" \
  -X POST -d '{"role_id": 3, "named": "cortex"}' \
  https://192.168.1.207/api/auth_keys
```

### Out of Disk Space

```bash
# Check sizes
du -sh ./misp-docker/*
docker exec misp-db du -sh /var/lib/mysql

# Prune old events
docker exec misp-db mysql -u misp -p${MYSQL_PASSWORD} -e \
  "DELETE FROM events WHERE timestamp < UNIX_TIMESTAMP(DATE_SUB(NOW(), INTERVAL 1 YEAR));"
```

---

## Security Best Practices

### Access Control

- Use strong, unique API keys (25+ characters)
- Rotate API keys quarterly
- Set expiration dates on API keys
- Use RBAC strictly
- Restrict distribution levels

### Data Protection

- Enable TLS/HTTPS
- Encrypt database backups
- Restrict API key permissions
- Monitor API key usage

### Feed Security

- Use HTTPS for feed sources
- Verify feed signatures (GPG/PGP)
- Monitor for anomalies
- Disable untrusted feeds

---

## Deployment Checklist

- [ ] `.env` file created and secured (chmod 600)
- [ ] All 5 containers running
- [ ] Database initialized successfully
- [ ] Web UI accessible at BASE_URL
- [ ] Admin password changed
- [ ] API key generated and tested
- [ ] At least 2 threat feeds enabled
- [ ] No secrets in Git repository
- [ ] Backup strategy documented
- [ ] Monitoring configured

---

## References

- **MISP Documentation**: https://misp.readthedocs.io/
- **MISP Docker**: https://github.com/MISP/misp-docker
- **MISP Taxonomies**: https://github.com/MISP/misp-taxonomies
- **STIX 2.0 Spec**: https://oasis-open.github.io/cti-documentation/

---

**Last Updated**: August 31, 2026  
**MISP Version**: 2.5.44  
**Status**: Production Ready
