# Retrieve Critical, Error, and Warning Events for a specified time

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

# Convert to DateTime (today's date)
##$Date = (Get-Date -Format 'yyyy-MM-dd').AddDays(-1)
$SpecifiedTime = [datetime]::ParseExact("$Date $StartTime", 'yyyy-MM-dd HH:mm', $null)

# Calculate time window (±30 minutes)
$StartDateTime = $SpecifiedTime.AddMinutes(-$Duration)
$EndDateTime   = $SpecifiedTime.AddMinutes($Duration)

# Define event levels and logs
$Levels = @(1, 2, 3)  # 1=Critical, 2=Error, 3=Warning

Write-Host "`nRetrieving events between $StartDateTime and $EndDateTime..." -ForegroundColor Cyan

# --- Application Log ---
Write-Host "`n=== Application Log Events ===" -ForegroundColor Yellow
$AppResults = Get-WinEvent -FilterHashtable @{
    LogName   = 'Application'
    Level     = $Levels
    StartTime = $StartDateTime
    EndTime   = $EndDateTime
} | Select-Object TimeCreated, Id, LevelDisplayName, ProviderName, Message

$AppResults | Format-Table -AutoSize -Wrap

# --- System Log ---
Write-Host "`n=== System Log Events ===" -ForegroundColor Yellow
$SysResults = Get-WinEvent -FilterHashtable @{
    LogName   = 'System'
    Level     = $Levels
    StartTime = $StartDateTime
    EndTime   = $EndDateTime
} | Select-Object TimeCreated, Id, LevelDisplayName, ProviderName, Message

$SysResults | Format-Table -AutoSize -Wrap

# =========================
# NEW (Optional): Export a modern styled HTML report
# =========================
function Export-ModernEventHtml {
    param(
        [Parameter(Mandatory)] $AppResults,
        [Parameter(Mandatory)] $SysResults,
        [Parameter(Mandatory)] [string] $Path,
        [Parameter(Mandatory)] [datetime] $StartDateTime,
        [Parameter(Mandatory)] [datetime] $EndDateTime
    )

    Add-Type -AssemblyName System.Web

    $all = @()
    $all += $AppResults | ForEach-Object {
        [pscustomobject]@{
            Log = "Application"
            TimeCreated = $_.TimeCreated
            Id = $_.Id
            Level = $_.LevelDisplayName
            Provider = $_.ProviderName
            Message = $_.Message
        }
    }
    $all += $SysResults | ForEach-Object {
        [pscustomobject]@{
            Log = "System"
            TimeCreated = $_.TimeCreated
            Id = $_.Id
            Level = $_.LevelDisplayName
            Provider = $_.ProviderName
            Message = $_.Message
        }
    }

    $all = $all | Sort-Object TimeCreated

    $total = $all.Count
    $appCount = ($all | Where-Object Log -eq "Application").Count
    $sysCount = ($all | Where-Object Log -eq "System").Count

    $levelCounts = $all | Group-Object Level | Sort-Object Name

    $levelBadges = ($levelCounts | ForEach-Object {
        $name = $_.Name
        $count = $_.Count
        $cls = ("lvl-" + ($name -replace '\s','').ToLower())
        "<span class='badge {0}'>{1}: {2}</span>" -f $cls, $name, $count
    }) -join " "

    $rows = foreach ($e in $all) {
        $msg = [System.Web.HttpUtility]::HtmlEncode([string]$e.Message)
        $provider = [System.Web.HttpUtility]::HtmlEncode([string]$e.Provider)
        $lvl = [System.Web.HttpUtility]::HtmlEncode([string]$e.Level)
        $log = [System.Web.HttpUtility]::HtmlEncode([string]$e.Log)
        $time = $e.TimeCreated.ToString("yyyy-MM-dd HH:mm:ss")
        $id = [System.Web.HttpUtility]::HtmlEncode([string]$e.Id)

        $lvlClass = "lvl-" + (($e.Level -replace '\s','').ToLower())

        @"
<tr>
  <td><span class='pill pill-$log'>$log</span></td>
  <td class='mono'>$time</td>
  <td class='mono'>$id</td>
  <td><span class='badge $lvlClass'>$lvl</span></td>
  <td>$provider</td>
  <td>
    <details>
      <summary>View</summary>
      <pre class='msg'>$msg</pre>
    </details>
  </td>
</tr>
"@
    }

    $startS = $StartDateTime.ToString("yyyy-MM-dd HH:mm:ss")
    $endS   = $EndDateTime.ToString("yyyy-MM-dd HH:mm:ss")
    $generated = (Get-Date).ToString("yyyy-MM-dd HH:mm:ss")

    $html = @"
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Windows Event Report ($startS → $endS)</title>
<style>
:root{
  --bg:#0b1220; --card:#111a2e; --muted:#9fb0d0; --text:#e7eefc;
  --accent:#6ea8fe; --border:rgba(255,255,255,.08);
  --ok:#36d399; --warn:#fbbf24; --err:#f87171; --crit:#fb7185;
  --shadow: 0 12px 30px rgba(0,0,0,.35);
}
*{box-sizing:border-box}
body{margin:0;background:linear-gradient(180deg,#070b14 0%, #0b1220 60%, #070b14 100%);
     color:var(--text); font-family: "Segoe UI", system-ui, -apple-system, Arial, sans-serif;}
.container{max-width:1200px;margin:32px auto;padding:0 16px;}
.header{display:flex;gap:16px;align-items:flex-start;justify-content:space-between;flex-wrap:wrap}
.h-title{font-size:22px;margin:0}
.sub{color:var(--muted);margin-top:6px;font-size:13px}
.card{background:rgba(17,26,46,.92); border:1px solid var(--border);
      border-radius:14px; box-shadow:var(--shadow); padding:18px;}
.kpis{display:flex;gap:12px;flex-wrap:wrap;margin-top:10px}
.kpi{background:rgba(255,255,255,.03); border:1px solid var(--border);
     border-radius:12px; padding:10px 12px; min-width:160px}
.kpi b{display:block;font-size:18px}
.kpi span{color:var(--muted);font-size:12px}
.badges{display:flex;gap:8px;flex-wrap:wrap;margin-top:10px}
.badge{display:inline-flex;align-items:center;gap:6px; padding:4px 10px;
       border-radius:999px; font-size:12px; border:1px solid var(--border);
       background:rgba(255,255,255,.04)}
.lvl-critical{background:rgba(251,113,133,.16);border-color:rgba(251,113,133,.35)}
.lvl-error{background:rgba(248,113,113,.16);border-color:rgba(248,113,113,.35)}
.lvl-warning{background:rgba(251,191,36,.16);border-color:rgba(251,191,36,.35)}
.lvl-critical,.lvl-error,.lvl-warning{color:var(--text)}
.toolbar{display:flex;gap:10px;align-items:center;justify-content:space-between;flex-wrap:wrap;margin:16px 0}
.search{flex:1;min-width:260px;display:flex;gap:10px;align-items:center}
input[type="search"]{
  width:100%; padding:10px 12px; border-radius:10px;
  border:1px solid var(--border); background:rgba(0,0,0,.22); color:var(--text);
  outline:none
}
select{
  padding:10px 12px;border-radius:10px;border:1px solid var(--border);
  background:rgba(0,0,0,.22); color:var(--text); outline:none
}
table{width:100%; border-collapse:separate; border-spacing:0; overflow:hidden}
thead th{
  text-align:left; font-size:12px; color:var(--muted);
  padding:12px 10px; border-bottom:1px solid var(--border);
  position:sticky; top:0; background:rgba(17,26,46,.96); backdrop-filter: blur(8px);
}
tbody td{padding:12px 10px; border-bottom:1px solid var(--border); vertical-align:top}
tbody tr:hover{background:rgba(255,255,255,.03)}
.mono{font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, "Liberation Mono", monospace; font-size:12px}
.pill{display:inline-flex;align-items:center;padding:4px 10px;border-radius:999px;
      border:1px solid var(--border); background:rgba(255,255,255,.04); font-size:12px}
.pill-Application{border-color:rgba(110,168,254,.35); background:rgba(110,168,254,.12)}
.pill-System{border-color:rgba(54,211,153,.35); background:rgba(54,211,153,.12)}
details summary{cursor:pointer; color:var(--accent)}
pre.msg{white-space:pre-wrap; margin:10px 0 0; padding:10px; border-radius:10px;
        border:1px solid var(--border); background:rgba(0,0,0,.25)}
.footer{margin-top:14px;color:var(--muted);font-size:12px}
</style>
</head>
<body>
  <div class="container">
    <div class="header">
      <div class="card" style="flex:1">
        <h1 class="h-title">Windows Event Report</h1>
        <div class="sub">Window: <span class="mono">$startS</span> → <span class="mono">$endS</span> • Generated: <span class="mono">$generated</span></div>
        <div class="kpis">
          <div class="kpi"><b>$total</b><span>Total events</span></div>
          <div class="kpi"><b>$appCount</b><span>Application</span></div>
          <div class="kpi"><b>$sysCount</b><span>System</span></div>
        </div>
        <div class="badges">$levelBadges</div>
      </div>
    </div>

    <div class="toolbar">
      <div class="search">
        <input id="q" type="search" placeholder="Search messages, provider, ID..." />
        <select id="logFilter">
          <option value="">All logs</option>
          <option value="Application">Application</option>
          <option value="System">System</option>
        </select>
        <select id="levelFilter">
          <option value="">All levels</option>
          <option value="Critical">Critical</option>
          <option value="Error">Error</option>
          <option value="Warning">Warning</option>
        </select>
      </div>
      <div class="footer">Tip: Click “View” to expand long messages.</div>
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
          $rows
        </tbody>
      </table>
    </div>
  </div>

<script>
(function(){
  const q = document.getElementById('q');
  const logFilter = document.getElementById('logFilter');
  const levelFilter = document.getElementById('levelFilter');
  const tbody = document.querySelector('#events tbody');
  const rows = Array.from(tbody.querySelectorAll('tr'));

  function apply(){
    const term = (q.value || '').toLowerCase();
    const log = logFilter.value;
    const lvl = levelFilter.value;

    rows.forEach(r=>{
      const text = r.innerText.toLowerCase();
      const logCell = r.children[0].innerText.trim();
      const lvlCell = r.children[3].innerText.trim();

      const okTerm = !term || text.indexOf(term) !== -1;
      const okLog  = !log || logCell === log;
      const okLvl  = !lvl || lvlCell === lvl;

      r.style.display = (okTerm && okLog && okLvl) ? '' : 'none';
    });
  }
  q.addEventListener('input', apply);
  logFilter.addEventListener('change', apply);
  levelFilter.addEventListener('change', apply);
})();
</script>
</body>
</html>
"@

    $html | Out-File -FilePath $Path -Encoding UTF8
}

# =========================
# Optional: Exports
# =========================
$export = Read-Host "Optional export: (C)SV, (H)TML, (B)oth, or press Enter to skip"
switch ($export.ToUpper()) {
    "C" {
        $AppResults | Export-Csv -Path "$env:USERPROFILE\Desktop\ApplicationEvents.csv" -NoTypeInformation
        $SysResults | Export-Csv -Path "$env:USERPROFILE\Desktop\SystemEvents.csv" -NoTypeInformation
        Write-Host "CSV exports written to Desktop." -ForegroundColor Green
    }
    "H" {
        $htmlPath = "$env:USERPROFILE\Desktop\EventReport.html"
        Export-ModernEventHtml -AppResults $AppResults -SysResults $SysResults -Path $htmlPath -StartDateTime $StartDateTime -EndDateTime $EndDateTime
        Write-Host "HTML report written to: $htmlPath" -ForegroundColor Green
        Invoke-Item $htmlPath
    }
    "B" {
        $AppResults | Export-Csv -Path "$env:USERPROFILE\Desktop\ApplicationEvents.csv" -NoTypeInformation
        $SysResults | Export-Csv -Path "$env:USERPROFILE\Desktop\SystemEvents.csv" -NoTypeInformation
        $htmlPath = "$env:USERPROFILE\Desktop\EventReport_$date.html"
        Export-ModernEventHtml -AppResults $AppResults -SysResults $SysResults -Path $htmlPath -StartDateTime $StartDateTime -EndDateTime $EndDateTime
        Write-Host "CSV + HTML exports written to Desktop." -ForegroundColor Green
        Invoke-Item $htmlPath
    }
    default {
        # do nothing
    }
}
```
