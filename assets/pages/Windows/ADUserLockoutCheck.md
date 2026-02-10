# Checks for AD User Lockouts in the past 24 hours 🔄🖥️

```Powershell
# =============================================================================
#   Active Directory Account Lockout Checker - Enhanced
#   Professional version for client support requests
#   Run directly in PowerShell ISE (paste and press F5)
# =============================================================================

Clear-Host

Write-Host "🔍 Active Directory Account Lockout Analysis" -ForegroundColor Cyan
Write-Host "=============================================" -ForegroundColor Cyan
Write-Host "This script checks current locked accounts and recent lockout events.`n" -ForegroundColor White

# Import required module
try {
    Import-Module ActiveDirectory -ErrorAction Stop
} catch {
    Write-Host "❌ Active Directory module not found. Please run this on a Domain Controller or a machine with RSAT tools installed." -ForegroundColor Red
    break
}

# ------------------------------------------------------------------
# 1. Select Domain Controller (defaults to PDC Emulator)
# ------------------------------------------------------------------
$dcInput = Read-Host "Enter a Domain Controller name (or press Enter to use PDC Emulator)"

if ([string]::IsNullOrWhiteSpace($dcInput)) {
    $dc = (Get-ADDomain).PDCEmulator
    Write-Host "✅ Using PDC Emulator: $dc" -ForegroundColor Green
} else {
    $dc = $dcInput
    Write-Host "✅ Using specified Domain Controller: $dc" -ForegroundColor Green
}

# ------------------------------------------------------------------
# 2. Check currently locked-out accounts
# ------------------------------------------------------------------
Write-Host "`n📋 Checking currently locked-out user accounts..." -ForegroundColor Cyan

$lockedAccounts = Search-ADAccount -LockedOut -UsersOnly | 
    Get-ADUser -Properties lockoutTime, LastBadPasswordAttempt, LastLogonDate | 
    Select-Object @{
        Name = "Name"
        Expression = { $_.Name }
    }, @{
        Name = "SamAccountName"
        Expression = { $_.SamAccountName }
    }, @{
        Name = "Lockout Time"
        Expression = { if ($_.lockoutTime) { [DateTime]::FromFileTime($_.lockoutTime) } else { "N/A" } }
    }, @{
        Name = "Last Bad Password"
        Expression = { $_.LastBadPasswordAttempt }
    }, @{
        Name = "Last Logon"
        Expression = { $_.LastLogonDate }
    }

if ($lockedAccounts.Count -eq 0) {
    Write-Host "✅ No user accounts are currently locked out." -ForegroundColor Green
} else {
    Write-Host "⚠️  $($lockedAccounts.Count) locked-out account(s) found:" -ForegroundColor Yellow
    $lockedAccounts | Format-Table -AutoSize
}

# ------------------------------------------------------------------
# 3. Get lockout events (Event ID 4740) from PDC
# ------------------------------------------------------------------
Write-Host "`n📋 Lockout event lookup (Event ID 4740)" -ForegroundColor Cyan
Write-Host "Enter number of hours back to check (e.g. 24, 72), or type 'all' for all available events" -ForegroundColor White
$input = Read-Host "Hours back or 'all' (default: 48)"

$useAllEvents = $false
$hoursBack = 48

if ([string]::IsNullOrWhiteSpace($input)) {
    $hoursBack = 48
} elseif ($input -match '^\d+$') {
    $hoursBack = [int]$input
} elseif ($input -eq 'all' -or $input -eq 'ALL' -or $input -eq 'All') {
    $useAllEvents = $true
    Write-Host "→ Retrieving ALL available lockout events (this may take a moment on busy DCs)" -ForegroundColor Magenta
} else {
    Write-Host "→ Unrecognised input — falling back to default 48 hours" -ForegroundColor Yellow
    $hoursBack = 48
}

Write-Host "`nRetrieving lockout events from $dc..." -ForegroundColor Cyan

try {
    $filter = @{
        LogName   = 'Security'
        ID        = 4740
    }

    if (-not $useAllEvents) {
        $startTime = (Get-Date).AddHours(-$hoursBack)
        $filter['StartTime'] = $startTime
        Write-Host "→ Time range: last $hoursBack hours" -ForegroundColor White
    } else {
        Write-Host "→ Time range: all available events" -ForegroundColor White
    }

    $events = Get-WinEvent -ComputerName $dc -FilterHashtable $filter -ErrorAction Stop

    $eventDetails = $events | ForEach-Object {
        [PSCustomObject]@{
            Time             = $_.TimeCreated
            User             = $_.Properties[0].Value   # TargetUserName
            "Source Computer" = $_.Properties[1].Value  # CallerComputerName
        }
    }

    if ($eventDetails) {
        Write-Host "⚠️  $($eventDetails.Count) lockout event(s) found:" -ForegroundColor Yellow
        $eventDetails | Sort-Object Time -Descending | Format-Table -AutoSize
        
        Write-Host "`n💡 Key Information:" -ForegroundColor Magenta
        Write-Host "   • 'Source Computer' = device responsible for the bad password attempts" -ForegroundColor White
        Write-Host "   • Common culprits: old credentials in services, mapped drives, mobile devices, scheduled tasks, or stale VPN sessions" -ForegroundColor White
        Write-Host "   • If many events → look for patterns (same user + same source over time)" -ForegroundColor White
    } else {
        Write-Host "✅ No lockout events found in the selected range." -ForegroundColor Green
    }
} catch {
    Write-Host "⚠️  Could not retrieve events from $dc. Error: $($_.Exception.Message)" -ForegroundColor Yellow
    Write-Host "   Common fixes: run as Domain Admin, check WinRM/firewall rules, or try running directly on the PDC" -ForegroundColor Yellow
}

# ------------------------------------------------------------------
# 4. Unlock accounts (optional)
# ------------------------------------------------------------------
if ($lockedAccounts.Count -gt 0) {
    Write-Host "`n🔓 Account Unlock Section" -ForegroundColor Cyan
    $unlockChoice = Read-Host "Would you like to unlock any accounts? (Y/N)"

    while ($unlockChoice -eq 'Y' -or $unlockChoice -eq 'y') {
        $userToUnlock = Read-Host "Enter the SamAccountName to unlock (or press Enter to cancel)"
        
        if ([string]::IsNullOrWhiteSpace($userToUnlock)) { break }
        
        try {
            Unlock-ADAccount -Identity $userToUnlock -Server $dc -ErrorAction Stop
            Write-Host "✅ Successfully unlocked: $userToUnlock" -ForegroundColor Green
        } catch {
            Write-Host "❌ Failed to unlock $userToUnlock : $($_.Exception.Message)" -ForegroundColor Red
        }
        
        $unlockChoice = Read-Host "Unlock another account? (Y/N)"
    }
}

# ------------------------------------------------------------------
# 5. Final summary
# ------------------------------------------------------------------
Write-Host "`n=============================================" -ForegroundColor Cyan
Write-Host "✅ Analysis complete!" -ForegroundColor Green
Write-Host "You can copy the entire output above and paste it into your support ticket or email." -ForegroundColor White
Write-Host "💡 Next step if lockouts persist: focus on the most frequent 'Source Computer(s)' and clear cached/stale credentials there." -ForegroundColor Magenta
```