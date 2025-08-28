# 🔑 PowerShell Password Generator

Easily generate secure, random passwords using PowerShell.

---

## 🚀 Usage

Copy and paste the following function into your PowerShell session or script.  
You can specify the desired password length and generate as many passwords as you need.

```powershell
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
        $result[$i] = $charSet[$bytes[$i] % $charSet.Length]
    }
    return -join $result
}

# Generate 20 random passwords, each 14 characters long
for ($i = 1; $i -le 20; $i++) 
{
    Generate-RandomPassword 14
}
```

---

## 📝 Example Output

```
tLm149BsRayufQ
JbVybzGsvngJpD
zn%WUH@vFkG1!7
w6tQUJO2tHewDy
mjSVVcctlkN1E*
@H!fNu&KYHFfcF
bsiCL1a4QKIz3O
n2MGyuKTUDhO47
IiM&lxna&%GOS0
hb^qMmC22zBWuF
y*xs4UESkLaqU@
Fhp0#dAaVCiRcc
tw3we95s9SKPPr
J0rFT1sxJ33lrN
kN5mX!RsoYOGmc
JnAsrUKymBjBEX
i&FSFDmzxC4LHt
GOFeOlW#0I25#J
@%0yLJCoaeqIsI
xM5F$lXdLG!2f9
```

---

## ⚠️ Security Notes

- Uses cryptographically secure random number generation.
- You can adjust the `$charSet` to include/exclude special characters as needed.
- Always store passwords securely!

---

## 💡 Tip

To generate a single password of custom length, run:

```powershell
Generate-RandomPassword 20
```