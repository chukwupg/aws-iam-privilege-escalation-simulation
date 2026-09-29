# Lessons Learned

## On `iam:PassRole`

`iam:PassRole` is easy to under scope because it looks like a minor, administrative permission rather than a dangerous one. Combined with the ability to create any resource that accepts a role (EC2 instances, Lambda functions, ECS tasks, and others), it becomes a direct path to whatever privilege the passed role holds. The permission itself never appears in the list of actions that "did the damage," which is part of what makes it easy to overlook during a policy review.

## On scoping permissions

A permission should be scoped by asking not only what a user needs to do, but what the most damaging thing is that this permission would allow if combined with everything else the user, or anything they can reach, can already do. `iam:PassRole` on its own does nothing. Paired with `ec2:RunInstances` and a high privilege role sitting in the account, it is a complete escalation path.

## On IMDSv2

IMDSv2 (`HttpTokens: required`) is a meaningful hardening step against remote credential theft, particularly against server side request forgery (SSRF) style attacks where an external attacker tricks an application into querying the metadata endpoint on its behalf. It is not, however, a substitute for scoping `iam:PassRole` correctly, since it does nothing to stop an attacker who already has legitimate shell access to the instance itself, which is exactly the scenario demonstrated in this lab.

## On the fix

Restricting `iam:PassRole` to specific role ARNs, and adding an `iam:PassedToService` condition, closes this class of escalation while preserving the legitimate workflow the permission exists to support. This is a general pattern worth carrying forward: wildcard resources on IAM management actions (`PassRole`, `CreatePolicyVersion`, `AttachUserPolicy`, `CreateAccessKey`, and similar) deserve particular scrutiny, since their danger usually comes from what they let a user reach, not from what they do directly.

## On testing a fix

Testing a fix is not complete until both the exploit path is confirmed blocked and the legitimate, intended use case is confirmed still working. A policy that blocks everything is not a least privilege policy, it is a broken one. Verifying both directions, in Section 5.4 and 5.5 of the fix, is what makes this a credible remediation rather than an overcorrection.

## Summary

This exercise showed a complete, realistic privilege escalation chain from a single over broad permission to full administrator access, and a fix that was verified from both sides: the attack is blocked, and the legitimate workflow still works. That combination, not just closing the hole but proving the door still works for its intended purpose, is the actual goal of least privilege IAM design.
