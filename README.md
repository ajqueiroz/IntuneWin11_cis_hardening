# CIS Windows 11 – Intune policy set

Built from **CIS Microsoft Intune for Windows 11 Benchmark v5.0.0** and **CIS Microsoft Intune for Microsoft Defender Antivirus Benchmark v1.0.0**.
Use it with the Qualys policy **"CIS Microsoft Intune for Windows 11 Benchmark v5.0.0"**.

## What is in this folder

| Folder / file | What it is |
|---|---|
| `SettingsCatalog\` | 23 Settings Catalog policies: Windows 11 Level 1 (14), Level 2 (5), BitLocker (1), Defender Antivirus (3) |
| `CompliancePolicies\` | 1 compliance policy (BitLocker, Secure Boot, code integrity, firewall, TPM, antivirus, antispyware) |
| `Remediation\` | Detection + remediation scripts that disable the CIS services (section 82) |
| `Import-CISPolicies.ps1` | Imports everything above into Intune through Microsoft Graph. Creates nothing that already exists and assigns nothing |
| `CIS_Win11_Intune_coverage.xlsx` | Every CIS recommendation → which policy or script covers it |

## Before importing

1. **Edit `SettingsCatalog\CIS W11 v5 - L1 - Organisation Settings (EDIT BEFORE IMPORT).json`**:
   replace the four `CHANGE-ME` values (renamed admin account, renamed guest account, logon banner title and text).
   The import script skips this file while it still contains `CHANGE-ME`.
2. **Remove overlapping baselines from the target devices.** If the Microsoft Security Baseline, another CIS set or any
   other hardening policies are assigned to the same devices, settings configured twice with different values conflict and
   are applied by neither.
3. **Check your choices** (sheet *Your choices* in the workbook): RDP idle limit 15 minutes, telemetry *Required*,
   UAC admin prompt *consent on secure desktop*, firewall log paths, Defender scan times and ASR modes.

## Import

**Option A – script (all at once)** – PowerShell 7, account with the Intune Administrator role:

```powershell
.\Import-CISPolicies.ps1 -TenantId <your-tenant-id> -WhatIf     # preview
.\Import-CISPolicies.ps1 -TenantId <your-tenant-id>             # import
# optional: -SkipLevel2  -SkipBitLocker
```

**Option B – Intune portal (one by one):**
Settings Catalog files: *Devices > Configuration > Create > Import policy*.
Compliance and Remediation: use the script, or recreate them in the portal from the JSON / .ps1 files.

## Rollout order (recommended)

1. Assign **everything to a pilot group** (a few IT devices).
2. Watch these first – they are the ones that most often break something:
   - *SMB Client and Server* – SMB 3.1.1 minimum and encryption can block old NAS, printers and servers.
   - *User Rights* – who can log on locally and through Remote Desktop.
   - *Device Guard, VBS and LSA* – Credential Guard / memory integrity on older hardware and drivers.
   - *Firewall* – inbound traffic is blocked on all profiles.
   - *Attack Surface Reduction rules* – 6 rules are in **Audit** mode (CIS allows it); review Defender reports before moving them to Block.
3. Add the **Remediation** with a daily schedule. Level 2 services (Print Spooler, Remote Desktop, Server/file sharing,
   WinRM, Bluetooth, Error Reporting…) are **off**; set `$IncludeLevel2 = $true` in **both** scripts only if you want them.
4. Scan the pilot devices with Qualys, fix anything unexpected, then roll out in rings.

## How this was built and checked

- Windows 11: every setting ID and value comes from a reviewed CIS-to-Intune mapping. It was checked against your CIS PDF:
  all 419 recommendations present with the same level, values spot-checked against the PDF remediation text,
  405 of 414 setting IDs also appear in real Intune exports. Two corrections were made: *Restrict clients allowed to make
  remote calls to SAM* uses the SDDL value Intune expects, and *Hardened UNC Paths* uses one entry per path with all three flags.
- Defender: 44 settings use structures copied from real Intune exports with CIS values. These 11 setting IDs follow
  Microsoft's naming and were not seen in an export – if Intune rejects one on import, remove only that setting from the JSON:

```
device_vendor_msft_defender_configuration_behavioralnetworkblocks_remoteencryptionprotection_remoteencryptionprotectionaggressiveness
device_vendor_msft_defender_configuration_behavioralnetworkblocks_remoteencryptionprotection_remoteencryptionprotectionconfiguredstate
device_vendor_msft_defender_configuration_daysuntilaggressivecatchupquickscan
device_vendor_msft_policy_config_admx_microsoftdefenderantivirus_disableblockatfirstseen
device_vendor_msft_policy_config_admx_microsoftdefenderantivirus_disableroutinelytakingaction
device_vendor_msft_policy_config_admx_microsoftdefenderantivirus_realtimeprotection_disablescanonrealtimeenable
device_vendor_msft_policy_config_admx_microsoftdefenderantivirus_reporting_disablegenericreports
device_vendor_msft_policy_config_admx_microsoftdefenderantivirus_spynet_localsettingoverridespynetreporting
device_vendor_msft_policy_config_defender_scanparameter
device_vendor_msft_policy_config_defender_schedulescanday
device_vendor_msft_policy_config_defender_schedulescantime
```

- Nothing here has been imported into a real tenant yet. Test in a pilot group first.
