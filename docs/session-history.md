# Session history archive

Older entries from `## Last session notes` in CLAUDE.md, archived to keep the live notes section to ~5 entries. Newest archived entry on top.

---

## 2026-04-29 — victim-lab (VM_B2) — Phase 0 + Phase 1 (with VM_B1 collateral)

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

---

## 2026-04-29 — soc-core (VM_A1) — Phase 1 progress

Done since last entry:
- ZeroTier network ID changed from 743993800ffa3724 to cf719fd54008e4d1 (user re-created the network); updated in CLAUDE.md and PROJECT-MASTER-PLAN.md
- VM_A1 joined cf719fd54008e4d1 (network name "my-first-network")
- User authorized the node in admin console; status OK, IP 192.168.1.50/24 live on interface ztdiyzommr
- VM_A1 ZeroTier node ID: 785fd1806c (reference for re-authorizing if needed)

Notes:
- Sudo password is in local memory; user must restate it per session for the harness to accept it (auto-pipe was blocked)
- Plugin marketplace install (everything-claude-code) was started in background but is still cloning at last check — separate side task, not part of phase 1

---

## 2026-04-29 — incident-mgmt (VM_B1) — Phase 0 clone

Done:
- Installed git via apt (was missing on this VM)
- Cloned soc-shared to ~/soc-shared/ (initially landed in ~/pfe/, then moved to canonical ~/ location)
- Created empty ~/soc-project/ (local-only working folder)
- Verified `cd ~/soc-shared && git pull` returns "Already up to date."
- Confirmed VM identity: hostname=incident-mgmt, ZeroTier IP=192.168.1.51 (interface ztdiyzommr)

Notes:
- On VM_B1 the user prefers the canonical ~/soc-shared/ path over working out of ~/pfe/. Resolved the open question from the previous note.

---

## 2026-04-29 — soc-core (VM_A1) — Phase 0 bootstrap

Done:
- Created GitHub repo https://github.com/kchaouhabib/soc-shared (public)
- Bootstrapped ~/soc-shared/ with CLAUDE.md, PROJECT-MASTER-PLAN.md, README.md, docs/.gitkeep
- Initial commit 7429194 pushed to origin/main
- Created empty ~/soc-project/ (local-only working folder, not on Git)
- Git identity already configured globally (Habib Kchaou / kchaou.habib67@gmail.com)
