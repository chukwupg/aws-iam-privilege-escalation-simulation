# Cleanup

Every resource created for this exercise was removed after documentation was complete, so that no billable resources or unused credentials were left active in the AWS account. Commands are grouped in the order they were actually run: compute first, then the key pair, then IAM, then networking, since IAM resources cannot be deleted while still referenced by a running instance profile. All account IDs and key material below are redacted as `<ACCOUNT_ID>` / `<REDACTED>`.

## 1. Terminate compute

```powershell
aws ec2 describe-instances --filters "Name=tag:Name,Values=privesc-*" --query "Reservations[].Instances[].[InstanceId,State.Name]" --profile bincom-admin
```

```json
[
    [ "i-04450e4e36f6480bd", "running" ],
    [ "i-02d2b0551e8653ed2", "running" ]
]
```

Two instances were still running: the original exploit instance (`i-04450e4e36f6480bd`) and the legitimate path instance launched with `EC2-Limited-Role` during remediation verification (`i-02d2b0551e8653ed2`). Both were terminated together:

```powershell
aws ec2 terminate-instances --instance-ids i-04450e4e36f6480bd i-02d2b0551e8653ed2 --profile bincom-admin
```

A follow up `describe-instances` confirmed both moved to `shutting-down` and then `terminated`.

## 2. Delete the key pair

```powershell
aws ec2 delete-key-pair --key-name bincom-privesc-key --profile bincom-admin
Remove-Item bincom-privesc-key.pem
```

No output on success. Verified later in Section 5 below.

## 3. Remove the vulnerable user

```powershell
aws iam detach-user-policy --user-name bincom-dev-user --policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/BincomDevUserPolicy --profile bincom-admin

aws iam delete-access-key --user-name bincom-dev-user --access-key-id AKIA36RW26SYBGO5E5AZ --profile bincom-admin

aws iam delete-user --user-name bincom-dev-user --profile bincom-admin
```

All three commands returned silently, which confirms success: had the policy still been attached, `delete-user` would have errored instead.

## 4. Remove IAM roles and instance profiles

```powershell
aws iam remove-role-from-instance-profile --instance-profile-name EC2-Admin-Instance-Profile --role-name EC2-Admin-Role --profile bincom-admin
aws iam delete-instance-profile --instance-profile-name EC2-Admin-Instance-Profile --profile bincom-admin
aws iam detach-role-policy --role-name EC2-Admin-Role --policy-arn arn:aws:iam::aws:policy/AdministratorAccess --profile bincom-admin
aws iam delete-role --role-name EC2-Admin-Role --profile bincom-admin

aws iam remove-role-from-instance-profile --instance-profile-name EC2-Limited-Instance-Profile --role-name EC2-Limited-Role --profile bincom-admin
aws iam delete-instance-profile --instance-profile-name EC2-Limited-Instance-Profile --profile bincom-admin
aws iam detach-role-policy --role-name EC2-Limited-Role --policy-arn arn:aws:iam::aws:policy/AmazonSSMReadOnlyAccess --profile bincom-admin
aws iam delete-role --role-name EC2-Limited-Role --profile bincom-admin
```

All eight commands returned silently, confirming success for both the escalation target role and the scoped comparison role created during remediation.

## 5. Remove the vulnerable policy

`BincomDevUserPolicy` had two versions (`v1`, the original vulnerable policy, and `v2`, the fixed policy set as default). Only the default version is deleted automatically with the policy, so the non default version was deleted explicitly first:

```powershell
aws iam list-policy-versions --policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/BincomDevUserPolicy --profile bincom-admin
aws iam delete-policy-version --policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/BincomDevUserPolicy --version-id v1 --profile bincom-admin
aws iam delete-policy --policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/BincomDevUserPolicy --profile bincom-admin
```

## 6. Remove networking

```powershell
aws ec2 disassociate-route-table --association-id rtbassoc-0bec94c1767a5c329 --profile bincom-admin
aws ec2 delete-route-table --route-table-id rtb-01e8c4232eda085f3 --profile bincom-admin
aws ec2 detach-internet-gateway --internet-gateway-id igw-0b2db7ece94976dd2 --vpc-id vpc-01d4ff00ae4c24ee2 --profile bincom-admin
aws ec2 delete-internet-gateway --internet-gateway-id igw-0b2db7ece94976dd2 --profile bincom-admin

aws ec2 delete-security-group --group-id sg-07dd6d0efee71dcaf --profile bincom-admin
```

```json
{
    "Return": true,
    "GroupId": "sg-07dd6d0efee71dcaf"
}
```

```powershell
aws ec2 delete-subnet --subnet-id subnet-014b46080b850a527 --profile bincom-admin
aws ec2 delete-vpc --vpc-id vpc-01d4ff00ae4c24ee2 --profile bincom-admin
```

## 7. Final verification pass

Every category of resource created for this lab was checked after teardown, to confirm nothing was left behind.

```powershell
aws ec2 describe-route-tables --filters "Name=tag:Name,Values=bincom-privesc-rt" --profile bincom-admin
```
```json
{ "RouteTables": [] }
```

```powershell
aws ec2 describe-internet-gateways --filters "Name=tag:Name,Values=bincom-privesc-igw" --profile bincom-admin
```
```json
{ "InternetGateways": [] }
```

```powershell
aws ec2 describe-vpcs --filters "Name=tag:Name,Values=bincom-privesc-vpc" --profile bincom-admin
```
```json
{ "Vpcs": [] }
```

```powershell
aws ec2 describe-key-pairs --key-names bincom-privesc-key --profile bincom-admin
```
```text
An error occurred (InvalidKeyPair.NotFound) when calling the DescribeKeyPairs operation:
The key pair 'bincom-privesc-key' does not exist
```

```powershell
aws iam get-policy --policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/BincomDevUserPolicy --profile bincom-admin
```
```text
An error occurred (NoSuchEntity) when calling the GetPolicy operation:
Policy arn:aws:iam::<ACCOUNT_ID>:policy/BincomDevUserPolicy was not found.
```

```powershell
aws iam list-users --profile bincom-admin
```
```json
{
    "Users": [
        {
            "UserName": "<REDACTED>",
            "Arn": "arn:aws:iam::<ACCOUNT_ID>:user/<REDACTED>",
            "CreateDate": "2026-09-23T15:46:50+00:00"
        }
    ]
}
```
Only the original administrator user remains. `bincom-dev-user` is confirmed gone.

```powershell
aws iam list-roles --query "Roles[?contains(RoleName, 'EC2-Admin') || contains(RoleName, 'EC2-Limited')]" --profile bincom-admin
```
```json
[]
```
No roles from this exercise remain in the account.

![Terminal output of the final verification pass: route table, Internet Gateway, VPC, key pair, policy, users, and roles all confirmed gone or empty](screenshots/cleanup/final-verification-pass.png)
*Final verification pass: account confirmed clean*

## 8. Local cleanup

```powershell
aws configure list-profiles
```

The `bincom-dev-user` and `stolen-admin-creds` profile entries were removed from the local `~/.aws/credentials` and `~/.aws/config` files once the exercise was complete, since the stolen session credentials had expired and were no longer relevant. The local `bincom-privesc-key.pem` file was deleted in Step 2.

## Result

Every resource created for this exercise, compute, IAM users, roles, instance profiles, the vulnerable policy, and all networking components, was confirmed removed through a dedicated verification pass rather than assumed from command success alone. No billable or security relevant resources from this lab remain in the account.
