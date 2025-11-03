# SkillsCenter LMS (Power Platform) — README

A solution‑aware Power Apps + Power Automate implementation for the SkillsCenter LMS. This repository stores the **unpacked solution source** (for version control) and uses CI/CD (optional) to build/import managed packages.

---

## Contents
- [Overview](#overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [SharePoint Provisioning](#sharepoint-provisioning)
- [Environment Variables](#environment-variables)
- [Connection References](#connection-references)
- [Importing the Solution](#importing-the-solution)
- [Post‑Import Checklist](#post-import-checklist)
- [Smoke Tests](#smoke-tests)
- [Local Build (Pack) & Deploy](#local-build-pack--deploy)
- [GitHub Actions (Optional)](#github-actions-optional)
- [Troubleshooting](#troubleshooting)

---

## Overview
The SkillsCenter solution tracks **module submissions**, **grading**, **student summaries**, **certificates**, **milestones**, and **enrollment sync**. All apps/flows are packaged as a Power Platform **Solution**; SharePoint lists/libraries act as the data layer.

**One source of truth**: store only the **unpacked** solution under `/src`. Build installable `.zip` files at release time.

---

## Architecture
- **Canvas Apps**: Student App, Proctor/Grader App
- **Flows**: grading, archive + certificate generation, student summary recalculation, enrollment sync, milestone logic, etc.
- **SharePoint**: lists for configuration and data; libraries for files
- **Environment Variables**: site URLs, list/library names, document paths
- **Connection References**: SharePoint, Teams, Forms, OneDrive, Excel

---

## Prerequisites
- Power Platform environment access (Dev/Test/Prod)
- SharePoint site Owner on the target site (e.g., **CU Boulder Skills Center – Proctors**)
- Power Platform CLI (for local pack):
  ```bash
  dotnet tool update --global Microsoft.PowerApps.CLI.Tool
  ```

---

## SharePoint Provisioning
Create these **lists** and **libraries** first. Display names can match below; ensure the **URL segment** matches what you set in Environment Variables.

### Lists
1. **Modules**
   - *ModuleID* (Single line, unique optional)
   - *ModuleName* (Single line)
   - *ModuleHours* (Number)
   - *Active* (Choice: Yes/No)

2. **Grades**
   - *StudentID* (Person **or** Text)
   - *Module* (Lookup → Modules: ModuleID; include **ModuleHours** as extra field)
   - *Status* (Choice: Submitted / In Review / Completed / Rejected / Accepted)
   - *Attempts* (Number)
   - *RawScore* (Number)
   - *CalculatedGrade* (Number/Text)
   - *ProctorComments* (Multiple lines, plain)
   - *FirstSubmissionDate*, *RecentSubmissionDate* (Date/Time)
   - *LatestSubmissionLink* (Hyperlink)
   - *StudentModuleKey* (Single line, **Unique values = Yes**)

3. **StudentSummary** (1 row per student)
   - *StudentID* (Person or Text)
   - *CurrentSemester* (Text), *CreditHoursCurrent* (Number)
   - *ExpectedHoursCurrent*, *StartingRolloverHours*, *HoursThisSemester*, *TotalModuleHoursCurrent* (Number)
   - *ModulePctCurrent*, *ParticipationPctCurrent*, *ExtraCreditHoursCurrent*, *ExtraCreditPctCurrent*, *FinalPctCurrent* (Number)
   - *NextRequiredModules* (Number), *NextDueDate* (Date/Time)
   - *CompletedModulesJSON*, *InProgressModulesJSON* (Multiple lines, plain)
   - *Status* (Choice: Active / Closed / Dropped)

4. **Admin** (single row: Title = "Default")
   - *Milestone1Date … Milestone7Date* (Date/Time; leave blanks if unused)
   - *(optional)* *Milestone1Req … Milestone7Req* (Number)

5. **MMTList** (if used)
   - *StudentID* (Person/Text), *ModuleID* (Text), *Notes* (Multiple lines)

6. **ModuleLinks** (optional)
   - *LinkType* (Choice: Guide / Rubric / Template), *Url* (Hyperlink)

7. **TemplateList** (optional)
   - *Path* (Text) e.g., `/Certifications/Certificate.docx`

### Libraries
1. **Student Records** *(aka Module Submissions)* — Document Library
   - Columns: *StudentID* (Person/Text), *ModuleID* (Lookup or Text), *AttemptNumber* (Number), *SubmissionDate* (Date/Time), *DocumentType* (Choice: Submission / ArchiveSnapshot / Certificate), *Semester* (Text)
   - Turn **Versioning** on

2. **Certifications** — Document Library
   - Store generated PDFs; versioning recommended

> Place the **certificate Word template** at the path you will set in EVs (e.g., `/Certifications/Certificate.docx`).

---

## Environment Variables
Set these **during solution import** (or after, in Solution → Environment Variables):

| EV Name | Example Value | Notes |
|---|---|---|
| **CertificateFile** | `/Certifications/Certificate.docx` | Template path inside the site |
| **CertificatePath** | `/Certifications` | Library (URL segment) |
| **StudentRecordPath** | `/Student Records` *(or `/Module Submissions`)* | Library (URL segment; match actual URL) |
| **SyncTable** | `/Documents/ActiveStudents.xlsx` | Excel with table `Enrollments` |
| **CU Boulder Skills Center – Proctors** | *(Site connection reference)* | Pick your site connection |
| **SkillsCenter** | *(Site connection reference)* | Site-level CR if used |
| **StudentSummary** | `StudentSummary` | List name bound via CR |
| **Grades** | `Grades` | List name bound via CR |
| **Modules** | `Modules` | List name bound via CR |
| **MMTList** | `MMTList` | If used |
| **ModuleLinks** | `ModuleLinks` | If used |
| **TemplateList** | `TemplateList` | If used |
| **Admin** | `Admin` | Settings list |
| **Certifications** | `Certifications` | Library name bound via CR |

> **URL segment vs Display Name:** verify the library/list URL in **Library/List Settings → URL**. Use that in EVs (spaces often appear as `%20`).

---

## Connection References
Map these to connections in the **target** environment (prefer a service account):
- **SharePoint** (site-level)
- **Microsoft Teams**
- **Microsoft Forms**
- **Excel Online (Business)**
- **OneDrive for Business**

All flows/apps reference these CRs—no personal connections should remain in Prod.

---

## Importing the Solution
1. Go to **make.powerapps.com → Solutions → Import** and select the solution `.zip`.
2. **Connections**: sign in / choose existing; wait for green checks.
3. **Environment Variables**: set the values per table above.
4. Click **Import**.

---

## Post‑Import Checklist
- **Environment Variables**: confirm values are correct.
- **Connection References**: all green, bound to target connections.
- **Cloud Flows**: set to **On**. Open each → **Edit → Save** to rebind to EVs/CRs.
- **Canvas Apps**: open, confirm data sources resolve, **Save** and **Publish**.
- **Permissions**: service account = Contribute on `Student Records` & `Certifications`; proctors on data lists; students read per app/row‑level rules.

---

## Smoke Tests
1. **Modules**: create a module with `ModuleHours`.
2. **StudentSummary**: seed a student row (or run the enrollment sync to create it).
3. **Grades**: create a Completed grade linked to the module; ensure recalculation updates StudentSummary.
4. **File flow**: upload a submission DOCX to **Student Records**; verify archive/certificate flows and metadata updates.
5. **Milestones**: set a future date in **Admin**; check Student App displays “You should have X modules by DATE.”

---

## Local Build (Pack) & Deploy
From repo root (where `/src` exists):
```bash
# Pack unmanaged (Dev)
pac solution pack \
  --folder ./src \
  --zipfile ./out/SkillsCenterLMS_unmanaged.zip \
  --managed false

# Pack managed (Test/Prod)
pac solution pack \
  --folder ./src \
  --zipfile ./out/SkillsCenterLMS_managed.zip \
  --managed true

# Import to an environment
pac auth create --url https://YOUR-ENV.crm.dynamics.com --applicationId <APP_ID> --clientSecret <SECRET> --tenant <TENANT>
pac auth select --name Default
pac solution import --path ./out/SkillsCenterLMS_managed.zip --activate-plugins true --overwrite true
```

---

## GitHub Actions (Optional)
- **export-solution.yml**: export+unpack from Dev on push
- **deploy-solution.yml**: pack and import to Test/Prod on dispatch
- Store creds in **GitHub Secrets**; never commit .zip artifacts—attach them to **Releases**.

---

## Troubleshooting
- **List/Library not in dropdown during import** → wrong site CR or the list/library wasn’t created yet. Fix the site CR, refresh EVs.
- **404/Path errors** in flows → EV path must match the library **URL segment** (e.g., `/Student%20Records`).
- **Flows Off after import** → turn On, open, Edit→Save to rebind.
- **Excel sync fails** → workbook must have table `Enrollments` with exact headers (`StudentEmail, StudentName, Class, Semester, CreditHours`).
- **Permissions issues** → flows use the connection’s identity; ensure that account has the required SharePoint rights.

---

## Conventions
- **One row per student** in StudentSummary; archive prior terms in a history list if needed.
- **Rollover** stores hours as a number; do not move past‑term files.
- **Milestones** are read from Admin; blanks are ignored.

---

**Maintainers:** Update this README when adding/removing lists, libraries, EVs, or CRs. Keep `/src` as the canonical source; generate .zip packages via CI or local `pac` commands.

