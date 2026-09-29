# Cloud Privilege Escalation Simulation (AWS IAM)

This repository documents a hands on AWS IAM security lab: creating a deliberately misconfigured IAM user, using that misconfiguration to escalate from a low privilege identity to full administrator access, and then closing the gap with a verified least privilege fix.

The technique demonstrated is privilege escalation via `iam:PassRole` combined with `ec2:RunInstances`, one of the most common and well documented IAM privilege escalation paths in AWS. 

All work was performed in an isolated, purpose built VPC inside a personal AWS sandbox account, and every resource created was torn down at the end of the exercise.

> This project was built for a training environment only. Every technique shown here was executed against resources I own, inside an isolated account and network, with no other systems or data involved. Do not run privilege escalation techniques against any AWS account you do not own or have explicit, written authorization to test.

## What this demonstrates

1. **A misconfigured IAM policy**, granting `iam:PassRole` with no resource restriction, alongside `ec2:RunInstances`.
2. **A working exploitation chain**: launch an EC2 instance as the low privilege user, pass it a high privilege role, SSH in, and pull temporary administrator credentials from the instance metadata service.
3. **A verified fix**: a scoped `iam:PassRole` policy (specific role ARN plus an `iam:PassedToService` condition) that blocks the exploit while leaving the user's legitimate workflow intact.

## Repository structure

```text
bincom-assignment5-iam-privesc/
├── README.md                        This file
├── docs/
│   ├── 01-vulnerability-design.md   The misconfiguration and why it is exploitable
│   ├── 02-exploitation-steps.md     Full step by step privilege escalation walkthrough
│   ├── 03-remediation.md            The least privilege fix and its verification
│   └── 04-lessons-learned.md        What this exercise teaches about IAM hygiene
├── screenshots/
│   ├── iam-setup/                   Network, role, and vulnerable user screenshots
│   ├── privesc-attempt/             Exploitation screenshots
│   └── remediation/                 Fix and verification screenshots
├── cli-logs/
│   ├── setup-commands.log           VPC, IAM role, and vulnerable user creation
│   ├── privesc-commands.log         The exploitation sequence
│   └── remediation-commands.log     Fix application and re-test
├── policies/
│   ├── vulnerable-policy.json       The original, misconfigured policy
│   └── fixed-policy.json            The corrected, least privilege policy
└── cleanup.md                       Full teardown of every resource created
```

## Technique summary

| Item | Detail |
|---|---|
| Vulnerability | Unrestricted `iam:PassRole` (`Resource: "*"`) combined with `ec2:RunInstances` |
| Escalation path | Launch EC2 instance with a high privilege role attached, SSH in, query IMDS for the role's temporary credentials |
| Impact | Low privilege user (`bincom-dev-user`) obtained full `AdministratorAccess` in the account |
| Fix | Scope `iam:PassRole` to a specific, approved role ARN, with an `iam:PassedToService` condition |
| Result | Exploit attempt returns `UnauthorizedOperation`, legitimate EC2 launches with the approved role still succeed |

See [`docs/01-vulnerability-design.md`](docs/01-vulnerability-design.md) for the full writeup.

## Environment

- **Cloud provider:** AWS
- **Region:** us-east-1
- **Network:** dedicated, isolated VPC created specifically for this exercise
- **Identities used:** an administrator profile (environment builder and reviewer), a deliberately under scoped developer user (the attacker persona), and the temporary credentials obtained through the exploit
- **Access method:** SSH, over a security group restricted to a single IP address

All account IDs, access keys, secret keys, and session tokens in this repository are redacted. Where a value is shown, it has been replaced with a placeholder such as `<ACCOUNT_ID>` or `<REDACTED>`.

## Reproducing this lab

The full command sequence, with expected output, is documented in `cli-logs/` and referenced throughout `docs/`. Reproducing it requires:

1. An AWS account you own or are authorized to test in, ideally a sandbox account.
2. The AWS CLI v2, configured with an administrator profile.
3. A willingness to tear everything down afterward. See `cleanup.md`.

## Author

### **Bincom Academy**

**Carried Out By:** CHUKWU PRAISEGOD
