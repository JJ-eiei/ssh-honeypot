#  SSH Honeypot Lab

Monitor and analyze real-world SSH attacks using Cowrie Honeypot on GCP.

## Stack
- **Cowrie** :  SSH Honeypot
- **Grafana + Loki** :  Dashboard & Log Aggregation  
- **Promtail**  : Log Shipper
- **Telegram Bot** :  Real-time Alert
- **GCP e2-micro** : Cloud Infrastructure

## Architecture
Internet → GCP Firewall (port 2222) → Cowrie Honeypot
↓
JSON Log File
↓
Promtail → Loki → Grafana
↓
Telegram Alert

## Key Findings
- Detected **10+ unique attacker IPs** within days of deployment
- Bots specifically targeted **IoT default credentials** (orangepi/orangepi)
- Observed advanced bot behavior: **TCP tunneling via Cloudflare**
- Most common passwords attempted: admin, orangepi, P, root

## Setup Guide
See [docs/setup.md](docs/setup.md) for full installation steps.

## Write-up
See [docs/writeup.md](docs/writeup.md) for full analysis.
