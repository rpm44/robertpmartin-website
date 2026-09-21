---
title: "User ACP Info"
date: Sat, 03 Oct 2020 16:06:54 +0000
slug: 2020/9/25/an-easy-way-to-add-senders-to-a-individuals-trusted-senders-list-kb3cl-55d2k
status: publish
tags:
  - powershell
  - microsoft-365
---

This script collects the ACP (Audio Conference Provider) information for your users in your tenant so you can find out what provider they are using. This requires you to already be logged into Skype for Business within PowerShell. The command I use for this is:

```
Import-Module SkypeOnlineConnector
```

```
$sfbSession = New-CsOnlineSession -Credential $cred
```

```
Import-PSSession $sfbSession
```

If you have a different domain name in your login than the domain you’re trying to log in to, you have to modify the second line to the following:

```
$sfbSession = New-CsOnlineSession -Credential $cred -OverrideAdminDomain "domainyou'relogginginto.onmicrosoft.com"
```

```
######################################
## User ACP Info                    ##
## Created by Robert Martin         ##
## Version 1.0 -- 25 Sep 2020       ##
## robert.p.martin(at)gmail(dot)com ##
## https://www.robertpmartin.com    ##
######################################

# Dial In Users
$InputData = Get-CsOnlineUser -Filter {(AcpInfo -ne $null) -and (WindowsEmailAddress -like '*@domain.com')} -ResultSize Unlimited -ErrorAction SilentlyContinue -WarningAction SilentlyContinue | Select-Object DisplayName, UserPrincipalName, WindowsEmailAddress, ACPInfo
$Rollup = New-Object System.Collections.ArrayList
$i = 0

ForEach ($User in $InputData){
    [xml]$PersonAcpInfo = $User.AcpInfo
    $item = [PSCustomObject]@{
        'Display Name' = $User.DisplayName
        'UPN' = $User.UserPrincipalName
        'Primary Email' = $User.WindowsEmailAddress
        'ACP Provider' = $PersonAcpInfo.AcpInformation.name
    }
    $Rollup.Add($item) > $null
    $i++
    Write-Progress -activity "Researching User" -status "Set: $i of $($InputData.Count)" -percentComplete (($i / $InputData.Count) * 100)
}
$Rollup | Sort-Object 'Display Name'
```
