---
name: generating-permission-set
description: "Generates deployable Salesforce permission set metadata (PermissionSet XML) AND analyzes/audits permission access. TRIGGER when: creating or editing permission set metadata, object permissions, field-level security (FLS), tab/app visibility, agent access, or .permissionset-meta.xml / .permissionsetgroup-meta.xml files; OR when asked \"who has access to X?\", to view a permission hierarchy (PSG to PS), analyze what a user has, or export permission configuration. DO NOT TRIGGER when: defining the underlying object (use generating-custom-object), defining fields (use generating-custom-field), deploying metadata (use deploying-metadata), or Apex-managed sharing logic (use generating-apex)."
license: MIT
compatibility: Salesforce Metadata API v60.0+
metadata:
  author: "Jag Valaiyapathy"
  version: "1.2"
  inspiration: "PSLab by Oumaima Arbani (github.com/OumArbani/PSLab)"
---

## When to Use This Skill

This skill has two capabilities:

1. **Generation** - author or edit permission set metadata (object permissions, FLS, user permissions, tab/app visibility, Apex/VF access, record types, agent access).
2. **Analysis and auditing** - answer "who has access to X?", view PSG-to-PS hierarchies, analyze a user's permissions, and export permission configuration.

Use it whenever the work is granting access through permission sets OR investigating existing access. For object/field/validation metadata definitions, use [generating-custom-object](../generating-custom-object/SKILL.md), [generating-custom-field](../generating-custom-field/SKILL.md), and [generating-validation-rule](../generating-validation-rule/SKILL.md). For rollout, use [deploying-metadata](../deploying-metadata/SKILL.md).

---

# Part A: Generating Permission Set Metadata

## Step 1: Define Core Properties

Start by defining the required permission set properties:

```xml
<PermissionSet xmlns="http://soap.sforce.com/2006/04/metadata">
    <fullName>YourPermissionSetName</fullName>
    <label>Display Name for Administrators</label>
    <description>Clear description of purpose and intended audience</description>
</PermissionSet>
```

**Naming conventions:**
- Use descriptive API names (e.g., `Sales_Manager_Access`)

## Step 2: Configure Object Permissions

Add CRUD permissions for standard and custom objects:

```xml
<objectPermissions>
    <allowCreate>true</allowCreate>
    <allowRead>true</allowRead>
    <allowEdit>true</allowEdit>
    <allowDelete>false</allowDelete>
    <modifyAllRecords>false</modifyAllRecords>
    <viewAllRecords>false</viewAllRecords>
    <viewAllFields>false</viewAllFields>
    <object>Account</object>
</objectPermissions>
```

## Step 3: Set Field-Level Security

Define field permissions for sensitive or custom fields:

```xml
<fieldPermissions>
    <editable>true</editable>
    <readable>true</readable>
    <field>Account.SSN__c</field>
</fieldPermissions>
```

**Important:**
- Required fields must NEVER appear in list of field permissions. Granting field-level security on required fields is not allowed by the platform and will cause deployment failure. 
- Before adding any field, confirm from the object metadata that the field exists and is not required
- A field is required when its metadata contains `<required>true</required>`:
- Formula fields cannot be editable
- Master-detail fields are required fields on the child (detail) object

```xml
<fields>
    <fullName>FieldName__c</fullName>
    <required>true</required>
</fields>
```
- Use format `ObjectName.FieldName` for field references
- Set both readable and editable to true when the user needs edit access; editable implies readable
- If all fields should be visible, can alternatively enable the "viewAllFields" object permission

## Step 4: Grant User Permissions

Add system-level permissions for features and capabilities:

```xml
<userPermissions>
    <enabled>true</enabled>
    <name>ApiEnabled</name>
</userPermissions>
<userPermissions>
    <enabled>true</enabled>
    <name>RunReports</name>
</userPermissions>
```

**Common permissions:**
- `ApiEnabled`: API access
- `ViewSetup`: View Setup menu
- `ManageUsers`: User management
- `RunReports`: Report execution

**Security review required for:**
- `ViewAllData`: Read all records
- `ModifyAllData`: Edit all records
- `ManageUsers`: User administration

## Step 5: Configure App and Tab Visibility

Make applications and tabs visible to users:

```xml
<applicationVisibilities>
    <application>Sales_Console</application>
    <visible>true</visible>
</applicationVisibilities>
<tabSettings>
    <tab>CustomTab__c</tab>
    <visibility>Visible</visibility>
</tabSettings>
```

**Application visibility options:**
- <visible> can be true or false

**Tab visibility options:**
- `Visible`: The tab is available on the All Tabs page and appears in the visible tabs for its associated app. Can be customized.
- `Available`: The tab is available on the All Tabs page. Individual users can customize their display to make the tab visible in any app
- `None`: Not visible

**CRITICAL - Tab Naming:**
- Custom object tabs: MUST include the __c suffix (e.g., MyCustomObject__c)
- Standard object tabs: Use the object name with "standard-" prefix (e.g., standard-Account, standard-Contact)
- The tab name matches the object's API name exactly

## Step 6: Add Apex and Visualforce Access (Optional)

Grant access to custom code:

```xml
<classAccesses>
    <apexClass>CustomController</apexClass>
    <enabled>true</enabled>
</classAccesses>
<pageAccesses>
    <apexPage>CustomPage</apexPage>
    <enabled>true</enabled>
</pageAccesses>
```

## Step 7: Set License and Record Type Settings (Optional)

Specify license requirements and record type visibility:

```xml
<license>Salesforce</license>
<hasActivationRequired>false</hasActivationRequired>
<recordTypeVisibilities>
    <recordType>Account.Business</recordType>
    <visible>true</visible>
    <default>true</default>
</recordTypeVisibilities>
```
## Step 8: Set Agent Access (Optional)
                                              
Enable access to Agentforce Employee Agents for users assigned to this permission set:

<agentAccesses>
    <agentName>Sales_Assistant_Agent</agentName>
    <enabled>true</enabled>
</agentAccesses>

Field requirements:
- agentName (Required): The developer name of the employee agent
- enabled (Required): Set to true to grant access, false to deny

Important:
- Agent names must match existing Agentforce Employee Agent developer names

## Validation Checklist

Before deploying, verify:
- [ ] fullName, label, description set
- [ ] Permissions follow least privilege
- [ ] No required fields in `<fieldPermissions>`
- [ ] No duplicate permissions
- [ ] No lengthy comments

## What Causes Deployment Failure

- **Field permissions on required fields:** Any required field in `<fieldPermissions>` fails deployment. Required fields cannot have FLS; omit them entirely. Always confirm from object/field metadata that a field exists and is not required - never assume.
- **Incorrect API names:** Using the wrong name or missing suffixes (e.g. missing `__c` for custom objects, fields, tabs) cause failure.

## Deployment

Deploy using Salesforce CLI. Hand rollout to [deploying-metadata](../deploying-metadata/SKILL.md) when the user needs validation, scratch-org, or CI/CD support.

---

# Part B: Analyzing and Auditing Permission Access

Use this part when the question is about existing access rather than authoring new metadata: "who has access to X?", hierarchy visualization, user permission analysis, or exporting a permission set.

## Capabilities

| Request shape | Capability | Script entry point |
|---|---|---|
| "who has access to X?" (object, field, Apex, VF, flow, custom permission, system permission) | permission detector | `scripts/cli.py detect ...` |
| "what does this user have?" | user analyzer | `scripts/cli.py user <username>` |
| "show me the hierarchy" (PSG to PS) | hierarchy viewer | `scripts/cli.py hierarchy` |
| "export this permset" | exporter (CSV/JSON) | `scripts/cli.py export <name>` |

## Setup and Run

```bash
python3 -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install -r requirements.txt
sf org login web --set-default     # ensure sf auth before analysis
```

```bash
# View org permission hierarchy (ASCII tree or Mermaid)
python scripts/cli.py hierarchy
python scripts/cli.py hierarchy --format mermaid > hierarchy.md

# Detect who has specific access
python scripts/cli.py detect object Account --access delete
python scripts/cli.py detect field Account.AnnualRevenue --access edit
python scripts/cli.py detect apex MyApexClass
python scripts/cli.py detect custom Can_Approve_Expenses

# Analyze a single user's permissions
python scripts/cli.py user john.smith@company.com

# Export a permission set
python scripts/cli.py export Sales_Manager --output /tmp/sales_manager.csv
```

## High-Signal Analysis Rules

- Distinguish DIRECT Permission Set grants from grants inherited via a Permission Set Group.
- Be explicit about whether access is object-level, field-level, class-level, flow-level, or custom-permission-based.
- Use the Tooling API (handled by `scripts/tooling_api.py`) where required for setup entities and advanced visibility questions.
- For agent access questions, verify exact agent-name matching in the permission metadata.
- Prefer the narrowest useful query over broad org-wide scans unless the user explicitly wants a full audit.
- Render with ASCII trees/tables for terminal work; use Mermaid only when documentation benefit is clear.

## Analysis Output Format

Report in this order:
1. What was analyzed
2. Org / subject scope
3. Which permissions grant access
4. Whether access is direct or inherited (via a group)
5. Recommended follow-up

---

## Cross-Skill Integration

| Need | Delegate to | Reason |
|---|---|---|
| Define the object access is granted on | [generating-custom-object](../generating-custom-object/SKILL.md) | metadata authoring |
| Define fields needing FLS | [generating-custom-field](../generating-custom-field/SKILL.md) | field-type and FLS eligibility |
| Deploy permission changes | [deploying-metadata](../deploying-metadata/SKILL.md) | rollout |
| Identify Apex classes needing grants | [generating-apex](../generating-apex/SKILL.md) | implementation context |
| Bulk user assignment analysis | [handling-sf-data](../handling-sf-data/SKILL.md) | larger data operations |

---

## Reference File Index

### Generation

| File | Use For |
|------|---------|
| [references/fls-best-practices.md](references/fls-best-practices.md) | Field-level security design: eligibility, required/formula/master-detail exclusions |
| [references/permset-auto-generation.md](references/permset-auto-generation.md) | Default permission-set follow-up when new objects/fields are created |
| [references/profile-permission-guide.md](references/profile-permission-guide.md) | When to use profiles vs permission sets |
| [references/permission-set-example.md](references/permission-set-example.md) | Worked example of a complete permission set (Invoice Manager) |
| [references/agent-access-guide.md](references/agent-access-guide.md) | Agentforce agent access permissions and visibility troubleshooting |
| [assets/permission-sets/permission-set.xml](assets/permission-sets/permission-set.xml) | Copy-ready `.permissionset-meta.xml` template |
| [assets/profiles/profile.xml](assets/profiles/profile.xml) | Profile template for profile-based access |
| [hooks/scripts/generate_permission_set.py](hooks/scripts/generate_permission_set.py) | Auto-generate a permission set from object/field metadata |

### Analysis and Auditing

| File | Use For |
|------|---------|
| [references/permission-model.md](references/permission-model.md) | How permissions resolve across permission sets and groups |
| [references/soql-reference.md](references/soql-reference.md) | SOQL queries used for permission detection and user analysis |
| [references/workflow-examples.md](references/workflow-examples.md) | Common audit workflows (for example, "who can delete Accounts?") |
| [references/usage-examples.md](references/usage-examples.md) | Real-world CLI usage examples |
| [scripts/cli.py](scripts/cli.py) | Main CLI: hierarchy, detect, user, export |
| [scripts/hierarchy_viewer.py](scripts/hierarchy_viewer.py) | PSG to PS hierarchy data |
| [scripts/permission_detector.py](scripts/permission_detector.py) | "Who has access to X?" detection logic |
| [scripts/user_analyzer.py](scripts/user_analyzer.py) | Per-user permission analysis |
| [scripts/permission_exporter.py](scripts/permission_exporter.py) | CSV/JSON export of permission sets |
| [scripts/tooling_api.py](scripts/tooling_api.py) | Tooling API access for setup entities |
| [scripts/auth.py](scripts/auth.py) | `sf` connection bootstrap |
| [scripts/renderers/ascii_tree.py](scripts/renderers/ascii_tree.py) | ASCII tree/table rendering |
| [scripts/renderers/mermaid.py](scripts/renderers/mermaid.py) | Mermaid diagram rendering |
| [requirements.txt](requirements.txt) | Python dependencies (simple-salesforce, rich) |

---

## Score Guide

| Score | Meaning |
|---|---|
| 90+ | strong permission work with clear access sourcing or deployable metadata |
| 75-89 | useful result with minor gaps |
| 60-74 | partial visibility or incomplete metadata |
| < 60 | insufficient; expand analysis or correct metadata before deploy |