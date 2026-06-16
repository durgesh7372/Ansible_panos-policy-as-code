# PAN-OS Policy-as-Code

> Declare your Palo Alto firewall security baseline as version-controlled data.
> Enforce it idempotently with Ansible. Detect drift before an auditor does.

[![Lint](https://github.com/USERNAME/panos-policy-as-code/actions/workflows/lint.yml/badge.svg)](https://github.com/USERNAME/panos-policy-as-code/actions/workflows/lint.yml)
![Ansible](https://img.shields.io/badge/ansible-2.15%2B-blue)
![PAN--OS](https://img.shields.io/badge/PAN--OS-10.x%20%7C%2011.x-orange)

---

## The problem this solves

In a multi-site firewall estate, security policy *drifts*. Someone makes an
emergency change on one firewall during an incident and never replicates it.
Rule bases diverge. Shadowed and overly permissive rules accumulate. By audit
time, no one can say with confidence what the *intended* policy actually is.

This project treats the firewall security baseline as **code**: the desired set
of address objects, service objects, and security rules lives in a single
versioned YAML file. Ansible enforces that desired state idempotently, and a
`--check` run reports any drift without touching the device.

```
        ┌──────────────────────┐
        │  policy/baseline.yml │   ← single source of truth (Git)
        └──────────┬───────────┘
                   │
          ansible-playbook
                   │
        ┌──────────▼───────────┐        ┌─────────────────┐
        │  enforce.yml (role)  │───API──▶│  PAN-OS / Pano  │
        └──────────┬───────────┘        └─────────────────┘
                   │
          --check  │  (read-only)
                   ▼
            drift report
```

See [`docs/architecture.md`](docs/architecture.md) for the full diagram and design notes.

## What it does

- **Enforce**: converge the firewall to the baseline (objects → rules → commit)
- **Detect drift**: `--check --diff` shows what *would* change, read-only
- **Tear down**: cleanly remove everything the baseline defines
- **Idempotent**: a second enforce run reports zero changes

## Quick start

```bash
# 1. Install dependencies
python3 -m venv .venv && source .venv/bin/activate
pip install ansible pan-os-python xmltodict
ansible-galaxy collection install paloaltonetworks.panos

# 2. Point it at your firewall (use a VM-Series eval VM for a free lab)
cp group_vars/all.example.yml group_vars/all.yml
$EDITOR group_vars/all.yml      # set IP / credentials

# 3. Dry run — see what WOULD happen, change nothing
ansible-playbook playbooks/enforce.yml --check --diff

# 4. Enforce for real
ansible-playbook playbooks/enforce.yml

# 5. Prove idempotency — run again, expect changed=0
ansible-playbook playbooks/enforce.yml
```

## The baseline as data

The entire policy lives in [`policy/baseline.yml`](policy/baseline.yml). To change
firewall policy you edit data and open a pull request — never touch playbook logic:

```yaml
security_rules:
  - name: allow-corp-to-web
    source_zone: [trust]
    destination_zone: [dmz]
    source_ip: [corp-subnet]
    destination_ip: [web-server]
    application: [ssl, web-browsing]
    service: [application-default]
    action: allow
    log_end: true
```

## Repo layout

| Path | Purpose |
|------|---------|
| `policy/baseline.yml` | Single source of truth for the security baseline |
| `roles/panos_baseline/` | Reusable role: objects, rules, NAT, commit |
| `playbooks/enforce.yml` | Apply the baseline |
| `playbooks/drift_report.yml` | Read-only drift check wrapper |
| `playbooks/teardown.yml` | Remove everything the baseline defines |
| `docs/architecture.md` | Design notes + diagram |
| `.github/workflows/lint.yml` | CI: ansible-lint + yamllint on every push |

## Safety notes

- Credentials go in `group_vars/all.yml`, which is **git-ignored**. Encrypt with
  `ansible-vault` for anything beyond a throwaway lab.
- Always run `--check --diff` against production before a real enforce.
- This is a portfolio/lab project; review and adapt before any production use.

## Why I built it

I run multi-vendor firewall operations day to day and wanted policy management
that's reviewable, repeatable, and drift-aware rather than click-driven. This is
the pattern I'd reach for to keep a distributed Palo Alto estate consistent.

---

*Built with the official [`paloaltonetworks.panos`](https://galaxy.ansible.com/paloaltonetworks/panos) Ansible collection.*
