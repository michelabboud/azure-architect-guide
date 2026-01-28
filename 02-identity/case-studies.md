# Identity Case Studies

Real-world scenarios with architectural decisions explained.

---

## Case Study 1: Enterprise Identity Migration (Beginner)

### Scenario

**Company**: Mid-size financial services (2,000 employees)
**Current State**: On-premises Active Directory, Okta for SSO
**Goal**: Migrate to Azure/M365, consolidate identity to Entra ID

### The Debate

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        OPTION A: Hybrid Identity                            │
│                        (Keep AD, sync to Entra ID)                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────┐         ┌─────────────────┐         ┌──────────────┐   │
│  │  On-Premises    │  Sync   │   Entra ID      │  Auth   │  Cloud Apps  │   │
│  │  Active         │ ──────→ │   Connect       │ ──────→ │  M365        │   │
│  │  Directory      │         │                 │         │  Azure       │   │
│  └─────────────────┘         └─────────────────┘         └──────────────┘   │
│                                                                              │
│  PROS:                              CONS:                                   │
│  • Gradual migration               • Complexity (two directories)          │
│  • Familiar for IT team            • Sync issues possible                  │
│  • Legacy app support              • On-prem infrastructure required       │
│  • Password hash sync for DR       • Dual management overhead              │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                        OPTION B: Cloud-Only Identity                        │
│                        (Migrate fully to Entra ID)                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────┐         ┌─────────────────┐                            │
│  │   Entra ID      │  Auth   │  All Apps       │                            │
│  │   (Source of    │ ──────→ │  • M365         │                            │
│  │    Truth)       │         │  • Azure        │                            │
│  └─────────────────┘         │  • SaaS         │                            │
│                              └─────────────────┘                            │
│                                                                              │
│  PROS:                              CONS:                                   │
│  • Single identity source          • Legacy app compatibility              │
│  • No on-prem dependency           • Big-bang migration risk              │
│  • Full Entra ID features          • Retraining required                  │
│  • Lower TCO long-term             • Domain-joined devices?               │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### The Decision

**Chosen**: Option A (Hybrid Identity) with planned transition to cloud-only

**Reasoning**:
> 1. **Risk mitigation**: Gradual migration reduces blast radius of issues
> 2. **Legacy apps**: Several critical apps require Kerberos/NTLM
> 3. **Regulatory**: Financial regulators want audit trail of changes
> 4. **Timeline**: 18-month migration allows thorough testing
> 5. **Endpoint strategy**: Windows Autopilot + Hybrid Azure AD Join transitionally

### Architecture

```
Phase 1 (Months 1-6): Foundation
──────────────────────────────────
┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐
│   On-Prem AD    │      │  Entra ID       │      │  M365           │
│                 │─────→│  Connect        │─────→│  (Pilot users)  │
│                 │ Sync │                 │      │                 │
└─────────────────┘      └─────────────────┘      └─────────────────┘

Configuration:
• Password Hash Sync (PHS) enabled
• Seamless SSO configured
• Pilot group: 50 IT users
• Conditional Access: Report-only mode

Phase 2 (Months 7-12): Migration
────────────────────────────────
• Migrate all users to M365
• Enable Conditional Access policies
• Implement PIM for admins
• Decommission Okta

Phase 3 (Months 13-18): Optimization
────────────────────────────────────
• Move to cloud-only for new hires
• Migrate legacy apps or implement Azure AD App Proxy
• Transition to Azure AD Join devices
• Plan AD decommission (future)
```

### AWS Comparison

If this company were AWS-focused:
- Would use **AWS IAM Identity Center** with AD Connector
- **AWS Directory Service** for AD in cloud
- Similar hybrid approach but different tools

---

## Case Study 2: Zero Trust for Remote Workforce (Intermediate)

### Scenario

**Company**: Global consulting firm (10,000 employees)
**Challenge**: 80% remote workforce, BYOD common, sensitive client data
**Goal**: Implement Zero Trust without blocking productivity

### The Debate

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    OPTION A: Strict Device Compliance                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Policy: Only corporate-managed, compliant devices can access anything     │
│                                                                              │
│  PROS:                              CONS:                                   │
│  • Maximum security                 • Blocks BYOD completely              │
│  • Full device control              • Consultant laptops?                 │
│  • Consistent security posture      • Partner access blocked              │
│  • Easy to audit                    • Productivity impact HIGH            │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                    OPTION B: Risk-Based Adaptive Access                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Policy: Access controls vary based on risk signals and data sensitivity   │
│                                                                              │
│  PROS:                              CONS:                                   │
│  • Flexible for business needs      • More complex to implement           │
│  • Supports BYOD with controls      • Requires data classification        │
│  • Better user experience           • Multiple policies to manage         │
│  • Balances security/productivity   • Needs continuous tuning             │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### The Decision

**Chosen**: Option B (Risk-Based Adaptive Access)

**Reasoning**:
> 1. **Business reality**: Consulting requires partner/client collaboration
> 2. **User experience**: Overly strict controls = shadow IT
> 3. **Data-centric**: Protect the data, not just the perimeter
> 4. **Measurable**: Can demonstrate security posture to clients

### Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      ZERO TRUST ARCHITECTURE                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  DATA CLASSIFICATION:                                                        │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐                │
│  │ PUBLIC         │  │ INTERNAL       │  │ CONFIDENTIAL   │                │
│  │ • Marketing    │  │ • HR policies  │  │ • Client data  │                │
│  │ • Published    │  │ • Internal     │  │ • Financial    │                │
│  │   content      │  │   comms        │  │ • Strategy     │                │
│  └────────────────┘  └────────────────┘  └────────────────┘                │
│         │                   │                   │                           │
│         ▼                   ▼                   ▼                           │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │                    CONDITIONAL ACCESS POLICIES                      │    │
│  ├────────────────────────────────────────────────────────────────────┤    │
│  │                                                                      │    │
│  │  BASELINE (All apps):                                               │    │
│  │  • Require MFA                                                       │    │
│  │  • Block legacy auth                                                 │    │
│  │                                                                      │    │
│  │  INTERNAL DATA:                                                     │    │
│  │  • MFA + Entra registered device OR compliant device                │    │
│  │  • Session: 12 hours                                                │    │
│  │                                                                      │    │
│  │  CONFIDENTIAL DATA:                                                 │    │
│  │  • MFA + Compliant device only                                      │    │
│  │  • Location: Named locations only                                   │    │
│  │  • Session: 4 hours                                                 │    │
│  │  • App protection policy required                                   │    │
│  │                                                                      │    │
│  │  HIGH RISK SIGNALS:                                                 │    │
│  │  • Any sign-in risk = High → Block                                  │    │
│  │  • User risk = High → Password change + MFA                         │    │
│  │                                                                      │    │
│  └────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  DEVICE TIERS:                                                              │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐                │
│  │ COMPLIANT      │  │ REGISTERED     │  │ UNMANAGED      │                │
│  │ (Intune MDM)   │  │ (Entra Join)   │  │ (BYOD)         │                │
│  │                │  │                │  │                │                │
│  │ Full access    │  │ Internal data  │  │ Public data    │                │
│  │ to all data    │  │ via web only   │  │ + web M365     │                │
│  └────────────────┘  └────────────────┘  └────────────────┘                │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Implementation Details

**Policy Set**:

```
POLICY 1: Baseline MFA
──────────────────────
Users: All users
Apps: All cloud apps
Grant: Require MFA
Exclude: Break-glass accounts

POLICY 2: Block Legacy Auth
────────────────────────────
Users: All users
Apps: All cloud apps
Conditions: Client apps = Exchange ActiveSync, Other clients
Grant: Block

POLICY 3: Internal Data - Device Required
──────────────────────────────────────────
Users: All users
Apps: SharePoint sites tagged "Internal"
      (uses authentication context)
Grant: Require MFA AND (Compliant OR Registered device)
Session: Sign-in frequency 12 hours

POLICY 4: Confidential Data - Compliant Only
─────────────────────────────────────────────
Users: All users
Apps: SharePoint sites tagged "Confidential"
      Client data systems
Conditions:
  Locations: Include all, Exclude trusted
Grant: Require MFA AND Compliant device AND App protection
Session: Sign-in frequency 4 hours

POLICY 5: High Risk Block
──────────────────────────
Users: All users
Apps: All cloud apps
Conditions: Sign-in risk = High
Grant: Block

POLICY 6: User Risk Remediation
────────────────────────────────
Users: All users
Apps: All cloud apps
Conditions: User risk = High
Grant: Require MFA AND password change
```

### AWS Comparison

AWS equivalent would require:
- **AWS Verified Access** for conditional access (newer, less mature)
- **Third-party** MDM integration
- **Custom** risk scoring (no native equivalent to Identity Protection)
- Significantly more manual configuration

---

## Case Study 3: Multi-Tenant SaaS Identity (Expert)

### Scenario

**Company**: B2B SaaS platform serving enterprise customers
**Challenge**: Each customer wants their own identity provider
**Goal**: Support customer IdPs while maintaining unified platform identity

### The Debate

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    OPTION A: Single Tenant, B2B Guests                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────┐                                                        │
│  │ SaaS Provider   │                                                        │
│  │ Entra ID Tenant │                                                        │
│  │                 │                                                        │
│  │ ┌─────────────┐ │     ┌─────────────┐                                   │
│  │ │ Guest users │◄├─────│ Customer A  │                                   │
│  │ │ from        │ │     │ Entra ID    │                                   │
│  │ │ Customer A  │ │     └─────────────┘                                   │
│  │ └─────────────┘ │     ┌─────────────┐                                   │
│  │ ┌─────────────┐ │     │ Customer B  │                                   │
│  │ │ Guest users │◄├─────│ Okta        │                                   │
│  │ │ from        │ │     └─────────────┘                                   │
│  │ │ Customer B  │ │                                                        │
│  │ └─────────────┘ │                                                        │
│  └─────────────────┘                                                        │
│                                                                              │
│  PROS:                              CONS:                                   │
│  • Simple architecture             • All users in one directory            │
│  • Native Entra ID features        • Customer data isolation concerns      │
│  • Built-in B2B federation         • Scaling limits                        │
│                                    • Customer can't manage own users       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                    OPTION B: Entra ID B2C                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                         B2C Tenant                                   │   │
│  │  ┌─────────────────────────────────────────────────────────────┐    │   │
│  │  │              Custom Policies (Identity Experience)           │    │   │
│  │  └─────────────────────────────────────────────────────────────┘    │   │
│  │           │                    │                    │                │   │
│  │           ▼                    ▼                    ▼                │   │
│  │  ┌─────────────┐      ┌─────────────┐      ┌─────────────┐         │   │
│  │  │ Customer A  │      │ Customer B  │      │ Customer C  │         │   │
│  │  │ (SAML IdP)  │      │ (OIDC IdP)  │      │ (Local)     │         │   │
│  │  └─────────────┘      └─────────────┘      └─────────────┘         │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  PROS:                              CONS:                                   │
│  • Designed for this scenario      • Separate from workforce identity     │
│  • Custom branding per customer    • Complex custom policies              │
│  • Federation flexibility          • No Conditional Access (limited)      │
│  • Massive scale                   • Different feature set than Entra ID  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                    OPTION C: External Identities + Tenant Isolation         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    Multi-Tenant Application                          │   │
│  │                                                                       │   │
│  │  App registration configured for "Any Azure AD directory + personal"│   │
│  │                                                                       │   │
│  └───────────────────────────────┬───────────────────────────────────────┘   │
│                                  │                                          │
│         ┌────────────────────────┼────────────────────────┐                │
│         ▼                        ▼                        ▼                │
│  ┌─────────────┐          ┌─────────────┐          ┌─────────────┐        │
│  │ Customer A  │          │ Customer B  │          │ Customer C  │        │
│  │ Tenant      │          │ Tenant      │          │ Tenant      │        │
│  │             │          │             │          │             │        │
│  │ Users auth  │          │ Users auth  │          │ Users auth  │        │
│  │ in their    │          │ in their    │          │ in their    │        │
│  │ own tenant  │          │ own tenant  │          │ own tenant  │        │
│  └─────────────┘          └─────────────┘          └─────────────┘        │
│                                                                              │
│  PROS:                              CONS:                                   │
│  • Full tenant isolation           • Customers need Entra ID               │
│  • Customer controls their users   • App consent required per tenant       │
│  • Each customer's CA applies      • More complex app architecture         │
│  • Enterprise-ready                • Some customers won't have Entra ID    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### The Decision

**Chosen**: Hybrid approach - Option C for enterprise customers + Option B for others

**Reasoning**:
> 1. **Enterprise customers** (80% revenue) demand tenant isolation and want their Conditional Access policies to apply
> 2. **Smaller customers** need simpler onboarding—B2C handles this
> 3. **Compliance**: Enterprise customers have audit requirements that need tenant separation
> 4. **Architecture**: Multi-tenant app design supports both patterns

### Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    HYBRID IDENTITY ARCHITECTURE                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│                         ┌───────────────────────┐                           │
│                         │    SaaS Application   │                           │
│                         │    (Multi-tenant)     │                           │
│                         └───────────┬───────────┘                           │
│                                     │                                        │
│              ┌──────────────────────┼──────────────────────┐                │
│              │                      │                      │                │
│              ▼                      ▼                      ▼                │
│  ┌─────────────────────┐  ┌─────────────────┐  ┌─────────────────────────┐ │
│  │   Enterprise Path   │  │    SMB Path     │  │   Self-Service Path     │ │
│  │   (Multi-tenant     │  │    (B2C with    │  │   (B2C Local            │ │
│  │    Entra ID)        │  │     SAML/OIDC)  │  │    Accounts)            │ │
│  └──────────┬──────────┘  └────────┬────────┘  └────────────┬────────────┘ │
│             │                      │                         │              │
│             ▼                      ▼                         ▼              │
│  Customer's Entra ID       Customer's IdP          B2C Local Directory    │
│  - Full CA policies        - Federation only       - Self-service signup  │
│  - Customer-managed        - Limited policies      - Email verification   │
│  - Audit in their tenant   - SaaS manages         - Password policies     │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Implementation Considerations

**For Multi-Tenant Entra ID App**:
```json
// App manifest configuration
{
  "signInAudience": "AzureADMultipleOrgs",
  "accessTokenAcceptedVersion": 2,
  "optionalClaims": {
    "idToken": [
      { "name": "tid", "essential": true },  // Tenant ID
      { "name": "acct", "essential": true }  // Account type
    ]
  }
}
```

**For B2C Custom Policy (Federation)**:
```xml
<!-- Trust framework policy for SAML IdP -->
<ClaimsProvider>
  <DisplayName>Customer SAML IdP</DisplayName>
  <TechnicalProfiles>
    <TechnicalProfile Id="CustomerSAML">
      <Protocol Name="SAML2" />
      <Metadata>
        <Item Key="PartnerEntity">https://customer-idp.com/metadata</Item>
        <Item Key="WantsSignedAssertions">true</Item>
      </Metadata>
    </TechnicalProfile>
  </TechnicalProfiles>
</ClaimsProvider>
```

### AWS Comparison

AWS equivalent (Amazon Cognito):
- **Cognito User Pools** for B2C-like functionality
- **Cognito Identity Pools** for federation
- Less mature than Azure B2C for enterprise scenarios
- Would need custom development for equivalent multi-tenant support

---

## Summary: Decision Framework

When designing identity architecture, ask:

```
1. WHO are the users?
   ├── Internal employees → Entra ID (workforce)
   ├── External business partners → B2B Guest
   └── Customers/consumers → B2C

2. WHAT level of control does each party need?
   ├── Full control → Their own tenant
   ├── Limited control → Guest access with CA
   └── No control → B2C managed

3. WHAT are the compliance requirements?
   ├── Audit in customer's tenant → Multi-tenant app
   ├── Audit in your tenant → B2B or B2C
   └── Shared responsibility → Hybrid

4. WHAT's the scale?
   ├── < 50K users → B2B guests work fine
   ├── > 50K users → Consider B2C
   └── Millions → Definitely B2C

5. WHAT federation protocols do customers use?
   ├── All Entra ID → Multi-tenant app
   ├── Mixed (SAML, OIDC) → B2C with federation
   └── Unknown → B2C with local + federation options
```

---

*Back to: [Chapter Overview](README.md)* | *Next Chapter: [Security](../02-security/README.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [License](../LICENSE)*
