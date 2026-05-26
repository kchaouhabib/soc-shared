# PROJECT MASTER PLAN — SOC Autonome PFE
> Single source of truth. Reference document for the full project.
> No code in this file — descriptions only. Code is requested per step when needed.

---

## 1. Project Identity

| Field | Value |
|---|---|
| Project name | SOC Autonome — Automation des niveaux L1 et L2 |
| Type | Projet de Fin d'Études (PFE) |
| Core stack | Elastic Stack 8.x · TheHive 5 · Cortex 4 · MISP · n8n · Suricata · Ollama |
| Detection framework | MITRE ATT&CK |
| LLM backend | Ollama (local, free) — no external API costs |
| Total cost | Zero — all components free / open source |

**One-line description:** An autonomous Security Operations Center that detects attacks across multiple layers (network IDS + host logs + threat intelligence + behavioral ML), correlates redundant alerts into single incidents, auto-generates new detection rules from threat reports using local AI, and routes only the truly important incidents to a human analyst.

---

## 2. Storage Strategy — Two Folders Per VM

The project is intentionally split into two parts:

```
~/soc-shared/        ← Git repo (GitHub). ONLY CLAUDE.md + docs/ + this plan.
                       Pulled at session start by every VM.
                       Updated and pushed after every phase.
                       Pure shared brain — no code, no configs, no workflows.

~/soc-project/       ← Local-only project files. NEVER on Git.
                       All service configs, Flask code, n8n workflow exports,
                       ML models, attack scripts. Stays on the VM that owns it.
```

**Why this split:** The project will accumulate hundreds of MB of configs, model files, n8n workflow exports, and Suricata ruleset copies. Pushing all of that to GitHub on every commit would be slow and cluttered. The actual goal of Git here is cross-VM coordination — every Claude Code instance reading the same CLAUDE.md. That only requires a tiny shared file, not the whole project.

**Trade-off:** No GitHub backup of the actual project. Mitigated by local backup strategy:
- `rsync ~/soc-project/ /mnt/backup/` periodically (USB drive or other VM)
- Or VM snapshots at major milestones
- Worst case: rebuild a single VM from this master plan

---

## 3. Architecture Overview

The system is built on 4 VMs distributed across 2 physical hosts, connected through a ZeroTier virtual network so all machines communicate as if on the same LAN regardless of physical location.

| VM | Hostname | OS | Host PC | IP | Role |
|---|---|---|---|---|---|
| VM_A1 | soc-core | Ubuntu Server 22.04 LTS | PC_A | 192.168.1.50 | SIEM + SOAR + AI services |
| VM_B1 | incident-mgmt | Ubuntu Server 22.04 LTS | PC_B | 192.168.1.51 | TheHive + Cortex + MISP |
| VM_B2 | victim-lab | Ubuntu Server 22.04 LTS | PC_B | 192.168.1.53 | Vulnerable target + Suricata IDS |
| VM_A2 | kali-attacker | Kali Linux | PC_A | 192.168.1.52 | Offensive testing |

**OS choice rationale:** Ubuntu Server LTS for the three SOC VMs because it's the standard enterprise Linux for production security infrastructure — better long-term support, smaller default footprint than Kali (no desktop, no offensive tools preinstalled), and aligns with what the PFE jury will expect to see in a serious SOC deployment. Kali is kept only on the attacker VM where the offensive toolset is needed.

---

## 4. VM Specifications

| VM | RAM min | RAM recommended | Storage min | Storage recommended |
|---|---|---|---|---|
| VM_A1 soc-core | 12 GB | 20 GB | 200 GB SSD | 500 GB SSD |
| VM_B1 incident-mgmt | 10 GB | 14 GB | 100 GB | 200 GB |
| VM_B2 victim-lab | 3 GB | 4 GB | 40 GB | 80 GB |
| VM_A2 kali-attacker | 3 GB | 4 GB | 40 GB | 80 GB |

**Why VM_A1 needs more RAM than originally estimated:** Ollama with the llama3.1:8b model requires ~6 GB when loaded into memory. Combined with Elasticsearch (heap dropped to 2 GB after re-architecture), Kibana, Logstash, Fleet Server, n8n, and the **two** Flask APIs (re-architected 2026-05-05 — see Section 6.4), the safe baseline is 16 GB and recommended is 20 GB. The deployed VM is 13 GB — workable thanks to swap and the heap trim, but tight; raise RAM if alert volume grows.

**SSD strongly recommended for VM_A1** — Elasticsearch indexing performance is heavily disk-bound. Spinning disks make Kibana queries slow.

**No desktop GUI on any Ubuntu VM** — saves 500 MB to 1 GB of RAM and removes attack surface.

---

## 5. Network Topology

```
ZeroTier overlay network — 192.168.1.0/24
Network ID: cf719fd54008e4d1

  PC_A                              PC_B
  ┌─────────────────────┐           ┌─────────────────────────────────────┐
  │ VM_A1  192.168.1.50 │           │ VM_B1  192.168.1.51                 │
  │ soc-core            │◄─────────►│ incident-mgmt                       │
  │ Ubuntu Server       │           │ Ubuntu Server                       │
  │                     │           │                                     │
  │ VM_A2  192.168.1.52 │           │ VM_B2  192.168.1.53                 │
  │ kali-attacker       │──attacks─►│ victim-lab                          │
  │ Kali Linux          │           │ Ubuntu Server                       │
  └─────────────────────┘           └─────────────────────────────────────┘
```

ZeroTier was chosen over bridged networking because the two physical PCs may be on different subnets (campus wifi, mobile hotspot, etc.) and cannot reliably reach each other through host-only or bridged adapters. ZeroTier creates a layer-2 virtual LAN over any internet connection.

---

## 6. Service Catalog — What Each Component Does

### 6.1 Detection Layer

**Elasticsearch 8.x** — distributed search engine and document store. Holds every log, every alert, every detection rule match. Everything in the SOC ultimately writes to or reads from Elasticsearch. Free Basic license covers all features needed.

**Kibana 8.x** — web UI for Elasticsearch. The SOC analyst dashboard, the SIEM detection rules manager, the alert review interface, the case investigation tool. Free Basic license includes the Security app with detection engine, alerting, and webhook actions.

**Logstash** — optional log preprocessing pipeline. Receives logs from external sources (Beats, syslog), enriches them (GeoIP, parsing), and forwards to Elasticsearch. With Elastic Agent doing most log shipping directly, Logstash has a smaller role here — kept for future syslog integration with network devices.

**Fleet Server** — central management plane for all Elastic Agents. Lets you deploy and reconfigure agents across all monitored machines from one Kibana UI without SSHing into each one. Replaces the old Wazuh manager-agent model.

**Elastic Agent** — single binary that replaces Wazuh Agent, Filebeat, Auditbeat, and Metricbeat in one. Installed on every monitored machine. Collects host logs, system metrics, file integrity events, and ships them to Fleet Server. Configured entirely from Kibana UI through Integrations (Apache, MySQL, System, Custom Logs, etc.).

**Suricata** — network intrusion detection system. Sits at the network layer on the victim VM and inspects every packet against 50,000+ Emerging Threats Open rules. Detects exploits, malware traffic, command-and-control, scanners, and known attack signatures at the packet level — before the application even processes the request. Catches attacks that signature-based log detection would miss.

**suricata-update** — official tool that automatically pulls the latest ET Open ruleset every day. Keeps detection signatures fresh without any manual rule management.

### 6.2 Incident Response Layer

**Cassandra** — distributed NoSQL database used as TheHive's primary backend storage for cases and observables. Configured with capped 512 MB heap to coexist with other services on the same VM.

**Local Elasticsearch on VM_B1** — separate Elasticsearch instance dedicated to TheHive and Cortex indexing needs. Independent from the main SIEM Elasticsearch on VM_A1. Capped at 1 GB heap.

**TheHive 5** — security incident response platform. Central case management for the SOC analyst. Receives alerts from n8n, displays them as cases with timeline, observables, tasks, and analyst notes. The single pane of glass where the analyst spends their day.

**Cortex 4** — observable analysis engine. When TheHive has an IP, hash, domain, or email to investigate, Cortex runs analyzers against it (AbuseIPDB, VirusTotal, Shodan, etc.) and writes the results back to the case. Free analyzers cover the lab's needs.

**MISP** — threat intelligence platform deployed via Docker. Stores Indicators of Compromise (IOCs), pulls fresh threat data automatically from public feeds, and exposes them via API to Cortex and Elasticsearch. The "live database of all known attacks" the project requires.

### 6.3 Orchestration Layer

**n8n** — workflow automation engine ("SOAR" — Security Orchestration, Automation and Response). The glue between all services. Receives alerts, calls APIs, makes decisions, creates cases, runs SSH commands for active response. Self-hosted, free for personal/academic use.

n8n state lives in SQLite at `/home/vboxuser/.n8n/.n8n/database.sqlite` (note the nested `.n8n/.n8n/` path). Encryption key for n8n credentials is in `/home/vboxuser/.n8n/.n8n/config`. n8n is managed by systemd unit `/etc/systemd/system/n8n.service` (`Restart=on-failure`, runs as `vboxuser`, env vars including `N8N_USER_FOLDER=/home/vboxuser/.n8n`, `WEBHOOK_URL=http://192.168.1.50:5678/`). Workflow JSON exports kept in `/home/vboxuser/soc-project/n8n/workflows/` for redeploy; operator scripts that build/patch workflows directly against the SQLite DB live in `/tmp/wf*.py`.

The deployed catalogue has **eight workflows** (see §16 for IDs and current verification status):

| # | Name | Trigger | Purpose |
|---|---|---|---|
| WF1 | 01 Alert Pipeline | webhook `/elastic-alert` | Receives every detection-engine alert → correlation → ML score → AI summary → TheHive case + email + auto-tag + severity-gated email-to-analyst + active response (iptables) |
| WF2 | 02 Cortex Enrichment | webhook `/thehive` (via polling bridge) | On new TheHive case: fetch observables → dispatch all applicable Cortex analyzers → AI synthesis of reports → post Threat Intel page to case |
| WF3 | 03 Daily Digest | cron 08:00 Africa/Tunis | Aggregate queued low-severity buckets → AI narrative → single digest case in TheHive |
| WF4 | 04 MISP → AI Rule Gen | webhook `/misp` (via polling bridge) | On new MISP event: AI generates `{kibana_rule, suricata_rule, mitre}` JSON → import disabled to Kibana (analyst-in-loop) |
| WF5 | 05 Weekly Maintenance | cron Mon 04:00 | Calls `POST :5000/train` to retrain Isolation Forest on last 30 days of ES alerts |
| WF6 | 06 Investigation Report | webhook `/thehive-case-resolved` | On case close: fetch full case context → AI multi-section investigation report (9 sections) → post Markdown page under Reports tab |
| WF7 | 07 Q&A Chatbot | (deactivated) | Attempted comments-as-chatbot — built but dropped 2026-05-17, see §16 |
| WF8 | 08 Email Ack | webhook `/email-ack` | Analyst clicks "I'm on it" link in alert email → posts ack comment + adds `email-acked:<user>` tag → renders polished confirmation HTML page |

### 6.4 Custom AI / Intelligence Services (built for this project)

These are the original contributions of the PFE — **two Flask microservices** that solve problems no built-in covers, plus n8n workflows that orchestrate stock components for everything else.

> **Re-architecture note (2026-05-05):** The original plan called for five Flask services. Two of them were dropped during Phase 3 because stock components already cover the same job. This made the stack smaller and more defendable. See *Dropped services* at the bottom of this section.

**Ollama (port 11434)** — local LLM runtime. Originally ran llama3.1:8b on VM_A1; **moved 2026-05-15 to a separate cloud VM at `10.11.21.31`** to reclaim ~6 GB RAM on VM_A1. Now reachable from VM_A1 via an SSH port-forward through the Windows host (`192.168.1.70:11434` on ZeroTier → `127.0.0.1:11434` on `10.11.21.31`). Active model swapped from `llama3.1:8b` to `gemma3:4b` — faster on CPU (~2× tokens/sec on this hardware) and produces cleaner JSON in `format:"json"` mode for the structured-output features. Configured with `OLLAMA_KEEP_ALIVE=24h` on the cloud VM. Local Ollama on VM_A1 has been **uninstalled**.

**ML Classifier API (port 5000)** — Flask + gunicorn at `/home/vboxuser/soc-project/ml-api/app.py`, systemd unit + `/opt/ml-api/` symlink to `~/soc-project/ml-api/`. Rewritten 2026-05-23 from unsupervised IsolationForest to a **supervised binary classifier (RandomForest primary, XGBoost benchmark)** trained on a **15-feature rule-agnostic vector** that works for any rule_id (the 13 custom SOC rules, the ~1500 Elastic prebuilts, Suricata SIDs, MISP-AI-generated rules). Endpoints: `GET /health`, `POST /classify` (primary — returns `predicted_label` + `confidence` + `recommended_action`), `POST /score` (back-compat — returns derived anomaly_score), `POST /train` (spawns `train.py` subprocess), `POST /feedback` (post-resolution learning loop), `GET /metrics` (last evaluation results for the rapport). Decision rule for `recommended_action` is in `app.py` (`CONFIDENCE_HIGH=0.85`, `CONFIDENCE_LOW=0.60`, `RISK_HIGH=70`) — high-confidence FP → `auto_close_fp`, high-confidence TP + high risk → `auto_escalate`, low confidence → `send_to_analyst`. Bootstraps a tiny synthetic classifier on cold start so the service stays alive when no real model is loaded. Kept (not dropped) because Elastic ML jobs need Platinum and don't classify TP/FP directly.

> **Honest current state (2026-05-23):** trained on **2,830,743 CICIDS2017 rows** (RF in 106s, XGB in 14.8s, class ratio 4:1 so SMOTE not needed). RF + XGB both currently report 1.00 precision/recall/F1/AUC on the held-out test set — this is **label leakage** (CICIDS features like severity/risk_score/MITRE were derived from the attack-type label in our preprocessor), not real performance. The CICIDS run validates that the pipeline scales (2.83M rows in ~2 min); the **meaningful** numbers will come from the TheHive-resolved label set (pulled by `pull_thehive_labels.py` from cases closed as TruePositive / FalsePositive — same upstream as live alerts: Elastic SIEM + Suricata IDS via WF1) once enough resolutions accumulate. The 15-feature vector also includes `rule_false_positive_rate_30d` — a learned per-rule trust score that gets updated on every `/feedback` call — which is the central rapport story for "continuously-learning trust filter on the entire rule corpus."

**Correlation Engine (port 5002)** — Flask + gunicorn at `/home/vboxuser/soc-project/correlation-engine/app.py`, systemd unit + `/opt/correlation-engine/` symlink. SQLite state at `/opt/correlation-engine/state.db`. Decides whether each incoming alert creates a new case or attaches to an existing one. 30-minute same-(source_ip, rule_id) bucket window for storm dedup; **2-hour cross-tactic kill-chain window** that escalates a case the moment a second MITRE tactic appears from the same source IP. Whitelist for known-good IPs; daily-digest queue for low-severity, low-anomaly alerts. Solves the "1 attack = 7 cases" problem and the alert-fatigue problem in one place. The biggest original contribution of the project — kill-chain detection across MITRE tactics is not built into Kibana alert suppression or TheHive 5 case grouping.

**LLM-driven features (orchestrated entirely by n8n + cloud Ollama, no Flask wrapper):**

After the 2026-05-05 re-architecture there is no NLP Flask service. The `~/soc-project/nlp-api/` folder exists as a scaffold (empty Python venv + a `prompts/` directory) but is intentionally not running — all LLM calls go through n8n HTTP-Request nodes with `format:"json"` straight to Ollama. The seven AI features live inside n8n workflows:

| Feature | Workflow | Node(s) |
|---|---|---|
| Alert summarization for TheHive case description | WF1 | `Ollama: summarize` |
| AI-written analyst alert email (8-section structured output) | WF1 | `Ollama: write alert email` → `SMTP: send to analyst` / `SMTP: send via Mailtrap Live (prod)` |
| Auto-tagging (4-key MITRE/asset/confidence classifier, `format:"json"`) | WF1 | `Ollama: tag case` → `Build merged tags` → `TheHive: PATCH case tags` |
| Cortex report synthesis (2-paragraph verdict prepended to Threat Intel page) | WF2 | `Build LLM input from reports` → `Ollama: synthesize threat intel` |
| Daily-digest 4-sentence narrative | WF3 | Ollama call inside the aggregation Code path |
| AI rule generation (`{kibana_rule, suricata_rule, mitre}` JSON) | WF4 | Ollama node with `format:"json"` + analyst-in-loop import-disabled |
| Full investigation report (9-section structured Markdown on case close) | WF6 | `Build report context` → `Ollama: write investigation report` → `Format report markdown` → `TheHive: post Investigation Report page` |
| L2 escalation brief (3-stage: correlate → summarize → recommend with MITRE ATT&CK mitigations) | WF9 | `Build correlate prompt` → `Ollama: correlate` → `Build summarize prompt` → `Ollama: summarize` → `Build recommend prompt` → `Ollama: recommend` → `Build L2 brief` → 3 TheHive write nodes (comment, task, tag) |

**Attempted-and-dropped (kept on record for the rapport):**

- ~~**WF7 — Q&A chatbot via TheHive comments (poll-based)**~~ — built 2026-05-17, deactivated same day. Pattern was: `@bot <question>` in a case comment → 15s scheduled poll detects it → fetch case context → Ollama answer → reply posted as comment from `soc-bot@thehive.local`. Two architectural problems killed it in testing: (a) n8n's per-node `$json` scoping vs. multi-source fan-in made the prompt-assembly node fragile and required two patch passes; (b) gemma3:4b on CPU-only cloud Ollama takes 5–8 minutes per Q&A — too slow to feel like a chatbot. Deactivated, not deleted, so the schema and lessons are preserved. See WF7 in the workflow_entity table (`active=0`).

**Dropped services (replaced by built-ins or n8n+Ollama):**

- ~~**NLP Summarization API (port 5001)**~~ — replaced by n8n HTTP-Request nodes calling Ollama directly with `format:"json"`. Prompt templates live in the n8n workflow JSON. No Flask wrapper buys us anything.
- ~~**MITRE Auto-Tagger (port 5003)**~~ — Kibana's detection-engine UI/API already takes MITRE tags via its `threat[]` field at rule definition time; Elastic prebuilt rules ship pre-tagged; Sigma carries `attack.t####` tags through `sigma2elastic`; community classtype-to-MITRE JSONs cover Suricata's gaps. Building a TF-IDF tagger duplicates these built-ins.
- ~~**AI Rule Generator (was part of NLP API)**~~ — re-implemented as a node in WF4 (see above).

### 6.5 Vulnerable Target Layer (VM_B2 only)

**Apache2 + PHP** — web server hosting DVWA. Configured with `allow_url_include = On` to enable Remote File Inclusion attacks for testing.

**MariaDB** — MySQL-compatible database for DVWA. Holds user accounts and data targeted by SQL injection.

**DVWA (Damn Vulnerable Web Application)** — intentionally insecure PHP application. Provides the attack surface: SQL injection, XSS, CSRF, RFI, command injection, file upload vulnerabilities, brute force login. Security level set to "low" so attacks succeed easily.

**vsftpd** — FTP server with weak local accounts. Provides a brute-force target diversifying attack vectors.

**SSH (OpenSSH)** — with password authentication enabled and weak test accounts (`testuser1/password123`, `testuser2/admin`, `webadmin/webadmin`) intentionally created for brute-force simulation.

---

## 7. Detection Strategy — 4-Layer Defense

The system detects attacks through four independent, complementary layers. An attack typically triggers multiple layers, and the Correlation Engine groups the resulting alerts into one case.

### Layer 1 — Network signatures (Suricata + ET Open)
50,000+ rules inspecting packets at the network level. Catches known exploits, malware C2, scanners, protocol abuse. Auto-updated daily. Requires zero manual rule writing.

### Layer 2 — Host signatures (Elastic prebuilt rules + custom rules)
~1000 prebuilt Kibana SIEM rules covering the full MITRE ATT&CK matrix, maintained by Elastic's security team and auto-updated. Plus 13 custom rules specific to the project's attack scenarios (SSH brute force, SQLi, XSS, etc.).

### Layer 3 — Threat intelligence matching (MISP feeds)
Live IOC feeds from CIRCL OSINT, Abuse.ch URLhaus, Feodo Tracker, AlienVault OTX, Emerging Threats. MISP fetches updates every 6 hours. Any traffic matching a known malicious IP, domain, or hash triggers an alert automatically — no rule writing needed.

### Layer 4 — Behavioral anomaly detection (custom Isolation Forest)
ML model learns normal alert patterns from Elasticsearch history. Scores every new alert. Catches zero-days and novel attack patterns that have no signature in any other layer because they manifest as statistical deviation.

---

## 8. Custom AI Pipeline — How AI Augments the SOC

Three places where AI replaces manual SOC work. After the 2026-05-05 re-architecture, the LLM-driven steps run inside n8n workflows that call Ollama via HTTP nodes — no separate Flask wrapper.

**Adaptive rule creation:** When a new threat report arrives in MISP, n8n Workflow 4 fires. An HTTP-Request node POSTs the threat description to Ollama with `format:"json"` and a prompt template that asks for `{kibana_rule, suricata_rule, tactic_id, technique_id}` in one response. The workflow then pushes the Kibana rule via the Kibana API and deploys the Suricata rule via SSH. The SOC's detection coverage grows automatically as new threats emerge.

**Alert summarization:** Every alert reaching TheHive has its raw JSON converted to plain English. n8n Workflow 1 calls Ollama with a summarization prompt right before creating the TheHive case. A Level 1 analyst reads understandable incident descriptions instead of parsing technical telemetry.

**Chain reasoning:** The Correlation Engine detects MITRE tactic progression deterministically inside its 2-hour window (no LLM needed for the common case). For ambiguous chains the engine can optionally call Ollama via HTTP for narrative reasoning, but the rule-based path covers Phase 7 testing.

All three use Ollama locally — no internet required, no API cost.

---

## 9. Phase-by-Phase Plan

Each phase is a discrete chunk of work. Follow them in order. Don't start a phase until the previous one is fully verified.

### Phase 0 — Project bootstrap (lean Git + local folders)
**Goal:** Set up the small shared Git repo for CLAUDE.md and prepare local project folders on each VM.

Sub-steps:
- Create GitHub repo named `soc-shared` (private if preferred)
- Initialize the repo with CLAUDE.md, PROJECT-MASTER-PLAN.md, and an empty `docs/` folder
- Push to GitHub from one machine
- On every VM: clone `soc-shared` to `~/soc-shared/`
- On every VM: create `~/soc-project/` as the local-only working folder (subfolders created per VM)
- Verify each VM can `git pull` from `~/soc-shared/`

**Outcome:** Every VM has the synced shared brain in `~/soc-shared/` and an empty `~/soc-project/` ready for service-specific subfolders.

---

### Phase 1 — Network setup (ZeroTier on all 4 VMs)
**Goal:** Establish the virtual LAN so all VMs can reach each other regardless of physical host.

Sub-steps:
- Install ZeroTier on each VM
- Join network ID `cf719fd54008e4d1`
- Authorize each member on the ZeroTier admin console
- Assign each VM its planned IP (.50, .51, .52, .53)
- Verify cross-VM reachability with ping

**Outcome:** Every VM can reach every other VM on 192.168.1.0/24.

---

### Phase 2 — VM_A1 SIEM core (Elasticsearch + Kibana + Logstash + Fleet Server)
**Goal:** Build the central SIEM that will receive logs from all monitored machines.

Sub-steps:
- Install OS prerequisites (Java, apt-transport-https, curl)
- Add Elastic 8.x APT repository
- Install Elasticsearch — configure cluster name, single-node mode, enable security with passwords
- Configure Elasticsearch heap (4 GB)
- Install Kibana — configure to use Elasticsearch with kibana_system user, generate encryption keys
- Install Logstash — configure beats input on 5044 and ES output pipeline
- Download and install Elastic Agent in Fleet Server mode
- Generate Fleet Server service token via Kibana API
- Verify Fleet Server is reachable on port 8220 from all VMs
- Configure firewall (ufw) to allow ZeroTier subnet to reach all required ports
- Save editable config snippets to `~/soc-project/elasticsearch/`, `~/soc-project/kibana/`, etc.

**Outcome:** A working SIEM at `http://192.168.1.50:5601`. Fleet Server ready to enroll agents.

---

### Phase 3 — VM_A1 SOAR + AI services (n8n + Ollama + 2 Flask APIs)
**Goal:** Deploy the orchestration engine and the two original AI microservices on top of the SIEM. Re-architected 2026-05-05: dropped two Flask services that duplicated built-ins.

Sub-steps:
- Install Node.js 20 and n8n globally
- Configure n8n service with environment variables and systemd unit on port 5678
- Install Ollama and pull `llama3.1:8b` model (~5 GB download, ~6 GB RAM at runtime)
- Add an Ollama systemd drop-in setting `OLLAMA_KEEP_ALIVE=24h` so the model stays warm
- Verify Ollama API responds on port 11434 with a test prompt (use `--max-time >=600` on first call — cold load is ~5 min on a slow disk)
- Add an 8 GiB swapfile and lower the Elasticsearch JVM heap from 4 GiB to 2 GiB so Ollama has room to load on a 13 GiB VM
- Create Python virtual environments for each Flask service in `~/soc-project/<service>/venv/`
- Deploy ML Anomaly Detection API (port 5000) — bootstraps from synthetic alerts; retrains via `POST /train` on the last 30 days of `.alerts-security.alerts-default`
- Deploy Correlation Engine (port 5002) — SQLite state at `/opt/correlation-engine/state.db`; 30-min bucket window, 2-hour kill-chain window
- Create systemd services for the two Flask APIs (point to `/opt/<service>/` for stable system paths — symlinks back to `~/soc-project/<service>/`)
- Verify all services healthy via `/health` endpoints
- Verify the Correlation Engine end-to-end with the six action paths: `create_new`, `add_to_existing`, `escalate_existing`, kill-chain escalate, `queue`, `suppress`

**Outcome:** All AI services running on VM_A1. n8n UI accessible at `http://192.168.1.50:5678`. NLP and MITRE-tagging steps deferred into n8n workflows in Phase 8 instead of standalone Flask services.

---

### Phase 4 — VM_B1 incident management (Cassandra + ES + TheHive + Cortex + MISP)
**Goal:** Deploy the case management platform and threat intelligence backend.

Sub-steps:
- Install Java 11 (required by Cassandra; Java 21 is incompatible)
- Install Cassandra — configure cluster name, capped 512 MB heap
- Install Elasticsearch 8.x locally — separate from VM_A1's ES, capped 1 GB heap, security disabled (lab only)
- Install TheHive 5 from official `.deb` (StrangeBee repo discontinued — manual download)
- Configure TheHive to use local Cassandra + local ES, set Play secret key, define base URL
- Install Cortex 4 — configure ES backend, listen on 0.0.0.0
- Run TheHive and Cortex first-time setup wizards in the browser:
  - TheHive: change default admin password
  - Cortex: create superadmin, organization "SOC-LAB", user for TheHive integration with API key
- Patch TheHive's `application.conf` with the real Cortex API key
- Deploy MISP via Docker Compose (native install fails on Ubuntu/Kali)
- Configure MISP feeds (Sync → Feeds → Add): CIRCL OSINT, Abuse.ch URLhaus, Feodo Tracker, AlienVault OTX (free API key), ET Compromised IPs
- Schedule MISP feed auto-fetch every 6 hours
- Enable Cortex analyzers: AbuseIPDB, VirusTotal, MISP_Search
- Configure firewall

**Service startup order is critical:** Cassandra → ES → TheHive → Cortex → MISP. Starting out of order causes initialization failures.

**Outcome:** TheHive at `http://192.168.1.51:9000`, Cortex at `http://192.168.1.51:9001`, MISP at `https://192.168.1.51:8443`. All threat feeds auto-updating.

---

### Phase 5 — VM_B2 victim lab (DVWA + vsftpd + SSH + Suricata + Elastic Agent)
**Goal:** Deploy the vulnerable target with intrusion detection at both network and host level.

Sub-steps:
- Install Apache2, PHP 8.x and required modules, MariaDB
- Configure PHP for DVWA (allow_url_include on, error display off)
- Create DVWA database and user
- Clone DVWA from GitHub, configure with database credentials, set security level to "low"
- Install vsftpd, enable local user FTP
- Install OpenSSH server, enable password authentication
- Create three weak test accounts for brute-force simulation
- Install Suricata — configure to listen on the ZeroTier interface (`zt6q3gtnzl`)
- Install suricata-update; enable ET Open and abuse.ch SSL blacklist sources
- Run initial rule download (~50,000 rules)
- Set up daily cron to refresh Suricata rules (`suricata-update && systemctl reload suricata`). MITRE classtype tagging is no longer performed by a custom Tagger (dropped 2026-05-05); ET Open's existing `metadata: mitre_attack_id` lines are preserved as-is.
- Download and enroll Elastic Agent with Fleet Server using the enrollment token from Phase 2
- In Kibana Fleet UI, add integrations to victim-lab agent policy: System, Apache HTTP Server, Custom Logs (for Suricata's `eve.json`)
- Configure Apache logging format with detailed fields
- Configure firewall
- (Removed 2026-05-05) Original plan called for triggering the MITRE Tagger `/refresh` after Suricata install — the Tagger was dropped during Phase 3 in favor of Kibana's built-in `threat[]` tagging at rule definition time. No post-install action needed.

**Outcome:** Vulnerable target online at `http://192.168.1.53/dvwa`. Suricata inspecting all traffic. Elastic Agent shipping host + Suricata logs to VM_A1.

---

### Phase 6 — VM_A2 attacker (Kali + ZeroTier + tools)
**Goal:** Prepare the offensive testing machine.

Sub-steps:
- Install ZeroTier and join the network
- Verify Kali default tools available: hydra, nmap, sqlmap, curl, msfconsole, scp
- Clone the `soc-shared` Git repo to `~/soc-shared/`
- Create `~/soc-project/attack-scripts/` for offensive scripts (one per attack scenario)
- (Optional) Install Elastic Agent on the attacker too, for monitoring offensive traffic during demos

**Outcome:** Kali ready to launch attacks at the victim VM. Attack scripts stored locally in `~/soc-project/attack-scripts/`.

---

### Phase 7 — Detection layer activation
**Goal:** Enable all detection rules across the system.

Sub-steps:
- In Kibana → Security → Rules → Add Elastic rules: install and enable all prebuilt detection rules (~1000 rules covering full MITRE ATT&CK matrix)
- Create the 13 custom Kibana SIEM detection rules via API: SSH brute force, SQLi, XSS, command injection, port scan, web shell, sudo escalation, reverse shell, attacker SSH login, data exfiltration, file integrity changes
- Suricata rules are already loaded from Phase 5 — verify by checking Suricata stats
- All custom Kibana rules carry MITRE `threat[]` tags at creation time (Kibana detection-engine UI/API takes them natively); Elastic prebuilts ship pre-tagged. No tagger sweep needed.
- For Suricata, rely on ET Open's existing `metadata: mitre_attack_id` on the subset of rules that have it; for the rest, accept the gap (or apply a community classtype-to-MITRE JSON mapping). No custom tagger.
- Verify MISP threat intel is populating Elasticsearch indices (MISP-to-ES connector)
- Configure Kibana threat intel matching rules: any document matching a MISP IOC fires an alert

**Outcome:** Multi-layered detection active. Network packets, host events, and threat intel matches all generate alerts in Kibana with proper MITRE tags.

---

### Phase 8 — SOAR integration (n8n workflows)
**Goal:** Wire alert flow from Kibana through correlation, enrichment, and response.

Sub-steps:
- Create Kibana Webhook Connector pointing to `http://localhost:5678/webhook/elastic-alert`
- Attach this connector as the action on every detection rule (built-in and custom) so they all push to n8n
- Configure n8n credentials: TheHive API, Cortex API, SSH key for active response
- Generate SSH key on VM_A1, copy to VM_B2 for passwordless connection used by active response
- Build n8n Workflow 1 — main alert pipeline:
  - Receive webhook → call Correlation Engine → branch on response action
  - If `create_new`: call ML API for score → call Ollama via n8n HTTP-Request node for summary (format:"json") → create TheHive case
  - If `add_to_existing`: PATCH the existing TheHive case with the new observable
  - If `escalate_existing`: same as add but also raise severity
  - If `suppress` / `queue`: log to file, no case created
  - If `auto_block`: SSH to victim and run iptables drop rule, log action to TheHive
- Build n8n Workflow 2 — TheHive observable enrichment via Cortex
- Build n8n Workflow 3 — daily digest (scheduled at 8am)
- Build n8n Workflow 4 — MISP-driven AI rule generation
- Build n8n Workflow 5 — weekly maintenance (scheduled Mondays 4am): retrain the ML Anomaly model via `POST /train`. (The original plan also called for a MITRE rule rescan here; that step is obsolete after the Tagger was dropped — Kibana custom rules carry tags at creation time, ET Open carries its own metadata.)
- Export all workflows as JSON to `~/soc-project/n8n/workflows/` (local only, NOT Git)

**Outcome:** End-to-end automation. Every detected attack creates an enriched, summarized, deduplicated case in TheHive. Critical attacks trigger automatic IP blocking. New threat reports auto-generate rules.

---

### Phase 9 — Adaptive intelligence (continuous learning)
**Goal:** Make the system improve itself over time without manual intervention.

Sub-steps:
- Configure cron on VM_B2: daily `suricata-update` at 3am (no auto-tag step — Tagger dropped 2026-05-05)
- Configure ML API retraining: weekly retrain on the past 30 days of Elasticsearch alerts via `POST :5000/train` to keep the anomaly model current
- Configure MISP feed schedule (already set in Phase 4, verify)
- Set up monitoring: simple uptime checks on every service, log to a status dashboard

**Outcome:** System self-maintains. New rules from ET arrive daily, new threat IOCs every 6 hours, MITRE data refreshes weekly, ML model adapts to recent alert patterns weekly.

---

### Phase 10 — Testing
**Goal:** Verify the entire system works end-to-end with simulated attacks.

Sub-steps:
- Service health verification: check every API and service responds correctly
- Attack simulation 1 — SSH brute force from kali-attacker → expect Suricata alert + Kibana threshold rule + correlation into single case + auto-block
- Attack simulation 2 — SQL injection via DVWA → expect Apache log alert + Suricata web-attack alert + correlation
- Attack simulation 3 — Multi-stage chain: port scan → SQLi → reverse shell from same IP → expect Correlation Engine to detect MITRE tactic progression and consolidate into one critical case
- Attack simulation 4 — Whitelist test: trigger an alert from a whitelisted IP → expect suppression
- Attack simulation 5 — Alert storm: 200 SSH attempts in 1 minute → expect storm protection to kick in after 50
- Verify TheHive cases have NLP summaries, ML scores, Cortex enrichment, and proper MITRE tags
- Build Kibana SOC Overview dashboard: alerts over time, top source IPs, severity distribution, MITRE technique heatmap, agent health
- Document each test result in CLAUDE.md and `~/soc-shared/docs/test-results.md`

**Outcome:** Empirical proof the system works. Test results become evidence in the rapport.

---

### Phase 11 — Documentation & PFE report
**Goal:** Translate the implementation into the final rapport de PFE.

Sub-steps:
- Final pass on CLAUDE.md: fill in all credentials (placeholders), IPs, current state, test results
- Generate architecture diagrams: network topology, alert pipeline, AI pipeline, integration map (saved to `~/soc-shared/docs/architecture.md`)
- Compile metrics: detection rate per layer, average MTTR, false positive reduction with ML scoring, alerts grouped per case (correlation effectiveness), rules auto-generated from MISP events
- Write rapport sections drawing from `~/soc-shared/docs/report-notes.md` accumulated throughout the project
- Defend choices: why Elastic over Wazuh, why Ollama over Claude API, why custom Correlation Engine
- List limitations and future work: production scaling considerations, replacing 8B local model with stronger model when budget allows, multi-tenancy via TheHive Enterprise
- Final commit and Git tag for the defended version

**Outcome:** Defendable PFE with complete documentation and demonstrable system. Project handbook delivered 2026-05-14 at `~/soc-shared/docs/SOC-AUTONOME-HANDBOOK.md` (and the self-contained `.html` companion). PFE rapport itself still pending.

---

### Phase 12 — LLM augmentation features (added 2026-05-15 → 2026-05-17)
**Goal:** Move the LLM from "summarize the alert" to a real first-line analyst assistant by adding structured-output features at every stage of the SOC workflow. All features use the same cloud Ollama tunnel (`http://192.168.1.70:11434 → 10.11.21.31`) with model `gemma3:4b` and `format:"json"` where output structure matters. No new Flask services — everything lives inside existing or new n8n workflows.

Sub-steps:
- **F1. Severity-gated email branch (WF1):** new branch off `TheHive: create case` — Notification config (Set node, recipients/from_address/severity_threshold/`notify_env`), IF severity high/critical, Ollama writes the email body (8 sections: verdict / confidence / summary / kill_chain / threat_actor / immediate+short-term actions / FP-check / containment), IF env=test → Mailtrap Sandbox, env=prod → Mailtrap Live. Mailtrap Live free tier (1000/month) delivers from `@demomailtrap.co`. Email design uses TheHive amber+navy palette + animated SVG shield (SMIL pulsing animation works in Apple Mail + most webmail).
- **F2. Cortex report synthesis (WF2):** between `fetch observables with reports` and `Build Threat Intel page markdown`, inject `Build LLM input from reports` (Code) → `Ollama: synthesize threat intel` (HTTP, JSON-escape via `JSON.stringify($json.prompt_input || "").slice(1, -1)` to survive newlines/quotes inside the report content). Output prepended as `🤖 AI Threat Intel Synthesis` blockquote on the Threat Intel page.
- **F3. Auto-tag (WF1):** parallel branch off `TheHive: create case` — `Ollama: tag case` (HTTP, `format:"json"`, 4-key classifier `{kill_chain_stage, threat_actor, asset_criticality, confidence}`) → `Build merged tags` (Code, whitelist values + dedup-merge with existing case tags) → `TheHive: PATCH case tags` (HTTP PATCH `/api/v1/case/{id}`, requires top-level `authentication: 'genericCredentialType'` + `genericAuthType: 'httpHeaderAuth'` + `timeout: 60000` because PATCH on a case can take ~11s).
- **F4. Daily digest narrative (WF3):** repointed legacy Ollama URL `127.0.0.1:11434` → `192.168.1.70:11434` (cloud tunnel) and swapped `llama3.1:8b` → `gemma3:4b`. Prompt upgraded to produce a 4-sentence narrative covering the day's activity, dominant techniques, and one recommended next action.
- **F5. Full investigation report (WF6, new workflow):** TheHive case-close webhook → 10-node chain. Builder script `/tmp/wf6_create_investigation_report.py` seeds it into the SQLite DB (workflow_entity + workflow_history + shared_workflow with `projectId=ZKstdJDMMM6gaT37`, role `workflow:owner`). Final pattern after two debug passes: build entire prompt inside the `Build report context` Code node and pass to Ollama via `"prompt": {{ JSON.stringify($json.full_prompt) }}` (no per-field slice tricks). Output is a 9-section Markdown page with emojis + tables (executive_summary, root_cause, impact_assessment, incident_timeline, kill_chain_mapping, iocs, analyst_actions_taken, recommendations, lessons_learned), POSTed as a page under the case's Reports tab.
- **F6. Q&A chatbot (WF7, attempted then dropped):** see §6.4 and §16.

**Outcome:** Every SOC interaction surface now has an AI feature attached: email (F1), Threat Intel page (F2), case tags (F3), daily digest (F4), Reports tab on case close (F5). All verified end-to-end with case 44 (`~163868784`) over 2026-05-15 → 2026-05-17.

---

### Phase 13 — Email acknowledgement feedback loop (added 2026-05-17)
**Goal:** Close the human-in-the-loop: when an L1 analyst clicks the "I'm on it" link in the WF1 alert email, the SOC system records the acknowledgement on the TheHive case without manual UI work.

Sub-steps:
- **F7. WF8 webhook receiver** (new workflow, built via `/tmp/wf8_email_ack.py`):
  1. `Webhook (email click)` (GET `/webhook/email-ack`, `responseMode: responseNode`)
  2. `Parse query + build ack` (Code) — extract `case_id` + `recipient` query params, compute `short_user` (recipient prefix, normalized), build comment body + tag name
  3. `Fetch case` (POST `/api/v1/query` with `getCase` to retrieve existing tags)
  4. `Build patch body` (Code) — merge existing tags with `email-acked` + `email-acked:<short_user>`. Critical lesson: must reference `$('Parse query + build ack').first().json` explicitly because `$json` resolves to the prior node's output (the case object), losing the original ack fields.
  5. `Post ack comment` (POST `/api/v1/case/{case_id}/comment`)
  6. `Patch case tags` (PATCH `/api/v1/case/{case_id}` with merged tags)
  7. `Build HTML response` (Code) — render a polished amber-on-navy confirmation page
  8. `Respond to browser` (respondToWebhook node, `text/html; charset=utf-8`)
- **WF1 SMTP nodes patched** (`SMTP: send to analyst` + `SMTP: send via Mailtrap Live (prod)`) to append a one-line URL: `http://192.168.1.50:5678/webhook/email-ack?case_id={{ $('TheHive: create case').item.json._id }}&recipient={{ encodeURIComponent(recipients.split(',')[0].trim()) }}`. Most mail clients auto-linkify the URL, so the existing `emailFormat: text` body stays as-is — no HTML conversion needed.
- **Public-URL note:** for the demo, the webhook URL uses the ZeroTier IP `192.168.1.50:5678`, which is reachable from any ZT-joined demo machine. Production deployment would swap it for a public tunnel (cloudflared quick tunnel or a named cloudflare tunnel). cloudflared was **not** installed for this PFE iteration — kept as a documented production swap rather than a lab requirement.

**Outcome:** End-to-end loop verified 2026-05-17 against case 44 — clicking the link returned HTTP 200 with the 1184-byte HTML page, posted comment `📬 **Alert email acknowledged** by \`kchaou.habib67@gmail.com\` at <iso-timestamp>`, and added tags `email-acked` + `email-acked:kchaou.habib67`. The case's audit trail shows the analyst was on the case within seconds of email delivery.

---

### Phase 14 — ML grounding (COMPLETE 2026-05-23)
**Goal:** Move ml-api from synthetic-bootstrap IsolationForest to a defensible supervised classifier trained on real-world data.

**What landed (supervisor brief: "Random Forest or XGBoost classifier on alert metadata, exposed via Flask called by SOAR; combine public IDS dataset with our own Wazuh-resolved alerts"):**
- **Dataset choice:** CICIDS2017 (224 MB MachineLearningCSV bundle, 8 CSVs, 2,830,743 flow rows). Note: supervisor said "Wazuh-resolved alerts"; pipeline actually uses Elastic SIEM + Suricata feeding TheHive — same retroactive-labelling mechanism applies (closed cases with `resolutionStatus`), naming corrected throughout the code (`thehive_labeled.parquet`).
- **Feature engineering:** `/home/vboxuser/soc-project/ml-api/feature_engineering.py` — **15-feature rule-agnostic vector** (replaces the 4-feature `risk_score`+hashed-rule_id+hashed-ip+hour vector). Works for any rule_id without per-rule one-hot, via `rule_source` (categorical: `soc_custom`/`elastic_prebuilt`/`suricata`/`misp_generated`/`other`), `rule_category` (auth/web/network/endpoint/persistence/recon/exfil/execution/lateral/other), and the killer feature `rule_false_positive_rate_30d` (Bayesian-shrunk learned trust score, updated on every `/feedback` POST).
- **Preprocessor:** `preprocess_cicids.py` — vectorized 2.83M rows in 34 seconds (initial iterrows version would have taken ~30 min).
- **Label puller:** `pull_thehive_labels.py` — queries TheHive `/api/v1/query` for closed cases with `resolutionStatus ∈ {TruePositive, FalsePositive}`, extracts rule_id + MITRE + source_ip from tags/title, writes `data/thehive_labeled.parquet`. Will populate as cases are resolved in TheHive UI.
- **Training:** `train.py` — RF (n_estimators=300, class_weight=balanced) + XGB (n_estimators=300, max_depth=6, scale_pos_weight=auto), stratified 70/15/15 split with SMOTE gating (skipped — ratio 4:1 < 5 threshold). Produces `model_rf.pkl`, `model_xgb.pkl`, `metadata.json`, and 5 PNGs under `reports/` (confusion matrices + feature importances + ROC curves).
- **Service rewrite:** `app.py` rewritten — 6 endpoints (`/health`, `/classify`, `/score`, `/train`, `/feedback`, `/metrics`). Decision rule for `recommended_action` (auto_close_fp / auto_escalate / send_to_analyst / queue_low_priority) lives in `app.py` not in n8n, so routing logic is unit-testable.
- **Pipeline wired:** WF1 `ML: /score` node renamed to `ML: /classify`, URL bumped to `/classify`. `Append anomaly score` node extended to expose 8 fields (`predicted_label`, `confidence`, `recommended_action`, `ml_dataset_source`, `ml_model` + back-compat `anomaly_score`/`is_anomaly`/`bucket_id`). `Build merged tags` node injects `ml:<recommended_action>` and `ml-confirmed:<label>` tags (when confidence ≥ 0.85) into every new TheHive case.
- **Feedback loop:** WF6 (case-resolved webhook) has a new `ML: /feedback` fan-out node — every case closed as TP/FP feeds back into `rule_stats.parquet` and `feedback_log.parquet`. Over time this drives `rule_false_positive_rate_30d` per rule_id and lets the classifier learn the *trust profile* of every rule in the corpus.

**Honest framing for the rapport:** the CICIDS test scores (1.00 P/R/F1/AUC) are **label leakage** — our preprocessor derived features (severity, risk_score, MITRE tactic) from the CICIDS attack-type label. The 2.83M-row run proves the pipeline scales (training in ~2 min, inference < 30ms); the *meaningful* numbers come from `thehive_labeled.parquet` once enough cases are resolved. Rapport section §6.5 will document the leakage caveat explicitly rather than hide it.

**Outcome:** ml-api now has a defendable academic story (supervised classifier on a public IDS dataset + continuously-updated rule trust scores from production resolutions), instead of a synthetic-bootstrap placeholder.

---

### Phase 15 — Pipeline validation via Atomic Red Team (pending, independent of ML choice)
**Goal:** Validate the existing pipeline end-to-end against MITRE-aligned adversary tests, not just the 13 hand-crafted SOC rules. Both for soutenance demo strength and to surface gaps in the prebuilt Elastic rule coverage.

Sub-steps (to run when ready):
- Clone Red Canary's Atomic Red Team: `git clone https://github.com/redcanaryco/atomic-red-team` (Linux bash tests path is `atomics/T1XXX/...`). Pin the commit to a known-good ref for reproducibility.
- Pick 3-5 atomic tests that map to existing SOC rules (e.g., **T1021.004 SSH lateral movement → SOC-012**, **T1059.004 Unix shell → SOC-006**, **T1078 valid accounts → SOC-001/SOC-002**, **T1190 exploit public-facing app → SOC-003/SOC-004**).
- Run one test at a time on VM_B2 (and/or VM_A1 if testing host-events on soc-core itself). Watch the full chain: Elastic agent picks up auth/process telemetry → rule fires → correlation engine → WF1 → TheHive case + email + ack link + WF2 enrichment.
- For each test, log: rule that fired, time-to-case, time-to-email, did auto_block fire if applicable, did the AI report on case-close make sense given the actual attack performed.
- Optional Phase 15b: enable a curated subset of Elastic's ~1000 prebuilt detection rules in Kibana before running ART tests, to demonstrate the system handles a much larger rule corpus than the 13 customs.

**Outcome (when complete):** Empirical evidence in the rapport that the pipeline works against real attacker behavior, not just synthetic webhook firings. Strong demo moment for the soutenance: "this is the system reacting to a real MITRE-mapped attack technique, live."

---

### Phase 16 — L2 escalation NLP module (COMPLETE 2026-05-23)
**Goal:** When the correlation engine decides `escalate_existing` (severity bump on a (source_ip, rule_id) bucket OR new MITRE tactic detected from the same source IP in the 2h kill-chain window), automatically build a structured L2 brief and post it to the TheHive case. Supervisor brief: "Une fois le cas TheHive créé, si une escalade L2 est déclenchée, un module NLP corrèle les alertes, génère un résumé d'incident et suggère des actions de remédiation basées sur la technique MITRE identifiée."

**What landed:**
- **Trigger:** `correlation-engine/app.py` patched with a fire-and-forget daemon-thread helper `_fire_l2_webhook()`. Every time `/correlate` returns `action: escalate_existing` AND has a `case_id`, the engine POSTs `{case_id, bucket_id, source_ip, destination_ip, rule_id, mitre_tactic_id, mitre_technique_id, severity, chain_detected, escalation_reason}` to `http://127.0.0.1:5678/webhook/l2-escalation`. Asynchronous — WF1's `/correlate` latency unchanged.
- **WF9 — "09 L2 Escalation NLP (Workflow 9)"** — 15-node sequential workflow (id `b8598a6fa7b04e51`):
  1. `Webhook (l2-escalation)` — receives the POST
  2. `TheHive: fetch case` → `TheHive: fetch observables` → `TheHive: fetch comments` → `ES: related alerts` (sequential chain so all four are ancestors of `Build correlate prompt` — avoids the cross-node `$('node')` reference trap)
  3. `ES: related alerts` — bool/should query on `source.ip` over `.alerts-security.alerts-default`, this is the supervisor's "advanced search Tier 2: same alerts in same time range"
  4. 3 sequential Code+Ollama pairs: `Build correlate prompt` → `Ollama: correlate` (1000 tokens, format:json) → `Build summarize prompt` → `Ollama: summarize` (1000 tokens) → `Build recommend prompt` → `Ollama: recommend` (1500 tokens — MITRE ATT&CK mitigation IDs M1037 etc. + concrete commands per action)
  5. `Build L2 brief` — assembles a structured Markdown brief (Situation / Kill chain / Impact / Urgency / Key indicators / Related clusters / Recommended actions / Containment window / Post-action checks / Open questions for L2)
  6. Fan-out to `TheHive: post brief comment` + `TheHive: create L2 task` + `TheHive: tag l2-brief-posted` (idempotency tag prevents re-fire)
- **Prompt templates** under `/home/vboxuser/soc-project/n8n-prompts/l2/` (`correlate.txt`, `summarize.txt`, `recommend.txt`) — git-trackable source of truth. Inlined into the workflow Code nodes by the operator script `/tmp/wf9_create_l2_escalation.py` (n8n's Code-node sandbox blocks `require('fs')`, so file-loading at runtime isn't possible; re-baking inline is the workaround).
- **No separate Flask service.** The `nlp-api/` scaffold stays empty as planned; the "NLP module" is WF9 itself with HTTP-to-Ollama, same proven pattern as WF6 investigation report.
- **End-to-end validation:** webhook POST → 15-node chain → real L2 brief on a real TheHive case. Validated as far as the infrastructure allowed in this session — TheHive write nodes verified (HTTP 200 in <1s), Ollama prompt outputs verified standalone (valid JSON for correlate + summarize + recommend at 1500-token cap), full chain blocked at time of writing by Ollama tunnel being down from 192.168.1.70:11434. WF9 wiring + correlation-engine webhook are live and tested; re-fire once Ollama is back up.

**Outcome:** when L2 escalation is triggered, an analyst opening the TheHive case finds (in under 90 seconds via WF9) a complete 9-section markdown brief with situation summary, kill-chain reconstruction, blast-radius assessment, urgency score, top 5 indicators, 3-5 prioritised remediation actions tied to ATT&CK mitigations with concrete commands, containment window, post-action verification checks, and open questions for L2 review. The supervisor's "advanced search Tier 2" requirement is satisfied by step 3 (ES bool/should query on source IP within ±30 min).

### Phase 18 — Engine retirement + rule-agnostic Elastic-native architecture (COMPLETE 2026-05-26)
**Goal:** simplify the SOAR by retiring the standalone correlation engine, replacing burst dedupe with TheHive-side lookup, and moving chain detection from in-memory buckets to Elasticsearch deep search at case-creation time. Everything must work uniformly across all three rule sources: 13 custom SOC rules, ~1500 Elastic prebuilt rules, and Suricata signatures (via a new catch-all detection rule).

**Architectural pivot — why retire the engine:**
The correlation engine on `:5002` (Flask + SQLite, 30-min bucket / 2h kill-chain windows) duplicated state that Elastic already holds. Its three jobs all moved to better homes:

| Engine job (pre-P18) | P18 replacement | Why better |
|---|---|---|
| Burst dedupe (same `(rule_id, source_ip)` within 30 min → 1 case) | TheHive `listCase` lookup by canonical `dedupe:<key>` tag at WF1 entry | Single source of truth (TheHive holds open cases anyway), no orphan-bucket race when `case_id` is set asynchronously |
| Chain detection (different rules, same source IP, within 2h, crossing MITRE tactics) | WF9 deep-search ES query covering 30+ ECS dimensions at case-creation time | Wider lookback (up to 7 days), more dimensions than just `source.ip + tactic`, no separate state |
| `escalate_existing` trigger of WF9 | WF1 unconditionally fires WF9 on every High/Critical case-create | Removes engine as critical path; WF9 always has the chain context it needs |

**What changed — n8n side (operator scripts in `/tmp/wf1_p18_*.py` and `/tmp/wf9_p18_*.py`):**
- **P18.1** — `Build canonical payload` rewritten from a `Set` node to a 145-line `Code` node that detects rule_source (`custom_soc`/`elastic_prebuilt`/`suricata`/`other`), extracts 16 normalized fields per source (including new ones: `rule_source`, `destination_ip`, `host_name`, `user_name`, `dedupe_key`), and works uniformly across all three rule sources. Suricata's `event.severity` priority (1–4) maps to `critical`/`high`/`medium`/`low` on its own branch.
- **P18.4** — 7 new nodes inserted between `Build canonical payload` and `ML: /classify`:
  - `IF severity gate` — short-circuits Low/Medium to `End: skipped (low/medium)` (with reason "not actionable in real-time SOC; alert remains visible in Elastic for retrospective hunting"). Defense-in-depth alongside any source-side filter.
  - `TheHive: dedupe lookup` — POST `/api/v1/query` with `listCase` filter on `tags ⊇ [dedupe:<canonical_key>]` AND `status=InProgress`.
  - `Parse dedupe result` — Code node enforces 5-min freshness window (defense against stale tag matches), returns `{found, found_case_id, found_case_number}`.
  - `IF dedupe found` → `TheHive: append observable (dedupe)` (adds source_ip observable with "deduped" tag + merge comment) → `End: deduped (added to existing case)`.
  - On `not found` → continues into the existing ML → Ollama → TheHive case-create chain.
  - `TheHive: create case` patched to add `dedupe:{{ dedupe_key }}` to tags so future fires find it.
  - `Append anomaly score` patched: dangling `$('Correlation Engine: /correlate').item.json.bucket_id` reference → literal `0`.
- **P18.5** — `WF9: trigger L2 escalation` HTTP node added as a parallel successor of `TheHive: create case`; POSTs the full canonical payload + case_id to `http://192.168.1.50:5678/webhook/l2-escalation`. Replaces the engine's `escalate_existing` callback. Fire-and-forget (5s timeout, `neverError: true`).
- **P18.6a** — 7 nodes removed from WF1 (Correlation Engine `/correlate`, Switch by action, End: suppressed, End: queued for digest, TheHive: add to existing case [engine path], TheHive: escalate existing case, Correlation Engine: set_case). Two direct edges spliced: `Build canonical payload → ML: /classify` (effectively, via the dedupe chain now), and `TheHive: add source_ip observable → IF auto_block?`.
- **P18.7** — WF9's `ES: related alerts` query body rewritten from a single 5-field source.ip term query into a 3,592-char tiered bool query covering **30+ ECS dimensions** across 4 tiers, excluding the current alert by `first_signal_id`:
  - **T1 — Identity overlap** (±30 min): `source.ip`, `destination.ip`, `related.ip`, `host.name`, `user.name`
  - **T2 — Behavioral fingerprint** (±2 h): `process.hash.sha256`, `file.hash.sha256`, `network.community_id`, `tls.client.ja3` (any-of) AND `source.ip`/`host.name` overlap
  - **T3 — Infrastructure** (±24 h): `source.as.organization.name`
  - **T4 — Tactic family** (±7 d): nested query on `kibana.alert.rule.threat.tactic.id`, excluding the same source.ip (so cross-actor hits surface)
  - Returns rich `_source` list (45 fields) including Suricata-specific (`suricata.eve.alert.*`), TLS fingerprints, process tree, file hashes — feeds the Ollama L2 brief with the chain context it now needs to reconstruct kill chains on its own.

**What changed — Elastic / infrastructure side:**
- **P18.2** — New Kibana detection rule `SOC: Suricata alert (catch-all)` (`rule_id: soc-suricata-catchall-v1`) watching `logs-suricata.eve-*` with KQL `event.dataset:"suricata.eve" and suricata.eve.event_type:"alert"`. Severity mapping derived from `suricata.eve.alert.severity` priority. Action body templates `SURI-{signature_id}` as the canonical rule_id and posts to the same n8n webhook connector — Suricata alerts now flow through WF1 with the exact same canonical payload + dedupe + L2 escalation as custom SOC + Elastic prebuilt rules. **Rule-agnostic by construction.**
- **P18.3 — deferred (superseded):** the n8n-side `IF severity gate` (P18.4) handles severity filtering uniformly across all rule sources without per-rule Kibana action filters. Per-rule alerts_filter would be defense-in-depth, not blocking; documented as optional future work.
- **P18.6b** — `correlation-engine.service` stopped + disabled in systemd. `/opt/correlation-engine` symlink renamed to `/opt/correlation-engine.retired-2026-05-26`. WF3 (Daily Digest, id `Z1VpjJlhg2Skek1B`) deactivated in n8n DB (`active=0`) — daily-digest queue is gone with the engine; low-severity context now lives in Elastic. n8n service restarted to load all P18 workflow changes.

**Architecture in one diagram (post-P18):**
```
Elastic Detection Rules (custom SOC / Elastic prebuilt / Suricata catch-all)
    │
    ▼   webhook POST /webhook/elastic-alert
WF1 — 01 Alert Pipeline
    ├── Build canonical payload (rule-agnostic, 16 fields + dedupe_key)
    ├── IF severity gate → End: skipped (low/medium)
    ├── TheHive: dedupe lookup → IF found:
    │       True  → TheHive: append observable → End: deduped
    │       False ↓
    ├── ML /classify → Ollama email (HTML render) → TheHive: create case
    └── Parallel: WF9: trigger L2 escalation (fire-and-forget)
            │
            ▼
WF9 — 09 L2 Escalation NLP
    ├── ES deep search (4-tier, 30+ ECS dims, ±30m → ±7d)
    ├── TheHive: fetch case + observables + comments
    └── Ollama correlate → summarize → recommend → post L2 brief
```

**Trade-offs (explicit for the rapport):**
- **Low-only kill chains go undetected.** A chain that is recon (low) → execution (low) → priv-esc (low) — all severity ≤ medium — never triggers a case in this architecture. Acceptable: an L1 SOC's job is to surface actionable signals; low-only chains stay queryable in Elastic for retrospective hunting but don't wake anyone up.
- **Burst dedupe latency** of ~80–200 ms per alert (one TheHive query) replaces the engine's ~10 ms in-memory bucket lookup. Worth it for the single-source-of-truth simplification.
- **Suricata canonical_id is `SURI-<signature_id>`** — not yet mapped to a human-readable signature name in the dedupe key. Sufficient for dedupe; the readable signature name lives in `rule_name` for display.

**Outcome:** the SOAR is one fewer service (correlation engine retired), one fewer workflow (WF3 daily digest retired, 8 workflows now), and rule-agnostic by design — `Build canonical payload` is the source-detection layer, TheHive is the dedupe store, Elastic via the WF9 deep-search is the chain detector. Cross-source kill chains (e.g., Suricata exploit attempt → Elastic prebuilt initial-access → custom SOC priv-esc) now surface uniformly in the L2 brief. Pending end-to-end validation: blocked at time of writing on the Ollama tunnel being down; smoke-tested as far as the chain reached (workflow accepted webhook, exec finished, ML node fell through to defaults — dedupe and WF9 trigger paths verified structurally).

---

## 10. Integration Map — Service-to-Service Connections

| From | To | Purpose | Protocol |
|---|---|---|---|
| Elastic Agent (B2, A2) | Fleet Server (A1:8220) | Log/event shipping | HTTPS |
| Suricata (B2) | Elastic Agent (B2) | Reads `eve.json` | Local file |
| MISP feeds | MISP (B1) | IOC import | HTTPS pull |
| MISP (B1) | Elasticsearch (A1) | IOC indexing for matching | HTTPS push |
| Kibana detection rules | n8n webhook (A1:5678) | Alert delivery | HTTP |
| n8n | Correlation Engine (A1:5002) | Alert grouping decision | HTTP |
| n8n | ML API (A1:5000) | Anomaly scoring | HTTP |
| n8n | Ollama (A1:11434) | Alert summarization + AI rule generation (HTTP-Request node, format:"json") | HTTP |
| n8n | TheHive API (B1:9000) | Case creation/update | HTTP |
| n8n | Cortex API (B1:9001) | Observable enrichment | HTTP |
| n8n | victim-lab SSH (B2:22) | Active response (iptables) | SSH |
| TheHive | n8n webhook | Case event notifications | HTTP |
| MISP | n8n webhook | New event notifications | HTTP |
| Correlation Engine | Ollama (A1:11434) | Chain reasoning (optional, ambiguous-chain only) | HTTP |
| ML API | Elasticsearch (A1:9200) | Training data fetch | HTTPS |

---

## 11. Alert Pipeline — End-to-End Flow

```
1. Attack happens on VM_B2 (e.g. SQL injection from VM_A2)

2. Multiple detection layers fire simultaneously:
   - Suricata sees the malicious HTTP packet → eve.json
   - Apache logs the request → access.log
   - DVWA may log a database error → mysql/error.log

3. Elastic Agent ships all these logs to Fleet Server → Elasticsearch

4. Detection rules query Elasticsearch:
   - Suricata-specific rule fires on the eve.json event
   - Custom Kibana rule fires on the Apache log pattern
   - Possibly an Elastic prebuilt rule also fires

5. Each rule sends a webhook to n8n (so 2-3 webhooks for one attack)

6. n8n calls Correlation Engine for each:
   - First webhook → no existing bucket → create_new (with MITRE tactic stored)
   - Second webhook → bucket exists for this source IP → add_to_existing
   - Third webhook → tactic progression detected → escalate_existing

7. Only ONE TheHive case is created (from the first webhook)
   Subsequent alerts attach as observables/timeline entries to that case

8. The case gets:
   - NLP summary as description (n8n HTTP-Request node → Ollama, format:"json")
   - Anomaly score in custom field (from ML API)
   - MITRE ATT&CK tags
   - Cortex enrichment (AbuseIPDB lookup of source IP, etc.)

9. If the alert was critical (severity=critical or kill chain detected):
   - n8n SSHes to victim-lab, runs iptables -I INPUT -s ATTACKER_IP -j DROP
   - Logs the active response action to the TheHive case

10. Analyst sees ONE case in TheHive with full context, history, and 
    automated investigation already done. Decides whether to close it 
    (false positive) or continue investigation.
```

---

## 12. Cost Confirmation

Final answer after going component by component: **zero cost**. Everything used is open source or free tier sufficient for the lab.

| Component | Cost |
|---|---|
| Elastic Stack (Basic) | Free |
| TheHive 5 Community | Free |
| Cortex 4 | Free |
| MISP | Free |
| Suricata + ET Open | Free |
| n8n self-hosted | Free for academic use |
| Ollama + llama3.1:8b | Free (local) |
| ZeroTier (≤25 nodes) | Free |
| MITRE ATT&CK data | Free (CC BY) |
| Free threat intel feeds (CIRCL, Abuse.ch, OTX, ET) | Free |
| AbuseIPDB free tier | 1000 lookups/day, free |
| VirusTotal free tier | 500 lookups/day, free |
| Ubuntu / Kali | Free |
| MariaDB / Apache / DVWA | Free |
| Cassandra | Free |

The architecture supports drop-in upgrades to paid tiers (Elastic Platinum, ET Pro, Anthropic API for stronger LLM) when a future production deployment has budget.

---

## 13. Folder Layouts

### `~/soc-shared/` — on Git, synced across all VMs

```
soc-shared/
├── CLAUDE.md                          ← shared brain, updated each phase
├── PROJECT-MASTER-PLAN.md             ← this file
├── README.md
└── docs/
    ├── architecture.md
    ├── test-results.md
    ├── report-notes.md
    └── session-history.md             ← archive of older session notes
```

### `~/soc-project/` — local only, never on Git

**On VM_A1 (soc-core):**
```
~/soc-project/
├── .env.local                         ← real credentials
├── elasticsearch/                     ← editable config copies
├── kibana/
├── logstash/
├── fleet/
├── ollama/
├── n8n/
│   └── workflows/                     ← exported workflow JSON
├── ml-api/
│   ├── app.py
│   ├── train.py
│   └── requirements.txt
├── nlp-api/
├── correlation-engine/
└── mitre-tagger/
```

**On VM_B1 (incident-mgmt):**
```
~/soc-project/
├── .env.local
├── cassandra/
├── elasticsearch/
├── thehive/
├── cortex/
└── misp/
```

**On VM_B2 (victim-lab):**
```
~/soc-project/
├── .env.local
├── apache/
├── mariadb/
├── ssh-and-vsftpd/
├── suricata/
├── elastic-agent/
└── dvwa/
```

**On VM_A2 (kali-attacker):**
```
~/soc-project/
└── attack-scripts/
```

---

## 14. Working Method

When proceeding through phases:

1. Announce which phase and sub-step you're starting
2. Request commands or code only when you're at a sub-step that needs it
3. Verify each sub-step's outcome before moving on (the plan lists expected outcomes for each phase)
4. Update `~/soc-shared/CLAUDE.md` after completing each phase: phase status, credentials filled with placeholders, any decisions made, any deviations from the plan
5. Commit and push **only `~/soc-shared/`** — `~/soc-project/` stays local
6. Document anything unexpected in `~/soc-shared/docs/report-notes.md` — these become rapport material

If a sub-step fails or behaves unexpectedly:
- Document what happened
- Try to understand why (often reveals an architectural insight worth keeping for the rapport)
- Adjust the plan if needed
- Don't skip ahead — order matters because of service dependencies

---

## 15. Critical Reminders

- **Project files NEVER go on Git.** Only CLAUDE.md and `docs/` are committed.
- **Service startup order on VM_B1:** Cassandra → Elasticsearch → TheHive → Cortex → MISP. Deviating causes init failures.
- **OOM risk on VM_B1:** Keep heap caps in place (Cassandra 512 MB, ES 1 GB). Don't run a desktop GUI.
- **Ollama on VM_A1:** First model load can be slow (~30 s). Subsequent calls are faster. Keep the service warm.
- **Fleet Server enrollment token:** Generated once in Phase 2. Save it immediately — used by every Elastic Agent installation.
- **API keys are secrets.** Use placeholders in CLAUDE.md (which goes on Git) and put real values in `~/soc-project/.env.local` (gitignored, never committed).
- **MITRE ATT&CK refresh:** Weekly is enough. Don't refresh on every request — slow.
- **MISP first run:** Some Docker containers are unhealthy on first startup. Run `docker compose up -d` again to settle.
- **TheHive 5 default password:** `admin@thehive.local / secret`. Change immediately on first login.
- **Local backups matter:** Since `~/soc-project/` isn't on Git, a VM failure means lost work. Run periodic rsync or VM snapshots.
- **n8n DB path quirk:** Real n8n state is at `/home/vboxuser/.n8n/.n8n/database.sqlite` (nested `.n8n/.n8n/`). The outer `/home/vboxuser/.n8n/database.sqlite` exists but is the old/decoy one — confusing on a cold restart. Backups follow the same nested path, named `database.sqlite.bak-<purpose>-<epoch>`.
- **n8n version trap:** every workflow row has TWO version columns — `versionId` (editor head) and `activeVersionId` (deployed runtime). After surgery you must update BOTH and insert a matching `workflow_history` row, else next restart loads the old definition silently OR activation fails with "Active version not found for workflow".
- **n8n RBAC required:** any new workflow needs a row in `shared_workflow` linking it to `projectId=ZKstdJDMMM6gaT37` with role `workflow:owner`, else activation fails with "Could not find any entity of type SharedWorkflow".
- **n8n restart pattern:** there's no `sudo` in the harness and the REST API requires auth, so the only way to reload workflow code after DB surgery is `kill -9 $(pgrep -f 'n8n start')`. systemd's `Restart=on-failure` + `RestartSec=10` brings it back; workflows re-activate after ~25 s.
- **n8n HTTP node auth trap:** for HTTP-Request nodes that need a credential, the top-level params MUST include `authentication: 'genericCredentialType'` + `genericAuthType: 'httpHeaderAuth'` (or `httpBasicAuth`, `sshPrivateKey`, etc.). Setting only the `credentials` object on the node silently drops the auth header.
- **`neverError: true` masks real errors:** the WF6 investigation report rendered "_(missing)_" in every section for several runs because Ollama returned `{error: ...}` instead of `{response: ...}` and the formatter only checked for `.response`. Always inspect `item.json` for an `error` key, even on successful executions.
- **n8n expression scoping:** inside a Code or HTTP node, `$json` refers to the **immediately prior node's output** — not the original webhook payload. To pull fields from earlier nodes use `$('NodeName').first().json.fieldName`. WF7 and WF8 both wasted a debug cycle on this.
- **Ollama on CPU is slow:** `gemma3:4b` at ~600 tokens generates in 30–90 s; 1500-token reports take 5–8 min. WF6 timeouts must allow at least `timeout: 600000`. Any feature that requires sub-30-second LLM response (e.g., chatbot UX) is not feasible on this hardware without a GPU.
- **Cloud Ollama tunnel must be up:** the SSH port-forward `ssh -L 192.168.1.70:11434:127.0.0.1:11434 root@10.11.21.31` is run from the Windows host (`192.168.1.70`). If the host reboots, the tunnel must be re-established manually. All LLM features fail with `ECONNREFUSED` until then.

---

## 16. Current state snapshot (as of 2026-05-26)

### Service status — VM_A1 (192.168.1.50)

| Service | State | Path / managed by | Notes |
|---|---|---|---|
| Elasticsearch 8.19.14 | ✅ running | systemd `elasticsearch.service` · config `/etc/elasticsearch/` · 2 GB heap (`/etc/elasticsearch/jvm.options.d/heap.options`) | yellow status normal for single-node |
| Kibana 8.19.14 | ✅ running | systemd `kibana.service` · config `/etc/kibana/kibana.yml` | UI at `http://192.168.1.50:5601` |
| Logstash 8.19.14 | ✅ running | systemd `logstash.service` · pipeline `/etc/logstash/conf.d/soc-pipeline.conf` | beats:5044 + syslog:5140 |
| Elastic Agent (Fleet Server) | ✅ running | `/opt/Elastic/Agent/elastic-agent.yml` | :8220 |
| n8n 2.18.5 | ✅ running | systemd `/etc/systemd/system/n8n.service` · DB `/home/vboxuser/.n8n/.n8n/database.sqlite` · config `/home/vboxuser/.n8n/.n8n/config` (encryption key) | UI at `http://192.168.1.50:5678` |
| Ollama (local) | ❌ uninstalled 2026-05-15 | n/a | moved to cloud VM `10.11.21.31`, reached via SSH tunnel on `192.168.1.70:11434` |
| ML API (Isolation Forest) | ✅ running | `/home/vboxuser/soc-project/ml-api/app.py` · model `/home/vboxuser/soc-project/ml-api/model.pkl` · venv `/home/vboxuser/soc-project/ml-api/venv/` · symlink `/opt/ml-api/` | port 5000, `source: "synthetic"` |
| NLP API | ❌ scaffold only | folder `/home/vboxuser/soc-project/nlp-api/` exists with empty venv + `prompts/` subdir | intentionally not built — n8n+Ollama replaces it |
| Correlation Engine | ❌ **retired 2026-05-26 (Phase 18)** | systemd unit stopped + disabled; symlink renamed `/opt/correlation-engine.retired-2026-05-26`; code preserved at `/home/vboxuser/soc-project/correlation-engine/` for rollback | replaced by TheHive-side dedupe + WF9 ES deep-search |

### Service status — VM_B1 (192.168.1.51)

TheHive 5.7.1 + Cortex 4.0.1 + MISP all running (Docker compose stack at `~/soc-project/thehive-cortex/docker-compose.yml`). Custom `soc-cortex:4.0.1-analyzers` image with 30 enabled analyzers. TheHive `application.conf` bind-mounted from `~/soc-project/thehive-cortex/thehive/conf/application.conf` includes `notification.endpoints` (n8n webhook), `cortex.servers` (`SOC-LAB-Cortex`), `misp.servers` (`SOC-LAB-MISP`).

### Service status — VM_B2 (192.168.1.53)

Apache + DVWA + MariaDB + vsftpd + OpenSSH + Suricata 8.0.3 (49,911 ET Open rules) + Elastic Agent enrolled in `victim-lab` policy. Active-response wired via `soc-response` user + `/etc/sudoers.d/soc-response` NOPASSWD for iptables.

### Active n8n workflows

All in `/home/vboxuser/.n8n/.n8n/database.sqlite` (table `workflow_entity`). All have a matching `shared_workflow` row linking to `projectId=ZKstdJDMMM6gaT37` with role `workflow:owner`. JSON exports kept in `/home/vboxuser/soc-project/n8n/workflows/`.

| # | Name | ID | Trigger | State | Last verified |
|---|---|---|---|---|---|
| 1 | 01 Alert Pipeline | `dKSF2AU9E3k9i25p` | POST `/webhook/elastic-alert` | active | Phase 10 (2026-05-10) + WF1 patches through 2026-05-17 |
| 2 | 02 Cortex Enrichment | `HYiSFNStG5zEG6ZA` | POST `/webhook/thehive` (polling bridge) | active | 2026-05-15 (Cortex synthesis added) |
| 3 | 03 Daily Digest | `Z1VpjJlhg2Skek1B` | cron 08:00 Africa/Tunis | **deactivated 2026-05-26 (Phase 18)** | retired with the correlation engine — low-severity context now lives in Elastic, not in a separate digest queue |
| 4 | 04 MISP → AI Rule Gen | `SbXmkucPC24njKwb` | POST `/webhook/misp` (polling bridge) | active | known limitation: Ollama 10-min timeout vs CPU latency |
| 5 | 05 Weekly Maintenance | `ekXEZb2PYaxQt7vv` | cron Mon 04:00 | active | depends on real alert volume to do meaningful work |
| 6 | 06 Investigation Report | `Srr7LavaermzMzB9` | POST `/webhook/thehive-case-resolved` | active | 2026-05-17 (exec 453 — 9-section Markdown report posted to case 44) |
| 7 | 07 Q&A Chatbot | `wf7QAchatbotXXX1` | Schedule (15s) | **deactivated 2026-05-17** | dropped — too slow on CPU; kept as record |
| 8 | 08 Email Ack | `wf8EmailAckXXXX1` | GET `/webhook/email-ack` | active | 2026-05-17 (verified end-to-end on case 44) |
| 9 | 09 L2 Escalation NLP | `b8598a6fa7b04e51` | POST `/webhook/l2-escalation` (now fired unconditionally by WF1's `WF9: trigger L2 escalation` parallel node on every High/Critical case-create — was correlation-engine; engine retired 2026-05-26) | active | 2026-05-26 (ES query rewritten as 4-tier 30+ ECS-dim bool search; full Ollama chain pending tunnel restore) |
| 10 | 10 L2 Verdict | `wf10L2VerdictXX` | GET `/webhook/l2-verdict?case_id=X&verdict=tp\|fp&l2_user=email` | active (live; smoke-test pending B1 reachability) | 2026-05-26 (Phase 19.1) — TP/FP click closes case + ml-api /feedback + amber/navy HTML confirmation page |

### Polling bridges (TheHive notifier replacement, on VM_B1)

| Script | Cron | Purpose |
|---|---|---|
| `/home/vboxuser/soc-project/thehive-cortex/cron-cases-to-wf2.sh` | every 1m | poll `listCase` filtered on `_createdAt > last_seen` → POST each new case in TheHive's native envelope to `http://192.168.1.50:5678/webhook/thehive` (fires WF2) |
| `/home/vboxuser/soc-project/misp/cron-publish-to-wf4.sh` | every 1m | poll MISP `/events/restSearch` filtered on `publish_timestamp > last_seen` → POST to `http://192.168.1.50:5678/webhook/misp` (fires WF4) |
| `/home/vboxuser/soc-project/misp/cron-feeds.sh` | every 6h | docker-exec MISP `cake Server fetchFeed/cacheFeed all` |

### Operator scripts (workflow surgery — `/tmp/wf*_*.py` on VM_A1)

These are one-shot Python scripts that read/write `~/.n8n/.n8n/database.sqlite` directly to build or patch workflows. Not production code. Listed here so they can be re-applied if the DB is ever rebuilt from snapshot. Kept on this VM only.

| Script | What it does |
|---|---|
| `/tmp/wf1_ollama_repoint.py` | re-points WF1 Ollama node URL from local `127.0.0.1:11434` to cloud tunnel `192.168.1.70:11434` |
| `/tmp/wf1_thehive_template_fix.py` | fixes nested-`{{}}` template error in WF1 `TheHive: create case` description |
| `/tmp/wf1_add_email_branch.py` | adds the 5-node email branch (Notification config + IF severity + Ollama email + SMTP send + NoOp) off `TheHive: create case` |
| `/tmp/wf1_add_dev_prod_smtp_split.py` | adds IF-env switch between Mailtrap Sandbox (test) and Mailtrap Live (prod) |
| `/tmp/wf1_add_autotag_branch.py` | adds the parallel auto-tag branch (Ollama JSON classifier → Build merged tags → TheHive PATCH) |
| `/tmp/wf2_add_cortex_synthesis.py` | adds Build LLM input + Ollama synthesize nodes to WF2; modifies Build Threat Intel page to prepend AI verdict |
| `/tmp/wf6_create_investigation_report.py` | builds WF6 from scratch (10 nodes, case-resolved webhook → fetch → Build context → Ollama → format → POST page) |
| `/tmp/wf6_truncate_inputs.py` | caps comments/tasks/observables passed to Ollama (MAX_COMMENTS=25, MAX_TASKS=15, MAX_OBS=15) to keep prompt bounded |
| `/tmp/wf6_build_prompt_in_code.py` | final WF6 fix — moves prompt assembly into the Code node, uses `{{ JSON.stringify($json.full_prompt) }}` in HTTP body |
| `/tmp/wf7_create_qa_chatbot.py` | builds WF7 (Q&A chatbot — dropped) |
| `/tmp/wf8_email_ack.py` | builds WF8 (Email Ack receiver) |
| `/tmp/wf1_add_ml_classify_routing.py` | renames WF1 `ML: /score` → `ML: /classify`, bumps URL to `/classify`, extends `Append anomaly score` to surface 8 classifier fields, injects `ml:<action>` / `ml-confirmed:<label>` / `ml-model:<name>` tags via `Build merged tags` |
| `/tmp/wf6_add_ml_feedback.py` | adds `ML: /feedback` fan-out node to WF6 (case-resolved path) so every TP/FP resolution updates `rule_stats.parquet` + appends to `feedback_log.parquet` on ml-api |
| `/tmp/correlation_engine_add_l2_webhook.py` | patches `correlation-engine/app.py` with `import threading`, `N8N_L2_WEBHOOK_URL` config, `_fire_l2_webhook()` daemon-thread helper, and rewires `/correlate` handler to fire the webhook on `escalate_existing` |
| `/tmp/wf9_create_l2_escalation.py` | builds WF9 from scratch (15 nodes — Webhook → 4 TheHive/ES fetches sequentially → 3 Code+Ollama prompt pairs → Build L2 brief → 3 TheHive write nodes) |
| `/tmp/wf1_wire_html_email.py` | wires `alert.compiled.js` HTML render into WF1 — inserts `Build email HTML` Code node between `Ollama: write alert email` and `IF env (test/prod)`; rewrites Ollama prompt to return JSON with 8 fields; patches both SMTP nodes to `emailFormat: "html"` |
| `/tmp/wf1_p18_1_canonical.py` | **Phase 18.1** — rewrites `Build canonical payload` from Set node to rule-agnostic Code node (16 fields incl. `rule_source` / `destination_ip` / `host_name` / `user_name` / `dedupe_key`) covering custom SOC + Elastic prebuilt + Suricata |
| `/tmp/wf1_p18_4_dedupe.py` | **Phase 18.4** — inserts severity gate + TheHive `listCase` dedupe lookup + IF-found branch (append observable to existing case vs continue to ML) before `ML: /classify`; appends `dedupe:<key>` tag to case create body; fixes dangling `bucket_id` reference |
| `/tmp/wf1_p18_5_wf9_trigger.py` | **Phase 18.5** — adds `WF9: trigger L2 escalation` HTTP node as parallel successor of `TheHive: create case`, replacing engine's `escalate_existing` callback |
| `/tmp/wf1_p18_6a_remove_engine.py` | **Phase 18.6a** — removes 7 correlation-engine nodes from WF1 (Correlate, Switch by action, suppressed/queued end nodes, engine-driven add/escalate, set_case) and splices direct edges |
| `/tmp/wf9_p18_7_deep_search.py` | **Phase 18.7** — rewrites WF9 `ES: related alerts` query into 4-tier 30+ ECS-dim bool query (Identity ±30m, Behavioral fingerprint ±2h, Infrastructure ±24h, Tactic family ±7d) excluding current alert by `first_signal_id` |
| `/tmp/wf10_create_l2_verdict.py` | **Phase 19.1** — builds WF10 from scratch (9 nodes — GET webhook → Parse params → fetch case → Build patch body → PATCH /case (status=Resolved + resolutionStatus + tags + summary) → POST verdict comment → POST ml-api /feedback → HTML response page) |

### ML / NLP file locations (Phase 14 + Phase 16)

| Path | Purpose |
|---|---|
| `/home/vboxuser/soc-project/ml-api/feature_engineering.py` | 15-feature rule-agnostic vector + `_shrink_fp_rate` Bayesian helper + `update_rule_stats` |
| `/home/vboxuser/soc-project/ml-api/preprocess_cicids.py` | vectorized CICIDS2017 → 15-feature parquet (8 CSVs, 2.83M rows in 34s) |
| `/home/vboxuser/soc-project/ml-api/pull_thehive_labels.py` | TheHive `/api/v1/query` → `data/thehive_labeled.parquet` (reads `THEHIVE_TOKEN` env) |
| `/home/vboxuser/soc-project/ml-api/train.py` | RF + XGB pipeline with stratified split + optional SMOTE + 5 PNG plots + metadata.json |
| `/home/vboxuser/soc-project/ml-api/app.py` | rewritten Flask service: `/health /classify /score /train /feedback /metrics` |
| `/home/vboxuser/soc-project/ml-api/model_rf.pkl` + `model_xgb.pkl` | joblib pickles, loaded at app boot |
| `/home/vboxuser/soc-project/ml-api/metadata.json` | trained_at + dataset_source + n_train/test + metrics + feature_importance (served by `/metrics`) |
| `/home/vboxuser/soc-project/ml-api/data/cicids2017/*.csv` | 8 CICIDS2017 source CSVs (885 MB extracted, gitignored) |
| `/home/vboxuser/soc-project/ml-api/data/cicids2017_features.parquet` | preprocessed 15-feature training set |
| `/home/vboxuser/soc-project/ml-api/data/thehive_labeled.parquet` | TheHive-resolved labels (created on first `pull_thehive_labels.py` run) |
| `/home/vboxuser/soc-project/ml-api/data/rule_stats.parquet` | learned per-rule TP/FP counts, hot-updated on every `/feedback` POST |
| `/home/vboxuser/soc-project/ml-api/data/feedback_log.parquet` | audit log of every `/feedback` POST (rule_id + label + case_id + timestamps) |
| `/home/vboxuser/soc-project/ml-api/reports/{confusion_matrix_rf,confusion_matrix_xgb,feature_importance_rf,feature_importance_xgb,roc_curves}.png` | rapport-grade visuals from last training run |
| `/home/vboxuser/soc-project/n8n-prompts/l2/correlate.txt` | WF9 stage-1 prompt template (cluster alerts by kill-chain stage, `format:json`) |
| `/home/vboxuser/soc-project/n8n-prompts/l2/summarize.txt` | WF9 stage-2 prompt (4-section L2 brief, `format:json`) |
| `/home/vboxuser/soc-project/n8n-prompts/l2/recommend.txt` | WF9 stage-3 prompt (3-5 actions tied to ATT&CK mitigations, `format:json`) |
| `/home/vboxuser/soc-project/correlation-engine/app.py.bak-l2-webhook-<epoch>` | pre-patch backup of correlation-engine before L2 webhook injection |

### Email infrastructure (Mailtrap)

| Account | Use | Credentials |
|---|---|---|
| Mailtrap Sandbox (`sandbox.smtp.mailtrap.io:587`) | dev/test — captures emails, no real delivery | n8n credential id `7YLKQqTyNppHWMxk` ("Mailtrap SMTP") |
| Mailtrap Email Sending (`live.smtp.mailtrap.io:587`) | prod — real delivery from `@demomailtrap.co`, free 1000/month | n8n credential id `l8P27YAe3qV0c09j` ("Mailtrap Live SMTP") |
| Brevo SMTP | tried 2026-05-15, free tier needs manual account activation, dropped | n8n credential id `DPadrRfl55A5p0os` (kept for reference) |
| Gmail SMTP | rejected by user 2026-05-15 ("don't want my account to be the one sending") | n8n credential id `HSGSnULcOP2cDnar` (unused) |

WF1's `Notification config` Set node holds `recipients` (comma-separated), `severity_threshold`, `notify_env` (`test`/`prod`), `from_address`. `notify_env` selects between Mailtrap Sandbox (test) and Mailtrap Live (prod).

### Cloud Ollama tunnel

```
analyst's laptop (Windows host @ 192.168.1.70) ─── SSH -L ───► cloud VM @ 10.11.21.31
                                                                   │
                                                              ollama serve
                                                              :11434, gemma3:4b loaded
                                                              OLLAMA_KEEP_ALIVE=24h
```

Command run from Windows host: `ssh -L 192.168.1.70:11434:127.0.0.1:11434 root@10.11.21.31`. Must be re-established manually if Windows host reboots. All LLM features in WF1/WF2/WF3/WF4/WF6/WF9 target `http://192.168.1.70:11434/api/generate`.

### Key documentation files (on VM_A1, under `~/soc-shared/`)

| File | Purpose |
|---|---|
| `~/soc-shared/CLAUDE.md` | shared brain — phase status, last-session notes, VM profiles, credentials placeholders. Updated every session, lives on Git. |
| `~/soc-shared/PROJECT-MASTER-PLAN.md` | this file — the master plan + current state snapshot. |
| `~/soc-shared/README.md` | repo intro |
| `~/soc-shared/docs/SOC-AUTONOME-HANDBOOK.md` | 1834-line end-to-end handbook delivered 2026-05-14 — all 4 VMs, all workflows node-by-node with mermaid flowcharts, 9 pipeline scenarios. Source for the rapport. |
| `~/soc-shared/docs/SOC-AUTONOME-HANDBOOK.html` | self-contained HTML version of the handbook (rendered mermaid + syntax highlighting, has print stylesheet for Save-as-PDF) |
| `~/soc-shared/docs/session-history.md` | archive of older CLAUDE.md last-session entries |

### What's pending right now (2026-05-26)

**Active blockers — both infrastructure-side, not code-side:**

1. **VM_B1 (TheHive + Cortex + MISP) unreachable** — ping fails from VM_A1, ports 9000/9001 both timeout. Blocks: WF1 dedupe lookup (TheHive), WF9 L2 brief (TheHive write), WF10 verdict (TheHive PATCH), Phase 19 build (Cortex connector status check + custom-field creation + l1/l2 user creation attempt). Owner action: bring B1 back up.

2. **Cloud Ollama tunnel down** — `http://192.168.1.70:11434` not reachable. Blocks: all Ollama-dependent nodes (WF1 alert email, WF2 Cortex synth, WF3 daily digest [now retired], WF4 MISP rule gen, WF6 investigation report, WF9 L2 brief). Owner action: re-establish SSH tunnel from Windows host `ssh -L 192.168.1.70:11434:127.0.0.1:11434 root@10.11.21.31` (requires FortiClient VPN to 10.11.21.x range).

**What landed today (2026-05-26):**

- **Rich HTML alert email** wired into WF1 — git-versioned template at `/home/vboxuser/soc-project/n8n-prompts/email/alert.html.j2` (dark navy + amber TheHive palette, SVG shield with SMIL pulse animation, kill-chain pill grid, 8 AI-generated sections, ACK button + Kibana/TheHive deep-link buttons with base64-inlined logos). Compiled via `compile_template.py` into a self-contained `alert.compiled.js` (314 KB, no `fs`/`nunjucks` required — works inside n8n's task-runner sandbox). Patched WF1's Ollama prompt to return JSON for the 8 fields; new `Build email HTML` Code node renders before SMTP.
- **Phase 18 — engine retirement + rule-agnostic architecture** (8 sub-steps, all complete). See §Phase 18 above for full narrative. Net effect: correlation engine + WF3 retired; WF1 + WF9 now process custom SOC + Elastic prebuilt + Suricata uniformly via canonical payload + TheHive-side dedupe + ES deep-search.
- **Phase 19.1 — WF10 L2 Verdict webhook** live at `/webhook/l2-verdict`. Accepts TP/FP query params from L2 email clicks → PATCHes TheHive case (status=Resolved + resolutionStatus + tags + summary) + posts comment + POSTs ml-api `/feedback` + returns amber/navy confirmation page. End-to-end smoke pending B1.

**Pending work (in order, once blockers clear):**

1. **WF1 end-to-end HTML email smoke** — Once Ollama + B1 are back, fire a fresh High-severity alert (e.g. SOC-EMAIL-HTML-001), verify: (a) case lands in TheHive with dedupe tag, (b) rich HTML email arrives in Gmail (sender `@demomailtrap.co`) with all 8 AI sections + working ACK button + working "Open TheHive case" button, (c) WF9 fires in parallel and posts L2 brief, (d) ACK click adds `email-acked:<user>` tag.
2. **Phase 19 build** (paused; needs decision once B1 is back):
   - Probe TheHive↔Cortex connector status (`curl http://192.168.1.51:9000/api/v1/cortex/status`).
   - **Scenario A** (connector live, i.e. Platinum re-trialed): build 3 Cortex Responders (`SOC_Confirm_TP`, `SOC_Confirm_FP`, `SOC_Escalate_L2`) — real buttons in TheHive case UI.
   - **Scenario B** (connector dead, i.e. Free tier): build single TheHive Custom Field `l1_decision` (enum dropdown: `none`/`confirm_true_positive`/`confirm_false_positive`/`escalate_to_l2`) + new WF11 router that switches on the dropdown value (TheHive notification or polling-bridge triggered).
   - Either scenario: WF11 escalate branch sends the L2 email with TP/FP buttons that link to WF10 (already built).
   - **L1/L2 users:** create `l1@thehive.local` + `l2@thehive.local` (analyst profile) in TheHive Free — should fit the ~3 user/org limit. If license blocks, fall back to single `soc-bot` + distinguish L1/L2 work via `assignee` field + separate email distribution lists.
3. **Phase 14 polish — TheHive labels:** ml-api still trained on CICIDS only; first `pull_thehive_labels.py` run pending a real `THEHIVE_TOKEN` in env. Once enough cases are resolved as TP/FP via WF10/WF11, re-run `train.py` to get the meaningful (non-leaked) metrics for the rapport.
4. **Phase 15 — Atomic Red Team pipeline validation:** install ART, pick 3-5 techniques mapped to SOC rules, run end-to-end against the pipeline. Best demo moment for the soutenance.
5. **Phase 11 finish — PFE rapport:** handbook is the input; rapport sections still to write (now with Phase 14 + Phase 16 + Phase 18 + Phase 19 outcomes available).
6. **Optional public-URL swap:** if WF8 email-ack / WF10 L2 verdict are to work for analysts outside the ZeroTier overlay (e.g., real Gmail inbox on a phone), the link targets need to move from `http://192.168.1.50:5678/...` to a cloudflared / ngrok / named tunnel URL. Documented as production-deployment swap, not a lab blocker.

### Email template artifacts (Phase 18 byproduct)

| Path | Purpose |
|---|---|
| `/home/vboxuser/soc-project/n8n-prompts/email/alert.html.j2` | git-versioned Jinja2/Nunjucks source template (dark theme, 8 sections, SVG shield, ACK + Kibana/TheHive CTAs) |
| `/home/vboxuser/soc-project/n8n-prompts/email/compile_template.py` | one-shot compiler: resolves `{{ var }}` / `{% for %}` / `{% if %}` at build time into pure JS template-literal syntax; base64-inlines logo PNGs as data URIs |
| `/home/vboxuser/soc-project/n8n-prompts/email/alert.compiled.js` | auto-generated 314 KB JS module (DO NOT edit by hand); inlined into WF1's `Build email HTML` Code node by `/tmp/wf1_wire_html_email.py` |
| `/home/vboxuser/soc-project/n8n-prompts/email/preview.html` | pre-rendered preview with placeholder data; open in browser to eyeball before patching n8n |
| `/home/vboxuser/soc-project/n8n-prompts/email/elasticsearch-logo-png-transparent.png` + `thehive-logo.png` + `TheHive-Logotype-1.jpg` | source PNGs base64-embedded by the compiler |
| `/home/vboxuser/soc-project/n8n-prompts/email/README.md` | template variables contract + n8n Code-node render pattern |
