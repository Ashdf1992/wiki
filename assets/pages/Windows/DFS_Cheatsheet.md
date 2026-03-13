# DFS Cheatsheet

## Replication Summary for All DFS Members
```Powershell 2
# Get all DFS Replication Groups
$RGGroups = Get-DfsReplicationGroup

Write-Host "============================================================" -ForegroundColor Cyan
Write-Host " DFS Replication Backlog Report" -ForegroundColor Cyan
Write-Host " Generated on: $(Get-Date -Format 'yyyy-MM-dd HH:mm:ss')" -ForegroundColor Cyan
Write-Host "============================================================`n" -ForegroundColor Cyan

foreach ($RG in $RGGroups) {
    Write-Host "`nReplication Group: $($RG.GroupName)" -ForegroundColor Yellow
    $RFolders = Get-DfsReplicatedFolder -GroupName $RG.GroupName

    foreach ($RF in $RFolders) {
        Write-Host "  Folder: $($RF.FolderName)" -ForegroundColor Green
        $RCons = Get-DfsrConnection -GroupName $RG.GroupName

        foreach ($RC in $RCons) {
            try {
                $date = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
                [double]$count = (Get-DfsrBacklog -GroupName $RG.GroupName `
                                  -FolderName $RF.FolderName `
                                  -DestinationComputerName $RC.DestinationComputerName `
                                  -SourceComputerName $RC.SourceComputerName `
                                  -Verbose 4>&1).Message.Split(":")[2]

                $statusColor = if ($count -eq 0) { "Green" } else { "Red" }
                Write-Host ("    [{0}] Source: {1} → Destination: {2} | Backlog: {3}" -f `
                            $date, $RC.SourceComputerName, $RC.DestinationComputerName, $count) -ForegroundColor $statusColor
            }
            catch {
                Write-Host ("    [{0}] Error retrieving backlog for {1} → {2}: {3}" -f `
                            $date, $RC.SourceComputerName, $RC.DestinationComputerName, $_.Exception.Message) -ForegroundColor Red
            }
        }
    }
}

Write-Host "`n============================================================" -ForegroundColor Cyan
Write-Host " Report Complete" -ForegroundColor Cyan
Write-Host "============================================================" -ForegroundColor Cyan
```

## Check Replication Backlog (including file names), and Check Replication State
```Powershell
#requires -Modules DFSR
Add-Type -AssemblyName System.Windows.Forms
Add-Type -AssemblyName System.Drawing

# ============================================================
# Console formatting helpers
# ============================================================
function Write-Line {
    param(
        [string]$Char = "═",
        [int]$Length = 76,
        [ConsoleColor]$Color = [ConsoleColor]::DarkCyan
    )
    Write-Host ($Char * $Length) -ForegroundColor $Color
}

function Write-Banner {
    param(
        [string]$Title,
        [string]$Subtitle = ""
    )

    Write-Host ""
    Write-Line -Color DarkCyan
    Write-Host ("  {0}" -f $Title) -ForegroundColor Cyan
    if ($Subtitle) {
        Write-Host ("  {0}" -f $Subtitle) -ForegroundColor DarkGray
    }
    Write-Line -Color DarkCyan
    Write-Host ""
}

function Write-Section {
    param([string]$Title)

    Write-Host ""
    Write-Host ("▶ {0}" -f $Title) -ForegroundColor Yellow
    Write-Line -Char "─" -Length 76 -Color DarkGray
}

function Write-Status {
    param(
        [ValidateSet("INFO","SUCCESS","WARNING","ERROR")]
        [string]$Type,
        [string]$Message
    )

    switch ($Type) {
        "INFO"    { $icon = "[i]";  $color = "Cyan" }
        "SUCCESS" { $icon = "[+]";  $color = "Green" }
        "WARNING" { $icon = "[!]";  $color = "Yellow" }
        "ERROR"   { $icon = "[x]";  $color = "Red" }
    }

    Write-Host ("{0} {1}" -f $icon, $Message) -ForegroundColor $color
}

function Write-KeyValueBlock {
    param(
        [hashtable]$Data,
        [ConsoleColor]$KeyColor = [ConsoleColor]::Gray,
        [ConsoleColor]$ValueColor = [ConsoleColor]::White
    )

    if (-not $Data -or $Data.Count -eq 0) { return }

    $maxKeyLength = ($Data.Keys | Measure-Object -Maximum Length).Maximum
    foreach ($key in $Data.Keys | Sort-Object) {
        $padding = " " * ($maxKeyLength - $key.Length)
        Write-Host ("  {0}{1} : " -f $key, $padding) -NoNewline -ForegroundColor $KeyColor
        Write-Host $Data[$key] -ForegroundColor $ValueColor
    }
}

function Show-FormattedTable {
    param(
        [Parameter(Mandatory)]
        $InputObject
    )

    $table = $InputObject | Out-String -Width 240
    Write-Host $table -ForegroundColor White
}

# ============================================================
# Backlog helper (true count + displayed items)
# ============================================================
function Get-DfsrBacklogReportData {
    param(
        [Parameter(Mandatory)]
        [string]$SourceComputerName,

        [Parameter(Mandatory)]
        [string]$DestinationComputerName,

        [Parameter(Mandatory)]
        [string]$GroupName,

        [Parameter(Mandatory)]
        [string]$FolderName
    )

    $raw = @(Get-DfsrBacklog `
        -SourceComputerName $SourceComputerName `
        -DestinationComputerName $DestinationComputerName `
        -GroupName $GroupName `
        -FolderName $FolderName `
        -Verbose 4>&1)

    $items = @(
        $raw | Where-Object {
            $_ -and $_.PSObject.Properties.Name -contains 'FileName'
        }
    )

    $actualCount = $items.Count

    $verboseMessages = @(
        $raw | Where-Object {
            $_ -and $_.PSObject.Properties.Name -contains 'Message'
        }
    )

    foreach ($msg in $verboseMessages) {
        $text = [string]$msg.Message

        if ($text -match '(?i)\bcount\b\s*:\s*(\d+)') {
            $actualCount = [int]$matches[1]
            break
        }

        if ($text -match '(\d+)\s*$') {
            $actualCount = [int]$matches[1]
        }
    }

    [pscustomobject]@{
        ActualCount    = $actualCount
        DisplayedCount = $items.Count
        Items          = $items
        VerboseLines   = $verboseMessages
    }
}

# ============================================================
# HTML report helpers
# ============================================================
function Get-DfsrReportDirectory {
    $reportDirectory = Join-Path $env:TEMP "DFSR-HealthReports"
    if (-not (Test-Path $reportDirectory)) {
        New-Item -Path $reportDirectory -ItemType Directory -Force | Out-Null
    }
    return $reportDirectory
}

function Encode-Html {
    param([AllowNull()][string]$Text)
    return [System.Net.WebUtility]::HtmlEncode([string]$Text)
}

function New-DfsrHtmlReport {
    param(
        [Parameter(Mandatory)]
        [pscustomobject]$Selection,

        [Parameter()]
        [array]$Backlog = @(),

        [Parameter()]
        [int]$BacklogActualCount = 0,

        [Parameter()]
        [array]$State = @(),

        [Parameter(Mandatory)]
        [hashtable]$RunSummary
    )

    $reportDirectory = Get-DfsrReportDirectory
    $reportPath = Join-Path $reportDirectory ("DFSR-HealthCheck-{0}.html" -f (Get-Date -Format "yyyyMMdd-HHmmss"))

    $generatedAt = Get-Date -Format "dd MMM yyyy HH:mm:ss"

    $groupName  = Encode-Html $Selection.GroupName
    $folderName = Encode-Html $Selection.FolderName
    $sourceName = Encode-Html $Selection.SourceComputerName
    $destName   = Encode-Html $Selection.DestinationComputerName

    $backlogDisplayedCount = if ($Backlog) { $Backlog.Count } else { 0 }
    $backlogCount = if ($BacklogActualCount -ge 0) { $BacklogActualCount } else { $backlogDisplayedCount }
    $stateCount   = if ($State) { $State.Count } else { 0 }

    $overallHealthText = if ($RunSummary.ContainsKey("Health")) { [string]$RunSummary["Health"] } else { "Unknown" }
    $overallHealthCss  = switch ($overallHealthText) {
        "Healthy" { "healthy" }
        "Warning" { "warning" }
        "Error"   { "error" }
        default   { "info" }
    }

    $backlogRows = if ($Backlog -and $Backlog.Count -gt 0) {
        ($Backlog | ForEach-Object {
            $fileName = Encode-Html $_.FileName
            $fullPath = Encode-Html $_.FullPathName
            $sizeKb   = if ($_.FileSize) { [math]::Round($_.FileSize / 1KB, 2) } else { "" }
            $updated  = if ($_.UpdateTime) { Encode-Html ([string]$_.UpdateTime) } else { "" }

@"
<tr>
    <td>$fileName</td>
    <td>$fullPath</td>
    <td class="num">$sizeKb</td>
    <td>$updated</td>
</tr>
"@
        }) -join "`r`n"
    }
    else {
@"
<tr>
    <td colspan="4" class="empty">No backlog detected.</td>
</tr>
"@
    }

    $stateRows = if ($State -and $State.Count -gt 0) {
        ($State | ForEach-Object {
            $fileName = Encode-Html $_.FileName
            $path     = Encode-Html $_.FilePath
            $status   = Encode-Html $_.UpdateStatus

@"
<tr>
    <td>$fileName</td>
    <td>$path</td>
    <td>$status</td>
</tr>
"@
        }) -join "`r`n"
    }
    else {
@"
<tr>
    <td colspan="3" class="empty">No replication activity detected at this time.</td>
</tr>
"@
    }

    $backlogNotice = ""
    if ($backlogCount -gt $backlogDisplayedCount) {
        $backlogNotice = @"
<div class="notice">
    <strong>Important:</strong> The total backlog count is <code>$backlogCount</code>, but this table is only showing <code>$backlogDisplayedCount</code> item(s). <code>Get-DfsrBacklog</code> displays a maximum of 100 backlog items, while the verbose output provides the true total backlog count.
</div>
"@
    }

    $duration       = Encode-Html $RunSummary["Duration"]
    $completed      = Encode-Html $RunSummary["Completed"]
    $health         = Encode-Html $overallHealthText
    $errors         = Encode-Html $RunSummary["Errors"]
    $warnings       = Encode-Html $RunSummary["Warnings"]
    $displayedItems = Encode-Html $RunSummary["DisplayedItems"]
    $stateCountMeta = Encode-Html $RunSummary["StateCount"]

    $html = @"
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <title>DFS-R Health Check Report</title>
    <style>
        :root {
            --bg: #f4f7fb;
            --panel: #ffffff;
            --panel-soft: #f8fbff;
            --text: #0f172a;
            --muted: #64748b;
            --border: #e2e8f0;
            --shadow: 0 8px 30px rgba(15, 23, 42, 0.08);
            --primary: #2563eb;
            --primary-2: #1d4ed8;
            --healthy: #16a34a;
            --warning: #d97706;
            --error: #dc2626;
            --info: #0891b2;
        }

        * { box-sizing: border-box; }

        body {
            margin: 0;
            background: linear-gradient(180deg, #eef4fb 0%, #f8fafc 160px, #f4f7fb 100%);
            color: var(--text);
            font-family: "Segoe UI", Arial, sans-serif;
        }

        .hero {
            background: linear-gradient(135deg, #0f172a 0%, #1d4ed8 100%);
            color: white;
            padding: 34px 40px 40px 40px;
            box-shadow: var(--shadow);
        }

        .hero h1 {
            margin: 0 0 8px 0;
            font-size: 30px;
            font-weight: 700;
            letter-spacing: -0.02em;
        }

        .hero p {
            margin: 0;
            color: #dbeafe;
            font-size: 14px;
        }

        .wrap {
            max-width: 1380px;
            margin: -24px auto 40px auto;
            padding: 0 24px;
        }

        .grid {
            display: grid;
            grid-template-columns: repeat(auto-fit, minmax(230px, 1fr));
            gap: 18px;
            margin-bottom: 24px;
        }

        .card {
            background: var(--panel);
            border-radius: 18px;
            box-shadow: var(--shadow);
            padding: 20px 22px;
            border: 1px solid rgba(226, 232, 240, 0.9);
        }

        .card .label {
            font-size: 12px;
            font-weight: 600;
            text-transform: uppercase;
            letter-spacing: 0.06em;
            color: var(--muted);
            margin-bottom: 8px;
        }

        .card .value {
            font-size: 22px;
            font-weight: 700;
            word-break: break-word;
            line-height: 1.25;
        }

        .pill {
            display: inline-block;
            padding: 7px 12px;
            border-radius: 999px;
            font-size: 13px;
            font-weight: 700;
            letter-spacing: 0.02em;
        }

        .pill.healthy { background: rgba(22,163,74,0.12); color: var(--healthy); }
        .pill.warning { background: rgba(217,119,6,0.12); color: var(--warning); }
        .pill.error   { background: rgba(220,38,38,0.12); color: var(--error); }
        .pill.info    { background: rgba(8,145,178,0.12); color: var(--info); }

        .section {
            background: var(--panel);
            border-radius: 18px;
            box-shadow: var(--shadow);
            margin-bottom: 24px;
            overflow: hidden;
            border: 1px solid rgba(226, 232, 240, 0.9);
        }

        .section-header {
            padding: 18px 22px;
            border-bottom: 1px solid var(--border);
            background: linear-gradient(180deg, #fbfdff 0%, #f8fbff 100%);
        }

        .section-header h2 {
            margin: 0;
            font-size: 18px;
            letter-spacing: -0.01em;
        }

        .section-header p {
            margin: 6px 0 0 0;
            color: var(--muted);
            font-size: 13px;
        }

        .section-body {
            padding: 20px 22px 24px 22px;
        }

        .notice {
            margin-bottom: 18px;
            background: #fff7ed;
            border: 1px solid #fdba74;
            color: #9a3412;
            border-radius: 14px;
            padding: 14px 16px;
            font-size: 14px;
            line-height: 1.45;
        }

        table {
            width: 100%;
            border-collapse: separate;
            border-spacing: 0;
            font-size: 14px;
            overflow: hidden;
            border: 1px solid var(--border);
            border-radius: 14px;
        }

        thead th {
            background: #f8fafc;
            color: #334155;
            text-transform: uppercase;
            letter-spacing: 0.04em;
            font-size: 12px;
            font-weight: 700;
        }

        th, td {
            padding: 12px 14px;
            text-align: left;
            vertical-align: top;
            border-bottom: 1px solid var(--border);
        }

        tbody tr:last-child td {
            border-bottom: none;
        }

        tbody tr:hover td {
            background: #f8fbff;
        }

        td.num {
            text-align: right;
            white-space: nowrap;
        }

        .empty {
            text-align: center;
            color: var(--muted);
            padding: 18px;
            font-style: italic;
        }

        .two-col {
            display: grid;
            grid-template-columns: 1fr 1fr;
            gap: 24px;
        }

        @media (max-width: 1050px) {
            .two-col {
                grid-template-columns: 1fr;
            }
        }

        .footer {
            text-align: center;
            font-size: 12px;
            color: var(--muted);
            padding: 14px 0 6px 0;
        }

        .meta-list {
            display: grid;
            grid-template-columns: repeat(auto-fit, minmax(210px, 1fr));
            gap: 14px;
        }

        .meta-item {
            background: var(--panel-soft);
            border: 1px solid var(--border);
            border-radius: 14px;
            padding: 14px 16px;
        }

        .meta-item .k {
            font-size: 12px;
            text-transform: uppercase;
            letter-spacing: 0.05em;
            color: var(--muted);
            margin-bottom: 6px;
            font-weight: 700;
        }

        .meta-item .v {
            font-size: 16px;
            font-weight: 700;
            word-break: break-word;
        }

        code {
            font-family: Consolas, "Courier New", monospace;
            background: rgba(15,23,42,0.06);
            padding: 2px 6px;
            border-radius: 8px;
        }
    </style>
</head>
<body>
    <div class="hero">
        <h1>DFS-R Health Check Report</h1>
        <p>Generated on $generatedAt</p>
    </div>

    <div class="wrap">
        <div class="grid">
            <div class="card">
                <div class="label">Replication Group</div>
                <div class="value">$groupName</div>
            </div>
            <div class="card">
                <div class="label">Replicated Folder</div>
                <div class="value">$folderName</div>
            </div>
            <div class="card">
                <div class="label">Source Computer</div>
                <div class="value">$sourceName</div>
            </div>
            <div class="card">
                <div class="label">Destination Computer</div>
                <div class="value">$destName</div>
            </div>
            <div class="card">
                <div class="label">Health Status</div>
                <div class="value"><span class="pill $overallHealthCss">$health</span></div>
            </div>
            <div class="card">
                <div class="label">Backlog Count</div>
                <div class="value">$backlogCount</div>
            </div>
        </div>

        <div class="section">
            <div class="section-header">
                <h2>Run Summary</h2>
                <p>High-level execution information for this DFS-R check.</p>
            </div>
            <div class="section-body">
                <div class="meta-list">
                    <div class="meta-item">
                        <div class="k">Completed</div>
                        <div class="v">$completed</div>
                    </div>
                    <div class="meta-item">
                        <div class="k">Duration</div>
                        <div class="v">$duration</div>
                    </div>
                    <div class="meta-item">
                        <div class="k">Warnings</div>
                        <div class="v">$warnings</div>
                    </div>
                    <div class="meta-item">
                        <div class="k">Errors</div>
                        <div class="v">$errors</div>
                    </div>
                    <div class="meta-item">
                        <div class="k">Active Replication Entries</div>
                        <div class="v">$stateCountMeta</div>
                    </div>
                    <div class="meta-item">
                        <div class="k">Displayed Backlog Items</div>
                        <div class="v">$displayedItems</div>
                    </div>
                </div>
            </div>
        </div>

        <div class="two-col">
            <div class="section">
                <div class="section-header">
                    <h2>Backlog</h2>
                    <p>Files currently reported as pending replication.</p>
                </div>
                <div class="section-body">
                    $backlogNotice
                    <table>
                        <thead>
                            <tr>
                                <th>File Name</th>
                                <th>Full Path</th>
                                <th>Size (KB)</th>
                                <th>Last Updated</th>
                            </tr>
                        </thead>
                        <tbody>
                            $backlogRows
                        </tbody>
                    </table>
                </div>
            </div>

            <div class="section">
                <div class="section-header">
                    <h2>Replication State</h2>
                    <p>Current DFS-R replication activity reported at run time.</p>
                </div>
                <div class="section-body">
                    <table>
                        <thead>
                            <tr>
                                <th>File Name</th>
                                <th>Path</th>
                                <th>Status</th>
                            </tr>
                        </thead>
                        <tbody>
                            $stateRows
                        </tbody>
                    </table>
                </div>
            </div>
        </div>

        <div class="footer">
            DFS-R Health Check • PowerShell generated report
        </div>
    </div>
</body>
</html>
"@

    $html | Out-File -FilePath $reportPath -Encoding UTF8
    return $reportPath
}

# ============================================================
# DFS-R selection form
# ============================================================
function Show-DfsrSelectionForm {
    [CmdletBinding()]
    param()

    Import-Module DFSR -ErrorAction Stop

    function Set-ComboData {
        param(
            [System.Windows.Forms.ComboBox]$ComboBox,
            [array]$Items,
            [string]$DisplayMember,
            [string]$ValueMember
        )

        $ComboBox.DataSource = $null
        $ComboBox.Items.Clear()

        $list = New-Object System.Collections.ArrayList
        foreach ($item in $Items) {
            [void]$list.Add($item)
        }

        $ComboBox.DisplayMember = $DisplayMember
        $ComboBox.ValueMember   = $ValueMember
        $ComboBox.DataSource    = $list
    }

    $groups = @(Get-DfsReplicationGroup | Sort-Object GroupName)

    if (-not $groups -or $groups.Count -eq 0) {
        throw "No DFS Replication Groups were found."
    }

    # Form
    $form = New-Object System.Windows.Forms.Form
    $form.Text = "DFS-R Backlog Checker"
    $form.Size = New-Object System.Drawing.Size(580, 355)
    $form.StartPosition = "CenterScreen"
    $form.FormBorderStyle = "FixedDialog"
    $form.MaximizeBox = $false
    $form.MinimizeBox = $false
    $form.TopMost = $true
    $form.BackColor = [System.Drawing.Color]::FromArgb(245, 247, 250)
    $form.Font = New-Object System.Drawing.Font("Segoe UI", 9)

    # Header panel
    $headerPanel = New-Object System.Windows.Forms.Panel
    $headerPanel.Location = New-Object System.Drawing.Point(0, 0)
    $headerPanel.Size = New-Object System.Drawing.Size(580, 60)
    $headerPanel.BackColor = [System.Drawing.Color]::FromArgb(0, 120, 212)

    $lblHeader = New-Object System.Windows.Forms.Label
    $lblHeader.Text = "DFS Replication Health Check"
    $lblHeader.Location = New-Object System.Drawing.Point(18, 10)
    $lblHeader.Size = New-Object System.Drawing.Size(380, 22)
    $lblHeader.ForeColor = [System.Drawing.Color]::White
    $lblHeader.Font = New-Object System.Drawing.Font("Segoe UI Semibold", 12)

    $lblHeaderSub = New-Object System.Windows.Forms.Label
    $lblHeaderSub.Text = "Select the replication group, folder, source, and destination members."
    $lblHeaderSub.Location = New-Object System.Drawing.Point(18, 33)
    $lblHeaderSub.Size = New-Object System.Drawing.Size(500, 18)
    $lblHeaderSub.ForeColor = [System.Drawing.Color]::FromArgb(230, 240, 255)
    $lblHeaderSub.Font = New-Object System.Drawing.Font("Segoe UI", 8.5)

    $headerPanel.Controls.AddRange(@($lblHeader, $lblHeaderSub))

    # Group box
    $grpSelection = New-Object System.Windows.Forms.GroupBox
    $grpSelection.Text = "Replication Selection"
    $grpSelection.Location = New-Object System.Drawing.Point(18, 72)
    $grpSelection.Size = New-Object System.Drawing.Size(530, 185)
    $grpSelection.Font = New-Object System.Drawing.Font("Segoe UI Semibold", 9)

    # Labels
    $lblGroup = New-Object System.Windows.Forms.Label
    $lblGroup.Text = "Replication Group"
    $lblGroup.Location = New-Object System.Drawing.Point(18, 32)
    $lblGroup.Size = New-Object System.Drawing.Size(130, 20)

    $lblFolder = New-Object System.Windows.Forms.Label
    $lblFolder.Text = "Replicated Folder"
    $lblFolder.Location = New-Object System.Drawing.Point(18, 67)
    $lblFolder.Size = New-Object System.Drawing.Size(130, 20)

    $lblSource = New-Object System.Windows.Forms.Label
    $lblSource.Text = "Source Computer"
    $lblSource.Location = New-Object System.Drawing.Point(18, 102)
    $lblSource.Size = New-Object System.Drawing.Size(130, 20)

    $lblDestination = New-Object System.Windows.Forms.Label
    $lblDestination.Text = "Destination Computer"
    $lblDestination.Location = New-Object System.Drawing.Point(18, 137)
    $lblDestination.Size = New-Object System.Drawing.Size(130, 20)

    # ComboBoxes
    $cmbGroup = New-Object System.Windows.Forms.ComboBox
    $cmbGroup.Location = New-Object System.Drawing.Point(160, 28)
    $cmbGroup.Size = New-Object System.Drawing.Size(340, 26)
    $cmbGroup.DropDownStyle = [System.Windows.Forms.ComboBoxStyle]::DropDownList

    $cmbFolder = New-Object System.Windows.Forms.ComboBox
    $cmbFolder.Location = New-Object System.Drawing.Point(160, 63)
    $cmbFolder.Size = New-Object System.Drawing.Size(340, 26)
    $cmbFolder.DropDownStyle = [System.Windows.Forms.ComboBoxStyle]::DropDownList

    $cmbSource = New-Object System.Windows.Forms.ComboBox
    $cmbSource.Location = New-Object System.Drawing.Point(160, 98)
    $cmbSource.Size = New-Object System.Drawing.Size(340, 26)
    $cmbSource.DropDownStyle = [System.Windows.Forms.ComboBoxStyle]::DropDownList

    $cmbDestination = New-Object System.Windows.Forms.ComboBox
    $cmbDestination.Location = New-Object System.Drawing.Point(160, 133)
    $cmbDestination.Size = New-Object System.Drawing.Size(340, 26)
    $cmbDestination.DropDownStyle = [System.Windows.Forms.ComboBoxStyle]::DropDownList

    $grpSelection.Controls.AddRange(@(
        $lblGroup, $lblFolder, $lblSource, $lblDestination,
        $cmbGroup, $cmbFolder, $cmbSource, $cmbDestination
    ))

    # Status label
    $lblStatus = New-Object System.Windows.Forms.Label
    $lblStatus.Location = New-Object System.Drawing.Point(20, 268)
    $lblStatus.Size = New-Object System.Drawing.Size(400, 22)
    $lblStatus.ForeColor = [System.Drawing.Color]::FromArgb(70, 70, 70)
    $lblStatus.Text = "Select a replication group to load folders and members."

    # Buttons
    $btnOK = New-Object System.Windows.Forms.Button
    $btnOK.Text = "Run Check"
    $btnOK.Location = New-Object System.Drawing.Point(356, 292)
    $btnOK.Size = New-Object System.Drawing.Size(90, 30)
    $btnOK.BackColor = [System.Drawing.Color]::FromArgb(0, 120, 212)
    $btnOK.ForeColor = [System.Drawing.Color]::White
    $btnOK.FlatStyle = "Flat"
    $btnOK.FlatAppearance.BorderSize = 0

    $btnCancel = New-Object System.Windows.Forms.Button
    $btnCancel.Text = "Cancel"
    $btnCancel.Location = New-Object System.Drawing.Point(456, 292)
    $btnCancel.Size = New-Object System.Drawing.Size(90, 30)
    $btnCancel.FlatStyle = "Flat"

    $form.AcceptButton = $btnOK
    $form.CancelButton = $btnCancel

    $form.Controls.AddRange(@(
        $headerPanel, $grpSelection, $lblStatus, $btnOK, $btnCancel
    ))

    # Load groups
    Set-ComboData -ComboBox $cmbGroup -Items $groups -DisplayMember "GroupName" -ValueMember "GroupName"

    $loadGroupData = {
        try {
            $selectedGroup = $cmbGroup.SelectedValue
            if (-not $selectedGroup) { return }

            $lblStatus.Text = "Loading folders and members for group '$selectedGroup'..."
            $form.Refresh()

            $folders = @(Get-DfsReplicatedFolder -GroupName $selectedGroup | Sort-Object FolderName)
            $members = @(Get-DfsrMember -GroupName $selectedGroup | Sort-Object ComputerName)

            if (-not $folders -or $folders.Count -eq 0) {
                throw "No replicated folders found in group '$selectedGroup'."
            }

            if (-not $members -or $members.Count -eq 0) {
                throw "No member computers found in group '$selectedGroup'."
            }

            Set-ComboData -ComboBox $cmbFolder -Items $folders -DisplayMember "FolderName" -ValueMember "FolderName"
            Set-ComboData -ComboBox $cmbSource -Items $members -DisplayMember "ComputerName" -ValueMember "ComputerName"

            $membersCopy = $members | ForEach-Object {
                [pscustomobject]@{
                    ComputerName = $_.ComputerName
                }
            }

            Set-ComboData -ComboBox $cmbDestination -Items $membersCopy -DisplayMember "ComputerName" -ValueMember "ComputerName"

            if ($cmbSource.Items.Count -gt 0) { $cmbSource.SelectedIndex = 0 }

            if ($cmbDestination.Items.Count -gt 1) {
                $cmbDestination.SelectedIndex = 1
            }
            elseif ($cmbDestination.Items.Count -gt 0) {
                $cmbDestination.SelectedIndex = 0
            }

            $lblStatus.Text = "Ready."
        }
        catch {
            [System.Windows.Forms.MessageBox]::Show(
                "Failed to load DFS-R data.`r`n`r`n$($_.Exception.Message)",
                "DFS-R Selection Error",
                [System.Windows.Forms.MessageBoxButtons]::OK,
                [System.Windows.Forms.MessageBoxIcon]::Error
            ) | Out-Null
            $lblStatus.Text = "Failed to load DFS-R data."
        }
    }

    $cmbGroup.add_SelectedIndexChanged($loadGroupData)

    $btnOK.Add_Click({
        if (-not $cmbGroup.SelectedValue -or -not $cmbFolder.SelectedValue -or -not $cmbSource.SelectedValue -or -not $cmbDestination.SelectedValue) {
            [System.Windows.Forms.MessageBox]::Show(
                "Please select all values before continuing.",
                "Validation",
                [System.Windows.Forms.MessageBoxButtons]::OK,
                [System.Windows.Forms.MessageBoxIcon]::Warning
            ) | Out-Null
            return
        }

        if ($cmbSource.SelectedValue -eq $cmbDestination.SelectedValue) {
            [System.Windows.Forms.MessageBox]::Show(
                "Source and destination computers must be different.",
                "Validation",
                [System.Windows.Forms.MessageBoxButtons]::OK,
                [System.Windows.Forms.MessageBoxIcon]::Warning
            ) | Out-Null
            return
        }

        $script:DfsrSelection = [pscustomobject]@{
            GroupName               = [string]$cmbGroup.SelectedValue
            FolderName              = [string]$cmbFolder.SelectedValue
            SourceComputerName      = [string]$cmbSource.SelectedValue
            DestinationComputerName = [string]$cmbDestination.SelectedValue
        }

        $form.DialogResult = [System.Windows.Forms.DialogResult]::OK
        $form.Close()
    })

    $btnCancel.Add_Click({
        $form.DialogResult = [System.Windows.Forms.DialogResult]::Cancel
        $form.Close()
    })

    $form.Add_Shown({
        & $loadGroupData
        $form.Activate()
    })

    $result = $form.ShowDialog()

    if ($result -eq [System.Windows.Forms.DialogResult]::OK) {
        return $script:DfsrSelection
    }

    return $null
}

# ============================================================
# Main
# ============================================================
$sw = [System.Diagnostics.Stopwatch]::StartNew()

$Host.UI.RawUI.WindowTitle = "DFS-R Backlog Checker"
Clear-Host

$selection = Show-DfsrSelectionForm

if (-not $selection) {
    Write-Status -Type WARNING -Message "Operation cancelled by user."
    return
}

$sourceComputer      = $selection.SourceComputerName
$destinationComputer = $selection.DestinationComputerName
$replicationGroup    = $selection.GroupName
$replicatedFolder    = $selection.FolderName

$backlog = @()
$state   = @()
$warningCount = 0
$errorCount   = 0
$backlogActualCount = 0
$backlogDisplayedCount = 0

Write-Banner -Title "DFS Replication Health Check" -Subtitle ("Started: {0}" -f (Get-Date -Format "dd MMM yyyy HH:mm:ss"))

Write-Section "Selected Configuration"
Write-KeyValueBlock -Data @{
    "Replication Group"    = $replicationGroup
    "Replicated Folder"    = $replicatedFolder
    "Source Computer"      = $sourceComputer
    "Destination Computer" = $destinationComputer
}

# ============================================================
# Backlog check
# ============================================================
Write-Section "Backlog Check"
Write-Status -Type INFO -Message "Checking DFS Replication backlog..."

try {
    $backlogResult = Get-DfsrBacklogReportData `
        -SourceComputerName $sourceComputer `
        -DestinationComputerName $destinationComputer `
        -GroupName $replicationGroup `
        -FolderName $replicatedFolder

    $backlog = @($backlogResult.Items)
    $backlogActualCount = [int]$backlogResult.ActualCount
    $backlogDisplayedCount = [int]$backlogResult.DisplayedCount

    if ($backlogActualCount -eq 0) {
        Write-Status -Type SUCCESS -Message "No backlog detected. Replication is up to date."
    }
    else {
        $warningCount++
        Write-Status -Type WARNING -Message ("Backlog detected: {0} file(s) pending replication." -f $backlogActualCount)

        if ($backlogActualCount -gt $backlogDisplayedCount) {
            Write-Status -Type INFO -Message ("Displaying the first {0} backlog item(s) returned by Get-DfsrBacklog." -f $backlogDisplayedCount)
        }

        Write-Host ""

        $backlogTable = $backlog |
            Select-Object `
                @{Name="File Name";    Expression={$_.FileName}},
                @{Name="Size (KB)";    Expression={[math]::Round($_.FileSize / 1KB, 2)}},
                @{Name="Last Updated"; Expression={$_.UpdateTime}}

        Show-FormattedTable -InputObject ($backlogTable | Format-Table -AutoSize)
    }
}
catch {
    $errorCount++
    $backlog = @()
    $backlogActualCount = 0
    $backlogDisplayedCount = 0
    Write-Status -Type ERROR -Message ("Error retrieving backlog: {0}" -f $_.Exception.Message)
}

# ============================================================
# Replication state
# ============================================================
Write-Section "Replication State"
Write-Status -Type INFO -Message "Checking current DFS-R replication activity..."

try {
    $state = @(Get-DfsrState)

    if ($state -and $state.Count -gt 0) {
        Write-Status -Type INFO -Message "Current replication activity detected."
        Write-Host ""

        $stateTable = $state |
            Select-Object `
                @{Name="File Name"; Expression={$_.FileName}},
                @{Name="Path";      Expression={$_.FilePath}},
                @{Name="Status";    Expression={$_.UpdateStatus}}

        Show-FormattedTable -InputObject ($stateTable | Format-Table -AutoSize)
    }
    else {
        Write-Status -Type SUCCESS -Message "No replication activity detected at this time."
    }
}
catch {
    $errorCount++
    Write-Status -Type ERROR -Message ("Error retrieving replication state: {0}" -f $_.Exception.Message)
}

# ============================================================
# Completion
# ============================================================
$sw.Stop()

$health = if ($errorCount -gt 0) {
    "Error"
}
elseif ($warningCount -gt 0) {
    "Warning"
}
else {
    "Healthy"
}

$runSummary = @{
    "Completed"      = (Get-Date -Format "dd MMM yyyy HH:mm:ss")
    "Duration"       = ("{0:N2} seconds" -f $sw.Elapsed.TotalSeconds)
    "Group"          = $replicationGroup
    "Folder"         = $replicatedFolder
    "Source"         = $sourceComputer
    "Destination"    = $destinationComputer
    "Warnings"       = $warningCount
    "Errors"         = $errorCount
    "Health"         = $health
    "BacklogCount"   = $backlogActualCount
    "DisplayedItems" = $backlogDisplayedCount
    "StateCount"     = if ($state) { $state.Count } else { 0 }
}

Write-Section "Summary"
Write-KeyValueBlock -Data @{
    "Completed"       = $runSummary["Completed"]
    "Duration"        = $runSummary["Duration"]
    "Group"           = $replicationGroup
    "Folder"          = $replicatedFolder
    "Source"          = $sourceComputer
    "Destination"     = $destinationComputer
    "Backlog Count"   = $runSummary["BacklogCount"]
    "Displayed Items" = $runSummary["DisplayedItems"]
    "State Entries"   = $runSummary["StateCount"]
}

Write-Host ""
Write-Line -Color DarkCyan
Write-Host "  Check Complete" -ForegroundColor Cyan
Write-Line -Color DarkCyan
Write-Host ""

# ============================================================
# HTML report generation
# ============================================================
try {
    $reportPath = New-DfsrHtmlReport `
        -Selection $selection `
        -Backlog $backlog `
        -BacklogActualCount $backlogActualCount `
        -State $state `
        -RunSummary $runSummary

    Write-Section "HTML Report"
    Write-Status -Type SUCCESS -Message "HTML report generated successfully."
    Write-KeyValueBlock -Data @{
        "Report Path" = $reportPath
    }

    Start-Process $reportPath
}
catch {
    Write-Section "HTML Report"
    Write-Status -Type ERROR -Message ("Failed to generate HTML report: {0}" -f $_.Exception.Message)
}
```

## Pre-seed Data
Running from ASH-WEB-01, copying data to ASH-WEB-02, excluding DFSR Private
``` Powershell
# Define source and destination paths
$Source      = "D:\IIS-Content\"
$Destination = "\\ASH-WEB-02\d$\IIS-Content\"
 
# Robocopy options:
# /MIR      - Mirror source to destination (includes subdirectories)
# /COPYALL  - Copy all file info (data, attributes, timestamps, security, owner, audit)
# /R:3      - Retry 3 times on failure
# /W:5      - Wait 5 seconds between retries
# /XD       - Exclude directory (DfsrPrivate)
# /LOG      - Optional: log output to a file
# /MT:16    - Multi-threaded copy (16 threads for speed)
 
Robocopy $Source $Destination /TEE /V /MIR /COPYALL /R:3 /W:5 /XD "DfsrPrivate" /MT:16 /LOG:"C:\Robocopy_Preseed.log"
```

## Clearing the DFS Replication Database

Occasionally, you may need to reset the DFS Replication (DFSR) database on a volume to resolve replication issues. Follow these steps carefully:

IMPORTANT NOTES:
<br>
-Ensure you have a valid backup of your data before proceeding.
<br>
-This process will force DFSR to rebuild its database, which may temporarily increase replication traffic.
<br>
-Run all commands in an elevated PowerShell session (Administrator).

Steps:
<br>
-Stop the DFS Replication Service
This prevents DFSR from accessing the database during the reset.
``` Powershell
Stop-Service DFSR
```
<br>

-Navigate to the root of the drive where the DFSR database resides (e.g., C:).
``` Powershell
Set-Location C:\
```
<br>

-Create an empty folder to use as a source for the mirror operation.
``` Powershell
New-Item -ItemType Directory -Path "C:\empty" -Force
```
<br>

-Mirror the empty folder into the DFSR database folder using robocopy.
This effectively clears the DFSR database contents.
```Powershell
robocopy "C:\empty" "C:\System Volume Information\DFSR" /MIR
```
/MIR = Mirror source to destination (delete extra files in destination).

<br>

-Restart the DFS Replication Service
This will trigger DFSR to rebuild its database.
``` Powershell
Start-Service DFSR
```

<br>

### ✅ Script (All Steps Combined)
IMPORTANT NOTES:
<br>
-Edit the following values accordingly:
<br>
$emptyPath = "C:\empty"
<br>
$dbPath = "C:\System Volume Information\DFSR"

``` Powershell
# Clear DFSR Database on C: Drive
# IMPORTANT: Run as Administrator and ensure backups exist before proceeding.

Write-Host "============================================================" -ForegroundColor Cyan
Write-Host " Clearing DFS Replication Database (C:) " -ForegroundColor Cyan
Write-Host "============================================================`n" -ForegroundColor Cyan

try {
    Write-Host "Stopping DFS Replication Service..." -ForegroundColor Yellow
    Stop-Service DFSR -ErrorAction Stop
    Write-Host "✅ DFS Replication Service stopped." -ForegroundColor Green

    Write-Host "`nPreparing empty folder for mirror operation..." -ForegroundColor Yellow
    $emptyPath = "C:\empty"
    if (-not (Test-Path $emptyPath)) {
        New-Item -ItemType Directory -Path $emptyPath -Force | Out-Null
    }
    Write-Host "✅ Empty folder ready at $emptyPath." -ForegroundColor Green
    Write-Host "`nClearing DFSR database using robocopy..." -ForegroundColor Yellow
    $dbPath = "C:\System Volume Information\DFSR"
    robocopy $emptyPath $dbPath /MIR | Out-Null
    Write-Host "✅ DFSR database cleared." -ForegroundColor Green

    Write-Host "`nRestarting DFS Replication Service..." -ForegroundColor Yellow
    Start-Service DFSR -ErrorAction Stop
    Write-Host "✅ DFS Replication Service restarted successfully." -ForegroundColor Green
}
catch {
    Write-Host "`n❌ An error occurred: $($_.Exception.Message)" -ForegroundColor Red
}

Write-Host "`n============================================================" -ForegroundColor Cyan
Write-Host " Operation Complete" -ForegroundColor Cyan
Write-Host "============================================================" -ForegroundColor Cyan
```
