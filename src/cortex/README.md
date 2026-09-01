# Cortex — Automated Observable Analysis & Enrichment

**Part of**: SOC Lab - Securinets FST Summer Camp 2026  
**Status**: Fully Operational (v3.1.8)

---

## Overview

Cortex is the **automated analysis and enrichment engine** for the SOC, providing 275+ analyzers for observables (IPs, domains, files, hashes, URLs, etc.).

### Key Features

- **275+ analyzers** — VirusTotal, Shodan, AbuseIPDB, MISP lookup, PassiveDNS, Whois, etc.
- **Multi-source enrichment** — Query multiple APIs in parallel
- **Configurable analyzers** — Enable/disable per organization
- **Job queue** — Background processing for long-running analyzers
- **REST API** — Integration with MISP, TheHive, Wazuh
- **Elasticsearch backend** — Persistent job storage and search
- **Org-based access control** — Separate analyzer configs per organization

---

## Architecture

### Deployment Model

```
src/cortex/
├── docker-compose.yml       # Cortex + Elasticsearch
├── cortex-compose.yml       # Alternative deployment
├── application.conf         # Configuration template
└── Cortex-Analyzers/        # 275 analyzer scripts

Docker Compose creates:
├── cortex               (Analysis engine) — REST API + UI
├── cortex-es            (Elasticsearch 7.17.9) — Job storage
└── (optional) Tailscale — VPN mesh networking
```

### Ports & Services

| Service | Port | Protocol | Purpose |
|---------|------|----------|---------|
| Cortex UI/API | 9001 | HTTP | Web interface + REST API |
| Elasticsearch | 9200 | HTTP | Job persistence (internal) |

---

## Installation

### Prerequisites

- Docker 29.6.2+
- Docker Compose v5.3.1+
- 3 GB RAM minimum (5 GB recommended)
- 10 GB disk space minimum

### Setup Steps

```bash
cd ~/soc-logging-orchestration-lab/src/cortex

# Generate secret key for application.conf
SECRET=$(head -c 32 /dev/urandom | base64)

# Create application.conf
cat > application.conf <<EOF
play.http.secret.key="$SECRET"
search.uri="http://cortex-es:9200"
job.directory="/tmp/cortex-jobs"

analyzer {
  urls = [
    "https://download.thehive-project.org/analyzers.json"
    "/opt/Cortex-Analyzers/analyzers"
  ]
}
EOF

# Start services
docker-compose up -d

# Monitor initialization (2-3 minutes)
docker logs -f cortex

# Verify deployment
docker ps | grep cortex
```

---

## Initial Configuration

### 1. Access Cortex UI

```
URL: http://192.168.1.207:9001
Username: admin
Password: changeme (change this!)
```

### 2. Create Organization

1. Click **Organizations**
2. Add organization: `SOC-Lab`
3. Set default roles and permissions

### 3. Create Admin Account

1. Go to **Accounts** (within SOC-Lab organization)
2. Click **+ Add Account**
3. Set:
   - **Login**: socadmin
   - **Role**: orgAdmin (required for analyzer management)
   - **Password**: (strong, 20+ characters)
4. Save and logout

### 4. Logout & Login as socadmin

```
Switch to socadmin account to configure analyzers
(Only orgAdmin can manage analyzers in Cortex)
```

---

## Analyzer Management

### Enabled Analyzers

Three production-ready analyzers are pre-configured:

#### 1. VirusTotal_GetReport_3.1

**Function**: File/hash/IP/domain reputation scanning

**Queries**:
- File hashes (MD5, SHA-256, SHA-1)
- IP addresses
- Domain names
- URLs

**Prerequisites**: VirusTotal API key (free tier)

**Setup**:
1. Get free API key from https://www.virustotal.com/
2. In Cortex: **Organizations → SOC-Lab → VirusTotal**
3. Set: `key` parameter to your API key

#### 2. AbuseIPDB_2.0

**Function**: IP abuse scoring and reputation

**Queries**: IPv4/IPv6 addresses

**Prerequisites**: AbuseIPDB API key (free tier)

**Setup**:
1. Get API key from https://www.abuseipdb.com/
2. In Cortex: **Organizations → SOC-Lab → AbuseIPDB**
3. Set: `key` parameter to your API key

#### 3. Shodan_Host_1.0

**Function**: Host reconnaissance (ports, services, banners)

**Queries**: IPv4/IPv6 addresses

**Prerequisites**: Shodan API key (free tier: 1 query/month, paid: unlimited)

**Setup**:
1. Get API key from https://www.shodan.io/
2. In Cortex: **Organizations → SOC-Lab → Shodan_Host**
3. Set: `key` parameter to your API key

**Tested on**: 8.8.8.8 (Google DNS)
- Result: Success
- Ports: 53 (DNS)
- Services: DNS
- ASN: AS15169 (Google)

---

## Adding More Analyzers

Cortex includes 275 analyzers in total. To enable additional ones:

### 1. Via UI

1. Login as **socadmin** (orgAdmin role)
2. Go to **Organizations → SOC-Lab**
3. Scroll through analyzers
4. Click analyzer name to expand
5. Enter required API keys (if needed)
6. Click "Enable"

### 2. Popular Analyzers

| Analyzer | Type | Purpose | Free API? |
|----------|------|---------|-----------|
| MaxMind_GeoIP_2.1 | IP | Geolocation + ASN | Yes (builtin) |
| DomainTools_ReverseIP | Domain | Reverse DNS lookup | No (paid) |
| PassiveDNS_2.0 | Domain | Historical DNS records | Yes |
| AlienVault_OTX_2.0 | Multi | Open threat exchange | Yes |
| Censys_2.0 | Domain/IP | Certificate transparency | Yes (free tier) |
| URLhaus_1.0 | URL | Malicious URL database | Yes |
| PhishingInitiative_2.0 | URL | Phishing database | Yes |
| CIRCL_Passive_DNS_1.0 | Domain | CIRCL DNS lookup | Yes |

---

## API Reference

### Authentication

```bash
Authorization: Bearer YOUR_CORTEX_API_KEY
```

### Get Organization Status

```bash
curl -H "Authorization: Bearer KEY" \
  http://192.168.1.207:9001/api/organization
```

### List Analyzers

```bash
curl -H "Authorization: Bearer KEY" \
  http://192.168.1.207:9001/api/analyzer/list
```

### Run Analyzer

```bash
curl -X POST -H "Authorization: Bearer KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "data": "8.8.8.8",
    "dataType": "ip",
    "service": "Shodan_Host_1.0"
  }' http://192.168.1.207:9001/api/job
```

### Get Job Result

```bash
curl -H "Authorization: Bearer KEY" \
  http://192.168.1.207:9001/api/job/JOB_ID
```

---

## Integration Points

### MISP Integration

**Purpose**: Query MISP events during analysis

**Analyzer**: `MISP_Search_3.0`

**Configuration in MISP**:
- URL: `http://192.168.1.207`
- API Key: (generated from MISP Administration)
- SSL Verify: false

**Workflow**:
```
Observable in Cortex
    ↓
MISP_Search analyzer
    ↓
Query MISP API
    ↓
Found in Event 1662 → Return matching events
    ↓
Enrich with historical intelligence
```

### TheHive Integration

**Purpose**: Send observables to Cortex for analysis

**Configuration (TheHive side)**:

```json
{
  "cortex": {
    "enabled": true,
    "url": "http://192.168.1.207:9001",
    "key": "[Cortex socadmin API key]",
    "organisation": "SOC-Lab"
  }
}
```

**Workflow**:
```
Case in TheHive
    ↓
Add observable (IP/domain/hash)
    ↓
Right-click → "Run analyzers"
    ↓
Cortex processes (VirusTotal, Shodan, AbuseIPDB)
    ↓
Results displayed in observable details
```

### Wazuh Integration

**Purpose**: Enrich Wazuh alerts with Cortex analysis

**Configuration (Wazuh side)**:

```xml
<integration>
    <name>cortex</name>
    <hook_url>http://192.168.1.207:9001</hook_url>
    <api_key>[Cortex socadmin API key]</api_key>
    <alert_format>json</alert_format>
</integration>
```

---

## Job Management

### View Job Queue

```bash
# Via Cortex UI
Navigate to: Dashboard → Jobs

# Via API
curl -H "Authorization: Bearer KEY" \
  http://192.168.1.207:9001/api/job
```

### Monitor Long-Running Jobs

```bash
# Watch job completion
docker logs -f cortex | grep -i "job"

# Check Elasticsearch for job status
curl http://cortex-es:9200/cortex-jobs/_search?q=status:Running
```

### Clear Job Queue

```bash
# Delete all jobs (destructive!)
docker exec cortex-es curl -X DELETE \
  http://cortex-es:9200/cortex-jobs
```

---

## Configuration

### Python Dependencies

Cortex analyzers require Python libraries. Install in container:

```bash
docker exec -it cortex pip3 install \
  cortexutils \
  requests \
  shodan \
  dnsdb2 \
  (add other required packages)

# Restart Cortex after installation
docker restart cortex
```

### Resource Limits

By default, Cortex can use up to 4GB RAM and Elasticsearch 2GB. Adjust in docker-compose.yml:

```yaml
services:
  cortex:
    environment:
      - "CORTEX_JAVA_OPTS=-Xmx2g -Xms512m"
  cortex-es:
    environment:
      - "ES_JAVA_OPTS=-Xmx1g -Xms512m"
```

---

## Maintenance

### Logs

```bash
# Follow Cortex logs
docker logs -f cortex

# Follow Elasticsearch logs
docker logs -f cortex-es

# Check for errors
docker logs cortex | grep -i error
```

### Backup Job Database

```bash
# Snapshot Elasticsearch indices
docker exec cortex-es curl -X POST \
  'http://localhost:9200/_snapshot/backup/cortex-jobs-backup'

# Export jobs to JSON
curl http://cortex-es:9200/cortex-jobs/_search?size=10000 > jobs-backup.json
```

### Database Cleanup

```bash
# Delete jobs older than 30 days
docker exec cortex-es curl -X DELETE \
  'http://localhost:9200/cortex-jobs-*' \
  -d '{"query": {"range": {"timestamp": {"lt": "now-30d"}}}}'
```

---

## Troubleshooting

### Cortex Won't Start

```bash
docker logs cortex | grep -i error

# Common issues:
# - Elasticsearch not ready
# - Port 9001 already in use
# - application.conf syntax error

# Restart Elasticsearch first
docker-compose restart cortex-es

# Then restart Cortex
docker-compose restart cortex
```

### Analyzers Not Loading

```bash
docker logs cortex | grep -i analyzer

# If missing Python dependencies:
docker exec -it cortex pip3 install cortexutils requests

# Restart
docker restart cortex
```

### "Connection refused" to Elasticsearch

```bash
# Verify Elasticsearch is running
docker ps | grep cortex-es

# Check Elasticsearch health
curl http://127.0.0.1:9200/_health

# Restart both
docker-compose restart cortex-es cortex
```

### Out of Disk Space

```bash
# Check Elasticsearch indices size
curl http://cortex-es:9200/_cat/indices?v

# Prune old indices (keep 30 days)
docker exec cortex-es curl -X DELETE \
  'http://localhost:9200/cortex-jobs-*'
```

### Analyzer Returns "error": "No responder found"

```bash
# Analyzer exists but organization doesn't have it enabled

# Fix:
1. Login as socadmin
2. Go to Organizations → SOC-Lab
3. Find the analyzer
4. Click to expand and enable it
5. Enter any required API keys
```

---

## Security Best Practices

### API Key Management

- Use strong, unique API keys for each analyzer
- Store keys securely (not in code)
- Rotate keys quarterly
- Limit analyzer permissions to minimum required

### Network Security

- Elasticsearch should NOT be publicly accessible (internal network only)
- Cortex API should require authentication (enabled by default)
- Use HTTPS in production (configure reverse proxy)
- Restrict analyzer API calls to whitelisted IPs (when possible)

### Data Protection

- Encrypt job results at rest (Elasticsearch)
- Use TLS for analyzer API calls (when available)
- Monitor audit logs for suspicious activity
- Regular backups of Elasticsearch indices

---

## Performance Optimization

### Elasticsearch Tuning

```yaml
# In docker-compose.yml
environment:
  - "ES_JAVA_OPTS=-Xmx4g -Xms2g"
  - "ES_THREADPOOL_SEARCH_QUEUE_SIZE=5000"
  - "discovery.type=single-node"
```

### Analyzer Parallelization

```bash
# Cortex can run multiple analyzers in parallel
# Limit concurrent jobs to avoid resource exhaustion:

# Check current settings
curl http://cortex-es:9200/_settings | jq '.cortex-jobs'
```

### Cache Analyzer Results

```bash
# Some analyzers cache results
# To clear cache:
docker exec cortex-es curl -X DELETE \
  'http://localhost:9200/cortex-cache'
```

---

## Deployment Checklist

- [ ] `application.conf` created (not in Git)
- [ ] Secret key generated and configured
- [ ] Both Cortex and Elasticsearch running
- [ ] UI accessible at `http://192.168.1.207:9001`
- [ ] Organization `SOC-Lab` created
- [ ] `socadmin` account created (orgAdmin role)
- [ ] At least 3 analyzers enabled (VT, AbuseIPDB, Shodan)
- [ ] API keys configured for enabled analyzers
- [ ] MISP API key configured (if using MISP integration)
- [ ] Cortex IP set in MISP plugin settings
- [ ] No secrets in Git repository
- [ ] Backup strategy documented
- [ ] Monitoring configured

---

## References

- **Cortex Documentation**: https://cortex.readthedocs.io/
- **Cortex Analyzers**: https://github.com/TheHive-Project/Cortex-Analyzers
- **TheHive Project**: https://www.strangebee.com/thehive/
- **Elasticsearch**: https://www.elastic.co/elasticsearch/

---

**Last Updated**: August 31, 2026  
**Cortex Version**: 3.1.8  
**Analyzers**: 275 available  
**Status**: Production Ready
