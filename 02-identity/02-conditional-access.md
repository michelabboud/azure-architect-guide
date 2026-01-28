# Conditional Access Deep Dive

## What is Conditional Access?

Conditional Access is Azure's Zero Trust policy engine. It sits between authentication and authorization, deciding **whether** to grant access based on conditions—not just **who** you are.

### AWS Comparison

```
AWS:                                    Azure:
────                                    ─────

IAM Policy Conditions                   Conditional Access
(Limited scope)                         (Comprehensive)

{                                       IF:
  "Condition": {                        • User is in "Executives" group
    "IpAddress": {                      • Accessing "Office 365"
      "aws:SourceIp": "10.0.0.0/8"     • From outside corporate network
    },                                  • On unmanaged device
    "Bool": {                           • Risk level is "medium"
      "aws:MultiFactorAuthPresent":
        "true"                          THEN:
    }                                   • Require MFA
  }                                     • Require compliant device
}                                       • Limit session to 1 hour
                                        • Block download of sensitive files
Only applies to specific API calls.
                                        Applies to ALL authentication.
```

### The Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│  USER ATTEMPTS TO ACCESS RESOURCE                                           │
│  ─────────────────────────────────                                          │
│                                                                              │
│  ┌──────────────┐     ┌──────────────────────────────────────────────────┐  │
│  │ 1. User      │     │ 2. Entra ID Authentication                       │  │
│  │    provides  │────→│    • Validates credentials                       │  │
│  │    credentials     │    • Checks password, MFA if configured          │  │
│  └──────────────┘     └────────────────────┬─────────────────────────────┘  │
│                                            │                                 │
│                                            ▼                                 │
│                       ┌──────────────────────────────────────────────────┐  │
│                       │ 3. CONDITIONAL ACCESS EVALUATION                 │  │
│                       │                                                   │  │
│                       │    Evaluates ALL applicable policies:            │  │
│                       │    • Who is this user?                           │  │
│                       │    • What app are they accessing?                │  │
│                       │    • Where are they? (location)                  │  │
│                       │    • What device? (compliant? managed?)          │  │
│                       │    • What's the risk level?                      │  │
│                       │    • What client app? (browser? mobile?)         │  │
│                       │                                                   │  │
│                       │    ┌────────────────────────────────────────┐    │  │
│                       │    │ Policy 1: Require MFA for all users   │    │  │
│                       │    │ Policy 2: Block legacy auth           │    │  │
│                       │    │ Policy 3: Require compliant device    │    │  │
│                       │    │           for sensitive apps          │    │  │
│                       │    └────────────────────────────────────────┘    │  │
│                       │                                                   │  │
│                       └────────────────────┬─────────────────────────────┘  │
│                                            │                                 │
│                          ┌─────────────────┴─────────────────┐              │
│                          │                                   │              │
│                          ▼                                   ▼              │
│              ┌───────────────────────┐          ┌───────────────────────┐   │
│              │ 4a. ACCESS GRANTED    │          │ 4b. ACCESS BLOCKED    │   │
│              │     (with controls)   │          │     or                │   │
│              │                       │          │     ADDITIONAL AUTH   │   │
│              │  • Session limited    │          │     REQUIRED          │   │
│              │  • Download restricted│          │                       │   │
│              └───────────────────────┘          └───────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Policy Components

### Assignments (IF conditions)

#### Users and Groups
```
Include:
├── All users
├── Specific users
├── Specific groups
├── Directory roles (e.g., Global Administrators)
├── Guest users
└── External users (by organization)

Exclude:
├── Break-glass accounts (ALWAYS exclude!)
├── Service accounts
└── Specific groups
```

#### Cloud Apps or Actions
```
Apps:
├── All cloud apps
├── Office 365
├── Azure Management
├── Microsoft Graph
├── Custom enterprise apps
└── Specific apps by ID

Actions (Preview):
├── Register security information
├── Register or join devices
└── User actions
```

#### Conditions
```
┌─────────────────────────────────────────────────────────────────────────┐
│ User Risk        │ Sign-in Risk     │ Device Platform   │ Location     │
│ • High           │ • High           │ • Android         │ • All        │
│ • Medium         │ • Medium         │ • iOS             │ • Named      │
│ • Low            │ • Low            │ • Windows         │   locations  │
│ • No risk        │ • No risk        │ • macOS           │ • Trusted    │
│                  │                  │ • Linux           │ • MFA trusted│
├──────────────────┴──────────────────┴───────────────────┴──────────────┤
│ Client Apps                         │ Device State                      │
│ • Browser                           │ • Device marked as compliant      │
│ • Mobile apps and desktop clients   │ • Hybrid Azure AD joined          │
│ • Exchange ActiveSync               │ • Device filtered by rule         │
│ • Other clients (legacy)            │                                   │
└─────────────────────────────────────┴───────────────────────────────────┘
```

### Access Controls (THEN actions)

#### Grant Controls
```
Block access
─────────────
• Complete denial

Grant access (with requirements):
─────────────────────────────────
• Require MFA
• Require device to be marked as compliant (Intune)
• Require Hybrid Azure AD joined device
• Require approved client app
• Require app protection policy
• Require password change
• Require authentication strength (passwordless, phishing-resistant)

Multiple controls:
• Require ALL selected controls (AND)
• Require ONE of selected controls (OR)
```

#### Session Controls
```
• App enforced restrictions (works with Office 365)
• Use Conditional Access App Control (MCAS integration)
• Sign-in frequency (force re-auth after X hours)
• Persistent browser session (remember on trusted devices)
• Customize token lifetime
• Disable resilience defaults
```

---

## Case Studies: Beginner → Expert

### Beginner: Require MFA for All Users

**Scenario**: Basic security—require MFA for everyone accessing any cloud app.

**Policy Configuration**:
```
Name: Require MFA - All Users

ASSIGNMENTS
├── Users: All users
├── Exclude:
│   └── Group: "Break-glass-accounts" (CRITICAL!)
├── Cloud apps: All cloud apps
└── Conditions: (none)

ACCESS CONTROLS
├── Grant: Grant access
│   └── Require: Multi-factor authentication
└── Session: (none)

State: Report-only → On
```

**Why This Design**:
> - **Why exclude break-glass accounts?** If MFA is down or misconfigured, you need emergency access. These accounts should have:
>   - Strong passwords (32+ characters)
>   - Stored securely offline
>   - Monitored for any sign-in
>   - Never used except emergencies
>
> - **Why Report-only first?** See impact before enforcement. Review sign-in logs for users who would be blocked.

**AWS Equivalent**: MFA enforcement in IAM Identity Center or IAM policies with `aws:MultiFactorAuthPresent` condition. But AWS can't enforce MFA on console sign-in the same way—users can still reach the console, then get denied on actions.

---

### Beginner: Block Legacy Authentication

**Scenario**: Legacy protocols (IMAP, SMTP, POP3) don't support MFA and are a major attack vector.

**The Problem**:
```
Modern Auth:                     Legacy Auth:
────────────                     ───────────
User → MFA → Access             User → Password only → Access
                                 (No MFA possible!)

Attackers love legacy auth:
• Password spray attacks
• Credential stuffing
• No MFA to stop them
```

**Policy Configuration**:
```
Name: Block Legacy Authentication

ASSIGNMENTS
├── Users: All users
├── Exclude:
│   └── Group: "Legacy-auth-exceptions" (if absolutely needed)
├── Cloud apps: All cloud apps
└── Conditions:
    └── Client apps:
        ├── Exchange ActiveSync clients
        └── Other clients

ACCESS CONTROLS
└── Grant: Block access

State: On
```

**Why This Design**:
> - **Why block entirely?** Legacy auth can't do MFA. Period. Blocking is the only secure option.
> - **Exceptions?** Some old applications genuinely need legacy auth. Document them, plan migration, and monitor closely.

---

### Intermediate: Require Compliant Device for Sensitive Apps

**Scenario**: Sensitive applications (HR systems, financial data) should only be accessed from company-managed, compliant devices.

**Architecture**:
```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│  COMPLIANT DEVICE (Intune-managed)          NON-COMPLIANT DEVICE        │
│  ──────────────────────────────────         ────────────────────        │
│                                                                          │
│  ┌──────────────────────┐                   ┌──────────────────────┐    │
│  │ • BitLocker enabled  │                   │ • Personal laptop    │    │
│  │ • AV up to date      │                   │ • No MDM             │    │
│  │ • OS patched         │                   │ • Unknown security   │    │
│  │ • Firewall on        │                   │   status             │    │
│  │ • Enrolled in Intune │                   │                      │    │
│  └──────────────────────┘                   └──────────────────────┘    │
│           │                                           │                  │
│           ▼                                           ▼                  │
│  ┌──────────────────────┐                   ┌──────────────────────┐    │
│  │ ✓ Access to HR       │                   │ ✗ Blocked from HR    │    │
│  │ ✓ Access to Finance  │                   │ ✗ Blocked from       │    │
│  │ ✓ Full Office 365    │                   │   Finance            │    │
│  └──────────────────────┘                   │ ○ Basic Office 365   │    │
│                                             │   (web only)         │    │
│                                             └──────────────────────┘    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

**Policy Configuration**:
```
Name: Require Compliant Device - Sensitive Apps

ASSIGNMENTS
├── Users: All users
├── Exclude: Break-glass accounts
├── Cloud apps:
│   ├── Workday (HR)
│   ├── SAP Finance
│   └── Custom LOB apps
└── Conditions:
    └── Device platforms: All platforms

ACCESS CONTROLS
├── Grant: Grant access
│   ├── Require: Device to be marked as compliant
│   └── Require: Multi-factor authentication
│   └── Operator: Require ALL selected controls
└── Session: (none)

State: On
```

**Integration with Intune**:
```
Intune Compliance Policy defines "compliant":
┌─────────────────────────────────────────────────────────────────────┐
│ Windows Compliance Policy                                            │
├─────────────────────────────────────────────────────────────────────┤
│ System Security:                                                     │
│ ├── Require BitLocker: Yes                                          │
│ ├── Require Secure Boot: Yes                                        │
│ └── Require TPM: Yes                                                │
│                                                                      │
│ Device Health:                                                       │
│ ├── Require device to be at or under threat level: Low             │
│ └── Windows health attestation: Required                            │
│                                                                      │
│ Device Properties:                                                   │
│ ├── Minimum OS version: 10.0.19045                                  │
│ └── Maximum OS version: (blank)                                     │
│                                                                      │
│ Actions for noncompliance:                                          │
│ ├── Mark device noncompliant: Immediately                           │
│ ├── Send email to user: After 1 day                                │
│ └── Retire device: After 30 days                                   │
└─────────────────────────────────────────────────────────────────────┘
```

**Why This Design**:
> - **Why device compliance AND MFA?** Defense in depth. Stolen compliant device still needs MFA.
> - **Why not all apps?** Business reality—contractors, partners may need some access from personal devices. Apply strict controls to sensitive apps.

---

### Expert: Risk-Based Adaptive Access

**Scenario**: Dynamically adjust access requirements based on real-time risk assessment.

**Risk Signals (Entra ID Protection)**:
```
Sign-in Risk:                          User Risk:
─────────────                          ──────────
• Anonymous IP address                 • Leaked credentials detected
• Atypical travel                      • Threat intelligence
• Malware-linked IP                    • Azure AD behavior analytics
• Unfamiliar sign-in properties        • Anomalous user activity
• Password spray attack
• Suspicious browser
```

**Multi-Policy Architecture**:
```
┌─────────────────────────────────────────────────────────────────────────┐
│                        RISK-BASED POLICY SET                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │ POLICY 1: Baseline - All users get MFA                             │ │
│  │ Users: All  |  Apps: All  |  Grant: Require MFA                    │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │ POLICY 2: Medium Sign-in Risk - Additional verification            │ │
│  │ Users: All  |  Apps: All  |  Condition: Sign-in risk = Medium     │ │
│  │ Grant: Require MFA + Require compliant device                      │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │ POLICY 3: High Sign-in Risk - Block and investigate                │ │
│  │ Users: All  |  Apps: All  |  Condition: Sign-in risk = High       │ │
│  │ Grant: Block access                                                 │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │ POLICY 4: High User Risk - Force password change                   │ │
│  │ Users: All  |  Apps: All  |  Condition: User risk = High          │ │
│  │ Grant: Require MFA + Require password change                       │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘

Result:
─────────
Normal sign-in (no risk): MFA only
Medium risk sign-in: MFA + compliant device
High risk sign-in: Blocked
High risk user: MFA + password change
```

**Implementation Details**:

```json
// Policy 3: Block High Sign-in Risk (ARM/Bicep representation)
{
  "displayName": "Block High Sign-in Risk",
  "state": "enabled",
  "conditions": {
    "users": {
      "includeUsers": ["All"],
      "excludeGroups": ["break-glass-accounts-group-id"]
    },
    "applications": {
      "includeApplications": ["All"]
    },
    "signInRiskLevels": ["high"]
  },
  "grantControls": {
    "operator": "OR",
    "builtInControls": ["block"]
  }
}
```

**Why This Design**:
> - **Why multiple policies?** Conditional Access evaluates ALL applicable policies. The most restrictive wins. This creates a layered defense.
> - **Why block high risk instead of just MFA?** High-risk sign-ins often mean active attack. Blocking and investigating is safer than hoping MFA stops them.
> - **Why force password change on high user risk?** If credentials leaked, password change invalidates them.

---

## Common Policy Patterns

### Pattern 1: Geographic Restrictions

```
Name: Block Access from High-Risk Countries

ASSIGNMENTS
├── Users: All users
├── Cloud apps: All cloud apps
└── Conditions:
    └── Locations:
        ├── Include: Selected locations
        │   └── "High-risk-countries" (named location)
        └── Exclude: None

ACCESS CONTROLS
└── Grant: Block

Named Location "High-risk-countries":
├── Countries: [List of countries where you have no business]
└── Mark as trusted: No
```

### Pattern 2: Privileged Access Workstations

```
Name: Require PAW for Admin Portals

ASSIGNMENTS
├── Users:
│   └── Directory roles: Global Administrator, Security Administrator, etc.
├── Cloud apps:
│   ├── Microsoft Azure Management
│   ├── Microsoft 365 admin center
│   └── Entra ID admin portal
└── Conditions:
    └── Filter for devices:
        └── Rule: device.extensionAttribute1 -eq "PAW"

ACCESS CONTROLS
├── Grant: Grant access
│   ├── Require: MFA
│   ├── Require: Device to be marked as compliant
│   └── Operator: Require ALL selected controls
└── Session:
    └── Sign-in frequency: 4 hours
```

### Pattern 3: Session Controls for Contractors

```
Name: Limited Session for External Users

ASSIGNMENTS
├── Users:
│   └── User types: Guest users
├── Cloud apps: All cloud apps
└── Conditions: None

ACCESS CONTROLS
├── Grant: Grant access
│   └── Require: MFA
└── Session:
    ├── Sign-in frequency: 1 hour
    └── Persistent browser session: Never persistent
```

---

## Troubleshooting Conditional Access

### Sign-in Logs Analysis

```kusto
// KQL: Find blocked sign-ins by CA
SigninLogs
| where TimeGenerated > ago(24h)
| where ConditionalAccessStatus == "failure"
| project TimeGenerated,
          UserPrincipalName,
          AppDisplayName,
          ConditionalAccessPolicies,
          ResultType,
          ResultDescription
| order by TimeGenerated desc
```

### Common Issues

| Symptom | Likely Cause | Fix |
|---------|--------------|-----|
| User blocked unexpectedly | Policy too broad | Check exclusions, use Report-only first |
| MFA not triggered | User excluded or app not covered | Review policy assignments |
| "Can't get there from here" | Location-based block | Check named locations, user VPN |
| Compliant device rejected | Intune sync delay or policy mismatch | Check Intune portal, wait for sync |
| Policy not applying | Policy in Report-only mode | Change to On |
| Service Principal blocked | CA applied to workload identity | Exclude or use Workload Identity policies |

### What-If Tool

```
Entra ID Portal → Security → Conditional Access → What If

Input:
• User: john.doe@contoso.com
• Cloud app: Office 365
• IP address: 203.0.113.50
• Device platform: Windows
• Client app: Browser

Output:
• Policy A: Will apply (requires MFA)
• Policy B: Will not apply (different user group)
• Policy C: Will apply (requires compliant device)
• Result: User must complete MFA AND have compliant device
```

---

## AWS Comparison: What Conditional Access Can Do That AWS Can't

| Capability | Conditional Access | AWS Equivalent |
|------------|-------------------|----------------|
| Block based on device compliance | ✓ Native | ✗ Need 3rd party |
| Risk-based policies | ✓ Native | ✗ Limited |
| Force MFA before any access | ✓ Native | ~ Console MFA only |
| Session time limits | ✓ Native | ~ STS duration |
| App-specific controls | ✓ Granular | ✗ Limited |
| Location-based access | ✓ Named locations | ~ IP conditions in policies |
| Client app restrictions | ✓ Block legacy | ✗ Not available |
| Real-time risk evaluation | ✓ Native | ✗ GuardDuty is detective |

---

*Next: [Governance](03-governance.md)* | *Back to [Quick Reference](quick-reference.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [APACHE 2.0 License](../LICENSE)*
