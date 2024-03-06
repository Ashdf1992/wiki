# Installing a Language Pack on Windows Server 2022 using the Feature on Demand ISO

### Useful Websites:
<br>
https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-2022
<br>
<br>
https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/add-language-packs-to-windows?view=windows-11#build-a-custom-fod-and-language-pack-repository
<br>
<br>

### Process:
> Firstly you will need to download the ISO, from the following link:
> https://go.microsoft.com/fwlink/p/?linkid=2195333

<br>

> Then mount the ISO

<br>

> Go to 'LanguagesAndOptionalFeatures'

<br>

> Find the language pack you need IE 'Microsoft-Windows-Server-Language-Pack_x64_en-gb.cab'

<br>

> Copy the Language Pack locally.

<br>

> Go to Start

<br>

> Type Powershell.exe

<br>

```Powershell
lpksetup.exe
```

<br>

> Browse to the location where you stored the Cab File, and install. 


[comment]: <Source> (https://stackoverflow.com/questions/3487265/powershell-script-to-return-versions-of-net-framework-on-a-machine)
