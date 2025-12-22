# Searching IIS Log Files

## Summary

This PowerShell script searches IIS W3C-format .log files in the current directory and presents matching entries in a clear, colorized, professional format.

Features
- Locates .log files in the current working directory and lists them for selection (single file or All).
- Hour filtering: select a specific hour (00–23), All hours, or Last 24 hours (timestamp-based rolling window).
- Status-code filtering: validated HTTP status code input (100–599).
- Accurate timestamp parsing for IIS W3C logs; skips header/comment lines.
- Colorized console output with either line numbers or timestamps and highlighted hour/status tokens.
- Per-file counts and a final total summary.
- PowerShell 5.0+ compatible.

Usage
- Run in PowerShell (ISE or console) from the directory containing your IIS .log files.
- Choose a file index or "A" for all files, pick an hour mode (00–23, A, or 24), then enter the HTTP status code to filter.
- Results show matching lines with context, highlighting, and a summary table.

Notes
- "Last 24 hours" uses parsed timestamps (more accurate than simple token matching).
- Exact token formatting is required for reliable matching (e.g., status token formatted as " - 500 ").
- If a line’s timestamp cannot be parsed in 24-hour mode, that line is skipped to avoid false positives.

<br>

```Powershell

# ===========================
# IIS Log Filter (Hour + HTTP Status Code)
# PowerShell 5 compatible - paste into PowerShell ISE and Run
# ===========================

if ($PSVersionTable.PSVersion.Major -lt 5) {
    Write-Host "This script requires PowerShell 5.0 or newer." -ForegroundColor Yellow
    return
}

Set-StrictMode -Version 2.0
$ErrorActionPreference = 'Stop'

function Write-Section {
    param(
        [Parameter(Mandatory=$true)][string]$Text,
        [ConsoleColor]$Color = 'Cyan'
    )
    Write-Host ""
    Write-Host ("=" * 78) -ForegroundColor DarkGray
    Write-Host ("  " + $Text) -ForegroundColor $Color
    Write-Host ("=" * 78) -ForegroundColor DarkGray
}

function Read-ValidatedInt {
    param(
        [Parameter(Mandatory=$true)][string]$Prompt,
        [Parameter(Mandatory=$true)][int]$Min,
        [Parameter(Mandatory=$true)][int]$Max
    )

    while ($true) {
        $raw = Read-Host $Prompt
        $val = 0
        if ([int]::TryParse($raw, [ref]$val) -and $val -ge $Min -and $val -le $Max) {
            return $val
        }
        Write-Host "Invalid input. Enter a number between $Min and $Max." -ForegroundColor Yellow
    }
}

function Write-HighlightedLine {
    param(
        [Parameter(Mandatory=$true)][string]$Line,
        [string]$HourToken,   # e.g. " 19:" (can be $null/empty)
        [Parameter(Mandatory=$true)][string]$CodeToken    # e.g. " - 500 "
    )

    # Find token positions (first occurrence)
    $cIndex = $Line.IndexOf($CodeToken, [StringComparison]::Ordinal)

    if ([string]::IsNullOrWhiteSpace($HourToken)) {
        # Only highlight status token
        if ($cIndex -lt 0) {
            Write-Host $Line -ForegroundColor Gray
            return
        }

        $before = $Line.Substring(0, $cIndex)
        $tok    = $Line.Substring($cIndex, $CodeToken.Length)
        $after  = $Line.Substring($cIndex + $CodeToken.Length)

        Write-Host $before -NoNewline -ForegroundColor Gray
        Write-Host $tok    -NoNewline -ForegroundColor Red
        Write-Host $after  -ForegroundColor Gray
        return
    }

    $hIndex = $Line.IndexOf($HourToken, [StringComparison]::Ordinal)

    # If tokens can't be found, write the line plainly
    if ($hIndex -lt 0 -or $cIndex -lt 0) {
        Write-Host $Line -ForegroundColor Gray
        return
    }

    # Print in segments and highlight tokens
    $tokens = @(
        [pscustomobject]@{ Index=$hIndex; Length=$HourToken.Length; Color='Cyan' },
        [pscustomobject]@{ Index=$cIndex; Length=$CodeToken.Length; Color='Red'  }
    ) | Sort-Object Index

    $pos = 0
    foreach ($t in $tokens) {
        if ($t.Index -gt $pos) {
            Write-Host ($Line.Substring($pos, $t.Index - $pos)) -NoNewline -ForegroundColor Gray
        }
        Write-Host ($Line.Substring($t.Index, $t.Length)) -NoNewline -ForegroundColor $t.Color
        $pos = $t.Index + $t.Length
    }

    if ($pos -lt $Line.Length) {
        Write-Host ($Line.Substring($pos)) -ForegroundColor Gray
    } else {
        Write-Host ""
    }
}


function Try-ParseIisTimestamp {
    param(
        [Parameter(Mandatory=$true)][string]$Line,
        [ref]$Timestamp
    )

    # Skip IIS comments/headers
    if ($Line.StartsWith("#")) { return $false }

    # Typical IIS W3C format: yyyy-MM-dd HH:mm:ss ...
    # Split on whitespace, take first two columns
    $parts = $Line -split '\s+'
    if ($parts.Count -lt 2) { return $false }

    $dtString = "$($parts[0]) $($parts[1])"

    # Strongly type the out variable (important for PS5)
    $dt = [datetime]::MinValue

    $culture = [System.Globalization.CultureInfo]::InvariantCulture
    $styles  = [System.Globalization.DateTimeStyles]::AssumeLocal

    # Prefer exact format for IIS W3C
    if ([datetime]::TryParseExact($dtString, 'yyyy-MM-dd HH:mm:ss', $culture, $styles, [ref]$dt)) {
        $Timestamp.Value = $dt
        return $true
    }

    # Fallback: try a more forgiving parse if exact fails
    if ([datetime]::TryParse($dtString, [ref]$dt)) {
        $Timestamp.Value = $dt
        return $true
    }

    return $false
}


# ---------------------------
# Main
# ---------------------------

Write-Section -Text "IIS Log Filter — Hour + HTTP Status Code" -Color Cyan

# 1) Find .log files in the current directory
$logFiles = Get-ChildItem -File -Filter *.log -ErrorAction SilentlyContinue | Sort-Object Name

if (-not $logFiles -or $logFiles.Count -eq 0) {
    Write-Host "No .log files found in: $((Get-Location).Path)" -ForegroundColor Yellow
    return
}

Write-Host "Log files found in current directory:" -ForegroundColor Green
for ($i = 0; $i -lt $logFiles.Count; $i++) {
    Write-Host ("  [{0}] {1}" -f $i, $logFiles[$i].Name) -ForegroundColor White
}
Write-Host ""
Write-Host "Select a file index, or type 'A' for ALL files." -ForegroundColor DarkCyan

# 2) Choose file(s)
$selection = Read-Host "Your choice"
$selectedFiles = @()

if ($selection -match '^(?i)a$') {
    $selectedFiles = $logFiles
} else {
    $idx = -1
    if ([int]::TryParse($selection, [ref]$idx) -and $idx -ge 0 -and $idx -lt $logFiles.Count) {
        $selectedFiles = @($logFiles[$idx])
    } else {
        Write-Host "Invalid selection. Exiting." -ForegroundColor Yellow
        return
    }
}

# 3) Hour selection menu (like file selection)
Write-Host ""
Write-Host "Hour options:" -ForegroundColor Green
Write-Host "  [00] through [23]  -> filter by that hour token (e.g. [19] means ' 19:')" -ForegroundColor White
Write-Host "  [A]               -> ALL hours (no hour filtering)" -ForegroundColor White
Write-Host "  [24]              -> Last 24 hours (rolling window, timestamp-based)" -ForegroundColor White
Write-Host ""

$hourMode = Read-Host "Choose hour (00-23), 'A', or '24'"

$useAllHours     = $false
$useLast24Hours  = $false
$hourToken       = $null

switch -Regex ($hourMode) {
    '^(?i)a$' {
        $useAllHours = $true
        $hourToken = $null
        break
    }
    '^24$' {
        $useLast24Hours = $true
        $hourToken = $null
        break
    }
    '^\d{1,2}$' {
        $h = 0
        if ([int]::TryParse($hourMode, [ref]$h) -and $h -ge 0 -and $h -le 23) {
            # IMPORTANT: exact formatting " 19:" (leading space)
            $hourToken = (" {0:D2}:" -f $h)
            break
        }
        Write-Host "Invalid hour. Exiting." -ForegroundColor Yellow
        return
    }
    default {
        Write-Host "Invalid hour selection. Exiting." -ForegroundColor Yellow
        return
    }
}

# 4) Prompt for status code (validated)
$code = Read-ValidatedInt -Prompt "Enter HTTP status code (100-599). Example: 500" -Min 100 -Max 599

# IMPORTANT: exact formatting " - 500 "
$codeToken = (" - {0} " -f $code)

# Describe mode
$modeDesc = if ($useLast24Hours) { "Last 24 hours (rolling window)" }
           elseif ($useAllHours) { "ALL hours (no hour filter)" }
           else { "Hour token filter: [$hourToken]" }

Write-Section -Text ("Filtering: {0} AND Status token: [{1}]" -f $modeDesc, $codeToken) -Color Magenta
Write-Host ("Selected file(s): {0}" -f ($selectedFiles.Name -join ", ")) -ForegroundColor DarkGray

# 5) Search and output
$totalMatches = 0
$perFileSummary = @()

$cutoff = (Get-Date).AddHours(-24)

foreach ($f in $selectedFiles) {

    Write-Host ""
    Write-Host ("File: {0}" -f $f.Name) -ForegroundColor Green
    Write-Host ("-" * 78) -ForegroundColor DarkGray

    $count = 0

    if ($useLast24Hours) {
        # Timestamp-based scan (accurate last-24-hours window)
        # We read line-by-line so we can parse timestamps.
        foreach ($line in [System.IO.File]::ReadLines($f.FullName)) {

            if ($line.StartsWith("#")) { continue } # skip headers/comments

            # Must contain status token
            if (-not $line.Contains($codeToken)) { continue }

            $ts = $null
            if (Try-ParseIisTimestamp -Line $line -Timestamp ([ref]$ts)) {
                if ($ts -lt $cutoff) { continue }
            } else {
                # If timestamp can't be parsed, skip (safer than false positives)
                continue
            }

            $count++
            $totalMatches++

            # No line number available with ReadLines; show timestamp prefix instead
            Write-Host ("[{0}] " -f $ts.ToString("yyyy-MM-dd HH:mm:ss")) -NoNewline -ForegroundColor DarkGray
            Write-HighlightedLine -Line $line -HourToken $null -CodeToken $codeToken
        }
    }
    elseif ($useAllHours) {
        # No hour filter: just status token
        $hits = Select-String -Path $f.FullName -Pattern $codeToken -SimpleMatch
        foreach ($h in $hits) {
            $count++
            $totalMatches++
            Write-Host ("[{0,6}] " -f $h.LineNumber) -NoNewline -ForegroundColor DarkGray
            Write-HighlightedLine -Line $h.Line -HourToken $null -CodeToken $codeToken
        }
    }
    else {
        # Hour token + status token
        $hits = Select-String -Path $f.FullName -Pattern $hourToken -SimpleMatch |
                Where-Object { $_.Line.Contains($codeToken) }

        foreach ($h in $hits) {
            $count++
            $totalMatches++
            Write-Host ("[{0,6}] " -f $h.LineNumber) -NoNewline -ForegroundColor DarkGray
            Write-HighlightedLine -Line $h.Line -HourToken $hourToken -CodeToken $codeToken
        }
    }

    $perFileSummary += [pscustomobject]@{
        File  = $f.Name
        Count = $count
    }

    if ($count -eq 0) {
        Write-Host "No matches in this file." -ForegroundColor Yellow
    } else {
        Write-Host ("Matches in file: {0}" -f $count) -ForegroundColor Cyan
    }
}

# 6) Summary
Write-Section -Text "Summary" -Color Cyan
$perFileSummary | Sort-Object Count -Descending | Format-Table -AutoSize

Write-Host ""
Write-Host ("TOTAL matching events: {0}" -f $totalMatches) -ForegroundColor Green
Write-Host ""
