# vuln-remediation-playbooks

Ansible playbooks used by AWX to remediate findings reported by OpenVAS/GVM against hosts in this lab. Every playbook here is designed to be launched **only** through an AWX Workflow Job Template that pauses at a human Approval node first — nothing in this repo is meant to run unattended.

## How this fits together

```
OpenVAS/GVM scan
      |
      v
gvm_awx_bridge.py  (classifies each finding, launches an AWX WORKFLOW — never a bare job)
      |
      v
AWX Workflow Job Template
  Start -> [Approval node, paused] -> [Job Template node -> runs a playbook from this repo]
      |
      v
Human clicks Approve (or Deny) in AWX
      |
      v
Playbook actually executes against the target host
      |
      v
Next scan + bridge script run confirms the finding is gone
```

No finding in this pipeline, regardless of severity or category, skips the Approval step. The bridge script's only AWX API calls are `workflow_job_templates/<id>/launch/` (creates a paused request) and read-only status checks — it has no code path that can approve anything itself.

## Playbooks

### `patch-security-updates.yml`

Applies pending OS security updates via `unattended-upgrades` (security pocket only). Optionally reboots afterward if a reboot is required and explicitly allowed.

Variables:
- `target_hosts` (default `all`) — set via AWX `limit` at launch time.
- `allow_reboot` (default `false`) — set to `true` via `extra_vars` to allow an automatic reboot when the OS reports one is required after patching.

Routed to from findings matching security-advisory identifiers (`Security Advisory`, `USN-`, `RHSA-`, `DSA-`).

### `remediate-tls-icmp-rfds.yml`

Combined playbook covering three unrelated finding categories, each independently gated so it only acts on a host where it's actually relevant:

- **ICMP Timestamp Reply Information Disclosure** (CVE-1999-0524) — drops incoming ICMP type-13 (timestamp) requests via `iptables`, persisted with `iptables-persistent`.
- **Deprecated TLS 1.0/1.1** — disables TLSv1.0/1.1 in Apache and/or nginx if either is present, restarting the relevant service. Leaves a warning if neither web server config is found so the playbook can be extended to whatever's actually serving that port.
- **Missing RFDS kernel mitigation** (CVE-2023-28746 / INTEL-SA-00898) — updates `intel-microcode` and reports current mitigation status; reboots only if microcode was actually updated and reboots are explicitly allowed.

Variables:
- `target_hosts` (default `all`)
- `allow_reboot` (default `false`) — required for the RFDS fix to actually take effect.

Routed to from findings matching `TLS`/`SSL`, `ICMP Timestamp`, or `kernel`/`mitigation`.

## Requirements

- Ansible with `become: true` (sudo) rights on target hosts.
- `community.general` collection for the `ansible.builtin.iptables` module used in the ICMP fix (usually bundled with the AWX execution environment already; confirm if a playbook errors on that module).
- Debian/Ubuntu targets — both playbooks gate on `ansible_os_family == "Debian"`. Extend with `RedHat`-family equivalents if you add non-Debian hosts.

## Adding a new remediation category

1. Write a new playbook here, following the existing pattern: gate every task with a `stat`/`when` check so it's a no-op on hosts where it doesn't apply, and never assume a reboot is wanted — always gate that behind an explicit `allow_reboot` variable.
2. Push it to this repo, then sync the AWX Project (Projects don't auto-pull on push — trigger a sync via the UI or `POST /api/v2/projects/<id>/update/`, and confirm `scm_revision` matches your latest commit before it'll show up in a Job Template's playbook dropdown).
3. Create a plain Job Template pointing at the new playbook, with **Prompt on Launch** enabled for both **Limit** and **Variables**.
4. Wrap it in a new Workflow Job Template: `Start -> Approval node -> Job Template node`. Enable **Prompt on Launch** for Limit/Variables on **both** the node and the Workflow Job Template itself — the workflow-level setting is what actually lets the launch API accept a `limit`/`extra_vars` payload; the node-level setting is what lets those values flow down into the actual playbook run.
5. Add a new entry to `JOB_TEMPLATE_MAP` in `gvm_awx_bridge.py` with a regex matching the finding name(s) this playbook addresses, pointing at the new workflow's ID.

## Safety notes

- Every workflow's Approval node should have a **Timeout** set so an un-acted-on request eventually fails closed rather than sitting open forever.
- The AWX `status` field on a `workflow_job` reads `"running"` for the entire time a workflow is in progress — **including while it's sitting at an unapproved Approval node.** Don't take that field alone as evidence a playbook has executed; check the actual `workflow_approvals` record (or the Workflow Visualizer, where an un-started Job Template node stays uncolored) for the real approval decision.
- Full setup and operational documentation, including the AWX build-out this repo depends on, lives in the companion guide (`awx-vuln-patching-guide.md`) tracked alongside this pipeline.
