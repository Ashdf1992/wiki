# Better Event Viewer (HTML)

<p align="center">
    <img src="https://github.com/Ashdf1992/wiki/blob/main/assets/images/WindowsEventReport/WindowsErrorReport.png"/>
</p>

## Powershell
>Open Powershell or Powershell ISE

>Run the Following
```Powershell
# =========================
# NEW (Optional): GUI date/time + duration picker (WinForms)
# =========================
function Show-EventWindowPicker {
    Add-Type -AssemblyName System.Windows.Forms, System.Drawing

    $form = New-Object System.Windows.Forms.Form
    $form.Text = "Event Log Time Window"
    $form.StartPosition = "CenterScreen"
    $form.Size = New-Object System.Drawing.Size(420, 220)
    $form.FormBorderStyle = "FixedDialog"
    $form.MaximizeBox = $false
    $form.MinimizeBox = $false
    $form.Topmost = $true

    $labelDate = New-Object System.Windows.Forms.Label
    $labelDate.Text = "Date"
    $labelDate.Location = New-Object System.Drawing.Point(20, 20)
    $labelDate.AutoSize = $true

    # Date picker (shows a calendar dropdown)
    $datePicker = New-Object System.Windows.Forms.DateTimePicker
    $datePicker.Location = New-Object System.Drawing.Point(140, 16)
    $datePicker.Width = 240
    $datePicker.Format = [System.Windows.Forms.DateTimePickerFormat]::Custom
    $datePicker.CustomFormat = "yyyy-MM-dd"

    $labelTime = New-Object System.Windows.Forms.Label
    $labelTime.Text = "Start time (24h)"
    $labelTime.Location = New-Object System.Drawing.Point(20, 60)
    $labelTime.AutoSize = $true

    # Time picker (up/down selector)
    $timePicker = New-Object System.Windows.Forms.DateTimePicker
    $timePicker.Location = New-Object System.Drawing.Point(140, 56)
    $timePicker.Width = 240
    $timePicker.Format = [System.Windows.Forms.DateTimePickerFormat]::Custom
    $timePicker.CustomFormat = "HH:mm"
    $timePicker.ShowUpDown = $true

    $labelDur = New-Object System.Windows.Forms.Label
    $labelDur.Text = "Duration (minutes ±)"
    $labelDur.Location = New-Object System.Drawing.Point(20, 100)
    $labelDur.AutoSize = $true

    $durationBox = New-Object System.Windows.Forms.NumericUpDown
    $durationBox.Location = New-Object System.Drawing.Point(140, 96)
    $durationBox.Width = 120
    $durationBox.Minimum = 1
    $durationBox.Maximum = 1440
    $durationBox.Value = 60

    $okButton = New-Object System.Windows.Forms.Button
    $okButton.Text = "OK"
    $okButton.Location = New-Object System.Drawing.Point(220, 140)
    $okButton.Width = 75
    $okButton.DialogResult = [System.Windows.Forms.DialogResult]::OK

    $cancelButton = New-Object System.Windows.Forms.Button
    $cancelButton.Text = "Cancel"
    $cancelButton.Location = New-Object System.Drawing.Point(305, 140)
    $cancelButton.Width = 75
    $cancelButton.DialogResult = [System.Windows.Forms.DialogResult]::Cancel

    $form.AcceptButton = $okButton
    $form.CancelButton = $cancelButton

    $form.Controls.AddRange(@(
        $labelDate, $datePicker,
        $labelTime, $timePicker,
        $labelDur, $durationBox,
        $okButton, $cancelButton
    ))

    $result = $form.ShowDialog()
    if ($result -ne [System.Windows.Forms.DialogResult]::OK) { return $null }

    # Combine chosen date + time into one DateTime
    $picked = Get-Date -Year $datePicker.Value.Year -Month $datePicker.Value.Month -Day $datePicker.Value.Day `
                      -Hour $timePicker.Value.Hour -Minute $timePicker.Value.Minute -Second 0

    [pscustomobject]@{
        DateTime = $picked
        DurationMinutes = [int]$durationBox.Value
    }
}

# =========================
# NEW: Choose GUI input (default) or fallback to original prompts
# =========================
$UseGuiPicker = $true   # set to $false if you ever want Read-Host prompts again

if ($UseGuiPicker) {
    $picked = Show-EventWindowPicker
    if ($null -eq $picked) {
        Write-Host "Cancelled by user." -ForegroundColor Yellow
        return
    }

    # Populate your existing variables (no core logic change)
    $DateInput = $picked.DateTime.ToString('yyyy-MM-dd')
    $Date      = $DateInput
    $StartTime = $picked.DateTime.ToString('HH:mm')
    $Duration  = [string]$picked.DurationMinutes
}
else {
    # --- ORIGINAL INPUT PROMPTS (unchanged) ---
    $DateInput = Read-Host "Enter date (yyyy-MM-dd) or press Enter for TODAY'S DATE"
    if ([string]::IsNullOrWhiteSpace($DateInput)) {
        $Date = (Get-Date).ToString('yyyy-MM-dd')
        Write-Host "Using today's date: $Date" -ForegroundColor Green
    }
    else {
        if ($DateInput -notmatch '^\d{4}-\d{2}-\d{2}$') {
            Write-Host "Invalid date format. Please use yyyy-MM-dd (e.g., 2026-02-18)." -ForegroundColor Red
            return
        }
        try {
            [void][datetime]::ParseExact($DateInput, 'yyyy-MM-dd', $null)
            $Date = $DateInput
        }
        catch {
            Write-Host "The date you entered is not a valid calendar date. Please check and try again." -ForegroundColor Red
            return
        }
    }

    $StartTime = Read-Host "Enter start time (hh:mm, 24-hour format)"
    $Duration  = Read-Host "Enter duration to check for before and after the specified time ((60), mm, minute format)"
}

# Validate input
if ($StartTime -notmatch '^\d{2}:\d{2}$') {
    Write-Host "Invalid time format. Please use hh:mm (e.g., 09:30)." -ForegroundColor Red
    return
}

# Convert to DateTime
$SpecifiedTime = [datetime]::ParseExact("$Date $StartTime", 'yyyy-MM-dd HH:mm', $null)

# Calculate time window
$StartDateTime = $SpecifiedTime.AddMinutes(-$Duration)
$EndDateTime   = $SpecifiedTime.AddMinutes($Duration)

# Define logs to query — all events collected, filtering is handled in the HTML report
$AllLogs = @("Application", "System", "Security", "Setup")

Write-Host "`nRetrieving events between $StartDateTime and $EndDateTime..." -ForegroundColor Cyan

# Collect all events
$AllResults = @()

# Query all logs without any level filter
foreach ($LogName in $AllLogs) {
    Write-Host "`n=== $LogName Log Events ===" -ForegroundColor Yellow

    $filter = @{ LogName = $LogName }

    if ($StartDateTime) { $filter['StartTime'] = [DateTime]$StartDateTime }
    if ($EndDateTime)   { $filter['EndTime']   = [DateTime]$EndDateTime   }

    $LogResults = Get-WinEvent -FilterHashtable $filter -ErrorAction SilentlyContinue |
        Select-Object TimeCreated, Id, LevelDisplayName, ProviderName, Message

    if ($LogResults) {
        $LogResults | Format-Table -AutoSize -Wrap
        $AllResults += $LogResults | ForEach-Object {
            [pscustomobject]@{
                Log         = $LogName
                TimeCreated = $_.TimeCreated
                Id          = $_.Id
                Level       = $_.LevelDisplayName
                Provider    = $_.ProviderName
                Message     = $_.Message
            }
        }
    } else {
        Write-Host "No $LogName events found." -ForegroundColor Gray
    }
}

# =========================
# Export HTML Report (AUTOMATIC)
# =========================
function Export-ModernEventHtml {
    param(
        [Parameter(Mandatory)] $Events,
        [Parameter(Mandatory)] [string] $Path,
        [Parameter(Mandatory)] [datetime] $StartDateTime,
        [Parameter(Mandatory)] [datetime] $EndDateTime
    )

    Add-Type -AssemblyName System.Web

    $all = $Events | Sort-Object TimeCreated

    $total = $all.Count
    $logCounts = $all | Group-Object Log | Sort-Object Name

    $logCountBadges = ($logCounts | ForEach-Object {
        $name = $_.Name
        $count = $_.Count
        "<span class='badge'>$name`: $count</span>"
    }) -join " "

    $levelCounts = $all | Group-Object Level | Sort-Object Name

    $levelBadges = ($levelCounts | ForEach-Object {
        $name = $_.Name
        $count = $_.Count
        $cls = ("lvl-" + ($name -replace '\s','').ToLower())
        "<span class='badge $cls'>$name`: $count</span>"
    }) -join " "

    # Build unique provider list for JavaScript
    $uniqueProviders = ($all | Select-Object -ExpandProperty Provider -Unique | Sort-Object) -join '","'
    $providersJson = '["' + $uniqueProviders + '"]'

    $rows = foreach ($e in $all) {
        $msg = [System.Web.HttpUtility]::HtmlEncode([string]$e.Message)
        $provider = [System.Web.HttpUtility]::HtmlEncode([string]$e.Provider)
        $lvl = [System.Web.HttpUtility]::HtmlEncode([string]$e.Level)
        $log = [System.Web.HttpUtility]::HtmlEncode([string]$e.Log)
        $time = $e.TimeCreated.ToString("yyyy-MM-dd HH:mm:ss")
        $id = [System.Web.HttpUtility]::HtmlEncode([string]$e.Id)

        $lvlClass = "lvl-" + (($e.Level -replace '\s','').ToLower())

        @"
<tr data-level="$lvl">
  <td><span class='pill pill-$log'>$log</span></td>
  <td class='mono'>$time</td>
  <td class='mono'>$id</td>
  <td><span class='badge $lvlClass'>$lvl</span></td>
  <td>$provider</td>
  <td>
    <div class='msg-wrapper'>
      <button class='view-btn'>▶ View</button>
      <div class='msg-content' style='display:none;'>
        <button class='copy-btn' title='Copy message'>📋 Copy</button>
        <pre class='msg'>$msg</pre>
      </div>
    </div>
  </td>
</tr>
"@
    }

    $startS = $StartDateTime.ToString("yyyy-MM-dd HH:mm:ss")
    $endS   = $EndDateTime.ToString("yyyy-MM-dd HH:mm:ss")
    $generated = (Get-Date).ToString("yyyy-MM-dd HH:mm:ss")

    # Use single-quote here-string to avoid PowerShell variable expansion in JavaScript
    $htmlContent = @'
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Windows Event Report (START_TIME_PLACEHOLDER → END_TIME_PLACEHOLDER)</title>
<style>
@import url('https://fonts.googleapis.com/css2?family=Inter:wght@300;400;500;600;700&display=swap');

:root{
  --bg:#0b1220; --card:#111a2e; --muted:#9fb0d0; --text:#e7eefc;
  --accent:#6ea8fe; --border:rgba(255,255,255,.08);
  --ok:#36d399; --warn:#fbbf24; --err:#f87171; --crit:#fb7185;
  --shadow: 0 20px 50px rgba(0,0,0,.45);
}

[data-theme="light"] {
  --bg:#f5f7fa; --card:#ffffff; --muted:#64748b; --text:#0f172a;
  --accent:#3b82f6; --border:rgba(0,0,0,.08);
  --shadow: 0 20px 50px rgba(0,0,0,.08);
}

*{box-sizing:border-box}

body{
  margin:0;
  background: linear-gradient(180deg,#070b14 0%, #0b1220 60%, #070b14 100%);
  color:var(--text);
  font-family: "Inter", "Segoe UI", system-ui, -apple-system, Arial, sans-serif;
  position: relative;
  overflow-x: hidden;
}

/* Animated gradient background */
body::before {
  content: '';
  position: fixed;
  top: -50%;
  left: -50%;
  width: 200%;
  height: 200%;
  background: radial-gradient(circle at 20% 50%, rgba(110,168,254,.08) 0%, transparent 50%),
              radial-gradient(circle at 80% 80%, rgba(251,113,133,.06) 0%, transparent 50%);
  animation: gradientShift 20s ease infinite;
  pointer-events: none;
  z-index: 0;
}

@keyframes gradientShift {
  0%, 100% { transform: translate(0, 0) rotate(0deg); }
  50% { transform: translate(-5%, -5%) rotate(5deg); }
}

/* Noise texture overlay */
body::after {
  content: '';
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-image: url("data:image/svg+xml,%3Csvg viewBox='0 0 400 400' xmlns='http://www.w3.org/2000/svg'%3E%3Cfilter id='noiseFilter'%3E%3CfeTurbulence type='fractalNoise' baseFrequency='0.9' numOctaves='4' stitchTiles='stitch'/%3E%3C/filter%3E%3Crect width='100%25' height='100%25' filter='url(%23noiseFilter)' opacity='0.03'/%3E%3C/svg%3E");
  pointer-events: none;
  z-index: 1;
  opacity: 0.4;
}

.container{
  max-width:1200px;
  margin:32px auto;
  padding:0 16px;
  position: relative;
  z-index: 2;
}

/* Parallax header */
.header{
  display:flex;
  gap:16px;
  align-items:flex-start;
  justify-content:space-between;
  flex-wrap:wrap;
  transform: translateY(0);
  transition: transform 0.3s ease-out;
}

.h-title{font-size:22px;margin:0}
.sub{color:var(--muted);margin-top:6px;font-size:13px}

.card{
  background: rgba(17,26,46,.85);
  backdrop-filter: blur(18px);
  border:1px solid var(--border);
  border-radius:14px;
  box-shadow:var(--shadow);
  padding:18px;
  position: relative;
  overflow: hidden;
}

[data-theme="light"] .card {
  background: rgba(255,255,255,.95);
}

/* Gradient header */
.header .card {
  background: linear-gradient(145deg, rgba(20,30,55,.95), rgba(12,18,35,.95));
  border: 1px solid rgba(110,168,254,.15);
}

[data-theme="light"] .header .card {
  background: linear-gradient(145deg, rgba(255,255,255,.98), rgba(245,247,250,.98));
}

.kpis{display:flex;gap:12px;flex-wrap:wrap;margin-top:10px}
.kpi{
  background:rgba(255,255,255,.03);
  border:1px solid var(--border);
  border-radius:12px;
  padding:10px 12px;
  min-width:160px;
  transition: transform .2s ease, box-shadow .2s ease;
}

.kpi:hover {
  transform: translateY(-2px);
  box-shadow: 0 8px 20px rgba(0,0,0,.2);
}

.kpi b{display:block;font-size:18px}
.kpi span{color:var(--muted);font-size:12px}

.badges{display:flex;gap:8px;flex-wrap:wrap;margin-top:10px}
.badge{
  display:inline-flex;
  align-items:center;
  gap:6px;
  padding:4px 10px;
  border-radius:999px;
  font-size:12px;
  border:1px solid var(--border);
  background:rgba(255,255,255,.04);
  transition: transform .12s ease, box-shadow .12s ease;
}

.badge:hover { transform: translateY(-1px); }

/* Improved badge colors with glow */
.lvl-critical{
  background:rgba(251,113,133,.16);
  border-color:rgba(251,113,133,.35);
  box-shadow: 0 0 12px rgba(251,113,133,.25);
}
.lvl-error{
  background:rgba(248,113,113,.16);
  border-color:rgba(248,113,113,.35);
  box-shadow: 0 0 12px rgba(248,113,113,.25);
}
.lvl-warning{
  background:rgba(251,191,36,.16);
  border-color:rgba(251,191,36,.35);
  box-shadow: 0 0 12px rgba(251,191,36,.25);
}

/* Sticky search toolbar */
.toolbar{
  display:flex;
  gap:10px;
  align-items:center;
  justify-content:space-between;
  flex-wrap:wrap;
  margin:16px 0;
  position: sticky;
  top: 0;
  z-index: 1000;
  background: rgba(11,18,32,.85);
  backdrop-filter: blur(18px);
  padding: 14px;
  border-radius: 14px;
  border: 1px solid var(--border);
  box-shadow: 0 8px 30px rgba(0,0,0,.35);
}

[data-theme="light"] .toolbar {
  background: rgba(255,255,255,.85);
}

/* Sticky filter toolbar (floats below search) */
.filter-toolbar {
  position: sticky;
  top: 70px;
  z-index: 900;
  backdrop-filter: blur(18px);
  background: rgba(15, 23, 42, 0.75);
  border: 1px solid rgba(255,255,255,0.08);
  border-radius: 14px;
  padding: 14px;
  margin-bottom: 20px;
  box-shadow: 0 8px 30px rgba(0,0,0,0.35);
  transition: transform 0.25s ease, box-shadow 0.25s ease;
}

[data-theme="light"] .filter-toolbar {
  background: rgba(255,255,255,.75);
}

.search{flex:1;min-width:260px;display:flex;gap:10px;align-items:center;flex-wrap:wrap}

input[type="search"]{
  width:100%;
  padding:10px 12px;
  border-radius:10px;
  border:1px solid var(--border);
  background:rgba(0,0,0,.22);
  color:var(--text);
  outline:none;
  transition: border-color .2s ease, box-shadow .2s ease;
}

[data-theme="light"] input[type="search"] {
  background: rgba(0,0,0,.04);
}

input[type="search"]:focus {
  border-color: rgba(110,168,254,.55);
  box-shadow: 0 0 0 3px rgba(110,168,254,.18);
}

select{
  padding:10px 12px;
  border-radius:10px;
  border:1px solid var(--border);
  background:rgba(0,0,0,.22);
  color:var(--text);
  outline:none;
  transition: border-color .2s ease, box-shadow .2s ease;
}

[data-theme="light"] select {
  background: rgba(0,0,0,.04);
}

select:focus {
  border-color: rgba(110,168,254,.55);
  box-shadow: 0 0 0 3px rgba(110,168,254,.18);
}

/* Button styles with hover glow */
button {
  padding: 10px 16px;
  border-radius: 10px;
  border: 1px solid var(--border);
  background: rgba(110,168,254,.12);
  color: var(--accent);
  font-size: 13px;
  font-weight: 500;
  cursor: pointer;
  transition: all .2s ease;
  font-family: inherit;
}

button:hover {
  background: rgba(110,168,254,.22);
  box-shadow: 0 0 20px rgba(110,168,254,.3);
  transform: translateY(-1px);
}

button:active {
  transform: translateY(0);
}

.controls {
  display: flex;
  gap: 8px;
  align-items: center;
}

/* Theme toggle */
#themeToggle {
  padding: 8px 12px;
  font-size: 16px;
}

/* Density toggle */
#densityToggle {
  padding: 8px 12px;
  font-size: 12px;
}

body.compact tbody td {
  padding: 6px 10px;
  font-size: 11px;
}

body.compact .mono {
  font-size: 11px;
}

body.compact .badge, body.compact .pill {
  padding: 2px 8px;
  font-size: 10px;
}

table{width:100%; border-collapse:separate; border-spacing:0; overflow:hidden}

thead th{
  text-align:left;
  font-size:12px;
  color:var(--muted);
  padding:12px 10px;
  border-bottom:1px solid var(--border);
  position:sticky;
  top:0;
  background:rgba(17,26,46,.96);
  backdrop-filter: blur(8px);
  z-index: 3;
}

[data-theme="light"] thead th {
  background: rgba(255,255,255,.96);
}

tbody td{
  padding:12px 10px;
  border-bottom:1px solid var(--border);
  vertical-align:top;
  transition: all .2s ease;
}

/* Smooth hover row elevation */
tbody tr{
  transition: all .25s ease;
  cursor: pointer;
}

tbody tr:hover{
  background:rgba(255,255,255,.04);
  transform: translateY(-2px);
  box-shadow: 0 8px 18px rgba(0,0,0,.3);
}

[data-theme="light"] tbody tr:hover {
  background: rgba(0,0,0,.02);
  box-shadow: 0 8px 18px rgba(0,0,0,.08);
}

/* Row severity side bar */
tbody tr[data-level="Critical"] {
  border-left: 4px solid #fb7185;
}

tbody tr[data-level="Error"] {
  border-left: 4px solid #f87171;
}

tbody tr[data-level="Warning"] {
  border-left: 4px solid #fbbf24;
}

.mono{
  font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, "Liberation Mono", monospace;
  font-size:12px;
}

.pill{
  display:inline-flex;
  align-items:center;
  padding:4px 10px;
  border-radius:999px;
  border:1px solid var(--border);
  background:rgba(255,255,255,.04);
  font-size:12px;
  transition: transform .12s ease;
}

.pill:hover { transform: translateY(-1px); }

.pill-Application{border-color:rgba(110,168,254,.35); background:rgba(110,168,254,.12)}
.pill-System{border-color:rgba(54,211,153,.35); background:rgba(54,211,153,.12)}
.pill-Security{border-color:rgba(251,113,133,.35); background:rgba(251,113,133,.12)}
.pill-Setup{border-color:rgba(251,191,36,.35); background:rgba(251,191,36,.12)}

/* Message expansion */
.msg-wrapper {
  position: relative;
}

.view-btn {
  padding: 6px 12px;
  font-size: 12px;
  background: rgba(110,168,254,.12);
  border: 1px solid rgba(110,168,254,.3);
  color: var(--accent);
  cursor: pointer;
  border-radius: 6px;
  transition: all .2s ease;
}

.view-btn:hover {
  background: rgba(110,168,254,.22);
}

.msg-content {
  margin-top: 10px;
  position: relative;
}

/* Smooth expand animation */
.msg-content.expanded {
  animation: expand .25s ease-out;
}

@keyframes expand {
  from { opacity: 0; transform: translateY(-6px); }
  to { opacity: 1; transform: translateY(0); }
}

.copy-btn {
  position: absolute;
  top: 10px;
  right: 10px;
  padding: 6px 10px;
  font-size: 11px;
  background: rgba(110,168,254,.12);
  border: 1px solid rgba(110,168,254,.3);
  z-index: 1;
}

.copy-btn:hover {
  background: rgba(110,168,254,.22);
}

pre.msg{
  white-space:pre-wrap;
  margin:0;
  padding:10px 10px 10px 10px;
  border-radius:10px;
  border:1px solid var(--border);
  background:rgba(0,0,0,.25);
  position: relative;
}

[data-theme="light"] pre.msg {
  background: rgba(0,0,0,.04);
}

/* Smart highlighting */
mark {
  background: rgba(251,191,36,.35);
  color: var(--text);
  padding: 2px 4px;
  border-radius: 3px;
  font-weight: 500;
}

.footer{margin-top:14px;color:var(--muted);font-size:12px}

.is-hidden { display: none !important; }

/* Fade table on filter */
body.is-filtering tbody {
  opacity: .6;
  transition: opacity .2s ease;
}

@keyframes fadeUp {
  from { opacity: 0; transform: translateY(8px); }
  to   { opacity: 1; transform: translateY(0); }
}

.card { animation: fadeUp .35s ease-out both; }
tbody tr { animation: fadeUp .20s ease-out both; }

@media (max-width: 860px) {
  .toolbar { gap: 12px; }
  .search { flex-wrap: wrap; }
  .search > * { flex: 1 1 220px; }
  .footer { width: 100%; }
}

@media (max-width: 640px) {
  .kpis .kpi { min-width: 140px; }
  thead th:nth-child(5), tbody td:nth-child(5) { display: none; }
  .card { padding: 14px; }
}

.card { overflow: visible; }
.card > div { overflow-x: auto; }
table { min-width: 900px; display: block; overflow-x: auto; }

@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation: none !important;
    transition: none !important;
    scroll-behavior: auto !important;
  }
}

th.sortable {
  cursor: pointer;
  user-select: none;
  position: relative;
  padding-right: 26px;
}

th.sortable .sort-ind {
  position: absolute;
  right: 10px;
  top: 50%;
  transform: translateY(-52%);
  opacity: .65;
  font-size: 11px;
}

th.sortable.sorted-asc .sort-ind::after { content: "▲"; }
th.sortable.sorted-desc .sort-ind::after { content: "▼"; }

thead tr.col-filters th {
  padding: 8px 10px;
  border-bottom: 1px solid var(--border);
  background: rgba(17,26,46,.96);
  position: sticky;
  top: 44px;
  z-index: 3;
}

[data-theme="light"] thead tr.col-filters th {
  background: rgba(255,255,255,.96);
}

thead tr.col-filters input,
thead tr.col-filters select {
  width: 100%;
  padding: 8px 10px;
  border-radius: 10px;
  border: 1px solid var(--border);
  background: rgba(0,0,0,.22);
  color: var(--text);
  outline: none;
  font-size: 12px;
}

[data-theme="light"] thead tr.col-filters input,
[data-theme="light"] thead tr.col-filters select {
  background: rgba(0,0,0,.04);
}

thead tr.col-filters .range {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 8px;
}

input[type="datetime-local"] {
  width: 100%;
  padding: 8px 10px;
  border-radius: 10px;
  border: 1px solid var(--border);
  background: rgba(0,0,0,.22);
  color: var(--text);
  outline: none;
  font-size: 12px;
  color-scheme: dark;
}

[data-theme="light"] input[type="datetime-local"] {
  background: rgba(0,0,0,.04);
  color-scheme: light;
}

input[type="datetime-local"]::-webkit-calendar-picker-indicator {
  filter: invert(1);
  cursor: pointer;
}

[data-theme="light"] input[type="datetime-local"]::-webkit-calendar-picker-indicator {
  filter: invert(0);
}

/* Provider dropdown */
.provider-select-wrapper {
  position: relative;
  width: 100%;
}

.provider-select-btn {
  width: 100%;
  padding: 8px 10px;
  border-radius: 10px;
  border: 1px solid var(--border);
  background: rgba(0,0,0,.22);
  color: var(--text);
  font-size: 12px;
  text-align: left;
  cursor: pointer;
  display: flex;
  justify-content: space-between;
  align-items: center;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  transition: border-color .2s ease;
}

[data-theme="light"] .provider-select-btn {
  background: rgba(0,0,0,.04);
}

.provider-select-btn:hover {
  border-color: rgba(110,168,254,.35);
}

/* Provider dropdown slide animation */
.provider-dropdown {
  position: fixed;
  width: 300px;
  max-height: 400px;
  margin-top: 6px;
  background: rgba(17,26,46,.98);
  border: 1px solid var(--border);
  border-radius: 10px;
  overflow-y: auto;
  z-index: 9999;
  display: none;
  box-shadow: var(--shadow);
  transform: translateY(-6px);
  opacity: 0;
  transition: all .18s ease;
}

[data-theme="light"] .provider-dropdown {
  background: rgba(255,255,255,.98);
}

.provider-dropdown.show {
  display: block;
  transform: translateY(0);
  opacity: 1;
}

.provider-dropdown-header {
  position: sticky;
  top: 0;
  background: rgba(17,26,46,.98);
  padding: 8px;
  border-bottom: 1px solid var(--border);
  display: flex;
  gap: 6px;
  z-index: 1;
}

[data-theme="light"] .provider-dropdown-header {
  background: rgba(255,255,255,.98);
}

.provider-dropdown-header button {
  flex: 1;
  padding: 6px 10px;
  border-radius: 6px;
  border: 1px solid var(--border);
  background: rgba(0,0,0,.22);
  color: var(--accent);
  font-size: 11px;
  cursor: pointer;
}

[data-theme="light"] .provider-dropdown-header button {
  background: rgba(0,0,0,.04);
}

.provider-dropdown-header button:hover {
  background: rgba(110,168,254,.15);
}

.provider-option {
  display: flex;
  align-items: center;
  padding: 5px 12px;
  font-size: 12px;
  cursor: pointer;
  gap: 10px;
}

.provider-option:hover {
  background: rgba(255,255,255,.05);
}

[data-theme="light"] .provider-option:hover {
  background: rgba(0,0,0,.03);
}

.provider-option input[type="checkbox"] {
  margin: 0;
  cursor: pointer;
  flex-shrink: 0;
  width: 16px;
  height: 16px;
}

.provider-option label {
  flex: 1;
  cursor: pointer;
  margin: 0;
  line-height: 1.4;
  word-break: break-word;
}

/* Custom scrollbars */
::-webkit-scrollbar {
  width: 10px;
  height: 10px;
}

::-webkit-scrollbar-track {
  background: rgba(0,0,0,.1);
  border-radius: 10px;
}

::-webkit-scrollbar-thumb {
  background: rgba(110,168,254,.3);
  border-radius: 10px;
  transition: background .2s ease;
}

::-webkit-scrollbar-thumb:hover {
  background: rgba(110,168,254,.5);
}

[data-theme="light"] ::-webkit-scrollbar-thumb {
  background: rgba(59,130,246,.3);
}

[data-theme="light"] ::-webkit-scrollbar-thumb:hover {
  background: rgba(59,130,246,.5);
}

/* Result count styling */
#resultCount {
  text-align: center;
  padding: 10px;
  font-size: 13px;
  color: var(--accent);
  font-weight: 500;
}

/* Floating scroll-to-top button */
#scrollToTop {
  position: fixed;
  bottom: 30px;
  right: 30px;
  width: 50px;
  height: 50px;
  border-radius: 50%;
  background: rgba(110,168,254,.2);
  border: 1px solid rgba(110,168,254,.4);
  color: var(--accent);
  font-size: 20px;
  cursor: pointer;
  z-index: 1000;
  opacity: 0;
  transform: translateY(20px);
  transition: all .3s ease;
  box-shadow: 0 8px 20px rgba(0,0,0,.3);
}

#scrollToTop.visible {
  opacity: 1;
  transform: translateY(0);
}

#scrollToTop:hover {
  background: rgba(110,168,254,.35);
  box-shadow: 0 0 30px rgba(110,168,254,.4);
  transform: translateY(-3px);
}
</style>
</head>
<body>
  <div class="container">
    <div class="header">
      <div class="card" style="flex:1">
        <h1 class="h-title">Windows Event Report</h1>
        <div class="sub">Window: <span class="mono">START_TIME_PLACEHOLDER</span> → <span class="mono">END_TIME_PLACEHOLDER</span> • Generated: <span class="mono">GENERATED_TIME_PLACEHOLDER</span></div>
        <div class="kpis">
          <div class="kpi"><b>TOTAL_COUNT_PLACEHOLDER</b><span>Total events</span></div>
          LOG_COUNT_BADGES_PLACEHOLDER
        </div>
        <div class="badges">LEVEL_BADGES_PLACEHOLDER</div>
      </div>
    </div>

    <div class="toolbar">
      <div class="search">
        <input id="q" type="search" placeholder="Search messages, provider, ID..." />
      </div>
      <div class="controls">
        <button id="clearFilters">🗑️ Clear Filters</button>
        <button id="exportFiltered">💾 Export Filtered</button>
        <button id="densityToggle">📊 Compact</button>
        <button id="themeToggle">🌙</button>
      </div>
    </div>

    <div class="card">
      <table id="events">
        <thead>
          <tr>
            <th>Log</th>
            <th>TimeCreated</th>
            <th>ID</th>
            <th>Level</th>
            <th>Provider</th>
            <th>Message</th>
          </tr>
        </thead>
        <tbody>
          ROWS_PLACEHOLDER
        </tbody>
      </table>
      <div id="resultCount" class="footer" style="margin-top:12px;">
        Showing 0 of 0 events
      </div>
    </div>
  </div>

  <button id="scrollToTop" title="Scroll to top">↑</button>

<script>
const UNIQUE_PROVIDERS = PROVIDERS_JSON_PLACEHOLDER;

(function () {
  const table = document.getElementById('events');
  const thead = table.querySelector('thead');
  const headerRow = thead.querySelector('tr');
  const tbody = table.querySelector('tbody');
  const resultCountEl = document.getElementById('resultCount');

  function debounce(fn, wait) {
    let t;
    return (...args) => {
      clearTimeout(t);
      t = setTimeout(() => fn(...args), wait);
    };
  }

  function parseUtcFromYmdHms(s) {
    const m = String(s || '').match(/^(\d{4})-(\d{2})-(\d{2})\s+(\d{2}):(\d{2}):(\d{2})$/);
    if (!m) return NaN;
    return Date.UTC(+m[1], (+m[2]) - 1, +m[3], +m[4], +m[5], +m[6]);
  }

  function parseLocalDateTime(s) {
    if (!s) return NaN;
    const d = new Date(s);
    return isNaN(d.getTime()) ? NaN : d.getTime();
  }

  function naturalCompare(a, b) {
    return a.localeCompare(b, undefined, { numeric: true, sensitivity: 'base' });
  }

  let rows = Array.from(tbody.querySelectorAll('tr')).map((tr) => {
    const cells = tr.children;

    const log = (cells[0]?.innerText || '').trim();
    const timeStr = (cells[1]?.innerText || '').trim();
    const idStr = (cells[2]?.innerText || '').trim();
    const level = (cells[3]?.innerText || '').trim();
    const provider = (cells[4]?.innerText || '').trim();

    const fullText = (tr.textContent || '').toLowerCase();

    return {
      el: tr,
      log,
      timeStr,
      timeMs: parseUtcFromYmdHms(timeStr),
      id: parseInt(idStr, 10),
      idStr,
      level,
      provider,
      fullText,
      originalText: tr.textContent
    };
  });

  const filterRow = document.createElement('tr');
  filterRow.className = 'col-filters';

  const state = {
    global: '',
    log: '',
    level: '',
    timeFrom: '',
    timeTo: '',
    idList: '',
    providerSet: new Set(),
  };

  const uniq = (arr) => Array.from(new Set(arr)).sort(naturalCompare);
  const logOptions = uniq(rows.map(r => r.log).filter(Boolean));
  const lvlOptions = uniq(rows.map(r => r.level).filter(Boolean));

  function makeSelect(options, placeholder) {
    const sel = document.createElement('select');
    const opt0 = document.createElement('option');
    opt0.value = '';
    opt0.textContent = placeholder;
    sel.appendChild(opt0);
    options.forEach(v => {
      const o = document.createElement('option');
      o.value = v;
      o.textContent = v;
      sel.appendChild(o);
    });
    return sel;
  }

  function makeInput(ph, type = 'text') {
    const inp = document.createElement('input');
    inp.type = type;
    inp.placeholder = ph;
    return inp;
  }

  // Log filter (select)
  {
    const th = document.createElement('th');
    const sel = makeSelect(logOptions, 'All logs');
    sel.addEventListener('change', () => { state.log = sel.value; applyFilterChunked(); });
    th.appendChild(sel);
    filterRow.appendChild(th);
  }

  // TimeCreated filter (datetime-local inputs)
  {
    const th = document.createElement('th');
    const wrap = document.createElement('div');
    wrap.className = 'range';
    const from = makeInput('', 'datetime-local');
    const to = makeInput('', 'datetime-local');
    from.addEventListener('change', () => { state.timeFrom = from.value; applyFilterChunked(); });
    to.addEventListener('change', () => { state.timeTo = to.value; applyFilterChunked(); });
    wrap.appendChild(from);
    wrap.appendChild(to);
    th.appendChild(wrap);
    filterRow.appendChild(th);
  }

  // ID filter (comma-separated list)
  {
    const th = document.createElement('th');
    const inp = makeInput('IDs e.g. 2240, 2290');
    inp.addEventListener('input', debounce(() => { state.idList = inp.value.trim(); applyFilterChunked(); }, 180));
    th.appendChild(inp);
    filterRow.appendChild(th);
  }

  // Level filter (select)
  {
    const th = document.createElement('th');
    const sel = makeSelect(lvlOptions, 'All levels');
    sel.addEventListener('change', () => { state.level = sel.value; applyFilterChunked(); });
    th.appendChild(sel);
    filterRow.appendChild(th);
  }

  // Provider filter (multi-select dropdown)
  {
    const th = document.createElement('th');
    const wrapper = document.createElement('div');
    wrapper.className = 'provider-select-wrapper';

    const btn = document.createElement('button');
    btn.className = 'provider-select-btn';
    btn.innerHTML = '<span>Filter provider...</span><span>▼</span>';

    const dropdown = document.createElement('div');
    dropdown.className = 'provider-dropdown';

    const dropdownHeader = document.createElement('div');
    dropdownHeader.className = 'provider-dropdown-header';

    const selectAllBtn = document.createElement('button');
    selectAllBtn.textContent = 'Select All';
    selectAllBtn.addEventListener('click', (e) => {
      e.stopPropagation();
      const checkboxes = dropdown.querySelectorAll('input[type="checkbox"]');
      checkboxes.forEach(cb => {
        cb.checked = true;
        state.providerSet.add(cb.value);
      });
      updateProviderButtonText();
      applyFilterChunked();
    });

    const clearAllBtn = document.createElement('button');
    clearAllBtn.textContent = 'Clear All';
    clearAllBtn.addEventListener('click', (e) => {
      e.stopPropagation();
      const checkboxes = dropdown.querySelectorAll('input[type="checkbox"]');
      checkboxes.forEach(cb => {
        cb.checked = false;
      });
      state.providerSet.clear();
      updateProviderButtonText();
      applyFilterChunked();
    });

    dropdownHeader.appendChild(selectAllBtn);
    dropdownHeader.appendChild(clearAllBtn);
    dropdown.appendChild(dropdownHeader);

    UNIQUE_PROVIDERS.forEach(provider => {
      const option = document.createElement('div');
      option.className = 'provider-option';

      const checkbox = document.createElement('input');
      checkbox.type = 'checkbox';
      checkbox.value = provider;
      checkbox.id = 'provider-' + provider.replace(/[^a-zA-Z0-9]/g, '-');

      const label = document.createElement('label');
      label.htmlFor = checkbox.id;
      label.textContent = provider;

      checkbox.addEventListener('change', () => {
        if (checkbox.checked) {
          state.providerSet.add(provider);
        } else {
          state.providerSet.delete(provider);
        }
        updateProviderButtonText();
        applyFilterChunked();
      });

      option.appendChild(checkbox);
      option.appendChild(label);
      dropdown.appendChild(option);
    });

    function updateProviderButtonText() {
      const count = state.providerSet.size;
      const span = btn.querySelector('span:first-child');
      if (count === 0) {
        span.textContent = 'Filter provider...';
      } else if (count === 1) {
        const providerName = Array.from(state.providerSet)[0];
        span.textContent = providerName.length > 30 ? providerName.substring(0, 27) + '...' : providerName;
      } else {
        span.textContent = count + ' providers selected';
      }
    }

    function positionDropdown() {
      const rect = btn.getBoundingClientRect();
      dropdown.style.top = (rect.bottom + window.scrollY + 6) + 'px';
      dropdown.style.left = rect.left + 'px';
    }

    btn.addEventListener('click', (e) => {
      e.stopPropagation();
      const isShowing = dropdown.classList.contains('show');

      if (!isShowing) {
        positionDropdown();
        dropdown.classList.add('show');
      } else {
        dropdown.classList.remove('show');
      }
    });

    window.addEventListener('scroll', () => {
      if (dropdown.classList.contains('show')) {
        positionDropdown();
      }
    });

    window.addEventListener('resize', () => {
      if (dropdown.classList.contains('show')) {
        positionDropdown();
      }
    });

    document.addEventListener('click', (e) => {
      if (!wrapper.contains(e.target) && !dropdown.contains(e.target)) {
        dropdown.classList.remove('show');
      }
    });

    wrapper.appendChild(btn);
    document.body.appendChild(dropdown);
    th.appendChild(wrapper);
    filterRow.appendChild(th);
  }

  // Message filter hint
  {
    const th = document.createElement('th');
    const hint = document.createElement('div');
    hint.style.color = 'var(--muted)';
    hint.style.fontSize = '12px';
    hint.textContent = 'Click row to expand';
    th.appendChild(hint);
    filterRow.appendChild(th);
  }

  thead.appendChild(filterRow);

  const q = document.getElementById('q');

  if (q) q.addEventListener('input', debounce(() => { state.global = (q.value || '').toLowerCase(); applyFilterChunked(); }, 180));

  // Click row to expand
  tbody.addEventListener('click', (e) => {
    const row = e.target.closest('tr');
    if (!row) return;

    const viewBtn = row.querySelector('.view-btn');
    const msgContent = row.querySelector('.msg-content');

    if (!viewBtn || !msgContent) return;

    // Don't toggle if clicking copy button
    if (e.target.classList.contains('copy-btn')) return;

    const isExpanded = msgContent.style.display === 'block';

    if (isExpanded) {
      msgContent.style.display = 'none';
      viewBtn.textContent = '▶ View';
    } else {
      msgContent.style.display = 'block';
      msgContent.classList.add('expanded');
      viewBtn.textContent = '▼ Hide';
    }
  });

  // Copy button functionality
  tbody.addEventListener('click', (e) => {
    if (e.target.classList.contains('copy-btn')) {
      e.stopPropagation();
      const pre = e.target.nextElementSibling;
      if (pre) {
        navigator.clipboard.writeText(pre.textContent).then(() => {
          const originalText = e.target.textContent;
          e.target.textContent = '✓ Copied!';
          setTimeout(() => {
            e.target.textContent = originalText;
          }, 2000);
        });
      }
    }
  });

  // Smart highlighting function
  function highlightText(text, query) {
    if (!query) return text;
    const regex = new RegExp('(' + query.replace(/[.*+?^${}()|[\]\\]/g, '\\$&') + ')', 'gi');
    return text.replace(regex, '<mark>$1</mark>');
  }

  let filterRaf = null;
  let currentCount = 0;

  // Animated filter result count
  function animateCount(target) {
    const duration = 300;
    const start = currentCount;
    const startTime = performance.now();

    function step(currentTime) {
      const elapsed = currentTime - startTime;
      const progress = Math.min(elapsed / duration, 1);

      const easeOut = 1 - Math.pow(1 - progress, 3);
      const current = Math.round(start + (target - start) * easeOut);

      resultCountEl.textContent = 'Showing ' + current + ' of ' + rows.length + ' events';

      if (progress < 1) {
        requestAnimationFrame(step);
      } else {
        currentCount = target;
      }
    }

    requestAnimationFrame(step);
  }

  function applyFilterChunked() {
    if (filterRaf) cancelAnimationFrame(filterRaf);

    document.body.classList.add('is-filtering');

    const CHUNK = 350;
    let i = 0;
    let visibleCount = 0;

    const timeFromMs = state.timeFrom ? parseLocalDateTime(state.timeFrom) : NaN;
    const timeToMs = state.timeTo ? parseLocalDateTime(state.timeTo) : NaN;

    const idSet = (state.idList || '')
      .split(',')
      .map(s => s.trim())
      .filter(s => s.length)
      .map(s => parseInt(s, 10))
      .filter(n => Number.isFinite(n));
    const idFilterActive = idSet.length > 0;
    const idLookup = new Set(idSet);

    const providerFilterActive = state.providerSet.size > 0;

    function step() {
      const end = Math.min(i + CHUNK, rows.length);
      for (; i < end; i++) {
        const r = rows[i];

        const okGlobal = !state.global || r.fullText.indexOf(state.global) !== -1;
        const okLog = !state.log || r.log === state.log;
        const okLvl = !state.level || r.level === state.level;

        const okProvider = !providerFilterActive || state.providerSet.has(r.provider);

        const okTimeFrom = isNaN(timeFromMs) || (!isNaN(r.timeMs) && r.timeMs >= timeFromMs);
        const okTimeTo   = isNaN(timeToMs)   || (!isNaN(r.timeMs) && r.timeMs <= timeToMs);

        const okId = !idFilterActive || (Number.isFinite(r.id) && idLookup.has(r.id));

        const show = okGlobal && okLog && okLvl && okProvider && okTimeFrom && okTimeTo && okId;
        r.el.classList.toggle('is-hidden', !show);

        // Apply smart highlighting
        if (show && state.global) {
          const cells = r.el.querySelectorAll('td');
          cells.forEach((cell, idx) => {
            if (idx === 5) return; // Skip message column (has complex structure)
            const originalText = cell.textContent;
            cell.innerHTML = highlightText(originalText, state.global);
          });
        } else if (!state.global) {
          // Remove highlighting when no search
          const cells = r.el.querySelectorAll('td');
          cells.forEach((cell, idx) => {
            if (idx === 5) return;
            const marks = cell.querySelectorAll('mark');
            marks.forEach(mark => {
              mark.replaceWith(mark.textContent);
            });
          });
        }

        if (show) visibleCount++;
      }

      if (i < rows.length) {
        filterRaf = requestAnimationFrame(step);
      } else {
        filterRaf = null;
        document.body.classList.remove('is-filtering');
        animateCount(visibleCount);
      }
    }

    filterRaf = requestAnimationFrame(step);
  }

  const colDefs = [
    { name: 'Log',        key: r => r.log,     cmp: (a,b)=>naturalCompare(a.log,b.log) },
    { name: 'TimeCreated',key: r => r.timeMs,  cmp: (a,b)=>(a.timeMs - b.timeMs) || naturalCompare(a.timeStr,b.timeStr) },
    { name: 'ID',         key: r => r.id,      cmp: (a,b)=>(a.id - b.id) || naturalCompare(a.idStr,b.idStr) },
    { name: 'Level',      key: r => r.level,   cmp: (a,b)=>naturalCompare(a.level,b.level) },
    { name: 'Provider',   key: r => r.provider,cmp: (a,b)=>naturalCompare(a.provider,b.provider) },
    { name: 'Message',    key: r => r.fullText,cmp: (a,b)=>naturalCompare(a.fullText,b.fullText), noSort: true }
  ];

  let sortState = { index: -1, dir: 1 };
  const ths = Array.from(headerRow.querySelectorAll('th'));

  ths.forEach((th, idx) => {
    const def = colDefs[idx];
    if (!def || def.noSort) return;

    th.classList.add('sortable');
    const ind = document.createElement('span');
    ind.className = 'sort-ind';
    th.appendChild(ind);

    th.addEventListener('click', () => {
      const same = (sortState.index === idx);
      sortState.index = idx;
      sortState.dir = same ? (sortState.dir * -1) : 1;

      ths.forEach(h => h.classList.remove('sorted-asc','sorted-desc'));
      th.classList.add(sortState.dir === 1 ? 'sorted-asc' : 'sorted-desc');

      sortRowsChunked(def, sortState.dir);
    });
  });

  let sortRaf = null;
  function sortRowsChunked(def, dir) {
    if (sortRaf) cancelAnimationFrame(sortRaf);

    const decorated = rows.map((r, i) => ({ r, i }));
    decorated.sort((A, B) => {
      const base = def.cmp(A.r, B.r);
      if (base !== 0) return base * dir;
      return (A.i - B.i);
    });

    rows = decorated.map(x => x.r);

    const CHUNK = 400;
    let i = 0;

    function step() {
      const frag = document.createDocumentFragment();
      const end = Math.min(i + CHUNK, rows.length);
      for (; i < end; i++) frag.appendChild(rows[i].el);
      tbody.appendChild(frag);

      if (i < rows.length) sortRaf = requestAnimationFrame(step);
      else {
        sortRaf = null;
        applyFilterChunked();
      }
    }

    sortRaf = requestAnimationFrame(step);
  }

  // Clear all filters button
  document.getElementById('clearFilters').addEventListener('click', () => {
    state.global = '';
    state.log = '';
    state.level = '';
    state.timeFrom = '';
    state.timeTo = '';
    state.idList = '';
    state.providerSet.clear();

    q.value = '';
    logFilter.value = '';
    levelFilter.value = '';

    const filterInputs = filterRow.querySelectorAll('input, select');
    filterInputs.forEach(el => {
      if (el.type === 'checkbox') el.checked = false;
      else el.value = '';
    });

    const providerCheckboxes = document.querySelectorAll('.provider-option input[type="checkbox"]');
    providerCheckboxes.forEach(cb => cb.checked = false);

    const providerBtn = document.querySelector('.provider-select-btn span:first-child');
    if (providerBtn) providerBtn.textContent = 'Filter provider...';

    applyFilterChunked();
  });

  // Export filtered results
  document.getElementById('exportFiltered').addEventListener('click', () => {
    const visibleRows = rows.filter(r => !r.el.classList.contains('is-hidden'));

    let csv = 'Log,TimeCreated,ID,Level,Provider,Message\n';

    visibleRows.forEach(r => {
      const cells = r.el.querySelectorAll('td');
      const log = cells[0]?.textContent.trim() || '';
      const time = cells[1]?.textContent.trim() || '';
      const id = cells[2]?.textContent.trim() || '';
      const level = cells[3]?.textContent.trim() || '';
      const provider = cells[4]?.textContent.trim() || '';
      const message = (cells[5]?.querySelector('pre.msg')?.textContent || '').replace(/"/g, '""');

      csv += '"' + log + '","' + time + '","' + id + '","' + level + '","' + provider + '","' + message + '"\n';
    });

    const blob = new Blob([csv], { type: 'text/csv' });
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = 'filtered_events_' + new Date().toISOString().slice(0,10) + '.csv';
    a.click();
    URL.revokeObjectURL(url);
  });

  // Theme toggle
  const themeToggle = document.getElementById('themeToggle');
  let currentTheme = 'dark';

  themeToggle.addEventListener('click', () => {
    if (currentTheme === 'dark') {
      document.documentElement.setAttribute('data-theme', 'light');
      themeToggle.textContent = '☀️';
      currentTheme = 'light';
    } else {
      document.documentElement.removeAttribute('data-theme');
      themeToggle.textContent = '🌙';
      currentTheme = 'dark';
    }
  });

  // Density toggle
  const densityToggle = document.getElementById('densityToggle');
  let isCompact = false;

  densityToggle.addEventListener('click', () => {
    isCompact = !isCompact;
    document.body.classList.toggle('compact', isCompact);
    densityToggle.textContent = isCompact ? '📊 Normal' : '📊 Compact';
  });

  // Scroll to top button
  const scrollToTopBtn = document.getElementById('scrollToTop');

  window.addEventListener('scroll', () => {
    if (window.scrollY > 300) {
      scrollToTopBtn.classList.add('visible');
    } else {
      scrollToTopBtn.classList.remove('visible');
    }
  });

  scrollToTopBtn.addEventListener('click', () => {
    window.scrollTo({ top: 0, behavior: 'smooth' });
  });

  // Parallax header effect
  window.addEventListener('scroll', () => {
    const header = document.querySelector('.header');
    if (header) {
      const scrolled = window.scrollY;
      header.style.transform = 'translateY(' + (scrolled * 0.3) + 'px)';
    }
  });

  applyFilterChunked();

})();
</script>
</body>
</html>
'@

    # Now replace placeholders with PowerShell variables
    $htmlContent = $htmlContent -replace 'START_TIME_PLACEHOLDER', $startS
    $htmlContent = $htmlContent -replace 'END_TIME_PLACEHOLDER', $endS
    $htmlContent = $htmlContent -replace 'GENERATED_TIME_PLACEHOLDER', $generated
    $htmlContent = $htmlContent -replace 'TOTAL_COUNT_PLACEHOLDER', $total

    # Build log count KPIs dynamically
    $logCountKpis = ($logCounts | ForEach-Object {
        "<div class='kpi'><b>$($_.Count)</b><span>$($_.Name)</span></div>"
    }) -join "`n"

    $htmlContent = $htmlContent -replace 'LOG_COUNT_BADGES_PLACEHOLDER', $logCountKpis
    $htmlContent = $htmlContent -replace 'LEVEL_BADGES_PLACEHOLDER', $levelBadges
    $htmlContent = $htmlContent -replace 'ROWS_PLACEHOLDER', ($rows -join "`n")
    $htmlContent = $htmlContent -replace 'PROVIDERS_JSON_PLACEHOLDER', $providersJson

    $htmlContent | Out-File -FilePath $Path -Encoding UTF8
}

# =========================
# AUTOMATIC HTML EXPORT (NO PROMPTS)
# =========================
$date = Get-Date -Format "yyyyMMdd_HHmmss"
$htmlPath = "$env:USERPROFILE\Desktop\EventReport_$date.html"

Export-ModernEventHtml -Events $AllResults -Path $htmlPath -StartDateTime $StartDateTime -EndDateTime $EndDateTime

Write-Host ""
Write-Host "✅ Report generated successfully:" -ForegroundColor Green
Write-Host $htmlPath -ForegroundColor Cyan
Write-Host ""
Invoke-Item $htmlPath
```
