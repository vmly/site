---
title: "WSL2 and Beyond - Linux on Windows"
date: 2020-05-01T02:01:58+05:30
description: "No more Virtual Machines for Development ! Use WSL2 on Windows for Development. Common Issues and Customizing WSL2"
tags: [Primer, todo]
---

## Common Issues in WSL2 and Workarounds

Install Directions : https://docs.microsoft.com/en-us/windows/wsl/install

1. Static IP

- IP Changes everytime Windows is restarted
- Check using ipconfig on powershell

2. WSL genrates host address everytime
- Avoid this by creating wsl.conf
- Add IP from ipconfig as a hostname [resolv.conf]
- Find ip of the machine with `ipconfig`
File: resolv.conf
``` conf
nameserver IP_WINDOWS_MACHINE
nameserver 8.8.8.8     
nameserver 8.8.4.4     
nameserver 172.29.112.1
```

3. Time out of Sync

``` sh
sudo apt install ntpdate
sudo ntpdate -sb time.nist.gov
```

4. WSL Networking Problems

- Accessing Windows apps from WSL (Ingress)
- Accessing WSL Apps from Windows (Egress)

- Port Forwarding with Firewall rules, adding listeners
- Creating a Windows service to manage this

File : `wsl_forwarding.ps1` , run with `./wsl_forwarding.ps1`
```ps1 
# WSL2 network port forwarding script v1
#   for enable script, 'Set-ExecutionPolicy -ExecutionPolicy Bypass -Scope CurrentUser' in Powershell,
#   for delete exist rules and ports use 'delete' as parameter, for show ports use 'list' as parameter.
#   written by Daehyuk Ahn, Aug-1-2020

# Display all portproxy information
If ($Args[0] -eq "list") {
    netsh interface portproxy show v4tov4;
    exit;
} 

# If elevation needed, start new process
If (-NOT ([Security.Principal.WindowsPrincipal] [Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator))
{
  # Relaunch as an elevated process:
  Start-Process powershell.exe "-File",('"{0}"' -f $MyInvocation.MyCommand.Path),"$Args runas" -Verb RunAs
  exit
}

# You should modify '$Ports' for your applications 
$Ports = (22,80,443,8000, 8001, 8002, 8003, 8080, 27017, 6379)

# Check WSL ip address
wsl -d Ubuntu-18.04 hostname -I | Set-Variable -Name "WSL"
$found = $WSL -match '\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}';
if (-not $found) {
  echo "WSL2 cannot be found. Terminate script.";
  exit;
}

# Remove and Create NetFireWallRule
Remove-NetFireWallRule -DisplayName 'WSL 2 Firewall Unlock';
if ($Args[0] -ne "delete") {
  New-NetFireWallRule -DisplayName 'WSL 2 Firewall Unlock' -Direction Outbound -LocalPort $Ports -Action Allow -Protocol TCP;
  New-NetFireWallRule -DisplayName 'WSL 2 Firewall Unlock' -Direction Inbound -LocalPort $Ports -Action Allow -Protocol TCP;
}

# Add each port into portproxy
$Addr = "0.0.0.0"
Foreach ($Port in $Ports) {
    iex "netsh interface portproxy delete v4tov4 listenaddress=$Addr listenport=$Port | Out-Null";
    if ($Args[0] -ne "delete") {
        iex "netsh interface portproxy add v4tov4 listenaddress=$Addr listenport=$Port connectaddress=$WSL connectport=$Port | Out-Null";
    }
}

# Display all portproxy information
netsh interface portproxy show v4tov4;

# Give user to chance to see above list when relaunched start
If ($Args[0] -eq "runas" -Or $Args[1] -eq "runas") {
  Write-Host -NoNewLine 'Press any key to close! ';
  $null = $Host.UI.RawUI.ReadKey('NoEcho,IncludeKeyDown');
}
```

5. Accessing MongoDB on WSL 

- Ip Bind to 0.0.0.0  in mongo.cfg and restart the service

``` conf
# mongod.conf

# Where and how to store data.
storage:
  dbPath: C:\ProgramData\MongoDB\data\db
  journal:
    enabled: true

# where to write logging data.
systemLog:
  destination: file
  logAppend: true
  path:  C:\ProgramData\MongoDB\log\mongod.log

# network interfaces
net:
  port: 27017
  bindIp: 0.0.0.0
```

6. Docker using WSL2

- Setting up Docker with WSL2
- Using Docker command in WSL2, Windows
- Enable Kubernetes on Docker with Minikube

7. Setting up VScode with WSL2

- pyenv version resolve (different versions of python on WSL and Windows .python_version file)
- Installing VSCode Server on WSL2

8. Setting up Git

- Use Windows Credential Manager with WSL2
- Setting up SSH Keys

```
git config --global user.name "Your Name"
git config --global user.email "youremail@domain.com"

git config --global credential.helper "/mnt/c/Program\ Files/Git/mingw64/libexec/git-core/git-credential-manager-core.exe"
```

File: .gitattributes
```
* text=auto eol=lf
*.{cmd,[cC][mM][dD]} text eol=crlf
*.{bat,[bB][aA][tT]} text eol=crlf
```