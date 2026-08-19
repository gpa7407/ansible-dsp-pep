# `virtru.dsp_pep.sharepoint_rer`

Install and configure the **legacy RER SharePoint PEP** — the Remote Event
Receiver service (bundled in DSP) plus the **Virtru Claims Provider (CCP)** farm
solution, DSP-side `values.yaml` configuration, and document-library registration
via the `/sharepoint/sitelist` API.

> ⚠️ **This is the RER PEP, not the Module PEP.** For the IIS HTTP module (Virtru
> for Microsoft SharePoint V1.0) use [`virtru.dsp_pep.sharepoint`](../sharepoint/README.md)
> instead — and never run this role's `/sharepoint/sitelist` step against a Module
> PEP deployment (it silently drops `assertion_type`; FEDCD-1055).

The role **branches on `ansible_os_family`**, so you run it in a **two-play**
playbook ([`playbooks/install_sharepoint_rer.yml`](../../playbooks/install_sharepoint_rer.yml)):

| Target | Tasks |
| ------ | ----- |
| **Windows** (SharePoint server) | Patch `Machine.config` `<runtime>`, `Add`/`Install-SPSolution` the CCP `.wsp`, set the CCP config object (`PlatformEndpoint`/`ClientSecret`), derive the **CCP encoding**, restart `W3SVC`+`SPTimerV4`. |
| **Linux** (DSP/k8s host) | Render the `sharepoint` values fragment, `helm upgrade` + `kubectl rollout restart`, then `POST /sharepoint/sitelist` and **verify** with a `GET` (asserts `assertion_type` is non-empty). |

The CCP encoding derived in the Windows play is passed to the DSP play via
`hostvars` (wired in the example playbook).

## Requirements

- SharePoint Server 2019+ with a root site collection, TLS on 443 with an
  authoritative cert, **HTTP/2 disabled** on the IIS binding, and Alternate
  Access Mappings configured (DSP reaches SharePoint's REST API over HTTP/1).
- A dedicated SharePoint service account (registered managed account, configured
  as the web-app service account).
- Keycloak LDAP **Username attribute = `sAMAccountName`** (not `cn`), or
  visibility trimming breaks.
- DSP must trust SharePoint's CA (use a multi-SAN/wildcard cert from the same CA).
- The `Virtru.CP.wsp` + `assembly-bindings.config` from the DSP bundle
  (`peps/sharepoint/virtru-ccp`).
- On the DSP host: `helm` + `kubectl` on PATH and the chart `.tgz` + values files.

## Key variables

| Variable | Default | Description |
| -------- | ------- | ----------- |
| `sharepoint_rer_state` | `present` | `present` / `absent` |
| `sharepoint_rer_ccp_wsp_src` | — (**required**, Windows) | `Virtru.CP.wsp` path or URL |
| `sharepoint_rer_assembly_bindings_src` | `""` | `assembly-bindings.config` for the Machine.config patch (empty = patch by hand) |
| `sharepoint_rer_platform_endpoint` | — (**required**) | DSP platform endpoint |
| `sharepoint_rer_client_secret` | — (**required**) | OIDC client secret (same as the DSP PEP config) |
| `sharepoint_rer_ccp_encoding` | *(derived)* | Set on the SharePoint server; supply to the DSP play |
| `sharepoint_rer_chart` | — (**required**, DSP) | data-security-platform chart `.tgz` |
| `sharepoint_rer_values_files` | `[]` | Existing `--values` files to keep |
| `sharepoint_rer_farm_fqdn` | — (**required**, DSP) | e.g. `sps1.usa.lab` (stored as `sps1_usa_lab`) |
| `sharepoint_rer_domain` / `_username` / `_password` | `""` | Service-account credentials |
| `sharepoint_rer_dsp_fqdn` | — (**required** for sitelist) | Host for `/sharepoint` API |
| `sharepoint_rer_keycloak_token_url` | — (**required** for sitelist) | Token endpoint |
| `sharepoint_rer_token_client_secret` | — (**required** for sitelist) | Token client secret |
| `sharepoint_rer_sitelists` | `[]` | Document libraries to register (payload list) |

See `defaults/main.yml` / `meta/argument_specs.yml` for the full list.

## Example

```yaml
# group_vars or -e; used by both plays
sharepoint_rer_platform_endpoint: "https://platform.example.com"
sharepoint_rer_client_secret: "{{ vault_sp_pep_secret }}"

# Windows play
sharepoint_rer_ccp_wsp_src: "files/Virtru.CP.wsp"
sharepoint_rer_assembly_bindings_src: "files/assembly-bindings.config"

# DSP play
sharepoint_rer_chart: "./charts/data-security-platform-0.7.2.tgz"
sharepoint_rer_values_files: ["./playground-values.yaml", "./tagging-pdp-workflows.yaml"]
sharepoint_rer_farm_fqdn: "sps1.usa.lab"
sharepoint_rer_domain: "usa"
sharepoint_rer_username: "svc_sp_dsp"
sharepoint_rer_password: "{{ vault_sp_svc_password }}"
sharepoint_rer_dsp_fqdn: "dsp.example.com"
sharepoint_rer_keycloak_token_url: "https://kc.example.com/realms/opentdf/protocol/openid-connect/token"
sharepoint_rer_token_client_secret: "{{ vault_opentdf_secret }}"
sharepoint_rer_sitelists:
  - site_url: "https://sps1.usa.lab/sites/test"
    list_admin: "admin@usa.lab"
    default_tags: ["https://demo.com/attr/relto/value/usa"]
```

## Not automated (manual)

- **TLS/443, HTTP/2 disable, Alternate Access Mappings, root/test site
  collections, service-account creation** — SharePoint-admin prerequisites.
- **Seeding the FLOTs** and **testing in Classic Mode** — done by hand.
- Keycloak LDAP `sAMAccountName` and the DSP trusted-CA cert — platform/IdP side.

## Verify

```powershell
# SharePoint server
Get-SPSolution | ? { $_.Name -eq 'virtru.cp.wsp' }   # Deployed = True
Get-SPClaimProvider | ? { $_.Name -eq 'VirtruCP' }
```

The DSP play already asserts `assertion_type` survived registration — if that
assert fails, the sitelist payload field names are wrong; do not proceed to
testing (files would encrypt with no assertion policy).
