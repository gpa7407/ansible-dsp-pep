# `virtru.dsp_pep.desktop`

Install and configure the **Virtru Desktop** PEP on Windows 11 (amd64) and macOS
(Apple Silicon).

What it does:

1. **Windows only:** installs the Edge WebView2 runtime prerequisite if missing.
2. Installs the Virtru Desktop application from an installer you provide
   (`.exe` on Windows via silent `/S`, `.pkg` on macOS via `installer`), idempotently.
3. Renders the system-wide `virtru_desktop.yaml` with your DSP endpoint and OIDC
   client (and common options).

## Requirements

- Target host is Windows 11 Pro/Enterprise or macOS 11+ (Apple Silicon).
- The `ansible.windows` collection (for Windows targets) and a working WinRM/PSRP
  or SSH connection.
- An installer for the target OS, reachable as a path on the host or a URL
  (`desktop_installer_src`).
- A reachable DSP platform endpoint and a public OIDC client whose redirect URIs
  allow `http://127.0.0.1:*`.

## Key variables

| Variable | Default | Description |
| -------- | ------- | ----------- |
| `desktop_platform_endpoint` | — (**required**) | DSP endpoint incl. port, e.g. `https://platform.dsp.vm:8080` |
| `desktop_public_client_id` | `dsp-desktop` | Public OIDC client id |
| `desktop_installer_src` | `""` | Path/URL to the `.exe` (Windows) or `.pkg` (macOS) |
| `desktop_install_webview2` | `true` | Install the Edge WebView2 runtime on Windows if missing |
| `desktop_manage_config` | `true` | Render `virtru_desktop.yaml` |
| `desktop_insecure_tls_no_verify` | `false` | Skip TLS verification (labs only) |

See `defaults/main.yml` / `meta/argument_specs.yml` for the full list (logging,
tagging, watermark, `desktop_extra_dsp_config`, `desktop_experimental_features`).

Config paths written:

- Windows: `C:\ProgramData\VirtruCorporation\Virtru Desktop\virtru_desktop.yaml`
- macOS: `/Library/Application Support/VirtruCorporation/Virtru Desktop/virtru_desktop.yaml`

## Examples

Windows:

```yaml
- hosts: windows_endpoints
  roles:
    - role: virtru.dsp_pep.desktop
      vars:
        desktop_platform_endpoint: "https://platform.dsp.example.com:8080"
        desktop_public_client_id: dsp-desktop
        desktop_installer_src: 'C:\Installers\VirtruDesktop-amd64-installer.exe'
```

macOS:

```yaml
- hosts: macs
  roles:
    - role: virtru.dsp_pep.desktop
      vars:
        desktop_platform_endpoint: "https://platform.dsp.example.com:8080"
        desktop_installer_src: "https://downloads.example.com/VirtruDesktop.pkg"
```

## Notes

- The Windows installed-marker path (`desktop_windows_installed_marker`) and the
  macOS app path (`desktop_macos_app_path`) are used for idempotency; override
  them if your installer places the app elsewhere.
- The installer never overwrites an existing `virtru_desktop.yaml`; this role
  manages that file directly (idempotent via `template`).
