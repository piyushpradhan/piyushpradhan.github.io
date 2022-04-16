---
layout: default
title: Playing around on flaws.cloud
---

In the past week, I've been spending my time playing around with cloud services, deploying random stuff just to try and figure out how it's being used.
I also wanted to have a vulnerable environment to try and extract something that isn't usually made available.
It was then when I stumbled across this `flaws.cloud`.

It is basically a cloud CTF, deployed on AWS. Explains about some common misconfigurations and how to take advantage of those mistakes.

# Let's start with Level 1

### Finding the subdomain that contains s3 bucket for `flaws.cloud`

- AWS S3 buckets follow this pattern

```
bucket-name.s3.region.amazonaws.com
bucket-name.s3.amazonaws.com
```

In this case, a little bit of google dorking will be enough

```
flaws.cloud site:s3.amazonaws.com
```

**S3 Buckets should configured so that they are not publicly available. If the bucket policy is modified to allow everybody "s3:GetObject" privilege, then anyone can access the bucket.**

The first link had the contents of the s3 bucket in XML format
I thought I'll be able to find something interesting in robots.txt file...
![](/assets/images/flaws-cloud/xml-in-bucket.png)

but...
![](/assets/images/flaws-cloud/robots-contents.png)

Looking further I found something that actually was a hint
![](/assets/images/flaws-cloud/actual-hint.png)
**secret-dd02c7c.html** was the page I was looking for.
That page gave me the link to the **Second Level**

# Levelling up (Level 2)

For this challenge, use of `aws-cli` was required.

Setting it up is fairly easy, take the **Secret Key** and **Access Key** from your own AWS account and use those to configure awscli profile

```
aws --profile configure <profile-name>
```

After that just enter the keys and an output format.

Now, that we have an AWS account and the bucket url we can try to list the contents of the bucket.

```
aws s3 --profile <profile-name> ls s3://<bucket-url>
```
