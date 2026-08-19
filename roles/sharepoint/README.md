# `virtru.dsp_pep.sharepoint`

Install and configure the **Virtru for Microsoft SharePoint** IIS HTTP module on
on-premises SharePoint Server (2016+). Run this role **on each SharePoint server
in the farm**.

The solution is an IIS HTTP module plus a `DSP_API` ASP.NET Core app, backed by
a local SQL Server database and (optionally) an encrypted-search index. This role
automates the parts that can be scripted; the farm-wide registration is done with
a GUI Configuration Tool (see [Manual steps](#manual-steps)).

What it does:

1. Installs prerequisite runtimes — **.NET 8.0 Hosting Bundle** (required for
   `DSP_API` under IIS) and, optionally, **.NET Framework 4.8** (Server 2016) —
   then runs `iisreset` so the ASP.NET Core Module is registered.
2. Installs the module **MSI**.
3. **Merges** DSP/IdP settings into the two config files under
   `%ProgramData%\Virtru\Configs\` — preserving any keys written by the
   Configuration Tool:
   - `Virtru.Sharepoint.Module.config.json` — `keycloakAuthority`,
     `keycloakClientId`, `keycloakClientSecret`, `userIdentityClaim`, plus
     `sharepoint_module_config_extra`.
   - `Virtru.DotNet.SDK.config.json` — `OpenIdConnectConfig.ClientId` /
     `.ClientSecret`.
4. Runs `iisreset` to pick up config changes.
5. `sharepoint_state: absent` removes the module MSI.

## Requirements

- Microsoft SharePoint Server 2016+ (2019 / 2022 Subscription Edition
  recommended; **encrypted search requires 2022 SE**).
- A running DSP with SharePoint support; the `dsp-sharepoint` OIDC client
  (confidential, service account, with the audience/UPN/client-id mappers and
  token-exchange configured) — provisioned by `virtru.dsp_platform`, IdP-side.
- WinRM/PSRP access to the SharePoint server(s).
- Installer sources you supply: the module MSI, the .NET 8.0 Hosting Bundle, and
  (on Server 2016) the .NET Framework 4.8 installer.

## Key variables

| Variable | Default | Description |
| -------- | ------- | ----------- |
| `sharepoint_state` | `present` | `present` / `absent` |
| `sharepoint_installer_src` | — (**required** if installing) | Module MSI path or URL |
| `sharepoint_install_hosting_bundle` | `true` | Install the .NET 8.0 Hosting Bundle |
| `sharepoint_hosting_bundle_src` | — (**required** if above) | `dotnet-hosting-8.0.x-win.exe` path or URL |
| `sharepoint_install_dotnet48` | `false` | Install .NET Framework 4.8 (Server 2016) |
| `sharepoint_dotnet48_src` | — (**required** if above) | NDP48 installer path or URL |
| `sharepoint_manage_config` | `true` | Merge settings into the config JSON files |
| `sharepoint_keycloak_authority` | — (**required** if managing config) | OIDC realm URL |
| `sharepoint_client_id` | `dsp-sharepoint` | Service-account client id |
| `sharepoint_client_secret` | — (**required** if managing config) | Client secret |
| `sharepoint_user_identity_claim` | `email` | `email`, or the UPN claim URI (Option A) |
| `sharepoint_module_config_extra` | `{}` | Extra module-config keys (e.g. KAS URL, DSP API URL) |
| `sharepoint_iisreset` | `true` | `iisreset` after bundle install / config change |
| `sharepoint_reboot` | `false` | Allow a reboot if a runtime install requests one |

See `defaults/main.yml` / `meta/argument_specs.yml` for the full list.

## Example

```yaml
- hosts: sharepoint_servers
  roles:
    - role: virtru.dsp_pep.sharepoint
      vars:
        sharepoint_installer_src: 'C:\media\VirtruSharePoint.msi'
        sharepoint_hosting_bundle_src: 'C:\media\dotnet-hosting-8.0.24-win.exe'
        sharepoint_keycloak_authority: "https://idp.example.com/realms/opentdf"
        sharepoint_client_secret: "{{ vault_sharepoint_client_secret }}"
        sharepoint_module_config_extra:
          kasUrl: "https://dsp.example.com/kas/v2"
          dspApiUrl: "https://sharepoint.example.com/dsp_api"
```

## Manual steps

These are **not** automated by this role:

- **Configuration Tool (GUI).** After the MSI is installed, launch the Virtru for
  SharePoint Configuration Tool to register the HTTP module in `web.config`,
  register `DSP_API` in IIS, set the SQL Server database, and configure the
  SharePoint farm credentials / URL. It reads and writes the same
  `Virtru.Sharepoint.Module.config.json` this role merges into.
- **Keycloak / IdP client.** Create the confidential `dsp-sharepoint` client
  (Standard Flow + Service Accounts), redirect URIs, email/UPN/audience/client-id
  mappers, default `openid profile email` scopes, and enable token exchange with
  the `impersonation` role — see the DSP SharePoint docs and `virtru.dsp_platform`.
- **Encrypted search (2022 SE).** Install the `eng.traineddata` OCR pack via the
  Config Tool, run a full crawl after at least one TDF document exists, then map
  the `VirtruSearch` managed property.

Deployment checklist: `dsp-docs .../04-peps/09-sharepoint-module/08-checklist.md`.

## Verify

```powershell
# ASP.NET Core 8 runtime present (Hosting Bundle installed)
& 'C:\Program Files\dotnet\dotnet.exe' --list-runtimes | Select-String 'AspNetCore.App 8.'

# Config written
Get-Content 'C:\ProgramData\Virtru\Configs\Virtru.Sharepoint.Module.config.json'
```
