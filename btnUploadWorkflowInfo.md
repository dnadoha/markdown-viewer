# SOR Map Toolbar - Per-Tool File Map

Companion to `SOR-MapToolbar-Tools.md`. For each toolbar tool this file lists every source file involved, across every deployment that takes part in that tool, with `file:line` citations.

Purpose: deployment scoping. When a tool changes, this map shows which files are involved, in which repository, and what they are connected to.

## Contents

- [How to read this file](#how-to-read-this-file)
- [Entry 1: Workflow Info - Uploaden (btnUploadWorkflowInfo)](#entry-1-workflow-info---uploaden-btnuploadworkflowinfo)
- [Entry template for future tools](#entry-template-for-future-tools)

---

## How to read this file

Every entry follows the same shape: a one-line summary, an "At a glance" table, the call chain, the files per deployment, the connections to other tools, the environment coupling, a deployment checklist and open risks.

Repository roots:

| Short name | Repository root (working copy) | Built into / deployed as |
| --- | --- | --- |
| SOR | `D:\AO00\HIT.SOR\SOR\SOR` | The SOR web application (ASP.NET WebForms host) |
| WFM | `D:\AO00\Hit.WorkflowModule\Hit.WorkflowModule` | `build/js/WFM.js`; workflowmodule.chalois.com (prod), localhost:8085 (local) |
| API | `D:\AO00\APIs\HitGISApi` | HitGISApi Web API; bgtmodule.chalois.com/WebApi (prod), gbcmbmodules-acc.chalois.com/BGTWebApi (acc) |

Source facts were traced against the code itself (primary source), not against documentation. Line numbers are as of 2026-09-29 and drift as code changes.

## Duplicate working copies (read before anything else)

Several modules exist twice on this machine. The copies under `D:\AO00\HIT.GeoborgCombiApp` are older mirrors of the top-level repositories:

| Module | Active-looking copy | Older mirror |
| --- | --- | --- |
| WFM | `D:\AO00\Hit.WorkflowModule\Hit.WorkflowModule` (bundle built 2026-09-04) | `D:\AO00\HIT.GeoborgCombiApp\Hit.WorkflowModule\Hit.WorkflowModule` (bundle built 2026-07-28) |
| API | `D:\AO00\APIs\HitGISApi` (`WorkflowController.cs` changed 2026-09-04) | `D:\AO00\HIT.GeoborgCombiApp\APIs\HitGISApi` (`WorkflowController.cs` changed 2026-08-26) |
| BAG module | `D:\AO00\HIT.GeoborgCombiApp\Viewers\BAG_Viewer 2.0\BAG_Viewer` (no top-level equivalent found) | - |

Confirm which copy the build and deploy jobs use before shipping anything.

---

## Entry 1: Workflow Info - Uploaden (btnUploadWorkflowInfo)

Menu path: `Instellingen > Administratief > Workflow Info > Uploaden`

One-line summary: imports a workflow definition (Excel workbook plus action images) for one task type into the database and image store.

### At a glance

| Item | Value |
| --- | --- |
| Toolbar item | `Instellingen > Administratief > Workflow Info > Uploaden` (`btnUploadWorkflowInfo`) |
| SOR entry point | `Scripts/GenericSettings/SettingsUI.js:34` (click handler at `:38-39`) |
| Handler chain | `SOR.SettingsBusiness.UploadWorkflowInfo` (`SettingsBusiness.js:3-5`) -> `HIT.WFM.UploadWorkflowInfo` (`workflow.debug.js:37-42`) |
| Upload call | WFM module POSTs multipart to `Common/UploadFile` (`GenericFileUploader.js:178`) |
| Import call | WFM module POSTs JSON to `Workflow/ImportWorkflowInfoNew` (`Utilities.js:375`) |
| Deployments touched | SOR host, WFM module host, HitGISApi host, SYS database, project database |

### Call chain

| # | Deployment | File:line | What happens |
| --- | --- | --- | --- |
| 1 | SOR | `Scripts/GenericSettings/SettingsUI.js:32-44` | Defines menu item `btnUploadWorkflowInfo`; click calls `SOR.SettingsBusiness.UploadWorkflowInfo()` (`:38-39`) |
| 2 | SOR | `Scripts/MainLayout.js:1082-1105` | `btnSettings` gear button; its `beforeshow` pulls the items from `SOR.SettingsUI.GetSettingsMenuItems()` (`SettingsUI.js:265-270`) |
| 2a | SOR | `Scripts/GenericSettings/SettingsUI.js:12-29` | Parent menu items `btnAdministratief` and `btnWorkflowInfo` |
| 3 | SOR | `Scripts/GenericSettings/SettingsBusiness.js:3-5` | Calls `HIT.WFM.UploadWorkflowInfo(REPORT_CODE, USER, DOMAIN)`; the values come from `Scripts/Globals.js:5,6,12` |
| 4 | WFM | `Scripts/Apis/workflow.debug.js:37-42` | Facade stores the three values on `HIT.WFM` and calls `HIT.WFM.Util.UploadWorkflowInfo()` |
| 5 | WFM | `Scripts/Utilities.js:353-367` | Opens `HIT.WFM.GenericFileUploader`: context `WorkflowSettings`, folder mode, extension `xls, xlsx`, callback `HIT.WFM.Util.ImportWorkflowInfo`; loading mask from `Scripts/CustomMask/js/LoadCustomMask.js:125` |
| 6 | WFM | `Scripts/GenericFileUploader.js:125-217` | Validates the chosen folder (one base name `[Taaktype]_[Beschrijving]`, taaktype at most 3 characters, total size max 43,000,000 bytes by default); POSTs multipart (`domain`, `context=WorkflowSettings`, all files) to `HitGisApi + "Common/UploadFile"` (`:178`); passes the workbook path to the callback (`:186-195`) |
| 7 | API | `HitGISApi/Controllers/CommonController.cs:30-110` | Saves the files to `DocBasePath\<DOMAIN>\WorkflowSettings` (`:41-49`); deletes previous `<taaktype>_*` files (`:68-86`); returns the full paths with HTTP 201 (`:97`) |
| 8 | WFM | `Scripts/Utilities.js:368-397` | POSTs JSON `{ report_code, excelFilePath, domain }` to `HitGisApi + "Workflow/ImportWorkflowInfoNew"` (`:375`) via `HIT.WFM.Util.GetJsonData` (`:35`); feedback through the shared `messageBox` |
| 9 | API | `HitGISApi/Controllers/WorkflowController.cs:546-566` | Reads appSetting `WorkflowImgDirectory` and `Helper/Workflow/workflowinfo_mapping_new.json` (`:551-552`), then calls `GISBusiness.ImportWorkflowInfoNew` |

Server-side processing inside `API: HitGISApi/Helper/GISBusiness.cs:3617-3702`:

| Step | Where | Detail |
| --- | --- | --- |
| Validate file name | `ValidationCheck.ValidateFileName` (`Helper/ValidationCheck.cs`) | File name format check before any work |
| Read mapping | `GISBusiness.cs:3635` + `Helper/Workflow/workflowinfo_mapping_new.json` | Excel tab and column mapping for this import |
| Read workbook | `GISBusiness.cs:609` (`ReadExcelBook`) | Reads the uploaded Excel workbook |
| Validate tabs | `GISBusiness.cs:3644` (`ValidationCheck.ValidateMandatoryTabs`) | All mandatory tabs must exist |
| Parse file name | `GISBusiness.cs:3704-3717` (`ParseImportedFileName`) | `taaktype` and description parsed from `[Taaktype]_[Beschrijving]` |
| Build defaults | `GISBusiness.cs:3719-3766` (`BuildAndAssignDefaultWorkflowInfo`) | Builds default action and `hittasks_settings` rows |
| Extract images | `GISBusiness.cs:858-885` (`CopynSaveWorkflowImgs`) | Writes action images to `WorkflowImgDirectory\<domain>\<taaktype>\<actie>.png` |
| Save to database | `GISBusiness.cs:3768+` (`SaveWorkflowInfoNew`) | `taak_type_settings` to the SYS database (`BGT_SysDB_ConnString`); `action_mapping` and `action_settings` to the project database (`GetProjectDbConnString(report_code)`, `:396`) via `Helper/SQLProvider.cs` |

### Files by deployment

#### SOR web application

| File | Key lines | Role |
| --- | --- | --- |
| `Scripts/GenericSettings/SettingsUI.js` | `:32-44`, `:12-29`, `:265-270` | Defines the menu item, its two parents, and the settings menu registration |
| `Scripts/GenericSettings/SettingsBusiness.js` | `:3-5` | Bridge to the WFM module |
| `Scripts/MainLayout.js` | `:1082-1105` | Instellingen gear button and menu injection |
| `Scripts/Globals.js` | `:5`, `:6`, `:12` | Source of `USER`, `DOMAIN`, `REPORT_CODE` passed into the WFM call |
| `Default.aspx` | `:38`, `:50`, `:151-155`, `:315-316`, `:321` | Loads the scripts and the WFM module; declares `messageBox` |
| `lib/lobibox/js/messagePanel.js` | - | Provides the shared `messageBox` used for feedback |

#### WFM module (workflowmodule.chalois.com / localhost:8085)

| File | Key lines | Role |
| --- | --- | --- |
| `Scripts/Apis/workflow.debug.js` | `:37-42`, `:22-26` | `HIT.WFM.UploadWorkflowInfo` facade; also exposes the same action in the module's own menu |
| `Scripts/Utilities.js` | `:353-367`, `:368-397` | Launches the uploader and calls the import endpoint |
| `Scripts/GenericFileUploader.js` | `:125-217` | Folder picker, client-side validation and the multipart POST to `Common/UploadFile` |
| `Scripts/CustomMask/js/LoadCustomMask.js` | `:125` | `HIT.WFM.MASK` loading indicator used around the API calls |
| `Scripts/WorkflowConfig.js` | `:8` | API URLs for the ACC bundle (`HitGisApi`) |
| `Scripts/WorkflowConfig-PROD.js` | `:3` | API URLs for the PROD bundle (`HitGisApi`) |
| `gulpfile.js` / `gulpfile-PROD.js` | `:31`, `:62` | Build manifests: concatenate the file list into `build/js/WFM.js` |
| `build/js/WFM.js` (and `WFM.min.js`) | - | Deployment artifact loaded by SOR (`Default.aspx:153`) |
| `styles/wfm.css`, `Scripts/CustomMask/css/StyleSheetCutomMask.css` | - | Module styling loaded by SOR (`Default.aspx:154-155`) |
| `Images/` folder | - | Destination referenced by the API setting `WorkflowImgDirectory` |

#### HitGISApi (bgtmodule.chalois.com/WebApi / gbcmbmodules-acc.chalois.com/BGTWebApi)

| File | Key lines | Role |
| --- | --- | --- |
| `HitGISApi/Controllers/WorkflowController.cs` | `:546-566` | `ImportWorkflowInfoNew` endpoint |
| `HitGISApi/Controllers/CommonController.cs` | `:30-110` | `UploadFile` endpoint |
| `HitGISApi/Helper/GISBusiness.cs` | `:3617-3702`, `:858-885`, `:3768+` | Import logic, image extraction, database writes |
| `HitGISApi/Helper/ValidationCheck.cs` | - | File name and workbook tab validation |
| `HitGISApi/Helper/SQLProvider.cs` | - | SQL for delete and insert of task settings and mappings |
| `HitGISApi/Helper/Utilities.cs` | - | `GetAppKeyValue` config access |
| `HitGISApi/Models/WorkflowTables.cs` | `:171`, `:177` | `ImportWorkflowInfoRequestNew`, `hittasks_settings` |
| `HitGISApi/Models/DataModels.cs` | `:415` | `WorkflowImportResponse` |
| `HitGISApi/Helper/Workflow/workflowinfo_mapping_new.json` | - | Excel tab and column mapping used by this tool; declared as Content in `HitGISApi.csproj` |
| `HitGISApi/Helper/Workflow/workflowinfo_mapping.json` | - | Mapping for the older `ImportWorkflowInfo` endpoint; still present, not used by this tool |
| `HitGISApi/Web.config` | `:14`, `:30`, `:45`, `:46` | `DocBasePath`, `BGT_SysDB_ConnString`, `WorkflowImgDirectory` |
| `HitGISApi.dll` | - | Compiled deployment artifact; every `.cs` above is compiled into it |

### Connections to other tools

| Connected caller | Where | What is shared |
| --- | --- | --- |
| Werkstroom menu item "Importeer Workflow Info" | SOR `Scripts/BGT.js:221-225` | Calls the exact same `SOR.SettingsBusiness.UploadWorkflowInfo`; a change here affects both toolbar entries |
| WFM module's own menu "Importeer Workflow Info" | WFM `Scripts/Apis/workflow.debug.js:22-26` | Reaches the same WFM functions; the API endpoints are shared with the standalone module UI |
| WDO/Geo configuration upload | SOR `Scripts/GenericSettings/SettingsBusiness.js:66-68`; API `CommonController.cs:51-56` | Shares `Common/UploadFile`; only the `WorkflowSettings` context deletes previous `<taaktype>_*` files |
| Older import endpoint | API `WorkflowController.cs:208-227`, `GISBusiness.cs:749` | Kept for older clients; uses `workflowinfo_mapping.json`; do not remove when changing this tool |
| BAG API equivalent | `D:\AO00\APIs\HitBAGApi\HitBAGApi\Controllers\BAGWorkflowController.cs:23` | Separate API with its own import action |

### Environment and configuration

| Concern | Where | Current state |
| --- | --- | --- |
| Module blocks in the SOR host | `Default.aspx:98-115` (PROD), `:116-142` (ACC), `:144-177` (Local) | Local block active (`:153`); activate the matching block per environment |
| API base URL | WFM `Scripts/WorkflowConfig.js:8` (ACC) vs `Scripts/WorkflowConfig-PROD.js:3` (PROD) | Chosen at bundle build time by `gulpfile.js` vs `gulpfile-PROD.js`; the current `build/js/WFM.js` (2026-09-04) embeds the ACC URL |
| Workflow images directory | API `Web.config:46` | `D:\Work\AO00\HIT.GeoborgCombiApp\Hit.WorkflowModule\Hit.WorkflowModule\Images`; the API must be able to write there; no `Web.Release.config` transform exists for this key |
| Upload storage | API `Web.config:14,45` (`DocBasePath`) | `D:\Websites\data\GeoBorgBGT\`; uploads land in `<DocBasePath>\<DOMAIN>\WorkflowSettings` |
| SYS database | API `Web.config:30` (`BGT_SysDB_ConnString`) | Receives `taak_type_settings` |

### Deployment checklist

| Target | Ship | Notes |
| --- | --- | --- |
| SOR host | `SettingsUI.js`, `SettingsBusiness.js`; `MainLayout.js` / `Default.aspx` only if the wiring changed | These files are loaded without a cache-busting query, so browser caching can hide the update |
| WFM host | Rebuilt `build/js/WFM.js` (gulp) | Shipping sources alone does nothing; pick the ACC or PROD config variant |
| API host | `HitGISApi.dll` plus `Helper/Workflow/workflowinfo_mapping_new.json` and the `Web.config` keys | Verify `DocBasePath`, `WorkflowImgDirectory` and `BGT_SysDB_ConnString` on that host |
| Databases | SYS database and the project database for the target `report_code` | Must be reachable and writable |
| Input data | Workbook per `workflowinfo_mapping_new.json`; name `[Taaktype]_[Beschrijving].xlsx`, taaktype at most 3 characters | Enforced client-side (`GenericFileUploader`) and server-side (`ValidationCheck`) |

### Risks and open points

- Duplicate working copies: the WFM mirror is older (bundle 2026-07-28 vs 2026-09-04) and so is the API mirror (`WorkflowController.cs` 2026-08-26 vs 2026-09-04). `WorkflowImgDirectory` points at the mirror copy. Confirm which copy the build and deploy jobs use.
- The upload target API is decided by the WFM bundle, not by SOR configuration. The current local setup already sends imports to the ACC API because the locally served `WFM.js` was built with the ACC config.
- `iconCls btnUploaden` (`SettingsUI.js:35`) has no matching CSS rule in the SOR, WFM, BAG, BGT or BGT Editor styles scanned; the item renders without a custom icon.

---

## Entry template for future tools

Use this skeleton so every entry reads the same way. Keep the "At a glance" table short; put detail in the chain tables.

```
## Entry N: <Tool name> (<id>)

Menu path: <Instellingen > ... > label>

One-line summary: <what the tool does>.

### At a glance
| Item | Value |

### Call chain
| # | Deployment | File:line | What happens |

### Files by deployment
#### SOR web application
#### WFM module (if involved)
#### HitGISApi (if involved)
#### Other modules (if involved)

### Connections to other tools

### Environment and configuration

### Deployment checklist

### Risks and open points
```
