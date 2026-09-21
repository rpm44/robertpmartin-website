---
title: "Removing a Email Address from Unsafe Senders List in Exchange 2010 Using Powershell"
date: Wed, 02 Mar 2016 05:40:02 +0000
slug: 2016/3/1/removing-a-email-address-from-unsafe-senders-list-in-exchange-2010-using-powershell
status: publish
tags:
  - powershell
  - exchange-server
---

Today I had to remove an errant email address from a user's unsafe sender list.  I found out that using Powershell and Exchange Shell for Exchange 2010 you can add and remove email addresses from the already current list without removing everything else.

First, I used powershell to see what the current list of users that were blocked was.  The command I used was:

```
(get-mailboxjunkemailconfiguration "user's alias with no quotes").blockedsendersanddomains
```

This allowed me to see the entire list of senders that have been blocked.  I was able to double check and see the the email address that was having issued was indeed in that list.  To remove that specific user from the list I used the command below.

```
Set-MailboxJunkEmailConfiguration username -BlockedSendersAndDomains @{remove="name@domain.com"
```
