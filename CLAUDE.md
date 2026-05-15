# CLAUDE.md — SOC Autonome PFE
> **Shared intelligence file for every Claude Code instance across all 4 VMs.**
> This file lives in a small Git repo (`~/soc-shared/`) that ONLY contains shared documentation.
> The actual project files (configs, code, workflows) live in `~/soc-project/` on each VM and never touch Git.

---

## ⚠️ IMPORTANT — Two folders on every VM

Every VM has two distinct working folders:

```
~/soc-shared/        ← Git repo. ONLY CLAUDE.md and docs/.
                       Pulled at session start, pushed after every phase.
                       This is what synchronizes context across all 4 VMs.

~/soc-project/       ← Local-only project files. NEVER committed to Git.
                       Service configs, Flask code, n8n workflow exports,
                       attack scripts, ML models. Stays on the VM that owns it.
```

**Never put project files in `~/soc-shared/`.** That repo is intentionally tiny — it only carries the shared brain (this file plus a few docs).

---

## Session start protocol

When invoked on any machine, do this **first** before any other action:

```bash
hostname              # identify which VM you are on
ip -4 addr show       # confirm ZeroTier IP
cd ~/soc-shared && git pull   # sync latest CLAUDE.md and docs
```

Then:
1. Find your hostname in `## VM profiles` below
2. Read your machine's full section
3. Check `## Current state` — locate active phase and pending sub-steps
4. Check `## Last session notes` — see what the previous instance left for you
5. Proceed only with the next sub-step. Never skip ahead. Never re-do completed work.

After completing any work:
1. Update `## Current state` with new status (in `~/soc-shared/CLAUDE.md`)
2. Add an entry to `## Last session notes` for the next instance
3. If credentials, IPs, or paths changed, update them in the relevant section
4. Commit and push **only CLAUDE.md and docs/** — never the project files:
   ```bash
   cd ~/soc-shared
   git add CLAUDE.md docs/
   git commit -m "phase-N.M: <hostname>: <what was done>"
   git push
   ```

Project files (configs, code) created during the work stay in `~/soc-project/` on the VM. They are local. Other VMs do not need them.

---

## Project identity

| Field | Value |
|---|---|
| Project name | SOC Autonome — Automation des niveaux L1 et L2 |
| Type | Projet de Fin d'Études (PFE) |
| Stack | Elastic 8.x · TheHive 5 · Cortex 4 · MISP · n8n · Suricata · Ollama |
| Detection framework | MITRE ATT&CK |
| LLM backend | Ollama with `llama3.1:8b` (local, free) |
| Total external cost | Zero |
| Shared Git repo | https://github.com/kchaouhabib/soc-shared (CLAUDE.md + docs only) |
| ZeroTier network ID | cf719fd54008e4d1 |

**Core principle:** Every component is open source or free tier. No API keys with billing. No paid subscriptions. The architecture supports drop-in upgrades to paid tiers later but the PFE deployment runs at zero ongoing cost.

---

## Architecture summary

A 4-layer detection SOC with autonomous correlation, AI-driven enrichment, and self-updating threat intelligence:

- **Layer 1 — network signatures:** Suricata + Emerging Threats Open ruleset (~50,000 rules, daily auto-update)
- **Layer 2 — host signatures:** Elastic prebuilt SIEM rules (~1000) + 13 custom rules
- **Layer 3 — threat intelligence matching:** MISP feeds (CIRCL, Abuse.ch, Feodo, OTX, ET) auto-fetched every 6h
- **Layer 4 — behavioral anomaly:** Custom Isolation Forest ML model trained on alert history

All four layers feed into a single correlation pipeline that groups redundant alerts into one TheHive case, summarizes via local LLM, enriches via Cortex, and triggers active response when warranted.

---

## Network topology

```
ZeroTier overlay — 192.168.1.0/24

  PC_A                            PC_B
  ┌─────────────────────┐         ┌─────────────────────────────────────┐
  │ VM_A1  192.168.1.50 │         │ VM_B1  192.168.1.51                 │
  │ soc-core            │◄───────►│ incident-mgmt                       │
  │ Ubuntu Server 22.04 │         │ Ubuntu Server 22.04                 │
  │ SIEM·SOAR·AI        │         │ TheHive·Cortex·MISP                 │
  │                     │         │                                     │
  │ VM_A2  192.168.1.52 │         │ VM_B2  192.168.1.53                 │
  │ kali-attacker       │─attack─►│ victim-lab                          │
  │ Kali Linux          │         │ Ubuntu Server 22.04                 │
  └─────────────────────┘         └─────────────────────────────────────┘
```

---

## Alert pipeline

```
Attack on VM_B2
    │
    ├─► Suricata inspects packet → eve.json
    ├─► Apache logs HTTP request → access.log
    └─► System logs auth failure → auth.log
            │
            ▼
Elastic Agent (B2) ─► Fleet Server (A1:8220) ─► Elasticsearch (A1:9200)
            │
            ▼
Detection layers fire (multiple may match same attack):
  • Suricata-tagged eve event matched by Kibana rule
  • Custom Kibana rule (e.g. SOC-001 SSH brute force)
  • Elastic prebuilt rule
  • MISP threat intel match
  • ML anomaly job
            │
            ▼
Each rule sends webhook → n8n (A1:5678/webhook/elastic-alert)
            │
            ▼
n8n Workflow 1:
    1. POST → Correlation Engine (A1:5002/correlate)
       ├── action: suppress     → drop, end
       ├── action: queue        → daily digest, end
       ├── action: add_to_existing  → PATCH existing TheHive case
       ├── action: escalate_existing → PATCH + raise severity
       └── action: create_new   → continue ↓
    2. POST → ML API (A1:5000/score)             [anomaly score]
    3. POST → NLP API (A1:5001/summarize)        [via Ollama]
    4. POST → TheHive API (B1:9000)              [create case]
    5. POST → MITRE Tagger if rule lacks tags    [auto-tag]
    6. IF auto_block:
         SSH → victim-lab (B2)
         iptables -I INPUT -s ATTACKER_IP -j DROP
         Log action to TheHive case
            │
            ▼
n8n Workflow 2:
  TheHive case-created webhook → extract observables
  → POST → Cortex (B1:9001) AbuseIPDB/VirusTotal analyzers
  → PATCH TheHive case with enrichment
            │
            ▼
Analyst sees ONE enriched case in TheHive with full timeline,
NLP description, MITRE tags, ML score, Cortex enrichment, response status
```

---

## VM profiles

### VM_A1 — soc-core
**Hostname:** `SOC-Core` | **IP:** `192.168.1.50` | **Host:** PC_A | **OS:** Ubuntu 26.04 LTS (Resolute Raccoon)

**Role:** SIEM core + SOAR + AI services

**Specifications:**
- RAM: 12 GB minimum / 20 GB recommended
- Storage: 200 GB SSD minimum / 500 GB SSD recommended
- SSD strongly recommended (Elasticsearch is disk-bound)
- No desktop GUI

**Services:**

| Service | Version | Port | Purpose |
|---|---|---|---|
| Elasticsearch | 8.x | 9200 | Central log/alert store |
| Kibana | 8.x | 5601 | SIEM UI + detection rules engine |
| Logstash | 8.x | 5044 / 5140 | Log preprocessing |
| Fleet Server (Elastic Agent) | 8.x | 8220 | Central agent management |
| n8n | latest | 5678 | SOAR / workflow automation |
| Ollama | latest | 11434 | Local LLM runtime (llama3.1:8b) |
| ML API | — | 5000 | Isolation Forest anomaly scoring |
| NLP API | — | 5001 | Alert summarization + AI rule generation |
| Correlation Engine | — | 5002 | Alert grouping / deduplication |
| MITRE Auto-Tagger | — | 5003 | Auto MITRE ATT&CK tagging |

**Memory budget:**
- Elasticsearch: 4 GB heap
- Ollama (llama3.1:8b loaded): ~6 GB
- Logstash: 1 GB heap
- Kibana: ~1 GB
- Flask APIs combined: ~1 GB
- n8n: ~512 MB
- Fleet Server / Elastic Agent: ~512 MB
- OS overhead: ~1 GB
- **Total: ~15 GB** — fits in 16 GB but 20 GB gives breathing room

**System paths (where services live, set by package install):**
```
/etc/elasticsearch/elasticsearch.yml
/etc/elasticsearch/jvm.options.d/heap.options
/etc/kibana/kibana.yml
/etc/logstash/conf.d/soc-pipeline.conf
/opt/Elastic/Agent/elastic-agent.yml
/opt/n8n/.env
/opt/ml-api/{app.py, train.py, model.pkl}
/opt/nlp-api/{app.py, prompt_template.txt, .env}
/opt/correlation-engine/{app.py, state.db}
/opt/mitre-tagger/{app.py, attack_cache.json, classtype_map.json}
```

**Local project folder (mirror of editable configs/code, kept under version control via the VM's own backups, NOT Git):**
```
~/soc-project/
├── elasticsearch/         (copy of editable config snippets)
├── kibana/
├── logstash/
├── fleet/
├── ollama/
├── n8n/
│   └── workflows/         (exported workflow JSON for redeploy)
├── ml-api/
├── nlp-api/
├── correlation-engine/
└── mitre-tagger/
```

**Startup order (after reboot):**
```
elasticsearch  →  wait 30s
kibana         →  wait 60s (initial start is slow)
logstash
elastic-agent  (Fleet Server)
ollama
ml-api
nlp-api
correlation-engine
mitre-tagger
n8n
```

---

### VM_B1 — incident-mgmt
**Hostname:** `incident-mgmt` | **IP:** `192.168.1.51` | **Host:** PC_B | **OS:** Ubuntu Server 22.04 LTS

**Role:** Incident management + threat intelligence

**Specifications:**
- RAM: 10 GB minimum / 14 GB recommended
- Storage: 100 GB minimum / 200 GB recommended
- No desktop GUI

**Services:**

| Service | Version | Port | Purpose |
|---|---|---|---|
| Cassandra | 4.1.x | 9042 | TheHive backend storage |
| Elasticsearch (local) | 8.x | 9200 | TheHive + Cortex indexing |
| TheHive | 5.x | 9000 | Case management |
| Cortex | 4.x | 9001 | Observable enrichment |
| MISP | Docker | 8080 / 8443 | Threat intelligence platform |

**Memory budget (caps are intentional — do not raise):**
- Cassandra heap: 512 MB
- Elasticsearch heap: 1 GB
- TheHive: ~1 GB
- Cortex: ~500 MB
- MISP Docker stack: ~1 GB
- OS overhead: ~500 MB
- **Total: ~4.5 GB** — fits comfortably in 10 GB

**Critical startup order — never skip the sleeps:**
```
cassandra      →  wait 40s
elasticsearch  →  wait 20s
thehive        →  wait 30s
cortex
docker compose up -d (MISP stack)
```

**System paths:**
```
/etc/cassandra/cassandra-env.sh
/etc/cassandra/cassandra.yaml
/etc/elasticsearch/elasticsearch.yml
/etc/elasticsearch/jvm.options.d/heap.options
/etc/thehive/application.conf
/etc/cortex/application.conf
/opt/misp-docker/.env
/opt/misp-docker/docker-compose.yml
```

**Local project folder:**
```
~/soc-project/
├── cassandra/
├── elasticsearch/
├── thehive/
├── cortex/
└── misp/
```

---

### VM_B2 — victim-lab
**Hostname:** `victim-lab` | **IP:** `192.168.1.53` | **Host:** PC_B | **OS:** Ubuntu Server 22.04 LTS

**Role:** Vulnerable target + network IDS

**Specifications:**
- RAM: 3 GB minimum / 4 GB recommended
- Storage: 40 GB minimum / 80 GB recommended

**Services:**

| Service | Port | Purpose |
|---|---|---|
| Apache2 + PHP 8.x | 80 | Web server hosting DVWA |
| MariaDB | 3306 | DVWA database |
| vsftpd | 21 | FTP brute-force target |
| OpenSSH | 22 | SSH brute-force target |
| Suricata | — | Network IDS (listens on ZeroTier interface) |
| suricata-update | — | Daily ET ruleset auto-refresh |
| Elastic Agent | →8220 | Ships host + Suricata logs to A1 Fleet |

**Vulnerabilities (intentional, lab only):**
- DVWA at security level "low": SQLi, XSS, CSRF, RFI, command injection, file upload
- PHP `allow_url_include = On`
- SSH password auth enabled
- FTP local users enabled

**Weak test accounts (intentional brute-force targets):**
```
testuser1 / password123
testuser2 / admin
webadmin  / webadmin
```

**System paths:**
```
/etc/apache2/apache2.conf
/etc/apache2/conf-available/soc-logging.conf
/etc/php/8.x/apache2/php.ini
/var/www/html/dvwa/config/config.inc.php
/etc/ssh/sshd_config
/etc/vsftpd.conf
/etc/suricata/suricata.yaml
/etc/suricata/classification.config       ← used by MITRE Tagger
/var/lib/suricata/rules/suricata.rules
/var/log/suricata/eve.json                 ← read by Elastic Agent
/opt/Elastic/Agent/elastic-agent.yml
```

**Local project folder:**
```
~/soc-project/
├── apache/
├── mariadb/
├── ssh-and-vsftpd/
├── suricata/
├── elastic-agent/
└── dvwa/
```

---

### VM_A2 — kali-attacker
**Hostname:** `kali-attacker` | **IP:** `192.168.1.52` | **Host:** PC_A | **OS:** Kali Linux

**Role:** Offensive testing — generates simulated attacks

**Specifications:**
- RAM: 3 GB minimum / 4 GB recommended
- Storage: 40 GB minimum / 80 GB recommended

**Tools used:**
- `hydra` — SSH/FTP brute force
- `nmap` — port scanning
- `sqlmap` — automated SQL injection
- `curl` — manual HTTP attacks (XSS, command injection, RFI)
- `msfconsole` — Metasploit framework for reverse shells
- `scp`, `rsync`, `wget` — data exfiltration simulation

**Attack target endpoints:**
```
SSH:    ssh testuser1@192.168.1.53
HTTP:   http://192.168.1.53/dvwa
FTP:    ftp://192.168.1.53
```

**Local project folder:**
```
~/soc-project/
└── attack-scripts/
```

---

## Detection rules (custom — 13)

All custom rules are MITRE ATT&CK tagged and trigger n8n webhook.
Rules are exported as JSON and kept locally in `~/soc-project/n8n/exports/soc-rules.ndjson` on VM_A1.

| Rule ID | Name | Tactic | Technique | Severity | Action |
|---|---|---|---|---|---|
| SOC-001 | SSH brute force (8+/2min) | TA0006 | T1110.001 | Medium | webhook |
| SOC-002 | SSH brute force critical (15+/2min) | TA0006 | T1110.001 | Critical | webhook + auto_block |
| SOC-003 | SQL injection attempt | TA0001 | T1190 | Medium | webhook |
| SOC-004 | SQL injection critical payload | TA0001 | T1190 | High | webhook + auto_block |
| SOC-005 | XSS attempt | TA0002 | T1059.007 | Medium | webhook |
| SOC-006 | OS command injection | TA0002 | T1059.004 | High | webhook + auto_block |
| SOC-007 | Port scan (10+/1min) | TA0043 | T1046 | Low | webhook |
| SOC-008 | Critical system file modified | TA0003 | T1098 | High | webhook |
| SOC-009 | Web shell detected | TA0003 | T1505.003 | Critical | webhook + auto_block |
| SOC-010 | Sudo privilege escalation | TA0004 | T1548.003 | High | webhook |
| SOC-011 | Reverse shell command | TA0002 | T1059 | Critical | webhook + auto_block |
| SOC-012 | SSH login from attacker IP | TA0008 | T1021.004 | High | webhook |
| SOC-013 | Data exfiltration indicator | TA0010 | T1048 | High | webhook |

These 13 are layered on top of (not replacing) ~1000 Elastic prebuilt rules covering the full ATT&CK matrix and ~50,000 Suricata ET Open rules.

---

## n8n workflows

### Workflow 1 — Main alert pipeline
- **Trigger:** `POST /webhook/elastic-alert`
- **Steps:** Correlation decision → branch on action → ML score → NLP summary → TheHive case create/update → optional active response
- **Local export:** `~/soc-project/n8n/workflows/01-alert-pipeline.json`

### Workflow 2 — TheHive observable enrichment
- **Trigger:** TheHive case-created webhook → `POST /webhook/thehive`
- **Steps:** Extract IP/hash observables → call Cortex analyzers → PATCH case with results
- **Local export:** `~/soc-project/n8n/workflows/02-cortex-enrichment.json`

### Workflow 3 — Daily digest
- **Trigger:** Scheduled, daily at 8am
- **Steps:** Read queued low-priority alerts → consolidate via NLP API → create one summary TheHive case
- **Local export:** `~/soc-project/n8n/workflows/03-daily-digest.json`

### Workflow 4 — AI rule generation from MISP
- **Trigger:** MISP new-event webhook → `POST /webhook/misp`
- **Steps:** Extract threat report → call NLP API `/generate-rule` → call MITRE Tagger → deploy to Kibana via API + Suricata via SSH
- **Local export:** `~/soc-project/n8n/workflows/04-misp-rule-gen.json`

### Workflow 5 — Weekly maintenance
- **Trigger:** Scheduled, Mondays 4am
- **Steps:** Call MITRE Tagger `/scan/kibana` and `/scan/suricata` → call ML API `/train` → post summary to TheHive
- **Local export:** `~/soc-project/n8n/workflows/05-weekly-maintenance.json`

---

## Custom Flask APIs — interfaces

### ML Anomaly Detection API (A1:5000)

```
POST /score
  Input:  { "risk_score": int, "rule_id": str, "source_ip": str, ... }
  Output: { "anomaly_score": 0.87, "is_anomaly": true, "model": "IsolationForest" }

POST /train
  Retrain model on last 30 days of Elasticsearch alerts.

GET  /health
```

### NLP API (A1:5001)

```
POST /summarize
  Input:  alert JSON with rule_name, mitre_technique, source_ip, severity, etc.
  Output: { "summary": "human-readable text", "severity": "high",
            "recommended_action": "block_ip" }

POST /generate-rule
  Input:  MISP threat event JSON
  Output: { "kibana_rule": {...}, "suricata_rule": "alert ...",
            "confidence": 0.85 }

GET  /health
```

Backend: Ollama at `http://localhost:11434` with model `llama3.1:8b`.

### Correlation Engine (A1:5002)

```
POST /correlate
  Input:  { "source_ip": str, "rule_id": str, "mitre_tactic_id": str,
            "mitre_technique_id": str, "severity": str, "anomaly_score": float,
            "auto_block": bool }
  Output: { "action": "create_new"|"add_to_existing"|"escalate_existing"|"suppress"|"queue",
            "bucket_id": int, "case_id": str, "chain_detected": {...} }

POST /correlate/set_case
  Input:  { "bucket_id": int, "case_id": str }

GET  /state
POST /whitelist/add
POST /close
```

State: SQLite DB at `/opt/correlation-engine/state.db`.
Window: 30 minutes for grouping. Kill chain window: 2 hours.

### MITRE Auto-Tagger (A1:5003)

```
POST /tag
  Input:  { "rule_text": str, "rule_type": "suricata"|"kibana"|"generic" }
  Output: { "tactic_id": "TA0001", "technique_id": "T1190",
            "confidence": 0.85, "method": "tfidf"|"classtype_map"|"ai" }

POST /scan/kibana       # Sweep all Kibana rules, patch untagged
POST /scan/suricata     # Sweep Suricata rules file, inject metadata
POST /refresh           # Re-download MITRE ATT&CK JSON, rebuild caches
GET  /health
```

ATT&CK data cached at `/opt/mitre-tagger/attack_cache.json` (refreshes weekly).
Classtype map cached at `/opt/mitre-tagger/classtype_map.json` (rebuilds when `classification.config` mtime changes).

---

## Credentials & endpoints

> Replace `FILL_AFTER_SETUP` with real values after each phase.
> **Never put real credentials in this Git repo.** This file is shared on Git.
> Keep real values in a local `~/soc-project/.env.local` on each VM.

### VM_A1 — soc-core

```yaml
elasticsearch:
  url: https://192.168.1.50:9200       # TLS, self-signed cert
  user: elastic
  password: see ~/soc-project/.env.local on VM_A1 (ELASTIC_PASSWORD)
  ca_fingerprint: 3c05387e1bd8f68441f718f08611bcc7d7d22d02e3be8901beeced45976965d4

kibana:
  url: http://192.168.1.50:5601        # plain HTTP, lab; auth via elastic user
  user: elastic
  password: same as elasticsearch (ELASTIC_PASSWORD)

fleet_server:
  url: https://192.168.1.50:8220       # self-signed cert; agents enroll with --insecure
  service_token: see ~/soc-project/.env.local on VM_A1 (FLEET_SERVER_SERVICE_TOKEN)
  fleet_server_policy_id: fleet-server-policy
  victim_lab_policy_id: c226ca2c-fcd2-40c8-9ca6-11392fc7e24e   # "victim-lab policy" — System integration + monitoring
  enrollment_token_victim_lab: generate via Fleet API (policy-bound, rotatable). See "Agent enrollment" below.
  enrollment_token_attacker: descoped (VM_A2 out of scope)

n8n:
  url: http://192.168.1.50:5678
  encryption_key: FILL_AFTER_SETUP

ollama:
  url: http://localhost:11434
  model: llama3.1:8b

flask_apis:
  ml_api:        http://192.168.1.50:5000
  nlp_api:       http://192.168.1.50:5001
  correlation:   http://192.168.1.50:5002
  mitre_tagger:  http://192.168.1.50:5003
```

**Agent enrollment — generate a fresh victim-lab token (run on VM_A1):**
```bash
curl -s -u elastic:$ELASTIC_PASSWORD -H "kbn-xsrf: soc" -H "Content-Type: application/json" \
  -X POST "http://localhost:5601/api/fleet/enrollment_api_keys" \
  -d '{"policy_id":"c226ca2c-fcd2-40c8-9ca6-11392fc7e24e","name":"victim-lab-key-'"$(date +%s)"'"}' \
  | python3 -c "import json,sys; print(json.load(sys.stdin)['item']['api_key'])"
```

**Then on VM_B2 (or any agent host) — install command template:**
```bash
sudo ./elastic-agent install \
  --url=https://192.168.1.50:8220 \
  --enrollment-token=<token from above> \
  --insecure \
  --non-interactive
```
After enrollment, add **Apache HTTP Server**, **Custom Logs** (eve.json, access_soc.log, vsftpd.log) integrations to the policy via Kibana → Fleet → victim-lab policy → Add integration.

### VM_B1 — incident-mgmt

```yaml
thehive:
  url: http://192.168.1.51:9000
  default_login: admin@thehive.local / secret  # CHANGE ON FIRST LOGIN
  api_key: FILL_AFTER_SETUP

cortex:
  url: http://192.168.1.51:9001
  org: SOC-LAB
  thehive_user_api_key: FILL_AFTER_SETUP

misp:
  url: https://192.168.1.51:8443
  default_login: admin@admin.test / admin  # CHANGE ON FIRST LOGIN
  api_key: FILL_AFTER_SETUP

cortex_analyzer_keys:
  abuseipdb: FILL_AFTER_SETUP    # free tier, 1000/day
  virustotal: FILL_AFTER_SETUP   # free tier, 500/day
  otx: FILL_AFTER_SETUP          # free
```

### n8n credentials (configured via UI, IDs stable)

```yaml
thehive_api:    Header Auth, Bearer FILL_AFTER_SETUP
cortex_api:     Header Auth, Bearer FILL_AFTER_SETUP
ssh_victim:     SSH, host 192.168.1.53, user soc-response, key ~/.ssh/soc_response
elasticsearch:  Basic Auth, elastic / FILL_AFTER_SETUP
```

### SSH key for active response (VM_A1 → VM_B2)

```
Generated: ~/.ssh/soc_response (ed25519)
Public key copied to: soc-response@192.168.1.53:~/.ssh/authorized_keys
Sudo NOPASSWD for iptables on VM_B2: /etc/sudoers.d/soc-response
```

---

## Threat intelligence sources

All free, no paid tier required.

| Source | Type | Refresh | Enabled in |
|---|---|---|---|
| Emerging Threats Open | Suricata rules | Daily (suricata-update) | VM_B2 |
| CIRCL OSINT | MISP feed | 6h | VM_B1 MISP |
| Abuse.ch URLhaus | MISP feed | 6h | VM_B1 MISP |
| Abuse.ch Feodo Tracker | MISP feed | 6h | VM_B1 MISP |
| AlienVault OTX | MISP feed | 6h | VM_B1 MISP |
| ET Compromised IPs | MISP feed | 6h | VM_B1 MISP |
| MITRE ATT&CK STIX | JSON download | Weekly | VM_A1 MITRE Tagger |
| AbuseIPDB | API analyzer | Per-query | VM_B1 Cortex |
| VirusTotal | API analyzer | Per-query | VM_B1 Cortex |

---

## Cron schedules

```
VM_A1 (soc-core):
  0 4 * * 0   curl -X POST http://localhost:5003/refresh           # Weekly MITRE refresh, Sundays 4am

VM_B2 (victim-lab):
  0 3 * * *   suricata-update && systemctl reload suricata && \    # Daily ET ruleset
              curl -X POST http://192.168.1.50:5003/scan/suricata  # Daily auto-tag

n8n scheduled workflows (set via UI):
  Daily 08:00   →  Workflow 3 (daily digest)
  Mondays 04:00 →  Workflow 5 (weekly maintenance: ML retrain + rule rescan)
```

---

## Shared Git repo structure (`~/soc-shared/`)

This is the ONLY thing on GitHub. Everything else stays local on each VM.

```
soc-shared/
├── CLAUDE.md                  ← this file (updated by every VM)
├── PROJECT-MASTER-PLAN.md     ← reference plan, rarely changes
├── README.md
└── docs/
    ├── architecture.md        ← architecture notes for the rapport
    ├── test-results.md        ← Phase 10 outcomes
    └── report-notes.md        ← raw material for rapport de PFE
```

---

## Local project folder structure (`~/soc-project/` on each VM)

This is **never on Git**. Each VM has only the folders relevant to it.

```
~/soc-project/                 (VM_A1 example)
├── .env.local                 ← real credentials, NEVER committed
├── elasticsearch/
├── kibana/
├── logstash/
├── fleet/
├── ollama/
├── n8n/
│   └── workflows/             ← exported workflow JSON
├── ml-api/
│   ├── app.py
│   ├── train.py
│   └── requirements.txt
├── nlp-api/
├── correlation-engine/
└── mitre-tagger/
```

---

## Git workflow

**The only repo on Git is `soc-shared`. The only files committed are CLAUDE.md and `docs/`.**

Before starting any work on a VM:
```bash
cd ~/soc-shared && git pull
```

After completing a phase / sub-step:
```bash
cd ~/soc-shared
# Update CLAUDE.md (current state, last session notes)
# If documentation belongs in docs/, update it too
git add CLAUDE.md docs/
git commit -m "phase-N.M: <hostname>: <what was done>"
git push
```

**What never goes on Git, ever:**
- Anything in `~/soc-project/` — configs, code, workflows, ML models, logs, attack scripts
- Real credentials (passwords, API keys, tokens)
- Any binary, large file, or data export

If a config file or workflow needs to be referenced from CLAUDE.md, describe its location with a path on the relevant VM, not the file contents.

**Backup strategy for `~/soc-project/`:**
Since project files aren't on Git, back them up locally:
- Periodic `rsync ~/soc-project/ /backup/path/` to a USB drive or another VM
- Or take VM snapshots at major milestones
- Worst case: a single VM failure means rebuilding that VM from the master plan

---

## Current state

```yaml
phase_0_git_setup:                    complete       # VM_A1, VM_B1, VM_B2 done; VM_A2 descoped (see notes 2026-04-29)
phase_1_zerotier:                     complete       # VM_A1 .50 (node 785fd1806c), VM_B1 .51 (node 9ab369cb6c), VM_B2 .53 (node aa429ed844) all reachable; VM_A2 descoped
phase_2_vm_a1_siem_core:              complete       # ES 8.19.14 single-node (heap dropped 4G→2G in phase 3 to fit Ollama; backup at heap.options.bak-20260505), Kibana 8.19.14, Logstash 8.19.14 (beats:5044, syslog:5140 → ES), Elastic Agent + Fleet Server 8.19.14 on :8220 enrolled in fleet-server-policy. ufw active with allow from 192.168.1.0/24. SOC-Core OS is Ubuntu 26.04 LTS (matches VM_B1, plan said 22.04).
phase_3_vm_a1_soar_and_ai:            complete       # n8n 2.18.5 systemd on :5678; Ollama 0.22.1 + llama3.1:8b warm with KEEP_ALIVE=24h (cold load ~5min, ~0.3 tok/s CPU-only); 8G swap added. RE-ARCHITECTED: only 2/4 Flask APIs built — ML Anomaly :5000 (IsolationForest, /train pulls .alerts-security.alerts-default), Correlation Engine :5002 (SQLite, 30-min bucket / 2h kill-chain windows; 6 actions verified). DROPPED: NLP API (replaced by n8n→Ollama directly via HTTP node) and MITRE Auto-Tagger (replaced by Kibana's built-in threat[] field at rule definition + Sigma tags + community classtype JSONs). systemd units for ml-api + correlation-engine; /opt/<svc> symlinks back to ~/soc-project/<svc>.
phase_4_vm_b1_incident_mgmt:          complete       # Cassandra + ES + TheHive + Cortex (custom soc-cortex:4.0.1-analyzers image, process mode) + MISP all running; 4 MISP feeds enabled with 6h cron; 30 Cortex analyzers active in SOC-LAB org (4 original verified end-to-end + 26 no-key bulk-enabled 2026-05-14); MISP analyzer URL fixed 127.0.0.1→192.168.1.51 (2026-05-14); TheHive→Cortex + TheHive→MISP connector servers registered via thehive/conf/application.conf (cortex.servers + misp.servers, 2026-05-14) — TheHive UI "Run Analyzer" + MISP-pull scheduled actor now functional; ufw active with ZT-only allow rules
phase_5_vm_b2_victim_lab:             complete       # Apache+PHP8.5+MariaDB up; DVWA at /dvwa; vsftpd:21 + ssh:22 + 3 weak users; Suricata 8.0.3 with 49911 ET Open rules; Elastic Agent enrolled in victim-lab policy (agent_id c328f63a-4d33-437b-9cc1-cdfbb060df45) HEALTHY; integrations attached on VM_A1 Kibana: system-2 + apache-victim-lab (/var/log/apache2/access_soc.log+access.log+error.log) + suricata-victim-lab (/var/log/suricata/eve.json) + vsftpd-victim-lab (Custom Logs /var/log/vsftpd.log → dataset 'vsftpd'). Used native Suricata + Apache integrations (NOT Custom Logs) — pre-parsed/ECS/dashboards. Suricata index growing live (~58k+ docs at attach time). Active-response wired 2026-05-07: soc-response user (uid 1004, password locked) with VM_A1's ed25519 pubkey in /home/soc-response/.ssh/authorized_keys + /etc/sudoers.d/soc-response NOPASSWD for both /sbin/iptables and /usr/sbin/iptables; verified end-to-end by Phase 10 SQLi test (WF1 SSH+iptables rc=0). SSH brute-force log shape fixed 2026-05-08: PerSourcePenalties no in /etc/ssh/sshd_config.d/99-soc-lab.conf so repeated auth failures emit standard "Failed password" lines instead of OpenSSH 9.x's "penalty: failed authentication" — required for SOC-001/SOC-002 to fire (per VM_A1 Phase 10 note). OBSOLETE: original "MITRE Tagger /scan/suricata cron + /refresh from classification.config" deferred items dropped in Phase 3 re-architecture (Kibana threat[] + Sigma attack.t#### + community classtype JSONs replace it).
phase_6_vm_a2_kali_attacker:          descoped       # user descoped 2026-04-29 — attacks can be launched from any reachable host or skipped
phase_7_detection_layer_activation:   complete       # 1644 Elastic prebuilt rules installed; ES license trial-activated for .webhook connector; n8n webhook connector id 7c351a6c-4de6-4c07-8146-fa337033c735 (POST → http://192.168.1.50:5678/webhook/elastic-alert); 13 SOC custom rules (SOC-001..SOC-013) imported from ~/soc-project/kibana/soc-rules.ndjson and enabled, all with native MITRE threat[] tagging, webhook action, meta.auto_block flag (true on 002/004/006/009/011); SOC-008 partial-failure benign (FIM indices not yet present); Suricata/Apache rules waiting on VM_B2 to come back online
phase_8_soar_integration:             complete       # WF1+WF2 ACTIVE; B1 wiring done 2026-05-07: real bearers minted (TheHive=rtHZuTw01P0UwaeDKmcT04bBQZmxvlGM for soc-bot@thehive.local in new SOC-LAB org [analyst profile]; Cortex=existing cortex-user key 6MWnt7E3FdH0muqjoG6Xyd+5msDQf4S2 reused). VM_A1 owner pastes bearers into n8n UI for cred ids Ux32rgVuHoXKc1GY / HK1qH743oIbnpSbk (n8n public API has no PATCH for creds; UI edit keeps id stable). TheHive→WF2 wired via polling bridge (~/soc-project/thehive-cortex/cron-cases-to-wf2.sh, every 1m) — TheHive 5.7.1 native notification.endpoints+items config accepts the rule but actor never dispatches; bridge posts Case JSON in TheHive's native envelope shape ({operation,objectType,objectId,object}) directly to /webhook/thehive. Smoke verified case=~32880 wf2=200. WF2 DYNAMIC DISPATCH refactor 2026-05-14 (VM_A1): replaced 8 hardcoded Cortex node fanouts with a single dynamic per-observable dispatcher — Fetch live analyzers (GET /api/v1/connector/cortex/analyzer?range=all → 31 analyzers post-connector-registration) + Build dispatch list (Code, filters by observable.dataType, emits {observableId, analyzerId} pairs) + batched HTTP POST /api/connector/cortex/job (cortexId="SOC-LAB-Cortex", batching {batchSize:1, batchInterval:1500} for Sink.asPublisher race). 16 IP analyzers now dispatched per IP observable (vs hardcoded 4); workflow auto-adapts when org enables more analyzers. Verified case=~162205800: 16 dispatches succeeded, 9 Cortex Success / 5 Cortex Failure (analyzer-side config gaps), 1 InProgress, real cortexJobIds assigned.
phase_9_adaptive_intelligence:        complete       # WF3+WF4+WF5 ACTIVE; MISP→WF4 wired via polling bridge (~/soc-project/misp/cron-publish-to-wf4.sh, every 1m, uses /events/restSearch with publish_timestamp filter — MISP 2.5 has no native single-URL outbound webhook). WF4 Ollama timeout caveat unchanged.
phase_10_testing:                     complete       # 2026-05-10: 7 SOC rules verified end-to-end (cases #13/14/15/17/18/19 + correlation absorption); 7 bug fixes applied this Phase: #1 sshd→sshd-session OpenSSH 9.x rename (SOC-001/002/012 KQL), #2 SOC-005 KQL bare-`<script>` invalid → URL-encoded only, #3 SOC-011 KQL wildcards-around-quoted-phrase invalid → `message:"bash -i" or ...`, #4 (same as #1 for SOC-012), #5 WF1 source_ip array-of-string fix via IIFE pick-first-IPv4 (Bug confirmed when SOC-011 multi-NIC host emitted JSON-string-of-array breaking JSON body interpolation), #6 SOC-011 self-trigger fix via `not host.name:"soc-core"`, #7 WF2 fetch-observables GET→POST query API (TheHive 5 has no GET /case/{id}/observables; was returning 404 silently breaking Cortex enrichment forever). New WF1 node `wf1-14b-add-observable` added (POST /case/{id}/observable with source_ip → ip dataType, tlp 2, ioc true). Auto-block path verified end-to-end SOC-006 from .52 (kali) → iptables DROP on B2 targeting .52 → A2 lost B2 access, A1 retained access (selective-source verification). WF2 Cortex remaining gap: AbuseIPDB worker not registered (worker not found); requires user-supplied API key in Cortex UI — config gap, not pipeline bug. Latency: fast paths (no NLP) 0.4-2.8s, slow paths (Ollama summarize) 60-632s. Correlation reduction observed: SOC-006 second batch 36 alerts → 1 case via Kibana action summary mode + correlation 30-min bucket; SOC-006 first batch absorbed into existing SOC-012 bucket from same source (escalate_existing).
phase_11_documentation:               in_progress   # handbook delivered 2026-05-14 (docs/SOC-AUTONOME-HANDBOOK.md + .html, 1834 lines covering all 4 VMs / 5 workflows / 9 pipeline scenarios) — PFE rapport itself still pending

last_updated: 2026-05-15
updated_by: incident-mgmt (VM_B1)
```

---

## Last session notes

> Latest entry on top. Each entry: what was done, what's pending, anything next instance needs to know.
> Maximum 5 entries kept; older ones archived in `docs/session-history.md`.

```
2026-05-15 (latest) — incident-mgmt (VM_B1) — Full project audit + ack of A1's WF2 dispatcher pull + Platinum trial expired
  Done:
    - Pulled A1's commit 9c832ec ("phase-8: soc-core: WF2 dynamic Cortex
      dispatcher + B1-side polling gap"). WF2 now fans out to all 31
      analyzers dynamically — supersedes the 4-static-analyzer fanout
      and resolves the WF2-side enrichment ceiling.
    - Performed full project round (12 findings, prioritized below).
      Methodology: re-read all phase states + per-component verification
      against running stacks on B1 + cross-check against A1's last
      session note + git history.

  Findings — needs action (priority order):
    F1 [URGENT/EXPIRED] TheHive Platinum trial expired 2026-05-14 23:59 UTC.
       Today is 2026-05-15 — already lapsed. Demo cases #13–19 + tag-driven
       case templates were tied to Platinum features. Export anything still
       needed before downgrade auto-applies; UI may also drop org-admin
       views post-expiry. ACTION: dump cases via /api/case/_search +
       /api/observable/_search to JSON, store under
       ~/soc-project/thehive-cortex/exports/2026-05-15-pre-downgrade/
    F2 [CRITICAL — B1] TheHive→Cortex job-status polling stuck (A1
       surfaced this in 9c832ec). 15/16 jobs InProgress 8+ min after
       Cortex Success. Likely cause: my application.conf uses
       `refreshDelay = 1 minute` + `statusCheckInterval = 1 minute` —
       keys may not even be valid TheHive 5.7 settings (need to verify
       against TheHive boot log for "unrecognized setting" warnings).
       Recommended next step: grep TheHive boot log for "cortex" config
       lines + try `refreshDelay = 10 seconds` (or migrate to Cortex
       job-completed webhook → TheHive push model). NOT YET ACTIONED —
       gated on user approval (requires application.conf edit + TheHive
       restart; Cassandra/ES untouched).
    F3 [HIGH — A1] WF1 broken (exec 412 errored at TheHive: "create case
       with invalid syntax"). Today's alert pipeline is dead until fixed.
       Likely a schema-drift issue introduced when one of WF1's helper
       nodes was last edited; needs A1-side n8n DB inspection.
    F4 [HIGH — security] 4 leaked secrets in this CLAUDE.md (Credentials
       section) to rotate: THEHIVE_BOT_API_KEY, SOCADMIN_KEY,
       CORTEX_USER_API_KEY, MISP_API_KEY. Repo is public; assume
       compromised. Rotate + scrub the keys from CLAUDE.md (replace with
       placeholders like <THEHIVE_BOT_API_KEY> and document fetch path
       e.g. `vault read soc/thehive`) + update n8n credentials store
       (A1 owner action for the n8n bearer cred ids Ux32rgVuHoXKc1GY /
       HK1qH743oIbnpSbk + the WF2 dispatch node).
    F5 [HIGH — B1] MISP default admin@admin.test / admin still active.
       Either change password + enable MFA, or create a dedicated
       PFE admin and disable the default account.
    F6 [MEDIUM — A1/B1] 4 SOC rules never fired end-to-end:
       SOC-008 (FIM — indices missing on B2), SOC-009, SOC-010, SOC-013.
       Phase 10 marked complete but only 7/13 rules verified.
    F7 [MEDIUM — A1] WF4 Ollama timeout (existing caveat), WF5 weekly
       maintenance broke 2026-05-11 — A1-side; no recovery yet.
    F8 [LOW — B1] 5 stale "DELETE ME" test cases (#25–29, tag=DIAG)
       in TheHive. Auto-mode blocked bulk delete previously; user
       cleanup via UI still pending. May moot now that Platinum
       expired (F1).
    F9 [LOW — B1] Cortex has 0 responders. Mailer + MISP_create_event +
       MISP_WarningLists are easy wins (no API keys needed for the
       MISP responders since MISP is local).
    F10 [LOW — B1] TheHive: 0 case templates, 0 custom fields,
        0 taxonomies imported. Would make case triage much more
        structured for the rapport demo.
    F11 [LOW — B1] MISP catalog under-utilized: 0/168 taxonomies enabled,
        0/123 warninglists, only 4/96 feeds active. Bulk-enable scripts
        could be a single curl loop.
    F12 [DELIVERABLE — all VMs] Phase 11 PFE rapport not written
        (handbook delivered as input; rapport itself still pending).
        Also missing: docs/architecture.md, docs/report-notes.md,
        defended-version git tag, metrics-compilation script.

  Pending (B1-side, needs user authorization before action):
    - F2 polling fix (application.conf edit — see above for the
      proposed refreshDelay = 10 seconds change + Cortex webhook
      alternative)
    - F1 case export before TheHive Platinum downgrade kicks in fully
    - F5 MISP default-admin rotation
    - F9/F10/F11 catalog enablement (low priority, can batch)

  Notes for next instance:
    - A1's WF2 dispatcher (9c832ec) and B1's connector wiring
      (1fb97ae) together solved the "Cortex isn't getting reached"
      side. The remaining enrichment ceiling is TheHive's read-side
      polling, not the dispatch path — F2 is the single highest-value
      B1 fix right now.
    - Soc-shared repo is public on GitHub — F4 secret rotation is not
      a hypothetical; treat the four keys as already-leaked.
    - All B1 work today done with the standing constraints intact:
      Cassandra + local ES untouched, organization+user data untouched,
      TheHive API key not rotated.

2026-05-14 — soc-core (VM_A1) — WF2 dynamic Cortex dispatcher + gaps surfaced for B1
  Done:
    - WF2 SURGERY: replaced static 4-analyzer fanout with dynamic
      per-observable dispatcher. The old WF2 had 8 hardcoded HTTP nodes
      (one per fixed analyzer ID, organized by Switch-by-dataType branches:
      ip → AbuseIPDB+VirusTotal+OTX+MISP; hash → 2 nodes; url → 2 nodes).
      These all referenced analyzer IDs by hand and never auto-updated.
      Now there is exactly ONE dispatch node that:
        1) GETs /api/v1/connector/cortex/analyzer?range=all (Fetch live
           analyzers — 31 analyzers visible since B1's connector
           registration earlier today)
        2) Pass observables to loop (Code: passthrough so splitInBatches
           sees observables, not the analyzer list which auto-splits
           into 10+ items)
        3) Loop observables (splitInBatches v3, batchSize 1)
        4) Build dispatch list (Code, filters analyzers whose dataTypeList
           includes observable.dataType, emits one item per pair)
        5) Skip if no analyzer applies (IF on _skip flag)
        6) Cortex via Hive: dispatch (batched) — POST /api/connector/cortex/job
           with cortexId="SOC-LAB-Cortex", batching {batchSize:1,
           batchInterval:1500} for the Sink.asPublisher race
        7) Loop feedback wired: dispatch leaf + skip-true branch both
           point BACK to Loop observables — REQUIRED for splitInBatches v3
           to advance to done branch
    - Verified end-to-end via case ~162205800 (smoke v6):
        31 analyzers fetched (up from 10 visible without ?range=all)
        16 IP-applicable analyzers dispatched on 8.8.8.8 (up from
          hardcoded 4)
        16/16 dispatches got real cortexJobIds (vs the "-" failures
          before the cortexId=SOC-LAB-Cortex fix)
        Cortex ran the jobs: 9 Success, 5 Failure (analyzer-side
          issues), 2 still running
        Threat Intel page generated with DShield "suspicious"
          verdict (3 threatfeeds)
    - Bug fixes consumed in this surgery (debug-time mistakes I made and
      then corrected — recording so next instance doesn't repeat them):
        #1 cortexId: the dispatch body MUST use "SOC-LAB-Cortex"; if
           you send "cortex0" or any guess, TheHive accepts the request,
           creates a Job record with status:Waiting, then within 1s
           flips it to status:Failure with cortexJobId="-" and
           analyzerName="-" — silent failure, no useful HTTP-side error
        #2 splitInBatches v3 needs the per-item branch terminals
           connected BACK to its own input (feedback cycle) to advance
           past the first batch — without it, the loop fires once and
           the "done" output is never reached. Originally the WF2
           backup had each Cortex POST node terminating with a connection
           back to "Loop observables (batchSize 1)"; preserve that pattern
        #3 ?range=all matters: default analyzer endpoint returns ~10
           items (pagination); always pass ?range=all to enumerate
        #4 n8n HTTP node auto-splits array responses into one item per
           array element. A GET that returns a JSON array of 10 analyzers
           becomes 10 items downstream. To preserve a single item
           carrying the array, either route the result through a Code
           passthrough that pulls the original input back, OR (cleaner)
           don't put the analyzer fetch in the main flow — use $('Fetch
           live analyzers').all() inside a sibling Code node and route
           the main flow around it
        #5 n8n workflow_entity has TWO version-id columns: versionId
           (editor) and activeVersionId (deployed). Updating only the
           first leaves the runtime running the old definition. Always
           bump BOTH and INSERT a matching workflow_history row keyed
           by the new id; otherwise the next n8n restart still loads
           the old activeVersionId snapshot

  Pending (B1-side — needs incident-mgmt instance):
    - CRITICAL: TheHive's Cortex job-status polling is too slow or
      broken. After WF2 dispatched 16 jobs and Cortex completed 9 of
      them as Success within 30 seconds, TheHive's listJob view still
      showed 15/16 as InProgress 8+ minutes later. Direct Cortex query
      confirms AbuseIPDB job 4zujKJ4BRp6pWzMKtG_t = Success with end
      timestamp, but TheHive never reflects this. Only DShield happened
      to sync (probably the very first poll). Symptoms cascade: the
      observable.reports map only contains DShield, so the WF2 Threat
      Intel page shows only DShield's verdict even though 9 analyzers
      returned data.
      Suggested investigation order:
        1) Check cortex.refresh-delay / poll interval in
           ~/soc-project/thehive-cortex/thehive/conf/application.conf
           — the cortex { servers = [...] } block likely has no refresh
           override and defaults to something multi-minute
        2) Inspect TheHive logs for the CortexActor heartbeat to
           confirm polling is happening at all
        3) Consider configuring Cortex job-completed webhook to push
           status to TheHive instead of relying on TheHive's pull
        Note: this is NOT a Cortex-side bug — Cortex has the answers
        ready. It is a TheHive→Cortex polling cadence issue.
    - Analyzer-side gaps (Cortex Failure status) — these are config
      problems on the analyzers themselves, low priority:
        AbuseIPDB → needs API key (status Success only because we
          previously set one; new dispatches still Success on this side)
        Robtex_Reverse_PDNS_Query, ThreatMiner — likely external
          service rejection (rate limit / region)
        CyberCrime-Tracker, ValidateObservable, Abuse_Finder — may
          need Cortex worker reinstall or analyzer cfg overrides
    - The previously listed B1 standing items are unchanged: 0 case
      templates, 0 custom fields, 0 taxonomies imported, only
      CaseCreated notification trigger, 0 Cortex responders.

  Note for next instance (on any VM):
    - The WF2 jsonBody hard-codes cortexId="SOC-LAB-Cortex". If B1
      ever renames its Cortex server, the dispatch will silently fail
      again. The dispatch jsonBody is in n8n DB (workflow HYiSFNStG5zEG6ZA,
      node "Cortex via Hive: dispatch (batched)") — there is no
      .env or external config to update.
    - WF2 backup before surgery is at
      ~/soc-project/n8n/workflows/backups/WF2-20260514-203440-pre-dynamic-analyzers.json
      on VM_A1 (in case rollback ever needed).
    - The dispatch is on a 1.5s batching interval to dodge TheHive's
      Sink.asPublisher race. With 16 IP analyzers per IP observable
      that is 16 × 1.5s ≈ 24s of dispatch time per observable. Cases
      with multiple IPs scale linearly.
    - WF2's static "Wait 180s for Cortex jobs" is currently the
      bottleneck — even if you fix TheHive's polling, the wait still
      caps total enrichment time. Long-term improvement: replace with
      a polling loop that exits when all dispatched jobs reach
      terminal status (Success/Failure) in TheHive's job records.

2026-05-14 (later) — incident-mgmt (VM_B1) — TheHive→Cortex + TheHive→MISP connector servers registered
  Done:
    - Added `cortex { servers = [...] }` and `misp { servers = [...] }`
      blocks to ~/soc-project/thehive-cortex/thehive/conf/application.conf
      (bind-mounted at /etc/thehive/application.conf inside the container).
      Cortex server points to http://192.168.1.51:9001 with the cortex-user
      bearer; MISP server points to https://192.168.1.51:8443 with the
      admin key and acceptAnyCertificate:true + 1-hour pull interval.
    - Restarted TheHive (Cassandra + ES untouched per standing constraint).
      Boot log confirms: `TheHiveMispClientImpl: Add MISP connection
      SOC-LAB-MISP` and `Starting actor MISP` as cluster singleton.
    - Verified via TheHive's connector layer:
        GET /api/connector/cortex/analyzer?range=all → 31 analyzers
          all tagged cortexIds=['SOC-LAB-Cortex'] (matches Cortex direct view)
        MISP integration confirmed by the MispActor singleton starting
          on boot — sync is scheduled (no manual HTTP route), first pull
          happens within the 1h interval window.
    - Net effect: TheHive UI "Run Analyzer" / "Run Responder" buttons
      now work directly against Cortex (no need to go through n8n WF2 for
      manual enrichment), and MISP events will start appearing as TheHive
      alerts on the scheduled poll. The existing n8n bridges
      (cron-cases-to-wf2.sh and cron-publish-to-wf4.sh) remain in place
      and unchanged — they were never blocked by the connector gap.

  Configuration trap discovered:
    - PUT /api/config/organisation/cortex returns 500 with
      "No configuration setting found for key 'organisation.defaults.cortex'"
      until a baseline `cortex { servers = [...] }` exists in
      application.conf. Per-org override via the config API ONLY works
      when the connector has a global default. Same for misp.
    - Org-admin SOCADMIN_KEY can read/write `/api/config/organisation/*`
      but the global `/api/config/cortex` path requires a SUPERADMIN
      bearer (admin@thehive.local). For lab use, just put it in
      application.conf — no superadmin needed.

  Pending / still missing in TheHive (small items):
    - 0 case templates (no SOC playbooks for SOC-001/006/etc.)
    - 0 custom fields (no mitre-tactic, originating-rule-id, etc.)
    - 0 taxonomies imported (no TLP-extended, MISP-galaxy)
    - Only 1 notification trigger (CaseCreated); could add CaseClosed,
      AlertCreated, TaskClosed for finer-grained SOAR signals
    - 0 Cortex responders (Mailer, MISP_create_event) — Cortex-side config,
      not TheHive's; would enable "Run Responder" from TheHive UI to
      auto-push observables back to MISP, send analyst emails, etc.

  Note for next instance:
    - application.conf changes require TheHive container restart to take
      effect (no hot-reload). Cassandra + ES stay running through the
      restart; TheHive recovers itself in ~30-60s.
    - The thehive/conf/application.conf file lives in
      ~/soc-project/thehive-cortex/thehive/conf/ on VM_B1 and is NOT in
      git (per repo convention — soc-shared only carries CLAUDE.md+docs).
      The current contents are: notification.endpoints (n8n-soc webhook),
      cortex.servers (SOC-LAB-Cortex), misp.servers (SOC-LAB-MISP).

2026-05-14 — incident-mgmt (VM_B1) — Project handbook delivered + Cortex analyzer expansion (4→30) + MISP analyzer URL fix + perf rescue
  Done:
    - PROJECT HANDBOOK delivered: docs/SOC-AUTONOME-HANDBOOK.md
      (1834 lines, ~18988 words) — full end-to-end description of all 4
      VMs, network topology, all 5 n8n workflows node-by-node with mermaid
      flowcharts, TheHive/Cortex/MISP stack, detection stack, 9 pipeline
      scenarios (4 success + 5 failure modes), credentials reference,
      outstanding work, operational cheatsheet. Companion
      docs/SOC-AUTONOME-HANDBOOK.html (self-contained viewer with rendered
      mermaid + syntax highlighting, opens in any browser, has a print
      stylesheet for Save-as-PDF).
    - CORTEX ANALYZER EXPANSION (4 → 30 instances active in SOC-LAB org):
      Previously enabled: AbuseIPDB, OTXQuery, MISP, VirusTotal_GetReport.
      Bulk-enabled all 26 no-key analyzers via
        POST /api/organization/analyzer/{definitionId}
      with default cfg {auto_extract_artifacts:false, check_tlp:true,
      max_tlp:2, check_pap:true, max_pap:2}. New analyzers (no API key
      needed, ready to use immediately):
        MaxMind_GeoIP, CIRCLHashlookup, Abuse_Finder, TeamCymruMHR,
        DShield_lookup, ThreatMiner, MaxMind_GeoIP, Mnemonic_pDNS_Public,
        QrDecode, DomainMailSPFDMARC, Inoitsu,
        MSDefenderOffice365_SafeLinksDecoder, SpamhausDBL, JA4_FoxIO,
        Robtex_Forward_PDNS_Query, Urlscan.io_Search, ValidateObservable,
        Cyberprotect_ThreatScore, Msg_Parser, DNSdumpster_report,
        CyberCrime-Tracker, Crt_sh_Transparency_Logs, GoogleDNS_resolve,
        DNS_Lookingglass, Robtex_Reverse_PDNS_Query, Robtex_IP_Query,
        ClamAV_FileInfo, UnshortenLink.
    - MISP ANALYZER URL FIX: existing MISP_2_1 instance was configured
      with url=["https://127.0.0.1:8443"] which never worked — MISP's
      docker container only binds the ZeroTier IP, not loopback. Patched
      via PATCH /api/analyzer/{instanceId} (NOT the org enable endpoint
      which is POST/insert-only and returns 404 on existing instances):
        url: https://127.0.0.1:8443 → https://192.168.1.51:8443
      Re-tested with 8.8.8.8 → Success, 0 events matched (expected for
      a clean IP). MISP analyzer is now end-to-end functional.
    - LIVE TEST OF ALL 4 ORIGINAL ANALYZERS (post-fix):
        OTXQuery + 8.8.8.8 → pulse_count=0, ASN AS15169 Google, US ✓
        AbuseIPDB + 8.8.8.8 → 42 abuse reports (port-scan reporters),
                                isp=Google LLC, country=US, score=0 ✓
        VirusTotal_GetReport + EICAR SHA256
          (275a021bbfb6489e54d471899f7db9d1663fc695ec2fe2a2c4538aabf651fd0f)
          → 66/75 engines malicious, meaningful_name=eicar.com-38682 ✓
        MISP + 8.8.8.8 → results=[], 0 events matched ✓
    - PERFORMANCE RESCUE (machine was crashing under load):
      Diagnosis: load avg 26 / 27 / 25 on 10 cores, %soft=17.66%,
      %sys=34% — classic VirtualBox I/O virt overhead. Swap thrashing
      (2.7G/4G used). After Hyper-V removal on host (user ran
      `bcdedit /set hypervisorlaunchtype off` + reboot) + in-VM cleanup
      (stopped misp-misp-modules-1 container, disabled GNOME animations,
      restarted stuck TheHive container which had hung on JanusGraph
      cluster init), load dropped to 4.5 / 3.6 / 2.4. RAM 7.2/13G, swap
      unused.
    - Host-side recommendations given to user (not yet all applied):
      switch VirtualBox NIC to virtio-net, drop vCPU 10→6/8, drop RAM
      13→10 GB, disable 3D accel, disable Memory Integrity / Core Isolation
      in Windows Security, run full
      `dism /online /disable-feature /featurename:Microsoft-Hyper-V-All`
      (and VirtualMachinePlatform / HypervisorPlatform / WSL).

  Pending / next session:
    - 5 test cases (#25–29) left from earlier TheHive notification work
      titled "PFE webhook ... DELETE ME", tag=DIAG. Auto-mode blocked
      bulk DELETE — user can clean up via UI.
    - Cortex still has 0 responders configured. Easy wins: Mailer,
      MISP_create_event, MISP_WarningLists.
    - Cortex superadmin-key work pending: SOCADMIN_KEY listed as org-admin
      only, can't list orgs/users. If superadmin needed for global config,
      mint one via admin@thehive.local in TheHive (NOT the Cortex
      superadmin which would also need creation).
    - Phase 11 (PFE rapport documentation) — handbook is the deliverable
      input but the rapport itself still pending.
    - Standing TODOs unchanged: rotate leaked keys in this CLAUDE.md
      (THEHIVE_BOT_API_KEY, SOCADMIN_KEY, CORTEX_USER_API_KEY, MISP_API_KEY);
      TheHive Platinum trial expires today (2026-05-14).

  Notes for next instance:
    - Cortex API gotcha: `POST /api/organization/analyzer/{definitionId}`
      ONLY creates instances. To update an existing instance, use
      `PATCH /api/analyzer/{instanceId}` with the full configuration
      payload — the POST path returns 404 "Worker not found" on existing.
    - Cortex `/api/analyzer` list endpoint is paginated and returns ONLY
      10 items by default. Use `?range=all` (or a numeric range like
      `?range=0-50`) to get the full set. Easy thing to miss.
    - The MISP analyzer URL trap: even with Cortex in `network_mode: host`,
      `127.0.0.1:8443` doesn't reach MISP because docker's port publishing
      `192.168.1.51:8443->80/tcp` ONLY binds the named interface, not the
      loopback. Always use the ZeroTier IP for MISP.
    - The `cortex-user` orgadmin key has NO access to /api/organization,
      /api/user, or analyzer-definition with full structure beyond the
      basic list. For org/user mgmt, would need a superadmin key.

2026-05-10 — soc-core (VM_A1) — Both A1-side bugs B1 flagged are patched
  Done:
    - Bug A1-#1 (WF2 Cortex analyzer ID mismatch): root cause was deeper
      than version mismatch. Cortex's run endpoint requires the analyzer
      INSTANCE UUID, not the analyzer DEFINITION ID — POSTing to
      /api/analyzer/AbuseIPDB_2_0/run also returns 404 (verified directly
      with curl + cortex-user bearer). B1's hint was right: instance UUID.
      Patched all 3 Cortex calls in WF2 (`HYiSFNStG5zEG6ZA`):
        Cortex: AbuseIPDB        → 6b1c7570c74b55db697a69aa2c719b4f
        Cortex: VirusTotal hash  → 8cac1902c2e7879f2d258a3bfc7ba1f5
        Cortex: VirusTotal URL   → 8cac1902c2e7879f2d258a3bfc7ba1f5
      Saved to ~/soc-project/n8n/workflows/02-cortex-enrichment.json.
      Verified end-to-end: WF2 exec 292 against case #19 → AbuseIPDB job
      zQP2E54B5tZAINNEkyVJ created (status=InProgress) → completed in <5s →
      report taxonomies: Usage=Reserved, Score=0, Reports=0 (private IP,
      expected). Cortex is now actually running the analyzer; the entire
      WF2 chain works.
      Caveat: WF2 fires-and-forgets the Cortex job (just logs "dispatched"
      to TheHive case after Cortex returns). The Cortex report does NOT
      auto-attach to the TheHive observable — that would require the
      TheHive "run analyzer" endpoint instead of direct Cortex calls.
      Acceptable for Phase 10 (analyzers prove they work and run); a
      proper Phase-11-or-later refactor could move the orchestration to
      TheHive's responder framework so reports land on observables.
    - Bug A1-#2 (WF3 Daily Digest control-char): patched WF3 (`Z1VpjJlhg2Skek1B`)
      "TheHive: create digest case" jsonBody to apply
      `String(x).replace(/[\x00-\x1f]/g,' ')` to all three string-injected
      fields (AI summary, by_rule, lines), not just the AI summary as
      before. Saved to ~/soc-project/n8n/workflows/03-daily-digest.json.
      Validation deferred — n8n public API doesn't expose manual workflow
      execution; next scheduled run is 2026-05-11 08:00 UTC. The fix is
      the literal recommendation B1 made.
    - Bug A1-#2b (historical exec 253 host-unreachable): noted, no action —
      already resolved by B1's boot-orchestration fix.

  Phase 10 status delta:
    - WF2 enrichment routing: previously broken end-to-end (404 on /run).
      Now fully functional through Cortex job creation + completion.
    - Phase 10 metric "active analyzer instances reachable from WF2": 1/3
      verified (AbuseIPDB), 2/3 patched but not yet smoke-tested
      end-to-end through WF2 (VirusTotal hash + URL — same UUID-resolution
      mechanism, expected to work).

  Pending / known gaps:
    - Phase 11 (rapport) — only open phase. Material in ~/pfe/docs/REPORT.md
      and soc-shared docs/.
    - WF2 → TheHive observable report-attach (design choice, see caveat
      above) — optional polish for Phase 11 demo.
    - WF3 manual smoke at 2026-05-11 08:00 UTC will confirm Bug A1-#2 fix
      end-to-end; meanwhile the patch is staged.

  Things to know:
    - Cortex instance UUIDs are stable across redeploys when persisted via
      Cortex DB. If the SOC-LAB org's analyzers are ever recreated, the
      UUIDs change and WF2 needs re-patching. To future-proof, WF2 could
      query `/api/analyzer?range=0-10` at workflow start and resolve
      `name=AbuseIPDB` → `id` dynamically — but that's another node and
      left as a Phase-11 polish item.
    - The Cortex bearer for SOC-LAB org admin (cortex-user) is the same
      one B1 documented, value unchanged. WF2 uses n8n credential id
      `HK1qH743oIbnpSbk` (Header Auth, `Cortex Bearer`).

2026-05-10 — incident-mgmt (VM_B1) — Cortex analyzers verified + OTXQuery hang fixed; flagging two A1-side bugs
  Done:
    - Re-verified all 4 Cortex analyzer instances in SOC-LAB org via the
      orgadmin key (cortex-user, /api/analyzer):
        AbuseIPDB           (defId=AbuseIPDB_2_0)            key set
        VirusTotal_GetReport (defId=VirusTotal_GetReport_3_1) key set
        OTXQuery            (defId=OTXQuery_2_0)             key set
        MISP                (defId=MISP_2_2)                 key set
      They were already registered with valid free-tier keys from earlier
      Phase 4. Soc-core's "AbuseIPDB worker not registered" report was a
      naming-version mismatch, not a missing instance — see Bug A1-#1 below.
    - Live-tested AbuseIPDB (8.8.8.8 → Whitelisted/CDN/score 0) and
      VirusTotal_GetReport (8.8.8.8 → 0/92 with 200 resolutions). Both
      return in <5s. SUCCESS.
    - OTXQuery was hanging forever. Root cause: upstream otxquery.py
      had no HTTP timeout, and the `passive_dns` + `url_list` sections
      return massive payloads for popular IPs (millions of records for
      8.8.8.8) that never finish. The analyzer's blanket `except Exception`
      then masked the real error.
    - Patched /home/vboxuser/soc-project/thehive-cortex/cortex/Cortex-Analyzers/
      analyzers/OTXQuery/otxquery.py on the host:
        - timeout=15 on every requests.get (all 4 callsites)
        - otx_query_ip: dropped passive_dns + url_list sections (kept
          general, reputation, geo, malware); wrapped each section in its
          own try/except so a single section failure no longer kills the
          whole job; partial_errors[] reported when any section fails.
    - Rebuilt soc-cortex:4.0.1-analyzers image and recreated the cortex
      container (no host downtime — TheHive recovered automatically).
      Verified via /api/analyzer/{id}/run on 198.51.100.1 (4 pulses,
      0 malicious) and 185.220.101.1 (50 pulses, 1 malicious, known Tor
      exit). Final smoke test 3/3 PASS in ~30s each. SUCCESS.
    - WF3 (Daily Digest) error exec 274 inspected. NOT WF3-fatal but should
      be fixed before Phase 11 — see Bug A1-#2 below.

  Bugs flagged for VM_A1 (soc-core owner) — both unrelated to today's
  cortex work but discovered while debugging it:

    Bug A1-#1 (WF2 Cortex enrichment): "fetch observables" path now works
              after the GET→POST/query fix, but the analyzer dispatch is
              still calling `AbuseIPDB_1_0` (v1.0) which doesn't exist on
              this Cortex install. The registered instance is
              `AbuseIPDB_2_0` (v2.0). Cortex logs show:
                  warn POST /api/analyzer/AbuseIPDB_1_0/run returned 404
              Fix: WF2 should call AbuseIPDB by its INSTANCE id (not the
              definition id with version suffix). Either use the instance
              UUID 6b1c7570c74b55db697a69aa2c719b4f or POST to the
              analyzer-by-type endpoint. Same applies to any other
              hardcoded analyzer names — pull live definition IDs at
              workflow build time.

    Bug A1-#2 (WF3 Daily Digest): "TheHive: create digest case" node fails
              JSON parse with "Bad control character in string literal at
              position 280" on exec 274 (2026-05-10 07:00 UTC). Cause: the
              JSON body template interpolates `{{ $('Aggregate queued
              buckets').item.json.by_rule }}` and `.lines` directly into a
              JSON string field. Those values contain real newlines / tabs
              that aren't JSON-escaped. The existing `.replace(/\n/g,' ')`
              only handles the AI-summary field and doesn't cover the
              by_rule/lines fields. Fix: either JSON.stringify the entire
              description and slice the outer quotes, or apply
              `.replace(/[\x00-\x1f]/g,' ')` to every interpolated
              field before injecting into the body. Note: the same
              Ollama node also wrapped its own error object into the
              output (since neverError: true), which masked the actual
              failure. Recommend adding an If-node after Ollama to
              short-circuit on missing $json.response.

    Bug A1-#2b (WF3 historical): exec 253 (2026-05-08 07:00) failed at
              the same node with "host is unreachable" — that was during
              the 6-hour ES outage on B1, already resolved by the
              permanent boot orchestration fix on 2026-05-10. Not a
              standing issue.

  Pending / known gaps (unchanged from prior log):
    - Phase 11 (PFE rapport) — only open phase. Now unblocked.
    - TheHive Platinum trial expires 2026-05-14 — assess Community
      fallback impact (org-managed users may need to consolidate to
      the 2-user license cap).
    - Leaked secrets that hit public soc-shared CLAUDE.md earlier need
      rotation (THEHIVE_BOT_API_KEY, SOCADMIN_KEY, CORTEX_USER_API_KEY,
      MISP_API_KEY).

  Things to know:
    - The otxquery.py patch is in soc-project on B1 — that tree is
      NOT in soc-shared (it's a vendored upstream clone). If A1 has its
      own cortex install for any reason, the patch needs to be re-applied
      there. B1's cortex is the canonical one for SOC-LAB so this isn't
      blocking.
    - Cortex analyzer call latency: AbuseIPDB ~1-3s, VT_GetReport ~3-5s,
      OTXQuery ~25-35s (still slow due to the 4 sequential section
      calls; could parallelize later if needed). All within reasonable
      enrichment SLAs.
    - SOC-LAB org now has these analyzer instance IDs (use these when
      referencing analyzers in workflows, not the *_2_0 definition ids):
        AbuseIPDB             = 6b1c7570c74b55db697a69aa2c719b4f
        VirusTotal_GetReport  = 8cac1902c2e7879f2d258a3bfc7ba1f5
        OTXQuery              = eb540d51238c71257ca2713bafd84d2e
        MISP                  = b20c109bfc76cda9b8690ebf77f77931

2026-05-10 — soc-core (VM_A1) — Phase 10 COMPLETE: 7 bug fixes + auto_block end-to-end + observable pipeline fix
  Done:
    - Phase 10 closed. All 13 custom SOC rules either (a) fired and pipelined
      to a TheHive case with observable + Cortex routing this session,
      (b) verified in previous sessions, or (c) deferred (SOC-008/009/010/013
      require local execution on B2 which is out of scope for this pass).
    - 7 distinct bugs found and patched this Phase:
        Bug #1: SOC-001/002/012 KQL `process.name:"sshd"` → did not match
                because OpenSSH 9.x renames the per-connection child to
                `sshd-session`. Broadened to `(sshd OR sshd-session)`.
        Bug #2: SOC-005 KQL had bare `*<script*` token — invalid KQL
                ("Expected ),AND,OR but '<' found"). Dropped the bare clause;
                URL-encoded variants (*%3Cscript*) cover the actual HTTP
                traffic anyway.
        Bug #3: SOC-011 KQL `*"bash -i"*` with wildcards around quoted
                phrase = invalid KQL syntax. Rewrote without wildcards:
                `message:"bash -i" or message:"/dev/tcp/" or ...`.
        Bug #4: SOC-012 same OpenSSH 9.x sshd→sshd-session rename (variant
                of Bug #1, separate rule).
        Bug #5: WF1 "Build canonical payload" — source_ip resolution.
                When the alert host has multiple NICs (B2's host.ip is an
                array of 5 entries: lo IPv6, ZeroTier IPv4, ZeroTier IPv6,
                etc.), the array serialized as JSON-string-of-array which
                broke the downstream JSON body interpolation in TheHive
                create-case. Fixed with an IIFE that picks the first IPv4
                from the array.
        Bug #6: SOC-011 KQL was firing on its own past failure messages
                stored in soc-core's event-log indices (rule logs contain
                the trigger pattern as plain text). Fixed by prepending
                `not host.name:"soc-core"` to the rule query.
        Bug #7: WF2 "TheHive: fetch observables" was GETting
                `/api/v1/case/{id}/observables` which TheHive 5 doesn't
                expose (returns 404 with `{"type":"NotFoundError"...}`).
                Cortex enrichment had been silently broken since the start —
                the 404 response had no `dataType` field so Switch routed
                every observable to "Skip: unsupported dataType". Patched
                to POST `/api/v1/query` with `getCase + observables` shape.
                Verified: now correctly fetches the observable and routes
                to AbuseIPDB analyzer node.
    - New WF1 node `wf1-14b-add-observable` (n8n-nodes-base.httpRequest)
      inserted between "TheHive: create case" and "Correlation Engine:
      set_case". Creates a source_ip observable (dataType=ip, tlp=2, pap=2,
      ioc=true, sighted=true, tags=[soc-auto, source.ip]) on every new
      TheHive case. Verified populates the Observables tab on cases
      #17/18/19 created this session.
    - Auto_block path verified end-to-end via SOC-006 from .52 (kali):
      apache.access logs ingest → SOC-006 rule fires → WF1 create_new path
      → ML score + Ollama summary + TheHive case #19 + observable creation
      → SSH from A1 → B2 → `iptables -I INPUT -s 192.168.1.52 -j DROP`
      (exit 0) → TheHive auto-block comment. Verified A2 lost B2 access
      (curl timed out, http=000), A1 retained access via soc-response key
      (selective source-IP blocking confirmed). Cleaned up the iptables
      rule post-test (`iptables -D INPUT -s 192.168.1.52 -j DROP`).
    - Closed 19 stale correlation buckets at session start (some 5 days
      old). Bucket window is 30 min so they were dead anyway but they
      showed up in /state and made debugging noisy.
    - Verified TheHive native webhook notifier IS firing — case-creation
      now triggers WF2 automatically (exec 284 fired ~3 min after case
      #18 was created, no bridge cron involvement). Bridge-cron path
      from Phase 8 remains as backup.
  Phase 10 cases (rapport-grade):
    #13 SOC-001 SSH brute (sev 4)        — prior session
    #14 SOC-005 XSS (sev 2)              — prior session
    #15 SOC-011 reverse shell (sev 4)    — prior session (Bug #5 confirmed live in case title: "from 10.0.2.15")
    #17 SOC-005 XSS (sev 2)              — this session, observable validated post-Bug #7 fix
    #18 SOC-012 SSH from non-mgmt (sev 4) — this session, escalated by SOC-006 absorption
    #19 SOC-006 cmd injection (sev 3)    — this session, auto_block fired
    SOC-007 absorbed into bucket via correlation queue path (no case)
  Latency observations:
    - Fast WF1 path (no NLP, correlation = add_to_existing/queue): 0.4-2.8s
    - Slow WF1 path (Ollama summarize on create_new): 60-632s
    - Worst case: SOC-005 first this session = 614s (Ollama cold);
      subsequent SOC-006 = 135s (Ollama warm).
    - SOC-006 second batch: 36 alerts → 1 case (Kibana action summary mode
      collapses to one webhook + correlation absorbs duplicates).
  Pending / known gaps:
    - Cortex AbuseIPDB analyzer worker not registered ("worker AbuseIPDB_1_0
      not found"). Requires user to add free-tier API key in Cortex UI
      (Org SOC-LAB → Analyzers → enable AbuseIPDB_1_0 with key). Same
      applies to VirusTotal_GetReport_3_1 and OTXQuery_2_0.
    - SOC-008 (file mod), SOC-009 (web shell), SOC-010 (sudo escalation),
      SOC-013 (data exfil) not run — these require local execution on B2
      victim. Deferred to a focused B2 attack session.
    - Daily Digest (WF3) errored at 07:00:01Z today (exec 274 status=error).
      Not a blocker but should be inspected before Phase 11 doc.
    - A2 (kali) has no internet — `apt install` failed. Set up apt offline
      cache or proxy through A1 if more tools needed.
  Things to know:
    - A2 sshd was disabled at session start; user enabled with
      `sudo systemctl enable --now ssh` on A2 console. Now reachable.
    - SOC-012 fires only on successful SSH login from a non-management
      host. Triggering it requires sshing FROM .52 TO .53 (not from A1).
      Used expect on A2 to chain through password auth.
    - WF1 + WF2 saved locally to ~/soc-project/n8n/workflows/ on A1.
    - For demos: re-fire SOC-006 from a fresh source IP after closing any
      existing bucket from that source — same-source within 30 min hits
      add_to_existing/escalate_existing branch and skips the auto_block
      node.

2026-05-08 — victim-lab (VM_B2) — SSH brute-force log shape fix (PerSourcePenalties off)
  Done:
    - Closed the SOC-001/SOC-002 detection-blind-spot identified in
      VM_A1's Phase 10 entry. OpenSSH 9.x ships PerSourcePenalties on by
      default (effective config showed authfail:5, noauth:1, ... before
      change). Repeated failed-auth attempts caused sshd to "drop
      connection ... penalty: failed authentication" instead of emitting
      standard "Failed password" lines, which Elastic's system
      integration does not parse to event.outcome:"failure" — so the
      SOC-001/SOC-002 KQL on event.outcome never matched.
    - Appended `PerSourcePenalties no` to /etc/ssh/sshd_config.d/99-soc-lab.conf
      with an inline comment explaining why and pointing to VM_A1's note.
      `sshd -t` validated; restarted ssh.socket + ssh.service.
    - Verified effective config: `sshd -T | grep persourcepenalties` →
      `persourcepenalties no`. Banner check on 127.0.0.1:22 →
      "SSH-2.0-OpenSSH_10.2p1 Ubuntu-2ubuntu3.2".
    - Snapshot updated at ~/soc-project/ssh-and-vsftpd/99-soc-lab.conf.
    - Pre-flight Phase 5 service health check before the change:
      apache2 / mariadb / vsftpd / elastic-agent / suricata all active;
      ssh.service "inactive" but ssh.socket active (socket-activation —
      expected design). All green.
  Pending / known gaps:
    - Behavioral test (i.e. "does an actual brute-force attempt now emit
      Failed password lines that the system integration parses?")
      deferred to VM_A1 — they're the ones who'd re-run SOC-001/002 to
      confirm the rules fire. From this side `sshd -T` is sufficient
      evidence the kernel-level behavior changed.
    - VM_B1 (TheHive + Cortex + MISP) was unreachable on all service
      ports as of this session — host pings, nothing on :9000/:9001/:443.
      Either docker stack didn't auto-start after a B1 reboot or
      services are stopped. Flagged to user; B1's own session needs to
      bring it back. Until then WF1's TheHive create node will 5xx and
      bridge crons will idle.
  Things to know:
    - Drop-in is the right place for this — main /etc/ssh/sshd_config
      stays distro-default and the lab-specific tweaks live in
      99-soc-lab.conf, which already had PasswordAuthentication=yes and
      PermitRootLogin=no with a "DO NOT COPY TO PRODUCTION" warning.
    - If the demo wants to show PerSourcePenalties protection later
      (i.e. "see how OpenSSH 9.x mitigates brute-force"), simply remove
      the new line from the drop-in and restart ssh.

2026-05-07 (latest) — victim-lab (VM_B2) — Active-response wired + post-test cleanup
  Done:
    - Created soc-response user (uid 1004, gid 1004, /home/soc-response,
      /bin/bash, GECOS "SOC active-response (n8n auto-block)"). passwd -l
      to lock the password — SSH key auth is the only login path.
    - Installed VM_A1's ed25519 public key (ssh-ed25519 AAAAC3...sAZ
      soc-response@soc-core) into /home/soc-response/.ssh/authorized_keys
      (mode 600, owner soc-response). .ssh dir is 700.
    - Created /etc/sudoers.d/soc-response (mode 440, owner root) with
      `soc-response ALL=(root) NOPASSWD: /usr/sbin/iptables, /sbin/iptables`.
      Validated with `visudo -cf` before move. Both paths are listed
      because sudo policy matches the literal command path the caller
      typed; on this Ubuntu both symlink to /etc/alternatives/iptables →
      /usr/sbin/xtables-nft-multi but n8n's WF1 SSH node may invoke either.
    - Local sanity test passed: `sudo -u soc-response sudo -n
      /usr/sbin/iptables -L INPUT -n` and the /sbin variant both succeed
      without prompting.
    - Cleared the iptables DROP -s 192.168.1.50 -j DROP rule that VM_A1's
      Phase 10 SQLi attack had auto-installed (rule that proved the
      auto-block path works end-to-end). VM_A1 reachable again from B2
      (~9-26ms over ZeroTier).
    - Updated phase_5 status line: appended the active-response wiring,
      removed the now-obsolete "DEFERRED until VM_A1 phase 3 MITRE Tagger"
      tail (Phase 3 re-architecture dropped that whole approach).
  Pending / known gaps:
    - SSH-based rules SOC-001/002 remain partially blind to OpenSSH 9.x
      brute-force because PerSourcePenalties drops repeated auth attempts
      with a "penalty: failed authentication" message that doesn't map to
      event.outcome:"failure" via the system integration. Three options
      noted in VM_A1's Phase 10 entry; if the rapport demo wants those
      rules to fire, easiest is `PerSourcePenalties no` in
      /etc/ssh/sshd_config on this VM.
    - Phase 10 itself stays in-progress on VM_A1 until they declare it
      done; nothing more for B2 until Phase 11 (rapport docs).
  Things to know:
    - Public key is non-secret. Private half stays on VM_A1 only
      (~/.ssh/soc_response, no passphrase, used by n8n cred id
      VqpfYno0QpnoseF6).
    - If someone needs to revoke the auto-block path: remove
      /etc/sudoers.d/soc-response (or replace authorized_keys).
      User stays usable for any future expansion.
    - When WF1's auto-block fires it leaves an INPUT DROP rule that
      survives reboot only if iptables-persistent is installed (not on
      this VM). For lab purposes the rule is in-memory only — ok for
      demo runs.

2026-05-07 (later) — soc-core (VM_A1) — Phase 10 first full green end-to-end run
  Done:
    - Patched the two n8n credentials via `PATCH /api/v1/credentials/{id}` —
      contradicting B1's note that no PATCH exists. Live n8n on this build
      DOES expose PATCH. TheHive Bearer (Ux32rgVuHoXKc1GY) and Cortex Bearer
      (HK1qH743oIbnpSbk) now hold the real soc-bot and cortex-user keys.
      Direct curl to TheHive case-create with the new bearer returned 201
      and case ~36896 (verification only).
    - Patched WF1 Ollama node `num_predict` 80 → 30 via `PUT /workflows/{id}`
      (had to strip `binaryMode` from settings — public-API schema only
      accepts a closed set of settings keys).
    - Pulled gemma3:4b for the model swap experiment. Benchmarked at
      ~0.47 tok/s warm vs llama's ~0.3 tok/s — not the 3× win the user
      expected. Decision: skip the model swap, stick with llama3.1:8b plus
      num_predict=30. gemma3:4b stays on disk for future experimentation.
    - Discovery #1 (testing): SSH brute force at modern OpenSSH 9.x
      triggers PerSourcePenalties immediately. The auth.log lines are
      "drop connection ... penalty: failed authentication", which the
      Elastic Agent system integration does NOT parse to
      event.outcome:"failure". Result: SOC-001's KQL won't match this
      attack pattern as written. Pivoted to SOC-003/004 (HTTP SQLi via
      Apache integration) which are shape-compatible.
    - Discovery #2 (testing — root cause of the 3 failed exec rounds 13, 15,
      17): the action body template on ALL 13 SOC rules used non-existent
      Mustache variables. `{{rule.rule_id}}`, `{{rule.severity}}`,
      `{{rule.risk_score}}` don't exist in the SIEM summary action context;
      they render to empty strings, producing malformed JSON
      (`"risk_score":,`). That cascaded to ES fetch (term query with empty
      rule_id → 0 hits), Build canonical (all-null fields), and TheHive
      create (BadRequest because tags[2] was the empty rule_id).
      The CORRECT paths are `{{rule.params.ruleId}}`, `{{rule.params.severity}}`,
      `{{rule.params.riskScore}}`. Bulk-patched all 13 SOC rules via
      `PATCH /api/detection_engine/rules` with the corrected body and per-rule
      `auto_block` values from the rule's `meta.auto_block` field. After
      that, every node downstream got real data.
    - Closed all phantom buckets in correlation engine (left over from the
      failed rounds, which had set case_id=~45168 to a non-existent case)
      via `POST /5002/close` so the next /correlate call would return
      action=create_new and force fresh TheHive case creation.
    - Fired 6 fresh SQLi requests (lambda..pi series). SOC-003 rule fired
      at 23:34:21Z with 12 active alerts and the action succeeded. WF1
      execution 22 ran all 14 nodes successfully:
        Webhook → ES → Kibana → Build canonical → /correlate (action:create_new
        bucket_id:10) → ML score 0.6335 → Ollama summary 200s → TheHive
        create case ~40984800 number 11 → set_case → IF auto_block(true) →
        SSH iptables -I INPUT -s 192.168.1.50 -j DROP on victim-lab
        (return code 0) → TheHive comment "Auto-block executed: …".
      The case in SOC-LAB org has full payload: severity=HIGH, tags
      [PFE, SOC-004, SOC-Lab, mitre:T1190, mitre:TA0001], assignee=
      soc-bot@thehive.local, AI summary in description.
    - Bridge fired: WF2 (Cortex enrichment) execution 24 at 23:38:01Z —
      B1's cron-cases-to-wf2.sh polled and posted the new SOC-LAB case to
      /webhook/thehive within 60s of creation, exactly as designed.
    - Auto-block confirmed working: VM_B2 became unreachable from this VM
      immediately after WF1 finished; iptables rule survives on B2.
  Pending / known gaps:
    - VM_A1 (this VM) is currently DROP-blocked at VM_B2's iptables.
      To clear so re-tests can run, on VM_B2 run:
        sudo /sbin/iptables -D INPUT -s 192.168.1.50 -j DROP
      The B2 session can do this from its local terminal.
    - SSH-based rules (SOC-001/002) need either: (a) PerSourcePenalties
      disabled on B2's sshd (`/etc/ssh/sshd_config: PerSourcePenalties no`),
      or (b) the rule query updated to also match
      message:"penalty: failed authentication" lines, or (c) skip them
      and rely on SOC-003..013 for the rapport demo.
    - Phase 10 still has Phase 11 to follow (rapport documentation).
    - WF1 Ollama 200s for 30 tokens on this run (vs 339s previously) —
      timing is variable but well under the 600s timeout. No more changes
      needed there.
  Things to know:
    - SIEM action variables for `siem.queryRule` summary mode:
        rule.name        → display name (works)
        rule.id          → rule UUID (the kibana-internal uuid)
        rule.tags        → array
        rule.params.ruleId      → user-defined SOC-NNN
        rule.params.severity    → low/medium/high/critical
        rule.params.riskScore   → integer
        rule.params.threat      → MITRE array (use {{{...}}} triple-stash)
        context.results_link    → Kibana deep link (use {{{...}}})
        context.alerts          → JSON-stringified alerts array
    - n8n public API DOES expose PATCH on credentials and PUT on workflows,
      contrary to B1's earlier note. PUT requires sanitized `settings`
      (only the documented keys: executionOrder, saveExecutionProgress,
      saveDataSuccessExecution, saveDataErrorExecution; binaryMode etc.
      are rejected as "additional properties").
    - Correlation engine bucket reset: any time the engine has stale
      case_id pointers (from earlier failed runs), close them via
      POST /5002/close {bucket_id:N} or the workflow will silently 404 on
      the "add to existing case" branch.

2026-05-07 — incident-mgmt (VM_B1) — Phase 8/9 wiring on B1 done; both VMs now wired
  Done:
    - Recovered Cassandra: corrupt commit-log segment from Apr 30 22:21
      (CommitLog-7-1777587684535.log, paired _536) blocked startup with
      "Could not read commit log descriptor". Moved both segments to
      /tmp/cassandra-bad-commitlogs-20260507/ (lab — no in-flight data
      depended on those Apr 30 writes; SSTables hold all real data).
      Cassandra now healthy on 9042, TheHive joined cleanly.
    - TheHive: created SOC-LAB organisation, two new users:
        soc-bot@thehive.local      profile=analyst   (case-create bearer for n8n WF1)
        socadmin@thehive.local     profile=org-admin (manages SOC-LAB org config)
      API keys (in ~/soc-project/.env.local on B1):
        THEHIVE_BOT_API_KEY=rtHZuTw01P0UwaeDKmcT04bBQZmxvlGM
        SOCADMIN_KEY=qr5vp1AdFOaEUzWt8E6Qj5R7F2QQg4V/   (operational, not committed)
    - Cortex: reused existing cortex-user bearer for n8n cred (already in
      .env.local as CORTEX_USER_API_KEY=6MWnt7E3FdH0muqjoG6Xyd+5msDQf4S2).
      Same key TheHive uses via TH_CORTEX_KEYS — fine for a lab.
    - Handed both bearers to VM_A1 owner for UI paste into n8n cred ids
      Ux32rgVuHoXKc1GY (TheHive) and HK1qH743oIbnpSbk (Cortex). UI edit
      keeps id stable; n8n public API exposes no PATCH/PUT for creds.
    - TheHive notifier (org-level): tried the native notification framework
      via static config.
        - Endpoint registered at system level via bind-mounted partial
          /etc/thehive/application.conf:
            notification.endpoints = [{name="n8n-soc", type="webhook",
              url="http://192.168.1.50:5678/webhook/thehive", ...}]
          Confirmed visible in /api/config after restart.
        - Org-level rule persisted via PUT /api/v1/config/organisation/notification
          under socadmin (org-admin in SOC-LAB):
            {value:{endpoints:[...], items:[{trigger:{name:"AnyEvent"},
                                            endpoint:"n8n-soc",
                                            filter:{}, enabled:true,
                                            delegate:true}]}}
        - But: NotificationActor logs "Starting fixed thread pool with 2
          threads" at boot and then never emits anything on case create.
          Tried trigger names AnyEvent, CaseCreated; tried both shapes
          (rules-as-array vs rules-inside-{endpoints,items}); all silent.
          Audit records ARE created (verified via listAudit query) and the
          webhook URL IS reachable from inside the container (curl 200).
          Time-to-fix > value. Documented and moved on.
    - Workaround: polling bridge (~/soc-project/thehive-cortex/cron-cases-to-wf2.sh)
      runs every minute, queries listCase filter _gt _createdAt, posts each
      new case in TheHive's native notifier envelope shape
      ({operation:"create", objectType:"Case", objectId, object:<full case>})
      to http://192.168.1.50:5678/webhook/thehive. Verified end-to-end:
      case=~32880 wf2=200. Same pattern as MISP→WF4.
    - MISP→WF4 (Phase 9 deferred): polling bridge
      ~/soc-project/misp/cron-publish-to-wf4.sh, every minute, uses
      /events/restSearch with publish_timestamp filter and sends matching
      Event JSONs to /webhook/misp. First-run seeds state to "now" (avoids
      backfilling 1606 historical feed events into WF4). MISP 2.5 has no
      native single-URL outbound webhook, only ZeroMQ/email — bridge is
      simpler and survives restarts cleanly.
    - Bind-mount ./thehive/conf/application.conf → /etc/thehive/application.conf
      added to docker-compose.yml so the notifier endpoint config persists
      across container recreates.
    - Cron now has 3 jobs:
        */6h MISP feed fetch
        every-1m MISP→WF4 bridge
        every-1m TheHive→WF2 bridge
  Pending / known gaps:
    - VM_A1 owner: paste the two bearers into n8n UI (cred ids
      Ux32rgVuHoXKc1GY, HK1qH743oIbnpSbk), save. After that, re-fire WF1
      smoke from A1 — it should now succeed at the TheHive case-create
      step (case lands in SOC-LAB, then bridge picks it up within 60s and
      fires WF2 enrichment).
    - VM_B2: still need to install ~/.ssh/soc_response.pub for soc-response
      user + /etc/sudoers.d/soc-response (NOPASSWD iptables) so WF1's
      auto_block path works.
    - TheHive native notifier: not chasing further. If it ever starts
      working (TheHive update, config tweak, etc.), the bridge will
      coexist harmlessly — WF2 is idempotent on duplicate POSTs (n8n
      executions are independent).
    - Phase 10 (end-to-end testing) and Phase 11 (rapport) still pending.
  Things to know:
    - The notifier bridge timestamp uses ms epoch ($(date +%s)*1000) —
      matches TheHive's _createdAt field. `date +%s%3N` returned wrong
      output on this Ubuntu (full nanos not truncated to 3 digits).
    - SOC-LAB org now exists in TheHive. WF1's TheHive case-create node
      should NOT need an X-Organisation header since soc-bot's
      defaultOrganisation=SOC-LAB. If it sets one explicitly, it must be
      "SOC-LAB" exactly.
    - Don't run socadmin's API key from WF1 — it has org-admin perms; use
      THEHIVE_BOT_API_KEY (analyst) for n8n's case-create flow.

2026-05-06 — soc-core (VM_A1) — Phase 8 (WF2) + Phase 9 (WF3,WF4,WF5) complete + status report
  Done:
    - Created Cortex Bearer credential (id HK1qH743oIbnpSbk, type httpHeaderAuth,
      placeholder bearer until VM_B1 boots).
    - Authored, imported, activated 4 more workflows via n8n public API:
        WF2 02-cortex-enrichment.json     id HYiSFNStG5zEG6ZA  webhook /webhook/thehive
        WF3 03-daily-digest.json          id Z1VpjJlhg2Skek1B  cron 0 8 * * *
        WF4 04-misp-rule-gen.json         id SbXmkucPC24njKwb  webhook /webhook/misp
        WF5 05-weekly-maintenance.json    id ekXEZb2PYaxQt7vv  cron 0 4 * * 1
      All four files in ~/soc-project/n8n/workflows/. All ACTIVE per
      GET /api/v1/workflows. All 3 webhook URLs respond HTTP 200.
    - Smoke tests:
        WF4 — synthetic MISP payload triggered the workflow. Webhook → Extract
              indicators (IPs/URLs/hashes) → Ollama format:json → after 600s
              timeout returned empty → Validate detected ok:false → IF routed
              correctly to "log generation failure" → fails at TheHive POST
              (B1 offline). Pipeline logic correct; CPU-only Ollama needs
              ~22 min for 400-token JSON, exceeding 10-min timeout. Workflow
              gracefully handles failure case.
        ML /train { days: 1 } — direct probe returned { ok: true,
              source: "synthetic:1000", trained_at: 1778078256 }. WF5's first
              step works.
    - Authored ~/pfe/docs/08-phase8-soar-integration.md and
      ~/pfe/docs/09-phase9-adaptive-intelligence.md following the same
      structure as the other phase docs (Objective / Components built / Decisions
      / Verification / Known gaps / Where this fits).
    - Authored ~/pfe/docs/REPORT.md — single-page status: TL;DR, phase status,
      what works, what doesn't, manual steps for the user when VMs come back,
      key file locations, open architectural questions, scoreboard. This is
      the file to read first.
    - Updated ~/pfe/docs/README.md index to reference 08, 09, REPORT.md.
  Pending / known gaps (unchanged from prior entry, restated for clarity):
    1. ON VM_B1 (TheHive + Cortex + MISP): generate API keys, update n8n
       credentials Ux32rgVuHoXKc1GY (TheHive) and HK1qH743oIbnpSbk (Cortex).
    2. ON VM_B2: install ~/.ssh/soc_response.pub for soc-response user +
       /etc/sudoers.d/soc-response for NOPASSWD iptables.
    3. ON VM_B1: add TheHive notifier → http://192.168.1.50:5678/webhook/thehive
       (wires WF2). Add MISP server entry → http://192.168.1.50:5678/webhook/misp
       (wires WF4).
    4. WF4 Ollama timeout: either drop num_predict to 150 or move Ollama to GPU.
    5. Phase 10 (end-to-end testing) and Phase 11 (rapport documentation) still
       pending.
  Things to know:
    - All 5 workflows follow the n8n-skill convention: HTTP Request nodes with
      authentication=genericCredentialType + genericAuthType=httpBasicAuth/Header,
      SSH node with explicit authentication: privateKey, webhook payloads
      accessed via $('node').item.json.body.field, Switch by case parameter,
      IF using 2.2 typeVersion with looseTypeValidation when comparing booleans.
    - Cron timezone: Africa/Tunis on WF3 and WF5 (project setting).
    - n8n's saveDataSuccessExecution=all + saveDataErrorExecution=all is on
      every workflow so executions are inspectable via /api/v1/executions/{id}
      ?includeData=true for debugging.
    - WF4 deliberately imports rules with enabled=false — analyst-in-the-loop
      guardrail against LLM hallucinations producing too-broad queries.

2026-05-06 — soc-core (VM_A1) — Phase 8 complete-with-deferred-thehive: SOAR Workflow 1
  Done:
    - n8n owner account set up via UI (one-time first-run form). Required N8N_SECURE_COOKIE=false
      env var added to /etc/systemd/system/n8n.service so the UI works over plain HTTP on the
      ZeroTier overlay (no TLS in the lab). Public API key issued by user, stored in
      ~/soc-project/.env.local as N8N_API_KEY.
    - Generated ed25519 SSH keypair ~/.ssh/soc_response (no passphrase) for the auto-block
      path. Public key NOT yet installed on VM_B2 (offline) — that's a deferred manual step.
    - 3 n8n credentials created via POST /api/v1/credentials:
        Elasticsearch (elastic)         id=cTJMkUYUxWVtlD8K  type=httpBasicAuth
        TheHive Bearer                  id=Ux32rgVuHoXKc1GY  type=httpHeaderAuth (placeholder)
        VM_B2 soc-response SSH          id=VqpfYno0QpnoseF6  type=sshPrivateKey
      Placeholder bearer for TheHive will be replaced once VM_B1 boots and the user generates
      a real API key in TheHive UI.
    - Workflow 1 authored at ~/soc-project/n8n/workflows/01-alert-pipeline.json (19 nodes):
        [Webhook (POST /webhook/elastic-alert)]
        → ES: fetch signals (.alerts-security.alerts-default, last 15m, by rule_id)
        → Kibana: fetch rule definition (so we can read threat[].tactic.id)
        → Set: build canonical payload (rule_id/name/severity/risk_score/auto_block/
                source_ip [w/ threshold-rule fallback]/mitre_tactic_id/mitre_technique_id)
        → POST /correlate
        → Switch by action: 5 outputs
            • suppress    → NoOp (end)
            • queue       → NoOp (end, daily-digest will pick it up)
            • add_to_existing  → TheHive PATCH /api/v1/case/{case_id} (append note)
            • escalate_existing → TheHive PATCH (severity=4, flag=true, escalation note)
            • create_new  → ML /score → Set anomaly → Ollama /api/generate (llama3.1:8b,
                            num_predict=80, timeout 600s) → TheHive POST /api/v1/case
                            → POST /correlate/set_case → IF auto_block?
                                → true: SSH (sshPrivateKey credential) iptables -I INPUT
                                        -s {source_ip} -j DROP on 192.168.1.53
                                        → TheHive POST .../comment to log the action
                                → false: NoOp end
      Imported via POST /api/v1/workflows → id dKSF2AU9E3k9i25p. Activated via
      POST /api/v1/workflows/{id}/activate. Webhook URL live: http://192.168.1.50:5678/webhook/elastic-alert
    - Bug found and fixed during smoke test: HTTP Request nodes had
      authentication=predefinedCredentialType + nodeCredentialType=httpBasicAuth which
      n8n silently ignored (auth headers never sent → ES returned "missing authentication
      credentials"). Correct shape is authentication=genericCredentialType +
      genericAuthType=httpBasicAuth/httpHeaderAuth. All 6 auth-bearing HTTP Request nodes
      patched (script in this session). Re-imported via PUT.
    - Bug found and fixed: n8n SSH node v1 requires an explicit "authentication: privateKey"
      parameter to recognise the sshPrivateKey credential. Without it, activation rejects
      the workflow with "Missing required credential: sshPassword". Patched.
    - Smoke tests: 5 webhook executions exercised through n8n's executions API. Final
      execution (id=5) ran the create_new path end-to-end:
        Webhook ✓ → ES (auth ok) ✓ → Kibana (auth ok, rule fetched) ✓ → Set ✓
        → /correlate ✓ (action=create_new, bucket_id=7) → Switch ✓ → ML /score ✓
        (anomaly_score=0.6445 is_anomaly=True) → Append ✓ → Ollama timed out at 120s →
        TheHive create case ✗ (host unreachable, VM_B1 offline). After patching Ollama
        timeout to 600s and num_predict to 80, Ollama itself probes clean directly
        (curl test returned "Affirmative" 4 tokens).
  Pending / known gaps (when VM_B1 + VM_B2 come back online):
    1. ON VM_B1 (TheHive): generate API key in UI → User Profile → API Key, then
       n8n_manage_credentials update id=Ux32rgVuHoXKc1GY data.value="Bearer <key>"
       (or via UI: Credentials → "TheHive Bearer" → paste).
    2. ON VM_B2: install /home/vboxuser/.ssh/soc_response.pub on victim-lab as the
       soc-response user's authorized_keys; create soc-response user if it doesn't exist;
       add /etc/sudoers.d/soc-response with NOPASSWD: /sbin/iptables.
       Pubkey content (safe to scp): ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIL0Fc6G/gJl/R2Ti00NGTykbiiNORXZW67ZxA4Dc9sAZ soc-response@soc-core
    3. Re-run end-to-end test by exercising a real SOC rule (e.g. brute SSH from
       192.168.1.51 to VM_A1 to trigger SOC-001) and verify full case creation.
    4. Workflows 2/3/4/5 (Cortex enrichment, daily digest, MISP→AI rule gen, weekly
       maintenance) still pending — Phase 8 only built Workflow 1 per the user's
       "5 separate workflows" architectural decision this session.
  Things to know:
    - The HTTP Request node mistake (predefinedCredentialType vs genericCredentialType)
      is easy to repeat on the next workflows. Always use genericCredentialType +
      genericAuthType for HTTP Basic / Bearer / Header auth.
    - The SSH node mistake (missing authentication: privateKey) is also easy to repeat.
      Always set it explicitly when using sshPrivateKey credential.
    - Webhook payload from Kibana is accessed via $('Webhook').item.json.body.field —
      NOT $json.field — because n8n nests webhook input under .body.
    - Ollama at 0.3 tok/s CPU-only: budget 1 second per token of expected output. The
      workflow's num_predict=80 + timeout 600s gives >7x headroom.
    - The placeholder TheHive bearer is intentionally NOT a real token; once a real
      token is set, the workflow's TheHive nodes will succeed.

2026-05-06 — soc-core (VM_A1) — Phase 7 complete: detection layer activated
  Done:
    - Set Kibana encryption keys (xpack.encryptedSavedObjects/reporting/security.encryptionKey)
      appended to /etc/kibana/kibana.yml — required by the actions framework before any
      connector can be created. Generated via `kibana-encryption-keys generate -q`. Restart
      clean.
    - Detection-engine signals index initialised (POST /api/detection_engine/index → ack).
    - Installed prebuilt Elastic SIEM rules pack: 1644 rules, 10 timelines (rules_installed:
      1644, timelines_installed: 10). All disabled by default — enable selectively in
      Phase 8/9 once SOAR pipeline is wired.
    - License: started 30-day trial via /_license/start_trial?acknowledge=true (was basic).
      REQUIRED for the .webhook connector type which is gated to Gold+ on basic. Decision
      defendable in rapport: trial unlocks production-equivalent feature surface for the
      PFE demo window. Fallback if trial expires: switch detection rules to a "noop"
      action and have n8n's Elasticsearch node poll .alerts-security.alerts-default
      (works on basic forever, adds polling latency).
    - Kibana action connector created: id 7c351a6c-4de6-4c07-8146-fa337033c735, name
      "n8n-soar-webhook", type .webhook, POST → http://192.168.1.50:5678/webhook/elastic-alert,
      hasAuth=false. n8n is on the same host so unauth localhost is fine; ufw blocks 5678
      from outside the ZT subnet.
    - Authored 13 SOC custom rules in NDJSON at ~/soc-project/kibana/soc-rules.ndjson
      (one rule per line). Each rule carries:
        * native MITRE threat[] — tactic + technique + subtechnique objects with refs
        * tags including "mitre:TA0XXX", "mitre:T1XXX", "auto_block" (where applicable),
          plus Elastic-convention "Domain: ...", "OS: Linux", "Tactic: ..." for grouping
        * meta: { auto_block, soc_layer, soc_id }
        * actions[] referencing the webhook connector with mustache body template:
            {"rule_id":"{{rule.rule_id}}","rule_name":"{{rule.name}}",
             "severity":"{{rule.severity}}","risk_score":{{rule.risk_score}},
             "auto_block":true,"results_link":"{{{context.results_link}}}"}
          (auto_block emitted only on the 5 critical rules). n8n workflow 1 will fetch
          full alert docs from .alerts-security.alerts-default by results_link / time
          window — webhook payload stays small.
    - Imported via POST /api/detection_engine/rules/_import?overwrite=true → 13 success,
      0 errors. Bulk-enabled via /api/detection_engine/rules/_bulk_action with filter
      tags:"SOC-Custom-13" → 13 succeeded, 0 failed.
    - Severity / auto_block matrix matches CLAUDE.md table:
        SOC-001 medium  | SOC-002 critical+block | SOC-003 medium      | SOC-004 high+block
        SOC-005 medium  | SOC-006 high+block     | SOC-007 low         | SOC-008 high
        SOC-009 critical+block | SOC-010 high    | SOC-011 critical+block | SOC-012 high
        SOC-013 high
    - Rule execution status (12 of 13 succeeded on first runs; SOC-011/012 just hadn't
      run yet because of 5m interval; SOC-008 reports partial failure — its index list
      includes logs-endpoint.events.file-* / auditbeat-* which don't exist on this
      cluster (no FIM agent yet). Benign: rule still runs against logs-system-* and
      will silently match nothing until FIM is enabled.
  Pending / known gaps:
    - VM_B2 (victim-lab) elastic-agent OFFLINE since 2026-05-05 21:09Z — last seen.
      Until B2 powers back up, Suricata + Apache rules (SOC-003..007, 009, 013) have no
      live data. Doesn't block Phase 7 closure but should be the first thing checked on
      VM_B2 next session.
    - SOC-008 silently no-op until FIM (file integrity monitoring) is added to system
      integration on agents. Defer to Phase 9 (adaptive intelligence) or earlier.
    - Trial license expires +30 days (2026-06-05). If still in lab use after that,
      either re-trial on a fresh cluster or fall back to n8n polling (see Done above).
    - Phase 8 next: build n8n Workflow 1 (webhook → /correlate → ML score → Ollama
      summarize → TheHive case create → optional auto-block via SSH to VM_B2).
  Things to know:
    - The .webhook connector is platinum/gold/trial-only on Elastic. If anyone reverts
      to basic, every detection rule that uses this connector will silently fail to send.
    - Kibana mustache in action params does NOT support {{rule.threat[0]...}} array
      indexing reliably — that's why MITRE IDs are pulled from rule.tags ("mitre:T...")
      in n8n workflow 1 instead of templated into the webhook body.
    - The 13 NDJSON rules are the canonical source of truth. Re-import with
      overwrite=true to redeploy after edits.
    - The webhook URL is unauthenticated. n8n test/run mode requires the workflow
      Active=ON for the production webhook path to accept POSTs — first POST when
      workflow is inactive returns 404. Phase 8 will activate it.

2026-05-05 — soc-core (VM_A1) — Phase 3 complete: SOAR + AI services with re-architecture
  Done:
    - n8n 2.18.5 systemd unit /etc/systemd/system/n8n.service running as vboxuser on :5678
      (env: N8N_HOST=0.0.0.0, WEBHOOK_URL=http://192.168.1.50:5678/, N8N_USER_FOLDER=~/.n8n).
      First-start migrations completed cleanly. HTTP 200 on /.
    - Ollama 0.22.1 + llama3.1:8b (4.9 GB, sha256 667b0c1932bc) pulled cleanly at ~5.4 MB/s
      (the network bottleneck from Phase 2 is gone). Systemd drop-in
      /etc/systemd/system/ollama.service.d/override.conf sets OLLAMA_KEEP_ALIVE=24h so the
      model stays warm. Cold load is ~5 min on this disk (mmap=false, ~17 MB/s tensor read);
      warm-path generation runs at ~0.3 tok/s CPU-only — slow but functional for batch SOAR
      use. First curl probe MUST use --max-time >=600 or the load is canceled mid-load.
    - Memory budget fix: VM has 13 GB total; with ES@4G + Kibana + Logstash + agentbeats,
      Ollama refused to load (needs 4.8 GB free). Added 8 G swapfile (/swapfile, persistent
      via /etc/fstab) AND dropped ES heap 4G→2G via /etc/elasticsearch/jvm.options.d/heap.options
      (backup at heap.options.bak-20260505). ES recovered to yellow (single-node, 72 primaries
      active, 29 unassigned replicas as expected).
    - Re-architecture decision (defendable for the rapport):
        DROPPED MITRE Auto-Tagger — Kibana detection-engine UI/API already takes MITRE tags
            via threat[] at rule definition; Elastic prebuilt rules ship pre-tagged; Sigma
            preserves attack.t#### tags through sigma2elastic; community classtype-to-MITRE
            JSONs cover Suricata. Building a TF-IDF tagger duplicates built-ins.
        DROPPED NLP API (Flask + Ollama wrapper) — n8n HTTP Request node can POST to
            Ollama directly with prompt templates stored in workflow JSON. No Flask shim
            needed. Workflow 1 will use n8n→Ollama; Workflow 4 (MISP→AI rule generation)
            also uses n8n→Ollama with format:json for structured rule output.
        KEPT ML Anomaly API on :5000 — IsolationForest is genuinely original; no Basic-tier
            Elastic ML equivalent. /opt/ml-api symlinks ~/soc-project/ml-api/. Bootstraps
            from synthetic alerts (n=1000, 5% anomalous), retrains via POST /train on
            .alerts-security.alerts-default once data exists. EnvironmentFile points at
            ~/soc-project/.env.local for ELASTIC_PASSWORD. Smoke /health + /score green.
        KEPT Correlation Engine on :5002 — kill-chain detection across MITRE tactics is
            not built into Kibana suppression or TheHive 5 alert grouping. SQLite state at
            /opt/correlation-engine/state.db. 30-min same-(ip,rule) bucket window; 2h
            cross-tactic chain window. All 6 actions verified end-to-end:
              create_new / add_to_existing / escalate_existing / kill-chain escalate /
              queue (low signal) / suppress (whitelist).
    - Both Flask APIs run under gunicorn (1 worker, 127.0.0.1 only — n8n on same host
      reaches them via localhost). Auto-restart on failure. Enabled at boot.
  Pending:
    - VM_A1 phase 7+: detection layer activation (custom 13 SOC rules, MITRE tagging at
      rule-definition time per the new architecture). Phase 5's deferred items
      ("MITRE Tagger /scan/suricata cron + /refresh from classification.config") are now
      OBSOLETE — that whole approach was dropped. Use the built-in path instead.
    - When real alerts start flowing, run POST :5000/train to retire the synthetic model.
    - Run an end-to-end SOAR smoke: n8n webhook → /correlate → n8n→Ollama summarize →
      TheHive case create. Belongs in phase 8.
  Things to know:
    - Sudo password unchanged. Memory rule "no custom code" was clarified by user this
      session: it means "don't reinvent built-ins" (e.g. Kibana's MITRE tagging UI is the
      built-in we should use), NOT "ask before any code". Memory file
      feedback_no_custom_code_or_configs.md updated accordingly.
    - The Fact-Forcing Gate hook fires on Bash/Write — present facts in the SAME assistant
      turn as the tool call, not the prior one. Otherwise it re-prompts.
    - ~/soc-project/nlp-api/ still exists with empty venv from before the drop — harmless;
      can be removed at any cleanup pass.
    - VM provisioned at 13 GB (plan called for 12 min / 20 recommended). Swap+heap-trim
      keeps things working. If alert volume grows, raise VM RAM.

2026-04-29 — victim-lab (VM_B2) — Phase 5 partial: Elastic Agent enrolled
  Done:
    - Downloaded elastic-agent 8.19.14 tarball (matches VM_A1 stack version), installed via
      `./elastic-agent install --url=https://192.168.1.50:8220 --enrollment-token=<...>
       --insecure --non-interactive`. Tarball install method matches VM_A1's; clean self-enroll.
    - Agent ID: c328f63a-4d33-437b-9cc1-cdfbb060df45, enrolled in victim-lab policy
      (id c226ca2c-fcd2-40c8-9ca6-11392fc7e24e). systemd unit elastic-agent.service active+enabled.
    - `elastic-agent status` reports HEALTHY / Connected to Fleet immediately after enroll.
    - Configs snapshotted at ~/soc-project/elastic-agent/ (elastic-agent.yml + INSTALL-NOTES.md).
      No secrets in either; install token is rotatable and not recorded.
  Pending:
    - VM_A1 must attach integrations to victim-lab policy in Kibana → Fleet → Add integration:
        System (host metrics + auth.log)
        Apache HTTP Server → /var/log/apache2/access_soc.log (extended `soc` format) + error.log
        Custom Logs → /var/log/suricata/eve.json (decode_json_fields), /var/log/vsftpd.log
    - MITRE Tagger /scan/suricata cron append + /refresh from classification.config still gated
      on VM_A1 Phase 3 (Flask APIs not up yet).
  Things to know:
    - Agent's local config at /opt/Elastic/Agent/elastic-agent.yml is just `fleet: enabled: true`.
      Real policy is delivered from Fleet — all integration tuning happens centrally.
    - 447 MB tarball was placed in /tmp during install; cleaned up post-enroll.

2026-04-29 — soc-core (VM_A1) — Phase 2 complete: SIEM core (ES + Kibana + Logstash + Fleet Server)
  Done:
    - All four services on Elastic 8.19.14, single-node mode, security/TLS enabled, http.host: 0.0.0.0
    - Elasticsearch: cluster.name=soc-core, node.name=soc-core, 4 GB heap (Xms=Xmx=4g), green, reachable at https://192.168.1.50:9200
    - Kibana: server.host: 0.0.0.0, server.publicBaseUrl: http://192.168.1.50:5601, available, reachable at http://192.168.1.50:5601 (HTTP, no TLS — lab)
    - Logstash: pipeline /etc/logstash/conf.d/soc-pipeline.conf with inputs beats:5044 + syslog:5140 → output ES https://192.168.1.50:9200 (ssl_verification_mode=none for lab self-signed). ELASTIC_PASSWORD passed via systemd drop-in /etc/systemd/system/logstash.service.d/override.conf (mode 600). Tried logstash-keystore first; the JRuby `create` step hung indefinitely on this slow VM, so switched to systemd Environment.
    - Elastic Agent + Fleet Server: installed via tarball (deb path doesn't expose --fleet-server flags), self-enrolled into agent policy id `fleet-server-policy` (created via Fleet API with has_fleet_server=true; auto-added fleet_server@1.6.0 package). Listens on :8220 with self-signed cert (--insecure). Fleet Server service token is in ~/soc-project/.env.local as FLEET_SERVER_SERVICE_TOKEN.
    - ufw active: default deny incoming, allow ssh from anywhere, allow ALL traffic from 192.168.1.0/24. SSH session stayed alive through `ufw enable` (conntrack OK).
    - Configs snapshotted to ~/soc-project/{elasticsearch,kibana,logstash,fleet}/ (local-only)
  Architectural notes for the rapport:
    - VM_A1 OS is Ubuntu 26.04 LTS (resolute), not the planned 22.04 — matches VM_B1. Elastic 8.x debs are codename-agnostic so no impact.
    - Initial Fleet Server install attempt failed with "Waiting on default policy with Fleet Server integration" — root cause: POST /api/fleet/setup creates the service account but does NOT pre-create policies; the agent's auto-default-policy probing didn't add the fleet_server integration. Fix: explicit POST /api/fleet/agent_policies with has_fleet_server=true, then pass --fleet-server-policy=fleet-server-policy. Documented in ~/soc-project/fleet/install-notes.md.
    - Network bottleneck on this VM: Wi-Fi-bridged virtio-net averages ~80–100 KB/s for Elastic CDN downloads. Each large package took ~1h. Phase 2 took several wall-clock hours mostly waiting on apt.
  This unblocks VM_B2:
    - VM_B2 can now enroll its Elastic Agent. Use Kibana UI Fleet → Add Agent → pick the default Agent policy (NOT fleet-server-policy), copy the install command. Or generate enrollment token via:
        curl -s -u elastic:$ELASTIC_PASSWORD -H "kbn-xsrf: soc" -X POST http://localhost:5601/api/fleet/enrollment_api_keys -d '{"policy_id":"<agent-policy-id>"}'
      Fleet Server URL for agents: https://192.168.1.50:8220 (self-signed → use --insecure)
    - VM_B2 should add Fleet integrations: System, Apache HTTP Server, Custom Logs (eve.json, access_soc.log, vsftpd.log)
  Pending on this VM (Phase 3):
    - n8n, Ollama (llama3.1:8b ~5 GB), 4 Flask APIs (ML, NLP, Correlation, MITRE Tagger). Big disk + RAM step — confirm before starting.
  Real credentials live in ~/soc-project/.env.local (mode 600). CLAUDE.md only references placeholders.

2026-04-29 — incident-mgmt (VM_B1) — Phase 4 complete: MISP + analyzers + UFW
  Architectural deviations from the plan (defendable for the rapport):
    - Cortex analyzers — built a custom image soc-cortex:4.0.1-analyzers (Dockerfile in
      ~/soc-project/thehive-cortex/cortex/Dockerfile) extending thehiveproject/cortex:4.0.1.
      Reason: docker-mode (mounting /var/run/docker.sock) is unsafe — the upstream cortex
      entrypoint runs `chown 1001:1001 /var/run/docker.sock` and that bind-mount mutates the
      HOST socket's ownership, locking vboxuser out of docker. Process mode + on-disk
      Cortex-Analyzers (cloned at build time) is the durable fix. Image bakes in cortexutils,
      pymisp, OTXv2, vt-py, python-magic, libmagic1.
    - MISP port binding — bound 192.168.1.51:8080 / 192.168.1.51:8443 only, not 0.0.0.0.
      UFW INPUT rules don't filter docker-published ports (FORWARD chain), so we restrict
      the listener at the source. Compose vars in ~/soc-project/misp/.env: CORE_HTTP_PORT
      and CORE_HTTPS_PORT have an IP prefix.
    - AlienVault OTX — integrated as a Cortex analyzer (OTXQuery_2_0), NOT as a MISP feed.
      MISP's bundled defaults don't include OTX, and the OTX→MISP bridge would be a
      cron-driven pull script. Cortex analyzer gives on-demand enrichment which is more
      useful for the SOC pipeline anyway. Document the choice in the rapport.
    - Cortex CSRF gotcha — Cortex 4.0.1 uses a custom header `X-CORTEX-XSRF-TOKEN` for
      session-auth writes (cookie name is `CORTEX-XSRF-TOKEN`). Bearer token auth bypasses
      CSRF. Documented because session auth attempts kept hitting "No CSRF token found".
  Done:
    - MISP Docker stack at https://192.168.1.51:8443 (5 containers: misp-core, misp-modules,
      db (mariadb 10.11), redis (valkey 7.2), mail). First boot needed `up -d` twice — modules
      health check window is shorter than its actual readiness. Documented in known-issues
      already; matches expectation.
    - MISP admin API key generated via UI; saved to ~/soc-project/.env.local as MISP_API_KEY.
    - MISP feeds: loaded all 96 bundled defaults via /feeds/loadDefaultFeeds, then enabled
      caching+fetch on:
        id=1  CIRCL OSINT Feed
        id=4  ET Compromised IPs (rules.emergingthreats.net)
        id=12 Feodo IP Blocklist
        id=65 Abuse.ch URLhaus
      Initial fetch queued via /feeds/fetchFromAllFeeds.
    - MISP feed auto-fetch cron — host crontab on VM_B1 (vboxuser):
        `0 */6 * * * /home/vboxuser/soc-project/misp/cron-feeds.sh`
      Script docker-execs `cake Server fetchFeed/cacheFeed all` and rotates its own log.
      Used host cron (not in-DB MISP scheduler) because the misp-core image doesn't run
      the `scheduler` background worker — only default/email/cache/prio/update.
    - Cortex analyzer instances (all 4 verified live, end-to-end):
        AbuseIPDB (id=6b1c…) on 8.8.8.8 → safe; on 185.220.101.1 → 100/malicious, Tor=True
        OTXQuery (id=eb54…)  on 8.8.8.8 → 0 pulses
        VirusTotal_GetReport (id=8cac…) on WannaCry SHA256 → 69/75 malicious
        MISP (id=b20c…) wired to https://192.168.1.51:8443 (cert_check=false), tested OK
      cortex-user roles: read,analyze,**orgadmin** (added so the same key can configure
      analyzer instances + run them — saves a second key for TheHive).
    - Superadmin API key generated and saved (CORTEX_SUPERADMIN_API_KEY in .env.local).
    - UFW active. Default deny-in / allow-out / deny-routed.
        9000 (TheHive), 9001 (Cortex), 8080+8443 (MISP), 22 (SSH future-proof) — ALLOW from
        192.168.1.0/24 only. 9993/udp (ZeroTier control) ALLOW from anywhere. 9042/9200
        kept localhost-only by service config (no UFW rule needed).
    - Earlier 192.168.1.60 connection — confirmed by user, that's their host machine on the
      ZT network. No rogue actor. Reachability from .60 is via the new UFW rules.
  Files added on this VM (not committed; live in ~/soc-project/):
    - thehive-cortex/cortex/Dockerfile             custom cortex image
    - thehive-cortex/cortex/Cortex-Analyzers/      cloned shallow from GitHub (191MB)
    - thehive-cortex/docker-compose.yml            updated: build.network=host, soc-cortex tag
    - misp/cron-feeds.sh                           feed fetch + cache, log rotation
    - misp/.env                                    CORE_HTTP/HTTPS_PORT prefixed with ZT IP
    - .env.local                                   added MISP_API_KEY, OTX/ABUSEIPDB/VT keys,
                                                   CORTEX_SUPERADMIN_API_KEY, SUDO_PASS
  Memory snapshot at end of phase 4: ~7.7G used / 9.0G total. Tight but stable. The bulk is
    Cortex JVM, ES JVM, TheHive JVM, MISP PHP-FPM. No swap available — don't add a desktop GUI.
  Things to know for next session:
    - On reboot, the docker compose files come up on their own (restart: unless-stopped).
      Cassandra/ES are systemd services. Crontab survives reboot.
    - The CSRF header name `X-CORTEX-XSRF-TOKEN` — write this down. If you ever need to
      script user/org admin via session auth, that's the gotcha.
    - cortex-user's API key is the do-everything key on this VM (orgadmin in SOC-LAB +
      analyze + read). TheHive uses it via TH_CORTEX_KEYS env var.
    - 192.168.1.60 is user's host (ZT-joined) — expected source for browser access.

2026-04-29 — victim-lab (VM_B2) — Phase 5 (vulnerable target + IDS, Elastic Agent deferred)
  Done:
    - Apache 2 + PHP 8.5 + libapache2-mod-php + php-mysqli/gd/xml/cli installed (Ubuntu 25.10 ships PHP 8.5, newer than the master plan's 8.x assumption — DVWA still works, just emits deprecation warnings that we silence with display_errors=Off)
    - PHP config patched: allow_url_include=On (enables RFI), display_errors=Off, log_errors=On, allow_url_fopen=On
    - MariaDB installed; root uses unix_socket auth (no password); created `dvwa` DB + `dvwa@localhost` user with random 24-char password (saved to ~/soc-project/.env.local as DVWA_DB_PASSWORD)
    - DVWA cloned to /var/www/html/dvwa from github.com/digininja/DVWA; chown www-data; config/ and hackable/uploads/ writable; config.inc.php patched with real DB password; default_security_level flipped from 'impossible' to 'low' (cookie 'security' now defaults low for any new session)
    - DB tables created via /dvwa/setup.php (users, guestbook, access_log, security_log); admin/password login confirmed
    - vsftpd installed; anonymous_enable=NO (default), local_enable=YES, write_enable=YES, xferlog_file=/var/log/vsftpd.log; listening on :21
    - openssh-server installed; uses ssh.socket (socket activation pattern on modern Ubuntu — `systemctl is-active ssh` shows inactive, that's normal); drop-in /etc/ssh/sshd_config.d/99-soc-lab.conf forces PasswordAuthentication yes + PermitRootLogin no
    - Three weak users created (no sudo): testuser1/password123, testuser2/admin, webadmin/webadmin — verified all 3 can log in via FTP (curl --user → 226)
    - Suricata 8.0.3 + suricata-update installed; af-packet interface set to ztdiyzommr (NOT enp0s3, which is VirtualBox NAT); HOME_NET already covers 192.168.0.0/16; 49,911 ET Open rules loaded (of 65,786 total, 15k disabled by default); 6 worker threads; eve.json being written; first hit was "ET INFO Spotify P2P Client" (low-sev, ignore)
    - /etc/cron.d/suricata-update at 03:00 daily — refreshes rules + reloads suricata
    - Apache extended log: LogFormat 'soc' defined in /etc/apache2/conf-available/soc-logging.conf; CustomLog directive added inside 000-default.conf vhost (LogFormat at server level wasn't enough — vhost CustomLog overrides server-level); writes to /var/log/apache2/access_soc.log with timing(µs), bytes_in/out, query string, referer, UA, X-Forwarded-For; chowned root:adm so the future Elastic Agent can read it
    - Snapshot of all editable configs copied to ~/soc-project/{apache,mariadb,ssh-and-vsftpd,suricata,dvwa}/
  DEFERRED — needs VM_A1 first:
    - Elastic Agent enrollment: needs Fleet Server URL + enrollment token from VM_A1 Phase 2 (Elasticsearch+Kibana+Fleet not deployed yet)
    - Daily MITRE auto-tag call: append `&& curl -sS -X POST http://192.168.1.50:5003/scan/suricata` to /etc/cron.d/suricata-update once VM_A1 Phase 3 (Flask APIs) is done
    - MITRE Tagger /refresh after install: trigger once classification.config can be read by the tagger (either copy /etc/suricata/classification.config to A1 or have A1 SSH to read it)
  Surface for next instance:
    - VM_B2 services to add to Kibana Fleet integrations once A1 is up: System, Apache HTTP Server (parses access.log combined format natively), Custom Logs for /var/log/suricata/eve.json AND /var/log/apache2/access_soc.log AND /var/log/vsftpd.log
    - Reboot performed at end of session to verify all services persist (apache2, mariadb, vsftpd, ssh.socket, suricata, zerotier-one all `systemctl enable`d)
    - VM_B2 sudo password is 'kali' (in ~/soc-project/.env.local, not in this file)

2026-04-29 — incident-mgmt (VM_B1) — Phase 4 partial: Cassandra + ES + TheHive + Cortex
  Architectural deviation from the plan (defendable for the rapport):
    - Master plan called for `apt install thehive` and `apt install cortex` from StrangeBee's APT repo.
    - StrangeBee killed deb.strangebee.com / archives.strangebee.com (DNS no longer resolves, even via 8.8.8.8).
    - TheHive 5.x is no longer distributed as .deb; only via Docker image strangebee/thehive.
    - Decision: shifted TheHive 5 + Cortex 4 to Docker containers (network_mode: host) so they can reach Cassandra + ES on host's 127.0.0.1.
    - Cassandra (4.1.11) and the local Elasticsearch (8.19.14) remain native apt installs because their .deb channels are still public.
    - All other services on this VM continue per the plan.
  Done:
    - OS: Ubuntu 26.04 LTS (resolute, codename "resolute"). Defaults differ from the 22.04 the plan assumes — flagged but no functional impact so far.
    - openjdk-11-jdk installed (11.0.30); JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64 set in /etc/profile.d/java_home.sh
    - Cassandra 4.1.11 from Apache repo (https://debian.cassandra.apache.org 41x main)
        cluster_name: SOC-Cluster, listen+rpc on 127.0.0.1, MAX_HEAP_SIZE=512M, HEAP_NEWSIZE=100M
        ports listening (localhost only): 9042 CQL, 7199 JMX, 7000 internal
        node UN at 127.0.0.1; nodetool healthy after first start
    - Elasticsearch 8.19.14 (local to this VM, distinct from VM_A1's SIEM ES which doesn't exist yet)
        cluster.name: thehive-cortex-cluster, node.name: incident-mgmt-es
        bound 127.0.0.1:9200/9300 only, xpack.security.enabled: false (defensible because of localhost binding)
        heap: 1g via /etc/elasticsearch/jvm.options.d/heap.options
        cluster status green, single-node mode
    - Docker 29.4.1 + compose v5.1.3 from official Docker repo (used noble channel — no resolute packages yet)
    - TheHive 5.7.1 container (strangebee/thehive:5.7.1) + Cortex 4.0.1 container (thehiveproject/cortex:4.0.1)
        compose file: ~/soc-project/thehive-cortex/docker-compose.yml
        secrets in   ~/soc-project/thehive-cortex/.env (mode 600, never committed)
        bind mounts: ./thehive/data → /data (uid 1000), ./cortex/data → /var/cortex-jobs (uid 1001)
        TheHive points at TH_CQL_HOSTNAMES=127.0.0.1, TH_INDEX_BACKEND=elasticsearch, TH_ELASTICSEARCH_HOSTNAMES=127.0.0.1
        TheHive↔Cortex wired via TH_CORTEX_HOSTNAMES=127.0.0.1, TH_CORTEX_KEYS=<key in .env.local>
        Logs confirm: `Analyzer templates already present (found 273)` ⇒ TheHive successfully called Cortex API at startup
    - Browser-side wizards completed by user:
        TheHive: admin@thehive.local password changed from default 'secret'
        Cortex:  superadmin created, organization SOC-LAB created, cortex-user (read,analyze) with API key generated
  Things to know:
    - TheHive 5 ships with auto-loaded 15-day Platinum trial license; lab uses only Community-tier features for the rapport
    - One 35s GC pause was logged on TheHive during initial bootstrap. It settled after warmup. If recurs, add JAVA_OPTS=-Xms1g -Xmx1g to TheHive container env.
    - cqlsh is broken on Ubuntu 26.04 (six.moves missing in system Python 3.13). Cassandra server unaffected. Use `nodetool` for admin or pip install six --break-system-packages if needed.
    - Real credentials live in ~/soc-project/.env.local (mode 600). Never commit. CLAUDE.md only references placeholders.
    - Memory snapshot after Cassandra+ES+TheHive+Cortex: 6.0G used / 9.0G total. ~3G free for MISP (~1G) + headroom.
  Pending on this VM (Phase 4 remainder):
    - MISP via Docker Compose at https://192.168.1.51:8443 (self-signed cert)
    - MISP feeds: CIRCL OSINT, Abuse.ch URLhaus, Feodo Tracker, AlienVault OTX, ET Compromised IPs (auto-fetch every 6h)
    - Cortex analyzers: pip install cortexutils + AbuseIPDB / VirusTotal / MISP_Search; configure free-tier API keys
    - ufw firewall: open 9000/9001/8443 to ZeroTier subnet only; keep 9042/9200 localhost-only
  Outside connections seen:
    - 192.168.1.60 hit /api/v1/status/public on TheHive — not one of our planned VMs (.50/.51/.52/.53). Likely user from a separate ZT-joined machine. Worth confirming.

2026-04-29 — VM_A2 (kali-attacker) descope decision
  - User declared VM_A2 out of scope for the SOC build going forward.
  - Rationale: attack simulations (Phase 6 / parts of Phase 10) can be launched from any reachable host.
  - Effect: Phase 6 marked descoped; phase_0 / phase_1 flipped to complete despite VM_A2 not catching up.

[older entries archived to docs/session-history.md: VM_B2 Phase 0+1, VM_A1 Phase 1 progress, VM_B1 Phase 0 clone, VM_A1 Phase 0 bootstrap]
```

---

## Known issues & gotchas

- **Project files stay local:** Anything you create in `~/soc-project/` lives only on that VM. Other VMs cannot see it. The shared brain is CLAUDE.md only.
- **VM_B1 OOM risk:** Cassandra and Elasticsearch heap caps (512 MB / 1 GB) must stay enforced. Don't run desktop GUI. Service startup order matters.
- **Ollama first prompt slow:** Initial model load takes ~30 seconds. Keep Ollama warm or accept cold-start latency.
- **Fleet Server tokens:** Generated once. If lost, regenerate via Kibana > Fleet > Settings, but all enrolled agents need re-enrollment.
- **MITRE Tagger before Suricata installed:** Until VM_B2's Phase 5 completes and `/etc/suricata/classification.config` exists (file synced to A1 or accessed remotely), the classtype map is empty. Trigger `/refresh` after Phase 5.
- **TheHive 5 install:** StrangeBee discontinued the public APT repo. Install from manually downloaded `.deb`.
- **MISP Docker first start:** Some containers report unhealthy on first boot. `docker compose down && up -d` again to settle.
- **Self-signed certificates:** MISP uses HTTPS with self-signed cert. Cortex's MISP analyzer config needs `cert_check: false` for the lab.
- **n8n webhook URLs:** Defined when activating a workflow. Document path changes in this file.
- **ZeroTier interface name:** Auto-generated (e.g. `zt6q3gtnzl`). Use `ip -4 addr show` and grep for the 192.168.1.x address.
- **Java for Cassandra:** TheHive 5 / Cassandra 4 require Java 11. Newer Java versions break Cassandra startup.

---

## PFE rapport notes

> Append findings, decisions, and metrics here as the project progresses.
> This becomes the source for the rapport de fin d'études.

### Architectural decisions

**Elastic Stack chosen over Wazuh as SIEM/HIDS engine:**
- Native Kibana SIEM detection engine with EQL/KQL is more expressive than Wazuh's XML rules
- Single Elastic Agent replaces Wazuh Agent + Filebeat + Auditbeat in one binary
- Fleet Server provides centralized agent management without per-host SSH
- Built-in ML jobs and Security app dashboards reduce custom development

**n8n chosen over Shuffle as SOAR:**
- Open-source self-hosted, free for academic use
- Visual workflow editor reduces reliance on documentation for review
- Built-in nodes for SSH, HTTP, schedule make active response trivial

**Ollama chosen over Anthropic API as LLM backend:**
- Zero ongoing cost
- No external network dependency for AI features
- Self-contained — defendable in security-sensitive contexts
- Trade-off: llama3.1:8b is less capable than Claude/GPT-4 on complex reasoning, sufficient for structured outputs
- Architecture is provider-agnostic: `LLM_BACKEND` env var switches at deployment time

**Custom Correlation Engine instead of TheHive's built-in alert grouping:**
- TheHive 5 alert clustering is rule-based, not ML/tactic aware
- Custom engine integrates ML anomaly score and MITRE tactic progression for chain detection
- Solves the "1 attack = 7 cases" problem documented in academic SOC literature
- Strongest original contribution of the project

**MITRE Auto-Tagger built from MITRE ATT&CK STIX data:**
- Eliminates manual rule-tagging burden which would otherwise scale linearly with rule count
- Three-tier approach: existing metadata > TF-IDF keyword match > LLM disambiguation
- Auto-rebuilds when MITRE publishes new ATT&CK versions
- Suricata classtype map auto-generated from `classification.config`, not hardcoded

**Lean Git repo strategy:**
- Only CLAUDE.md and shared docs go on Git
- Configs, code, models, workflows stay local on each VM
- Justification: PFE is sequential and demonstrative, not collaborative — Git's value here is cross-VM context sync, not full source control
- Trade-off: local backups required for VM resilience

### Metrics to capture during testing (fill after Phase 10)

- [ ] Number of attack scenarios simulated
- [ ] Detection rate per layer (Suricata / Kibana custom / Kibana prebuilt / MISP intel / ML)
- [ ] Average time from attack to TheHive case creation (MTTD)
- [ ] Average alerts per attack before correlation
- [ ] Average alerts per case after correlation (target: ≥ 3:1 reduction)
- [ ] False positive rate before vs after ML score gating
- [ ] MITRE ATT&CK technique coverage (techniques covered / techniques in framework)
- [ ] Auto-generated rules from MISP events: count and confidence distribution
- [ ] Latency: ML API, NLP API, Correlation Engine, MITRE Tagger
- [ ] Active response success rate (auto-block firing as expected)

---

*This file is updated continuously. The version on `main` is the canonical state of the project.*
