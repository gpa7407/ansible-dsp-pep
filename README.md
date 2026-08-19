# Ansible Collection: virtru.dsp_pep

Install and configure Virtru Data Security Platform (DSP) **Policy Enforcement
Points (PEPs)**. Each PEP is a role under `virtru.dsp_pep`, so you invoke them as
`virtru.dsp_pep.<pep_name>`.

## Ansible version compatibility

Tested against Ansible **>=2.15**.

## Dependencies

- `ansible.windows` (used by PEP roles that install on Windows endpoints)

## Roles (PEPs)

| Role | PEP | Target OS | Status |
| ---- | --- | --------- | ------ |
| [desktop](roles/desktop/README.md) | Virtru Desktop | Windows 11, macOS (Apple Silicon) | Available |
| [windows](roles/windows/README.md) | Virtru File Services (file-watcher) | Windows 11 24H2 (x64) | Available |
| sharepoint | SharePoint PEP | Windows Server | Planned |
| outlook | Outlook PEP | Exchange / Outlook | Planned |

Additional PEPs are added as new roles under `roles/<pep_name>`.

## Installation

```bash
ansible-galaxy collection install virtru.dsp_pep
```

## Usage

Each PEP is a role; its variables, requirements, and examples are documented in
that role's README:

- **desktop** — [roles/desktop/README.md](roles/desktop/README.md)
- **windows** (Virtru File Services) — [roles/windows/README.md](roles/windows/README.md)

Ready-to-run example playbooks are in [`playbooks/`](playbooks/).

## License

GNU General Public License v3.0 or later. See [COPYING](COPYING).
