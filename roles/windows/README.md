# `virtru.dsp_pep.windows`

Install and configure **Virtru File Services** — the Windows file-watcher PEP
that provides OS-level (minifilter driver) TDF protection via the `virtruflt`,
`virtrusvc`, and `fesfpolicy` services. **Windows 11 24H2 (x64) only.**

What it does:

1. Installs the **Visual C++ 2022 x64 runtime** prerequisite (idempotent).
2. Imports the **driver-signing certificates** (`rootca.pem` → Trusted Root,
   `testsign.pem` → Personal) — before the MSI, so the drivers load without the
   "install this driver anyway" prompt. Optionally trusts the DSP platform TLS cert.
3. Optionally enables **test-signing mode** (`bcdedit /set TESTSIGNING ON`) and
   reboots — required until the drivers are Microsoft cross-signed (needs Secure
   Boot disabled).
4. Installs the **`virtru_file_services-x64-rel.msi`** with its config properties
   (`CLIENT_ID`, `CLIENT_SECRET`, `AUDIENCE`, `PLATFORM_ENDPOINT`,
   `DEFAULT_ATTRIBUTES`) → written to `HKLM\Software\Virtru`. Reboots if required.
5. Optionally applies extra registry tuning keys.

## Requirements

- Windows 11 **24H2** (x64); `ansible.windows` and a WinRM/PSRP connection.
- The MSI and cert files (`rootca.pem`, `testsign.pem`) staged on the target
  (or the MSI reachable by URL).
- The `dsp-windows-filewatcher` Keycloak client with **OAuth 2.0 Device
  Authorization Grant enabled** (off by default — enable it in
  Keycloak → Clients → dsp-windows-filewatcher → Capability config) and a public
  loopback redirect `http://127.0.0.1:*`. Without it you get
  `OAUTH2_DEVICE_AUTH_ERROR` ("the flow is disabled for the client").
- The client must trust the DSP platform TLS certificate (import it via
  `windows_dsp_platform_cert_src` for self-signed / non-domain-joined clients).
- Administrator privileges; Secure Boot disabled if using test-signing.

## Key variables

| Variable | Default | Description |
| -------- | ------- | ----------- |
| `windows_platform_endpoint` | — (**required**) | DSP endpoint incl. port |
| `windows_client_id` | `dsp-windows-filewatcher` | PEP OIDC client id |
| `windows_client_secret` | — (**required**) | PEP OIDC client secret |
| `windows_audience` | = `windows_client_id` | Token audience (matches the client id) |
| `windows_default_attributes` | `""` | `;`-delimited FQNs for new TDFs (**required for PEP ≤ 1.2**) |
| `windows_installer_src` | `""` | Path/URL to the MSI |
| `windows_install_vc_redist` | `true` | Install VC++ 2022 runtime |
| `windows_import_certs` | `true` | Import driver-signing certs |
| `windows_rootca_cert_src` / `windows_testsign_cert_src` | `""` | Cert paths on the target |
| `windows_cert_store_location` | `LocalMachine` | Cert store scope (`LocalMachine` / `CurrentUser`) |
| `windows_dsp_platform_cert_src` | `""` | Optional DSP TLS cert to trust |
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
  Full restart order (stop): `fesfpolicy` → `virtrusvc` → `virtruflt`; start in reverse.
- Service names differ across builds — the FDE 1.1 guide lists `fesfpolicy` while
  older DSP docs reference `fe2policy`. Adjust `windows_services` to match your build.
- MSI config properties contain the client secret; the install task uses `no_log`.
- Upgrades: the vendor installer does not support in-place upgrade — uninstall then install.
