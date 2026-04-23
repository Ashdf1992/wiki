# Enumerate WWNs for Disks

## V2 (Modern with HTML and CSV Report)
>Run the following within Powershell or Powershell ISE:
```Powershell
#requires -Version 5.0
<#
.SYNOPSIS
    Produces a professional SAN / RDM disk mapping report for clustered Windows servers.

.DESCRIPTION
    Uses Win32_LogicalDisk -> Win32_LogicalDiskToPartition -> Win32_DiskDriveToDiskPartition
    to reliably map drive letters to Disk Management disk numbers, then correlates those
    disk numbers to Get-PhysicalDisk.DeviceID.

    Designed for VMware RDM / SAN-presented clustered disks.

.NOTES
    - PowerShell 5 and above
    - No parameters required
    - Run elevated
#>

Clear-Host

# ------------------------------------------------------------
# Configuration
# ------------------------------------------------------------
$ExportCsv          = $false
$ExportHtml         = $true
$OpenHtmlAfterExport = $true

$Timestamp = Get-Date -Format 'yyyyMMdd_HHmmss'
$CsvPath   = "C:\Temp\SAN_RDM_Disk_Report_$($env:COMPUTERNAME)_$Timestamp.csv"
$HtmlPath  = "C:\Temp\SAN_RDM_Disk_Report_$($env:COMPUTERNAME)_$Timestamp.html"

# ------------------------------------------------------------
# Helper Functions
# ------------------------------------------------------------
function Convert-SizeToFriendly {
    param (
        [Nullable[decimal]]$Bytes
    )

    if ($null -eq $Bytes -or $Bytes -lt 0) {
        return "-"
    }

    if ($Bytes -ge 1TB) {
        return ("{0:N2} TB" -f ($Bytes / 1TB))
    }
    elseif ($Bytes -ge 1GB) {
        return ("{0:N2} GB" -f ($Bytes / 1GB))
    }
    elseif ($Bytes -ge 1MB) {
        return ("{0:N2} MB" -f ($Bytes / 1MB))
    }
    else {
        return ("{0:N0} Bytes" -f $Bytes)
    }
}

function Write-Section {
    param (
        [string]$Text
    )

    Write-Host ""
    Write-Host $Text -ForegroundColor Cyan
    Write-Host ("-" * $Text.Length) -ForegroundColor Cyan
}

function Get-PrintableReport {
    param (
        [Parameter(Mandatory = $true)]
        [array]$InputObject
    )

    $InputObject | Select-Object `
        'Drive Letter',
        'Volume Label',
        'Disk Number',
        'UniqueID',
        'Friendly Name',
        'Allocated LUN Size',
        'Used Space',
        'Free Space'
}

function Export-ModernSanHtml {
    param(
        [Parameter(Mandatory = $true)]
        [array]$Rows,

        [Parameter(Mandatory = $true)]
        [string]$Path,

        [Parameter(Mandatory = $true)]
        [string]$ServerName,

        [Parameter(Mandatory = $true)]
        [datetime]$GeneratedTime,

        [Parameter(Mandatory = $true)]
        [int]$MountedCount,

        [Parameter(Mandatory = $true)]
        [int]$UnmappedCount,

        [Parameter(Mandatory = $true)]
        [int]$PhysicalDiskCount,

        [Parameter(Mandatory = $true)]
        [decimal]$TotalAllocatedBytes,

        [Parameter(Mandatory = $true)]
        [decimal]$TotalUsedBytes,

        [Parameter(Mandatory = $true)]
        [decimal]$TotalFreeBytes
    )

    Add-Type -AssemblyName System.Web

    $generated = $GeneratedTime.ToString("yyyy-MM-dd HH:mm:ss")

    # Section badges (rendered as real HTML, not escaped text)
    $sectionBadgeList = @()

    foreach ($sectionName in @(
        'System / Local',
        'Cluster / SAN',
        'Additional LUN / No Drive Letter'
    )) {
        $count = @($Rows | Where-Object { $_.Section -eq $sectionName }).Count

        if ($count -lt 1) {
            continue
        }

        $cls = switch ($sectionName) {
            'System / Local'                  { 'sec-system' }
            'Cluster / SAN'                   { 'sec-cluster' }
            default                           { 'sec-unmapped' }
        }

        $displayName = switch ($sectionName) {
            'System / Local'                   { 'System / Local' }
            'Cluster / SAN'                    { 'Cluster / SAN' }
            'Additional LUN / No Drive Letter' { 'No Drive Letter' }
            default                            { $sectionName }
        }

        $sectionBadgeList += "<span class='badge $cls'>$displayName <strong>$count</strong></span>"
    }

    $sectionBadges = $sectionBadgeList -join "`n"

    # Build table rows
    $rowsHtml = @()

    foreach ($row in ($Rows | Sort-Object SectionSortOrder, SortDriveLetter, SortDiskNumber)) {

        $section      = [System.Web.HttpUtility]::HtmlEncode([string]$row.Section)
        $driveLetter  = [System.Web.HttpUtility]::HtmlEncode([string]$row.'Drive Letter')
        $volumeLabel  = [System.Web.HttpUtility]::HtmlEncode([string]$row.'Volume Label')
        $diskNumber   = [System.Web.HttpUtility]::HtmlEncode([string]$row.'Disk Number')
        $uniqueId     = [System.Web.HttpUtility]::HtmlEncode([string]$row.UniqueID)
        $friendlyName = [System.Web.HttpUtility]::HtmlEncode([string]$row.'Friendly Name')
        $allocated    = [System.Web.HttpUtility]::HtmlEncode([string]$row.'Allocated LUN Size')
        $used         = [System.Web.HttpUtility]::HtmlEncode([string]$row.'Used Space')
        $free         = [System.Web.HttpUtility]::HtmlEncode([string]$row.'Free Space')

        $sectionClass = switch ($row.Section) {
            'System / Local'                   { 'sec-system' }
            'Cluster / SAN'                    { 'sec-cluster' }
            default                            { 'sec-unmapped' }
        }

        $usedSort = if ($null -ne $row.UsedBytesRaw) { [string][decimal]$row.UsedBytesRaw } else { "0" }
        $freeSort = if ($null -ne $row.FreeBytesRaw) { [string][decimal]$row.FreeBytesRaw } else { "0" }

        $rowsHtml += @"
<tr data-section="$section">
  <td data-sort="$($row.SectionSortOrder)"><span class='pill $sectionClass'>$section</span></td>
  <td data-sort="$($row.SortDriveLetter)"><span class='mono'>$driveLetter</span></td>
  <td data-sort="$volumeLabel">$volumeLabel</td>
  <td data-sort="$($row.SortDiskNumber)" class='mono'>$diskNumber</td>
  <td data-sort="$uniqueId" class='mono'>$uniqueId</td>
  <td data-sort="$friendlyName">$friendlyName</td>
  <td data-sort="$([string][decimal]$row.AllocatedBytesRaw)" class='mono'>$allocated</td>
  <td data-sort="$usedSort" class='mono'>$used</td>
  <td data-sort="$freeSort" class='mono'>$free</td>
</tr>
"@
    }

    # HTML template
    $htmlContent = @'
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>SAN / RDM Disk Mapping Report</title>
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
  opacity: 0.35;
}

.container{
  max-width:1400px;
  margin:32px auto;
  padding:0 16px;
  position: relative;
  z-index: 2;
}

.header{
  display:flex;
  gap:16px;
  align-items:flex-start;
  justify-content:space-between;
  flex-wrap:wrap;
}

.h-title{
  font-size:24px;
  margin:0;
}

.sub{
  color:var(--muted);
  margin-top:8px;
  font-size:13px;
  line-height:1.5;
}

.card{
  background: rgba(17,26,46,.85);
  backdrop-filter: blur(18px);
  border:1px solid var(--border);
  border-radius:16px;
  box-shadow:var(--shadow);
  padding:22px;
  position: relative;
  overflow: visible;
  animation: fadeUp .35s ease-out both;
}

[data-theme="light"] .card {
  background: rgba(255,255,255,.95);
}

.header .card {
  background: linear-gradient(145deg, rgba(20,30,55,.95), rgba(12,18,35,.95));
  border: 1px solid rgba(110,168,254,.15);
}

[data-theme="light"] .header .card {
  background: linear-gradient(145deg, rgba(255,255,255,.98), rgba(245,247,250,.98));
}

.kpis{
  display:grid;
  grid-template-columns:repeat(auto-fit,minmax(180px,1fr));
  gap:12px;
  margin-top:16px;
}

.kpi{
  background:rgba(255,255,255,.03);
  border:1px solid var(--border);
  border-radius:14px;
  padding:12px 14px;
  min-height:82px;
  transition: transform .2s ease, box-shadow .2s ease;
}

.kpi:hover {
  transform: translateY(-2px);
  box-shadow: 0 8px 20px rgba(0,0,0,.2);
}

.kpi b{
  display:block;
  font-size:24px;
  line-height:1.1;
  margin-bottom:6px;
}

.kpi span{
  color:var(--muted);
  font-size:12px;
}

.summary-strip{
  margin-top:18px;
  padding-top:14px;
  border-top:1px solid var(--border);
}

.summary-strip-title{
  font-size:12px;
  color:var(--muted);
  text-transform:uppercase;
  letter-spacing:.08em;
  margin-bottom:10px;
}

.badges{
  display:flex;
  gap:8px;
  flex-wrap:wrap;
}

.badge{
  display:inline-flex;
  align-items:center;
  gap:6px;
  padding:6px 12px;
  border-radius:999px;
  font-size:12px;
  border:1px solid var(--border);
  background:rgba(255,255,255,.04);
}

.badge strong{
  font-weight:700;
  margin-left:4px;
}

.sec-system{
  background:rgba(110,168,254,.16);
  border-color:rgba(110,168,254,.35);
  box-shadow: 0 0 12px rgba(110,168,254,.25);
}
.sec-cluster{
  background:rgba(54,211,153,.16);
  border-color:rgba(54,211,153,.35);
  box-shadow: 0 0 12px rgba(54,211,153,.25);
}
.sec-unmapped{
  background:rgba(251,191,36,.16);
  border-color:rgba(251,191,36,.35);
  box-shadow: 0 0 12px rgba(251,191,36,.25);
}

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

.search{
  flex:1;
  min-width:260px;
  display:flex;
  gap:10px;
  align-items:center;
  flex-wrap:wrap;
}

input[type="search"], select{
  width:100%;
  padding:10px 12px;
  border-radius:10px;
  border:1px solid var(--border);
  background:rgba(0,0,0,.22);
  color:var(--text);
  outline:none;
  transition: border-color .2s ease, box-shadow .2s ease;
}

[data-theme="light"] input[type="search"],
[data-theme="light"] select {
  background: rgba(0,0,0,.04);
}

input[type="search"]:focus,
select:focus {
  border-color: rgba(110,168,254,.55);
  box-shadow: 0 0 0 3px rgba(110,168,254,.18);
}

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

.controls {
  display:flex;
  gap:8px;
  align-items:center;
}

body.compact tbody td {
  padding: 6px 10px;
  font-size: 11px;
}

body.compact .mono {
  font-size: 11px;
}

body.compact .badge,
body.compact .pill {
  padding: 2px 8px;
  font-size: 10px;
}

.card > div {
  overflow-x: auto;
}

table{
  width:100%;
  border-collapse:separate;
  border-spacing:0;
  min-width:1200px;
}

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

tbody tr{
  transition: all .25s ease;
}

tbody tr:hover{
  background:rgba(255,255,255,.04);
  transform: translateY(-1px);
  box-shadow: 0 8px 18px rgba(0,0,0,.22);
}

[data-theme="light"] tbody tr:hover {
  background: rgba(0,0,0,.02);
  box-shadow: 0 8px 18px rgba(0,0,0,.08);
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
}

.footer{
  margin-top:14px;
  color:var(--muted);
  font-size:12px;
}

.is-hidden { display: none !important; }

#resultCount {
  text-align: center;
  padding: 10px;
  font-size: 13px;
  color: var(--accent);
  font-weight: 500;
}

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

@keyframes fadeUp {
  from { opacity: 0; transform: translateY(8px); }
  to   { opacity: 1; transform: translateY(0); }
}

@media (max-width: 860px) {
  .toolbar { gap: 12px; }
  .search { flex-wrap: wrap; }
  .search > * { flex: 1 1 220px; }
  .footer { width: 100%; }
}

@media (max-width: 640px) {
  .card { padding: 16px; }
  .kpi { min-height: 74px; }
  .kpi b { font-size: 20px; }
}

@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation: none !important;
    transition: none !important;
    scroll-behavior: auto !important;
  }
}
</style>
</head>
<body>
  <div class="container">
    <div class="header">
      <div class="card" style="flex:1">
        <h1 class="h-title">SAN / RDM Disk Mapping Report</h1>
        <div class="sub">Server: <span class="mono">SERVER_PLACEHOLDER</span> • Generated: <span class="mono">GENERATED_PLACEHOLDER</span></div>

        <div class="kpis">
          <div class="kpi"><b>PHYSICAL_DISKS_PLACEHOLDER</b><span>Total physical disks</span></div>
          <div class="kpi"><b>MOUNTED_COUNT_PLACEHOLDER</b><span>Mounted volumes matched</span></div>
          <div class="kpi"><b>UNMAPPED_COUNT_PLACEHOLDER</b><span>LUNs without drive letter</span></div>
          <div class="kpi"><b>TOTAL_ALLOCATED_PLACEHOLDER</b><span>Total allocated LUN size</span></div>
          <div class="kpi"><b>TOTAL_USED_PLACEHOLDER</b><span>Total used space</span></div>
          <div class="kpi"><b>TOTAL_FREE_PLACEHOLDER</b><span>Total free space</span></div>
        </div>

        <div class="summary-strip">
          <div class="summary-strip-title">Section breakdown</div>
          <div class="badges">SECTION_BADGES_PLACEHOLDER</div>
        </div>
      </div>
    </div>

    <div class="toolbar">
      <div class="search">
        <input id="q" type="search" placeholder="Search drive, label, disk number, unique ID..." />
        <select id="sectionFilter">
          <option value="">All sections</option>
          <option value="System / Local">System / Local</option>
          <option value="Cluster / SAN">Cluster / SAN</option>
          <option value="Additional LUN / No Drive Letter">Additional LUN / No Drive Letter</option>
        </select>
      </div>
      <div class="controls">
        <button id="clearFilters">🗑️ Clear Filters</button>
        <button id="exportFiltered">💾 Export Filtered</button>
        <button id="densityToggle">📊 Compact</button>
        <button id="themeToggle">🌙</button>
      </div>
    </div>

    <div class="card">
      <div>
        <table id="disks">
          <thead>
            <tr>
              <th>Section</th>
              <th>Drive Letter</th>
              <th>Volume Label</th>
              <th>Disk Number</th>
              <th>UniqueID</th>
              <th>Friendly Name</th>
              <th>Allocated LUN Size</th>
              <th>Used Space</th>
              <th>Free Space</th>
            </tr>
          </thead>
          <tbody>
            ROWS_PLACEHOLDER
          </tbody>
        </table>
      </div>
      <div id="resultCount" class="footer" style="margin-top:12px;">
        Showing 0 of 0 rows
      </div>
    </div>
  </div>

  <button id="scrollToTop" title="Scroll to top">↑</button>

<script>
(function () {
  const table = document.getElementById('disks');
  const thead = table.querySelector('thead');
  const headerRow = thead.querySelector('tr');
  const tbody = table.querySelector('tbody');
  const q = document.getElementById('q');
  const sectionFilter = document.getElementById('sectionFilter');
  const resultCountEl = document.getElementById('resultCount');
  const clearBtn = document.getElementById('clearFilters');
  const exportBtn = document.getElementById('exportFiltered');
  const themeToggle = document.getElementById('themeToggle');
  const densityToggle = document.getElementById('densityToggle');
  const scrollToTopBtn = document.getElementById('scrollToTop');

  function debounce(fn, wait) {
    let t;
    return function () {
      const args = arguments;
      clearTimeout(t);
      t = setTimeout(function () { fn.apply(null, args); }, wait);
    };
  }

  function naturalCompare(a, b) {
    return String(a).localeCompare(String(b), undefined, { numeric: true, sensitivity: 'base' });
  }

  let rows = Array.from(tbody.querySelectorAll('tr')).map(function (tr) {
    const cells = tr.children;
    return {
      el: tr,
      section: (tr.getAttribute('data-section') || '').trim(),
      values: Array.from(cells).map(function (td) {
        return {
          sort: td.getAttribute('data-sort') || td.textContent.trim(),
          text: td.textContent.trim()
        };
      }),
      fullText: tr.textContent.toLowerCase()
    };
  });

  function updateResultCount() {
    const visible = rows.filter(function (r) { return !r.el.classList.contains('is-hidden'); }).length;
    resultCountEl.textContent = 'Showing ' + visible + ' of ' + rows.length + ' rows';
  }

  function applyFilter() {
    const search = (q.value || '').toLowerCase();
    const section = sectionFilter.value || '';

    rows.forEach(function (r) {
      const okSearch = !search || r.fullText.indexOf(search) !== -1;
      const okSection = !section || r.section === section;
      const show = okSearch && okSection;
      r.el.classList.toggle('is-hidden', !show);
    });

    updateResultCount();
  }

  q.addEventListener('input', debounce(applyFilter, 150));
  sectionFilter.addEventListener('change', applyFilter);

  clearBtn.addEventListener('click', function () {
    q.value = '';
    sectionFilter.value = '';
    applyFilter();
  });

  exportBtn.addEventListener('click', function () {
    const visibleRows = rows.filter(function (r) { return !r.el.classList.contains('is-hidden'); });

    let csv = 'Section,Drive Letter,Volume Label,Disk Number,UniqueID,Friendly Name,Allocated LUN Size,Used Space,Free Space\n';

    visibleRows.forEach(function (r) {
      const cols = r.values.map(function (v) {
        return '"' + String(v.text).replace(/"/g, '""') + '"';
      });
      csv += cols.join(',') + '\n';
    });

    const blob = new Blob([csv], { type: 'text/csv' });
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = 'san_rdm_disk_report_filtered.csv';
    a.click();
    URL.revokeObjectURL(url);
  });

  let currentTheme = 'dark';
  themeToggle.addEventListener('click', function () {
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

  let isCompact = false;
  densityToggle.addEventListener('click', function () {
    isCompact = !isCompact;
    document.body.classList.toggle('compact', isCompact);
    densityToggle.textContent = isCompact ? '📊 Normal' : '📊 Compact';
  });

  window.addEventListener('scroll', function () {
    if (window.scrollY > 300) {
      scrollToTopBtn.classList.add('visible');
    } else {
      scrollToTopBtn.classList.remove('visible');
    }
  });

  scrollToTopBtn.addEventListener('click', function () {
    window.scrollTo({ top: 0, behavior: 'smooth' });
  });

  const colDefs = [
    { index: 0, type: 'number' },
    { index: 1, type: 'text'   },
    { index: 2, type: 'text'   },
    { index: 3, type: 'number' },
    { index: 4, type: 'text'   },
    { index: 5, type: 'text'   },
    { index: 6, type: 'number' },
    { index: 7, type: 'number' },
    { index: 8, type: 'number' }
  ];

  let sortState = { index: -1, dir: 1 };
  const ths = Array.from(headerRow.querySelectorAll('th'));

  ths.forEach(function (th, idx) {
    th.classList.add('sortable');
    const ind = document.createElement('span');
    ind.className = 'sort-ind';
    th.appendChild(ind);

    th.addEventListener('click', function () {
      const same = sortState.index === idx;
      sortState.index = idx;
      sortState.dir = same ? (sortState.dir * -1) : 1;

      ths.forEach(function (x) { x.classList.remove('sorted-asc', 'sorted-desc'); });
      th.classList.add(sortState.dir === 1 ? 'sorted-asc' : 'sorted-desc');

      const def = colDefs[idx];

      rows.sort(function (a, b) {
        const av = a.values[idx] ? a.values[idx].sort : '';
        const bv = b.values[idx] ? b.values[idx].sort : '';

        if (def.type === 'number') {
          const an = parseFloat(av);
          const bn = parseFloat(bv);
          const aNum = isNaN(an) ? 0 : an;
          const bNum = isNaN(bn) ? 0 : bn;

          if (aNum !== bNum) {
            return (aNum - bNum) * sortState.dir;
          }
        }

        return naturalCompare(av, bv) * sortState.dir;
      });

      const frag = document.createDocumentFragment();
      rows.forEach(function (r) { frag.appendChild(r.el); });
      tbody.appendChild(frag);
      applyFilter();
    });
  });

  applyFilter();
})();
</script>
</body>
</html>
'@

    # Safe, plain string replacement (no regex escaping)
    $serverEncoded    = [System.Web.HttpUtility]::HtmlEncode([string]$ServerName)
    $generatedEncoded = [System.Web.HttpUtility]::HtmlEncode([string]$generated)
    $allocatedEncoded = [System.Web.HttpUtility]::HtmlEncode((Convert-SizeToFriendly -Bytes $TotalAllocatedBytes))
    $usedEncoded      = [System.Web.HttpUtility]::HtmlEncode((Convert-SizeToFriendly -Bytes $TotalUsedBytes))
    $freeEncoded      = [System.Web.HttpUtility]::HtmlEncode((Convert-SizeToFriendly -Bytes $TotalFreeBytes))

    $htmlContent = $htmlContent.Replace('SERVER_PLACEHOLDER', $serverEncoded)
    $htmlContent = $htmlContent.Replace('GENERATED_PLACEHOLDER', $generatedEncoded)
    $htmlContent = $htmlContent.Replace('PHYSICAL_DISKS_PLACEHOLDER', [string]$PhysicalDiskCount)
    $htmlContent = $htmlContent.Replace('MOUNTED_COUNT_PLACEHOLDER', [string]$MountedCount)
    $htmlContent = $htmlContent.Replace('UNMAPPED_COUNT_PLACEHOLDER', [string]$UnmappedCount)
    $htmlContent = $htmlContent.Replace('TOTAL_ALLOCATED_PLACEHOLDER', $allocatedEncoded)
    $htmlContent = $htmlContent.Replace('TOTAL_USED_PLACEHOLDER', $usedEncoded)
    $htmlContent = $htmlContent.Replace('TOTAL_FREE_PLACEHOLDER', $freeEncoded)

    # These are HTML fragments and must NOT be encoded
    $htmlContent = $htmlContent.Replace('SECTION_BADGES_PLACEHOLDER', $sectionBadges)
    $htmlContent = $htmlContent.Replace('ROWS_PLACEHOLDER', ($rowsHtml -join "`n"))

    $folder = Split-Path -Path $Path -Parent
    if (-not (Test-Path $folder)) {
        New-Item -Path $folder -ItemType Directory -Force | Out-Null
    }

    [System.IO.File]::WriteAllText($Path, $htmlContent, [System.Text.Encoding]::UTF8)
}

# ------------------------------------------------------------
# Header
# ------------------------------------------------------------
$GeneratedTime = Get-Date

Write-Host ""
Write-Host "==============================================================" -ForegroundColor Cyan
Write-Host "              SAN / RDM Disk Mapping Report" -ForegroundColor Cyan
Write-Host "==============================================================" -ForegroundColor Cyan
Write-Host ("Server      : {0}" -f $env:COMPUTERNAME) -ForegroundColor White
Write-Host ("Generated   : {0}" -f $GeneratedTime.ToString('dd/MM/yyyy HH:mm:ss')) -ForegroundColor White
Write-Host ""

# ------------------------------------------------------------
# Gather physical disks
# ------------------------------------------------------------
try {
    $physicalDisks = Get-PhysicalDisk -ErrorAction Stop | Sort-Object DeviceId
}
catch {
    Write-Host "Failed to query Get-PhysicalDisk. Please run PowerShell as Administrator." -ForegroundColor Red
    Write-Host $_.Exception.Message -ForegroundColor Red
    return
}

# ------------------------------------------------------------
# Build mapping: Drive Letter -> Disk Number
# Using CIM associations (more reliable for clustered disks)
# ------------------------------------------------------------
$diskVolumeMap = @{}

try {
    $logicalDisks = Get-CimInstance -ClassName Win32_LogicalDisk -Filter "DriveType = 3" -ErrorAction Stop |
                    Sort-Object DeviceID
}
catch {
    Write-Host "Failed to query Win32_LogicalDisk." -ForegroundColor Red
    Write-Host $_.Exception.Message -ForegroundColor Red
    return
}

foreach ($logicalDisk in $logicalDisks) {

    $partitions = Get-CimAssociatedInstance -InputObject $logicalDisk -Association Win32_LogicalDiskToPartition -ErrorAction SilentlyContinue

    foreach ($partition in $partitions) {

        $diskDrives = Get-CimAssociatedInstance -InputObject $partition -Association Win32_DiskDriveToDiskPartition -ErrorAction SilentlyContinue

        foreach ($diskDrive in $diskDrives) {

            $diskNumber = $null

            try {
                $diskNumber = [int]$diskDrive.Index
            }
            catch {
                $diskNumber = $null
            }

            if ($null -eq $diskNumber) {
                continue
            }

            if (-not $diskVolumeMap.ContainsKey($diskNumber)) {
                $diskVolumeMap[$diskNumber] = @()
            }

            $existing = $diskVolumeMap[$diskNumber] | Where-Object {
                $_.DriveLetter -eq $logicalDisk.DeviceID
            }

            if (-not $existing) {
                $diskVolumeMap[$diskNumber] += [PSCustomObject]@{
                    DriveLetter = $logicalDisk.DeviceID
                    VolumeLabel = if ($logicalDisk.VolumeName) { $logicalDisk.VolumeName } else { "-" }
                    SizeBytes   = [int64]$logicalDisk.Size
                    FreeBytes   = [int64]$logicalDisk.FreeSpace
                    UsedBytes   = [int64]($logicalDisk.Size - $logicalDisk.FreeSpace)
                }
            }
        }
    }
}

# ------------------------------------------------------------
# Build final report
# ------------------------------------------------------------
$report = @()

foreach ($physicalDisk in $physicalDisks) {

    $deviceId = [int]$physicalDisk.DeviceId
    $matchedVolumes = @()

    if ($diskVolumeMap.ContainsKey($deviceId)) {
        $matchedVolumes = $diskVolumeMap[$deviceId]
    }

    if ($matchedVolumes.Count -gt 0) {
        foreach ($volume in ($matchedVolumes | Sort-Object DriveLetter)) {

            $isSystemDisk = ($volume.DriveLetter -eq 'C:' -or $physicalDisk.FriendlyName -like '*VMware Virtual disk*')
            $sectionName = if ($isSystemDisk) { 'System / Local' } else { 'Cluster / SAN' }
            $sectionSortOrder = if ($isSystemDisk) { 0 } else { 1 }

            $report += [PSCustomObject]@{
                SortOrder             = 0
                SectionSortOrder      = $sectionSortOrder
                Section               = $sectionName
                SortDriveLetter       = $volume.DriveLetter
                SortDiskNumber        = $deviceId
                IsSystemDisk          = $isSystemDisk
                AllocatedBytesRaw     = [decimal]$physicalDisk.Size
                UsedBytesRaw          = [decimal]$volume.UsedBytes
                FreeBytesRaw          = [decimal]$volume.FreeBytes
                'Drive Letter'        = $volume.DriveLetter
                'Volume Label'        = $volume.VolumeLabel
                'Disk Number'         = $deviceId
                'UniqueID'            = $physicalDisk.UniqueId
                'Friendly Name'       = $physicalDisk.FriendlyName
                'Allocated LUN Size'  = Convert-SizeToFriendly -Bytes $physicalDisk.Size
                'Used Space'          = Convert-SizeToFriendly -Bytes $volume.UsedBytes
                'Free Space'          = Convert-SizeToFriendly -Bytes $volume.FreeBytes
            }
        }
    }
    else {
        $report += [PSCustomObject]@{
            SortOrder             = 1
            SectionSortOrder      = 2
            Section               = 'Additional LUN / No Drive Letter'
            SortDriveLetter       = 'ZZZ'
            SortDiskNumber        = $deviceId
            IsSystemDisk          = $false
            AllocatedBytesRaw     = [decimal]$physicalDisk.Size
            UsedBytesRaw          = $null
            FreeBytesRaw          = $null
            'Drive Letter'        = '-'
            'Volume Label'        = '-'
            'Disk Number'         = $deviceId
            'UniqueID'            = $physicalDisk.UniqueId
            'Friendly Name'       = $physicalDisk.FriendlyName
            'Allocated LUN Size'  = Convert-SizeToFriendly -Bytes $physicalDisk.Size
            'Used Space'          = '-'
            'Free Space'          = '-'
        }
    }
}

$report = @($report | Sort-Object SortOrder, SortDriveLetter, SortDiskNumber)

# ------------------------------------------------------------
# Split output into cleaner sections
# ------------------------------------------------------------
$mappedReport   = @($report | Where-Object { $_.'Drive Letter' -ne '-' })
$unmappedReport = @($report | Where-Object { $_.'Drive Letter' -eq '-' })

$systemReport   = @($mappedReport | Where-Object { $_.IsSystemDisk -eq $true }  | Sort-Object SortDriveLetter, SortDiskNumber)
$clusterReport  = @($mappedReport | Where-Object { $_.IsSystemDisk -eq $false } | Sort-Object SortDriveLetter, SortDiskNumber)

# ------------------------------------------------------------
# Display - System / Local Volumes
# ------------------------------------------------------------
Write-Section "System / Local Volumes"

if ($systemReport.Count -gt 0) {
    Get-PrintableReport -InputObject $systemReport |
        Format-Table `
            @{Label='Drive Letter';       Expression={$_.'Drive Letter'};       Width=12}, `
            @{Label='Volume Label';       Expression={$_.'Volume Label'};       Width=20}, `
            @{Label='Disk Number';        Expression={$_.'Disk Number'};        Width=12}, `
            @{Label='UniqueID';           Expression={$_.'UniqueID'};           Width=34}, `
            @{Label='Friendly Name';      Expression={$_.'Friendly Name'};      Width=20}, `
            @{Label='Allocated LUN Size'; Expression={$_.'Allocated LUN Size'}; Width=18}, `
            @{Label='Used Space';         Expression={$_.'Used Space'};         Width=12}, `
            @{Label='Free Space';         Expression={$_.'Free Space'};         Width=12} -AutoSize
}
else {
    Write-Host "No system / local volumes identified." -ForegroundColor Yellow
}

# ------------------------------------------------------------
# Display - Cluster / SAN Mounted Volumes
# ------------------------------------------------------------
Write-Section "Cluster / SAN Mounted Volumes"

if ($clusterReport.Count -gt 0) {
    Get-PrintableReport -InputObject $clusterReport |
        Format-Table `
            @{Label='Drive Letter';       Expression={$_.'Drive Letter'};       Width=12}, `
            @{Label='Volume Label';       Expression={$_.'Volume Label'};       Width=20}, `
            @{Label='Disk Number';        Expression={$_.'Disk Number'};        Width=12}, `
            @{Label='UniqueID';           Expression={$_.'UniqueID'};           Width=34}, `
            @{Label='Friendly Name';      Expression={$_.'Friendly Name'};      Width=20}, `
            @{Label='Allocated LUN Size'; Expression={$_.'Allocated LUN Size'}; Width=18}, `
            @{Label='Used Space';         Expression={$_.'Used Space'};         Width=12}, `
            @{Label='Free Space';         Expression={$_.'Free Space'};         Width=12} -AutoSize
}
else {
    Write-Host "No clustered SAN volumes with drive letters were matched." -ForegroundColor Yellow
}

# ------------------------------------------------------------
# Display - Additional LUNs with no drive letter
# ------------------------------------------------------------
if ($unmappedReport.Count -gt 0) {
    Write-Section "Additional LUNs with no drive letter"

    Get-PrintableReport -InputObject $unmappedReport |
        Format-Table `
            @{Label='Drive Letter';       Expression={$_.'Drive Letter'};       Width=12}, `
            @{Label='Volume Label';       Expression={$_.'Volume Label'};       Width=20}, `
            @{Label='Disk Number';        Expression={$_.'Disk Number'};        Width=12}, `
            @{Label='UniqueID';           Expression={$_.'UniqueID'};           Width=34}, `
            @{Label='Friendly Name';      Expression={$_.'Friendly Name'};      Width=20}, `
            @{Label='Allocated LUN Size'; Expression={$_.'Allocated LUN Size'}; Width=18}, `
            @{Label='Used Space';         Expression={$_.'Used Space'};         Width=12}, `
            @{Label='Free Space';         Expression={$_.'Free Space'};         Width=12} -AutoSize
}

# ------------------------------------------------------------
# Summary
# ------------------------------------------------------------
$totalAllocated = ($mappedReport | Measure-Object -Property AllocatedBytesRaw -Sum).Sum
$totalUsed      = ($mappedReport | Measure-Object -Property UsedBytesRaw -Sum).Sum
$totalFree      = ($mappedReport | Measure-Object -Property FreeBytesRaw -Sum).Sum

if ($null -eq $totalAllocated) { $totalAllocated = 0 }
if ($null -eq $totalUsed)      { $totalUsed = 0 }
if ($null -eq $totalFree)      { $totalFree = 0 }

Write-Section "Summary"

Write-Host ("Mounted volumes matched    : {0}" -f $mappedReport.Count) -ForegroundColor Green
Write-Host ("LUNs without drive letter  : {0}" -f $unmappedReport.Count) -ForegroundColor Yellow
Write-Host ("Total physical disks found : {0}" -f $physicalDisks.Count) -ForegroundColor Cyan
Write-Host ("Total allocated LUN size   : {0}" -f (Convert-SizeToFriendly -Bytes $totalAllocated)) -ForegroundColor White
Write-Host ("Total used space           : {0}" -f (Convert-SizeToFriendly -Bytes $totalUsed)) -ForegroundColor White
Write-Host ("Total free space           : {0}" -f (Convert-SizeToFriendly -Bytes $totalFree)) -ForegroundColor White

# ------------------------------------------------------------
# Optional CSV export
# ------------------------------------------------------------
if ($ExportCsv) {
    $folder = Split-Path -Path $CsvPath -Parent

    if (-not (Test-Path $folder)) {
        New-Item -Path $folder -ItemType Directory -Force | Out-Null
    }

    $report |
        Select-Object 'Drive Letter','Volume Label','Disk Number','UniqueID','Friendly Name','Allocated LUN Size','Used Space','Free Space' |
        Export-Csv -Path $CsvPath -NoTypeInformation -Encoding UTF8

    Write-Host ""
    Write-Host ("CSV exported to: {0}" -f $CsvPath) -ForegroundColor Green
}

# ------------------------------------------------------------
# Optional HTML export
# ------------------------------------------------------------
if ($ExportHtml) {
    Export-ModernSanHtml `
        -Rows $report `
        -Path $HtmlPath `
        -ServerName $env:COMPUTERNAME `
        -GeneratedTime $GeneratedTime `
        -MountedCount $mappedReport.Count `
        -UnmappedCount $unmappedReport.Count `
        -PhysicalDiskCount $physicalDisks.Count `
        -TotalAllocatedBytes ([decimal]$totalAllocated) `
        -TotalUsedBytes ([decimal]$totalUsed) `
        -TotalFreeBytes ([decimal]$totalFree)

    Write-Host ""
    Write-Host ("HTML exported to: {0}" -f $HtmlPath) -ForegroundColor Green

    if ($OpenHtmlAfterExport -and (Test-Path $HtmlPath)) {
        Invoke-Item $HtmlPath
    }
}

Write-Host ""
Write-Host "Report complete." -ForegroundColor Green
Write-Host ""
```

## V1 (legacy)
>Run the Following
```Powershell
Get-PhysicalDisk | fl uniqueid,DeviceID,FriendlyName,@{Name="GB";Expression={$_.size/1GB}}
```
