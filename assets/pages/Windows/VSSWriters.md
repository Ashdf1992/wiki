# VSS Writer Summary

>Open Powershell or Powershell ISE
>Run the Following
```Powershell
<#
.SYNOPSIS
  Pretty output for `vssadmin list writers` and `vssadmin list providers`.

.DESCRIPTION
  Parses the raw text output from vssadmin using regexes that tolerate:
    - Single or double quotes around names
    - State formats: "State: [1] Stable" OR "State: 1 (Stable)"
    - "Writer Instance Id:" vs "Instance Id:"
    - Inconsistent whitespace/casing

.PARAMETER ExportCsvPath
  Optional folder path to export Writers/Providers CSVs.

.PARAMETER AsJson
  Emit JSON for both objects instead of tables (useful for tooling).

.EXAMPLE
  .\Pretty-VSS.ps1

.EXAMPLE
  .\Pretty-VSS.ps1 -ExportCsvPath C:\Temp

.EXAMPLE
  .\Pretty-VSS.ps1 -AsJson
#>
param(
  [string]$ExportCsvPath,
  [switch]$AsJson
)

function Parse-VSSWriters {
    $raw = vssadmin list writers
    $writers = @()
    $current = [ordered]@{}

    foreach ($line in $raw) {
        $l = ($line -as [string]).Trim()
        if (-not $l) { continue }

        # New writer block
        if ($l -imatch "^\s*Writer name:\s+['""]?(.+?)['""]?\s*$") {
            if ($current.Count) { $writers += [pscustomobject]$current; $current = [ordered]@{} }
            $current.Name = $Matches[1]
            continue
        }

        # Writer Id
        if ($l -imatch "^\s*Writer Id:\s+(\{[0-9a-fA-F\-]+\})\s*$") {
            $current.WriterId = $Matches[1]
            continue
        }

        # Writer Instance Id
        if ($l -imatch "^\s*Writer Instance Id:\s+(\{[0-9a-fA-F\-]+\})\s*$") {
            $current.InstanceId = $Matches[1]
            continue
        }
        # (Some builds show just 'Instance Id:')
        if (-not $current.InstanceId -and $l -imatch "^\s*Instance Id:\s+(\{[0-9a-fA-F\-]+\})\s*$") {
            $current.InstanceId = $Matches[1]
            continue
        }

        # State formats:
        #   State: [1] Stable
        #   State: 1 (Stable)
        if ($l -imatch "^\s*State:\s*\[(\d+)\]\s*(.+)\s*$") {
            $current.StateCode  = [int]$Matches[1]
            $current.StateText  = $Matches[2]
            continue
        }
        if ($l -imatch "^\s*State:\s*(\d+)\s*\((.+)\)\s*$") {
            $current.StateCode  = [int]$Matches[1]
            $current.StateText  = $Matches[2]
            continue
        }

        # Last error
        if ($l -imatch "^\s*Last error:\s*(.+)\s*$") {
            $current.LastError = $Matches[1]
            continue
        }
    }
    if ($current.Count) { $writers += [pscustomobject]$current }
    return $writers
}

function Parse-VSSProviders {
    $raw = vssadmin list providers
    $providers = @()
    $current = [ordered]@{}

    foreach ($line in $raw) {
        $l = ($line -as [string]).Trim()
        if (-not $l) { continue }

        # New provider block
        if ($l -imatch "^\s*Provider name:\s+['""]?(.+?)['""]?\s*$") {
            if ($current.Count) { $providers += [pscustomobject]$current; $current = [ordered]@{} }
            $current.Name = $Matches[1]
            continue
        }

        if ($l -imatch "^\s*Provider Id:\s+(\{[0-9a-fA-F\-]+\})\s*$") {
            $current.ProviderId = $Matches[1]
            continue
        }

        if ($l -imatch "^\s*Version:\s*(.+?)\s*$") {
            $current.Version = $Matches[1]
            continue
        }

        if ($l -imatch "^\s*Type:\s*(.+?)\s*$") {
            $current.Type = $Matches[1]
            continue
        }
    }
    if ($current.Count) { $providers += [pscustomobject]$current }
    return $providers
}

function Show-WriterTable {
    param($writers)

    if (-not $writers -or -not $writers.Count) {
        Write-Host "No VSS Writers found." -ForegroundColor Yellow
        return
    }

    # Add friendly state if missing
    $writers |
        Select-Object `
            @{n='Name'; e={$_.Name}},
            @{n='StateCode'; e={$_.StateCode}},
            @{n='State'; e={$_.StateText}},
            @{n='LastError'; e={$_.LastError}},
            @{n='WriterId'; e={$_.WriterId}},
            @{n='InstanceId'; e={$_.InstanceId}} |
        Sort-Object Name |
        Format-Table -AutoSize
}

function Show-ProviderTable {
    param($providers)

    if (-not $providers -or -not $providers.Count) {
        Write-Host "No VSS Providers found." -ForegroundColor Yellow
        return
    }

    $providers |
        Select-Object Name, Type, Version, ProviderId |
        Sort-Object Name |
        Format-Table -AutoSize
}

# ----------------- MAIN -----------------
$writers   = Parse-VSSWriters
$providers = Parse-VSSProviders

if ($AsJson) {
    [pscustomobject]@{
        Writers   = $writers
        Providers = $providers
    } | ConvertTo-Json -Depth 5
    return
}

Write-Host "`n=== VSS Writers ===" -ForegroundColor Cyan
Show-WriterTable -writers $writers

Write-Host "`n=== VSS Providers ===" -ForegroundColor Cyan
Show-ProviderTable -providers $providers

# Optional exports
if ($ExportCsvPath) {
    try {
        if (-not (Test-Path $ExportCsvPath)) { New-Item -Path $ExportCsvPath -ItemType Directory -Force | Out-Null }
        $writers   | Export-Csv (Join-Path $ExportCsvPath 'VSSWriters.csv')   -NoTypeInformation -Encoding UTF8
        $providers | Export-Csv (Join-Path $ExportCsvPath 'VSSProviders.csv') -NoTypeInformation -Encoding UTF8
        Write-Host "CSV exported to $ExportCsvPath" -ForegroundColor Green
    } catch {
        Write-Warning "CSV export failed: $($_.Exception.Message)"
    }
}

# Optional: quick color-coded summary for writers
if ($writers -and $writers.Count) {
    Write-Host "`n--- Writer State Summary ---"
    foreach ($w in $writers) {
        $state = $w.StateText
        $name  = $w.Name
        $color = if ($state -match 'stable') { 'Green' }
                 elseif ($state -match 'failed|retry' ) { 'Red' }
                 else { 'Yellow' }
        Write-Host ("{0,-40} : {1}" -f $name, $state) -ForegroundColor $color
    }
}
```

Example Output:
```Powershell
=== VSS Writers ===

Name                            StateCode State  LastError WriterId                               InstanceId                            
----                            --------- -----  --------- --------                               ----------                            
ASR Writer                              1 Stable No error  {be000cbe-11fe-4426-9c58-531aa6355fc4} {a3b08074-6781-4975-a323-093d01bce1d7}
COM+ REGDB Writer                       1 Stable No error  {542da469-d3e1-473c-9f4f-7847f01fc64f} {0dcd8805-2d7c-4564-a46c-29b3228dcbd4}
Microsoft Hyper-V VSS Writer            1 Stable No error  {66841cd4-6ded-4f4b-8f17-fd23f8ddc3de} {a4c698dd-f317-4e55-a558-e09eaefef3b4}
Performance Counters Writer             1 Stable No error  {0bada1de-01a9-4625-8278-69e735f39dd2} {f0086dda-9efc-47c5-8eb6-a944c3d09381}
Registry Writer                         1 Stable No error  {afbab4a2-367d-4d15-a586-71dbb18f8485} {cfe29013-75f2-4cf4-8c02-44c94f672c3a}
Shadow Copy Optimization Writer         1 Stable No error  {4dc3bdd4-ab48-4d07-adb0-3bee2926fd7f} {f295ada8-baa2-4085-81f4-fe25b869ebfb}
System Writer                           1 Stable No error  {e8132975-6f93-4464-a53e-1050253ae220} {33f40bfc-add0-4380-8dd9-e34b780e3e8d}
Task Scheduler Writer                   1 Stable No error  {d61d61c8-d73a-4eee-8cdd-f6f9786b7124} {1bddd48e-5052-49db-9b07-b96f96727e6b}
VSS Metadata Store Writer               1 Stable No error  {75dfb225-e2e4-4d39-9ac9-ffaff65ddf06} {088e7a7d-09a8-4cc6-a609-ad90e75ddc93}
WMI Writer                              1 Stable No error  {a6ad56c2-b509-4e6c-bb19-49d8f43532f0} {6ccd404e-4b27-45ad-8b25-590c3ee50836}



=== VSS Providers ===

Name                                        Type Version ProviderId                            
----                                        ---- ------- ----------                            
Microsoft File Share Shadow Copy provider        1.0.0.1 {89300202-3cec-4981-9171-19f59559e0f2}
Microsoft Software Shadow Copy provider 1.0      1.0.0.7 {b5946137-7b9f-4925-af80-51abd60b20d5}



--- Writer State Summary ---
Task Scheduler Writer                    : Stable
VSS Metadata Store Writer                : Stable
Performance Counters Writer              : Stable
System Writer                            : Stable
Microsoft Hyper-V VSS Writer             : Stable
ASR Writer                               : Stable
Shadow Copy Optimization Writer          : Stable
WMI Writer                               : Stable
Registry Writer                          : Stable
COM+ REGDB Writer                        : Stable
```
