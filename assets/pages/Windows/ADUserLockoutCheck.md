# Checks for AD User Lockouts in the past 24 hours 🔄🖥️

```Powershell
# Requires: Windows PowerShell 5.x, RSAT ActiveDirectory, Out-GridView capability
# Purpose : Interactive ISE-only tool to review AD account lockouts and related security events (4740)
# Author  : Your friendly Enterprise Windows Engineer toolset 🛠️

# ---- Pre-flight checks -------------------------------------------------------
if (-not $psISE) {
    Write-Warning "This tool is intended to run in Windows PowerShell ISE. Please open PowerShell ISE and run it there."
    return
}

# Ensure AD module is available
try {
    if (-not (Get-Module -ListAvailable -Name ActiveDirectory)) {
        throw "The 'ActiveDirectory' module is not available. Please install RSAT: Active Directory tools."
    }
    Import-Module ActiveDirectory -ErrorAction Stop
} catch {
    Write-Error "❗ $($_.Exception.Message)"
    return
}

# Improve emoji support in console (best effort on WinPS5)
try { [Console]::OutputEncoding = [System.Text.Encoding]::UTF8 } catch {}

# ---- Helper: Safe date/time formatting --------------------------------------
function New-FriendlyDateString {
    param([datetime]$dt)
    if ($null -eq $dt) { return '' }
    return $dt.ToString('yyyy-MM-dd HH:mm:ss')
}

# ---- UI: Prompt for inputs (Windows Forms) -----------------------------------
Add-Type -AssemblyName System.Windows.Forms | Out-Null
Add-Type -AssemblyName System.Drawing        | Out-Null

$dcList = @()
try {
    $dcList = (Get-ADDomainController -Filter * | Sort-Object HostName).HostName
} catch {
    # Fallback: leave blank; user can type manually
}

$form                 = New-Object System.Windows.Forms.Form
$form.Text            = "Account Lockout Review (AD / Event 4740)"
$form.Size            = New-Object System.Drawing.Size(560,360)
$form.StartPosition   = "CenterScreen"
$form.TopMost         = $true

# Labels
$lblDc               = New-Object System.Windows.Forms.Label
$lblDc.Text          = "Domain Controller:"
$lblDc.Location      = New-Object System.Drawing.Point(20,25)
$lblDc.AutoSize      = $true

$lblHours            = New-Object System.Windows.Forms.Label
$lblHours.Text       = "Look-back (hours):"
$lblHours.Location   = New-Object System.Drawing.Point(20,75)
$lblHours.AutoSize   = $true

$lblUser             = New-Object System.Windows.Forms.Label
$lblUser.Text        = "Filter by user (sAMAccountName) - optional:"
$lblUser.Location    = New-Object System.Drawing.Point(20,125)
$lblUser.AutoSize    = $true

# Controls
$cmbDc                   = New-Object System.Windows.Forms.ComboBox
$cmbDc.Location          = New-Object System.Drawing.Point(20,45)
$cmbDc.Size              = New-Object System.Drawing.Size(500,26)
$cmbDc.DropDownStyle     = 'DropDown'
$cmbDc.AutoCompleteMode  = 'SuggestAppend'
$cmbDc.AutoCompleteSource= 'ListItems'
[void]$cmbDc.Items.AddRange($dcList)

$numHours                = New-Object System.Windows.Forms.NumericUpDown
$numHours.Location       = New-Object System.Drawing.Point(20,95)
$numHours.Size           = New-Object System.Drawing.Size(120,26)
$numHours.Minimum        = 1
$numHours.Maximum        = 168
$numHours.Value          = 24

$txtUser                 = New-Object System.Windows.Forms.TextBox
$txtUser.Location        = New-Object System.Drawing.Point(20,145)
$txtUser.Size            = New-Object System.Drawing.Size(500,26)

$chkHtml                 = New-Object System.Windows.Forms.CheckBox
$chkHtml.Text            = "Export HTML report to Desktop"
$chkHtml.Location        = New-Object System.Drawing.Point(20,185)
$chkHtml.AutoSize        = $true

# Buttons
$btnOK                   = New-Object System.Windows.Forms.Button
$btnOK.Text              = "Run"
$btnOK.Location          = New-Object System.Drawing.Point(330,240)
$btnOK.Size              = New-Object System.Drawing.Size(90,32)

$btnCancel               = New-Object System.Windows.Forms.Button
$btnCancel.Text          = "Cancel"
$btnCancel.Location      = New-Object System.Drawing.Point(430,240)
$btnCancel.Size          = New-Object System.Drawing.Size(90,32)

$form.Controls.AddRange(@($lblDc,$cmbDc,$lblHours,$numHours,$lblUser,$txtUser,$chkHtml,$btnOK,$btnCancel))

$selected = $null
$btnOK.Add_Click({
    if ([string]::IsNullOrWhiteSpace($cmbDc.Text)) {
        [System.Windows.Forms.MessageBox]::Show(
            "Please specify a Domain Controller hostname (FQDN recommended).",
            "Input required",
            [System.Windows.Forms.MessageBoxButtons]::OK,
            [System.Windows.Forms.MessageBoxIcon]::Warning
        ) | Out-Null
        return
    }
    $selected = @{
        DC       = $cmbDc.Text.Trim()
        Hours    = [int]$numHours.Value
        Username = $txtUser.Text.Trim()
        Export   = $chkHtml.Checked
    }
    $form.DialogResult = [System.Windows.Forms.DialogResult]::OK
    $form.Close()
})
$btnCancel.Add_Click({
    $form.DialogResult = [System.Windows.Forms.DialogResult]::Cancel
    $form.Close()
})

[void]$form.ShowDialog()
if ($form.DialogResult -ne [System.Windows.Forms.DialogResult]::OK) {
    Write-Host "🔁 Operation cancelled by user."
    return
}

$dc       = $selected.DC
$hours    = $selected.Hours
$username = $selected.Username
$export   = $selected.Export

$startTime = (Get-Date).AddHours(-1 * $hours)

Write-Host "🔎 Gathering account lockout information from DC: $dc (last $hours hours) ..." -ForegroundColor Cyan

# ---- Data collection ---------------------------------------------------------
# 1) Currently locked-out users (replicated attribute)
try {
    $lockedNow = Search-ADAccount -LockedOut -UsersOnly -ErrorAction Stop |
        Get-ADUser -Properties DisplayName, SamAccountName, LockedOut, LastBadPasswordAttempt, BadLogonCount, whenChanged
} catch {
    Write-Warning "Could not query current locked-out accounts: $($_.Exception.Message)"
    $lockedNow = @()
}

# Optionally filter by username
if ($username) {
    $lockedNow = $lockedNow | Where-Object { $_.SamAccountName -ieq $username }
}

# 2) Security events: 4740 (User account locked out) on specified DC
$events = @()
try {
    $events = Get-WinEvent -ComputerName $dc -FilterHashtable @{
        LogName   = 'Security'
        Id        = 4740
        StartTime = $startTime
    } -ErrorAction Stop
} catch {
    Write-Warning ("Could not read Security log (Event 4740) from ${dc}: {0}" -f $($_.Exception.Message))
}

# Parse event message for TargetUserName and CallerComputerName
function Parse-4740 {
    param([System.Diagnostics.Eventing.Reader.EventRecord]$e)
    $msg = $e.FormatDescription()
    $user = $null; $caller = $null
    if ($msg) {
        if ($msg -match "Target User Name:\s*(.+)")     { $user   = ($matches[1] -split "`r?`n")[0].Trim() }
        if ($msg -match "Caller Computer Name:\s*(.+)")  { $caller = ($matches[1] -split "`r?`n")[0].Trim() }
    }
    [pscustomobject]@{
        TimeCreated        = $e.TimeCreated
        TargetUserName     = $user
        CallerComputerName = $caller
        ProviderName       = $e.ProviderName
        Id                 = $e.Id
        RecordId           = $e.RecordId
        MachineName        = $e.MachineName
    }
}

$parsedEvents = $events | ForEach-Object { Parse-4740 $_ }

if ($username) {
    $parsedEvents = $parsedEvents | Where-Object { $_.TargetUserName -and ($_.TargetUserName -ieq $username) }
}

# ---- Correlate & Shape results ----------------------------------------------
# Build a lookup for AD user display attributes
$displayLookup = @{}

if ($username) {
    try {
        $u = Get-ADUser -Identity $username -Properties DisplayName,SamAccountName,LastBadPasswordAttempt,BadLogonCount,LockedOut
        if ($u) { $displayLookup[$u.SamAccountName.ToLower()] = $u }
    } catch {}
} else {
    $names = ($parsedEvents | Where-Object { $_.TargetUserName }) |
             Select-Object -ExpandProperty TargetUserName -Unique
    foreach ($n in $names) {
        try {
            $u = Get-ADUser -LDAPFilter "(sAMAccountName=$n)" -Properties DisplayName,SamAccountName,LastBadPasswordAttempt,BadLogonCount,LockedOut
            if ($u) { $displayLookup[$u.SamAccountName.ToLower()] = $u }
        } catch {}
    }
}

# Compose a unified record set
$records = New-Object System.Collections.Generic.List[object]

foreach ($e in ($parsedEvents | Sort-Object TimeCreated)) {
    $sam = ''
    if ($null -ne $e.TargetUserName) { $sam = $e.TargetUserName.ToString() }

    $lu  = $null
    if (-not [string]::IsNullOrEmpty($sam)) {
        $lu = $displayLookup[$sam.ToLower()]
    }

    $whenTxt = ''
    if ($e.TimeCreated) { $whenTxt = New-FriendlyDateString $e.TimeCreated }

    $records.Add([pscustomobject]@{
        When                  = $whenTxt
        User                  = $sam
        DisplayName           = if ($lu) { $lu.DisplayName } else { '' }
        CurrentlyLockedOut    = if ($lu) { [bool]$lu.LockedOut } else { $null }
        LastBadPassword       = if ($lu) { New-FriendlyDateString $lu.LastBadPasswordAttempt } else { '' }
        BadLogonCount         = if ($lu) { $lu.BadLogonCount } else { $null }
        OriginatingComputer   = $e.CallerComputerName
        DomainController      = $dc
        SecurityEventRecordId = $e.RecordId
    }) | Out-Null
}

# Add a row for any users currently locked out who didn't show in the window
$existingUsers = $records | Select-Object -ExpandProperty User -Unique
$orphans = $lockedNow | Where-Object { $_.SamAccountName -and ($existingUsers -notcontains $_.SamAccountName) }
foreach ($u in $orphans) {
    $records.Add([pscustomobject]@{
        When                  = ''
        User                  = $u.SamAccountName
        DisplayName           = $u.DisplayName
        CurrentlyLockedOut    = [bool]$u.LockedOut
        LastBadPassword       = New-FriendlyDateString $u.LastBadPasswordAttempt
        BadLogonCount         = $u.BadLogonCount
        OriginatingComputer   = ''
        DomainController      = $dc
        SecurityEventRecordId = ''
    }) | Out-Null
}

# ---- Output: Grid and Ticket-Ready Summary ----------------------------------
if ($records.Count -gt 0) {
    $title = "AD Account Lockouts (Last $hours h) on $dc"
    $records |
        Select-Object When,User,DisplayName,CurrentlyLockedOut,LastBadPassword,BadLogonCount,OriginatingComputer,DomainController,SecurityEventRecordId |
        Out-GridView -Title $title

    # Build a concise, client-friendly summary
    $lockCount    = ($records | Where-Object { $_.When -ne '' }).Count
    $distinctUsers= ($records | Select-Object -ExpandProperty User -Unique | Where-Object { $_ -ne '' }).Count
    $stillLocked  = ($records | Where-Object { $_.CurrentlyLockedOut -eq $true } | Select-Object -ExpandProperty User -Unique).Count

    $summary = New-Object System.Text.StringBuilder
    [void]$summary.AppendLine("🔒 **Account Lockout Review**")
    [void]$summary.AppendLine("**Domain Controller:** $dc")
    [void]$summary.AppendLine(("**Window:** last {0} hour(s) (from {1} to {2})" -f $hours, (New-FriendlyDateString $startTime), (New-FriendlyDateString (Get-Date))))
    if ($username) { [void]$summary.AppendLine("**User filter:** $username") }
    [void]$summary.AppendLine("")
    [void]$summary.AppendLine("**Key Findings:**")
    [void]$summary.AppendLine("- 🕒 Lockout events found: **$lockCount**")
    [void]$summary.AppendLine("- 👤 Distinct accounts impacted: **$distinctUsers**")
    [void]$summary.AppendLine("- ✅ Currently locked out: **$stillLocked**")
    [void]$summary.AppendLine("")
    [void]$summary.AppendLine("**Details (chronological):**")

    foreach ($r in ($records | Sort-Object {
        if ($_.When) { Get-Date $_.When } else { Get-Date '1900-01-01' }
    })) {
        $stamp = (if ($r.When) { $r.When } else { '(no timestamp)' })
        $line  = "• $stamp – User: $($r.User) ($($r.DisplayName))"
        if ($r.OriginatingComputer)   { $line += " – Source: $($r.OriginatingComputer)" }
        if ($null -ne $r.CurrentlyLockedOut) { $line += " – LockedNow: $($r.CurrentlyLockedOut)" }
        [void]$summary.AppendLine($line)
    }

    [void]$summary.AppendLine("")
    [void]$summary.AppendLine("> ℹ️ **Note:** Event 4740 indicates the DC recorded a lockout. The `OriginatingComputer` is typically the source that triggered the lockout (e.g., stale credentials, mapped drives, services, or mobile mail profiles).")

    # Print and copy to clipboard
    $summaryText = $summary.ToString()
    Write-Host ""
    Write-Host "===== Client-Ready Summary =====" -ForegroundColor Green
    Write-Host $summaryText
    Write-Host "================================" -ForegroundColor Green

    # Copy to clipboard for easy pasting into a ticket
    try {
        Add-Type -AssemblyName PresentationCore | Out-Null
        Add-Type -AssemblyName PresentationFramework | Out-Null
        [System.Windows.Clipboard]::SetText($summaryText)
        Write-Host "📋 Summary copied to clipboard." -ForegroundColor Yellow
    } catch {
        Write-Warning "Could not copy to clipboard automatically: $($_.Exception.Message)"
    }

    # Optional HTML export
    if ($export) {
        $desktop  = [Environment]::GetFolderPath('Desktop')
        $htmlPath = Join-Path $desktop ("Lockout-Report_{0:yyyyMMdd_HHmmss}.html" -f (Get-Date))
        $htmlBody = ($records |
            Select-Object When,User,DisplayName,CurrentlyLockedOut,LastBadPassword,BadLogonCount,OriginatingComputer,DomainController,SecurityEventRecordId |
            ConvertTo-Html -Title $title -PreContent "<h2 style='font-family:Segoe UI;'>$title</h2><p style='font-family:Segoe UI;'>Generated: $(Get-Date)</p>" |
            Out-String)
        try {
            $htmlBody | Set-Content -Path $htmlPath -Encoding UTF8
            Write-Host "📝 HTML report exported to: $htmlPath" -ForegroundColor Yellow
        } catch {
            Write-Warning "Failed to write HTML report: $($_.Exception.Message)"
        }
    }
} else {
    Write-Host "✅ No lockout events or locked-out users found in the last $hours hour(s) on $dc." -ForegroundColor Green
}

Write-Host "Completed. 🧩" -ForegroundColor Cyan
```