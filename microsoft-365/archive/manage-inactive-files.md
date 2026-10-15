---
title: Manage inactive files with SharePoint Advanced Management
ms.author: chucked
author: chuckedmonson
manager: jtremper
audience: admin
ms.reviewer: sreelakshmi
ms.date: 10/15/2026
ms.topic: how-to
ms.service: microsoft-365-archive
ms.custom: archive
search.appverid:
ms.collection:
    - essentials-manage
ms.localizationpriority: medium
description: Learn how to use SharePoint Advanced Management inactive files policies to automatically archive eligible inactive files.
---

# Manage inactive files with SharePoint Advanced Management (preview)

Inactive files policies in SharePoint Advanced Management help administrators automatically archive eligible SharePoint files based on when they were last accessed. Use these policies to manage inactive files within sites that remain in use.

Microsoft 365 Archive provides the archive storage tier and file reactivation experience. SharePoint Advanced Management policies provide the rules and automation that select eligible files for archiving.

## Availability

Public preview is planned for October 2026. General availability is planned by the end of 2026. These dates are targets and may change.

This article describes a PowerShell-based workflow for SharePoint sites. It doesn't cover OneDrive policy configuration, inactive site policies, or Microsoft Purview retention policy configuration.

OneDrive sites and OneDrive for Business policy configuration aren't supported yet in this preview.

## Choose the right approach

| Approach | Use it when |
|---|---|
| Inactive files policy | You want to automatically archive eligible inactive files within SharePoint sites. |
| Inactive site policy | You want to identify and manage entire sites that are no longer active. |
| Manual file archiving | Users need to select and archive individual files or supported groups of files. |
| Microsoft Purview retention | You need to retain or delete content according to compliance requirements. |

Archiving changes the file's storage and access state; it doesn't replace retention requirements.

For site-level governance, see [SharePoint Advanced Management](/sharepoint/advanced-management).

## Before you begin

Before deploying a policy, confirm:

- The public preview is available for your tenant.
- Your tenant has the required SharePoint Advanced Management entitlement.
- Your administrator account has the SharePoint Administrator or Global Administrator role.
- You're using Microsoft.Online.SharePoint.PowerShell version 16.0.27709.12000 or later.
- Your tenant meets the applicable Microsoft 365 Archive service and billing prerequisites.

For more information, see:

- [SharePoint Advanced Management](/sharepoint/advanced-management)
- [Connect to SharePoint Online PowerShell](/powershell/sharepoint/sharepoint-online/connect-sharepoint-online)
- [Set up Microsoft 365 Archive](archive-setup.md)

Manual file archiving and automatic policy archiving are separate controls. Disabling the user-facing Archive action doesn't, by itself, prevent automatic policies from archiving files.

## Select an inactivity threshold

Inactive files policies use a file's last access date, not the last modified date. Last access date tracks human access. It doesn't track non-human access, such as a script, automation, app, or background process reading the file. A file that is accessed regularly by a script can still be considered inactive by the policy.

A file can remain important even if it hasn't been edited recently. Select a threshold that reflects your organization's access patterns and business requirements.

The policy commands accept an inactivity threshold from 6 through 48 months. The default is 24 months.

> [!IMPORTANT]
> Last access dates before July 2025 might be missing activity signals from some clients. A file can therefore appear older than its actual use. Review this data-quality limitation before enabling enforcement. The default threshold isn't a recommendation for every organization.

The examples below use 12 months to illustrate configuration. That value isn't a universal deployment recommendation.

## Configure scope and exclusions

Inactive files policies support these SharePoint scope types:

| Policy type | Scope |
|---|---|
| SelectedSites | Applies only to the SharePoint sites that you explicitly add. |
| AllSites | Applies broadly to eligible SharePoint sites, except sites that you explicitly exclude. |

A SelectedSites policy must have at least one included site before it can be activated.

Use inclusions only with SelectedSites policies. Use exclusions only with AllSites policies. Adding an inclusion to AllSites, or adding an exclusion to SelectedSites, returns an error.

A site must exist and be eligible for archiving before it can be associated with a policy. A single policy can contain a maximum of 1,000 associated sites. For SelectedSites policies, those associations are inclusions. For AllSites policies, those associations are exclusions.

A site exclusion affects future processing by that policy. It doesn't reactivate files already archived, reverse previous policy actions, or exempt the site from another archive policy or manual archiving method.

## Understand scale and throughput limits

Inactive files policies process at service-managed scale. Processing might span multiple runs depending on scope, file volume, service load, and eligibility results.

The following limits and guidance apply during preview:

| Measure | Value | Type |
|---|---:|---|
| Sites scanned per day | 100,000 sites | Hard limit |
| Files archived per day | 1-2 million files | Throughput guidance |
| Site associations per policy | 1,000 sites | Hard limit |

The sites-scanned limit is a daily processing limit. If a broad policy targets more sites than can be scanned in one day, processing can continue in later runs.

The files-archived value is throughput guidance, not a guarantee. Actual archive volume depends on how many scanned files are eligible, file state, built-in exclusions, site availability, throttling, service health, and policy configuration.

Files evaluated, skipped, failed, and successfully archived are different measures. Don't treat the evaluated-file count as the archived-file count. Review outcome reasons and processing coverage before drawing conclusions from a run.

## Configure file type filters

File type filters let you include specified extensions or exclude specified extensions. Built-in exclusions still apply regardless of those filters.

The Site Assets library doesn't support file-level archiving.

## Understand state and simulation mode

A policy's state and simulation setting are separate.

| State | IsWhatIfMode | Intended processing |
|---|---|---|
| Inactive | `$true` | Simulation is configured, but the policy isn't active. |
| Active | `$true` | Evaluate and report without archiving files. |
| Inactive | `$false` | Enforcement is configured, but the policy isn't active. |
| Active | `$false` | Archive files that qualify when the policy processes them. |

Creating a policy doesn't activate it. Likewise, activating a policy in simulation mode doesn't enable enforcement.

Simulation results are estimates, not a guarantee of the files or storage that will ultimately be archived. File activity, eligibility, policy configuration, and processing coverage can affect the outcome.

## Create a selected-sites simulation

Start with a limited pilot. Inform the relevant site owners and agree on how results are reviewed before enabling enforcement.

Connect to your SharePoint admin endpoint:

```powershell
Connect-SPOService -Url "https://contoso-admin.sharepoint.com"
```

Create an inactive policy with simulation explicitly enabled:

```powershell
New-SPOFileArchivePolicy -Name "MarketingInactiveFilesPreview" -PolicyType "SelectedSites" -LastAccessDateCriteria 12 -FileTypeCriteria @(".docx", ".pdf") -IsWhatIfMode $true
```

List policies and identify the new policy's ID:

```powershell
Get-SPOFileArchivePolicy
```

Enter that ID, add the pilot site, and inspect the configuration:

```powershell
$policyId = [Guid](Read-Host "Enter the new policy ID")
Add-SPOSiteToFileArchivePolicy -PolicyId $policyId -Site "https://contoso.sharepoint.com/sites/marketing"
Get-SPOFileArchivePolicySites -PolicyId $policyId
Get-SPOFileArchivePolicy -PolicyId $policyId
```

Activate simulation:

```powershell
Set-SPOFileArchivePolicy -PolicyId $policyId -State "Active"
```

Activation allows the policy to process. It doesn't guarantee immediate completion or an immediately available report.

## Create an all-sites simulation with exclusions

Use AllSites only when broad tenant scope is intentional. Start in simulation mode and identify sites that must be exempt before activation.

Create the policy:

```powershell
New-SPOFileArchivePolicy -Name "TenantInactiveFilesPreview" -PolicyType "AllSites" -LastAccessDateCriteria 24 -IsWhatIfMode $true
```

List policies and enter the new policy's ID:

```powershell
Get-SPOFileArchivePolicy
$policyId = [Guid](Read-Host "Enter the new policy ID")
```

Add a site exclusion by using `-Exclude`:

```powershell
Add-SPOSiteToFileArchivePolicy -PolicyId $policyId -Site "https://contoso.sharepoint.com/sites/legal" -Exclude
```

Review the policy and its exclusions before activation:

```powershell
Get-SPOFileArchivePolicySites -PolicyId $policyId
Get-SPOFileArchivePolicy -PolicyId $policyId
```

Activate simulation only after confirming the intended broad scope and exclusions:

```powershell
Set-SPOFileArchivePolicy -PolicyId $policyId -State "Active"
```

For an AllSites policy, `Get-SPOFileArchivePolicySites` returns the excluded sites that are exempt from that policy.

## Review simulation results

Retrieve the policy's run reports:

```powershell
Get-SPOFileArchivePolicyReport -PolicyId $policyId
```

The report command returns run history for the preceding 12 months. Simulation reports describe files and storage that would have been archived.

Before enabling enforcement, review:

- Policy configuration
- Processing coverage
- Projected impact
- Outcome reasons
- Business requirements
- Whether processing spans multiple runs

Don't interpret an empty report or a zero count as proof that the entire intended scope was evaluated.

Don't assume that changing modes archives precisely the same set of files previously evaluated.

## Enable enforcement deliberately

Before switching modes, confirm that the pilot evaluation is complete, its coverage is understood, and the relevant stakeholders have approved archival.

Configure enforcement while the policy is inactive:

```powershell
Set-SPOFileArchivePolicy -PolicyId $policyId -State "Inactive"
Set-SPOFileArchivePolicy -PolicyId $policyId -IsWhatIfMode $false
Get-SPOFileArchivePolicy -PolicyId $policyId
```

Changing the state to Inactive must not be assumed to cancel an already queued or running operation immediately.

After reviewing the configuration and processing boundaries, activate enforcement:

```powershell
Set-SPOFileArchivePolicy -PolicyId $policyId -State "Active"
```

## Monitor policy activity

Use policy reports to review processing outcomes. Use the Microsoft 365 audit log to investigate file archive activity.

The Archived file (FileArchived) activity distinguishes user and system actions. Automatic policy archive events include the policy ID, policy version, and file's last access date.

For details, see [Audit log activities](/purview/audit-log-activities).

Keep the configuration used for a historical run separate from the policy's current settings when interpreting results.

## Change policy scope or stop a policy

Use `Set-SPOFileArchivePolicy` to update supported settings. Parameters you omit retain their existing values.

To make a policy inactive:

```powershell
Set-SPOFileArchivePolicy -PolicyId $policyId -State "Inactive"
```

To remove a site association:

```powershell
Remove-SPOSiteToFileArchivePolicy -PolicyId $policyId -Site "https://contoso.sharepoint.com/sites/marketing"
```

The effect depends on the policy type:

- For SelectedSites, removing an inclusion reduces future policy coverage.
- For AllSites, removing an exclusion expands future policy coverage to that site.

You can't remove the final included site from an active SelectedSites policy. Make the policy inactive first or add another included site.

To preview policy removal, then remove it intentionally:

```powershell
Remove-SPOFileArchivePolicy -PolicyId $policyId -WhatIf
Remove-SPOFileArchivePolicy -PolicyId $policyId
```

Here, `-WhatIf` previews the removal command. It's different from the policy's persistent `IsWhatIfMode` setting.

Removing a policy soft-deletes the policy; it doesn't delete or reactivate the files it archived. Accessing archived files still requires reactivation.

## Understand storage, access, and compliance

Archived file contents require reactivation before users can open them. Users with read access can reactivate files through the supported SharePoint experience.

SharePoint file reactivation is free. Reactivated files can't be archived again for 120 days.

Archiving files changes tenant-level active and archived storage accounting. It doesn't reduce site-level storage usage or bypass site quota enforcement.

Microsoft 365 Archive continues to honor applicable compliance protections. That compatibility doesn't establish that every file qualifies for automatic policy archiving. Policy-specific eligibility for retention, holds, records, locked content, and the Preservation Hold Library must be documented separately.

## Troubleshoot unexpected results

| Symptom | What to examine |
|---|---|
| The policy doesn't process. | Policy state, selected-site associations or broad-scope exclusions, preview eligibility, entitlement, and service prerequisites. |
| The policy processes but doesn't archive. | Simulation mode, activity criteria, filters, built-in restrictions, and processing outcomes. |
| A file appears unexpectedly eligible. | Historical access-signal limitations, non-human access not being counted as last access, and the configured threshold. |
| Archive results differ from simulation. | Changes in activity or eligibility, processing coverage, outcome reasons, and the configuration used for each run. |
| A report includes files that weren't archived. | Whether the reported count represents evaluated, skipped, failed, or successfully archived files; review the outcome reasons. |
| An excluded site is still affected. | Whether the exclusion was added to the correct AllSites policy, whether another policy or manual method applies, and whether the files were archived before the exclusion was added. |
| A stopped policy still has archived files. | Policy state changes don't reactivate files. |
| Site storage remains unchanged. | File archiving changes tenant storage accounting, not site quota usage. |
| A OneDrive site isn't processed. | OneDrive sites aren't supported yet in this preview. |

## Related commands

- [New-SPOFileArchivePolicy](/powershell/module/microsoft.online.sharepoint.powershell/new-spofilearchivepolicy)
- [Set-SPOFileArchivePolicy](/powershell/module/microsoft.online.sharepoint.powershell/set-spofilearchivepolicy)
- [Get-SPOFileArchivePolicy](/powershell/module/microsoft.online.sharepoint.powershell/get-spofilearchivepolicy)
- [Remove-SPOFileArchivePolicy](/powershell/module/microsoft.online.sharepoint.powershell/remove-spofilearchivepolicy)
- [Add-SPOSiteToFileArchivePolicy](/powershell/module/microsoft.online.sharepoint.powershell/add-spositetofilearchivepolicy)
- [Remove-SPOSiteToFileArchivePolicy](/powershell/module/microsoft.online.sharepoint.powershell/remove-spositetofilearchivepolicy)
- [Get-SPOFileArchivePolicySites](/powershell/module/microsoft.online.sharepoint.powershell/get-spofilearchivepolicysites)
- [Get-SPOFileArchivePolicyReport](/powershell/module/microsoft.online.sharepoint.powershell/get-spofilearchivepolicyreport)
