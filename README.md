# Tailscale AWS hostname monitor

https://tailscale.com

## Overview

This repository contains an AWS Lambda function intended to:
- receive notifications of changes in AWS tags across a
  number of AWS services
- check for an AWS tag named "ts-hostname"
- automatically populate the `Hosts` section of a Tailscale ACL
  Policy file to update the IP address of that hostname

This is intended to be used in conjunction with a Tailscale
[subnet router](https://tailscale.com/kb/1019/subnets/) offering
a Tailscale route to a private AWS VPC. ACLs can be defined for
the private 172.16.x.y VPC IP addresses, relying on this Lambda
function to update the IP address if the AWS instance is replaced.

## Setup

### Step 1: Lambda function
Create a Lambda function in the AWS Console in the AWS region where
the EC2/RDS/etc resources used with Tailscale are located.


### Step 2: Permissions
In Lambda > Configuration > Permissions, the ARN of the role created for the
Lambda function will be shown. It needs to be given several permissions.

a. It needs to be able to read EC2 Instance Details. Attach the
`AmazonEC2ReadOnlyAccess` policy to the role.
 
b. It needs to be able to get RDS instance details and tags. Create
an inline policy of:
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowRDSTagsRead",
            "Effect": "Allow",
            "Action": [
                "rds:DescribeDBInstances",
		"rds:ListTagsForResource"
            ],
            "Resource": "*"
        }
    ]
}
```

### Step 3: Configuration
Populate an Environment variables:
- `TAILSCALE_TAILNET`: the name of the tailnet, such as `example.com` or `octocat.github`
- `TAILSCALE_API_KEY`: a key for use with the [Tailscale API](https://tailscale.com/kb/1101/api/)

TODO: `TAILSCALE_API_KEY` should be in the AWS Secrets Manager, not an environment variable.

### Step 4: EventBridge
Create an EventBridge event with pattern:
```
{
  "source": ["aws.tag"],
  "detail-type": ["Tag Change on Resource"],
  "detail": {
    "service": ["ec2", "rds"]
  }
}
```

This will run the Lambda function whenever tags are changed on an EC2 or RDS instance.

### Step 5: Add `ts-hostname` AWS Tags
For any AWS resource for which you wish this Lambda function to keep the IP
address up to date, two things need to be done:
1. Add an initial entry for the hostname in the [Tailscale ACLs Hosts section](https://login.tailscale.com/admin/acls).
   This Lambda function will update a `Hosts` entry which is already present, but not
   add a new one.
1. Add an AWS Tag with key `ts-hostname` and value of the hostname in the Tailscale ACL
   file which it is to maintain.

## Bugs

Please file any issues about this function on
[the issue tracker](https://github.com/tailscale/tailscale/issues).

## Contributing

PRs welcome! But please file bugs. Commit messages should [reference
bugs](https://docs.github.com/en/github/writing-on-github/autolinked-references-and-urls).

We require [Developer Certificate of
Origin](https://en.wikipedia.org/wiki/Developer_Certificate_of_Origin)
`Signed-off-by` lines in commits.

## About Us

[Tailscale](https://tailscale.com/) is primarily developed by the
people at https://github.com/orgs/tailscale/people. For other contributors,
see:

* https://github.com/tailscale/tailscale/graphs/contributors
* https://github.com/tailscale/tailscale-android/graphs/contributors

## Legal

WireGuard is a registered trademark of Jason A. Donenfeld.
