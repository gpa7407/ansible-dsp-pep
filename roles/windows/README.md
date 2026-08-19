# `virtru.dsp_pep.windows`

Install and configure **Virtru File Services** — the Windows file-watcher PEP
that provides OS-level (minifilter driver) TDF protection via the `virtruflt`,
`virtrusvc`, and `fe2policy` services. **Windows 11 22H2+ (amd64) only.**

What it does:

1. Installs the **Visual C++ 2022 x64 runtime** prerequisite (idempotent).
2. Imports the **driver-signing certificates** (`rootca.pem` → LocalMachine\Root,
   `testsign.pem` → LocalMachine\My).
3. Optionally enables **test-signing mode** (`bcdedit /set TESTSIGNING ON`) and
   reboots — required until the drivers are Microsoft cross-signed (needs Secure
   Boot disabled).
4. Installs the **`virtru_file_services-x64-rel.msi`** with its config properties
   (`CLIENT_ID`, `CLIENT_SECRET`, `AUDIENCE`, `PLATFORM_ENDPOINT`,
   `DEFAULT_ATTRIBUTES`) → written to `HKLM\Software\Virtru`. Reboots if required.
5. Optionally applies extra registry tuning keys.

## Requirements

- Windows 11 22H2+ (amd64); `ansible.windows` and a WinRM/PSRP connection.
- The MSI and cert files staged on the target (or the MSI reachable by URL).
- A confidential OIDC client with **OAuth 2.0 Device Authorization** enabled, and
  a public loopback redirect `http://127.0.0.1:*`.
- Administrator privileges; Secure Boot disabled if using test-signing.

## Key variables

| Variable | Default | Description |
| -------- | ------- | ----------- |
| `windows_platform_endpoint` | — (**required**) | DSP endpoint incl. port |
| `windows_client_id` / `windows_client_secret` | — (**required**) | PEP OIDC client creds |
| `windows_audience` | = platform endpoint | Token audience |
| `windows_default_attributes` | `""` | `;`-delimited FQNs for new TDFs |
| `windows_installer_src` | `""` | Path/URL to the MSI |
| `windows_install_vc_redist` | `true` | Install VC++ 2022 runtime |
| `windows_import_certs` | `true` | Import driver-signing certs |
| `windows_rootca_cert_src` / `windows_testsign_cert_src` | `""` | Cert paths on the target |
| `windows_enable_test_signing` | `false` | Enable test-signing + reboot |
| `windows_registry_settings` | `[]` | Extra `win_regedit` entries |

See `defaults/main.yml` / `meta/argument_specs.yml` for the full list.

## Example

```yaml
- hosts: windows_filewatcher
  roles:
    - role: virtru.dsp_pep.windows
      vars:
        windows_platform_endpoint: "https://platform.dsp.example.com:443"
        windows_client_id: dsp-windows-filewatcher
        windows_client_secret: "{{ vault_fw_client_secret }}"
        windows_default_attributes: "https://demo.com/attr/classification/value/secret"
        windows_installer_src: 'C:\Installers\virtru_file_services-x64-rel.msi'
        windows_rootca_cert_src: 'C:\Installers\rootca.pem'
        windows_testsign_cert_src: 'C:\Installers\testsign.pem'
        windows_enable_test_signing: true
        # optional tuning:
        windows_registry_settings:
          - path: 'HKLM:\SYSTEM\CurrentControlSet\Services\virtruflt\Parameters'
            name: EncryptedViewSearchDepth
            data: 5
            type: dword
```

## Notes

- Registry tuning changes take effect after the relevant service restarts (or a reboot).
- MSI config properties contain the client secret; the install task uses `no_log`.
- Upgrades: the vendor installer does not support in-place upgrade — uninstall then install.
