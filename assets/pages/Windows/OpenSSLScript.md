# OpenSSL Powershell Script
## This script will allow you to manage SSL Certificates using OpenSSL. Note that the script must be ran from the same location that the openssl.exe is located

>Open Powershell ISE

>Run the Following
```Powershell
$time = Get-Date
        function Show-Menu
        {
            param ([string]$Title = 'OpenSSL Powershell Script')
            cls
            Write-Host "Time of initial run:";
            $time; 
            Write-Host "";
            Write-Host "Current Time:";
            Get-Date;
            Write-Host "";
            Write-Host "OpenSSL Powershell Script";
            Write-Host "Please Select an option from below";
            Write-Host "";
            Write-Host "";
            Write-Host "================ $Title ================";
            Write-Host "Open SSL";
            Write-Host "1: Press '1' to Export a Private Key from a PFX File";
            Write-Host "";
            Write-Host "2: Press '2' to Convert a PFX to CRT";
            Write-Host "";
            Write-Host "3: Press '3' to Decrypt an Encrypted Private Key";
            Write-Host "";
            Write-Host "4: Press '4' to Convert a Chain (.p7b) to a CRT";
            Write-Host "";
            Write-Host "5: Press '5' to Combine CRT and KEY to PFX";
            Write-Host "";
            Write-Host "";
			Write-Host "6: Press '5' to Combine CRT, KEY and CA-Bundle (Chain) to PFX";
            Write-Host "";
            Write-Host "";
            Write-Host "Q: Press 'Q' to quit.";
        }
        do
        {
            Show-Menu
            Write-host ""
            $input = Read-Host "Please make a selection"
            switch ($input)
            {
                '1' {
                        cls
                        'You chose option #1'
                        $SSL = Read-Host "Please enter the name of the PFX Certificate you want to export the private key from, excluding the extension (eg www.supercars.co.uk): "
						.\openssl.exe pkcs12 -in "$SSL.pfx" -nocerts -out "$SSL.key"
                    } 
					
                '2' {
                        cls
                        'You chose option #2'
                        $SSL = Read-Host "Please enter the name of the PFX Certificate you want to export the Certificate from, excluding the extension (eg www.supercars.co.uk): "
						.\openssl pkcs12 -in "$SSL.pfx" -clcerts -nokeys -out "$SSL.crt"
                    }

                
                '3' {
                        cls
                        'You chose option #3'
                        $SSL = Read-Host "Please enter the name of the KEY File that you wish to decrypt, excluding the extension (eg www.supercars.co.uk): "
						.\openssl rsa -in "$SSL.key" -outform PEM -out "$SSL.decrypt.key"
                    } 
					

                '4' {
                       cls
					   'You chose option #4'
					   $SSL = Read-Host "Please enter the name of the p7b File that you wish to convert to CRT, excluding the extension (eg gd-g2_iis_intermediates): "
					   .\openssl pkcs7 -print_certs -in "$SSL.p7b" -out "$SSL.crt"
                    } 
                 
                '5' {
                        cls
                        'You chose option #5'
						$PFX = Read-Host "Please enter the desired name of the PFX file you wish to be created, excluding the extension (eg www.supercars-pfx-export): "
						$DecyptedKey = Read-Host "Please enter the name of the Decypted Private Key, excluding the extension, also exclude .decypted (eg www.supercars.co.uk): "
						$SSL = Read-Host "Please enter the name of the CRT File, excluding the extension (eg www.supercars.co.uk): "
                        .\openssl.exe pkcs12 -export -out "$PFX.pfx" -inkey "$DecyptedKey.decrypt.key" -in "$SSL.crt"
                    } 

                '6' {
                        cls
                        'You chose option #5'
						$PFX = Read-Host "Please enter the desired name of the PFX file you wish to be created, excluding the extension (eg www.supercars-pfx-export): "
						$DecyptedKey = Read-Host "Please enter the name of the Decypted Private Key, excluding the extension (eg www.supercars.co.uk): "
						$SSL = Read-Host "Please enter the name of the CRT File, excluding the extension (eg www.supercars.co.uk): "
						$CABundle = Read-Host "Please enter the name of the CA Bundle, excluding the extension (eg gd-g2_iis_intermediates): "
                        .\openssl.exe pkcs12 -export -out "$PFX.pfx" -inkey "$DecryptedKey.key" -in "$SSL.crt" -certfile "$CABundle.crt"
                    }     					
                                        					
                'q' {
                        return
                    }
			}
            pause
        }
until ($input -eq 'q')
```

## Alternatively, click here to go to the latest zip folder containing both the script and OpenSSL. Simply right click OpenSSLScript.ps1 and click on 'Run with PowerShell'
[OpenSSL Powershell Script Download](https://github.com/Ashdf1992/wiki/blob/main/assets/attachments/OpenSSL-Script.zip)

