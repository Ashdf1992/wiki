# Powershell Password Generator

``` powershell
function Generate-RandomPassword 
{
    param ([Parameter(Mandatory)][int] $length)
    $charSet = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789!@#*$%£^&'.ToCharArray()
    $rng = New-Object System.Security.Cryptography.RNGCryptoServiceProvider
    $bytes = New-Object byte[]($length)
    $rng.GetBytes($bytes)
    $result = New-Object char[]($length)
  
    for ($i = 0 ; $i -lt $length ; $i++) 
    {
        $result[$i] = $charSet[$bytes[$i]%$charSet.Length]
    }
 
    return -join $result
}

$i = 1
while ($i -le 20)
{
      Generate-RandomPassword 14
      $i = $i + 1
}
```
