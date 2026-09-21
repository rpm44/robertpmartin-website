---
title: "Script to Query Multiple Recipient Types in Microsoft 365"
date: Wed, 23 Sep 2020 00:01:20 +0000
slug: 2020/9/22/script-to-query-multiple-recipient-types-in-microsoft-365
status: publish
---

Below is a script that I wrote to help me find all types of mail enabled items in Microsoft 365. It also provides a list of Users who have access to that mail asset at what rights they currently have.

```
################################################
## Recipient Type Info & Access Rights Report ##
## Created by Robert Martin                   ##
## Version 1.0 -- 22 Sep 2020                 ##
## robert.p.martin@gmail.com                  ##
## https://www.robertpmartin.com              ##
################################################

#####################################################
## PLEASE CHANGE $Domain & $CSVFile FOR YOUR NEEDS ##
#####################################################

################################################################
## $Domain IS THE DOMAIN THAT YOU ARE RUNNING THIS REPORT FOR ##
################################################################
$Domain = '*@domain.com'

#####################################################################
## $CSVfile IS THE LOCATION THAT YOU WANT TO SAVE YOUR CSV FILE TO ##
#####################################################################
$CSVFile = 'C:\Users\rmartin\OneDrive\Scripts\Mailbox Info.csv'

################################
## DON'T CHANGE THIS VARIABLE ##
################################
$MailboxType = New-Object System.Collections.ArrayList
#####################################################################################################################################################################################
## YOU CAN CHANGE OR COMMENT OUT THE MAILBOX TYPES IN $MailboxTypes THAT YOU'RE NOT INTERESTED IN.                                                                                 ##
## MOST COMMON ARE: UserMailbox, SharedMailbox, RoomMailbox, EquipmentMailbox, MailUniversalSecurityGroup                                                                          ##
## MORE INFORMATION HERE: https://answers.microsoft.com/en-us/msoffice/forum/msoffice_o365admin-mso_exchon-mso_o365b/recipient-type-values/7c2620e5-9870-48ba-b5c2-7772c739c651    ##
#####################################################################################################################################################################################
$MailboxTypes = 'EquipmentMailbox','RoomMailbox','SharedMailbox','UserMailbox','MailUniversalDistributionGroup'

################################################
## YOU DON'T HAVE TO TOUCH ANYTHING DOWN HERE ##
################################################

#Create Array list variable for the final rollup of information 
$Rollup = New-Object System.Collections.ArrayList

#Run a foreach loop to go through each mailbox recipient type that is in the list above in $MailboxTypes
foreach ($MailboxType in $MailboxTypes) {
    #If the mailbox type does not equal 'MailUniversialDistributionGroup' Run this block of code; if is is a distribution group, skip to where 'else' is a few lines down
    if ($MailboxType -ne 'MailUniversalDistributionGroup') {
        #Create the counter for the Write-Progress function at the bottom of this if statement
        $i = 0
        #Create a variable named Mailboxes that is the result of looking up all recipients that have the email address & primarySMTPAddress for the Domain we are interested in and have the correct RecipientTypeDetails Mailbox Type.
        $Mailboxes = Get-EXORecipient -ResultSize Unlimited -Filter "EmailAddresses -like '$Domain' -and RecipientTypeDetails -eq '$MailboxType'" | Where-Object PrimarySMTPAddress -like ($Domain) | Sort-Object PrimarySMTPAddress

        #Foreach recipient found in the result above get all mailbox information and the mailbox permission information for all users.
        foreach ($Mailbox in $Mailboxes){
            $Mailboxinfo = Get-ExoMailbox $Mailbox.PrimarySMTPAddress
            $MailboxUsers = Get-EXOMailboxPermission $Mailbox.PrimarySMTPAddress | Where-Object user -like '@'
            #For each mailbox user found in the Get-EXOMailboxPermission Command, get all info for the specific user.
            foreach ($MailboxUser in $MailboxUsers){
                $Name = Get-EXORecipient -Identity $MailboxUser.User
                #Add all the Mailbox and User information together into an array.
                $Item = [PSCustomObject]@{
                    'Mailbox Name' = $Mailboxinfo.DisplayName
                    'Mailbox Email' = $Mailboxinfo.PrimarySMTPAddress
                    'Mailbox Type' = $Mailboxinfo.RecipientTypeDetails
                    'User Name' = $Name.DisplayName
                    'User Email Address' = $Name.PrimarySMTPAddress
                    'Access Rights' = ($MailboxUser.AccessRights | Out-String).Trim()
                }
            #Add the user information from the foreach loop above to the final Array.
            $Rollup.Add($Item) > $Null
        }
        #Increment the Mailbox Counter, Display the activity on a blue and yellow bar on screen and move on to the next Mailbox
        $i++
        Write-Progress -activity "Researching $MailboxType Info" -status "Set: $i of $($Mailboxes.Count)" -percentComplete (($i / $Mailboxes.Count)  100)
        }
    }
    else {
        $i = 0
        $DistroGroups = Get-EXORecipient -ResultSize Unlimited -Filter "EmailAddresses -like '$Domain' -and RecipientTypeDetails -eq '$MailboxType'" | Where-Object PrimarySMTPAddress -like ($Domain) | Sort-Object PrimarySMTPAddress

        foreach ($DistroGroup in $DistroGroups){
            $Distroinfo = Get-DistributionGroup $DistroGroup.PrimarySMTPAddress
            $DistroUsers = Get-DistributionGroupMember $DistroGroup.PrimarySMTPAddress

            foreach ($DistroUser in $DistroUsers){
                $Name = Get-EXORecipient -Identity $DistroUser.Name

                $Item = [PSCustomObject]@{
                    'Mailbox Name' = $Distroinfo.DisplayName
                    'Mailbox Email' = $Distroinfo.PrimarySMTPAddress
                    'Mailbox Type' = $Distroinfo.RecipientTypeDetails
                    'User Name' = $Name.DisplayName
                    'User Email Address' = $Name.PrimarySMTPAddress
                    'Access Rights' = ""
                }
            $Rollup.Add($Item) > $Null
        }
        $i++
        Write-Progress -activity "Researching Distro Group Info" -status "Set: $i of $($DistroGroups.Count)" -percentComplete (($i / $DistroGroups.Count)  100)
        }
    }
}
#Take the final array and export it to a CSV file in the location that was specified at the beinging of the script.
$Rollup | Export-Csv $CSVFile -NoTypeInformation
```
