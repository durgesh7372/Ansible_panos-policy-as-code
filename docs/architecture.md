# Architecture & Design Notes

## Concept: policy as code

The firewall's intended security posture is expressed once, as data, in
`policy/baseline.yml`. Everything else is mechanism. This inverts the usual
firewall workflow — instead of clicking changes into a GUI and hoping they're
documented somewhere, the documentation *is* the source of truth and the
firewall is made to match it.

## Flow

```
   ┌─────────────────────┐
   │  policy/baseline.yml │   desired state, version-controlled
   └──────────┬──────────┘
              │ vars_files
              ▼
   ┌─────────────────────┐
   │  playbooks/*.yml     │   thin entrypoints (enforce / drift / teardown)
   └──────────┬──────────┘
              │ roles
              ▼
   ┌─────────────────────┐
   │  roles/panos_baseline│   idempotent tasks + commit handler
   └──────────┬──────────┘
              │ paloaltonetworks.panos modules (XML API)
              ▼
   ┌─────────────────────┐
   │  PAN-OS firewall /   │
   │  Panorama            │
   └─────────────────────┘
```

## Three entrypoints, one role

| Playbook | Mode | Effect |
|----------|------|--------|
| `enforce.yml` | normal | Converge device to baseline, commit if changed |
| `drift_report.yml` | `check_mode: true` | Read-only; reports drift, never writes |
| `teardown.yml` | normal | Remove everything the baseline defines |

All three consume the same role and the same data file. Behaviour differs only
by run mode — no duplicated logic.

## Why the ordering matters

Tasks run objects → security rules → NAT → commit. Rules reference address and
service objects by name, so the objects must exist first. The commit is a
*handler*, notified by change-producing tasks and run once at the end — so a
no-op run never triggers a needless commit job on the device.

## Idempotency

Every module call declares desired state (`state: present`). The PAN-OS modules
compare against the running config and only act on a delta. Practical proof:
run `enforce.yml` twice; the second run reports `changed=0`. This is the
property that makes the playbooks safe to run on a schedule.

## Standalone vs. Panorama

Set `panos_device_group` in `group_vars/all.yml`:

- **Empty** → targets a standalone NGFW's vsys directly.
- **Set** → targets that device group on a Panorama. (A production Panorama flow
  would add a `panos_commit_push` to push the committed config out to managed
  firewalls; left as a documented extension point in the commit handler.)

## Security considerations

- Real credentials live only in `group_vars/all.yml`, which is git-ignored.
  Use `ansible-vault encrypt` for anything beyond a throwaway lab.
- Run `drift_report.yml` (or `enforce --check --diff`) before any production
  enforce so you see the delta before it's applied.
- The catch-all `deny-and-log-all` rule in the sample baseline is intentional —
  it models an explicit-deny posture. Adjust zones/positioning for your design.

## Possible extensions

- Export drift output to JSON/HTML for a dashboard (ties into prior reporting work).
- Add `molecule` scenarios for automated role testing against a containerized mock.
- Pull the baseline from a CMDB/IPAM source instead of a static file.
