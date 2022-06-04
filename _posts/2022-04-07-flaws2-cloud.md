---
layout: default
title: Completing the next part
---

# FLAWS 2 challenge

## The attacker path

![](/assets/images/flaws2-cloud/level1-challenge.png)
To be honest, I wasn't sure how to approach this one, I checked my notes from flaws.cloud challenge (the first version)
I tried looking at an incorrect submission response. I'm not sure if this is a hint but I think this is an s3 bucket URL, it follows the format.
![](/assets/images/flaws2-cloud/level1-clue-1.png)
But I'm not allowed to access it using browser or...
![](/assets/images/flaws2-cloud/level1-failed-1.png)
via AWS CLI.
Completely out of ideas I decided to check the hint, and then it hit me. I didn't even try messing with the input!
It asked me not to bruteforce, but what if I try something else. In the source code, it's parsing the input into a float value and probably checking if the input is a finite number.
I tried putting in characters, numbers greater than the 100 digit limit didn't find anything useful. Remember the s3 bucket url, it was passing input through url params, I tried changing that and it worked.
Actually it didn't it showed me some errors but those errors exposed some sensitive information, like AWS Access ID of someone's account, who probably has access to that bucket.
![](/assets/images/flaws2-cloud/level1-almost-solved.png)
Well, that didn't go well, I tried adding this profile to my AWS CLI but it says this access key ID does not exist. That's because I hadn't added the session token, if MFA is enabled then I have to include the session token in the credentials as well.
![](/assets/images/flaws2-cloud/level1-identity.png)
However, I can't find a bucket with this url and I do not have permission to list buckets in this account.
Maybe that's not the bucket url...
![](/assets/images/flaws2-cloud/level1-solved.png)
Now that I look back at it, it was pretty straight forward.

### What went wrong ?

AWS Lamba obtains IAM role credentials as environment variables. These are not supposed to be accessible but someimes developers tend to dump these during errors to help with debugging. That's why when I used alphabets instead of digits the website behaved pretty well and showed me a alert dialog but the url it was sending request to showed the error message and those environment variables were exposed.

# The next target - Level 2

Target : http://container.target.flaws2.cloud/
For this one we already have a hint here, "ECR is named level2".

AWS ECR (Elastic Container Registry) allows users to manage and analyse various images.

In this case, I have the name of the repository, I'll just check if this actually exists.

![](/assets/images/flaws2-cloud/level2-repository-found.png)
![](/assets/images/flaw2-cloud/level2-repository-described.png)

I'm not sure if this going to be helpful, but I found docker login credentials.

![](/assets/images/flaws2-cloud/level2-login-clue.png)
I can use these creds to look into the docker image.
![](/assets/images/flaws2-cloud/level2-docker-image.png)

I'm pretty sure there is something useful in there. Or better yet, I'll just download the entire image.
I used the credentials I found earlier to download the entire image.

![](/assets/images/flaws2-cloud/level2-docker-pull.png)

Upon inspection the level2 image, I find several layers. Maybe I can extract some information from these layers using AWS CLI.
