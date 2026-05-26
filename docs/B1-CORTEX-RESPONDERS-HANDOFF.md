# Phase 19.2 Handoff — B1 Cortex Responders install

**For the B1 Claude (incident-mgmt @ 192.168.1.51).** Next concrete task: install 3 Cortex Responders so L1 analysts can click TP / FP / Escalate-to-L2 buttons inside TheHive case UI.

## Context

The A1 instance built Phase 19 on 2026-05-26:

- **WF10** — deployed on A1's n8n. `GET /webhook/l2-verdict?case_id=X&verdict=tp|fp&l2_user=email` — patches case (tag `l2-verdict:tp/fp` + summary + impactStatus) + `ml-api /feedback` + amber/navy HTML confirmation page. Live.
- **WF11** — deployed. `POST /webhook/l1-decision` body `{case_id, decision, l1_user}`. Three branches:
  - `tp` / `fp` → PATCH case (tags `l1-verdict:tp/fp`, `l1-decision-by:<user>`, summary, impactStatus) + `ml-api /feedback`. **Does NOT close case** — TheHive 5's mandatory-tasks gate blocks status transitions; L1 closes manually after reviewing. Defensible architectural choice for the rapport.
  - `escalate_l2` → fetch case → Ollama 4-section JSON (initial_description / process_analysis / threat_intel / network_logs / actions_taken / next_steps) → renders `l2_alert.compiled.js` (29 KB HTML, no inline logos so Gmail doesn't clip at 102 KB) → SMTPs to `tayechi.mrayen@gmail.com,kchaou.habib67@gmail.com` via Mailtrap Live → tags case `l1-decision:escalate-l2` + `l2-email-sent` + posts an audit comment.
- **L2 email** has 2 buttons (Confirm TP / Confirm FP) that link to WF10 `/webhook/l2-verdict?case_id=X&verdict=tp|fp&l2_user=...`. Single-click verdict for L2.

### User architecture (TheHive Free, 2 normal-user quota)

- `soc-bot@thehive.local` — automation API user + doubles as L1 analyst (display name "L1 SOC Analyst (also automation)").
- `socadmin@thehive.local` — was org-admin, **demoted to analyst** so we have 2 analysts. Acts as L2. Display name still "SOC-LAB Org Admin" (rename was license-blocked after demote). UI password unchanged; if L2 needs to log in, reset via `admin@thehive.local` (platform superadmin).
- `soc_analyst@thehive.local` — read-only (3rd account, not counted against the 2-analyst quota).
- The 3 Cortex Responders work regardless of which user clicks — the responder posts to n8n with whatever TheHive user clicked it.

### Important: TheHive↔Cortex connector IS live

Verified by A1: `curl http://192.168.1.51:9000/api/v1/connector/cortex/analyzer?range=0-1` returns MISP_2_1 with `cortexIds:["SOC-LAB-Cortex"]`. So Cortex Responders **do** appear as buttons in TheHive case UI (Scenario A). Earlier session notes from 2026-05-15 said Platinum quota was 0; whatever the user did since, the connector is functional now. Do not re-diagnose F1/F2.

---

## Your task

Drop the 3 responder scripts into the vendored `Cortex-Analyzers/responders/SOC/` tree, install Python deps inside the cortex container, restart cortex, verify registration, enable in SOC-LAB org, smoke-test.

### Step 1 — create the 7 files

```bash
mkdir -p ~/soc-project/thehive-cortex/cortex/Cortex-Analyzers/responders/SOC
cd ~/soc-project/thehive-cortex/cortex/Cortex-Analyzers/responders/SOC

cat > requirements.txt << 'EOF'
cortexutils
requests
EOF

cat > SOC_Confirm_TP.py << 'PYEOF'
#!/usr/bin/env python3
"""SOC_Confirm_TP — L1 verdict: confirm True Positive on a TheHive case via WF11."""
import requests
from cortexutils.responder import Responder

class SOCConfirmTP(Responder):
    def __init__(self):
        Responder.__init__(self)

    def run(self):
        Responder.run(self)
        data = self.get_data() or {}
        obj = data.get("object", {}) if isinstance(data, dict) else {}
        case_id = obj.get("_id") or obj.get("id") or ""
        if not case_id:
            self.error("No case _id in input data.")
            return
        n8n_url = self.get_param("config.n8n_webhook_url",
                                 "http://192.168.1.50:5678/webhook/l1-decision")
        l1_user = data.get("user") or obj.get("_updatedBy") or obj.get("_createdBy") or "soc-bot@thehive.local"
        body = {"case_id": case_id, "decision": "tp", "l1_user": l1_user, "source": "cortex_responder"}
        try:
            r = requests.post(n8n_url, json=body, timeout=20)
            r.raise_for_status()
            self.report({"message": "Case tagged TP via WF11", "http_status": r.status_code,
                         "case_id": case_id, "l1_user": l1_user})
        except Exception as e:
            self.error(f"WF11 webhook POST failed: {e}")

if __name__ == "__main__":
    SOCConfirmTP().run()
PYEOF

cat > SOC_Confirm_TP.json << 'JSONEOF'
{
  "name": "SOC_Confirm_TP",
  "version": "1.0",
  "author": "SOC-LAB (PFE)",
  "url": "https://soc-lab.local",
  "license": "AGPL-V3",
  "description": "L1 verdict: confirm True Positive — tags case via WF11 and feeds the ML model.",
  "dataTypeList": ["thehive:case"],
  "command": "SOC/SOC_Confirm_TP.py",
  "baseConfig": "SOC",
  "configurationItems": [
    {
      "name": "n8n_webhook_url",
      "description": "n8n WF11 L1 decision webhook URL",
      "type": "string",
      "multi": false,
      "required": true,
      "defaultValue": "http://192.168.1.50:5678/webhook/l1-decision"
    }
  ]
}
JSONEOF

cat > SOC_Confirm_FP.py << 'PYEOF'
#!/usr/bin/env python3
"""SOC_Confirm_FP — L1 verdict: confirm False Positive on a TheHive case via WF11."""
import requests
from cortexutils.responder import Responder

class SOCConfirmFP(Responder):
    def __init__(self):
        Responder.__init__(self)

    def run(self):
        Responder.run(self)
        data = self.get_data() or {}
        obj = data.get("object", {}) if isinstance(data, dict) else {}
        case_id = obj.get("_id") or obj.get("id") or ""
        if not case_id:
            self.error("No case _id in input data.")
            return
        n8n_url = self.get_param("config.n8n_webhook_url",
                                 "http://192.168.1.50:5678/webhook/l1-decision")
        l1_user = data.get("user") or obj.get("_updatedBy") or obj.get("_createdBy") or "soc-bot@thehive.local"
        body = {"case_id": case_id, "decision": "fp", "l1_user": l1_user, "source": "cortex_responder"}
        try:
            r = requests.post(n8n_url, json=body, timeout=20)
            r.raise_for_status()
            self.report({"message": "Case tagged FP via WF11", "http_status": r.status_code,
                         "case_id": case_id, "l1_user": l1_user})
        except Exception as e:
            self.error(f"WF11 webhook POST failed: {e}")

if __name__ == "__main__":
    SOCConfirmFP().run()
PYEOF

cat > SOC_Confirm_FP.json << 'JSONEOF'
{
  "name": "SOC_Confirm_FP",
  "version": "1.0",
  "author": "SOC-LAB (PFE)",
  "url": "https://soc-lab.local",
  "license": "AGPL-V3",
  "description": "L1 verdict: confirm False Positive — tags case via WF11 and feeds the ML model.",
  "dataTypeList": ["thehive:case"],
  "command": "SOC/SOC_Confirm_FP.py",
  "baseConfig": "SOC",
  "configurationItems": [
    {
      "name": "n8n_webhook_url",
      "description": "n8n WF11 L1 decision webhook URL",
      "type": "string",
      "multi": false,
      "required": true,
      "defaultValue": "http://192.168.1.50:5678/webhook/l1-decision"
    }
  ]
}
JSONEOF

cat > SOC_Escalate_L2.py << 'PYEOF'
#!/usr/bin/env python3
"""SOC_Escalate_L2 — L1 escalates a TheHive case to L2 via WF11."""
import requests
from cortexutils.responder import Responder

class SOCEscalateL2(Responder):
    def __init__(self):
        Responder.__init__(self)

    def run(self):
        Responder.run(self)
        data = self.get_data() or {}
        obj = data.get("object", {}) if isinstance(data, dict) else {}
        case_id = obj.get("_id") or obj.get("id") or ""
        if not case_id:
            self.error("No case _id in input data.")
            return
        n8n_url = self.get_param("config.n8n_webhook_url",
                                 "http://192.168.1.50:5678/webhook/l1-decision")
        l1_user = data.get("user") or obj.get("_updatedBy") or obj.get("_createdBy") or "soc-bot@thehive.local"
        body = {"case_id": case_id, "decision": "escalate_l2", "l1_user": l1_user, "source": "cortex_responder"}
        try:
            # Slightly longer timeout — WF11 fires Ollama which can take minutes (n8n returns the HTTP
            # response page after the SMTP send finishes; this can be ~60-120s when Ollama is warm).
            r = requests.post(n8n_url, json=body, timeout=30)
            r.raise_for_status()
            self.report({"message": "L2 escalation fired — L2 email queued via WF11",
                         "http_status": r.status_code, "case_id": case_id, "l1_user": l1_user})
        except Exception as e:
            self.error(f"WF11 webhook POST failed: {e}")

if __name__ == "__main__":
    SOCEscalateL2().run()
PYEOF

cat > SOC_Escalate_L2.json << 'JSONEOF'
{
  "name": "SOC_Escalate_L2",
  "version": "1.0",
  "author": "SOC-LAB (PFE)",
  "url": "https://soc-lab.local",
  "license": "AGPL-V3",
  "description": "L1 escalates the case to L2 — fires WF11 which sends the L2 brief email with TP/FP buttons.",
  "dataTypeList": ["thehive:case"],
  "command": "SOC/SOC_Escalate_L2.py",
  "baseConfig": "SOC",
  "configurationItems": [
    {
      "name": "n8n_webhook_url",
      "description": "n8n WF11 L1 decision webhook URL",
      "type": "string",
      "multi": false,
      "required": true,
      "defaultValue": "http://192.168.1.50:5678/webhook/l1-decision"
    }
  ]
}
JSONEOF

chmod +x SOC_*.py
ls -la
```

Verify all 7 files parse:
```bash
for f in *.json; do python3 -c "import json; json.load(open('$f'))" && echo "OK $f"; done
for f in *.py; do python3 -m py_compile "$f" && echo "OK $f"; done
```

### Step 2 — install Python deps inside the Cortex container

```bash
docker ps --format '{{.Names}}' | grep -i cortex   # verify container name; adjust below if different
docker exec cortex pip install --quiet cortexutils requests
docker exec cortex python -c "import cortexutils, requests; print('deps OK')"
```

### Step 3 — restart Cortex so it scans the new responders

```bash
docker restart cortex
for i in $(seq 1 30); do
  code=$(curl -s -m 3 -o /dev/null -w "%{http_code}" http://192.168.1.51:9001/api/status)
  echo "attempt $i: HTTP $code"
  [ "$code" = "200" ] && break
  sleep 3
done
```

### Step 4 — verify the 3 responders are registered

```bash
B=$(grep '^CORTEX_USER_API_KEY=' ~/soc-project/.env.local | cut -d= -f2-)
curl -s -H "Authorization: Bearer $B" "http://192.168.1.51:9001/api/responderdefinition?range=all" \
  | python3 -c "
import sys, json
defs = json.load(sys.stdin)
soc = [d for d in defs if d.get('name','').startswith('SOC_')]
print(f'total responders: {len(defs)}, SOC_: {len(soc)}')
for d in soc:
    print(f'  - {d[\"name\"]} v{d.get(\"version\")} dataTypes={d.get(\"dataTypeList\")}')
"
```

Expected: 3 SOC_ responders listed.

### Step 5 — enable each responder in the SOC-LAB org

```bash
B=$(grep '^CORTEX_USER_API_KEY=' ~/soc-project/.env.local | cut -d= -f2-)
for name in SOC_Confirm_TP SOC_Confirm_FP SOC_Escalate_L2; do
  curl -s -H "Authorization: Bearer $B" -H "Content-Type: application/json" \
    -X POST "http://192.168.1.51:9001/api/organization/responder/${name}_1_0" \
    -d '{"name":"'"$name"'","configuration":{"auto_extract_artifacts":false,"check_tlp":true,"max_tlp":2,"check_pap":true,"max_pap":2,"n8n_webhook_url":"http://192.168.1.50:5678/webhook/l1-decision"}}' \
    -w "\n  $name -> HTTP %{http_code}\n" | tail -2
done
```

If `_1_0` suffix returns 404, check the `definitionId` from step 4's output (Cortex's actual id syntax) and use that.

Then list enabled instances:
```bash
B=$(grep '^CORTEX_USER_API_KEY=' ~/soc-project/.env.local | cut -d= -f2-)
curl -s -H "Authorization: Bearer $B" "http://192.168.1.51:9001/api/responder?range=all" \
  | python3 -c "
import sys, json
ts = json.load(sys.stdin)
soc = [t for t in ts if t.get('name','').startswith('SOC_')]
print(f'enabled SOC instances: {len(soc)}')
for t in soc: print(f'  - {t[\"name\"]} id={t[\"id\"]}')
"
```

### Step 6 — smoke-test one responder end-to-end

Find an open InProgress case:
```bash
THV=$(grep '^THEHIVE_BOT_API_KEY=' ~/soc-project/.env.local | cut -d= -f2-)
curl -s -H "Authorization: Bearer $THV" -H "Content-Type: application/json" \
  -X POST "http://192.168.1.51:9000/api/v1/query" \
  -d '{"query":[{"_name":"listCase"},{"_name":"filter","_and":[{"_field":"status","_value":"InProgress"}]},{"_name":"sort","_fields":[{"_createdAt":"desc"}]},{"_name":"page","from":0,"to":1}]}' \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print(d[0]['_id'] if d else 'no open case')"
```

In TheHive UI (`http://192.168.1.51:9000`) open that case → **Responders** tab → click **SOC_Confirm_TP**. Within seconds the case should gain tags `l1-verdict:tp` + `l1-decision-by:<your_short_user>` and a comment "## L1 Verdict" should appear.

## Report back via CLAUDE.md last-session-notes

When done, edit `~/soc-shared/CLAUDE.md` (append a new entry at the top of "Last session notes") with:

1. Output of step 4 (3 SOC_ responders listed?)
2. Output of step 5 (3 HTTP 2xx?)
3. Did the click test in step 6 add the `l1-verdict:tp` tag to the case?
4. Anything that broke + the fix.

Then:
```bash
cd ~/soc-shared
git add CLAUDE.md
git commit -m "phase-19.2: incident-mgmt: Cortex Responders installed (3 SOC_ buttons live)"
git push
```

A1 will pull and resume from there.
