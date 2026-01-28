# Information Protection (Sensitivity Labels)

## What is Information Protection?

Microsoft Purview Information Protection uses **sensitivity labels** to classify and protect your organization's data. Think of labels as persistent tags that travel with the content and can enforce protection automatically.

### AWS Comparison

```
┌─────────────────────────────────────────────────────────────────────────────┐
│              INFORMATION PROTECTION: AWS vs AZURE                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  AWS APPROACH:                         MICROSOFT PURVIEW:                   │
│  ──────────────                        ─────────────────                    │
│                                                                              │
│  Manual/Custom:                        Native & Unified:                    │
│  ┌─────────────────────────┐          ┌─────────────────────────────────┐   │
│  │ S3 Object tags          │          │ Sensitivity Labels:              │   │
│  │ • Key-value pairs       │          │ • Persistent metadata            │   │
│  │ • No encryption tied    │          │ • Built-in encryption            │   │
│  │ • No visual marking     │          │ • Visual markings (watermarks)   │   │
│  │                         │          │ • Access control                 │   │
│  │ KMS for encryption      │          │ • Works across M365 + beyond     │   │
│  │ • Separate from tags    │          │                                   │   │
│  │ • Manual key management │          │ Scope:                            │   │
│  │                         │          │ • Office files                    │   │
│  │ No equivalent for:      │          │ • PDFs                            │   │
│  │ • Office documents      │          │ • Emails                          │   │
│  │ • Email classification  │          │ • Teams/SharePoint sites          │   │
│  │ • Visual watermarks     │          │ • Power BI                        │   │
│  │ • Auto-classification   │          │ • Azure SQL columns               │   │
│  │                         │          │ • Schematized data assets         │   │
│  └─────────────────────────┘          └─────────────────────────────────┘   │
│                                                                              │
│  To achieve similar in AWS:            Microsoft approach:                  │
│  • Third-party DRM tools               One label system, everywhere        │
│  • Custom Lambda classification        Automatic classification            │
│  • Manual encryption management        Built-in Azure RMS encryption       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Sensitivity Label Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SENSITIVITY LABEL ARCHITECTURE                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│                  ┌────────────────────────────────────┐                     │
│                  │     PURVIEW COMPLIANCE CENTER      │                     │
│                  │                                    │                     │
│                  │  Define Labels & Policies          │                     │
│                  │  ┌────────────────────────────┐   │                     │
│                  │  │ Public                     │   │                     │
│                  │  │ Internal                   │   │                     │
│                  │  │ Confidential               │   │                     │
│                  │  │   └─ Confidential\HR       │   │                     │
│                  │  │   └─ Confidential\Finance  │   │                     │
│                  │  │ Highly Confidential        │   │                     │
│                  │  └────────────────────────────┘   │                     │
│                  └─────────────────┬──────────────────┘                     │
│                                    │                                         │
│             ┌──────────────────────┼──────────────────────┐                 │
│             │                      │                      │                 │
│             ▼                      ▼                      ▼                 │
│  ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐ │
│  │   OFFICE APPS       │  │   CLOUD SERVICES    │  │   OTHER PLATFORMS   │ │
│  │                     │  │                     │  │                     │ │
│  │  Word, Excel, PPT   │  │  SharePoint Online  │  │  Azure SQL          │ │
│  │  Outlook            │  │  OneDrive           │  │  Power BI           │ │
│  │  Office for Web     │  │  Teams              │  │  Azure Purview      │ │
│  │                     │  │  Exchange Online    │  │  Defender for Cloud │ │
│  │  Features:          │  │                     │  │                     │ │
│  │  • Manual labeling  │  │  Features:          │  │  Features:          │ │
│  │  • Auto-labeling    │  │  • Container labels │  │  • Schema labels    │ │
│  │  • Recommended      │  │  • Default labels   │  │  • Column-level     │ │
│  │  • Visual marks     │  │  • Access control   │  │  • Lineage tracking │ │
│  └─────────────────────┘  └─────────────────────┘  └─────────────────────┘ │
│                                                                              │
│                    PROTECTION TRAVELS WITH THE CONTENT                      │
│                                                                              │
│    ┌─────────┐     ┌─────────┐     ┌─────────┐     ┌─────────┐            │
│    │ Created │────▶│ Shared  │────▶│Downloaded────▶│ Opened  │            │
│    │ in Word │     │via Email│     │ locally │     │ on any  │            │
│    │         │     │         │     │         │     │ device  │            │
│    │ 🏷️ Label │     │ 🏷️ Label │     │ 🏷️ Label │     │ 🏷️ Label │            │
│    │ 🔒 Prot  │     │ 🔒 Prot  │     │ 🔒 Prot  │     │ 🔒 Prot  │            │
│    └─────────┘     └─────────┘     └─────────┘     └─────────┘            │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Sensitivity Label Components

### Label Settings

```
LABEL CONFIGURATION OPTIONS:
────────────────────────────

┌────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  LABEL: Confidential                                                        │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 1. ENCRYPTION SETTINGS                                               │   │
│  │    ┌──────────────────────────────────────────────────────────────┐ │   │
│  │    │ ☑ Encrypt content                                            │ │   │
│  │    │                                                               │ │   │
│  │    │ Permissions:                                                  │ │   │
│  │    │ ○ Assign permissions now (static)                            │ │   │
│  │    │   • Specify users/groups who can access                      │ │   │
│  │    │   • Set expiration date                                      │ │   │
│  │    │                                                               │ │   │
│  │    │ ● Let users assign permissions (dynamic)                     │ │   │
│  │    │   • Outlook: Do Not Forward / Encrypt-Only                   │ │   │
│  │    │   • Office apps: User specifies who can access               │ │   │
│  │    │                                                               │ │   │
│  │    │ Rights:                                                       │ │   │
│  │    │ ☑ View      ☑ Edit       ☐ Copy                             │ │   │
│  │    │ ☐ Print     ☑ Save       ☐ Export                           │ │   │
│  │    │ ☐ Forward   ☐ Full Control                                  │ │   │
│  │    └──────────────────────────────────────────────────────────────┘ │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 2. CONTENT MARKING                                                   │   │
│  │    ┌──────────────────────────────────────────────────────────────┐ │   │
│  │    │ ☑ Add a watermark                                            │ │   │
│  │    │   Text: "CONFIDENTIAL"                                       │ │   │
│  │    │   Font: Arial, 12pt, Red                                     │ │   │
│  │    │   Position: Diagonal                                         │ │   │
│  │    │                                                               │ │   │
│  │    │ ☑ Add a header                                               │ │   │
│  │    │   Text: "Confidential - Internal Use Only"                   │ │   │
│  │    │                                                               │ │   │
│  │    │ ☑ Add a footer                                               │ │   │
│  │    │   Text: "Classification: Confidential"                       │ │   │
│  │    └──────────────────────────────────────────────────────────────┘ │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 3. AUTO-LABELING (Optional)                                         │   │
│  │    ┌──────────────────────────────────────────────────────────────┐ │   │
│  │    │ ☑ Auto-label content matching these conditions:              │ │   │
│  │    │                                                               │ │   │
│  │    │ IF content contains:                                         │ │   │
│  │    │ • Credit Card Number (High confidence, 1+ instances)         │ │   │
│  │    │ OR                                                            │ │   │
│  │    │ • U.S. SSN (High confidence, 1+ instances)                   │ │   │
│  │    │                                                               │ │   │
│  │    │ THEN: ● Recommend label  ○ Apply label automatically        │ │   │
│  │    │                                                               │ │   │
│  │    │ Message: "This document appears to contain sensitive PII.   │ │   │
│  │    │          We recommend applying the Confidential label."     │ │   │
│  │    └──────────────────────────────────────────────────────────────┘ │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## Creating Sensitivity Labels

### Example Label Hierarchy

```
RECOMMENDED LABEL TAXONOMY:
───────────────────────────

┌────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  LABEL               │ ENCRYPTION │ MARKING    │ SCOPE                    │
│  ────────────────────┼────────────┼────────────┼─────────────────────────  │
│                      │            │            │                           │
│  🟢 Public           │ None       │ None       │ Anyone can access         │
│                      │            │            │                           │
│  🔵 General          │ None       │ Footer     │ Internal only            │
│     (Default)        │            │            │ (no protection)          │
│                      │            │            │                           │
│  🟡 Confidential     │ AIP        │ Header,    │ Internal + approved      │
│     │                │ Encryption │ Footer     │ external                 │
│     │                │            │            │                           │
│     ├─ Conf\All Emp  │ Encrypt    │ Watermark  │ All employees            │
│     │                │            │            │                           │
│     ├─ Conf\HR       │ Encrypt    │ Watermark  │ HR group only            │
│     │                │            │            │                           │
│     └─ Conf\Finance  │ Encrypt    │ Watermark  │ Finance group only       │
│                      │            │            │                           │
│  🔴 Highly           │ Encrypt +  │ Watermark, │ Named users only         │
│     Confidential     │ No forward │ Header,    │ Expires in 30 days       │
│                      │ No print   │ Footer     │ No external sharing      │
│                      │            │            │                           │
└────────────────────────────────────────────────────────────────────────────┘
```

### PowerShell Configuration

```powershell
# Connect to Security & Compliance Center
Connect-IPPSSession -UserPrincipalName admin@contoso.com

# Create parent label
New-Label `
  -Name "Confidential" `
  -DisplayName "Confidential" `
  -Tooltip "Business data that could cause harm if disclosed" `
  -ContentType "File, Email"

# Create sub-label with encryption
New-Label `
  -Name "Confidential-AllEmployees" `
  -DisplayName "All Employees" `
  -ParentId (Get-Label -Identity "Confidential").Guid `
  -Tooltip "Confidential data accessible by all employees" `
  -ContentType "File, Email" `
  -EncryptionEnabled $true `
  -EncryptionProtectionType "Template" `
  -EncryptionTemplateId "All Employees" `
  -EncryptionContentExpiredOnDateInDaysOrNever "Never"

# Create sub-label for specific group
New-Label `
  -Name "Confidential-HROnly" `
  -DisplayName "HR Only" `
  -ParentId (Get-Label -Identity "Confidential").Guid `
  -Tooltip "HR confidential - restricted to HR team" `
  -ContentType "File, Email" `
  -EncryptionEnabled $true `
  -EncryptionProtectionType "UserDefined" `
  -EncryptionDoNotForward $false `
  -EncryptionEncryptOnly $false

# Add content marking
Set-Label -Identity "Confidential-HROnly" `
  -ApplyContentMarkingHeaderEnabled $true `
  -ApplyContentMarkingHeaderText "CONFIDENTIAL - HR ONLY" `
  -ApplyContentMarkingHeaderFontColor "#FF0000" `
  -ApplyContentMarkingHeaderFontSize 12 `
  -ApplyContentMarkingFooterEnabled $true `
  -ApplyContentMarkingFooterText "For HR department use only" `
  -ApplyContentMarkingWatermarkEnabled $true `
  -ApplyContentMarkingWatermarkText "HR CONFIDENTIAL"

# Publish labels (make available to users)
New-LabelPolicy `
  -Name "Global Label Policy" `
  -Labels "Public","General","Confidential","Confidential-AllEmployees","Confidential-HROnly","Highly Confidential" `
  -ExchangeLocation "All" `
  -ModernGroupLocation "All" `
  -Comment "Organization-wide sensitivity labels"

# Set default label for documents
Set-LabelPolicy -Identity "Global Label Policy" `
  -AdvancedSettings @{DefaultLabelId=(Get-Label -Identity "General").Guid}
```

---

## Auto-Labeling Policies

### Client-Side vs Service-Side

```
AUTO-LABELING COMPARISON:
─────────────────────────

┌────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  CLIENT-SIDE (Configured in Label)       SERVICE-SIDE (Auto-labeling Policy)│
│  ─────────────────────────────────       ──────────────────────────────────│
│                                                                             │
│  Where: Office apps (Word, Excel, etc.)  Where: SharePoint, OneDrive,      │
│                                                  Exchange (at rest)         │
│                                                                             │
│  When: As user works on document         When: Background scan of content  │
│                                                                             │
│  Action: Recommend or auto-apply         Action: Auto-apply only           │
│                                                                             │
│  User can: See recommendation,           User can: See applied label       │
│            accept/decline                          (already applied)        │
│                                                                             │
│  Best for: Interactive labeling          Best for: Retroactive labeling    │
│            User awareness                          Bulk classification      │
│                                                                             │
│  Licensing: E3 (recommend only)          Licensing: E5 required            │
│             E5 (auto-apply)                                                 │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

### Service-Side Auto-Labeling Policy

```
AUTO-LABELING POLICY CONFIGURATION:
───────────────────────────────────

┌────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  Policy: Auto-label PII Documents                                           │
│                                                                             │
│  Step 1: Choose content to label                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ ● Apply label to content containing sensitive info                   │   │
│  │ ○ Apply label to content shared with specific people                │   │
│  │                                                                       │   │
│  │ Sensitive Info Types:                                                │   │
│  │ ☑ Credit Card Number     (High confidence, 1+ instances)            │   │
│  │ ☑ U.S. Social Security   (High confidence, 1+ instances)            │   │
│  │ ☑ U.S. Bank Account      (High confidence, 5+ instances)            │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  Step 2: Choose locations                                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ ☑ SharePoint sites                                                   │   │
│  │   └── Include: https://contoso.sharepoint.com/sites/HR              │   │
│  │   └── Include: https://contoso.sharepoint.com/sites/Finance         │   │
│  │                                                                       │   │
│  │ ☑ OneDrive accounts                                                  │   │
│  │   └── Include: All users                                             │   │
│  │   └── Exclude: service accounts                                      │   │
│  │                                                                       │   │
│  │ ☑ Exchange email (mailboxes)                                         │   │
│  │   └── Include: All mailboxes                                         │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  Step 3: Define policy settings                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Label to apply: Confidential                                         │   │
│  │                                                                       │   │
│  │ ☑ If content matches multiple rules, apply:                         │   │
│  │   ● Highest priority label                                           │   │
│  │   ○ Last matching rule                                               │   │
│  │                                                                       │   │
│  │ ☑ Email notification to admins when label applied: weekly digest    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  Step 4: Test and turn on                                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ ● Run in simulation mode first (recommended)                        │   │
│  │ ○ Turn on policy immediately                                         │   │
│  │                                                                       │   │
│  │ Simulation results show:                                             │   │
│  │ • How many items would be labeled                                    │   │
│  │ • Sample of matched content (for validation)                        │   │
│  │ • False positive indicators                                         │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## Container Labels (Sites & Groups)

### What Are Container Labels?

```
CONTAINER LABELS:
─────────────────

Container labels apply sensitivity settings to the CONTAINER (site, team, group)
rather than individual files.

┌────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  SHAREPOINT SITE / TEAMS TEAM:                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                       │   │
│  │  Site: Project Alpha                                                  │   │
│  │  Label: 🔒 Confidential                                              │   │
│  │                                                                       │   │
│  │  Container Settings:                                                  │   │
│  │  ├── Privacy: Private (members only)                                 │   │
│  │  ├── External sharing: Disabled                                      │   │
│  │  ├── Unmanaged device access: Block                                  │   │
│  │  └── Default file label: Confidential                                │   │
│  │                                                                       │   │
│  │  Files Inside:                                                        │   │
│  │  ┌─────────────────────────────────────────────────────────────┐    │   │
│  │  │ 📄 Plan.docx      → Inherits Confidential label             │    │   │
│  │  │ 📊 Budget.xlsx    → Inherits Confidential label             │    │   │
│  │  │ 📧 Notes.docx     → Inherits Confidential label             │    │   │
│  │  └─────────────────────────────────────────────────────────────┘    │   │
│  │                                                                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  Container label controls ACCESS                                           │
│  File labels control PROTECTION (encryption, markings)                     │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

### Configure Container Labels

```powershell
# Enable sensitivity labels for containers (one-time setup)
Connect-AzureAD
$Setting = Get-AzureADDirectorySetting | Where-Object {$_.DisplayName -eq "Group.Unified"}
$Setting["EnableMIPLabels"] = "True"
Set-AzureADDirectorySetting -Id $Setting.Id -DirectorySetting $Setting

# Create label with container settings
New-Label `
  -Name "Project-Confidential" `
  -DisplayName "Project Confidential" `
  -ContentType "Site, UnifiedGroup" `
  -SiteAndGroupProtectionEnabled $true `
  -SiteAndGroupProtectionPrivacy "Private" `
  -SiteAndGroupProtectionAllowAccessToGuestUsers $false `
  -SiteAndGroupProtectionAllowEmailFromGuestUsers $false `
  -SiteAndGroupProtectionAllowFullAccess $false `
  -SiteAndGroupProtectionAllowLimitedAccess $false `
  -SiteAndGroupProtectionBlockAccess $true
```

---

## Label Examples by Complexity

### Example 1: Beginner - Basic Classification

```
LABEL: Internal
───────────────

Purpose: Mark content as internal-only (no protection, just awareness)

Configuration:
┌────────────────────────────────────────────────────────────────────────────┐
│ Name: Internal                                                              │
│ Color: Blue                                                                 │
│                                                                             │
│ Encryption: None                                                            │
│                                                                             │
│ Content Marking:                                                            │
│ • Footer: "Internal Use Only - Not for distribution outside Contoso"       │
│                                                                             │
│ Auto-labeling: None (manual only)                                          │
│                                                                             │
│ Scope: Files and emails                                                     │
│                                                                             │
│ Priority: 1 (lowest)                                                        │
└────────────────────────────────────────────────────────────────────────────┘

Use Case:
• General business documents
• Internal communications
• Non-sensitive presentations
```

### Example 2: Intermediate - Department-Specific Protection

```
LABEL: Confidential\HR
──────────────────────

Purpose: Protect HR documents, accessible only by HR team

Configuration:
┌────────────────────────────────────────────────────────────────────────────┐
│ Name: Confidential-HR                                                       │
│ Parent: Confidential                                                        │
│ Color: Yellow                                                               │
│                                                                             │
│ Encryption:                                                                 │
│ ┌─────────────────────────────────────────────────────────────────────┐   │
│ │ Enabled: Yes                                                          │   │
│ │ Assign permissions now:                                               │   │
│ │   • HR@contoso.com: Co-Owner (full control)                          │   │
│ │   • HRManagers@contoso.com: Co-Author (edit, no full control)       │   │
│ │   • Legal@contoso.com: Viewer (read only)                            │   │
│ │ Content expires: Never                                                │   │
│ │ Allow offline access: 7 days                                          │   │
│ └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│ Content Marking:                                                            │
│ • Header: "HR CONFIDENTIAL"                                                │
│ • Footer: "Authorized HR personnel only"                                   │
│ • Watermark: "HR CONFIDENTIAL" (diagonal, semi-transparent)               │
│                                                                             │
│ Auto-labeling (client-side):                                               │
│ • If document contains keywords: "salary", "performance review",          │
│   "termination", "employee investigation"                                  │
│ • Action: Recommend label (user confirms)                                  │
│                                                                             │
│ Priority: 3                                                                 │
└────────────────────────────────────────────────────────────────────────────┘
```

### Example 3: Expert - Dynamic Protection with Trainable Classifiers

```
LABEL: Highly Confidential\Board Materials
──────────────────────────────────────────

Purpose: Maximum protection for board-level documents with AI classification

Configuration:
┌────────────────────────────────────────────────────────────────────────────┐
│ Name: Highly-Confidential-Board                                             │
│ Parent: Highly Confidential                                                 │
│ Color: Red                                                                  │
│                                                                             │
│ Encryption:                                                                 │
│ ┌─────────────────────────────────────────────────────────────────────┐   │
│ │ Enabled: Yes                                                          │   │
│ │ Assign permissions now:                                               │   │
│ │   • BoardMembers@contoso.com: Co-Author                              │   │
│ │   • CEO@contoso.com: Co-Owner                                        │   │
│ │   • GeneralCounsel@contoso.com: Reviewer (view + comment)           │   │
│ │                                                                       │   │
│ │ Rights restrictions:                                                  │   │
│ │   ☐ Copy                                                              │   │
│ │   ☐ Print                                                             │   │
│ │   ☐ Extract content                                                   │   │
│ │   ☐ Forward email                                                     │   │
│ │   ☐ Reply all                                                         │   │
│ │                                                                       │   │
│ │ Content expires: 90 days after creation                              │   │
│ │ Allow offline access: 3 days                                          │   │
│ │ Double Key Encryption: Enabled (your key + Microsoft key)            │   │
│ └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│ Content Marking:                                                            │
│ • Header: "BOARD CONFIDENTIAL - DO NOT DISTRIBUTE"                        │
│ • Footer: "This document is subject to legal privilege"                   │
│ • Watermark: Dynamic (includes viewer's email)                            │
│                                                                             │
│ Auto-labeling:                                                              │
│ ┌─────────────────────────────────────────────────────────────────────┐   │
│ │ Trainable Classifier: "Board Materials" (custom trained)             │   │
│ │ Training data: 500+ board documents, meeting minutes, resolutions   │   │
│ │ Confidence threshold: 85%                                             │   │
│ │ Action: Auto-apply (no user prompt for high-confidence matches)     │   │
│ │                                                                       │   │
│ │ Additional conditions:                                                │   │
│ │ • Created by: Executive group members                                │   │
│ │ • OR contains phrases: "board resolution", "merger", "acquisition"  │   │
│ └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│ Container label (associated):                                               │
│ • Teams/SharePoint: Board Materials site                                   │
│ • Privacy: Private                                                          │
│ • Guest access: Blocked                                                     │
│ • Unmanaged devices: No access                                             │
│ • Conditional Access: Require compliant device + MFA                       │
│                                                                             │
│ Priority: 5 (highest)                                                       │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## Encryption Deep Dive

### Azure Rights Management (Azure RMS)

```
HOW AZURE RMS ENCRYPTION WORKS:
───────────────────────────────

┌────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  1. USER APPLIES LABEL                                                     │
│     ┌──────────────────────────────────────────────────────────────────┐  │
│     │  User: Creates document, applies "Confidential" label             │  │
│     │  Action: Word requests encryption from Azure RMS                  │  │
│     └──────────────────────────────────────────────────────────────────┘  │
│                                       │                                    │
│                                       ▼                                    │
│  2. AZURE RMS GENERATES KEYS                                               │
│     ┌──────────────────────────────────────────────────────────────────┐  │
│     │  Azure RMS:                                                        │  │
│     │  • Generates symmetric content key (AES-256)                      │  │
│     │  • Encrypts document with content key                             │  │
│     │  • Encrypts content key with tenant root key (RSA)               │  │
│     │  • Embeds encrypted key + policy in document                     │  │
│     └──────────────────────────────────────────────────────────────────┘  │
│                                       │                                    │
│                                       ▼                                    │
│  3. DOCUMENT IS PROTECTED                                                  │
│     ┌──────────────────────────────────────────────────────────────────┐  │
│     │                                                                    │  │
│     │  Protected Document:                                               │  │
│     │  ┌────────────────────────────────────────────────────────────┐  │  │
│     │  │  Encrypted Content (AES-256)                                │  │  │
│     │  │  ├── Document body                                          │  │  │
│     │  │  └── All embedded content                                   │  │  │
│     │  ├─────────────────────────────────────────────────────────────┤  │  │
│     │  │  Publishing License (signed by Azure RMS)                   │  │  │
│     │  │  ├── Encrypted content key                                  │  │  │
│     │  │  ├── Authorized users/groups                                │  │  │
│     │  │  ├── Rights (view, edit, print, etc.)                      │  │  │
│     │  │  └── Expiration date                                        │  │  │
│     │  └────────────────────────────────────────────────────────────┘  │  │
│     │                                                                    │  │
│     └──────────────────────────────────────────────────────────────────┘  │
│                                       │                                    │
│                                       ▼                                    │
│  4. USER OPENS DOCUMENT                                                    │
│     ┌──────────────────────────────────────────────────────────────────┐  │
│     │  Reader:                                                           │  │
│     │  • App extracts publishing license from document                  │  │
│     │  • Authenticates user with Azure AD                               │  │
│     │  • Sends license to Azure RMS with user identity                 │  │
│     │                                                                    │  │
│     │  Azure RMS:                                                        │  │
│     │  • Validates user is authorized                                   │  │
│     │  • Decrypts content key                                           │  │
│     │  • Issues use license with user's specific rights                │  │
│     │                                                                    │  │
│     │  App:                                                              │  │
│     │  • Uses content key to decrypt document                          │  │
│     │  • Enforces rights (e.g., disable print if not allowed)         │  │
│     └──────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

### Double Key Encryption (DKE)

```
DOUBLE KEY ENCRYPTION:
──────────────────────

For maximum control, use DKE - you hold one key, Microsoft holds the other.
Both keys required to decrypt content.

┌────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  STANDARD RMS:                         DOUBLE KEY ENCRYPTION:              │
│                                                                             │
│  ┌─────────────────┐                  ┌─────────────────┐                 │
│  │   Document      │                  │   Document      │                 │
│  │                 │                  │                 │                 │
│  │  Encrypted by:  │                  │  Encrypted by:  │                 │
│  │  • Microsoft    │                  │  • Microsoft    │                 │
│  │    managed key  │                  │    managed key  │                 │
│  │                 │                  │  • YOUR key     │                 │
│  └─────────────────┘                  │    (your infra) │                 │
│                                       └─────────────────┘                 │
│                                                                             │
│  Microsoft CAN decrypt                Microsoft CANNOT decrypt              │
│  (with proper authorization)          (without your key server)            │
│                                                                             │
│  Use cases for DKE:                                                        │
│  • Regulatory requirements (government, defense)                           │
│  • Zero-trust for cloud provider                                           │
│  • Most sensitive IP (trade secrets, M&A)                                 │
│                                                                             │
│  Limitations:                                                               │
│  • Office desktop apps only (no web/mobile)                               │
│  • No co-authoring                                                         │
│  • You manage key infrastructure                                           │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## Monitoring and Reports

### Label Analytics

```
LABEL ANALYTICS DASHBOARD:
──────────────────────────

┌────────────────────────────────────────────────────────────────────────────┐
│  SENSITIVITY LABEL USAGE - Last 30 Days                                     │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  LABEL DISTRIBUTION:                                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Public              │████                            │ 12%           │   │
│  │ General             │████████████████████████████████│ 55%           │   │
│  │ Confidential        │████████████████                │ 25%           │   │
│  │ Highly Confidential │████                            │  8%           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  LABELING METHOD:                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Manual by user          │██████████████████████████████│ 65%        │   │
│  │ Recommended & accepted  │████████████                  │ 20%        │   │
│  │ Auto-applied (client)   │████████                      │ 10%        │   │
│  │ Auto-applied (service)  │████                          │  5%        │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  LABEL CHANGES (Downgrades requiring justification):                       │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ From             │ To              │ Count │ Top Justification       │   │
│  │ Highly Conf      │ Confidential    │ 45    │ "Project completed"     │   │
│  │ Confidential     │ General         │ 123   │ "Approved for release"  │   │
│  │ Highly Conf      │ General         │ 12    │ "Manager approved"      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  TOP UNLABELED CONTENT:                                                     │
│  • SharePoint: Marketing site - 2,345 files                                │
│  • OneDrive: john@contoso.com - 567 files                                  │
│  • Exchange: 12,456 emails without labels                                  │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

### PowerShell Reports

```powershell
# Get label usage summary
Get-LabelActivity -StartDate (Get-Date).AddDays(-30) -EndDate (Get-Date) |
  Group-Object LabelName |
  Select-Object Name, Count |
  Sort-Object Count -Descending

# Export labeled files report
Search-UnifiedAuditLog `
  -StartDate (Get-Date).AddDays(-7) `
  -EndDate (Get-Date) `
  -RecordType SensitivityLabelAction |
  Select-Object CreationDate, UserIds, Operations, AuditData |
  Export-Csv "LabelActivity.csv"

# Find content with specific label
$searchName = "Highly-Confidential-Search"
New-ComplianceSearch -Name $searchName `
  -ExchangeLocation All `
  -SharePointLocation All `
  -ContentMatchQuery "SensitivityLabel:HighlyConfidential"

Start-ComplianceSearch -Identity $searchName

# Get search results
Get-ComplianceSearch -Identity $searchName | FL
```

---

## Best Practices

```
INFORMATION PROTECTION BEST PRACTICES:
──────────────────────────────────────

1. START SIMPLE
   ┌─────────────────────────────────────────────────────────────────────┐
   │ Begin with 3-5 labels maximum                                        │
   │ Add complexity only when needed                                      │
   │ Users won't adopt complex taxonomies                                │
   └─────────────────────────────────────────────────────────────────────┘

2. SET A DEFAULT LABEL
   ┌─────────────────────────────────────────────────────────────────────┐
   │ Default: "General" or "Internal"                                    │
   │ Ensures all new content is classified                               │
   │ Users can upgrade classification as needed                          │
   └─────────────────────────────────────────────────────────────────────┘

3. REQUIRE JUSTIFICATION FOR DOWNGRADES
   ┌─────────────────────────────────────────────────────────────────────┐
   │ If user changes from "Confidential" to "Public"                     │
   │ Require explanation (audit trail)                                   │
   │ Prevents accidental declassification                                │
   └─────────────────────────────────────────────────────────────────────┘

4. USE VISUAL MARKINGS WISELY
   ┌─────────────────────────────────────────────────────────────────────┐
   │ Watermarks for highly sensitive only (can be intrusive)            │
   │ Headers/footers for all protected content                           │
   │ Keep markings professional and readable                             │
   └─────────────────────────────────────────────────────────────────────┘

5. TEST EXTENSIVELY BEFORE ENFORCEMENT
   ┌─────────────────────────────────────────────────────────────────────┐
   │ Week 1-2: Labels available, no policy                               │
   │ Week 3-4: Default label, monitor adoption                          │
   │ Week 5-6: Auto-labeling in simulation                               │
   │ Week 7+:  Gradual enforcement, department by department            │
   └─────────────────────────────────────────────────────────────────────┘

6. TRAIN YOUR USERS
   ┌─────────────────────────────────────────────────────────────────────┐
   │ Explain WHY labels matter                                           │
   │ Show HOW to apply labels                                            │
   │ Clarify WHAT each label means                                       │
   │ Regular refresher training                                          │
   └─────────────────────────────────────────────────────────────────────┘
```

---

*Next: [Case Studies](case-studies.md)* | *Back to [DLP Policies](02-dlp-policies.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [APACHE 2.0 License](../LICENSE)*
