# Remediation: Least Privilege Fix

## Fix strategy

The root cause was an `iam:PassRole` permission with `Resource: "*"` and no condition. The fix preserves the legitimate need, `bincom-dev-user` must still be able to launch EC2 instances with a role attached, while removing the ability to choose an arbitrary, high privilege role. Two changes were made:

1. The `Resource` field was narrowed from `"*"` to the ARN of a single, harmless, purpose built role, `EC2-Limited-Role`, so no other role in the account can be passed by this user.
2. A `Condition` was added requiring `iam:PassedToService` to equal `ec2.amazonaws.com`, so the permission cannot be reused to pass a role to a different, unrelated AWS service (for example, Lambda).

## The fixed policy

The full file is at [`../policies/fixed-policy.json`](../policies/fixed-policy.json).

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EC2Access",
      "Effect": "Allow",
      "Action": [
        "ec2:RunInstances",
        "ec2:DescribeInstances",
        "ec2:DescribeInstanceStatus",
        "ec2:CreateTags",
        "ec2:TerminateInstances"
      ],
      "Resource": "*"
    },
    {
      "Sid": "RestrictedPassRole",
      "Effect": "Allow",
      "Action": "iam:PassRole",
      "Resource": "arn:aws:iam::<ACCOUNT_ID>:role/EC2-Limited-Role",
      "Condition": {
        "StringEquals": {
          "iam:PassedToService": "ec2.amazonaws.com"
        }
      }
    }
  ]
}
```

## Applying the fix

```powershell
aws iam create-role --role-name EC2-Limited-Role --assume-role-policy-document file://trust-policy.json --profile bincom-admin

aws iam attach-role-policy --role-name EC2-Limited-Role `
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMReadOnlyAccess --profile bincom-admin

aws iam create-policy-version --policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/BincomDevUserPolicy `
  --policy-document file://fixed-policy.json --set-as-default --profile bincom-admin
```

Result: `VersionId = v2`, set as the default version of `BincomDevUserPolicy`.

`EC2-Limited-Role` was attached to `AmazonSSMReadOnlyAccess`, a harmless, read only policy, chosen purely to give the role a distinct, non dangerous identity for contrast against `EC2-Admin-Role`.

Screenshot: `screenshots/remediation/fixed-policy-console.png`

## Verification: the exploit is blocked

The exact same escalation attempt, passing `EC2-Admin-Role`, was repeated after the fix was applied:

```powershell
aws ec2 run-instances --image-id ami-0bdbbea3e76315b75 --instance-type t3.small `
  --key-name bincom-privesc-key --security-group-ids sg-07dd6d0efee71dcaf `
  --subnet-id subnet-014b46080b850a527 `
  --iam-instance-profile Name=EC2-Admin-Instance-Profile `
  --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=privesc-retry-blocked}]" `
  --profile bincom-dev-user
```

Result:

```text
An error occurred (UnauthorizedOperation) when calling the RunInstances operation:
You are not authorized to perform this operation. User:
arn:aws:iam::<ACCOUNT_ID>:user/bincom-dev-user is not authorized to perform:
iam:PassRole on resource: arn:aws:iam::<ACCOUNT_ID>:role/EC2-Admin-Role
because no identity-based policy allows the iam:PassRole action.
```

The fixed policy correctly denies the exact action that made the original escalation possible, and names the specific permission that is missing.

Screenshot: `screenshots/remediation/access-denied-retry.png`

## Verification: legitimate access still works

To confirm the fix is least privilege rather than simply broken, `bincom-dev-user` was used to launch an instance passing the approved role, `EC2-Limited-Role`, instead of `EC2-Admin-Role`:

```powershell
aws ec2 run-instances --image-id ami-0bdbbea3e76315b75 --instance-type t3.small `
  --key-name bincom-privesc-key --security-group-ids sg-07dd6d0efee71dcaf `
  --subnet-id subnet-014b46080b850a527 `
  --iam-instance-profile Name=EC2-Limited-Instance-Profile `
  --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=privesc-legit-path}]" `
  --profile bincom-dev-user
```

This launch succeeded, confirming the user's legitimate ability to launch EC2 instances was preserved while the escalation path was closed.

Screenshot: 

!(legitimate path)[/screenshots/remediation/legitimate-path-ec2-limited-role.png]

Continue to [`04-lessons-learned.md`](04-lessons-learned.md) for takeaways from this exercise.
