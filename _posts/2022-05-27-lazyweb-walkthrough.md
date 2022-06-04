---
layout; default
title: LazyWeb Walkthrough
---

# LazyWeb Vulnerabilities

First we have the authentication, couldn't find anything in the first attempt so I decided to register and move on to check for other vulnerabilities.

There's an option in dashboard to "Add Site Form" it simply lets me add some url and then ping it. I'm not sure if this can be exploited but seems pretty interesting, so I'm going to try something.
It turns out it was actually vulnerable to command injection.

![](/assets/images/lazyweb/ping-exploit.png)

Apparently, the same form is vulnerable to XSS too

![](/assets/images/lazyweb/xss-add-site.png)

Now there is this profile picture upload feature on the update profile page.

There's this robots.txt lying around too, which I should have noticed sooner. It hinting towards another secret or s3cret directory.
I can't access this directory normally, I'm just getting redirected to the home page again
