# TheHive Setup

## Description
TheHive is used for incident response and case management in our SOC lab.

## Components
- TheHive
- Cassandra
- Elasticsearch

## Integrations
- Wazuh → TheHive
- TheHive → Cortex
- Optional: MISP

## Deployment
TheHive is deployed using Docker Compose.

## Network
The lab machines communicate through Tailscale.
