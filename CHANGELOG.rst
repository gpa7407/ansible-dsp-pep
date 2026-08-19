================================
DSP PEP Collection Release Notes
================================

.. contents:: Topics

v1.2.0
======

Minor Changes
-------------

- sharepoint role - document that the role is the Module PEP (not the legacy RER PEP) and warn that POST /sharepoint/sitelist must never be used with a Module PEP deployment (silent assertion_type drop, FEDCD-1055).
- sharepoint_rer role - new role ``virtru.dsp_pep.sharepoint_rer`` for the legacy RER SharePoint PEP. Branches on ansible_os_family - on the SharePoint server it installs/configures the Virtru Claims Provider (Machine.config runtime patch, Add/Install-SPSolution, CCP config object, CCP encoding derivation); on the DSP host it renders the SharePoint values fragment, runs helm upgrade + rollout restart, and registers document libraries via /sharepoint/sitelist with a mandatory GET verification that asserts assertion_type is non-empty.

v1.1.0
======

Minor Changes
-------------

- sharepoint role - new role ``virtru.dsp_pep.sharepoint`` to install and configure the Virtru for Microsoft SharePoint IIS HTTP module on SharePoint Server 2016+. Installs the .NET 8.0 Hosting Bundle (and optionally .NET Framework 4.8) and the module MSI, merges DSP/IdP settings into the module and SDK config JSON files (preserving keys managed by the GUI Configuration Tool), and restarts IIS. Farm registration and Keycloak client setup remain manual/platform-side.

v1.0.0
======

Release Summary
---------------

Initial release of the virtru.dsp_pep collection for installing DSP Policy Enforcement Points (PEPs), one role per PEP: desktop (Virtru Desktop on Windows/macOS), windows (Virtru File Services, the Windows file-watcher), and outlook (the Virtru for Microsoft Outlook add-in for Exchange).
