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

### 6.4 Custom AI / Intelligence Services (built for this project)

These are the original contributions of the PFE — **two Flask microservices** that solve problems no built-in covers, plus n8n workflows that orchestrate stock components for everything else.

> **Re-architecture note (2026-05-05):** The original plan called for five Flask services. Two of them were dropped during Phase 3 because stock components already cover the same job. This made the stack smaller and more defendable. See *Dropped services* at the bottom of this section.

**Ollama (port 11434)** — local LLM runtime. Runs llama3.1:8b on VM_A1. Powers all AI features in the project without any API cost. Replaces what would otherwise be Anthropic/OpenAI calls. Slower and slightly less capable than Claude or GPT-4, but free and self-contained. Configured with `OLLAMA_KEEP_ALIVE=24h` to keep the model warm in RAM (cold load is ~5 min from disk on this VM).

**ML Anomaly Detection API (port 5000)** — Flask service running an Isolation Forest model. Every alert that enters the system gets an anomaly score (0–1). High score = unusual = likely a real threat. Low score = matches normal patterns = probably routine. Used by n8n to gate which alerts become full cases versus go to a daily digest. Bootstraps from a synthetic alert distribution on first start, retrains via `POST /train` on the last 30 days of `.alerts-security.alerts-default` once real data accumulates. Kept because no Basic-tier Elastic feature provides equivalent scoring (Anomaly Detection jobs require a Platinum subscription).

**Correlation Engine (port 5002)** — Flask service with SQLite state. Decides whether each incoming alert creates a new case or attaches to an existing one. 30-minute same-(source_ip, rule_id) bucket window for storm dedup; **2-hour cross-tactic kill-chain window** that escalates a case the moment a second MITRE tactic appears from the same source IP. Whitelist for known-good IPs; daily-digest queue for low-severity, low-anomaly alerts. Solves the "1 attack = 7 cases" problem and the alert-fatigue problem in one place. The biggest original contribution of the project — kill-chain detection across MITRE tactics is not built into Kibana alert suppression or TheHive 5 case grouping.

**Dropped services (replaced by built-ins or n8n+Ollama):**

- ~~**NLP Summarization API (port 5001)**~~ — replaced by an n8n HTTP-Request node calling Ollama directly with `format:"json"`. Prompt templates live in the n8n workflow JSON. No Flask wrapper buys us anything.
- ~~**MITRE Auto-Tagger (port 5003)**~~ — Kibana's detection-engine UI/API already takes MITRE tags via its `threat[]` field at rule definition time; Elastic prebuilt rules ship pre-tagged; Sigma carries `attack.t####` tags through `sigma2elastic`; community classtype-to-MITRE JSONs cover Suricata's gaps. Building a TF-IDF tagger duplicates these built-ins.
- ~~**AI Rule Generator (was part of NLP API)**~~ — re-implemented as an n8n workflow node: MISP webhook → HTTP-Request to Ollama with `format:"json"` returning `{kibana_rule, suricata_rule, tactic_id, technique_id}` in one round-trip → Kibana API push + Suricata SSH deploy.

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

**Outcome:** Defendable PFE with complete documentation and demonstrable system.

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
