# How to manage Microsoft Defender Antivirus with PowerShell

[comment]: <> (https://www.windowscentral.com/how-manage-microsoft-defender-antivirus-powershell-windows-10#check_status_defender_powershell)

[comment]: <> (Next is How to delete active threat on Microsoft Defender)

You can manage settings and control virtually any aspect of the Microsoft Defender Antivirus using PowerShell commands, and in this guide, we'll help you get started.

---

## How to check status of Microsoft Defender
1. Open Start
2. Search for PowerShell, right-click the top result, and select the Run as administrator option.
3. Type the following command to see the Microsoft Defender Antivirus status and press Enter: 
```Powershell
Get-MpComputerStatus
```
<img src="https://github.com/Ashdf1992/wiki/blob/main/assets/images/1.get-mpcomputerstatus-cmmand_.jpg"/>

> In addition to checking whether the antivirus is running, the command output also displays other important information, such as the version of the engine and product version, real-time protection status, last time updated, and more.
{.is-info}

<br>

## How to check for updates on Microsoft Defender
1. Open Start
2. Search for PowerShell, right-click the top result, and select the Run as administrator option.
3. Type the following command to check to update Microsoft Defender Antivirus and press Enter:
```Powershell
Update-MpSignature
```
<img src="https://github.com/Ashdf1992/wiki/blob/main/assets/images/2.update-mpsignature-microsoft-defender-antivirus.jpg"/>

> Once you complete the steps, if new updates are available, they will download and install on your device.
{.is-info}

<br>

## How to perform quick virus scan with Microsoft Defender
1. Open Start
2. Search for PowerShell, right-click the top result, and select the Run as administrator option
3. Type the following command to start a quick virus scan and press Enter:
```Powershell
Start-MpScan -ScanType QuickScan
```
<img src="https://github.com/Ashdf1992/wiki/blob/main/assets/images/3.microsoft-defender-quick-scann-powershell.jpg"/>

> After you complete the steps, Microsoft Defender Antivirus will perform a quick virus scan on your device.
{.is-info}

<br>

## How to perform full virus scan with Microsoft Defender
1. Open Start
2. Search for PowerShell, right-click the top result, and select the Run as administrator option.
3. Type the following command to start a full virus scan and press Enter:
```Powershell
Start-MpScan -ScanType FullScan
```
<img src="https://github.com/Ashdf1992/wiki/blob/main/assets/images/4.defender-av-fullscan-powershell.webp"/>

> Once you complete the steps, Windows Defender will scan the entire system for any malware and malicious code.
{.is-info}

<br>

## How to perform full virus scan with Microsoft Defender
1. Open Start
2. Search for PowerShell, right-click the top result, and select the Run as administrator option
3. Type the following command to perform a custom Microsoft Defender Antivirus scan and press Enter:
```Powershell
Start-MpScan -ScanType CustomScan -ScanPath PATH\TO\FOLDER-FILES
```
> In the command, make sure to update the path with the folder location you want to scan. For example, this command scans the Downloads folder:
{.is-info}
```Powershell
Start-MpScan -ScanType CustomScan -ScanPath "C:\Users\user\Downloads"
```
<img src="https://github.com/Ashdf1992/wiki/blob/main/assets/images/5.start-mpscan-custom-folder-powershell.jpg"/>

> After you complete the steps, Microsoft Defender will only scan for viruses in the location you specified.
{.is-info}

<br>

## How to perform offline virus scan with Microsoft Defender
> Before proceeding, make sure to save any work you may have open, as the command will immediately restart the device to perform an offline scan.
{.is-warning}

1. Open Start
2. Search for PowerShell, right-click the top result, and select the Run as administrator option
3. Type the following command to start an offline virus scan and press Enter:
```Powershell
Start-MpWDOScan
```
> Once you complete the steps, the device will restart automatically. It'll boot into the recovery environment, and it'll perform a full scan to remove viruses that otherwise wouldn't be possible to detect during the normal operation of Windows 10. After the scan, the device will restart automatically, and then you can view the scan report on Windows Security > Virus & thread protection > Protection history.
{.is-info}

<br>

## How to delete active threat on Microsoft Defender
1. Open Start
2. Search for PowerShell, right-click the top result, and select the Run as administrator option
3. Type the following command to eliminate active threat using Microsoft Defender and press Enter:
```Powershell
Remove-MpThreat
```
<img src="https://github.com/Ashdf1992/wiki/blob/main/assets/images/6.remove-active-virus-powershell.jpg"/>

> After you complete the steps, the anti-malware solution will eliminate any active threats on the computer. Although this is an interesting command, it'll only work for threats that the antivirus hasn't already mitigated.
{.is-success}

<br>

## How to change preferences on Microsoft Defender
1. Open Start
2. Search for PowerShell, right-click the top result, and select the Run as administrator option
3. Type the following command to get a full list of the current configurations for the Microsoft Defender Antivirus and press Enter:
```Powershell
Get-MpPreference
```
<img src="https://github.com/Ashdf1992/wiki/blob/main/assets/images/7.microsoft-defender-preferences-powershell-command.jpg"/>

> Once you complete the steps, you'll understand all the settings that you can configure with the built-in antivirus. The following commands are some examples of the preferences that you can customize using PowerShell.
{.is-info}

<br>


### Exclude locations
**Add**
```Powershell
Set-MpPreference -ExclusionPath "C:\Users"
```
**Remove**
```Powershell
Remove-MpPreference -ExclusionPath "C:\Users"
```

---

<br>

### Exclude file type
**Add**
```Powershell
Set-MpPreference -ExclusionExtension docx
```
**Remove**
```Powershell
Remove-MpPreference -ExclusionExtension docx
```

---

<br>

### Quarantine time before deletion
**Add**
```Powershell
Set-MpPreference -QuarantinePurgeItemsAfterDelay 30
```
**Default**
```Powershell
Set-MpPreference -QuarantinePurgeItemsAfterDelay 90
```

---

<br>

### Schedule quick scan
**Add Schedule**
```Powershell
Set-MpPreference -ScanScheduleQuickScanTime 06:00:00
```
**Remove Schedule**
```Powershell
Set-MpPreference -ScanScheduleQuickScanTime 00:00:00
```

---

<br>

### Schedule full scan
**Add Schedule**
```Powershell
Set-MpPreference -ScanParameters 2
Set-MpPreference -RemediationScheduleDay 1
Set-MpPreference -RemediationScheduleTime 06:00:00
```
0 – Everyday
1 – Sunday
2 – Monday
3 – Tuesday
4 – Wednesday
5 – Thursday
6 – Friday
7 – Saturday
8 – Never
> When setting RemediationScheduleDate this is a numerical value that coincides with the specific day that you wish to run the scan on. See the list above for more details
{.is-info}

**Remove Schedule**
```Powershell
Set-MpPreference -RemediationScheduleDay 8
```

---

<br>

### Enable external drive scanning
**Enable**
```Powershell
Set-MpPreference -DisableRemovableDriveScanning $false
```
**Disable**
```Powershell
Set-MpPreference -DisableRemovableDriveScanning $true
```

---

<br>

### Disable archive scanning
**Disable**
```Powershell
Set-MpPreference -DisableArchiveScanning $true
```

**Defaults**
```Powershell
Set-MpPreference -DisableArchiveScanning $false
```

---

<br>

### Enable network drive scanning
**Enable**
```Powershell
Set-MpPreference -DisableScanningMappedNetworkDrivesForFullScan $false
```

**Defaults**
```Powershell
Set-MpPreference -DisableScanningMappedNetworkDrivesForFullScan $true
```
