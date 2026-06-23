# CLAUDE.md — Microsoft365DSC

## Repository Overview

Microsoft365DSC is a PowerShell Desired State Configuration (DSC) module that lets organizations automate deployment, configuration, reporting, and monitoring of Microsoft 365 tenants. Resources interact with M365 services (Azure AD/Entra, Exchange Online, SharePoint, Teams, Intune, Defender, Power Platform, etc.) via remote API calls; the agent machine only needs internet access to the M365 tenant.

Current module version: **1.25.827.1** (see `Modules/Microsoft365DSC/Microsoft365DSC.psd1`)

---

## Directory Structure

```
Microsoft365DSC/
├── Modules/Microsoft365DSC/          # The publishable module
│   ├── Microsoft365DSC.psd1          # Module manifest (version, exports, dependencies)
│   ├── DSCResources/                 # 498 DSC resources, one sub-folder each
│   │   └── MSFT_<WorkloadName>/
│   │       ├── MSFT_<Name>.psm1      # Resource implementation
│   │       ├── MSFT_<Name>.schema.mof# MOF schema (CIM types + resource class)
│   │       ├── settings.json         # Permissions manifest for this resource
│   │       └── readme.md             # Short description (populated during build)
│   ├── Examples/Resources/<Name>/    # Usage examples per resource (1-Create/2-Update/3-Remove)
│   ├── Dependencies/
│   │   └── Manifest.psd1             # Pinned versions of all required PS modules
│   └── Modules/                      # Internal helper modules (not DSC resources)
│       ├── M365DSCUtil.psm1          # Core utilities, connection, export, drift detection
│       ├── M365DSCReverse.psm1       # Config extraction / "reverse DSC" engine
│       ├── M365DSCPermissions.psm1   # Permission list and validation helpers
│       ├── M365DSCLogEngine.psm1     # Error logging and event log entries
│       ├── M365DSCReport.psm1        # HTML/delta report generation
│       ├── M365DSCDRGUtil.psm1       # Drift/export string utilities
│       ├── M365DSCDocGenerator.psm1  # Wiki/docs generator
│       ├── M365DSCTelemetryEngine.psm1
│       ├── M365DSCSchemaHandler.psm1
│       ├── M365DSCCheckProperties.psm1
│       ├── M365DSCAgent.psm1
│       ├── M365DSCConfigurationHelper.psm1
│       ├── M365DSCStubsUtility.psm1
│       ├── M365DSCExoResourceUtils.psm1
│       ├── M365DSCIntuneSettingsCatalogUtil.psm1
│       ├── WorkloadHelpers/          # Per-workload REST/SDK helpers (Azure, ADO, Fabric…)
│       └── EncodingHelpers/
│           └── M365DSCEmojis.psm1
├── Tests/
│   ├── TestHarness.psm1              # Orchestrates all test suites via Invoke-TestHarness
│   ├── Unit/
│   │   ├── UnitTestHelper.psm1       # New-M365DscUnitTestHelper factory
│   │   ├── Stubs/                    # Stub modules (Microsoft365.psm1, Generic.psm1)
│   │   └── Microsoft365DSC/          # One *.Tests.ps1 per resource (499 files)
│   ├── QA/
│   │   ├── Microsoft365DSC.Resources.Tests.ps1  # Schema/MOF validation
│   │   ├── Microsoft365DSC.Examples.Tests.ps1   # Examples compile correctly
│   │   ├── Microsoft365DSC.SettingsJson.Tests.ps1 # settings.json permission validation
│   │   ├── Microsoft365DSC.ModuleManifest.Tests.ps1
│   │   └── Graph.PermissionList.txt  # Allowed Graph permission names
│   └── Integration/                  # Live-tenant integration tests (require credentials)
├── ResourceGenerator/                # Code generator for new resources
│   └── readme.template.md
├── generator/                        # Node-based docs/wiki generator
├── docs/                             # mkdocs documentation site source
├── .github/
│   ├── workflows/                    # CI/CD pipelines (see CI section below)
│   └── ISSUE_TEMPLATE/
├── CHANGELOG.md
└── SchemaDefinition.json             # Shared CIM embedded-instance type definitions
```

---

## Workload Prefixes

Resources are named `MSFT_<WorkloadPrefix><ResourceName>`. The supported workload prefixes are:

| Prefix | Workload |
|--------|----------|
| `AAD` / `AADPIM` | Azure Active Directory / Entra ID (incl. PIM) |
| `ADO` | Azure DevOps |
| `AZURE` | Azure (Az modules) |
| `DEFENDER` / `Def` | Microsoft Defender |
| `EXO` / `EXOATP` / `EXOCAS` / `EXOEOP` / `EXOIRM` / `EXOOME` | Exchange Online |
| `FABRIC` / `Fab` | Microsoft Fabric |
| `INTUNE` / `Int` | Microsoft Intune |
| `O365` | Office 365 tenant settings |
| `OD` | OneDrive |
| `PLANNER` / `Pla` | Microsoft Planner |
| `PP` / `PPDLP` | Power Platform |
| `SC` / `SCDLP` | Security & Compliance |
| `SENTINEL` / `Sen` | Microsoft Sentinel |
| `SH` / `SHS` | Services Hub |
| `SPO` | SharePoint Online |
| `TEAMS` / `Tea` | Microsoft Teams |
| `VIVA` / `Viv` | Viva / Viva Connections |

The export workloads available in `Export-M365DSCConfiguration -Workloads` are: `AAD`, `ADO`, `AZURE`, `COMMERCE`, `DEFENDER`, `EXO`, `FABRIC`, `INTUNE`, `O365`, `OD`, `PLANNER`, `PP`, `SC`, `SENTINEL`, `SH`, `SPO`, `TEAMS`, `VIVA`.

---

## DSC Resource Anatomy

Every resource folder (`DSCResources/MSFT_<Name>/`) contains exactly four files.

### 1. `MSFT_<Name>.psm1` — Resource Implementation

Every resource **must** implement these four functions:

```powershell
# Required at top of file — loads required modules lazily
Confirm-M365DSCModuleDependency -ModuleName 'MSFT_<Name>'

function Get-TargetResource  { ... }   # Returns current state as hashtable
function Set-TargetResource  { ... }   # Enforces desired state
function Test-TargetResource { ... }   # Returns $true (in-sync) or $false (drift)
function Export-TargetResource { ... } # Exports tenant config to DSC syntax
```

**Standard authentication parameters** — all four functions share these at the bottom of their `param()` block:

```powershell
[Parameter()]
[System.Management.Automation.PSCredential]
$Credential,

[Parameter()]
[System.String]
$ApplicationId,

[Parameter()]
[System.String]
$TenantId,

[Parameter()]
[System.String]
$ApplicationSecret,

[Parameter()]
[System.String]
$CertificateThumbprint,

[Parameter()]
[System.Boolean]
$ManagedIdentity,

[Parameter()]
[System.String[]]
$AccessTokens
```

**Connection pattern** — always call `New-M365DSCConnection` at the start of each function body:

```powershell
$ConnectionMode = New-M365DSCConnection -Workload 'MicrosoftGraph' `
    -InboundParameters $PSBoundParameters
```

**Export script variable** — use `$Script:ExportMode` and `$Script:exportedInstance` to avoid redundant API calls during bulk export:

```powershell
if ($Script:ExportMode)
{
    $AADApp = $Script:exportedInstance
}
else
{
    # Live API call
}
```

**Error logging**:

```powershell
New-M365DSCLogEntry -Message 'Error retrieving data:' `
    -Exception $_ `
    -Source $($MyInvocation.MyCommand.Source) `
    -TenantId $TenantId `
    -Credential $Credential
```

**CRITICAL — no hardcoded Graph endpoints**: Never put `https://graph.microsoft.com` directly in resource `.psm1` files. Always use `Get-MSCloudLoginConnectionProfile -Workload MicrosoftGraph).ResourceUrl` to build the URL:

```powershell
$Uri = (Get-MSCloudLoginConnectionProfile -Workload MicrosoftGraph).ResourceUrl + "beta/..."
```

The `Validation Checks` CI workflow enforces this and will fail any PR that violates it.

### 2. `MSFT_<Name>.schema.mof` — MOF Schema

Defines the CIM class for the resource and any embedded CIM classes it uses.

```mof
[ClassVersion("1.0.0")]
class MSFT_EmbeddedType
{
    [Write, Description("...")] String SomeProperty;
};

[ClassVersion("1.0.0.0"), FriendlyName("ResourceFriendlyName")]
class MSFT_ResourceName : OMI_BaseResource
{
    [Key, Description("Primary key.")] String KeyProperty;
    [Write, Description("..."), EmbeddedInstance("MSFT_EmbeddedType")] String ComplexProp;
    [Write, Description("Present or Absent."), ValueMap{"Present","Absent"}, Values{"Present","Absent"}] String Ensure;
    // ... standard auth params ...
};
```

Property qualifiers: `Key` (mandatory key), `Write` (optional), `Required` (mandatory non-key), `Read` (output only).

### 3. `settings.json` — Permissions Manifest

Declares which MS Graph, Exchange, SharePoint, or other API permissions the resource needs:

```json
{
  "resourceName": "AADApplication",
  "description": "Configures an Azure AD Application.",
  "roles": { "read": ["Security Reader"], "update": [] },
  "permissions": {
    "graph": {
      "delegated": {
        "read": [{"name": "Application.Read.All"}],
        "update": [{"name": "Application.ReadWrite.All"}, {"name": "User.Read.All"}]
      },
      "application": {
        "read": [{"name": "Application.Read.All"}],
        "update": [{"name": "Application.ReadWrite.All"}, {"name": "User.Read.All"}]
      }
    }
  },
  "requiredModules": ["Microsoft.Graph.Applications", "Microsoft.Graph.Authentication", "..."]
}
```

All permissions must exist in `Tests/QA/Graph.PermissionList.txt`. The `Microsoft365DSC.SettingsJson.Tests.ps1` QA test validates this.

### 4. `readme.md` — Description

Short Markdown description used to generate the wiki. Template:

```markdown
# <ResourceFriendlyName>

## Description

<ResourceDescription>
```

---

## Examples

Every resource must have examples under `Modules/Microsoft365DSC/Examples/Resources/<FriendlyName>/`:

- `1-Create.ps1` — creates the resource (Ensure = "Present")
- `2-Update.ps1` — modifies properties
- `3-Remove.ps1` — removes the resource (Ensure = "Absent")

Example structure:

```powershell
Configuration Example
{
    param(
        [Parameter()] [System.String] $ApplicationId,
        [Parameter()] [System.String] $TenantId,
        [Parameter()] [System.String] $CertificateThumbprint
    )
    Import-DscResource -ModuleName Microsoft365DSC
    node localhost
    {
        AADApplication 'MyApp'
        {
            DisplayName           = "MyApp"
            Ensure                = "Present"
            ApplicationId         = $ApplicationId
            TenantId              = $TenantId
            CertificateThumbprint = $CertificateThumbprint
        }
    }
}
```

---

## Unit Tests

Each resource has a corresponding test at `Tests/Unit/Microsoft365DSC/Microsoft365DSC.<FriendlyName>.Tests.ps1`.

**Test setup pattern**:

```powershell
$M365DSCTestFolder = Join-Path -Path $PSScriptRoot -ChildPath '..\..\Unit' -Resolve
$CmdletModule   = Join-Path -Path $M365DSCTestFolder -ChildPath '\Stubs\Microsoft365.psm1' -Resolve
$GenericStubPath = Join-Path -Path $M365DSCTestFolder -ChildPath '\Stubs\Generic.psm1' -Resolve
Import-Module -Name (Join-Path -Path $M365DSCTestFolder -ChildPath '\UnitTestHelper.psm1' -Resolve)

$Global:DscHelper = New-M365DscUnitTestHelper -StubModule $CmdletModule `
    -DscResource 'AADApplication' -GenericStubModule $GenericStubPath

Describe -Name $Global:DscHelper.DescribeHeader -Fixture {
    InModuleScope -ModuleName $Global:DscHelper.ModuleName -ScriptBlock {
        Invoke-Command -ScriptBlock $Global:DscHelper.InitializeScript -NoNewScope
        BeforeAll {
            # Mock all external cmdlets
            Mock -CommandName New-M365DSCConnection -MockWith { return 'Credentials' }
            Mock -CommandName Write-M365DSCHost -MockWith { }
            Mock -ModuleName M365DSCUtil -CommandName Confirm-M365DSCDependencies -MockWith { }
            $Script:exportedInstance = $null
            $Script:ExportMode = $false
        }
        # ... Context blocks for Get, Set, Test, Export ...
    }
}
```

---

## Running Tests Locally

Tests require **Windows** and **PowerShell 5.1+** (CI runs on `windows-latest`). Required modules:

```powershell
Install-PSResource -Name ReverseDSC -Scope AllUsers -TrustRepository
Install-PSResource -Name DSCParser -Scope AllUsers -TrustRepository
Install-PSResource -Name PSDesiredStateConfiguration -Scope AllUsers -TrustRepository
Install-PSResource -Name Pester -Scope AllUsers -TrustRepository
```

Run all tests via the harness:

```powershell
Import-Module './Tests/TestHarness.psm1' -Force
$MaximumFunctionCount = 32767

# Quality checks (QA tests)
$results = Invoke-QualityChecksHarness
if ($results.FailedCount -gt 0) { throw "$($results.FailedCount) Quality Check(s) Failed" }

# Unit tests
$results = Invoke-TestHarness -IgnoreCodeCoverage
if ($results.FailedCount -gt 0) { throw "$($results.FailedCount) Unit Test(s) Failed" }
```

Run a single resource's unit test:

```powershell
Import-Module './Tests/TestHarness.psm1' -Force
$MaximumFunctionCount = 32767
Invoke-TestHarness -DscTestsPath './Tests/Unit/Microsoft365DSC/Microsoft365DSC.AADApplication.Tests.ps1' -IgnoreCodeCoverage
```

---

## CI/CD Workflows (`.github/workflows/`)

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| `Unit Tests.yml` | push / PR | Quality checks + Pester unit tests (Windows) |
| `Validation Checks.yml` | push / PR | Enforces no hardcoded Graph URLs in resources |
| `AzureCloud - Full-Circle - *.yml` | push / PR | Full integration tests per workload (live tenant) |
| `Global - Integration - *.yml` | push / PR | Cross-workload integration tests |
| `CodeCoverage.yml` | push to dev | Uploads to Codecov |
| `PublishToGallery.yml` | manual/release | Publishes to PSGallery |
| `Generate Wiki.yml` | merge to master | Regenerates documentation wiki |
| `Describe Resources Schemas.yml` | PR | Schema description validation |

All CI jobs only run when `github.repository == 'microsoft/Microsoft365DSC'` (not forks), except validation checks.

---

## Key Helper Functions

### `M365DSCUtil.psm1`

| Function | Purpose |
|----------|---------|
| `New-M365DSCConnection` | Establishes connection to a workload using the supplied auth params |
| `Export-M365DSCConfiguration` | Public entry point for tenant configuration export |
| `Confirm-M365DSCDependencies` | Validates required modules are installed |
| `Get-M365DSCAllResources` | Returns list of all resource friendly names |
| `Test-M365DSCParameterState` | Core drift detection — compares desired vs current values |
| `New-M365DSCLogEntry` | Writes errors to Windows Event Log and verbose output |
| `Write-M365DSCHost` | Wraps Write-Host for testability (always mocked in unit tests) |
| `Convert-M365DscHashtableToString` | Obfuscates sensitive parameters in log output |

### `M365DSCReverse.psm1`

| Function | Purpose |
|----------|---------|
| `Start-M365DSCConfigurationExtract` | Orchestrates bulk export from a live tenant |

Extraction modes: `Lite`, `Default` (default), `Full` (includes high-cardinality resources like `AADGroup`, `TeamsTeam`). High-cardinality resources are listed in `$Global:FullComponents` in M365DSCUtil.psm1.

### `M365DSCPermissions.psm1`

| Function | Purpose |
|----------|---------|
| `Get-M365DSCCompiledPermissionList` | Returns merged permission list for a set of resources |

---

## Authentication Methods

Resources support four mutually-exclusive authentication modes (passed in via standard parameters):

1. **Credentials** — `$Credential` (PSCredential, username must match `*.onmicrosoft.*`)
2. **Service Principal + Secret** — `$ApplicationId` + `$TenantId` + `$ApplicationSecret`
3. **Service Principal + Certificate** — `$ApplicationId` + `$TenantId` + `$CertificateThumbprint`
4. **Managed Identity** — `$ManagedIdentity = $true` + `$TenantId`
5. **Access Tokens** — `$AccessTokens` (string array of pre-obtained tokens)

`TenantId` should be the tenant name (`contoso.onmicrosoft.com`), not a GUID.

---

## Conventions to Follow

1. **Never hardcode `https://graph.microsoft.com`** in resource `.psm1` files. Use `(Get-MSCloudLoginConnectionProfile -Workload MicrosoftGraph).ResourceUrl`.

2. **Always call `Confirm-M365DSCModuleDependency`** as the very first line of each resource's `.psm1`, before any function definitions.

3. **Mock `Write-M365DSCHost`** in unit tests to suppress output — never call `Write-Host` directly in resources.

4. **Use `$Script:ExportMode` / `$Script:exportedInstance`** to short-circuit API calls during bulk exports.

5. **Use `New-M365DSCLogEntry`** for all error handling in `Get-TargetResource`, `Set-TargetResource`, and `Test-TargetResource`. Do not use `Write-Error` directly.

6. **All settings.json permissions** must be valid names from `Tests/QA/Graph.PermissionList.txt`.

7. **Each resource must have all four files**: `.psm1`, `.schema.mof`, `settings.json`, `readme.md`.

8. **Examples must be named** `1-Create.ps1`, `2-Update.ps1`, `3-Remove.ps1` and must compile without errors.

9. **MOF classes must have `ClassVersion`** qualifier. The main resource class must also have `FriendlyName` (the name without `MSFT_` prefix).

10. **`Test-TargetResource` must call `Test-M365DSCParameterState`** (from M365DSCUtil) to detect drift. Do not hand-roll comparison logic.

11. **`Ensure` parameter** is standard for resources that can be created/deleted. Use `ValueMap{"Present","Absent"}`.

12. **`IsSingleInstance`** is used (as the key) for singleton resources where only one instance can exist per tenant.

13. **Avoid `Az.*` or `Microsoft.Graph.*` direct REST calls** in resource code unless there is no SDK alternative. Use the appropriate PS SDK cmdlets.

14. **Do not use `[System.Net.WebClient]` or `Invoke-WebRequest`** for Graph API calls; use `Invoke-MgGraphRequest` instead.

---

## Dependencies

All dependency module versions are pinned in `Modules/Microsoft365DSC/Dependencies/Manifest.psd1`. Key dependencies include:

- `Microsoft.Graph.*` (v2.28.0) — Graph SDK modules
- `Microsoft.Graph.Beta.*` (v2.28.0) — Beta Graph SDK
- `ExchangeOnlineManagement` (v3.9.0)
- `Az.Accounts`, `Az.Resources`, `Az.Security`, etc.
- `MSCloudLoginAssistant` — connection management
- `ReverseDSC` — config extraction engine
- `DSCParser` — DSC document parsing

Install via:

```powershell
Install-Module -Name Microsoft365DSC -Force
Update-M365DSCModule   # installs/updates all pinned dependencies
```

---

## Resource Generator

New resources can be scaffolded using the `ResourceGenerator/` tooling. After generating, manually:

1. Implement `Get-/Set-/Test-/Export-TargetResource` in the generated `.psm1`.
2. Fill in `settings.json` with accurate permissions.
3. Update `readme.md` with a description.
4. Write unit tests in `Tests/Unit/Microsoft365DSC/`.
5. Create the three example files.
6. Add the module to `Microsoft365DSC.psd1` `DscResourcesToExport`.

---

## Branch Strategy

- **`master`** — latest published release. No direct contributions.
- **`dev`** — active development branch. All PRs target `dev`.
