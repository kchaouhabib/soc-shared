# B1 Cortex Responders — Phase 20 (5 new response-action buttons)

> **Read this first.** This is a self-contained brief for the **B1 Claude Code instance** (192.168.1.51) to install 5 new Cortex Responders that ride on top of the same pattern as the 3 Phase-19 responders. Same pipeline, same gotchas — see notes at the end.

---

## What's being added

5 new responders, all `dataType: thehive:case`, each POSTing to A1's new n8n WF12 webhook `http://192.168.1.50:5678/webhook/response-action`:

| Responder | Purpose | `action` field in payload |
|---|---|---|
| `SOC_Autoblock_Isolate` | L1 confirms iptables DROP of source IP | `ssh_autoblock` |
| `SOC_Kill_Process` | Kill malicious process on host | `kill_process` |
| `SOC_Disable_User` | `usermod -L` a compromised account | `disable_user` |
| `SOC_Quarantine_File` | Move file to `/var/quarantine/` + chmod 000 | `quarantine_file` |
| `SOC_Delete_Registry` | Windows registry delete (stub) | `delete_registry` |

POST body shape (identical for all 5):
```json
{ "case_id": "~9961488",
  "action": "ssh_autoblock",
  "confirm": true,
  "l1_user": "socadmin@thehive.local",
  "source": "cortex_responder" }
```

WF12 (newly built on A1) routes this to `response_runner.py`, which dispatches to the matching shell script under `/home/vboxuser/soc-project/response-scripts/`, then posts a comment + tag back to the TheHive case.

---

## Step 1 — Pull the 5 source files from A1

```bash
mkdir -p /tmp/soc-p20-responders
for f in SOC_Autoblock_Isolate SOC_Kill_Process SOC_Disable_User SOC_Quarantine_File SOC_Delete_Registry; do
  scp vboxuser@192.168.1.50:/home/vboxuser/soc-project/cortex-responders/SOC/${f}.py   /tmp/soc-p20-responders/
  scp vboxuser@192.168.1.50:/home/vboxuser/soc-project/cortex-responders/SOC/${f}.json /tmp/soc-p20-responders/
done
ls -la /tmp/soc-p20-responders/   # should show 10 files (5 py + 5 json)
```

The `.py` files are ~1.5 KB each, `.json` ~610 B — same shape as the Phase-19 ones already installed.

---

## Step 2 — Push into the running cortex container

> **DEVIATION FROM PRIOR HANDOFF** *(from Phase 19 install report)*: the analyzers/responders tree is baked into the Cortex image (only `./cortex/data` is bind-mounted from the host). `docker cp` survives `start/stop/restart` but **does not survive `docker rm`/recreate**. For permanence, either add a bind mount or keep the files in the build context and re-`docker cp` after each rebuild.

```bash
for f in /tmp/soc-p20-responders/*; do
  docker cp "$f" cortex:/opt/Cortex-Analyzers/responders/SOC/
done
docker exec cortex chmod +x /opt/Cortex-Analyzers/responders/SOC/SOC_*.py
```

---

## Step 3 — Ensure deps + restart cortex

`cortexutils` and `requests` were `pip install`ed during Phase 19. Double-check:

```bash
docker exec cortex python3 -c "import cortexutils, requests; print('ok')"
# If it fails:
# docker exec cortex pip install cortexutils requests
```

Restart cortex so it loads the new responder defs:

```bash
docker restart cortex
until curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:9001/api/status | grep -q 200; do sleep 3; done
```

---

## Step 4 — Verify all 8 responders are registered

```bash
CORTEX_KEY=$(grep '^CORTEX_USER_API_KEY' /opt/cortex/.env.local | cut -d= -f2)
curl -s -H "Authorization: Bearer $CORTEX_KEY" \
  "http://127.0.0.1:9001/api/responderdefinition?range=all" | \
  python3 -c "import sys,json; d=json.load(sys.stdin); soc=[r for r in d if r.get('name','').startswith('SOC_')]; print(f'{len(soc)} SOC responders:'); [print(' -', r['name'], r.get('version','')) for r in soc]"
```

Expected: **8 SOC responders** (3 from Phase 19 + 5 new).

---

## Step 5 — Enable the 5 new responders in the SOC-LAB org

> **DEVIATION FROM PRIOR HANDOFF**: the enable POST body must contain **ONLY** the `n8n_webhook_url` config item. Including the generic `auto_extract_artifacts` / `check_tlp` / etc. keys triggers HTTP 500 `UnknownAttributeError` (these responders' JSON declares only that one item).

```bash
ORG_API_BASE="http://127.0.0.1:9001/api/organization/responder"

for R in SOC_Autoblock_Isolate SOC_Kill_Process SOC_Disable_User SOC_Quarantine_File SOC_Delete_Registry; do
  echo "Enabling $R..."
  curl -s -X POST -H "Authorization: Bearer $CORTEX_KEY" \
       -H "Content-Type: application/json" \
       "${ORG_API_BASE}/${R}_1_0" \
       -d '{"name":"'"$R"'","configuration":{"n8n_webhook_url":"http://192.168.1.50:5678/webhook/response-action"}}' \
       -w '  HTTP %{http_code}\n'
done
```

Expected: **5× HTTP 201**.

---

## Step 6 — Smoke test from TheHive UI

Open any existing case in TheHive, click **Responders** tab. You should see all 8 SOC responders listed. Click one (recommendation: `SOC_Quarantine_File` first — lowest-impact, will dry-run if no matching observable on the case).

What should happen:
1. Cortex runs the responder script → POSTs to A1 WF12 webhook.
2. WF12 fetches the case, runs `response_runner.py` with `--confirm` (the responder always sends `confirm:true`).
3. WF12 posts a comment on the case: *"## SOC Response Action — quarantine_file / Status: ok|failed|dry_run / ..."*.
4. WF12 patches the case with tag `response:quarantine_file:<status>` plus `response-by:<short-user>`.

**Before clicking, sanity-check A1 reachability:**
```bash
curl -o /dev/null -w '%{http_code}\n' http://192.168.1.50:5678/healthz   # expect 200
```

If A1 n8n is down, the responder run will time out at the POST. Wait for A1 to recover.

---

## Phase 19 lessons (recap so we don't hit them again)

1. **`docker cp` is not durable across `docker rm`/recreate.** Either bind-mount the analyzers tree long-term or remember to re-`cp` after any `compose down`/`up`. Still pending a permanent fix.
2. **Enable POST body must contain ONLY the responder's declared `configurationItems` keys.** Generic keys like `auto_extract_artifacts`/`check_tlp` cause HTTP 500.
3. **`.env.local` key extraction:** lines with inline `#` comments need `awk '{print $1}'` after `cut`, otherwise you grab `key + comment` as a single bad 70-char token.

---

## Report back

After step 6 lands green, append one line to `/home/vboxuser/soc-shared/CLAUDE.md` under `## Last session notes`:

> `2026-05-27  B1 Claude:  Phase 20 — 5 new responders installed + enabled in SOC-LAB. Smoke-tested SOC_Quarantine_File against case #N. WF12 posted result comment + tag.`

Then commit + push:

```bash
cd ~/soc-shared
git add CLAUDE.md
git commit -m 'p20-b1: 5 response responders enabled'
git push
```

A1 picks up from there for end-to-end attack-replay testing.
