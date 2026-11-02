# IIS IP Reputation Dashboard

## Summary
This PowerShell application analyses IIS W3C-format log files, identifies the Top 10 public IP addresses by request count, and presents the results in an interactive HTML dashboard.
<br>
<br>
Optionally. If you specify an IP Abuse DB API Key it will automatically check the Top 10 IPs

## Features

- Browse for an IIS log and analyse the latest X minutes or a custom time range.
- Optimised reverse log reading for faster analysis of large IIS logs.
- Counts public IPv4 addresses and identifies the Top 10 by request volume.
- Optional IP Abuse DB integration to display reputation scores for the Top 10 IPs.
- Interactive HTML dashboard with search, sorting, colour-coded scores, light/dark themes and CSV export.
- Generates timestamped HTML and CSV reports.
- PowerShell 5.1 / PowerShell ISE compatible.

## Usage

- Run the script and select the IIS log file and required time range.
- Optionally enter an IP Abuse DB API key and choose the output folder.
- Select **Analyse Log** to generate the Top 10 IP reputation dashboard.

## Notes

- If no API key is supplied, analysis continues normally and reputation scores display as `N/A`.
- Only the Top 10 IPs are submitted to IP Abuse DB.
- Only public IPv4 addresses are analysed; private/non-routable ranges are excluded.
- The performance optimisation assumes the IIS log is chronological, with newer entries at the end.
- Generated reports may contain client IP addresses and should be handled appropriately.

<br>

```Powershell
#requires -Version 5.1

Set-StrictMode -Version 2.0
$ErrorActionPreference = 'Stop'

# -----------------------------------------------------------------------------
# Configuration
# -----------------------------------------------------------------------------
$DefaultLogFile = 'Specify the log file here'
$DefaultOutputFolder = 'C:\Temp'
$CacheFileName = 'AbuseIPDB-Cache.csv'
$MaxAgeInDays = 90

$ApiKey = ''

Add-Type -AssemblyName System.Windows.Forms
Add-Type -AssemblyName System.Drawing
Add-Type -AssemblyName System.Web
[System.Windows.Forms.Application]::EnableVisualStyles()

# -----------------------------------------------------------------------------
# Fast reverse log reader
# -----------------------------------------------------------------------------
if (-not ('FastReverseLogReader' -as [type])) {
    Add-Type -TypeDefinition @'
using System;
using System.Collections.Generic;
using System.IO;
using System.Text;

public static class FastReverseLogReader
{
    public static IEnumerable<string> ReadLines(string path, int blockSize)
    {
        using (FileStream stream = new FileStream(
            path, FileMode.Open, FileAccess.Read, FileShare.ReadWrite,
            blockSize, FileOptions.RandomAccess))
        {
            long position = stream.Length;
            byte[] block = new byte[blockSize];
            List<byte> reversedLine = new List<byte>(1024);

            while (position > 0)
            {
                int count = (int)Math.Min(blockSize, position);
                position -= count;
                stream.Position = position;

                int total = 0;
                while (total < count)
                {
                    int read = stream.Read(block, total, count - total);
                    if (read == 0) break;
                    total += read;
                }

                for (int i = total - 1; i >= 0; i--)
                {
                    byte b = block[i];
                    if (b == 10)
                    {
                        if (reversedLine.Count == 0) continue;
                        reversedLine.Reverse();
                        byte[] bytes = reversedLine.ToArray();
                        int length = bytes.Length;
                        if (length > 0 && bytes[length - 1] == 13) length--;
                        yield return Encoding.UTF8.GetString(bytes, 0, length);
                        reversedLine.Clear();
                    }
                    else
                    {
                        reversedLine.Add(b);
                    }
                }
            }

            if (reversedLine.Count > 0)
            {
                reversedLine.Reverse();
                byte[] bytes = reversedLine.ToArray();
                int length = bytes.Length;
                if (length > 0 && bytes[length - 1] == 13) length--;
                yield return Encoding.UTF8.GetString(bytes, 0, length);
            }
        }
    }
}
'@
}

# -----------------------------------------------------------------------------
# Selection window
# -----------------------------------------------------------------------------
function Show-IisAnalysisPicker {
    param(
        [string]$InitialLogFile,
        [string]$InitialOutputFolder
    )

    $form = New-Object System.Windows.Forms.Form
    $form.Text = 'IIS IP Analysis'
    $form.StartPosition = 'CenterScreen'
    $form.Size = New-Object System.Drawing.Size(720, 620)
    $form.MinimumSize = New-Object System.Drawing.Size(720, 620)
    $form.FormBorderStyle = 'FixedDialog'
    $form.MaximizeBox = $false
    $form.MinimizeBox = $false
    $form.BackColor = [System.Drawing.Color]::FromArgb(11, 18, 32)
    $form.ForeColor = [System.Drawing.Color]::FromArgb(231, 238, 252)
    $form.Font = New-Object System.Drawing.Font('Segoe UI', 9)

    $title = New-Object System.Windows.Forms.Label
    $title.Text = 'IIS IP Reputation Dashboard'
    $title.Font = New-Object System.Drawing.Font('Segoe UI Semibold', 16)
    $title.Location = New-Object System.Drawing.Point(24, 18)
    $title.AutoSize = $true

    $subtitle = New-Object System.Windows.Forms.Label
    $subtitle.Text = 'Select an IIS log and the period to analyse. Only the Top 10 IPs are checked with AbuseIPDB.'
    $subtitle.ForeColor = [System.Drawing.Color]::FromArgb(159, 176, 208)
    $subtitle.Location = New-Object System.Drawing.Point(27, 52)
    $subtitle.AutoSize = $true

    $lblLog = New-Object System.Windows.Forms.Label
    $lblLog.Text = 'IIS log file'
    $lblLog.Location = New-Object System.Drawing.Point(28, 91)
    $lblLog.AutoSize = $true

    $txtLog = New-Object System.Windows.Forms.TextBox
    $txtLog.Location = New-Object System.Drawing.Point(30, 113)
    $txtLog.Size = New-Object System.Drawing.Size(555, 25)
    $txtLog.Text = $InitialLogFile
    $txtLog.BackColor = [System.Drawing.Color]::FromArgb(17, 26, 46)
    $txtLog.ForeColor = [System.Drawing.Color]::FromArgb(231, 238, 252)
    $txtLog.BorderStyle = 'FixedSingle'

    $btnBrowseLog = New-Object System.Windows.Forms.Button
    $btnBrowseLog.Text = 'Browse...'
    $btnBrowseLog.Location = New-Object System.Drawing.Point(595, 111)
    $btnBrowseLog.Size = New-Object System.Drawing.Size(85, 28)
    $btnBrowseLog.FlatStyle = 'Flat'
    $btnBrowseLog.BackColor = [System.Drawing.Color]::FromArgb(24, 41, 72)
    $btnBrowseLog.ForeColor = [System.Drawing.Color]::FromArgb(110, 168, 254)

    $group = New-Object System.Windows.Forms.GroupBox
    $group.Text = 'Time range'
    $group.Location = New-Object System.Drawing.Point(30, 158)
    $group.Size = New-Object System.Drawing.Size(650, 165)
    $group.ForeColor = [System.Drawing.Color]::FromArgb(231, 238, 252)

    $rbLatest = New-Object System.Windows.Forms.RadioButton
    $rbLatest.Text = 'Analyse the latest period in the log'
    $rbLatest.Location = New-Object System.Drawing.Point(18, 29)
    $rbLatest.AutoSize = $true
    $rbLatest.Checked = $true

    $lblMinutes = New-Object System.Windows.Forms.Label
    $lblMinutes.Text = 'Minutes'
    $lblMinutes.Location = New-Object System.Drawing.Point(342, 31)
    $lblMinutes.AutoSize = $true

    $numMinutes = New-Object System.Windows.Forms.NumericUpDown
    $numMinutes.Location = New-Object System.Drawing.Point(400, 28)
    $numMinutes.Size = New-Object System.Drawing.Size(105, 25)
    $numMinutes.Minimum = 1
    $numMinutes.Maximum = 10080
    $numMinutes.Value = 90
    $numMinutes.BackColor = [System.Drawing.Color]::FromArgb(17, 26, 46)
    $numMinutes.ForeColor = [System.Drawing.Color]::FromArgb(231, 238, 252)

    $rbCustom = New-Object System.Windows.Forms.RadioButton
    $rbCustom.Text = 'Use a specific start and end time'
    $rbCustom.Location = New-Object System.Drawing.Point(18, 71)
    $rbCustom.AutoSize = $true

    $lblStart = New-Object System.Windows.Forms.Label
    $lblStart.Text = 'Start'
    $lblStart.Location = New-Object System.Drawing.Point(35, 111)
    $lblStart.AutoSize = $true

    $dtStart = New-Object System.Windows.Forms.DateTimePicker
    $dtStart.Location = New-Object System.Drawing.Point(80, 107)
    $dtStart.Size = New-Object System.Drawing.Size(220, 25)
    $dtStart.Format = [System.Windows.Forms.DateTimePickerFormat]::Custom
    $dtStart.CustomFormat = 'yyyy-MM-dd HH:mm:ss'
    $dtStart.Value = (Get-Date).AddMinutes(-90)
    $dtStart.Enabled = $false

    $lblEnd = New-Object System.Windows.Forms.Label
    $lblEnd.Text = 'End'
    $lblEnd.Location = New-Object System.Drawing.Point(330, 111)
    $lblEnd.AutoSize = $true

    $dtEnd = New-Object System.Windows.Forms.DateTimePicker
    $dtEnd.Location = New-Object System.Drawing.Point(370, 107)
    $dtEnd.Size = New-Object System.Drawing.Size(220, 25)
    $dtEnd.Format = [System.Windows.Forms.DateTimePickerFormat]::Custom
    $dtEnd.CustomFormat = 'yyyy-MM-dd HH:mm:ss'
    $dtEnd.Value = Get-Date
    $dtEnd.Enabled = $false

    $group.Controls.AddRange(@($rbLatest, $lblMinutes, $numMinutes, $rbCustom, $lblStart, $dtStart, $lblEnd, $dtEnd))

    $lblApiKey = New-Object System.Windows.Forms.Label
    $lblApiKey.Text = 'IP Abuse DB API key'
    $lblApiKey.Location = New-Object System.Drawing.Point(28, 342)
    $lblApiKey.AutoSize = $true

    $txtApiKey = New-Object System.Windows.Forms.TextBox
    $txtApiKey.Location = New-Object System.Drawing.Point(30, 364)
    $txtApiKey.Size = New-Object System.Drawing.Size(650, 25)
    $txtApiKey.UseSystemPasswordChar = $true
    $txtApiKey.BackColor = [System.Drawing.Color]::FromArgb(17, 26, 46)
    $txtApiKey.ForeColor = [System.Drawing.Color]::FromArgb(231, 238, 252)
    $txtApiKey.BorderStyle = 'FixedSingle'

    $infoPanel = New-Object System.Windows.Forms.Panel
    $infoPanel.Location = New-Object System.Drawing.Point(30, 397)
    $infoPanel.Size = New-Object System.Drawing.Size(650, 48)
    $infoPanel.BackColor = [System.Drawing.Color]::FromArgb(20, 45, 75)
    $infoPanel.BorderStyle = 'FixedSingle'

    $infoIcon = New-Object System.Windows.Forms.Label
    $infoIcon.Text = 'i'
    $infoIcon.Font = New-Object System.Drawing.Font('Segoe UI Semibold', 12)
    $infoIcon.ForeColor = [System.Drawing.Color]::FromArgb(110, 168, 254)
    $infoIcon.Location = New-Object System.Drawing.Point(12, 11)
    $infoIcon.AutoSize = $true

    $infoText = New-Object System.Windows.Forms.Label
    $infoText.Text = 'If you do not enter an API key, the report will still be created, but the IPs will not be checked against IP Abuse DB.'
    $infoText.ForeColor = [System.Drawing.Color]::FromArgb(190, 210, 240)
    $infoText.Location = New-Object System.Drawing.Point(36, 8)
    $infoText.Size = New-Object System.Drawing.Size(600, 34)

    $infoPanel.Controls.AddRange(@($infoIcon, $infoText))

    $lblOutput = New-Object System.Windows.Forms.Label
    $lblOutput.Text = 'Report output folder'
    $lblOutput.Location = New-Object System.Drawing.Point(28, 462)
    $lblOutput.AutoSize = $true

    $txtOutput = New-Object System.Windows.Forms.TextBox
    $txtOutput.Location = New-Object System.Drawing.Point(30, 484)
    $txtOutput.Size = New-Object System.Drawing.Size(555, 25)
    $txtOutput.Text = $InitialOutputFolder
    $txtOutput.BackColor = [System.Drawing.Color]::FromArgb(17, 26, 46)
    $txtOutput.ForeColor = [System.Drawing.Color]::FromArgb(231, 238, 252)
    $txtOutput.BorderStyle = 'FixedSingle'

    $btnBrowseOutput = New-Object System.Windows.Forms.Button
    $btnBrowseOutput.Text = 'Browse...'
    $btnBrowseOutput.Location = New-Object System.Drawing.Point(595, 482)
    $btnBrowseOutput.Size = New-Object System.Drawing.Size(85, 28)
    $btnBrowseOutput.FlatStyle = 'Flat'
    $btnBrowseOutput.BackColor = [System.Drawing.Color]::FromArgb(24, 41, 72)
    $btnBrowseOutput.ForeColor = [System.Drawing.Color]::FromArgb(110, 168, 254)

    $chkOpen = New-Object System.Windows.Forms.CheckBox
    $chkOpen.Text = 'Open the HTML report when complete'
    $chkOpen.Location = New-Object System.Drawing.Point(30, 524)
    $chkOpen.AutoSize = $true
    $chkOpen.Checked = $true

    $btnRun = New-Object System.Windows.Forms.Button
    $btnRun.Text = 'Analyse Log'
    $btnRun.Location = New-Object System.Drawing.Point(484, 541)
    $btnRun.Size = New-Object System.Drawing.Size(100, 32)
    $btnRun.FlatStyle = 'Flat'
    $btnRun.BackColor = [System.Drawing.Color]::FromArgb(29, 78, 137)
    $btnRun.ForeColor = [System.Drawing.Color]::White

    $btnCancel = New-Object System.Windows.Forms.Button
    $btnCancel.Text = 'Cancel'
    $btnCancel.Location = New-Object System.Drawing.Point(595, 541)
    $btnCancel.Size = New-Object System.Drawing.Size(85, 32)
    $btnCancel.FlatStyle = 'Flat'
    $btnCancel.DialogResult = [System.Windows.Forms.DialogResult]::Cancel

    $form.CancelButton = $btnCancel
    $form.Controls.AddRange(@(
        $title, $subtitle, $lblLog, $txtLog, $btnBrowseLog, $group,
        $lblApiKey, $txtApiKey, $infoPanel, $lblOutput, $txtOutput, $btnBrowseOutput, $chkOpen, $btnRun, $btnCancel
    ))

    $toggleMode = {
        $isLatest = $rbLatest.Checked
        $numMinutes.Enabled = $isLatest
        $dtStart.Enabled = -not $isLatest
        $dtEnd.Enabled = -not $isLatest
    }
    $rbLatest.Add_CheckedChanged($toggleMode)
    $rbCustom.Add_CheckedChanged($toggleMode)

    $btnBrowseLog.Add_Click({
        $dialog = New-Object System.Windows.Forms.OpenFileDialog
        $dialog.Title = 'Select IIS log file'
        $dialog.Filter = 'IIS log files (*.log;*.txt;*)|*.log;*.txt;*|All files (*.*)|*.*'
        $dialog.CheckFileExists = $true
        if (Test-Path -LiteralPath $txtLog.Text -PathType Leaf) {
            $dialog.InitialDirectory = Split-Path -LiteralPath $txtLog.Text -Parent
            $dialog.FileName = Split-Path -LiteralPath $txtLog.Text -Leaf
        }
        if ($dialog.ShowDialog() -eq [System.Windows.Forms.DialogResult]::OK) {
            $txtLog.Text = $dialog.FileName
        }
        $dialog.Dispose()
    })

    $btnBrowseOutput.Add_Click({
        $dialog = New-Object System.Windows.Forms.FolderBrowserDialog
        $dialog.Description = 'Select the report output folder'
        if (Test-Path -LiteralPath $txtOutput.Text -PathType Container) {
            $dialog.SelectedPath = $txtOutput.Text
        }
        if ($dialog.ShowDialog() -eq [System.Windows.Forms.DialogResult]::OK) {
            $txtOutput.Text = $dialog.SelectedPath
        }
        $dialog.Dispose()
    })

    $btnRun.Add_Click({
        if (-not (Test-Path -LiteralPath $txtLog.Text -PathType Leaf)) {
            [System.Windows.Forms.MessageBox]::Show('Select a valid IIS log file.', 'Invalid log file', 'OK', 'Warning') | Out-Null
            return
        }
        if ([string]::IsNullOrWhiteSpace($txtOutput.Text)) {
            [System.Windows.Forms.MessageBox]::Show('Select an output folder.', 'Invalid output folder', 'OK', 'Warning') | Out-Null
            return
        }
        if ($rbCustom.Checked -and $dtStart.Value -ge $dtEnd.Value) {
            [System.Windows.Forms.MessageBox]::Show('The start time must be earlier than the end time.', 'Invalid time range', 'OK', 'Warning') | Out-Null
            return
        }
        $form.DialogResult = [System.Windows.Forms.DialogResult]::OK
        $form.Close()
    })

    $result = $form.ShowDialog()
    if ($result -ne [System.Windows.Forms.DialogResult]::OK) {
        $form.Dispose()
        return $null
    }

    $selection = [PSCustomObject]@{
        LogFile      = $txtLog.Text
        OutputFolder = $txtOutput.Text
        Mode         = $(if ($rbLatest.Checked) { 'Latest' } else { 'Custom' })
        Minutes      = [int]$numMinutes.Value
        StartTime    = $dtStart.Value
        EndTime      = $dtEnd.Value
        OpenReport   = $chkOpen.Checked
        ApiKey       = $txtApiKey.Text.Trim()
    }
    $form.Dispose()
    return $selection
}

# -----------------------------------------------------------------------------
# Helpers
# -----------------------------------------------------------------------------
function Test-IsPublicIPv4 {
    param([Parameter(Mandatory)][string]$Address)

    if ($Address -notmatch '^((25[0-5]|2[0-4]\d|1?\d?\d)\.){3}(25[0-5]|2[0-4]\d|1?\d?\d)$') {
        return $false
    }

    $o = $Address.Split('.')
    $a = [int]$o[0]
    $b = [int]$o[1]
    $c = [int]$o[2]

    if ($a -eq 0 -or $a -eq 10 -or $a -eq 127 -or $a -ge 224) { return $false }
    if ($a -eq 100 -and $b -ge 64 -and $b -le 127) { return $false }
    if ($a -eq 169 -and $b -eq 254) { return $false }
    if ($a -eq 172 -and $b -ge 16 -and $b -le 31) { return $false }
    if ($a -eq 192 -and $b -eq 168) { return $false }
    if ($a -eq 192 -and $b -eq 0 -and $c -eq 0) { return $false }
    if ($a -eq 192 -and $b -eq 0 -and $c -eq 2) { return $false }
    if ($a -eq 198 -and ($b -eq 18 -or $b -eq 19)) { return $false }
    if ($a -eq 198 -and $b -eq 51 -and $c -eq 100) { return $false }
    if ($a -eq 203 -and $b -eq 0 -and $c -eq 113) { return $false }

    return $true
}

function Get-AbuseScoreClass {
    param($Score)
    if ($null -eq $Score) { return 'unknown' }
    if ([int]$Score -ge 75) { return 'critical' }
    if ([int]$Score -ge 50) { return 'high' }
    if ([int]$Score -ge 25) { return 'medium' }
    return 'low'
}

function Export-IisIpHtml {
    param(
        [Parameter(Mandatory)]$Results,
        [Parameter(Mandatory)][string]$Path,
        [Parameter(Mandatory)][string]$LogFile,
        [Parameter(Mandatory)][datetime]$StartTime,
        [Parameter(Mandatory)][datetime]$EndTime,
        [Parameter(Mandatory)][long]$LinesRead,
        [Parameter(Mandatory)][long]$LinesInPeriod,
        [Parameter(Mandatory)][long]$PublicMentions,
        [Parameter(Mandatory)][int]$UniqueIPs,
        [Parameter(Mandatory)][bool]$AbuseLookupEnabled
    )

    $encodedLog = [System.Web.HttpUtility]::HtmlEncode($LogFile)
    $startText = $StartTime.ToString('yyyy-MM-dd HH:mm:ss')
    $endText = $EndTime.ToString('yyyy-MM-dd HH:mm:ss')
    $generated = (Get-Date).ToString('yyyy-MM-dd HH:mm:ss')
    $totalTop10 = [long](($Results | Measure-Object Requests -Sum).Sum)
    $ValidScores = @(
    $Results |
        ForEach-Object { $_.'Abuse Score' } |
        Where-Object {
            $null -ne $_ -and "$_" -match '^\d+$'
        }
    )

    if ($ValidScores.Count -gt 0) 
        {
            $highestScore = (
            $ValidScores |
            Measure-Object -Maximum
            ).Maximum
        }
    else {
    $highestScore = 'N/A'
        }

    $rows = New-Object System.Collections.Generic.List[string]
    $rank = 0
    foreach ($item in ($Results | Sort-Object Requests -Descending)) {
        $rank++
        $ip = [System.Web.HttpUtility]::HtmlEncode([string]$item.'IP Address')
        $requests = [long]$item.Requests
        $score = $item.'Abuse Score'
        $scoreText = if ($null -eq $score) { 'N/A' } else { [string]$score }
        $scoreClass = Get-AbuseScoreClass -Score $score
        $percent = if ($totalTop10 -gt 0) { [math]::Round(($requests / $totalTop10) * 100, 1) } else { 0 }
        $rows.Add(@"
<tr data-ip="$ip" data-score="$scoreText" data-requests="$requests">
  <td class="rank">#$rank</td>
  <td>
    <div class="request-cell"><strong>$('{0:N0}' -f $requests)</strong><span>$percent% of Top 10 traffic</span></div>
    <div class="bar"><i style="width:$percent%"></i></div>
  </td>
  <td class="mono ip-cell">$ip</td>
  <td><span class="score score-$scoreClass">$scoreText</span></td>
</tr>
"@)
    }

    $rowsHtml = $rows -join "`n"

    $html = @'
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>IIS IP Reputation Report</title>
<style>
:root{--bg:#0b1220;--card:#111a2e;--muted:#9fb0d0;--text:#e7eefc;--accent:#6ea8fe;--border:rgba(255,255,255,.08);--ok:#36d399;--warn:#fbbf24;--err:#f87171;--crit:#fb7185;--shadow:0 20px 50px rgba(0,0,0,.45)}
[data-theme="light"]{--bg:#f5f7fa;--card:#fff;--muted:#64748b;--text:#0f172a;--accent:#3b82f6;--border:rgba(0,0,0,.08);--shadow:0 20px 50px rgba(0,0,0,.08)}
*{box-sizing:border-box}html{scroll-behavior:smooth}body{margin:0;background:linear-gradient(180deg,#070b14 0%,#0b1220 60%,#070b14 100%);color:var(--text);font-family:"Inter","Segoe UI",system-ui,-apple-system,Arial,sans-serif;min-height:100vh;overflow-x:hidden}
[data-theme="light"] body{background:linear-gradient(180deg,#eef3f9 0%,#f8fafc 65%,#eef3f9 100%)}
body:before{content:"";position:fixed;inset:-50%;background:radial-gradient(circle at 20% 50%,rgba(110,168,254,.08),transparent 45%),radial-gradient(circle at 80% 80%,rgba(251,113,133,.06),transparent 45%);animation:shift 20s ease infinite;pointer-events:none}body:after{content:"";position:fixed;inset:0;background-image:url("data:image/svg+xml,%3Csvg viewBox='0 0 400 400' xmlns='http://www.w3.org/2000/svg'%3E%3Cfilter id='n'%3E%3CfeTurbulence type='fractalNoise' baseFrequency='.9' numOctaves='4' stitchTiles='stitch'/%3E%3C/filter%3E%3Crect width='100%25' height='100%25' filter='url(%23n)' opacity='.03'/%3E%3C/svg%3E");opacity:.4;pointer-events:none}@keyframes shift{50%{transform:translate(-5%,-5%) rotate(5deg)}}
.container{max-width:1200px;margin:32px auto;padding:0 16px;position:relative;z-index:1}.card{background:rgba(17,26,46,.85);backdrop-filter:blur(18px);border:1px solid var(--border);border-radius:14px;box-shadow:var(--shadow);padding:18px;animation:fadeUp .35s ease-out both}[data-theme="light"] .card{background:rgba(255,255,255,.95)}@keyframes fadeUp{from{opacity:0;transform:translateY(8px)}to{opacity:1;transform:none}}
.hero{background:linear-gradient(145deg,rgba(20,30,55,.95),rgba(12,18,35,.95));border-color:rgba(110,168,254,.15)}[data-theme="light"] .hero{background:linear-gradient(145deg,rgba(255,255,255,.98),rgba(245,247,250,.98))}.hero-top{display:flex;justify-content:space-between;gap:16px;align-items:flex-start;flex-wrap:wrap}.h-title{font-size:24px;margin:0}.sub{color:var(--muted);margin-top:7px;font-size:13px;line-height:1.7}.mono{font-family:ui-monospace,SFMono-Regular,Menlo,Monaco,Consolas,"Liberation Mono",monospace}.actions{display:flex;gap:8px;flex-wrap:wrap}button{padding:9px 13px;border-radius:10px;border:1px solid var(--border);background:rgba(110,168,254,.12);color:var(--accent);cursor:pointer;font:inherit;transition:.2s}button:hover{background:rgba(110,168,254,.22);box-shadow:0 0 20px rgba(110,168,254,.3);transform:translateY(-1px)}
.kpis{display:grid;grid-template-columns:repeat(4,minmax(145px,1fr));gap:12px;margin-top:18px}.kpi{background:rgba(255,255,255,.03);border:1px solid var(--border);border-radius:12px;padding:13px;transition:.2s}.kpi:hover{transform:translateY(-2px);box-shadow:0 8px 20px rgba(0,0,0,.2)}.kpi b{display:block;font-size:20px}.kpi span{color:var(--muted);font-size:12px}.toolbar{display:flex;gap:10px;align-items:center;justify-content:space-between;flex-wrap:wrap;margin:16px 0;position:sticky;top:0;z-index:20;background:rgba(11,18,32,.85);backdrop-filter:blur(18px);padding:14px;border-radius:14px;border:1px solid var(--border);box-shadow:0 8px 30px rgba(0,0,0,.35)}[data-theme="light"] .toolbar{background:rgba(255,255,255,.85)}input[type="search"]{flex:1;min-width:260px;padding:10px 12px;border-radius:10px;border:1px solid var(--border);background:rgba(0,0,0,.22);color:var(--text);outline:none}[data-theme="light"] input[type="search"]{background:rgba(0,0,0,.04)}input[type="search"]:focus{border-color:rgba(110,168,254,.55);box-shadow:0 0 0 3px rgba(110,168,254,.18)}
.table-wrap{overflow-x:auto}table{width:100%;border-collapse:separate;border-spacing:0;min-width:700px}thead th{text-align:left;color:var(--muted);font-size:12px;padding:12px 10px;border-bottom:1px solid var(--border);cursor:pointer;user-select:none}tbody td{padding:15px 10px;border-bottom:1px solid var(--border);vertical-align:middle}tbody tr{transition:.22s}tbody tr:hover{background:rgba(255,255,255,.04);transform:translateY(-1px)}.rank{color:var(--muted);font-weight:700;width:60px}.ip-cell{font-size:13px}.request-cell{display:flex;align-items:baseline;gap:9px}.request-cell strong{font-size:16px}.request-cell span{font-size:11px;color:var(--muted)}.bar{margin-top:8px;width:100%;height:5px;border-radius:9px;background:rgba(255,255,255,.06);overflow:hidden}.bar i{display:block;height:100%;border-radius:9px;background:linear-gradient(90deg,#3b82f6,#6ea8fe)}
.score{display:inline-flex;min-width:58px;justify-content:center;padding:5px 11px;border-radius:999px;border:1px solid var(--border);font-weight:700}.score-low{color:var(--ok);background:rgba(54,211,153,.12);border-color:rgba(54,211,153,.35)}.score-medium{color:var(--warn);background:rgba(251,191,36,.12);border-color:rgba(251,191,36,.35)}.score-high{color:var(--err);background:rgba(248,113,113,.12);border-color:rgba(248,113,113,.35)}.score-critical{color:var(--crit);background:rgba(251,113,133,.15);border-color:rgba(251,113,133,.38);box-shadow:0 0 12px rgba(251,113,133,.2)}.score-unknown{color:var(--muted);background:rgba(255,255,255,.04)}.legend{display:flex;gap:8px;flex-wrap:wrap;margin:0 0 14px}.legend span{font-size:11px;color:var(--muted);padding:4px 8px;border:1px solid var(--border);border-radius:999px}.footer{margin-top:14px;color:var(--muted);font-size:12px;display:flex;justify-content:space-between;gap:12px;flex-wrap:wrap}.empty{padding:30px;text-align:center;color:var(--muted)}body.compact tbody td{padding:8px 10px}body.compact .bar{display:none}
@media(max-width:780px){.kpis{grid-template-columns:repeat(2,1fr)}.hero-top{display:block}.actions{margin-top:12px}}@media(max-width:460px){.kpis{grid-template-columns:1fr}.container{margin:16px auto}.h-title{font-size:20px}}@media(prefers-reduced-motion:reduce){*,*:before,*:after{animation:none!important;transition:none!important;scroll-behavior:auto!important}}
</style>
</head>
<body class="compact">
<div class="container">
  <section class="card hero">
    <div class="hero-top">
      <div>
        <h1 class="h-title">IIS IP Reputation Report</h1>
        <div class="sub">Window: <span class="mono">START_TIME</span> â†’ <span class="mono">END_TIME</span><br>Source: <span class="mono">LOG_FILE</span><br>Generated: <span class="mono">GENERATED</span></div>
      </div>
      <div class="actions"><button id="copyTop">ðŸ“‹ Copy table</button><button id="exportCsv">ðŸ’¾ Export CSV</button><button id="density">ðŸ“Š Compact</button><button id="theme">ðŸŒ™</button></div>
    </div>
    <div class="kpis">
      <div class="kpi"><b>PUBLIC_MENTIONS</b><span>Public IP mentions</span></div>
      <div class="kpi"><b>UNIQUE_IPS</b><span>Unique public IPs</span></div>
      <div class="kpi"><b>LINES_PERIOD</b><span>Lines in selected period</span></div>
      <div class="kpi"><b>HIGHEST_SCORE</b><span>Highest AbuseIPDB score</span></div>
    </div>
  </section>

  <section class="toolbar"><input id="search" type="search" placeholder="Search IP address, request count or score..."><div class="actions"><button id="clear">ðŸ—‘ï¸ Clear</button></div></section>

  <section class="card">
    <div class="legend"><span>0â€“24 Low</span><span>25â€“49 Medium</span><span>50â€“74 High</span><span>75â€“100 Critical</span></div>
    <div class="table-wrap">
      <table id="reportTable">
        <thead><tr><th data-type="number">Rank</th><th data-type="number">Requests</th><th data-type="text">IP Address</th><th data-type="number">Abuse Score</th></tr></thead>
        <tbody>ROWS</tbody>
      </table>
    </div>
    <div id="empty" class="empty" hidden>No rows match the current search.</div>
    <div class="footer"><span id="count"></span><span>Recent lines read: LINES_READ â€¢ LOOKUP_MODE</span></div>
  </section>
</div>
<script>
(function(){
 const table=document.getElementById('reportTable'), tbody=table.tBodies[0], rows=[...tbody.rows], search=document.getElementById('search'), count=document.getElementById('count'), empty=document.getElementById('empty');
 function visible(){return rows.filter(r=>!r.hidden)}
 function update(){const q=search.value.trim().toLowerCase();let n=0;rows.forEach(r=>{const show=!q||r.innerText.toLowerCase().includes(q);r.hidden=!show;if(show)n++});count.textContent='Showing '+n+' of '+rows.length+' IP addresses';empty.hidden=n!==0}
 search.addEventListener('input',update);document.getElementById('clear').onclick=()=>{search.value='';update()};
 document.querySelectorAll('th').forEach((th,index)=>{let asc=false;th.onclick=()=>{asc=!asc;const type=th.dataset.type;rows.sort((a,b)=>{let x=a.cells[index].innerText.replace(/[^0-9.-]/g,''),y=b.cells[index].innerText.replace(/[^0-9.-]/g,'');if(type==='number'){x=parseFloat(x)||0;y=parseFloat(y)||0;return asc?x-y:y-x}x=a.cells[index].innerText;y=b.cells[index].innerText;return asc?x.localeCompare(y):y.localeCompare(x)});rows.forEach(r=>tbody.appendChild(r));update()}});
 let light=false;document.getElementById('theme').onclick=function(){light=!light;document.documentElement.toggleAttribute('data-theme',light);this.textContent=light?'â˜€ï¸':'ðŸŒ™'};
 let compact=true;document.getElementById('density').textContent='ðŸ“Š Normal';document.getElementById('density').onclick=function(){compact=!compact;document.body.classList.toggle('compact',compact);this.textContent=compact?'ðŸ“Š Normal':'ðŸ“Š Compact'};
 function csv(){let out='Rank,Requests,IP Address,Abuse Score\n';visible().forEach(r=>out+= [...r.cells].map(c=>'"'+c.innerText.replace(/"/g,'""').replace(/\n/g,' ')+'"').join(',')+'\n');return out}
 document.getElementById('exportCsv').onclick=()=>{const blob=new Blob([csv()],{type:'text/csv'}),url=URL.createObjectURL(blob),a=document.createElement('a');a.href=url;a.download='IIS_IP_Reputation_Filtered.csv';a.click();URL.revokeObjectURL(url)};
 document.getElementById('copyTop').onclick=async function(){const text=[...table.rows].filter((r,i)=>i===0||!r.hidden).map(r=>[...r.cells].map(c=>c.innerText.replace(/\n/g,' ')).join('\t')).join('\n');try{await navigator.clipboard.writeText(text);const old=this.textContent;this.textContent='âœ“ Copied';setTimeout(()=>this.textContent=old,1500)}catch(e){alert('Clipboard access was blocked by the browser. Use Export CSV instead.')}};
 update();
})();
</script>
</body>
</html>
'@

    $html = $html.Replace('START_TIME', $startText)
    $html = $html.Replace('END_TIME', $endText)
    $html = $html.Replace('LOG_FILE', $encodedLog)
    $html = $html.Replace('GENERATED', $generated)
    $html = $html.Replace('PUBLIC_MENTIONS', ('{0:N0}' -f $PublicMentions))
    $html = $html.Replace('UNIQUE_IPS', ('{0:N0}' -f $UniqueIPs))
    $html = $html.Replace('LINES_PERIOD', ('{0:N0}' -f $LinesInPeriod))
    $html = $html.Replace('HIGHEST_SCORE', [string]$highestScore)
    $html = $html.Replace('LINES_READ', ('{0:N0}' -f $LinesRead))
    $lookupMode = if ($AbuseLookupEnabled) { 'Top 10 IP Abuse DB checks enabled' } else { 'IP Abuse DB checks skipped' }
    $html = $html.Replace('LOOKUP_MODE', $lookupMode)
    $html = $html.Replace('ROWS', $rowsHtml)

    $Utf8Bom = New-Object System.Text.UTF8Encoding($true)
    [System.IO.File]::WriteAllText($Path, $html, $Utf8Bom)
}

# -----------------------------------------------------------------------------
# Main
# -----------------------------------------------------------------------------
$selection = Show-IisAnalysisPicker -InitialLogFile $DefaultLogFile -InitialOutputFolder $DefaultOutputFolder
if ($null -eq $selection) { return }

$LogFile = $selection.LogFile
$ApiKey = $selection.ApiKey
$OutputFolder = $selection.OutputFolder
if (-not (Test-Path -LiteralPath $OutputFolder -PathType Container)) {
    New-Item -Path $OutputFolder -ItemType Directory -Force | Out-Null
}

$CacheFile = Join-Path $OutputFolder $CacheFileName
$stamp = Get-Date -Format 'yyyyMMdd_HHmmss'
$HtmlOutput = Join-Path $OutputFolder "IIS-IP-Reputation-$stamp.html"
$CsvOutput = Join-Path $OutputFolder "IIS-IP-Reputation-$stamp.csv"

$DateFormat = 'yyyy-MM-dd HH:mm:ss'
$Culture = [Globalization.CultureInfo]::InvariantCulture
$IPCounts = @{}
$LatestTimestamp = $null
$StartTime = $null
$EndTime = $null
$LinesRead = 0L
$LinesInPeriod = 0L
$PublicMentions = 0L

if ($selection.Mode -eq 'Custom') {
    $StartTime = $selection.StartTime
    $EndTime = $selection.EndTime
}

Write-Host ''
Write-Host 'Reading selected IIS log backwards...' -ForegroundColor Cyan
Write-Host $LogFile -ForegroundColor Gray

foreach ($Line in [FastReverseLogReader]::ReadLines($LogFile, 1048576)) {
    $LinesRead++

    if ([string]::IsNullOrWhiteSpace($Line) -or $Line.StartsWith('#') -or $Line.Length -lt 19) { continue }

    try {
        $Timestamp = [datetime]::ParseExact($Line.Substring(0, 19), $DateFormat, $Culture)
    }
    catch { continue }

    if ($null -eq $LatestTimestamp) {
        $LatestTimestamp = $Timestamp
        if ($selection.Mode -eq 'Latest') {
            $EndTime = $LatestTimestamp
            $StartTime = $LatestTimestamp.AddMinutes(-$selection.Minutes)
        }
        Write-Host ("Window: {0:yyyy-MM-dd HH:mm:ss} to {1:yyyy-MM-dd HH:mm:ss}" -f $StartTime, $EndTime) -ForegroundColor Yellow
    }

    if ($Timestamp -gt $EndTime) { continue }
    if ($Timestamp -lt $StartTime) { break }

    $LinesInPeriod++
    $lastSpace = $Line.LastIndexOf(' ')
    if ($lastSpace -lt 0 -or $lastSpace -ge ($Line.Length - 1)) { continue }

    foreach ($entry in $Line.Substring($lastSpace + 1).Split(',')) {
        $IP = $entry.Trim().TrimStart('+')
        if (-not (Test-IsPublicIPv4 -Address $IP)) { continue }

        if ($IPCounts.ContainsKey($IP)) { $IPCounts[$IP]++ }
        else { $IPCounts[$IP] = 1L }
        $PublicMentions++
    }

    if (($LinesRead % 100000) -eq 0) {
        Write-Host ("{0:N0} recent lines read..." -f $LinesRead) -ForegroundColor DarkGray
    }
}

if ($null -eq $LatestTimestamp) {
    [System.Windows.Forms.MessageBox]::Show('No valid IIS timestamps were found.', 'No data', 'OK', 'Warning') | Out-Null
    return
}
if ($IPCounts.Count -eq 0) {
    [System.Windows.Forms.MessageBox]::Show('No public IPv4 addresses were found in the selected range.', 'No data', 'OK', 'Information') | Out-Null
    return
}

$Top10 = @($IPCounts.GetEnumerator() | Sort-Object Value -Descending | Select-Object -First 10)

$Cache = @{}
$Today = (Get-Date).Date
if (Test-Path -LiteralPath $CacheFile -PathType Leaf) {
    Import-Csv -LiteralPath $CacheFile | ForEach-Object {
        try {
            $checked = [datetime]$_.CheckedAt
            if ($checked.Date -eq $Today) { $Cache[$_.IPAddress] = $_ }
        }
        catch { }
    }
}

[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
$Headers = @{ Key = $ApiKey; Accept = 'application/json' }
$Results = @()
$lookup = 0

Write-Host ''
if ([string]::IsNullOrWhiteSpace($ApiKey)) {
    Write-Host 'No IP Abuse DB API key supplied. Reputation lookups will be skipped.' -ForegroundColor Yellow
}
else {
    Write-Host 'Checking only the Top 10 IPs with AbuseIPDB...' -ForegroundColor Cyan
}

foreach ($item in $Top10) {
    $lookup++
    $IP = [string]$item.Key
    $Requests = [long]$item.Value
    $Score = $null
    Write-Progress -Activity 'AbuseIPDB lookups' -Status "$lookup of $($Top10.Count): $IP" -PercentComplete (($lookup / $Top10.Count) * 100)

    if ([string]::IsNullOrWhiteSpace($ApiKey)) {
        $Score = $null
    }
    elseif ($Cache.ContainsKey($IP)) {
        $Score = [int]$Cache[$IP].AbuseScore
    }
    else {
        $uri = 'https://api.abuseipdb.com/api/v2/check?ipAddress={0}&maxAgeInDays={1}' -f [Uri]::EscapeDataString($IP), $MaxAgeInDays
        try {
            $response = Invoke-RestMethod -Method Get -Uri $uri -Headers $Headers -ErrorAction Stop
            $Score = [int]$response.data.abuseConfidenceScore
            $Cache[$IP] = [PSCustomObject]@{ IPAddress = $IP; AbuseScore = $Score; CheckedAt = (Get-Date).ToString('o') }
        }
        catch {
            Write-Warning ("AbuseIPDB lookup failed for {0}: {1}" -f $IP, $_.Exception.Message)
        }
    }

    $Results += [PSCustomObject]@{ Requests = $Requests; 'IP Address' = $IP; 'Abuse Score' = $Score }
}
Write-Progress -Activity 'AbuseIPDB lookups' -Completed

if (-not [string]::IsNullOrWhiteSpace($ApiKey) -and $Cache.Count -gt 0) {
    $Cache.Values | Select-Object IPAddress, AbuseScore, CheckedAt | Export-Csv -LiteralPath $CacheFile -NoTypeInformation -Encoding UTF8
}
$Results | Sort-Object Requests -Descending | Select-Object Requests, 'IP Address', 'Abuse Score' | Export-Csv -LiteralPath $CsvOutput -NoTypeInformation -Encoding UTF8

Export-IisIpHtml -Results $Results -Path $HtmlOutput -LogFile $LogFile -StartTime $StartTime -EndTime $EndTime -LinesRead $LinesRead -LinesInPeriod $LinesInPeriod -PublicMentions $PublicMentions -UniqueIPs $IPCounts.Count -AbuseLookupEnabled (-not [string]::IsNullOrWhiteSpace($ApiKey))

Write-Host ''
Write-Host 'Report generated successfully:' -ForegroundColor Green
Write-Host $HtmlOutput -ForegroundColor Cyan
Write-Host ''
$Results | Sort-Object Requests -Descending | Format-Table Requests, 'IP Address', 'Abuse Score' -AutoSize

if ($selection.OpenReport) { Start-Process $HtmlOutput }

```