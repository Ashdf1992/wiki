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

/* --- Performance: hide rows via class (used by JS) --- */
.is-hidden { display: none !important; }

/* Optional: visual hint while filtering large sets */
body.is-filtering * { cursor: progress; }

/* --- Animations (subtle, modern) --- */
@keyframes fadeUp {
  from { opacity: 0; transform: translateY(8px); }
  to   { opacity: 1; transform: translateY(0); }
}

.card { animation: fadeUp .35s ease-out both; }
tbody tr { animation: fadeUp .20s ease-out both; }

/* Smooth hover + focus affordances */
.badge, .pill, input[type="search"], select {
  transition: transform .12s ease, box-shadow .12s ease, border-color .12s ease;
}
.badge:hover, .pill:hover { transform: translateY(-1px); }

/* Nicer focus for keyboard navigation */
input[type="search"]:focus, select:focus {
  border-color: rgba(110,168,254,.55);
  box-shadow: 0 0 0 3px rgba(110,168,254,.18);
}

/* --- Responsive layout improvements --- */
@media (max-width: 860px) {
  .toolbar { gap: 12px; }
  .search { flex-wrap: wrap; }
  .search > * { flex: 1 1 220px; }
  .footer { width: 100%; }
}

@media (max-width: 640px) {
  .kpis .kpi { min-width: 140px; }
  thead th:nth-child(5), tbody td:nth-child(5) { display: none; } /* hide Provider on very small screens */
  .card { padding: 14px; }
}

/* Allow horizontal scroll for table on narrow screens */
.card { overflow-x: auto; }
table { min-width: 900px; }

/* --- Respect reduced motion preferences --- */
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation: none !important;
    transition: none !important;
    scroll-behavior: auto !important;
  }
} /* prefers-reduced-motion guidance [5](https://developer.mozilla.org/en-US/docs/Web/CSS/Reference/At-rules/@media/prefers-reduced-motion)[6](https://www.w3.org/WAI/WCAG21/Techniques/css/C39.html) */
/* --- Sortable headers --- */
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

/* --- Column filter row --- */
thead tr.col-filters th {
  padding: 8px 10px;
  border-bottom: 1px solid var(--border);
  background: rgba(17,26,46,.96);
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
thead tr.col-filters .range {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 8px;
}

/* Make header & filter row sticky together */
thead th { position: sticky; top: 0; z-index: 2; }
thead tr.col-filters th { position: sticky; top: 42px; z-index: 2; }

/* Hide rows quickly */
.is-hidden { display:none !important; }

/* Responsive: allow horizontal scroll */
.card { overflow-x:auto; }
table { min-width: 900px; }
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

(function () {
  const table = document.getElementById('events');
  const thead = table.querySelector('thead');
  const headerRow = thead.querySelector('tr'); // existing header row
  const tbody = table.querySelector('tbody');

  // ---- Helpers ----
  function debounce(fn, wait) {
    let t;
    return (...args) => {
      clearTimeout(t);
      t = setTimeout(() => fn(...args), wait);
    };
  } // Debounce via setTimeout/clearTimeout [6](https://www.geeksforgeeks.org/javascript/debouncing-in-javascript/)

  function parseUtcFromYmdHms(s) {
    // expects "yyyy-MM-dd HH:mm:ss"
    // robust parsing (avoids Date.parse quirks)
    const m = String(s || '').match(/^(\d{4})-(\d{2})-(\d{2})\s+(\d{2}):(\d{2}):(\d{2})$/);
    if (!m) return NaN;
    return Date.UTC(+m[1], (+m[2]) - 1, +m[3], +m[4], +m[5], +m[6]);
  }

  function naturalCompare(a, b) {
    return a.localeCompare(b, undefined, { numeric: true, sensitivity: 'base' });
  }

  // ---- Build row cache (keeps filtering/sorting fast) ----
  let rows = Array.from(tbody.querySelectorAll('tr')).map((tr) => {
    const cells = tr.children;

    const log = (cells[0]?.innerText || '').trim();
    const timeStr = (cells[1]?.innerText || '').trim();
    const idStr = (cells[2]?.innerText || '').trim();
    const level = (cells[3]?.innerText || '').trim();
    const provider = (cells[4]?.innerText || '').trim();

    // message is inside details/pre, textContent includes it; keep full search text cached
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
      fullText
    };
  });

  // ---- Insert column-filter row into THEAD (no PowerShell HTML changes required) ----
  const filterRow = document.createElement('tr');
  filterRow.className = 'col-filters';

  // current filter state
  const state = {
    global: '',
    log: '',
    level: '',
    timeFrom: '',
    timeTo: '',
    idMin: '',
    idMax: '',
    provider: '',
    // if you want per-column text filters for others later, add them here
  };

  // Build option sets for selects
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

  function makeInput(ph) {
    const inp = document.createElement('input');
    inp.type = 'text';
    inp.placeholder = ph;
    return inp;
  }

  // Create 6 filter cells matching columns: Log, TimeCreated, ID, Level, Provider, Message
  // Log filter (select)
  {
    const th = document.createElement('th');
    const sel = makeSelect(logOptions, 'All logs');
    sel.addEventListener('change', () => { state.log = sel.value; applyFilterChunked(); });
    th.appendChild(sel);
    filterRow.appendChild(th);
  }

  // TimeCreated filter (range)
  {
    const th = document.createElement('th');
    const wrap = document.createElement('div');
    wrap.className = 'range';
    const from = makeInput('From (yyyy-mm-dd hh:mm:ss)');
    const to = makeInput('To (yyyy-mm-dd hh:mm:ss)');
    from.addEventListener('input', debounce(() => { state.timeFrom = from.value.trim(); applyFilterChunked(); }, 180));
    to.addEventListener('input', debounce(() => { state.timeTo = to.value.trim(); applyFilterChunked(); }, 180));
    wrap.appendChild(from); wrap.appendChild(to);
    th.appendChild(wrap);
    filterRow.appendChild(th);
  }

  // ID filter (min/max)
  {
    const th = document.createElement('th');
    const wrap = document.createElement('div');
    wrap.className = 'range';
    const min = makeInput('Min');
    const max = makeInput('Max');
    min.addEventListener('input', debounce(() => { state.idMin = min.value.trim(); applyFilterChunked(); }, 180));
    max.addEventListener('input', debounce(() => { state.idMax = max.value.trim(); applyFilterChunked(); }, 180));
    wrap.appendChild(min); wrap.appendChild(max);
    th.appendChild(wrap);
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

  // Provider filter (text contains)
  {
    const th = document.createElement('th');
    const inp = makeInput('Filter provider...');
    inp.addEventListener('input', debounce(() => { state.provider = inp.value.trim().toLowerCase(); applyFilterChunked(); }, 180));
    th.appendChild(inp);
    filterRow.appendChild(th);
  }

  // Message filter (use global search box already present, so leave empty / hint)
  {
    const th = document.createElement('th');
    const hint = document.createElement('div');
    hint.style.color = 'var(--muted)';
    hint.style.fontSize = '12px';
    hint.textContent = 'Use global search above';
    th.appendChild(hint);
    filterRow.appendChild(th);
  }

  thead.appendChild(filterRow);

  // ---- Wire global search + existing dropdowns if present (your UI already has these) ----
  const q = document.getElementById('q');
  const logFilter = document.getElementById('logFilter');
  const levelFilter = document.getElementById('levelFilter');

  if (q) q.addEventListener('input', debounce(() => { state.global = (q.value || '').toLowerCase(); applyFilterChunked(); }, 180));
  if (logFilter) logFilter.addEventListener('change', () => { state.log = logFilter.value; applyFilterChunked(); });
  if (levelFilter) levelFilter.addEventListener('change', () => { state.level = levelFilter.value; applyFilterChunked(); });

  // ---- Filtering (chunked with requestAnimationFrame for responsiveness) ----
  let filterRaf = null;
  function applyFilterChunked() {
    if (filterRaf) cancelAnimationFrame(filterRaf);

    const CHUNK = 350;
    let i = 0;

    // Pre-parse numeric/time ranges once per run
    const timeFromMs = state.timeFrom ? parseUtcFromYmdHms(state.timeFrom) : NaN;
    const timeToMs = state.timeTo ? parseUtcFromYmdHms(state.timeTo) : NaN;
    const idMin = state.idMin ? parseInt(state.idMin, 10) : NaN;
    const idMax = state.idMax ? parseInt(state.idMax, 10) : NaN;

    function step() {
      const end = Math.min(i + CHUNK, rows.length);
      for (; i < end; i++) {
        const r = rows[i];

        const okGlobal = !state.global || r.fullText.indexOf(state.global) !== -1;
        const okLog = !state.log || r.log === state.log;
        const okLvl = !state.level || r.level === state.level;

        const okProvider = !state.provider || r.provider.toLowerCase().indexOf(state.provider) !== -1;

        const okTimeFrom = isNaN(timeFromMs) || (!isNaN(r.timeMs) && r.timeMs >= timeFromMs);
        const okTimeTo   = isNaN(timeToMs)   || (!isNaN(r.timeMs) && r.timeMs <= timeToMs);

        const okIdMin = isNaN(idMin) || (!isNaN(r.id) && r.id >= idMin);
        const okIdMax = isNaN(idMax) || (!isNaN(r.id) && r.id <= idMax);

        const show = okGlobal && okLog && okLvl && okProvider && okTimeFrom && okTimeTo && okIdMin && okIdMax;
        r.el.classList.toggle('is-hidden', !show);
      }

      if (i < rows.length) filterRaf = requestAnimationFrame(step);
      else filterRaf = null;
    }

    filterRaf = requestAnimationFrame(step);
  }
  // requestAnimationFrame runs callbacks before repaint [3](https://developer.mozilla.org/en-US/docs/Web/API/Window/requestAnimationFrame)

  // ---- Sorting (click header; supports asc/desc; chunked DOM re-append) ----
  const colDefs = [
    { name: 'Log',        key: r => r.log,     cmp: (a,b)=>naturalCompare(a.log,b.log) },
    { name: 'TimeCreated',key: r => r.timeMs,  cmp: (a,b)=>(a.timeMs - b.timeMs) || naturalCompare(a.timeStr,b.timeStr) },
    { name: 'ID',         key: r => r.id,      cmp: (a,b)=>(a.id - b.id) || naturalCompare(a.idStr,b.idStr) },
    { name: 'Level',      key: r => r.level,   cmp: (a,b)=>naturalCompare(a.level,b.level) },
    { name: 'Provider',   key: r => r.provider,cmp: (a,b)=>naturalCompare(a.provider,b.provider) },
    { name: 'Message',    key: r => r.fullText,cmp: (a,b)=>naturalCompare(a.fullText,b.fullText), noSort: true }
  ];

  let sortState = { index: -1, dir: 1 }; // dir: 1 asc, -1 desc
  const ths = Array.from(headerRow.querySelectorAll('th'));

  ths.forEach((th, idx) => {
    const def = colDefs[idx];
    if (!def || def.noSort) return;

    th.classList.add('sortable');
    const ind = document.createElement('span');
    ind.className = 'sort-ind';
    th.appendChild(ind);

    th.addEventListener('click', () => {
      // toggle direction if same column, else default to asc
      const same = (sortState.index === idx);
      sortState.index = idx;
      sortState.dir = same ? (sortState.dir * -1) : 1;

      // visual state
      ths.forEach(h => h.classList.remove('sorted-asc','sorted-desc'));
      th.classList.add(sortState.dir === 1 ? 'sorted-asc' : 'sorted-desc');

      sortRowsChunked(def, sortState.dir);
    });
  });

  let sortRaf = null;
  function sortRowsChunked(def, dir) {
    if (sortRaf) cancelAnimationFrame(sortRaf);

    // Stable-ish sort by decorating with original index
    const decorated = rows.map((r, i) => ({ r, i }));
    decorated.sort((A, B) => {
      const base = def.cmp(A.r, B.r);
      if (base !== 0) return base * dir;
      return (A.i - B.i); // tiebreaker: original order
    });

    // Update primary rows array order (used by filter chunking)
    rows = decorated.map(x => x.r);

    // Re-append DOM nodes in chunks for performance
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
        // keep current filters applied after reorder
        applyFilterChunked();
      }
    }

    sortRaf = requestAnimationFrame(step);
  }

  // Initial filter pass (optional)
  applyFilterChunked();

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
        $AppResults | Export-Csv -Path "$env:USERPROFILE\Desktop\ApplicationEvents_$date.csv" -NoTypeInformation
        $SysResults | Export-Csv -Path "$env:USERPROFILE\Desktop\SystemEvents_$date.csv" -NoTypeInformation
        Write-Host "CSV exports written to Desktop." -ForegroundColor Green
    }
    "H" {
        $htmlPath = "$env:USERPROFILE\Desktop\EventReport_$date.html"
        Export-ModernEventHtml -AppResults $AppResults -SysResults $SysResults -Path $htmlPath -StartDateTime $StartDateTime -EndDateTime $EndDateTime
        Write-Host "HTML report written to: $htmlPath" -ForegroundColor Green
        Invoke-Item $htmlPath
    }
    "B" {
        $AppResults | Export-Csv -Path "$env:USERPROFILE\Desktop\ApplicationEvents_$date.csv" -NoTypeInformation
        $SysResults | Export-Csv -Path "$env:USERPROFILE\Desktop\SystemEvents_$date.csv" -NoTypeInformation
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