# Interaction diary

> Personal log of Claude Code sessions — focused on the interaction process, not just results.
> Goal: identify patterns, improve future prompting, reduce back-and-forth.

---

## 2026-05-04 — LinkedIn post series creation with /linkedin-post-creator skill

*Source: Claude Code CLI session — continuation of a compacted previous conversation.*

### Overview

Session focused on building a LinkedIn post series documenting the homelab project. Two parallel tracks: (1) iterating on the `/linkedin-post-creator` custom skill to match the desired writing style, and (2) generating posts from three source files — `stories_intro.md`, `stories_first_day.md`, `stories_feedbacks.md`. Each post was generated in two meaningfully different versions (A/B).

---

### Skill iteration workflow

**What I did:** invoked `/linkedin-post-creator` multiple times, compared generated output to my own rewrites, extracted style rules, and updated the skill file between runs.

**What worked:** the compare-and-extract loop was effective. Each rewrite surfaced concrete rules (opener format, emoji+title blocks, bullet list preservation, no hashtags, anti-buzzwords pass) that were directly added to the skill.

**Pattern to remember:** when tuning a custom skill, writing your own version of the output and diffing it against the generated one is more productive than trying to describe style rules upfront in the abstract.

---

### File persistence issue (Write tool)

**What went wrong:** the Write tool reported success for `2026-05-04_premier-deploiement_v1.md` and `v2.md`, but the files were not actually on disk. Discovered via `ls` when the user re-invoked the skill for the same source file.

**What I could have done better:** run a quick `ls stories/` after each batch of writes to verify files actually landed — especially after a session compaction, where the working state may differ from what was reported.

**Pattern to remember:** always verify file creation with `ls` after writing multiple files in one response. Tool call success ≠ file on disk.

---

### Source → post translation

**What I asked (effectively):** generate LinkedIn posts from raw notes that were written stream-of-consciousness, with typos, structural gaps, and objective notes mixed in.

**What Claude did:** cleaned the content, applied the skill's format rules, and generated two structurally distinct versions per source file.

**What worked well:** the skill's anti-buzzwords pass and version differentiation (chronological vs. thematic angle) were effective without extra prompting.

**What I could improve:** the raw notes could benefit from a short "angle" hint — e.g., "this post should emphasize the limitation of the AI, not the success." Without it, Claude picks an angle that may not match what I had in mind.

---

### Claude.ai vs ClaudeCode distinction in posts

The narrative clarified an important point: Claude.ai (mobile app) was used for the Proxmox OS installation (photo-based, step-by-step), while ClaudeCode (CLI) was used for everything infrastructure-related after that. This distinction matters for the blog audience and needs to stay explicit in the posts.

---

### General patterns observed today

| Pattern | Recommendation |
|---|---|
| Skill tuning by example | Write your own version of the output, then extract delta as rules to add to the skill |
| Write tool false positives | Run `ls` after batch file writes to confirm persistence |
| Raw notes as post source | Add a short "angle" note to raw source files to guide generation direction |
| Skill invoked without args | `/linkedin-post-creator` without an argument stops to ask — always pass the source file as an arg |

---

## 2026-05-03 — Full homelab day (Claude Code CLI)

*Source: this session only — Claude.ai sessions are not accessible from Claude Code.*

### Overview

Long session covering: Terraform API auth, Proxmox LXC provisioning, AdGuard Home deployment, local DNS setup on thallium, Let's Encrypt cert for proxmox.flefevre.fr, and finally AdGuard HTTPS with acme.sh.

---

### Terraform API token format

**What I asked:** switch Terraform auth from username/password to API token.

**What Claude did:** updated `terraform.tfvars` variables and the provider block.

**What went wrong:** the bpg/proxmox provider expects a single `api_token` string in the format `USER@REALM!TOKENID=UUID`, but I had two separate variables. Claude had to concatenate them as `"${var.proxmox_token_id}=${var.proxmox_api_token}"`. This wasn't obvious from the provider docs.

**What I could have done better:** paste the provider docs or the exact error message upfront instead of letting Claude discover the format from the error. Errors are much more useful than vague "it doesn't work".

---

### Proxmox SDN permission

**What I asked:** nothing specific — it came up as a `terraform apply` error.

**What Claude did:** identified the missing `SDN.Use` permission on the `TerraformProv` role and explained why (`vmbr0` is managed via Proxmox SDN).

**What I could have done better:** nothing — this was a runtime discovery. The error message was clear enough for Claude to diagnose it directly.

---

### AdGuard DNS rewrites not working (schema_version 34)

**What I asked:** `dig proxmox.lan` returns NXDOMAIN despite rewrites being in the config.

**What went wrong:** multiple fix attempts failed before finding the root cause. The template used `dns.rewrites` (old schema) instead of `filtering.rewrites` (schema_version 34), and each entry requires `enabled: true`. Claude fixed the structure twice before getting it right — first without `enabled: true`, then with.

**What I could have done better:** paste the full `AdGuardHome.yaml` content from the running container immediately. Claude was editing the Ansible template blind, without knowing what AdGuard actually generated at runtime. If I had shared the live config sooner, the fix would have been one step instead of three.

**Pattern to remember:** when something "should work" but doesn't, always share the actual running state (config files, logs) rather than the source templates — those can diverge.

---

### Local DNS setup on thallium (dhcpcd + systemd-resolved)

**What I asked:** make thallium use AdGuard as its DNS resolver.

**What went wrong:** multiple failed attempts and restarts before it worked. Key issues:
1. I didn't mention upfront that I use `dhcpcd + iwd` (not NetworkManager) — Claude started with generic advice
2. `wlan0` was injecting DNS via IPv6 Router Advertisements, overriding `resolv.conf`
3. The `nohook resolv.conf` line in dhcpcd.conf was silently blocking changes (added months ago, forgotten)
4. `Domains=~lan` in resolved.conf only routed `.lan`, not `.flefevre.fr`

**What I could have done better:** share `cat /etc/resolv.conf`, `systemctl status dhcpcd`, and `ip route` output immediately. DNS debugging is almost impossible without knowing the actual stack. One diagnostic dump would have saved 6+ back-and-forth exchanges.

**Pattern to remember:** for networking tasks, dump the full state first: `ip a`, `ip route`, `cat /etc/resolv.conf`, `systemctl status <service>`.

---

### Let's Encrypt cert for proxmox.flefevre.fr (Proxmox ACME)

**What I asked:** set up a cert for proxmox.flefevre.fr so the Proxmox UI is accessible via HTTPS without a certificate warning.

**What went wrong:** I mentioned Scaleway as my DNS provider — but `flefevre.fr` is actually hosted on Online.net (ns0/ns1.online.net). Claude initially pointed me toward the Scaleway DNS plugin. I had to correct this mid-session.

**What I could have done better:** mention the DNS registrar/nameserver upfront when asking about DNS-01 challenges. "My domain is hosted at X" is critical context for ACME DNS plugin selection.

**What went well:** once the right plugin was identified, Claude's step-by-step guidance through the Proxmox ACME UI (staging → production) was smooth.

---

### AdGuard HTTPS with acme.sh

**What I asked:** set up HTTPS for AdGuard Home using Let's Encrypt, with cert auto-renewal.

**What went wrong (multiple layers):**

1. **acme.sh install syntax** — Claude's first attempt used `curl ... | sh --install`, which creates a `----install` argument. The correct form is `curl ... | HOME=/root sh`.
2. **HOME not set in Ansible** — `become: true` doesn't set `HOME=/root`, so acme.sh couldn't find its config. Had to add `environment: HOME: /root`.
3. **ECC cert path** — acme.sh stores ECC certs in `domain_ecc/`, not `domain/`. The `creates:` idempotency check pointed to the wrong path, so Ansible always re-ran the task and got rc=2 (already issued).
4. **Wrong TLS field names** — AdGuard's config uses `certificate_path`/`private_key_path` (file paths), not `certificate_chain`/`private_key` (inline PEM). Claude had the wrong field names initially.
5. **`--cert-file` vs `--fullchain-file`** — AdGuard needs the full chain; acme.sh's `--cert-file` only installs the leaf cert.
6. **Duplicate fields in live config** — after multiple sed/template attempts on the live container, the config ended up with duplicate TLS fields, the second set being empty and winning. Fixed with a Python one-liner.

**What I could have done better:** share `acme.sh --help` output or the acme.sh docs link for `--install-cert`. Also, after each failed attempt, paste the full error and the actual config state — I sometimes just said "ça marche pas" without the details.

**Pattern to remember:** for any tool you're not familiar with, paste its `--help` or man page excerpt when asking Claude to use it. acme.sh has unusual conventions (ECC suffix, `--fullchain-file` vs `--cert-file`) that aren't guessable.

---

### General patterns observed today

| Pattern | Recommendation |
|---|---|
| Diagnosis without current state | Always share current file content / command output before asking "why doesn't X work" |
| Infrastructure-specific details missing | Mention OS, network stack, DNS registrar upfront for infrastructure tasks |
| Vague "ça marche pas" | Paste the exact error + the actual running config, not the source template |
| Multi-step fixes that diverge | After a failed fix, re-read the live state before the next attempt — don't chain fixes blindly |
| Tool-specific conventions (acme.sh, AdGuard schema) | Share docs/help output for niche tools Claude may not know deeply |
