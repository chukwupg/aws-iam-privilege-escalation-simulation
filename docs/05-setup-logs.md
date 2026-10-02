# Setup CLI Logs

This is a readable, markdown formatted version of the environment setup commands and their output. For the raw, unformatted terminal capture, see [`../cli-logs/setup-commands.log`](../cli-logs/setup-commands.log). All account IDs and key material are redacted as `<ACCOUNT_ID>` / `<REDACTED>`. Region: `us-east-1`.

## CLI configuration

```powershell
aws configure --profile bincom-admin
```
```text
AWS Access Key ID [None]: <REDACTED>
AWS Secret Access Key [None]: <REDACTED>
Default region name [None]: us-east-1
Default output format [None]: json
```

`aws sts get-caller-identity --profile bincom-admin` did not return output on first attempt in this terminal session. As a workaround, the public IP address needed for the security group rule below was retrieved via the EC2 console security group inbound rule editor, using its "My IP" auto detect feature, which is functionally equivalent to `curl ifconfig.me`.

## Network layer

```powershell
aws ec2 create-vpc --cidr-block 10.0.0.0/16 --tag-specifications "ResourceType=vpc,Tags=[{Key=Name,Value=bincom-privesc-vpc}]" --profile bincom-admin
```
```json
{
    "Vpc": {
        "OwnerId": "<ACCOUNT_ID>",
        "InstanceTenancy": "default",
        "CidrBlockAssociationSet": [
            {
                "AssociationId": "vpc-cidr-assoc-03e65aa96f9af26bd",
                "CidrBlock": "10.0.0.0/16",
                "CidrBlockState": { "State": "associated" }
            }
        ],
        "IsDefault": false,
        "Tags": [ { "Key": "Name", "Value": "bincom-privesc-vpc" } ],
        "VpcId": "vpc-01d4ff00ae4c24ee2",
        "State": "pending",
        "CidrBlock": "10.0.0.0/16",
        "DhcpOptionsId": "dopt-06bdc2bea37c72c0a"
    }
}
```

```powershell
aws ec2 create-subnet --vpc-id vpc-01d4ff00ae4c24ee2 --cidr-block 10.0.1.0/24 --availability-zone us-east-1a --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=bincom-privesc-subnet}]" --profile bincom-admin
```
Result: `SubnetId = subnet-014b46080b850a527`

```powershell
aws ec2 create-internet-gateway --tag-specifications "ResourceType=internet-gateway,Tags=[{Key=Name,Value=bincom-privesc-igw}]" --profile bincom-admin
aws ec2 attach-internet-gateway --vpc-id vpc-01d4ff00ae4c24ee2 --internet-gateway-id <IGW_ID> --profile bincom-admin
```

```powershell
aws ec2 create-route-table --vpc-id vpc-01d4ff00ae4c24ee2 --tag-specifications "ResourceType=route-table,Tags=[{Key=Name,Value=bincom-privesc-rt}]" --profile bincom-admin
aws ec2 create-route --route-table-id <ROUTE_TABLE_ID> --destination-cidr-block 0.0.0.0/0 --gateway-id <IGW_ID> --profile bincom-admin
aws ec2 associate-route-table --route-table-id <ROUTE_TABLE_ID> --subnet-id subnet-014b46080b850a527 --profile bincom-admin
aws ec2 modify-subnet-attribute --subnet-id subnet-014b46080b850a527 --map-public-ip-on-launch --profile bincom-admin
```

```powershell
aws ec2 create-security-group --group-name bincom-privesc-sg --description "SSH access for privesc lab" --vpc-id vpc-01d4ff00ae4c24ee2 --profile bincom-admin
```
Result: `GroupId = sg-07dd6d0efee71dcaf`

```powershell
aws ec2 authorize-security-group-ingress --group-id sg-07dd6d0efee71dcaf --protocol tcp --port 22 --cidr <YOUR_IP>/32 --profile bincom-admin
```

Screenshots for this section: [`02-exploitation-steps.md`, Section 1.1](02-exploitation-steps.md#11-network)

## IAM role and instance profile (escalation target)

```powershell
notepad trust-policy.json
```
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "ec2.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

```powershell
aws iam create-role --role-name EC2-Admin-Role --assume-role-policy-document file://trust-policy.json --profile bincom-admin
```
```json
{
    "Role": {
        "Path": "/",
        "RoleName": "EC2-Admin-Role",
        "RoleId": "AROA36RW26SYCHM6PPL4J",
        "Arn": "arn:aws:iam::<ACCOUNT_ID>:role/EC2-Admin-Role",
        "CreateDate": "2026-09-23T18:06:34+00:00"
    }
}
```

```powershell
aws iam attach-role-policy --role-name EC2-Admin-Role --policy-arn arn:aws:iam::aws:policy/AdministratorAccess --profile bincom-admin
```
No output on success.

```powershell
aws iam create-instance-profile --instance-profile-name EC2-Admin-Instance-Profile --profile bincom-admin
```
```json
{
    "InstanceProfile": {
        "Path": "/",
        "InstanceProfileName": "EC2-Admin-Instance-Profile",
        "InstanceProfileId": "AIPA36RW26SYGYODO432C",
        "Arn": "arn:aws:iam::<ACCOUNT_ID>:instance-profile/EC2-Admin-Instance-Profile",
        "CreateDate": "2026-09-23T18:14:16+00:00",
        "Roles": []
    }
}
```

```powershell
aws iam add-role-to-instance-profile --instance-profile-name EC2-Admin-Instance-Profile --role-name EC2-Admin-Role --profile bincom-admin
```
No output on success.

Screenshots for this section: [`02-exploitation-steps.md`, Section 1.2](02-exploitation-steps.md#12-high-privilege-role-the-escalation-target)

## Vulnerable IAM user

```powershell
notepad vulnerable-policy.json
```
See [`../policies/vulnerable-policy.json`](../policies/vulnerable-policy.json).

```powershell
aws iam create-user --user-name bincom-dev-user --profile bincom-admin
```
```json
{
    "User": {
        "Path": "/",
        "UserName": "bincom-dev-user",
        "UserId": "AIDA36RW26SYFDEXAFIT6",
        "Arn": "arn:aws:iam::<ACCOUNT_ID>:user/bincom-dev-user",
        "CreateDate": "2026-09-24T06:06:22+00:00"
    }
}
```

```powershell
aws iam create-policy --policy-name BincomDevUserPolicy --policy-document file://vulnerable-policy.json --profile bincom-admin
```
```json
{
    "Policy": {
        "PolicyName": "BincomDevUserPolicy",
        "PolicyId": "ANPA36RW26SYO5F2ZYMN6",
        "Arn": "arn:aws:iam::<ACCOUNT_ID>:policy/BincomDevUserPolicy",
        "Path": "/",
        "DefaultVersionId": "v1",
        "AttachmentCount": 0,
        "IsAttachable": true,
        "CreateDate": "2026-09-24T06:10:02+00:00"
    }
}
```

```powershell
aws iam attach-user-policy --user-name bincom-dev-user --policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/BincomDevUserPolicy --profile bincom-admin
```
No output on success.

```powershell
aws iam create-access-key --user-name bincom-dev-user --profile bincom-admin
```
```json
{
    "AccessKey": {
        "UserName": "bincom-dev-user",
        "AccessKeyId": "<REDACTED>",
        "Status": "Active",
        "SecretAccessKey": "<REDACTED>",
        "CreateDate": "2026-09-24T06:16:37+00:00"
    }
}
```

```powershell
aws configure --profile bincom-dev-user
aws sts get-caller-identity --profile bincom-dev-user
```
```json
{
    "UserId": "<REDACTED>",
    "Account": "<ACCOUNT_ID>",
    "Arn": "arn:aws:iam::<ACCOUNT_ID>:user/bincom-dev-user"
}
```

Screenshots for this section: [`02-exploitation-steps.md`, Section 1.3](02-exploitation-steps.md#13-the-vulnerable-user)

---

Continue to [`06-exploitation-logs.md`](06-exploitation-logs.md) for the exploitation sequence.
