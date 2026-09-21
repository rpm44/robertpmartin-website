---
title: "Update & Display User/s Blocklist/s"
date: Sun, 27 Sep 2020 00:15:16 +0000
slug: 2020/9/25/an-easy-way-to-add-senders-to-a-individuals-trusted-senders-list-kb3cl-knkey
status: draft
draft: true
---

This is another script I’ve created in two separate ways. One for individuals or small groups, and another method that uses a For loop to allow me to do entire divisions of employees without PowerShell timing out.

```
#########################################
## Update & Display User/s Blocklist/s ##
## Created by Robert Martin            ##
## Version 1.0 -- 25 Sep 2020          ##
## robert.p.martin(at)gmail(dot)com    ##
## https://www.robertpmartin.com       ##
#########################################

#Create an array for the recipient group
$Recipients = New-Object System.Collections.ArrayList
#Add your recipients to this section. Separate them by 'name@name.com','name2@name.com'
$Recipients = 'gail.sherwood@rocol.com'

#Create an array for the sender group
$Senders = New-Object System.Collections.ArrayList
#Add your senders to this section. Separate them by 'name@name.com','name2@name.com'
$Senders = 'EMEAAcc21@newbewerley.org.uk','cp-gail.sherwood.mt@z.o.o.m.o.cmvce.com','bob_heston@legalaccessplans.com','gkbws-limitspaceAWSremindinvite@kgsi.b.osi.otpvrfy036prfcbtwndots.com'

#Set the Trusted Sender and Domains list for each of the recipients and add the Sender/s from the group to the existing list
$recipients | Foreach-Object {
    Set-MailboxJunkEmailConfiguration -Identity $_ -BlockedSendersAndDomains @{Add=$Senders}
}

#Create an on screen print out of the blurb to put in the ticket.
Write-Host "Hello ,"
Write-Host "I have added $senders to $recipients Blocked Senders List/s."
Write-Host ""
Write-Host "Thank you for your time,"
Write-Host "- "
Write-Host ""

#Create an on screen print out of all the addresses in each of the Recipients Trusted Senders and Domains list.  Separated by User Email address ***** then **** and a blank line.
Write-Host "Blocked Senders List/s" -ForegroundColor Yellow
$recipients | Foreach-Object {
    Write-Host $_
    Write-Host ""
    Write-Host "*****" -ForegroundColor Yellow
    Get-MailboxJunkEmailConfiguration -Identity $_ | Select-Object -ExpandProperty BlockedSendersAndDomains
    Write-Host "*****" -ForegroundColor Yellow
    Write-Host ""
}

#######################################################
## Update & Display User/s Blocklist/s (Large Group) ##
## Created by Robert Martin                          ##
## Version 1.0 -- 25 Sep 2020                        ##
## robert.p.martin(at)gmail(dot)com                  ##
## https://www.robertpmartin.com                     ##
#######################################################

#### USE FOR GROUPS OF PEOPLE ####

#Define Variables
$Recipients = New-Object System.Collections.ArrayList
$Recipients = Get-EXORecipient -ResultSize Unlimited -Filter {(EmailAddresses -like '*@itwcap.com') -and (RecipientTypeDetails -eq 'UserMailbox')} | Where-Object PrimarySMTPAddress -like '*@itwcap.com' | Sort-Object DisplayName

$Senders = New-Object System.Collections.ArrayList
$Senders = 'acctg@galaxyballoon.com'

$Number = [math]::ceiling($Recipients.count / 10)
$Count = 0
$Inc = 0
$b = 0
#End Variable Definitions

#Count up from 0 to ($Recipients.Count / 100) by 100 users a time to get all employees#
For ($count=0; $count -le $Number; $Count++){

    $Users = New-Object System.Collections.ArrayList
    $i = 0

    $Users = $Recipients | Select-Object -Skip ($Inc+0) -First 10

    ForEach ($User in $Users) {
        Set-MailboxJunkEmailConfiguration -Identity $User.Identity -BlockedSendersAndDomains @{Add=$Senders}
        $i++
        Write-Progress -activity "Adding Blocked Sender to List" -status "Group: $b of $Number User: $i of $($Users.Count)" -percentComplete (($i / $Users.Count) * 100)
    }
    $b++
    Clear-Variable -Name 'Users'
    $Inc = $Inc+10
}
```
