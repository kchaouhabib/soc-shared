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
**Hostname:** `soc-core` | **IP:** `192.168.1.50` | **Host:** PC_A | **OS:** Ubuntu Server 22.04 LTS

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
  url: http://192.168.1.50:9200
  user: elastic
  password: FILL_AFTER_SETUP

kibana:
  url: http://192.168.1.50:5601
  user: elastic
  password: FILL_AFTER_SETUP

fleet_server:
  url: http://192.168.1.50:8220
  service_token: FILL_AFTER_SETUP
  enrollment_token_victim_lab: FILL_AFTER_SETUP
  enrollment_token_attacker: FILL_AFTER_SETUP

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
phase_2_vm_a1_siem_core:              pending
phase_3_vm_a1_soar_and_ai:            pending
phase_4_vm_b1_incident_mgmt:          in_progress    # Java 11, Cassandra 4.1.11 (heap 512M, cluster SOC-Cluster), local ES 8.19.14 (heap 1G, cluster thehive-cortex-cluster), Docker 29.4.1+compose v5, TheHive 5.7.1 + Cortex 4.0.1 (containers, network=host) all running; TheHive↔Cortex wired via API key; MISP install + feeds + Cortex analyzers + ufw still pending
phase_5_vm_b2_victim_lab:             in_progress    # Apache+PHP8.5+MariaDB up; DVWA at /dvwa (admin/password, default security 'low'); vsftpd:21 + ssh:22 (socket-activated) + 3 weak users (testuser1/password123, testuser2/admin, webadmin/webadmin); Suricata 8.0.3 on ztdiyzommr with 49911 ET Open rules + daily refresh cron; Apache extended log format 'soc' at access_soc.log. DEFERRED until VM_A1 ready: Elastic Agent enrollment, MITRE Tagger /scan/suricata in cron, MITRE Tagger /refresh from classification.config
phase_6_vm_a2_kali_attacker:          descoped       # user descoped 2026-04-29 — attacks can be launched from any reachable host or skipped
phase_7_detection_layer_activation:   pending
phase_8_soar_integration:             pending
phase_9_adaptive_intelligence:        pending
phase_10_testing:                     pending
phase_11_documentation:               pending

last_updated: 2026-04-29
updated_by: victim-lab (VM_B2)
```

---

## Last session notes

> Latest entry on top. Each entry: what was done, what's pending, anything next instance needs to know.
> Maximum 5 entries kept; older ones archived in `docs/session-history.md`.

```
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

2026-04-29 — victim-lab (VM_B2) — Phase 0 + Phase 1 (with VM_B1 collateral)
  Done:
    - This VM was cloned from VM_B1, so it inherited VM_B1's ZeroTier identity (shared node 9ab369cb6c, both presenting IP 192.168.1.51)
    - Purged zerotier-one + wiped /var/lib/zerotier-one/ → reinstalled → fresh node aa429ed844
    - Joined cf719fd54008e4d1; Central authorized aa429ed844 with IP 192.168.1.53
    - Hostname myguest → victim-lab (hostnamectl + 127.0.1.1 in /etc/hosts)
    - Cloned soc-shared to /home/vboxuser/soc-shared (after first landing in ~/pfe/, then moved); created /home/vboxuser/soc-project/
    - Switched git remote to SSH (git@github.com:kchaouhabib/soc-shared.git); SSH key on this VM: ~/.ssh/id_ed25519 (pubkey added to user's GitHub account, titled "victim-lab (VM_B2)")
  VM_B1 status:
    - User confirmed VM_B1's real ZT node ID is also 9ab369cb6c (same as the clone's, which was VM_B1's identity all along — this VM was the copy, not the original).
    - VM_B1 stayed authorized and reachable; ping from VM_B2 to 192.168.1.51 returns 0% loss (~7ms, direct P2P on same host).
    - No collateral damage from the deauth/rejoin dance — VM_B1 is fine.
  Connectivity verified from VM_B2:
    - 192.168.1.50 (VM_A1) ✅ 0% loss, ~174ms (via ZT root, different physical host)
    - 192.168.1.51 (VM_B1) ✅ 0% loss, ~7ms (P2P, same host)
    - 192.168.1.52 (VM_A2) ❌ unreachable (Phase 1 not done on A2 yet)
  Pending for next instances:
    - VM_A2: Phase 0 (clone+soc-project) + Phase 1 (install zerotier-one, join cf719fd54008e4d1, request user auth at IP .52)
    - Then verify all-4 cross-VM ping before flipping phase_1 to complete
  Notes:
    - VM_B2 still missing all Phase 5 services (Apache/MariaDB/DVWA/vsftpd/Suricata/Elastic Agent) — fresh install ahead
    - Reboot recommended before Phase 5 to confirm hostname persists
    - VM_B2 sudo password is in ~/soc-project/.env.local on the VM (NOT in this file, NOT in Git)

2026-04-29 — soc-core (VM_A1) — Phase 1 progress
  Done since last entry:
    - ZeroTier network ID changed from 743993800ffa3724 to cf719fd54008e4d1 (user re-created the network); updated in CLAUDE.md and PROJECT-MASTER-PLAN.md
    - VM_A1 joined cf719fd54008e4d1 (network name "my-first-network")
    - User authorized the node in admin console; status OK, IP 192.168.1.50/24 live on interface ztdiyzommr
    - VM_A1 ZeroTier node ID: 785fd1806c (reference for re-authorizing if needed)
    - Note: VM_B1 already reports IP 192.168.1.51 in its last session note, so VM_B1 likely needs to re-join the new network ID cf719fd54008e4d1 if it was on the old one. Verify with `sudo zerotier-cli listnetworks` on VM_B1.
  Pending for next instance:
    - VM_B1: confirm it's on cf719fd54008e4d1 (not the old 743993800ffa3724); rejoin if needed
    - VM_B2 / VM_A2: Phase 0 (clone+soc-project) AND Phase 1 (install zerotier-one, join cf719fd54008e4d1, await user auth + IP .53/.52)
    - Verify cross-VM ping once all four are up before flipping phase_1 to complete
  Notes:
    - Sudo password is in local memory; user must restate it per session for the harness to accept it (auto-pipe was blocked)
    - Plugin marketplace install (everything-claude-code) was started in background but is still cloning at last check — separate side task, not part of phase 1

2026-04-29 — incident-mgmt (VM_B1) — Phase 0 clone
  Done:
    - Installed git via apt (was missing on this VM)
    - Cloned soc-shared to ~/soc-shared/ (initially landed in ~/pfe/, then moved to canonical ~/ location)
    - Created empty ~/soc-project/ (local-only working folder)
    - Verified `cd ~/soc-shared && git pull` returns "Already up to date."
    - Confirmed VM identity: hostname=incident-mgmt, ZeroTier IP=192.168.1.51 (interface ztdiyzommr)
  Notes:
    - On VM_B1 the user prefers the canonical ~/soc-shared/ path over working out of ~/pfe/. Resolved the open question from the previous note.
    - No project work started yet on this VM — Phase 4 (Cassandra+ES+TheHive+Cortex+MISP) is still pending.

2026-04-29 — soc-core (VM_A1) — Phase 0 bootstrap
  Done:
    - Created GitHub repo https://github.com/kchaouhabib/soc-shared (public)
    - Bootstrapped ~/soc-shared/ with CLAUDE.md, PROJECT-MASTER-PLAN.md, README.md, docs/.gitkeep
    - Initial commit 7429194 pushed to origin/main
    - Created empty ~/soc-project/ (local-only working folder, not on Git)
    - Git identity already configured globally (Habib Kchaou / kchaou.habib67@gmail.com)
  Notes:
    - Source markdown files were copied from /home/vboxuser/pfe/ on VM_A1
    - Keep that pfe/ folder as the editable working location, or delete it now that ~/soc-shared/ is the canonical source — your call
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
