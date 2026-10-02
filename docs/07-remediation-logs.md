# Remediation CLI Logs

This is a readable, markdown formatted version of the remediation commands and their output. For the raw, unformatted terminal capture, see [`../cli-logs/remediation-commands.log`](../cli-logs/remediation-commands.log). All account IDs are redacted as `<ACCOUNT_ID>`.

## Create a legitimate, low privilege role for contrast

```powershell
aws iam create-role --role-name EC2-Limited-Role --assume-role-policy-document file://trust-policy.json --profile bincom-admin
```
```json
{
    "Role": {
        "Path": "/",
        "RoleName": "EC2-Limited-Role",
        "RoleId": "AROA36RW26SYHZC46WFPH",
        "Arn": "arn:aws:iam::<ACCOUNT_ID>:role/EC2-Limited-Role",
        "CreateDate": "2026-09-24T08:55:56+00:00"
    }
}
```

```powershell
aws iam attach-role-policy --role-name EC2-Limited-Role --policy-arn arn:aws:iam::aws:policy/AmazonSSMReadOnlyAccess --profile bincom-admin
```
No output on success.

## Save and apply the fixed policy

```powershell
notepad fixed-policy.json
```
See [`../policies/fixed-policy.json`](../policies/fixed-policy.json).

```powershell
aws iam create-policy-version --policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/BincomDevUserPolicy --policy-document file://fixed-policy.json --set-as-default --profile bincom-admin
```
```json
{
    "PolicyVersion": {
        "VersionId": "v2",
        "IsDefaultVersion": true,
        "CreateDate": "2026-09-24T09:06:26+00:00"
    }
}
```

Screenshot for this section: [`03-remediation.md`, "The fixed policy"](03-remediation.md#the-fixed-policy)

## Re-attempt the exploit: expect denial

```powershell
aws ec2 run-instances --image-id ami-0bdbbea3e76315b75 --instance-type t3.small --key-name bincom-privesc-key --security-group-ids sg-07dd6d0efee71dcaf --subnet-id subnet-014b46080b850a527 --iam-instance-profile Name=EC2-Admin-Instance-Profile --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=privesc-retry-blocked}]" --profile bincom-dev-user
```
```text
aws: [ERROR]: An error occurred (UnauthorizedOperation) when calling the
RunInstances operation: You are not authorized to perform this operation.
User: arn:aws:iam::<ACCOUNT_ID>:user/bincom-dev-user is not authorized to
perform: iam:PassRole on resource: arn:aws:iam::<ACCOUNT_ID>:role/EC2-Admin-Role
because no identity-based policy allows the iam:PassRole action.
Encoded authorization failure message: <REDACTED, long encoded string>
```

Screenshot for this section: [`03-remediation.md`, "Verification: the exploit is blocked"](03-remediation.md#verification-the-exploit-is-blocked)

## Verify the legitimate path still works

```powershell
aws iam create-instance-profile --instance-profile-name EC2-Limited-Instance-Profile --profile bincom-admin
aws iam add-role-to-instance-profile --instance-profile-name EC2-Limited-Instance-Profile --role-name EC2-Limited-Role --profile bincom-admin
```

```powershell
aws ec2 run-instances --image-id ami-0bdbbea3e76315b75 --instance-type t3.small --key-name bincom-privesc-key --security-group-ids sg-07dd6d0efee71dcaf --subnet-id subnet-014b46080b850a527 --iam-instance-profile Name=EC2-Limited-Instance-Profile --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=privesc-legit-path}]" --profile bincom-dev-user
```
Result: launch succeeded, instance `i-02d2b0551e8653ed2` running with `EC2-Limited-Role` attached.

Screenshot for this section: [`03-remediation.md`, "Verification: legitimate access still works"](03-remediation.md#verification-legitimate-access-still-works)

## Result

The escalation path via `EC2-Admin-Role` is blocked with a clear, specific `AccessDenied` / `UnauthorizedOperation` error. The legitimate workflow, launching an instance with the approved `EC2-Limited-Role`, continues to succeed. The fix is confirmed least privilege, not merely restrictive.

---

For the full teardown of every resource created in this lab, see [`../cleanup.md`](../cleanup.md).
