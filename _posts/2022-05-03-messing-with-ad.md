---
layout: default
title: Active Directory
---

### Here's my active directory setup:

Windows server 2012 R2 Datacenter Evaluation
Windows 10 - A (John Doe's machine)
Windows 10 - B (Foo Bar's machine)

My main motive is to understand how it all works BTS so that it is easier for me to attack it. I'll be exploring various methods of enumerating AD and making LDAP queries, learning how "crackmapexec" works in the in the process

## Get-DomainPolicy

It's supposed to show us the policies defined for this AD setup. For example, password policy like minimum and maximum age of password, what the password complexity should be stuff like that.

![Get-DomainPolicy](/assets/images/active-directory/Get-DomainPolicy-result.png)

## Get-DomainController

This one is to find the domain controllers, in my AD setup, I only have one domain controller.

![Get-DomainController](/assets/images/active-directory/Get-DomainController-result.png)
This image verifies that we're getting the right results

## Get-DomainUser

Next is enumerating the users, this command gives us a lot of details.
"John Doe" and "Foo Bar" are the names of the users I had configured on this AD setup, this lists all the important details about them and their computers, groups everything

![Get-DomainUser](/assets/images/active-directory/Get-DomainUser-result.png)

This does not look good, the description is clearly giving away the password for the SQL service account
![SQL service accountl](/assets/images/active-directory/SQL-service-compromised.png)
