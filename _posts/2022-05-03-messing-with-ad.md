---
layout: default
title: Active Directory
---

### Here's my active directory setup:

Windows server 2012 R2 Datacenter Evaluation
Windows 10 - A (John Doe's machine)
Windows 10 - B (Foo Bar's machine)

My main motive is to understand how it all works BTS so that it is easier for me to attack it. I'll be exploring various methods of enumerating AD and making LDAP queries, learning how "crackmapexec" works in the in the process

So, I finally decided to learn a bit about LDAP queries and make a CLI tool to query active directory. One way to go about doing that is using C# with .NET framework, it was quite timetaking to set it up but after that's done. I realised I don't really know how to make C# apps from scratch.

It was pretty clear what I had to do next, not learning C# but start writing random lines of code hoping Intellisense will do everything for me.
After googling stuff for time, I figured out how to make a query against AD.

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
            Console.WriteLine(prop);
        }
    }
}
catch (Exception e)
{
    Console.WriteLine("[!] Something went wrong");
}
```

The results obtained are of type "SearchResult" which just contain a set property and their respective values, pretty much like JSON. Just do result[key] and you'll be given a value.

