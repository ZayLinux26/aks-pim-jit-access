# Break-Glass Account Runbook

![Microsoft Entra ID](https://img.shields.io/badge/Microsoft_Entra_ID-0078D4?style=for-the-badge&logo=microsoftentraid&logoColor=white)
![AKS](https://img.shields.io/badge/Azure_Kubernetes_Service-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)
![Break Glass](https://img.shields.io/badge/Emergency_Access-C8102E?style=for-the-badge&logo=microsoft&logoColor=white)

## Purpose

The just-in-time model in this project removes standing privileged access from the cluster. That is the goal, but it introduces a failure mode: if PIM is unavailable, an approver is locked out, or MFA breaks, nobody can activate `cluster-admin` and the cluster becomes unreachable for administration at the worst possible time.

The break-glass account is the deliberate exception that covers that gap. It holds standing Cluster Admin, sits outside PIM, and is monitored so that any use is noticed and investigated.

## The account

| Field | Value |
|-------|-------|
| Display name | Break-Glass Account (Emergency Access) |
| UPN | `breakglass@isaiahherard26gmail.onmicrosoft.com` |
| Type | Cloud-only (`.onmicrosoft.com`, no federation) |
| Object ID | `c2d51c39-ed1f-4d61-bde4-393f0923038e` |
| Cluster access | Azure Kubernetes Service RBAC Cluster Admin on `aks-pim-lab`, standing |
| PIM | Not enrolled, by design |

Cloud-only is a requirement, not a convenience. A federated or synced account depends on infrastructure (the identity provider, the sync service) that could be the very thing that is down. A native cloud account has the fewest dependencies, so it is the most likely to still work in an outage.

## Why it is excluded

- **Excluded from PIM.** The account carries Cluster Admin as a permanent assignment, not an eligible one. If PIM is the thing that failed, an eligible-only account would be useless. This account does not depend on PIM to function.
- **Excluded from blocking Conditional Access.** Any CA policy that enforces MFA or restricts sign-in must exclude this account. If the MFA provider is down and the account is subject to MFA, it is locked out along with everyone else, which defeats the purpose. (In this lab tenant, exclusions are documented here; in production they are applied to every blocking CA policy.)

The tradeoff is understood: this account is powerful and lightly gated, so its credential handling and monitoring carry the weight that PIM and CA carry for everyone else.

## How the credential is handled

- The password is long and random, stored in a secrets vault, and checked out only under a declared emergency.
- It is not kept in a password manager used for daily work, not written down, and not shared in plaintext.
- After any use, the password is rotated and a new value is vaulted.

## How its use is monitored

Legitimate use of this account should be close to zero. Any sign-in is therefore worth an alert and a review.

- Entra sign-in logs stream to the `law-aks-pim` Log Analytics workspace.
- A scheduled alert fires on any sign-in by this account:

```kql
SigninLogs
| where TimeGenerated > ago(1h)
| where UserPrincipalName == "breakglass@isaiahherard26gmail.onmicrosoft.com"
| project TimeGenerated, UserPrincipalName, AppDisplayName, IPAddress, ResultType
| order by TimeGenerated desc
```

- Every alert is treated as an incident until proven to be an authorized emergency: who signed in, when, why, and whether the credential was rotated afterward.

## Emergency use procedure

1. Declare the emergency and record the reason and the ticket or incident number.
2. Check out the credential from the vault.
3. Sign in and perform only the work the emergency requires.
4. Sign out.
5. Rotate the password and re-vault it.
6. Review the sign-in alert that fired, and close the incident with notes.

## Review cadence

- Credential rotated on a fixed schedule even when unused, and after every use.
- The account's standing Cluster Admin assignment recertified during the quarterly access review alongside the eligible assignments.
- Conditional Access exclusions reviewed whenever CA policies change.
