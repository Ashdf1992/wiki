# Windows Server - Roles & Features Sync

## Powershell
>Open Powershell or Powershell ISE

>Run the Following
```Powershell
<#
.SYNOPSIS
    Audit and replicate Windows Server roles & features with an optional GUI.

.DESCRIPTION
    - Mode 'Export': Audits roles/features from a source server (default: localhost),
      writes CSV/JSON, prints a summary, and generates a separate installer .ps1.
    - Mode 'Install': Installs the same roles/features onto target server(s),
      using either the exported JSON or a live query from a source server.
    - Optional WinForms dialog (mirrors your DFS-R form style) to pick mode, source, targets,
      export folder (+ toggles for IncludeMgmtTools, Restart, Use latest JSON, Credentials, Feature Source, Installer name).

.NOTES
    - Requires ServerManager module (present on Windows Server).
    - Remote installs use PowerShell remoting (WinRM).
    - Designed to run from PowerShell ISE (STA). Console hosts should be started with -STA to use the GUI.
#>

#requires -Version 5.1

#region ========================= UI (Selection Form) ===========================

function Show-FeatureSyncSelectionForm {
    [CmdletBinding()]
    param(
        [string]$DefaultMode = 'Export',
        [string]$DefaultSourceComputer = 'localhost',
        [string[]]$DefaultTargets = @('SERVER-TARGET01'),
        [string]$DefaultExportFolder = 'C:\Temp\ServerFeatures',
        [bool]$DefaultIncludeManagementTools = $true,
        [bool]$DefaultRestartIfNeeded        = $false,
        [bool]$DefaultUseExportedJsonFirst   = $true,
        [bool]$DefaultUseCredential          = $false,
        [string]$DefaultFeatureSource        = '',
        [string]$DefaultInstallerScriptName  = 'Install-ServerFeatures.ps1'
    )

    Add-Type -AssemblyName System.Windows.Forms
    Add-Type -AssemblyName System.Drawing

    # Helpers to guarantee integer args & avoid op_Addition/op_Subtraction issues on PS 5.1
    function New-Point([int]$x,[int]$y) { New-Object System.Drawing.Point -ArgumentList $x,$y }
    function New-Size ([int]$w,[int]$h) { New-Object System.Drawing.Size  -ArgumentList $w,$h }

    # ---------- Form ----------
    $form = New-Object System.Windows.Forms.Form
    $form.Text = "Windows Server - Roles & Features Sync"
    $form.Size = New-Size 720 560
    $form.StartPosition = "CenterScreen"
    $form.FormBorderStyle = "FixedDialog"
    $form.MaximizeBox = $false
    $form.MinimizeBox = $false
    $form.TopMost = $true
    $form.BackColor = [System.Drawing.Color]::FromArgb(245, 247, 250)
    $form.Font = New-Object System.Drawing.Font("Segoe UI", 9)

    # ---------- Header ----------
    $headerPanel = New-Object System.Windows.Forms.Panel
    $headerPanel.Location = New-Point 0 0
    $headerPanel.Size     = New-Size 720 60
    $headerPanel.BackColor = [System.Drawing.Color]::FromArgb(0, 120, 212)

    $lblHeader = New-Object System.Windows.Forms.Label
    $lblHeader.Text = "Roles & Features Replication"
    $lblHeader.Location = New-Point 18 10
    $lblHeader.Size = New-Size 560 22
    $lblHeader.ForeColor = [System.Drawing.Color]::White
    $lblHeader.Font = New-Object System.Drawing.Font("Segoe UI Semibold", 12)
    $lblHeader.AutoSize = $false

    $lblHeaderSub = New-Object System.Windows.Forms.Label
    $lblHeaderSub.Text = "Select mode, source, targets, and options. Export folder shows in Export, or in Install when 'Use latest exported JSON' is ticked."
    $lblHeaderSub.Location = New-Point 18 33
    $lblHeaderSub.Size = New-Size 680 18
    $lblHeaderSub.ForeColor = [System.Drawing.Color]::FromArgb(230, 240, 255)
    $lblHeaderSub.Font = New-Object System.Drawing.Font("Segoe UI", 8.5)
    $lblHeaderSub.AutoSize = $false

    $headerPanel.Controls.AddRange(@($lblHeader, $lblHeaderSub))

    # ---------- Group box ----------
    $grpSelection = New-Object System.Windows.Forms.GroupBox
    $grpSelection.Text = "Selection"
    $grpSelection.Location = New-Point 18 72
    $grpSelection.Size = New-Size 680 400
    $grpSelection.Font = New-Object System.Drawing.Font("Segoe UI Semibold", 9)

    # Layout constants
    [int]$labelX  = 18
    [int]$fieldX  = 160
    [int]$fieldW1 = 500
    [int]$browseW = 90

    # Row Y positions (roomy rhythm)
    [int]$rowModeY      = 36
    [int]$rowSourceY    = 76
    [int]$rowTargetY    = 116
    [int]$rowHintY      = 144

    [int]$rowExportLblY = 180           # Export Folder label
    [int]$rowExportBoxY = 206           # Export Folder textbox + Browse

    [int]$rowChk1Y      = 244
    [int]$rowChk2Y      = 268
    [int]$rowChk3Y      = 292
    [int]$rowChk4Y      = 316

    [int]$rowSrcLblY    = 348           # Feature Source label (own row)
    [int]$rowSrcBoxY    = 374           # Feature Source textbox + Browse

    [int]$rowInstLblY   = 408
    [int]$rowInstBoxY   = 432

    # ---------- Labels ----------
    $lblMode = New-Object System.Windows.Forms.Label
    $lblMode.Text = "Mode"
    $lblMode.Location = New-Point $labelX $rowModeY
    $lblMode.Size = New-Size 130 20

    $lblSource = New-Object System.Windows.Forms.Label
    $lblSource.Text = "Source Computer"
    $lblSource.Location = New-Point $labelX $rowSourceY
    $lblSource.Size = New-Size 130 20

    $lblTargets = New-Object System.Windows.Forms.Label
    $lblTargets.Text = "Target Computer"
    $lblTargets.Location = New-Point $labelX $rowTargetY
    $lblTargets.Size = New-Size 130 20

    $lblTargetsHint = New-Object System.Windows.Forms.Label
    $lblTargetsHint.Text = "Comma-separated (e.g., SRV-A,SRV-B)"
    $lblTargetsHint.Location = New-Point $fieldX $rowHintY
    $lblTargetsHint.Size = New-Size 500 18
    $lblTargetsHint.ForeColor = [System.Drawing.Color]::FromArgb(90, 90, 90)
    $lblTargetsHint.Font = New-Object System.Drawing.Font("Segoe UI", 8)

    $lblExportFolder = New-Object System.Windows.Forms.Label
    $lblExportFolder.Text = "Export Folder"
    $lblExportFolder.Location = New-Point $labelX $rowExportLblY
    $lblExportFolder.Size = New-Size 130 20

    $lblFeatureSource = New-Object System.Windows.Forms.Label
    $lblFeatureSource.Text = "Feature Source"   # (no "(SxS)")
    $lblFeatureSource.Location = New-Point $labelX $rowSrcLblY
    $lblFeatureSource.Size = New-Size 130 20

    $lblInstallerName = New-Object System.Windows.Forms.Label
    $lblInstallerName.Text = "Installer Script Name"
    $lblInstallerName.Location = New-Point $labelX $rowInstLblY
    $lblInstallerName.Size = New-Size 130 20

    # ---------- Inputs ----------
    $cmbMode = New-Object System.Windows.Forms.ComboBox
    $cmbMode.Location = New-Point $fieldX ($rowModeY - 4)
    $cmbMode.Size = New-Size 220 26
    $cmbMode.DropDownStyle = [System.Windows.Forms.ComboBoxStyle]::DropDownList
    [void]$cmbMode.Items.Add('Export')
    [void]$cmbMode.Items.Add('Install')

    $txtSource = New-Object System.Windows.Forms.TextBox
    $txtSource.Location = New-Point $fieldX ($rowSourceY - 4)
    $txtSource.Size = New-Size $fieldW1 26

    $txtTargets = New-Object System.Windows.Forms.TextBox
    $txtTargets.Location = New-Point $fieldX ($rowTargetY - 4)
    $txtTargets.Size = New-Size $fieldW1 26

    # Export Folder textbox + Browse on a dedicated row
    $txtExportFolder = New-Object System.Windows.Forms.TextBox
    $txtExportFolder.Location = New-Point $fieldX ($rowExportBoxY - 4)
    $txtExportFolder.Size = New-Size ($fieldW1 - $browseW - 8) 26

    $btnBrowseExport = New-Object System.Windows.Forms.Button
    $btnBrowseExport.Text = "Browse..."
    $btnBrowseExport.Location = New-Point ($fieldX + $fieldW1 - $browseW) ($rowExportBoxY - 4)
    $btnBrowseExport.Size = New-Size $browseW 26
    $btnBrowseExport.FlatStyle = "Flat"
    $btnBrowseExport.UseVisualStyleBackColor = $true

    # Checkboxes (vertical stack)
    $chkIncludeMgmt = New-Object System.Windows.Forms.CheckBox
    $chkIncludeMgmt.Text = "Include management tools"
    $chkIncludeMgmt.Location = New-Point ($labelX + 4) $rowChk1Y
    $chkIncludeMgmt.Size = New-Size 360 20

    $chkRestart = New-Object System.Windows.Forms.CheckBox
    $chkRestart.Text = "Restart if install requires it"
    $chkRestart.Location = New-Point ($labelX + 4) $rowChk2Y
    $chkRestart.Size = New-Size 360 20

    $chkUseJson = New-Object System.Windows.Forms.CheckBox
    $chkUseJson.Text = "Use latest exported JSON (if available)"
    $chkUseJson.Location = New-Point ($labelX + 4) $rowChk3Y
    $chkUseJson.Size = New-Size 380 20

    $chkCred = New-Object System.Windows.Forms.CheckBox
    $chkCred.Text = "Prompt for credentials"
    $chkCred.Location = New-Point ($labelX + 4) $rowChk4Y
    $chkCred.Size = New-Size 360 20

    # Feature Source textbox + Browse on a dedicated row (under label)
    $txtFeatureSource = New-Object System.Windows.Forms.TextBox
    $txtFeatureSource.Location = New-Point $fieldX ($rowSrcBoxY - 4)
    $txtFeatureSource.Size = New-Size ($fieldW1 - $browseW - 8) 26

    $btnBrowseSource = New-Object System.Windows.Forms.Button
    $btnBrowseSource.Text = "Browse..."
    $btnBrowseSource.Location = New-Point ($fieldX + $fieldW1 - $browseW) ($rowSrcBoxY - 4)
    $btnBrowseSource.Size = New-Size $browseW 26
    $btnBrowseSource.FlatStyle = "Flat"
    $btnBrowseSource.UseVisualStyleBackColor = $true

    $txtInstallerName = New-Object System.Windows.Forms.TextBox
    $txtInstallerName.Location = New-Point $fieldX ($rowInstBoxY - 4)
    $txtInstallerName.Size = New-Size $fieldW1 26

    $grpSelection.Controls.AddRange(@(
        $lblMode, $cmbMode,
        $lblSource, $txtSource,
        $lblTargets, $txtTargets, $lblTargetsHint,
        $lblExportFolder, $txtExportFolder, $btnBrowseExport,
        $chkIncludeMgmt, $chkRestart, $chkUseJson, $chkCred,
        $lblFeatureSource, $txtFeatureSource, $btnBrowseSource,
        $lblInstallerName, $txtInstallerName
    ))

    # ---------- Status & Buttons ----------
    $lblStatus = New-Object System.Windows.Forms.Label
    $lblStatus.Location = New-Point 20 484   # above buttons
    $lblStatus.Size = New-Size 480 22        # fixed width so it won't reach buttons
    $lblStatus.ForeColor = [System.Drawing.Color]::FromArgb(70, 70, 70)
    $lblStatus.Text = "Ready."
    $lblStatus.AutoSize = $false
    $lblStatus.AutoEllipsis = $true

    $btnOK = New-Object System.Windows.Forms.Button
    $btnOK.Text = "Run"
    $btnOK.Location = New-Point 528 480
    $btnOK.Size = New-Size 80 30
    $btnOK.FlatStyle = "Flat"
    $btnOK.UseVisualStyleBackColor = $true

    $btnCancel = New-Object System.Windows.Forms.Button
    $btnCancel.Text = "Cancel"
    $btnCancel.Location = New-Point 618 480
    $btnCancel.Size = New-Size 80 30
    $btnCancel.FlatStyle = "Flat"
    $btnCancel.UseVisualStyleBackColor = $true

    $form.AcceptButton = $btnOK
    $form.CancelButton = $btnCancel

    $form.Controls.AddRange(@($headerPanel, $grpSelection, $lblStatus, $btnOK, $btnCancel))

    # ---------- Defaults ----------
    if ($DefaultMode -and ($DefaultMode -in @('Export','Install'))) { $cmbMode.SelectedItem = $DefaultMode } else { $cmbMode.SelectedItem = 'Export' }
    $txtSource.Text       = $DefaultSourceComputer
    $txtTargets.Text      = ($DefaultTargets -join ',')
    $txtExportFolder.Text = $DefaultExportFolder
    $chkIncludeMgmt.Checked = [bool]$DefaultIncludeManagementTools
    $chkRestart.Checked     = [bool]$DefaultRestartIfNeeded
    $chkUseJson.Checked     = [bool]$DefaultUseExportedJsonFirst
    $chkCred.Checked        = [bool]$DefaultUseCredential
    $txtFeatureSource.Text  = $DefaultFeatureSource
    $txtInstallerName.Text  = $DefaultInstallerScriptName

    # ---------- Visibility rules ----------
    $applyVisibility = {
        $isInstall = ($cmbMode.SelectedItem -eq 'Install')
        $isExport  = ($cmbMode.SelectedItem -eq 'Export')
        $needsExportFolder = $isExport -or ($isInstall -and $chkUseJson.Checked)

        # Target fields only in Install
        $lblTargets.Visible     = $isInstall
        $txtTargets.Visible     = $isInstall
        $lblTargetsHint.Visible = $isInstall

        # Export folder visible in Export, or in Install when "use JSON" is ticked
        $lblExportFolder.Visible = $needsExportFolder
        $txtExportFolder.Visible = $needsExportFolder
        $btnBrowseExport.Visible = $needsExportFolder

        if ($isExport) {
            $lblStatus.Text = "Export mode selected: choose export folder."
        } elseif ($chkUseJson.Checked) {
            $lblStatus.Text = "Install mode: using latest JSON from export folder (if found)."
        } else {
            $lblStatus.Text = "Install mode: querying source live."
        }
    }

    $cmbMode.add_SelectedIndexChanged($applyVisibility)
    $chkUseJson.add_CheckedChanged($applyVisibility)

    # ---------- Browsers ----------
    $btnBrowseExport.add_Click({
        $dlg = New-Object System.Windows.Forms.FolderBrowserDialog
        $dlg.Description = "Select folder to save/load CSV/JSON and generated installer"
        $dlg.ShowNewFolderButton = $true
        if ($txtExportFolder.Text) { $dlg.SelectedPath = $txtExportFolder.Text }
        if ($dlg.ShowDialog() -eq [System.Windows.Forms.DialogResult]::OK) {
            $txtExportFolder.Text = $dlg.SelectedPath
        }
    })

    $btnBrowseSource.add_Click({
        $dlg = New-Object System.Windows.Forms.FolderBrowserDialog
        $dlg.Description = "Select Feature Source path (e.g., D:\sources\sxs)"
        $dlg.ShowNewFolderButton = $false
        if ($txtFeatureSource.Text) { $dlg.SelectedPath = $txtFeatureSource.Text }
        if ($dlg.ShowDialog() -eq [System.Windows.Forms.DialogResult]::OK) {
            $txtFeatureSource.Text = $dlg.SelectedPath
        }
    })

    # ---------- Validate & return ----------
    $btnOK.add_Click({
        try {
            $mode = [string]$cmbMode.SelectedItem
            if (-not $mode) {
                [System.Windows.Forms.MessageBox]::Show("Please select a mode (Export or Install).","Validation",[System.Windows.Forms.MessageBoxButtons]::OK,[System.Windows.Forms.MessageBoxIcon]::Warning) | Out-Null
                return
            }

            $source = ($txtSource.Text).Trim()
            if ([string]::IsNullOrWhiteSpace($source)) {
                [System.Windows.Forms.MessageBox]::Show("Please enter a source computer.","Validation",[System.Windows.Forms.MessageBoxButtons]::OK,[System.Windows.Forms.MessageBoxIcon]::Warning) | Out-Null
                return
            }

            $targets = @()
            if ($mode -eq 'Install') {
                if ($txtTargets.Text -and -not [string]::IsNullOrWhiteSpace($txtTargets.Text)) {
                    $targets = ($txtTargets.Text -split ',') | ForEach-Object { $_.Trim() } | Where-Object { $_ }
                }
                if ($targets.Count -eq 0) {
                    [System.Windows.Forms.MessageBox]::Show("Please enter at least one target computer for Install mode.","Validation",[System.Windows.Forms.MessageBoxButtons]::OK,[System.Windows.Forms.MessageBoxIcon]::Warning) | Out-Null
                    return
                }
            }

            if ($mode -eq 'Export') {
                $exportFolder = $txtExportFolder.Text
                if ([string]::IsNullOrWhiteSpace($exportFolder)) {
                    [System.Windows.Forms.MessageBox]::Show("Please select an export folder.","Validation",[System.Windows.Forms.MessageBoxButtons]::OK,[System.Windows.Forms.MessageBoxIcon]::Warning) | Out-Null
                    return
                }
                if (-not (Test-Path -LiteralPath $exportFolder)) {
                    try { New-Item -ItemType Directory -Force -Path $exportFolder | Out-Null }
                    catch {
                        [System.Windows.Forms.MessageBox]::Show(("Unable to create export folder: {0}`r`n{1}" -f $exportFolder, $_.Exception.Message),"Folder Error",[System.Windows.Forms.MessageBoxButtons]::OK,[System.Windows.Forms.MessageBoxIcon]::Error) | Out-Null
                        return
                    }
                }
            } else {
                # In Install, we pass through whatever is typed; main script uses it only when UseExportedJsonFirst is true.
                $exportFolder = $txtExportFolder.Text
            }

            $script:FeatureSyncSelection = [pscustomobject]@{
                Mode                   = $mode
                SourceComputer         = $source
                TargetComputers        = $targets
                ExportFolder           = $exportFolder
                IncludeManagementTools = [bool]$chkIncludeMgmt.Checked
                RestartIfNeeded        = [bool]$chkRestart.Checked
                UseExportedJsonFirst   = [bool]$chkUseJson.Checked
                UseCredential          = [bool]$chkCred.Checked
                FeatureSource          = ($txtFeatureSource.Text).Trim()
                InstallerScriptName    = ($txtInstallerName.Text).Trim()
            }

            $form.DialogResult = [System.Windows.Forms.DialogResult]::OK
            $form.Close()
        } catch {
            [System.Windows.Forms.MessageBox]::Show("Unexpected error: $($_.Exception.Message)","Error",[System.Windows.Forms.MessageBoxButtons]::OK,[System.Windows.Forms.MessageBoxIcon]::Error) | Out-Null
        }
    })

    $btnCancel.add_Click({
        $form.DialogResult = [System.Windows.Forms.DialogResult]::Cancel
        $form.Close()
    })

    & $applyVisibility
    [void]$form.ShowDialog()

    if ($form.DialogResult -eq [System.Windows.Forms.DialogResult]::OK) {
        return $script:FeatureSyncSelection
    }
    return $null
}

#endregion ====================== UI (Selection Form) ==========================


#region ========================= CONFIG (EDIT ME) =============================

# Use the GUI? Set to $false to drive via variables below without the form.
$UseUi = $true

# Operation mode: 'Export' or 'Install' (UI overrides this)
$Mode = 'Export'

# Source and target servers (UI overrides these)
$SourceComputer   = 'localhost'
$TargetComputers  = @('SERVER-TARGET01')

# Output folder (used in Export mode and also for Install if UseExportedJsonFirst)
$ExportFolder     = 'C:\Temp\ServerFeatures'

# Generated installer script name
$GeneratedScriptName = 'Install-ServerFeatures.ps1'

# Install behavior
$IncludeManagementTools = $true       # include management tools during install
$RestartIfNeeded        = $false      # restart target if installation requires it
$UseExportedJsonFirst   = $true       # for Install mode, prefer using latest JSON from $ExportFolder

# Optional: Feature source (e.g., SxS path for .NET 3.5), leave empty if not needed
$FeatureSource = ''  # e.g., 'D:\sources\sxs' or '\\fileserver\WinSxS\Media'

# Optional: Credentials for remote Invoke-Command (if needed)
$UseCredential = $false
$Credential    = $null   # when $UseCredential = $true, the script will prompt for credentials

#endregion ====================== CONFIG (EDIT ME) =============================


#region ========================= SAFETY & PREREQS =============================

function Test-IsElevated {
    try {
        $current = [Security.Principal.WindowsIdentity]::GetCurrent()
        $principal = New-Object Security.Principal.WindowsPrincipal($current)
        return $principal.IsInRole([Security.Principal.WindowsBuiltinRole]::Administrator)
    } catch {
        return $false
    }
}

if (-not (Test-IsElevated)) {
    Write-Warning "This script should be run in an elevated PowerShell session (Run as Administrator)."
}

# If using UI, prompt for selection and override config
if ($UseUi) {
    $sel = Show-FeatureSyncSelectionForm -DefaultMode $Mode `
        -DefaultSourceComputer $SourceComputer `
        -DefaultTargets $TargetComputers `
        -DefaultExportFolder $ExportFolder `
        -DefaultIncludeManagementTools:$IncludeManagementTools `
        -DefaultRestartIfNeeded:$RestartIfNeeded `
        -DefaultUseExportedJsonFirst:$UseExportedJsonFirst `
        -DefaultUseCredential:$UseCredential `
        -DefaultFeatureSource $FeatureSource `
        -DefaultInstallerScriptName $GeneratedScriptName

    if (-not $sel) {
        Write-Host "Cancelled by user." -ForegroundColor Yellow
        return
    }

    $Mode                    = $sel.Mode
    $SourceComputer          = $sel.SourceComputer
    $TargetComputers         = $sel.TargetComputers
    if ($sel.ExportFolder)   { $ExportFolder = $sel.ExportFolder }
    $IncludeManagementTools  = $sel.IncludeManagementTools
    $RestartIfNeeded         = $sel.RestartIfNeeded
    $UseExportedJsonFirst    = $sel.UseExportedJsonFirst
    $UseCredential           = $sel.UseCredential
    $FeatureSource           = $sel.FeatureSource
    if ($sel.InstallerScriptName) { $GeneratedScriptName = $sel.InstallerScriptName }
}

# Ensure output folder exists (covers Export mode and Install when JSON is used)
try {
    if (-not (Test-Path -LiteralPath $ExportFolder)) {
        New-Item -ItemType Directory -Path $ExportFolder -Force | Out-Null
    }
} catch {
    Write-Error "Failed to create or access export folder '$ExportFolder'. $_"
    return
}

# Prepare logging
$timestamp = Get-Date -Format "yyyyMMdd_HHmmss"
$LogFile   = Join-Path $ExportFolder ("FeatureSync_{0}.log" -f $timestamp)
Start-Transcript -Path $LogFile -Append -ErrorAction SilentlyContinue | Out-Null

# Load ServerManager
try {
    Import-Module ServerManager -ErrorAction Stop
} catch {
    Write-Error "ServerManager module not found. Run on Windows Server (with ServerManager available). $_"
    Stop-Transcript | Out-Null
    return
}

# Helper: Detect Windows Server (best-effort)
function Get-ServerCaption([string]$ComputerName='localhost') {
    try {
        $os = Get-CimInstance -ClassName Win32_OperatingSystem -ComputerName $ComputerName -ErrorAction Stop
        return $os.Caption
    } catch {
        return $null
    }
}

#endregion ====================== SAFETY & PREREQS =============================


#region ========================= FEATURE DISCOVERY ============================

function Get-InstalledServerFeatures {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory=$true)][string]$ComputerName
    )
    try {
        $features = Get-WindowsFeature -ComputerName $ComputerName -ErrorAction Stop
        $installed = $features | Where-Object { $_.Installed -eq $true }
        $installed | Select-Object Name, DisplayName, FeatureType, Installed
    } catch {
        throw "Failed to query features from '$ComputerName'. $_"
    }
}

function Write-FeatureSummary {
    param([array]$FeatureObjects, [string]$ComputerName)
    $total = $FeatureObjects.Count
    $byType = $FeatureObjects | Group-Object FeatureType | Sort-Object Count -Descending

    Write-Host "══════════════════════════════════════════════════════════════════" -ForegroundColor Cyan
    Write-Host ("Server: {0}" -f $ComputerName) -ForegroundColor Cyan
    Write-Host ("Installed features total: {0}" -f $total) -ForegroundColor Cyan
    foreach ($g in $byType) {
        Write-Host (" - {0,-12}: {1,3}" -f $g.Name,$g.Count) -ForegroundColor Cyan
    }
    Write-Host "══════════════════════════════════════════════════════════════════" -ForegroundColor Cyan
}

#endregion ====================== FEATURE DISCOVERY ============================


#region ========================= EXPORT / GENERATE ============================

function Export-FeaturesAndGenerateInstaller {
    [CmdletBinding(SupportsShouldProcess=$true)]
    param(
        [Parameter(Mandatory=$true)][string]$ComputerName,
        [Parameter(Mandatory=$true)][string]$OutFolder,
        [Parameter(Mandatory=$true)][string]$InstallerScriptName,
        [switch]$IncludeManagementToolsDefault
    )

    $caption = Get-ServerCaption -ComputerName $ComputerName
    if ($caption -and ($caption -notmatch 'Windows Server')) {
        Write-Warning "Source '$ComputerName' reports OS: '$caption'. (Expected a Windows Server family OS.)"
    }

    $features = Get-InstalledServerFeatures -ComputerName $ComputerName
    Write-FeatureSummary -FeatureObjects $features -ComputerName $ComputerName

    $csvPath  = Join-Path $OutFolder ("{0}_InstalledFeatures_{1}.csv"  -f $ComputerName,$timestamp)
    $jsonPath = Join-Path $OutFolder ("{0}_InstalledFeatures_{1}.json" -f $ComputerName,$timestamp)

    $features | Sort-Object FeatureType, Name | Export-Csv -NoTypeInformation -Path $csvPath -Encoding UTF8
    $features | ConvertTo-Json -Depth 4 | Out-File -FilePath $jsonPath -Encoding UTF8

    Write-Host "Saved CSV:  $csvPath"  -ForegroundColor Green
    Write-Host "Saved JSON: $jsonPath" -ForegroundColor Green

    # -------- Build installer content safely (NO early expansion) --------
    $installerPath = Join-Path $OutFolder $InstallerScriptName
    $featureNames  = ($features | Select-Object -ExpandProperty Name | Sort-Object -Unique)

    # Template kept in a *single-quoted* here-string so $variables are NOT expanded now.
    $installerTemplate = @'
<#
.SYNOPSIS
    Install a predefined set of Windows Server roles & features on a target server.

.DESCRIPTION
    Generated by the Feature Sync script. Supply -ComputerName, optional -IncludeManagementTools/-Restart,
    and optional -Source for features that require media (e.g., .NET Framework 3.5).
    Works on Windows Server 2016/2019/2022/2025.

.PARAMETER ComputerName
    Target server (NetBIOS, FQDN, or 'localhost').

.PARAMETER IncludeManagementTools
    Adds management tools where available.

.PARAMETER Restart
    If the installation reports that a restart is needed, restart automatically.

.PARAMETER Source
    Optional media source for features that require it (e.g., D:\sources\sxs).

.EXAMPLE
    .\Install-ServerFeatures.ps1 -ComputerName SERVER-TARGET01 -IncludeManagementTools -Restart
#>

#requires -Version 5.1
[CmdletBinding(SupportsShouldProcess=$true)]
param(
    [Parameter(Mandatory=$true)][string]$ComputerName,
    [switch]$IncludeManagementTools,
    [switch]$Restart,
    [string]$Source
)

Import-Module ServerManager -ErrorAction Stop

# Show OS caption (best-effort)
try {
    $os = Get-CimInstance Win32_OperatingSystem -ComputerName $ComputerName -ErrorAction Stop
    Write-Host ("Target OS: {0}" -f $os.Caption) -ForegroundColor Cyan
} catch {
    Write-Warning "Unable to query OS for '$ComputerName'. Proceeding. $_"
}

# Feature set captured during export
$Features = @(
    # <<FEATURES_PLACEHOLDER>>
)

Write-Host ("Installing {0} features on '{1}'..." -f ($Features.Count), $ComputerName) -ForegroundColor Cyan

# Build parameters
$installParams = @{ Name = $Features }
if ($IncludeManagementTools) { $installParams['IncludeManagementTools'] = $true }
if ($Source)                 { $installParams['Source']               = $Source }

try {
    Invoke-Command -ComputerName $ComputerName -ScriptBlock {
        param($p, $doRestart)
        Import-Module ServerManager -ErrorAction Stop
        $result = Install-WindowsFeature @p -ErrorAction Stop
        $result | Format-Table DisplayName, Success, RestartNeeded -AutoSize
        if ($doRestart -and $result.RestartNeeded -eq 'Yes') {
            Write-Host "Restarting '$env:COMPUTERNAME' as requested..." -ForegroundColor Yellow
            Restart-Computer -Force
        }
    } -ArgumentList ($installParams, $Restart.IsPresent)
} catch {
    Write-Error "Installation failed on '$ComputerName'. $_"
    exit 1
}

Write-Host "Done." -ForegroundColor Green
'@

    # Prepare the injection block for the features
    # Each feature appears as:     'Feature-Name',
    $featureLines = ($featureNames | ForEach-Object { "    '{0}'," -f $_ }) -join "`r`n"

    # Replace placeholder
    $finalInstaller = $installerTemplate -replace [regex]::Escape("# <<FEATURES_PLACEHOLDER>>"), $featureLines

    # Write file as UTF-8 (no BOM issues)
    $finalInstaller | Set-Content -Path $installerPath -Encoding UTF8

    Write-Host "Generated installer script: $installerPath" -ForegroundColor Green

    [pscustomobject]@{
        SourceComputer  = $ComputerName
        CsvPath         = $csvPath
        JsonPath        = $jsonPath
        InstallerScript = $installerPath
        FeatureCount    = $featureNames.Count
    }
}

#endregion ====================== EXPORT / GENERATE ============================


#region ========================= INSTALL TO TARGETS ===========================

function Import-FeatureNamesFromJson {
    param([Parameter(Mandatory=$true)][string]$JsonPath)
    try {
        $items = Get-Content -LiteralPath $JsonPath -Raw | ConvertFrom-Json
        ($items | Where-Object { $_.Installed -eq $true } | Select-Object -ExpandProperty Name | Sort-Object -Unique)
    } catch {
        throw "Failed to load feature list from '$JsonPath'. $_"
    }
}

function Ensure-RemotingReady {
    param([string]$ComputerName)
    try {
        Test-WSMan -ComputerName $ComputerName -ErrorAction Stop | Out-Null
        return $true
    } catch {
        Write-Warning "WinRM/PowerShell remoting not ready on '$ComputerName'. Enable-PSRemoting may be required."
        return $false
    }
}

function Install-FeaturesOnTargets {
    [CmdletBinding(SupportsShouldProcess=$true)]
    param(
        [Parameter(Mandatory=$true)][string[]]$Targets,
        [Parameter(Mandatory=$true)][string[]]$FeatureNames,
        [switch]$IncludeManagementTools,
        [switch]$Restart,
        [string]$Source,
        [switch]$PromptForCredential
    )

    $creds = $null
    if ($PromptForCredential) {
        $creds = Get-Credential -Message "Credentials for remote installation"
    }

    foreach ($t in $Targets) {
        Write-Host "`n=== Target: $t ===" -ForegroundColor Magenta
        if (-not (Ensure-RemotingReady -ComputerName $t)) {
            Write-Error "Skipping '$t' because remoting is not available."
            continue
        }

        $sb = {
            param($names, $inclMgmt, $doRestart, $src)
            Import-Module ServerManager -ErrorAction Stop
            $p = @{ Name = $names }
            if ($inclMgmt) { $p['IncludeManagementTools'] = $true }
            if ($src)      { $p['Source'] = $src }

            $result = Install-WindowsFeature @p -ErrorAction Stop
            $result | Select-Object DisplayName, Success, RestartNeeded
            if ($doRestart -and $result.RestartNeeded -eq 'Yes') {
                Write-Host "Restarting '$env:COMPUTERNAME' as requested..." -ForegroundColor Yellow
                Restart-Computer -Force
            }
        }

        try {
            if ($PSCmdlet.ShouldProcess($t, "Install features ($($FeatureNames.Count))")) {
                if ($creds) {
                    Invoke-Command -ComputerName $t -ScriptBlock $sb -Credential $creds -ArgumentList ($FeatureNames, $IncludeManagementTools.IsPresent, $Restart.IsPresent, $Source) -ErrorAction Stop |
                        Format-Table -AutoSize
                } else {
                    Invoke-Command -ComputerName $t -ScriptBlock $sb -ArgumentList ($FeatureNames, $IncludeManagementTools.IsPresent, $Restart.IsPresent, $Source) -ErrorAction Stop |
                        Format-Table -AutoSize
                }
                Write-Host "Completed: $t" -ForegroundColor Green
            }
        } catch {
            Write-Error "Installation failed on '$t'. $_"
        }
    }
}

#endregion ====================== INSTALL TO TARGETS ===========================


#region ========================= MAIN FLOW ===================================

try {
    switch ($Mode.ToLower()) {
        'export' {
            $export = Export-FeaturesAndGenerateInstaller -ComputerName $SourceComputer `
                -OutFolder $ExportFolder `
                -InstallerScriptName $GeneratedScriptName `
                -IncludeManagementToolsDefault:($IncludeManagementTools)

            Write-Host "`nSummary:" -ForegroundColor Cyan
            $export | Format-List
        }

        'install' {
            if ($UseCredential -and -not $Credential) {
                $Credential = Get-Credential -Message "Credentials for remote installation"
            }

            # Locate most recent JSON if $UseExportedJsonFirst
            $jsonFeatures = $null
            if ($UseExportedJsonFirst) {
                $latestJson = Get-ChildItem -Path $ExportFolder -Filter '*_InstalledFeatures_*.json' -ErrorAction SilentlyContinue |
                              Sort-Object LastWriteTime -Descending | Select-Object -First 1
                if ($latestJson) {
                    Write-Host "Using feature list from JSON: $($latestJson.FullName)" -ForegroundColor Cyan
                    $jsonFeatures = Import-FeatureNamesFromJson -JsonPath $latestJson.FullName
                } else {
                    Write-Warning "No exported JSON found in '$ExportFolder'. Will query source live."
                }
            }

            # If no JSON found, query source live
            $featureNames = $null
            if ($jsonFeatures) {
                $featureNames = $jsonFeatures
            } else {
                Write-Host "Querying installed features live from '$SourceComputer'..." -ForegroundColor Cyan
                $featureNames = (Get-InstalledServerFeatures -ComputerName $SourceComputer |
                                 Select-Object -ExpandProperty Name | Sort-Object -Unique)
            }

            if (-not $featureNames -or $featureNames.Count -eq 0) {
                throw "No installed features found to install. Aborting."
            }

            if (-not $TargetComputers -or $TargetComputers.Count -eq 0) {
                throw "No target computers provided for Install mode. Aborting."
            }

            Write-Host "Will install $($featureNames.Count) features on targets: $($TargetComputers -join ', ')" -ForegroundColor Cyan
            Install-FeaturesOnTargets -Targets $TargetComputers `
                -FeatureNames $featureNames `
                -IncludeManagementTools:($IncludeManagementTools) `
                -Restart:($RestartIfNeeded) `
                -Source $FeatureSource `
                -PromptForCredential:($UseCredential)
        }

        default {
            throw "Unknown mode '$Mode'. Use 'Export' or 'Install'."
        }
    }
} catch {
    Write-Error $_
} finally {
    Stop-Transcript | Out-Null
}

#endregion ====================== MAIN FLOW ====================================
```
