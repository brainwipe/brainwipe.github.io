---

title: "Setting IAM Policy that forces MFA use on the command line"
date: 2022-04-06
categories: aws iam mfa powershell cli commandline
---

Securing your AWS IAM account with MFA is a really good idea. However, if you're not careful, the AWS IAM policy might stop you from using the AWS Powershell toolkit. This post is an [aide mémoire](https://en.wikipedia.org/wiki/Aide-m%C3%A9moire).

In IAM, you set policies that control access to resources. To force MFA, you might add a policy that looks like this:

```
# Don't blindly copy this, there's a problem here
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "BlockAnyAccessOtherThanAboveUnlessSignedInWithMFA",
            "Effect": "Deny",
            "NotAction": "iam:*",
            "Resource": "*",
            "Condition": {
                "BoolIfExists": {
                    "aws:MultiFactorAuthPresent": "false"
                }
            }
        }
    ]
}
```

On the surface, this seems sensible. You **Deny** access to all `Resource` that isn't `iam` if multi-factor authentication is not present. It's a mouthful but the logic is sound.

> You need access to IAM to login at all, so you can't deny that

https://aws.amazon.com/premiumsupport/knowledge-center/mfa-iam-user-aws-cli/