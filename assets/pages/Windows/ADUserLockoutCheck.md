# Checks for AD User Lockouts in the past 24 hours 🔄🖥️

```Powershell
# =============================================================================
#   Active Directory Account Lockout Checker
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
# 3. Get recent lockout events (Event ID 4740) from PDC
# ------------------------------------------------------------------
$hoursBack = Read-Host "`nHow many hours back should we check lockout events? (default: 48)"

if ([string]::IsNullOrWhiteSpace($hoursBack)) { $hoursBack = 48 }
$startTime = (Get-Date).AddHours(-$hoursBack)

Write-Host "`n📋 Retrieving lockout events (Event ID 4740) from the last $hoursBack hours on $dc..." -ForegroundColor Cyan

try {
    $events = Get-WinEvent -ComputerName $dc -FilterHashtable @{
        LogName   = 'Security'
        ID        = 4740
        StartTime = $startTime
    } -ErrorAction Stop

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
        Write-Host "   • The 'Source Computer' is the device that generated the bad password attempts." -ForegroundColor White
        Write-Host "   • Common causes: outdated credentials on workstations, services, mapped drives, phones, or VPNs." -ForegroundColor White
    } else {
        Write-Host "✅ No lockout events found in the selected time range." -ForegroundColor Green
    }
} catch {
    Write-Host "⚠️  Could not retrieve events from $dc. Error: $($_.Exception.Message)" -ForegroundColor Yellow
    Write-Host "   (Ensure you have permission and WinRM/Firewall allows remote event log access)" -ForegroundColor Yellow
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
Write-Host "You can copy the entire output above and paste it into your support ticket." -ForegroundColor White
Write-Host "💡 Tip: If lockouts continue, investigate the 'Source Computer' for stale credentials." -ForegroundColor Magenta
```