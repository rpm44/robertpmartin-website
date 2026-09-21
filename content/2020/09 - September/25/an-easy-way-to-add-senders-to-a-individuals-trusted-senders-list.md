---
title: "Add Trusted Senders & Display Trusted Senders List"
date: Fri, 25 Sep 2020 16:42:43 +0000
slug: 2020/9/25/an-easy-way-to-add-senders-to-a-individuals-trusted-senders-list
status: publish
tags:
  - powershell
  - microsoft-365
  - exchange-online
---

I created this script to allow our service desk an easier time to help individuals who could not or would not manage their own quarantine in Office365. The script does the following things; It allows the user of the script to input multiple recipient and sender addresses, takes those addresses, releases any email that matches the sender and recipient address that were entered. While it releases those messages it also tags those messages as False Positive for Microsoft and releases to all recipients in the email.

Finally, the script creates a small text block that explains to the end user what email were released and the addresses that were added to each of the recipient’s Trusted Senders List.

```
########################################################
## Add Trusted Senders & Display Trusted Senders List ##
## Created by Robert Martin                           ##
## Version 1.0 -- 25 Sep 2020                         ##
## robert.p.martin(at)gmail(dot)com                   ##
## https://www.robertpmartin.com                      ##
########################################################

#Create an array for the recipient group
$Recipients = New-Object System.Collections.ArrayList
#Add your recipients to this section. Separate them by 'name@name.com','name2@name.com'
$Recipients = 'ap@simco-ion.com'

#Create an array for the sender group
$Senders = New-Object System.Collections.ArrayList
#Add your senders to this section. Separate them by 'name@name.com','name2@name.com'
$Senders = 'verkoop@picardi.nl'

#Get all quarantined messages that are send inbound to the Recipient group from the Sender group, release those messages, report as false positive, Allow the Sender, and release to all recipients

Get-QuarantineMessage -RecipientAddress $Recipients -SenderAddress $Senders -Direction inbound | Release-QuarantineMessage -ReportFalsePositive -ReleaseToAll

#Set the Trusted Sender and Domains list for each of the recipients and add the Sender/s from the group to the existing list
$recipients | Foreach-Object {
    Set-MailboxJunkEmailConfiguration -Identity $_ -TrustedSendersAndDomains @{Add=$Senders}
}

#Create an on screen print out of the blurb to put in the ticket.
Write-Host "Hello ,"
Write-Host "I have released all email from $senders to $recipients and added $senders to $recipients Trusted Senders List/s."
Write-Host ""
Write-Host "Thank you for your time,"
Write-Host "- "
Write-Host ""

#Create an on screen print out of all the addresses in each of the Recipients Trusted Senders and Domains list.  Separated by User Email address ***** then **** and a blank line.
Write-Host "Trusted Senders List/s" -ForegroundColor DarkYellow
$recipients | Foreach-Object {
    Write-Host $_ -ForegroundColor DarkBlue
    Write-Host ""
    Write-Host "*****" -ForegroundColor DarkYellow
    Get-MailboxJunkEmailConfiguration -Identity $_ | Select-Object -ExpandProperty TrustedSendersAndDomains
    Write-Host "*****" -ForegroundColor DarkYellow
    Write-Host ""
}

#Create an on screen print out of all the Quarantined Messages
Write-Host ""
Write-Host "Items Released from Quarantine" -ForegroundColor Green
Get-QuarantineMessage -RecipientAddress $Recipients -SenderAddress $Senders -Direction inbound | Select-Object RecipientAddress,Subject,SenderAddress,Released
```

Below, this is what the output of the script looks like:

```
Hello 'Name',
I have added name@otherdomain.com to ap@domain.com Blocked Senders List/s.

Thank you for your time,
-'Name Here'

Blocked Senders List/s
ap@domain.com

*****
craigpp@bphbuild.com
EMEAAcc21@newbewerley.org.uk
cp-gail.sherwood.mt@z.o.o.m.o.cmvce.com
bob_heston@legalaccessplans.com
gkbws-limitspaceAWSremindinvite@kgsi.b.osi.otpvrfy036prfcbtwndots.com
*****
```
