---
title: "Add Domain to & Display Trusted Senders List"
date: Sat, 03 Oct 2020 15:54:09 +0000
slug: 2020/9/25/an-easy-way-to-add-senders-to-a-individuals-trusted-senders-list-kb3cl-d722n
status: publish
tags:
  - powershell
  - microsoft-365
  - exchange-online
---

There are two separate versions of this script. I have created one for individuals up to a small list of users, and a second version that I use for entire domain names of people. The first script is for individuals or small groups. It also produces an output that shows the end result of the script for each of the users affected.

```
##################################################
## Add Domain to & Display Trusted Senders List ##
## Created by Robert Martin                     ##
## Version 1.0 -- 25 Sep 2020                   ##
## robert.p.martin(at)gmail(dot)com             ##
## https://www.robertpmartin.com                ##
##################################################

#### USE THIS FOR INDIVIDUAL USERS ####

$Recipients = New-Object System.Collections.ArrayList
#Add your recipients to this section. Separate them by 'name@name.com','name2@name.com'
$Recipients = 'name@domain.com'

$Senders = New-Object System.Collections.ArrayList
#Add your senders to this section. Separate them by 'name@name.com','name2@name.com'
$Senders = 'senderdomain.com'

#Set the Trusted Sender and Domains list for each of the recipients and add the Sender/s from the group to the existing list
$i = 0
Foreach ($Recipient in $Recipients) {
    Set-MailboxJunkEmailConfiguration -Identity $Recipient -TrustedSendersAndDomains @{Add=$Senders}
    $i++
    Write-Progress -activity "Adding Sender" -status "Set: $i of $($Recipients.Count)" -percentComplete (($i / $Recipients.Count) * 100)
}

#Create an on screen print out of all the addresses in each of the Recipients Trusted Senders and Domains list.  Separated by User Email address ***** then **** and a blank line.
Write-Host "Trusted Senders List/s" -ForegroundColor DarkYellow
$Recipients | Foreach-Object {
    Write-Host $_ -ForegroundColor DarkRed
    Write-Host ""
    Write-Host "*****" -ForegroundColor DarkYellow
    Get-MailboxJunkEmailConfiguration -Identity $_ | Select-Object -ExpandProperty TrustedSendersAndDomains | Sort-Object
    Write-Host "*****" -ForegroundColor DarkYellow
    Write-Host ""
}
```

Here is the script I use to modify the Trusted Senders list while adding a domain for and entire domain of users:

```
#########################################################################
## Add Domain to & Display Entire Domain's Users Trusted Senders Lists ##
## Created by Robert Martin                                            ##
## Version 1.0 -- 25 Sep 2020                                          ##
## robert.p.martin(at)gmail(dot)com                                    ##
## https://www.robertpmartin.com                                       ##
#########################################################################

#### USE FOR GROUPS OF PEOPLE ####

#Define Variables
$Recipients = New-Object System.Collections.ArrayList
$Recipients = Get-EXORecipient -ResultSize Unlimited -Filter {(EmailAddresses -like '*@domain.com') -and (RecipientTypeDetails -eq 'UserMailbox')} | Where-Object PrimarySMTPAddress -like '*@domain.com' | Sort-Object DisplayName

$Senders = New-Object System.Collections.ArrayList
$Senders = 'senderdomain.com'

$Number = [math]::ceiling($Recipients.count / 10)
$Count = 0
$Inc = 0
$b = 0
#End Variable Definitions

Write-Host "Total number for mailboxes:" $Recipients.count
#Count up from 0 to ($Recipients.Count / 100) by 100 users a time to get all employees#
For ($count=0; $count -le $Number; $Count++){

    $Users = New-Object System.Collections.ArrayList
    $i = 0

    $Users = $Recipients | Select-Object -Skip ($Inc+0) -First 10

    ForEach ($User in $Users) {
        Set-MailboxJunkEmailConfiguration -Identity $User.Identity -TrustedSendersAndDomains @{Add=$Senders}
        $i++
        Write-Progress -activity "Researching Devices" -status "Group: $b of $Number User: $i of $($Users.Count)" -percentComplete (($i / $Users.Count) * 100)
    }
    $b++
    Clear-Variable -Name 'Users'
    $Inc = $Inc+10
}
```
