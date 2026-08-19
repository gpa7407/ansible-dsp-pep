# `virtru.dsp_pep.outlook`

Deploy the **Virtru for Microsoft Outlook** add-in to Microsoft Exchange
(Exchange Server 2019 CU12+). Run this role **on the Exchange server** (it uses
the Exchange Management Shell).

What it does:

1. (Optional) preflight-checks that the Outlook service config endpoint
   (`<endpoint>/outlook/config`) is reachable.
2. Downloads the add-in manifest from `<endpoint>/outlook/manifest.xml` to the
   Exchange server.
3. Installs it organization-wide with `New-App -OrganizationApp` (idempotent —
   skips if a matching `DisplayName` is already installed; `outlook_force_reinstall`
   refreshes it).
4. `outlook_state: absent` removes it with `Remove-App`.

## Requirements

- Microsoft Exchange Server 2019 CU12+ (role runs on the Exchange server via
  WinRM/PSRP, with the Exchange Management Shell available).
- A running DSP deployed with the Outlook service enabled — `platform.mode` must
  be `all` or include `outlook` (configured in the `virtru.dsp_platform`
  deployment, not this role).
- The `dsp-outlook` (service account, confidential) and `dsp-outlook-auth`
  (public) OIDC clients with the audience mapper — provisioned by
  `virtru.dsp_platform`. These are IdP-side prerequisites.
- End-user machines: Outlook 2021 (perpetual) + Edge WebView2 runtime.

## Key variables

| Variable | Default | Description |
| -------- | ------- | ----------- |
| `outlook_platform_endpoint` | — (**required**) | DSP base URL, e.g. `https://platform.dsp.vm` |
| `outlook_manifest_url` | `<endpoint>/outlook/manifest.xml` | Manifest source |
| `outlook_validate_certs` | `true` | Verify TLS when fetching (false for self-signed DSP) |
| `outlook_state` | `present` | `present` / `absent` |
| `outlook_enabled` | `true` | `New-App -Enabled` |
| `outlook_default_state_for_user` | `AlwaysEnabled` | `New-App -DefaultStateForUser` |
| `outlook_force_reinstall` | `false` | Remove + re-add to refresh the manifest |
| `outlook_exchange_snapin` | `Microsoft.Exchange.Management.PowerShell.SnapIn` | EMS snap-in loaded if `*-App` cmdlets are missing |
| `outlook_exchange_init_script` | `""` | Optional PowerShell before the cmdlets (e.g. `Connect-ExchangeOnline`) |

See `defaults/main.yml` / `meta/argument_specs.yml` for the full list.

## Example

```yaml
- hosts: exchange_servers
  roles:
    - role: virtru.dsp_pep.outlook
      vars:
        outlook_platform_endpoint: "https://platform.dsp.example.com"
        outlook_validate_certs: false   # self-signed lab DSP
        outlook_default_state_for_user: AlwaysEnabled
```

Verify on the Exchange server:

```powershell
Get-App -OrganizationApp | Format-List DisplayName,AppId   # should list "Virtru"
```

## Notes

- Exchange cmdlets often need the EMS loaded; the role adds `outlook_exchange_snapin`
  when `New-App` isn't already available. For **Exchange Online**, set
  `outlook_exchange_init_script` to your `Connect-ExchangeOnline ...` command.
- The Outlook PEP send path is obligation-based; configure `supported_obligations`
  in the platform's Outlook service config (platform-side, not this role).
