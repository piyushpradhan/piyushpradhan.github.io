---
layout: default
title: Enumerating active directory the weird way
---

# Enumerating active directory the hard way

### Here's my active directory setup:

Windows server 2012 R2 Datacenter Evaluation
Windows 10 - A (John Doe's machine)
Windows 10 - B (Foo Bar's machine)

My main motive is to understand how it all works BTS so that it is easier for me to attack it.

So, I decided to learn a bit about LDAP queries and make a CLI tool to query active directory. One way to go about doing that is using C# with .NET framework, it was quite time-taking to set it up but after that's done I realised I don't really know how to make C# apps from scratch.

It was pretty clear what I had to do next, not learning C# but start writing random lines of code hoping Intellisense will do everything for me.
After googling stuff for some time, I figured out how to make a query against AD.

```
DirectoryEntry dirEntry = new DirectoryEntry("LDAP://10.0.2.7", "JDoe", "FirstTarget1");
DirectorySearcher dirSearcher = new DirectorySearcher(dirEntry);
```

There's a lot going on in these two lines, I realised a little later when I actually tried to understand what this is doing. This "DirectoryEntry" class is an encapsulation of objects in ADDS which can be provided to query, update or manage the Active Directory entries. We are allowed to a do lot more than just querying entries, with certain privileges of course.
In this case, as I am running this code on a domain user and not on a domain controller I have supplied the remote path of the Domain Controller's LDAP server, along with my (domain user's) credentials. So, now it knows where to look for entries.
The "DirectorySearcher" actually uses the "DirectoryEntry" class to perform search operation based on the parameters supplied using LDAP.

Oh about the parameters that I was talking about, among them is one called "Filter", it's pretty obvious what it is. We can use LDAP filter syntax to construct a filter string and set it to DirectorySearcher object and it will make LDAP query with that filter.

```
DirectoryEntry dirEntry = new DirectoryEntry("LDAP://10.0.2.7", "JDoe", "FirstTarget1");
DirectorySearcher dirSearcher = new DirectorySearcher(dirEntry);
dirSearcher.Filter = "(&(objectCategory=user)(objectClass=user))";
try
{
    foreach (SearchResult result in dirSearcher.FindAll())
    {
        DirectoryEntry resDe = result.GetDirectoryEntry();

        ResultPropertyCollection resPropCollection = result.Properties;
        foreach (string prop in resPropCollection.PropertyNames)
        {
            foreach(Object value in resPropCollection)
            {
                Console.WriteLine("{0} - {1}", prop, value);
            }
        }
    }
}
catch (Exception e)
{
    Console.WriteLine("[!] Something went wrong");
}
```

The results obtained are of type "SearchResult" which just contain a set property and their respective values, pretty much like JSON. Just do result[key] and you'll be given a value. By the way, there has to be a better way to print results there's no way using nested loops is the only way of printing results, is it ?

Anyway, next I had to understand LDAP filters, which was pretty easy once you understand the snytax. After that you just have to match the attribute names and that's it.

Here's the results of one SearchResult, there are probably the attributes available which we can use in LDAP filters

```
name : John Doe
lastlogoff : 0
lastlogon : 132962424065881972
instancetype : 4
whenchanged : 5/3/2022 12:50:07 AM
lastlogontimestamp : 132960126072754036
pwdlastset : 132959275004082512
primarygroupid : 513
distinguishedname : CN=John Doe,CN=Users,DC=VULN,DC=local
accountexpires : 9223372036854775807
cn : John Doe
badpwdcount : 0
adspath : LDAP://10.0.2.7/CN=John Doe,CN=Users,DC=VULN,DC=local
dscorepropagationdata : 1/1/1601 12:00:00 AM
displayname : John Doe
Ewhencreated : 5/2/2022 1:11:40 AM
codepage : 0
logoncount : 41
useraccountcontrol : 66048
samaccounttype : 805306368
objectsid : System.Byte[]
objectguid : System.Byte[]
objectclass : top
objectclass : person
objectclass : organizationalPerson
objectclass : user
Eusncreated : 20558
sn : Doe
usnchanged : 36924
countrycode : 0
samaccountname : JDoe
givenname : John
badpasswordtime : 132961243653455721
userprincipalname : JDoe@VULN.local
objectcategory : CN=Person,CN=Schema,CN=Configuration,DC=VULN,DC=local
```

Based on this result, I can make some functions with pre-defined LDAP filters.
Well, that was easy. Now looking at my code I feel like it can be optimized a lot. But anyway, it works for now. I've also added some functions that will provide me with information about the computers on the domain.

Something like this.
![computer-info](/assets/images/active-directory/active-directory-computers.png)

The script itself is pretty simple and straightforward. I'll probably add a few more things into it as I learn more about AD.
