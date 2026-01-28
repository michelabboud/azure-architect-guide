# Entra ID Governance Deep Dive

## What is Identity Governance?

Identity Governance answers the critical questions:
- **Who** has access to **what**?
- **Should** they still have that access?
- **How** did they get it?
- **When** should it be removed?

### AWS Comparison

```
AWS:                                    Azure:
────                                    ─────

Manual processes + 3rd party tools      Native Entra ID Governance

• IAM Access Analyzer (detective)       • Access Reviews (proactive)
• Manual access reviews                 • Entitlement Management
• No native lifecycle management        • Lifecycle Workflows
• CloudTrail for audit                  • PIM for privileged access
                                        • Access packages
```

---

## The Three Pillars

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        ENTRA ID GOVERNANCE                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐  │
│  │  ACCESS REVIEWS     │  │    ENTITLEMENT      │  │    LIFECYCLE        │  │
│  │                     │  │    MANAGEMENT       │  │    WORKFLOWS        │  │
│  │  "Should they still │  │                     │  │                     │  │
│  │   have access?"     │  │  "How do they get   │  │  "What happens when │  │
│  │                     │  │   access?"          │  │   they join/move/   │  │
│  │  • Periodic reviews │  │                     │  │   leave?"           │  │
│  │  • Manager approval │  │  • Access packages  │  │                     │  │
│  │  • Auto-remediation │  │  • Request/approval │  │  • Auto-provision   │  │
│  │  • Audit trail      │  │  • Time-limited     │  │  • Auto-update      │  │
│  │                     │  │  • Self-service     │  │  • Auto-deprovision │  │
│  └─────────────────────┘  └─────────────────────┘  └─────────────────────┘  │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │              PRIVILEGED IDENTITY MANAGEMENT (PIM)                    │    │
│  │              "Just-in-time privileged access"                       │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Access Reviews

### Why Access Reviews?

```
The Access Creep Problem:
─────────────────────────

Day 1: John joins Marketing team
       → Gets Marketing SharePoint access

Month 3: John helps on Project X
         → Gets Project X access

Month 6: John covers for Sarah
         → Gets Finance reporting access

Month 12: John moves to Sales
          → Gets Sales access
          → Still has Marketing, Project X, Finance access!  ← PROBLEM

Without Reviews: John accumulates permissions indefinitely
With Reviews: Quarterly review removes stale access
```

### Access Review Types

| Type | Scope | Reviewers | Use Case |
|------|-------|-----------|----------|
| Group membership | Who's in the group? | Group owners, managers | Security groups, M365 groups |
| Application access | Who can access the app? | App owners, managers | Enterprise apps |
| Role assignments | Who has which roles? | Role admins | Azure RBAC, Entra roles |
| Access package assignments | Who has which packages? | Package owners | Entitlement management |

### Case Study: Quarterly Group Membership Review

**Scenario**: Review all security groups quarterly to remove stale memberships.

**Configuration**:
```
Review name: Q1 2024 Security Group Review

SCOPE
├── What to review: Teams + Groups
├── Review scope: All groups with guest users
└── Specify groups: Security groups only

REVIEWERS
├── Primary: Group owners
├── Fallback: Manager of user
└── Self-review: Not allowed

SETTINGS
├── Duration: 14 days
├── Recurrence: Quarterly
├── Start date: January 1, 2024
├── Auto-apply results: Yes
├── If reviewer doesn't respond:
│   └── Action: Remove access
├── At end of review:
│   └── Send notification to: Group owners
└── Enable reviewer decision helpers: Yes
    └── Show recommendations based on last sign-in
```

**Reviewer Experience**:
```
┌─────────────────────────────────────────────────────────────────────────────┐
│ Access Review: Marketing-Team Security Group                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│ User                    Last Sign-in    Recommendation    Your Decision     │
│ ─────────────────────────────────────────────────────────────────────────── │
│ john.doe@contoso.com    2 days ago      Approve           [Approve]         │
│ jane.smith@contoso.com  45 days ago     Review needed     [Approve] [Deny]  │
│ guest@partner.com       180 days ago    Deny              [Deny]            │
│ sarah.jones@contoso.com Never           Deny              [Deny]            │
│                                                                              │
│ Bulk actions: [Accept all recommendations]                                   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Why This Design**:
> - **Why auto-apply?** Ensures action is taken. Manual follow-up often gets delayed.
> - **Why remove on no response?** Secure by default. Unknown = revoke.
> - **Why show last sign-in?** Data-driven decisions. Never signed in = likely doesn't need access.

---

## Entitlement Management

### The Problem with Traditional Access Requests

```
Traditional (Manual):
─────────────────────

User: "I need access to Project X"
        │
        ▼
IT Helpdesk ticket
        │
        ▼
Wait 2 days...
        │
        ▼
IT: "What exactly do you need?"
        │
        ▼
Back and forth emails...
        │
        ▼
Finally get 5 different permissions
        │
        ▼
Project ends... permissions remain forever
```

```
Entitlement Management:
───────────────────────

User: Browses self-service catalog
        │
        ▼
Finds "Project X Access Package"
(includes all 5 needed permissions)
        │
        ▼
Requests access (provides justification)
        │
        ▼
Manager approves in My Access portal
        │
        ▼
Automatic provisioning (immediate)
        │
        ▼
90-day time limit, auto-removed
```

### Access Package Structure

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         ACCESS PACKAGE                                       │
│                    "Project Alpha Full Access"                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  RESOURCE ROLES INCLUDED:                                                   │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │ ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────────────┐ │   │
│  │ │ Entra ID Group  │ │ SharePoint Site │ │ Azure Resource Group    │ │   │
│  │ │ "Project-Alpha" │ │ "Alpha Docs"    │ │ "rg-project-alpha"      │ │   │
│  │ │                 │ │                 │ │                         │ │   │
│  │ │ Role: Member    │ │ Role: Contribute│ │ Role: Contributor       │ │   │
│  │ └─────────────────┘ └─────────────────┘ └─────────────────────────┘ │   │
│  │ ┌─────────────────┐ ┌─────────────────┐                             │   │
│  │ │ Teams Channel   │ │ Enterprise App  │                             │   │
│  │ │ "Alpha Team"    │ │ "Alpha Dashboard"                             │   │
│  │ │                 │ │                 │                             │   │
│  │ │ Role: Member    │ │ Role: User      │                             │   │
│  │ └─────────────────┘ └─────────────────┘                             │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  POLICIES:                                                                  │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │ Policy 1: Internal Employees                                         │   │
│  │ ├── Who can request: All members (excluding guests)                  │   │
│  │ ├── Approval: Required                                               │   │
│  │ │   └── Stage 1: User's manager                                      │   │
│  │ │   └── Stage 2: Project Alpha Owner                                 │   │
│  │ ├── Requestor justification: Required                                │   │
│  │ ├── Assignment duration: 90 days                                     │   │
│  │ ├── Allow extension: Yes (with re-approval)                          │   │
│  │ └── Access review: Quarterly by Project Owner                        │   │
│  ├──────────────────────────────────────────────────────────────────────┤   │
│  │ Policy 2: External Partners (partner.com)                            │   │
│  │ ├── Who can request: Users from partner.com                          │   │
│  │ ├── Approval: Required                                               │   │
│  │ │   └── Stage 1: Internal sponsor                                    │   │
│  │ │   └── Stage 2: Security team                                       │   │
│  │ ├── Assignment duration: 30 days                                     │   │
│  │ └── Access review: Monthly by Internal sponsor                       │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Creating an Access Package (Azure CLI / Graph API)

```bash
# Note: Full implementation typically done via Portal or Graph API

# Graph API Example (simplified)
POST https://graph.microsoft.com/v1.0/identityGovernance/entitlementManagement/accessPackages

{
  "displayName": "Project Alpha Access",
  "description": "Full access to Project Alpha resources",
  "catalog": {
    "id": "{catalog-id}"
  },
  "isHidden": false
}

# Add resource role to package
POST https://graph.microsoft.com/v1.0/identityGovernance/entitlementManagement/accessPackages/{package-id}/resourceRoleScopes

{
  "role": {
    "originId": "Member",
    "originSystem": "AadGroup"
  },
  "scope": {
    "originId": "{group-id}",
    "originSystem": "AadGroup"
  }
}
```

---

## Privileged Identity Management (PIM)

### The Problem with Standing Privileges

```
Traditional Admin Access:
─────────────────────────

Sarah is Global Administrator
  │
  ├── 24/7/365 full admin rights
  ├── Even when not doing admin tasks
  ├── Even when on vacation
  ├── Even from personal device at home
  │
  └── If compromised → Complete tenant takeover

Risk: Permanent standing privilege = permanent attack surface
```

```
Just-in-Time with PIM:
──────────────────────

Sarah is ELIGIBLE for Global Administrator
  │
  ├── Day-to-day: No admin rights
  ├── When needed:
  │   └── Activates role in PIM portal
  │       ├── Provides justification
  │       ├── Approver notified (optional)
  │       ├── MFA required
  │       └── Gets admin rights for 8 hours
  │
  └── After 8 hours → Auto-deactivates

Risk: Time-limited privilege = minimal attack surface
```

### PIM Configuration

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         PIM ROLE SETTINGS                                    │
│                    "Global Administrator"                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ACTIVATION SETTINGS                                                        │
│  ├── Maximum activation duration: 8 hours                                   │
│  ├── Require justification: Yes                                            │
│  ├── Require ticket information: Yes                                        │
│  ├── Require MFA on activation: Yes                                        │
│  └── Require approval: Yes                                                  │
│      └── Approvers: Security Operations Team                               │
│                                                                              │
│  ASSIGNMENT SETTINGS                                                        │
│  ├── Allow permanent eligible assignment: No                               │
│  ├── Maximum eligible assignment duration: 365 days                        │
│  ├── Allow permanent active assignment: No                                 │
│  └── Maximum active assignment duration: 8 hours                           │
│                                                                              │
│  NOTIFICATION SETTINGS                                                      │
│  ├── Send notifications when eligible: Yes → Admins                        │
│  ├── Send notifications when activated: Yes → Admins + Security            │
│  └── Send notifications when assigned: Yes → Admins + User                 │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Case Study: Tiered Admin Access

**Scenario**: Implement least-privilege admin access with different tiers.

```
TIER 0: Critical (Most Restricted)
──────────────────────────────────
Roles: Global Administrator, Privileged Role Administrator
Activation: 4 hours max, approval required, ticket required
Eligible: Security team only (3 people)

TIER 1: High Privilege
──────────────────────
Roles: User Administrator, Security Administrator, Exchange Administrator
Activation: 8 hours max, justification required
Eligible: IT Operations team (10 people)

TIER 2: Standard Admin
─────────────────────
Roles: Helpdesk Administrator, Password Administrator
Activation: 8 hours max, no approval needed
Eligible: Service Desk team (25 people)
```

**Implementation**:

```bash
# Using Graph API to configure PIM settings

# Get role definition ID for Global Administrator
GET https://graph.microsoft.com/v1.0/directoryRoleTemplates?$filter=displayName eq 'Global Administrator'

# Update role settings
PATCH https://graph.microsoft.com/v1.0/policies/roleManagementPolicies/{policy-id}

{
  "rules": [
    {
      "@odata.type": "#microsoft.graph.unifiedRoleManagementPolicyExpirationRule",
      "id": "Expiration_EndUser_Assignment",
      "isExpirationRequired": true,
      "maximumDuration": "PT4H"  // 4 hours max activation
    },
    {
      "@odata.type": "#microsoft.graph.unifiedRoleManagementPolicyApprovalRule",
      "id": "Approval_EndUser_Assignment",
      "setting": {
        "isApprovalRequired": true,
        "approvalStages": [
          {
            "approvalStageTimeOutInDays": 1,
            "isApproverJustificationRequired": true,
            "primaryApprovers": [
              {
                "@odata.type": "#microsoft.graph.groupMembers",
                "groupId": "{security-ops-group-id}"
              }
            ]
          }
        ]
      }
    },
    {
      "@odata.type": "#microsoft.graph.unifiedRoleManagementPolicyAuthenticationContextRule",
      "id": "AuthenticationContext_EndUser_Assignment",
      "isEnabled": true,
      "claimValue": "c1"  // Requires strong authentication
    }
  ]
}
```

---

## Lifecycle Workflows

### Automating Joiner/Mover/Leaver

```
JOINER WORKFLOW:
────────────────
Trigger: New employee created in Entra ID (from HR system)
         └── Attribute: employeeHireDate = today

Actions:
├── 1. Send welcome email
├── 2. Add to "All Employees" group
├── 3. Add to department group (based on department attribute)
├── 4. Generate temporary access pass (TAP)
├── 5. Notify manager
└── 6. Enable account


MOVER WORKFLOW:
───────────────
Trigger: Department attribute changed

Actions:
├── 1. Remove from old department group
├── 2. Add to new department group
├── 3. Trigger access review for old access
├── 4. Notify new manager
└── 5. Notify user of access changes


LEAVER WORKFLOW:
────────────────
Trigger: employeeLeaveDate = today

Actions:
├── 1. Disable sign-in
├── 2. Revoke all sessions
├── 3. Remove from all groups
├── 4. Disable PIM eligible assignments
├── 5. Remove license assignments
├── 6. Convert mailbox to shared (optional)
└── 7. Notify manager and IT
```

### Example: Pre-Hire Workflow

```
Trigger: employeeHireDate = today + 7 days (one week before start)

Actions:
┌─────────────────────────────────────────────────────────────────────┐
│ 1. Create user account (disabled)                                   │
│    └── Set: mail, displayName, department, manager                 │
├─────────────────────────────────────────────────────────────────────┤
│ 2. Add to groups based on rules:                                    │
│    └── IF department = "Engineering" THEN add to "Engineers" group │
├─────────────────────────────────────────────────────────────────────┤
│ 3. Assign licenses:                                                 │
│    └── Microsoft 365 E5                                            │
├─────────────────────────────────────────────────────────────────────┤
│ 4. Send email to manager:                                           │
│    └── "New team member {displayName} starts on {hireDate}"        │
└─────────────────────────────────────────────────────────────────────┘

Trigger: employeeHireDate = today (start date)

Actions:
┌─────────────────────────────────────────────────────────────────────┐
│ 1. Enable user account                                              │
├─────────────────────────────────────────────────────────────────────┤
│ 2. Generate Temporary Access Pass                                   │
│    └── Valid for 24 hours, one-time use                            │
├─────────────────────────────────────────────────────────────────────┤
│ 3. Send welcome email to user                                       │
│    └── Include: TAP, links to onboarding, IT support contact       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Governance Reporting & Audit

### Key Reports

```kusto
// KQL: Users with no sign-in in 90 days who still have group memberships
let InactiveThreshold = ago(90d);
SigninLogs
| summarize LastSignIn = max(TimeGenerated) by UserPrincipalName
| where LastSignIn < InactiveThreshold
| join kind=inner (
    IdentityInfo
    | where GroupMembership != ""
) on UserPrincipalName
| project UserPrincipalName, LastSignIn, GroupMembership
```

```kusto
// KQL: Access review decisions over time
AuditLogs
| where Category == "EntitlementManagement"
| where OperationName has "access review"
| extend Decision = tostring(AdditionalDetails.Decision)
| summarize
    Approved = countif(Decision == "Approve"),
    Denied = countif(Decision == "Deny"),
    NoResponse = countif(Decision == "NotReviewed")
    by bin(TimeGenerated, 1d)
| render timechart
```

### Compliance Dashboard Metrics

| Metric | Target | How to Measure |
|--------|--------|----------------|
| % of groups with owners | > 95% | Graph API: Groups without owners |
| Access review completion rate | > 90% | Access review reports |
| Average time to deprovision | < 24 hours | Audit logs: disable events |
| Standing privilege count | Minimize | PIM: Active vs Eligible assignments |
| Guest user review frequency | Quarterly | Access review schedule |

---

## AWS to Azure Governance Comparison

| Capability | AWS Approach | Azure Approach |
|------------|--------------|----------------|
| Access reviews | Manual + 3rd party | Native Access Reviews |
| Access request | ServiceNow, custom | Entitlement Management |
| Just-in-time admin | (No native solution) | PIM |
| Lifecycle automation | Lambda + EventBridge | Lifecycle Workflows |
| Access certification | Manual / 3rd party | Access Reviews |
| Separation of duties | IAM policies | Access packages + PIM |

---

*Next: [Case Studies](case-studies.md)* | *Back to [Quick Reference](quick-reference.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [License](../LICENSE)*
