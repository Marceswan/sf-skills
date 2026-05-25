---
name: generating-custom-object
description: "Use this skill when users need to create, generate, or validate Salesforce Custom Object metadata. TRIGGER when users mention custom objects, creating objects, object metadata, .object-meta.xml files, sharing models, name fields, record types, page layouts, or troubleshoot object deployment errors (sharing models, Master-Detail relationships). This is the metadata hub: it orchestrates the broader metadata family. DO NOT TRIGGER when work is isolated to: a single custom field (use generating-custom-field), a validation rule (use generating-validation-rule), permission-set/FLS generation (use generating-permission-set), permission access auditing (use generating-permission-set), or deploying metadata (use deploying-metadata)."
license: MIT
metadata:
  version: "1.2"
  author: "Jag Valaiyapathy"
  scoring: "120 points across 6 categories"
---

## When to Use This Skill

Use this skill when you need to:
- Create new custom objects
- Generate custom object metadata XML
- Configure object sharing and security settings
- Set up object features and capabilities
- Set up record types or page layouts for an object
- Troubleshoot deployment errors related to custom objects

This skill is the **metadata hub** for the granular generation family. Use the per-type skills for focused work:

| Need | Use |
|------|-----|
| A single custom field | [generating-custom-field](../generating-custom-field/SKILL.md) |
| A validation rule | [generating-validation-rule](../generating-validation-rule/SKILL.md) |
| Permission set / FLS / access audit | [generating-permission-set](../generating-permission-set/SKILL.md) |
| Custom tab | [generating-custom-tab](../generating-custom-tab/SKILL.md) |
| List view | [generating-list-view](../generating-list-view/SKILL.md) |
| FlexiPage / Lightning record page | [generating-flexipage](../generating-flexipage/SKILL.md) |
| Custom application | [generating-custom-application](../generating-custom-application/SKILL.md) |

## Required Context to Gather First

Ask for or infer before generating:
- the object's business intent and whether it is user-facing or system-facing
- whether it participates in a Master-Detail relationship (drives the sharing model)
- the name field strategy (Text vs AutoNumber)
- whether new custom objects or fields should also include **permission-set / FLS generation** (assume yes unless the user opts out)
- target package directory and, if querying existing schema, the org alias

## Specification

## 1. Overview and Purpose

This document defines the mandatory constraints for generating CustomObject metadata XML (`.object-meta.xml` file). The agent must verify these constraints before outputting XML to prevent Metadata API deployment errors.

**File extension:** `.object-meta.xml`

---

## 2. Syntactic Essentials (Tier 1)

The following constraints must be true for the XML body to deploy successfully.

**Note:** The API Name (fullName) is NOT a tag; it is the filename (e.g., `Vehicle__c.object-meta.xml`).

### Required Elements

| Element | Requirement | Notes |
|---------|-------------|-------|
| `<label>` | Required | Singular UI name |
| `<pluralLabel>` | Required | Plural UI name |
| `<sharingModel>` | Required | See Sharing Model Rules below |
| `<deploymentStatus>` | Required | Always set to `Deployed` |
| `<nameField>` | Required | Primary record identifier (requires `<label>` and `<type>`) |
| `<visibility>` | Required | Always set to `Public` |

### Sharing Model Rules

**Default:** Set `<sharingModel>` to `ReadWrite`.

**Exception:** If this object contains a Master-Detail relationship field, `<sharingModel>` MUST be `ControlledByParent`.

**Decision Logic:**
- IF object has NO Master-Detail field → use `ReadWrite`
- IF object has Master-Detail field → use `ControlledByParent`
- IF a Master-Detail field is being added to an existing child object → that existing object's `<sharingModel>` must also be updated to `ControlledByParent`

**❌ INCORRECT** - Will cause error: `Cannot set sharingModel to ReadWrite on a CustomObject with a MasterDetail relationship field`
```xml
<CustomObject xmlns="http://soap.sforce.com/2006/04/metadata">
  <label>Order Line Item</label>
  <pluralLabel>Order Line Items</pluralLabel>
  <sharingModel>ReadWrite</sharingModel>  <!-- WRONG: Object has a M-D field -->
  <deploymentStatus>Deployed</deploymentStatus>
</CustomObject>
```

**✅ CORRECT:**
```xml
<CustomObject xmlns="http://soap.sforce.com/2006/04/metadata">
  <label>Order Line Item</label>
  <pluralLabel>Order Line Items</pluralLabel>
  <sharingModel>ControlledByParent</sharingModel>  <!-- CORRECT -->
  <deploymentStatus>Deployed</deploymentStatus>
</CustomObject>
```

---

## 3. Smart Defaults & Decision Logic (Tier 2)

The agent must choose which features to enable based on the object's intended use case.

### A. The Name Field Decision

| Type | When to Use | Additional Requirements |
|------|-------------|------------------------|
| **Text** | Default for human-named entities (Projects, Locations, Teams) | None |
| **AutoNumber** | Use for transactions, logs, or IDs (Invoices, Requests, Tickets) | Must include `<displayFormat>` (e.g., `INV-{0000}`) and `<startingNumber>1</startingNumber>` |

**Text Name Field Example:**
```xml
<nameField>
  <label>Project Name</label>
  <type>Text</type>
</nameField>
```

**AutoNumber Name Field Example:**
```xml
<nameField>
  <label>Invoice Number</label>
  <type>AutoNumber</type>
  <displayFormat>INV-{0000}</displayFormat>
  <startingNumber>1</startingNumber>
</nameField>
```

### B. Object Description

**`<description>`**: Mandatory. Every object must contain a professional summary.

If the intent is vague, generate a summary:
> "Object used to track and manage [Intent] within the organization."
### C. Junction Object Naming

If the object is a many-to-many link between two parents, name the object by combining the two parent entities to ensure the schema remains intuitive.

**Examples:**
- `Position_Candidate__c` (links Position and Candidate)
- `Job_Application__c` (links Job and Application)

### D. Feature Enablement (Clean XML)

To maintain "Clean XML," only include optional tags when deviating from the Salesforce platform default of `false`.

**Scenario A: User-Facing Objects (Apps, Trackers, Business Entities)**
- Trigger: The object is intended for direct user interaction
- Action: Set `<enableSearch>`, `<enableReports>`, `<enableActivities>`, and `<enableHistory>` to `true`

**Scenario B: System-Facing Objects (Junctions, Background Logs)**
- Trigger: The object exists for technical associations or background data
- Action: Omit these tags to keep the UI clean and the XML lean

---

## 4. Critical Constraints & Common Failures

### Reserved Words

Never use reserved words as API names for Custom Objects or Custom Fields:

| Category | Reserved Words (Do Not Use as API Names) |
|----------|------------------------------------------|
| SOQL/SQL | `Select`, `From`, `Where`, `Limit`, `Order`, `Group` |
| System | `User`, `External`, `View`, `Type` |
| Temporal | `Date`, `Number` |

### Relationship Cap

Do not create more than **2 Master-Detail relationships** for a single object. If a third relationship is required, use a Lookup instead.

### XML Root Element

Do NOT include the `<fullName>` tag at the root of the `.object-meta.xml` file. The API name is derived from the filename.

**❌ INCORRECT:**
```xml
<CustomObject xmlns="http://soap.sforce.com/2006/04/metadata">
  <fullName>Vehicle__c</fullName>  <!-- WRONG: Remove this -->
  <label>Vehicle</label>
</CustomObject>
```

**✅ CORRECT:**
```xml
<CustomObject xmlns="http://soap.sforce.com/2006/04/metadata">
  <label>Vehicle</label>
  <!-- fullName comes from filename: Vehicle__c.object-meta.xml -->
</CustomObject>
```

### Validation Rule Naming Convention

Validation rule names follow different conventions than custom fields.

**Rules:**
- Must contain only alphanumeric characters and underscores
- Must begin with a letter
- Cannot end with an underscore
- Cannot contain two consecutive underscores
- **Must NOT end with `__c`** (unlike custom fields)

**❌ INCORRECT:**
```xml
<validationRules>
  <fullName>Require_Start_Date__c</fullName>  <!-- WRONG: Has __c suffix -->
  <active>true</active>
  <errorMessage>Start Date is required.</errorMessage>
  <formula>ISBLANK(Start_Date__c)</formula>
</validationRules>
```
**Error:** `The validation name can only contain alphanumeric characters, must begin with a letter, cannot end with an underscore...`

**✅ CORRECT:**
```xml
<validationRules>
  <fullName>Require_Start_Date</fullName>  <!-- CORRECT: No __c suffix -->
  <active>true</active>
  <errorMessage>Start Date is required.</errorMessage>
  <formula>ISBLANK(Start_Date__c)</formula>
</validationRules>
```

**Naming Pattern Reference:**

| Metadata Type | Naming Pattern | Example |
|---------------|----------------|---------|
| Custom Fields | Ends with `__c` | `Start_Date__c` |
| Validation Rules | No suffix | `Require_Start_Date` |
| Custom Objects | Ends with `__c` | `Vehicle__c` |

---

## 5. Verification Checklist

Before generating the Custom Object XML, verify:

### Syntactic Checks
- [ ] Are both `<label>` and `<pluralLabel>` present?
- [ ] Is `<deploymentStatus>` set to `Deployed`?
- [ ] Is `<visibility>` set to `Public`?
- [ ] Does `<nameField>` include both `<label>` and `<type>`?
- [ ] If `<type>` is `AutoNumber`, are `<displayFormat>` and `<startingNumber>` included?

### Sharing Model Check (Critical)
- [ ] Does this object have a Master-Detail relationship field?
    - If YES → `<sharingModel>` MUST be `ControlledByParent`
    - If NO → `<sharingModel>` should be `ReadWrite`

### Constraint Checks
- [ ] Is the API name free of reserved words?
- [ ] Are there 2 or fewer Master-Detail relationships?
- [ ] Is `<fullName>` absent from the XML root?

### Validation Rule Checks (if applicable)
- [ ] Do validation rule names NOT end with `__c`?
- [ ] Do validation rule names follow alphanumeric + underscore pattern?

### Architectural Checks
- [ ] Is `<description>` present with a meaningful summary?
- [ ] Are `<enableSearch>` and `<enableReports>` set to `true` if user-facing?
- [ ] Does the filename match the intended API name?

---

## 6. Permission Impact (Default Follow-Up)

Object CRUD alone does NOT make a custom object usable or its fields visible. Field-level security is the most common hidden blocker after deployment.

- When new custom objects or fields are created, default to generating or updating a Permission Set unless the user explicitly opts out.
- Prefer permission sets over profile-centric access patterns.
- Object permissions are not field permissions: include `fieldPermissions` for eligible custom fields rather than leaving FLS manual.
- Hand permission-set generation and access auditing to [generating-permission-set](../generating-permission-set/SKILL.md).

## 7. High-Signal Rules

- Create metadata before attempting Flow or data tasks that depend on it.
- Avoid hardcoded IDs in formulas or metadata logic.
- Validation rules should have an intentional bypass strategy when operationally necessary.
- Set the sharing model deliberately: a Master-Detail child must be `ControlledByParent`.

## 8. Cross-Skill Integration

| Need | Delegate to | Reason |
|------|-------------|--------|
| Add fields to the object | [generating-custom-field](../generating-custom-field/SKILL.md) | field-type rules and FLS eligibility |
| Add validation rules | [generating-validation-rule](../generating-validation-rule/SKILL.md) | formula and naming constraints |
| Grant access / FLS / audit access | [generating-permission-set](../generating-permission-set/SKILL.md) | permission-set authoring and access analysis |
| Build Flows on the new schema | [generating-flow](../generating-flow/SKILL.md) | declarative automation |
| Build Apex on the new schema | [generating-apex](../generating-apex/SKILL.md) | code against metadata |
| Deploy the metadata | [deploying-metadata](../deploying-metadata/SKILL.md) | rollout and validation |
| Seed test data after deploy | [handling-sf-data](../handling-sf-data/SKILL.md) | test data creation |

---

## Reference File Index

| File | Use For |
|------|---------|
| [references/metadata-family-orchestration.md](references/metadata-family-orchestration.md) | How this hub sequences with field, validation, flow, permission-set, and deploy skills in a full build |
| [references/metadata-types-reference.md](references/metadata-types-reference.md) | Common metadata types, file locations, and directory structure |
| [references/naming-conventions.md](references/naming-conventions.md) | Naming standards for objects, fields, rules, and related metadata |
| [references/custom-object-example.md](references/custom-object-example.md) | Worked example of a complete custom object definition |
| [references/sf-cli-commands.md](references/sf-cli-commands.md) | `sf` CLI v2 commands for metadata operations, describe, and deploy |
| [references/best-practices-scoring.md](references/best-practices-scoring.md) | 120-point scoring rubric across structure, naming, security, and deployment |
| [assets/objects/custom-object.xml](assets/objects/custom-object.xml) | Copy-ready `.object-meta.xml` template |
| [assets/record-types/record-type.xml](assets/record-types/record-type.xml) | Record type template |
| [assets/layouts/page-layout.xml](assets/layouts/page-layout.xml) | Page layout template |
| [hooks/scripts/validate_metadata.py](hooks/scripts/validate_metadata.py) | Pre-deploy metadata validation script |
| [hooks/scripts/run_validation.sh](hooks/scripts/run_validation.sh) | Wrapper to run metadata validation |

---

## Score Guide

| Score | Meaning |
|-------|---------|
| 108+ | strong production-ready metadata |
| 96-107 | good metadata with minor review items |
| 84-95 | acceptable but validate carefully |
| < 84 | block deployment until corrected |
