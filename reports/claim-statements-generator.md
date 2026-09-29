# Assessment - RPA.ClaimStatements.Generator

## Identification

**Repository Name**: claim-statements-generator (solution: `RPA.ClaimStatements.Generator`)
**Type**: Console application / scheduled batch job (document + data generator)
**Language**: C#
**Frameworks**: .NET Framework 4.5, Entity Framework 6.1.3, Unity 5.5.6 (DI container), EPPlus 4.1.0, DocumentFormat.OpenXml 2.7.2, OpenXmlPowerTools 4.5.3.2, NReco.PdfGenerator 1.1.15, CsvHelper, WinSCP 5.9.6
**Repository URL**: Local clone only — `claim-statements-generator/`

## Summary
RPA.ClaimStatements.Generator is a console application (`Console.WriteLine`-driven CLI) that generates rural payment "claim statements" (BPS — Basic Payment Scheme, and XB — cross-border/other scheme statements) for DEFRA's Rural Payments Agency. It is invoked either non-interactively with command-line arguments (`statementType`, `maxBatchSize`) or interactively via console prompts, and can alternatively run in an "I" (import) mode to pull data from upstream systems (SITI, XB, DAX/AP/AR feeds) into its local database.

Internally it uses the Unity DI container to wire up a large graph of services: import services (SITI, XB, AP, AR), bulk SQL insert services for many statement/reference types (SUM, SUM2, BPS, BPSPEN, GR, GRPEN, PR, YF, CLD, XB, XBData, AP, AR), document generation/serialization (`ISerializer<Claim>` → PDF/Excel via EPPlus/OpenXML/NReco), FTP transfer (WinSCP), email notification, and error/monitoring/logging services. This is one of the most complex, business-logic-dense applications in the portfolio.

## Service Dependencies

### Cloud Services (GCP/AWS/Azure)
- None found — fully on-prem, file-system, SMTP-relay and FTP-based

### Databases
- **ClaimStatementsContext** (`RPA.ClaimStatements` on `.\sqlexpress` locally, environment-specific per config transform): EF6 Code-First context storing/generating BPS, XB, and related claim statement/import data (SUM, SUM2, BPSPEN, GR, GRPEN, PR, YF, CLD tables inferred from `IBulkInsert<T>` registrations)
- Upstream import sources referenced by namespace: `Models.Entities.DAX`, `Models.Entities.SITI`, `Models.Entities.XB` — implies data is imported from Dynamics AX (DAX), a "SITI" system, and an "XB" (cross-border) system, likely via linked SQL Server databases or extract files

### Messaging
- None found

### Storage
- **Network file share** (`\\D1VMSMA001\inetpub\mailroot\Pickup`): SMTP pickup directory configured in `App.config` — mail is dropped as `.eml` files to a UNC path rather than sent via a live SMTP relay
- **FTP (via WinSCP.exe)**: file transfer service (`IFTPService`) — bundled `WinSCP.exe` binary and `WinSCP` NuGet package imply statements/files are pushed to/from a remote FTP/SFTP server

### APIs and External Integrations
- None found (no HTTP client dependencies observed) — all integration is via SQL Server linked data, flat-file import, FTP, and file-system-based email pickup

### Other Dependencies
- `Microsoft.VisualStudio.SlowCheetah`: config transformation tooling (Debug/Release/SIT/UAT/Production `App.{env}.config` variants present)
- `System.Management.Automation.dll` package reference — suggests PowerShell hosting/scripting invoked from within the app (consistent with `update-permissions.ps1`-style operational scripts seen elsewhere in the estate)
- `WinSCP.exe` checked directly into source control (binary dependency, not just a NuGet package)

## Communication

### Exposed Endpoints
- None — this is a batch/console application, not a service

### Consumed Endpoints
- None (no HTTP API calls found)

### Asynchronous Communication
| Type | Name | Action | Purpose |
|------|------|--------|---------|
| Produces (file) | SMTP pickup directory | Writes `.eml` file | Send generated-statement notification emails |
| Produces (file transfer) | FTP/SFTP endpoint (via WinSCP) | Upload/download | Transfer generated claim statement documents to/from a remote server |

### Communication Diagram

```mermaid
graph LR
    A[RPA.ClaimStatements.Generator - Console] -->|EF6| B[(SQL Server: RPA.ClaimStatements)]
    A -->|Import| C[(DAX / SITI / XB data sources)]
    A -->|FTP via WinSCP| D[Remote FTP/SFTP Server]
    A -->|Drop .eml file| E[\\D1VMSMA001 SMTP Pickup Share]
```

## Configuration

### Environment Variables
- None — configuration via `App.{Debug,Release,SIT,UAT,Production}.config` transforms (SlowCheetah)

### Configuration Files
- `App.config` (+ per-environment transforms): connection string, EF6 provider registration, SMTP pickup directory
- `packages.config`: legacy NuGet dependency pinning

### Secrets and Sensitive Parameters
- None visible in source — SQL connection uses Integrated Security; FTP credentials are presumably supplied at runtime/config (not found in the reviewed files, worth a targeted secret-scan before migration)

## Infrastructure

### Containerization
- **Dockerfile**: No
- **Base Image**: N/A — packaged as a Windows console `.exe` (icon file `Icon.ico` embedded), presumably run via Windows Task Scheduler or an orchestration tool

### Kubernetes/Helm
- **Manifests**: No

### Infrastructure as Code
- **Terraform/Bicep**: No

### CI/CD
- **Pipeline**: None found specific to this app (only the generic scaffolded workflow files injected by the migration tooling)
- **Notes**: `obfuscation-report.html` present, implying an external/manual obfuscation step in the release process, matching the pattern seen in `bank-holidays`

## Testing
- No dedicated `.Tests` project found for this repository (unlike most peers in the estate).

### Observations
Given the density of business logic (multiple statement types, bulk insert operations, document generation, FTP/email side effects), the total absence of automated tests is a significant migration risk — behavior-preservation during modernization will be difficult to verify without characterization tests.

## Points of Attention for Multi-Cloud/Azure Migration

### Cloud-Specific Dependencies
- None (fully on-prem) — but heavy **environment-specific infrastructure coupling**: UNC file share for mail pickup, FTP server via WinSCP binary, SQL Server Express locally

### Hardcoded Configurations
- `\\D1VMSMA001\inetpub\mailroot\Pickup` UNC path hardcoded per environment config — must migrate to a real SMTP relay (e.g., Azure Communication Services / SendGrid) since UNC paths are not available in Azure PaaS/containers
- `WinSCP.exe` checked into source and invoked as a subprocess — a very container/PaaS-unfriendly pattern; needs replacement with an SFTP client library (e.g., SSH.NET) callable in-process

### Legacy Code or Old Patterns
- .NET Framework 4.5 (out of support), EF6, Unity DI container — full rewrite/re-platform needed for .NET 8/10
- Heavy reliance on `System.Management.Automation` (PowerShell hosting) inside a batch job — anti-pattern for containerized/serverless targets
- No abstraction/interfaces around FTP or SMTP transport beyond the service-level interfaces already present (`IFTPService`, `IEmailService`) — good news: these existing seams make it easier to swap implementations during modernization

### Specific Recommendations
1. Treat this as a **high-complexity, high-business-risk** modernization target — largest number of distinct statement/import types of any app in the portfolio, and zero test coverage.
2. Before any code changes, build characterization/golden-file tests around the existing statement generation output (PDF/Excel) to guard against silent business-logic regressions.
3. Replace the `WinSCP.exe` subprocess pattern with SSH.NET (or Azure Storage/Blob-based file exchange if the remote party can be migrated too).
4. Replace UNC-path SMTP pickup with a proper SMTP relay or Azure Communication Services Email.
5. Re-platform to run as an **Azure Container Apps Job** or **Azure Functions (Timer/Durable)** given its batch/CLI invocation model, rather than a lift-and-shift VM.
6. Confirm whether `IDT.dll`-style internal libraries or shared "Security"/"People" databases are referenced anywhere in this app (none were found in this pass) before finalizing the dependency graph.

## Additional Observations
- This app imports from and/or shares reference data conceptually with BAQU's "DAX" dependency (`Models.Entities.DAX` namespace) — worth confirming in Phase 1 whether both applications point at the same Dynamics AX staging tables, which would be a cross-repo shared-dependency risk similar to the BAQU↔RPA.QualityPortal database coupling.
