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
| [windows](roles/windows/README.md) | Virtru File Services (file-watcher) | Windows 11 22H2+ | Available |
| sharepoint | SharePoint PEP | Windows Server | Planned |
| outlook | Outlook PEP | Exchange / Outlook | Planned |

Additional PEPs are added as new roles under `roles/<pep_name>`.

## Installation

```bash
ansible-galaxy collection install virtru.dsp_pep
```

## Usage

Install the Virtru Desktop PEP on a Windows endpoint:

```yaml
- name: Install Virtru Desktop PEP
  hosts: windows_endpoints
  roles:
    - role: virtru.dsp_pep.desktop
      vars:
        desktop_platform_endpoint: "https://platform.dsp.example.com:8080"
        desktop_public_client_id: dsp-desktop
        desktop_installer_src: "C:\\Installers\\VirtruDesktop-amd64-installer.exe"
```

See [`roles/desktop/README.md`](roles/desktop/README.md) for all variables and a
macOS example.

## License

GNU General Public License v3.0 or later. See [COPYING](COPYING).
