---
layout: default
title: Learning from HTB Challenges
---

Learning from HTB Challenges
============================

Racecar
=======

I had tried this one before but couldn’t solve it, after learning about binary exploitation and writing a few blogs on it I decided to give it another shot.

First things first, I fired up Ghidra and looked at the decompiled code.

![](https://miro.medium.com/max/1222/1*DaLdIECHNumMbvkzais3_g.png)Decompiled code of car\_menu() function from Ghidra

There it is, a “Format String Vulnerability” in that printf statement. What is a format string vulnerability you ask ? Simply put, it is a printf statement without format specifiers, which allows attackers like us to exploit that by supplying format specifiers like “%p” to leak pointer addresses from the program, by simply providing them in the input.

Now, that I have a way to obtain some information I just needed to figure out what I wanted to extract. Easy right ? The contents of flag.txt file, but for some reason it took me half an hour to figure that out.

![](https://miro.medium.com/max/1120/0*aVl2-fEoGOAPhljX.gif)

Anyway, I noticed that the contents of flag.txt file are being stored in “\_stream” variable and that value is printed only when we win the race.

![](https://miro.medium.com/max/922/1*FDFf5nJ1SsQ6dE1VlGIDQA.png)

We’re asked to give a victory speech after winning the race, that’s the entry point! I can send tons of “%p”, convert the hex into ASCII characters and challenge completed, easy… at least that’s what I thought, but it took a lot more time than what I had expected.

In the screenshot, there’s that “fgets” statement which clearly states that the value of “\_stream”, which actually contains the value of flag.txt file, is stored in “local\_3c”. Ghidra gave me the address of “local\_3c” variable, I took it over to gdb to examine the state of the stack right after I provide the malicious input.

![](https://miro.medium.com/max/1400/1*Wqlb00IiYxVFHGoi70UI7w.png)

The address of the “local\_3c” variable was 0x00001002, so I set a breakpoint on that address and ran the program. But the program crashed, why ? I still can’t figure it out, if any of you have any idea what happened here, please let me know. After an hour of trying several methods (none of them were finding an alternative to the way breakpoints can be set in gdb) I decided to look at a writeup and guess what I learned that there are other ways to set breakpoints too.

```
ged-peda$ break \* (car\_menu+881)
```

Oh, by the way that “car\_menu+881” came from that particular instruction at 0x00001002.

```
gdb-peda$ x/i 0x00001002
```

Now, I just input a bunch of “%p” in the victory speech part, but nothing happened, then I realized I had created an empty flag.txt. So, I put “AAAA” inside that file and examined the stack again…

![](https://miro.medium.com/max/1400/1*Y5-jMhmJ8o1G7qHj0MMPNA.png)

Just to make sure that this wasn’t some coincidence, I changed the contents of flag.txt to “BBBB AAAA” and checked again.

![](https://miro.medium.com/max/1400/1*jE54n6ihz2qeKA6NIpyaaw.png)

Yeah, the contents of the flag.txt can actually be read from the stack.

Did the same thing with the server too, provided a bunch of “%p” in the victory speech part and it returned hex!

![](https://miro.medium.com/max/1400/1*_mIytyQ0ngxJdmd3XpsZeg.png)

Put all of that, after removing those (nil) values, into CyberChef to decode it.

![](https://miro.medium.com/max/1400/1*6E-fxO9hiMOlrgUSPABbhQ.png)

And look at that, there’s something that looks like a flag. Remember the format of HTB flags, “HTB{some\_text}”. To convert this string into that pattern, I wrote a simple but horrible C code.

![](https://miro.medium.com/max/780/1*tNgvn7qlYJ-SNC1h8VaLbw.png)![](https://miro.medium.com/max/1382/1*66F_3Ut9t-1AUB_AuX_Cog.png)

It’s not very good, but it gets the job done :)