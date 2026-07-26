# MECM Health Monitor

This package installs a read-only MECM operations dashboard on Windows IIS. You
enter only server, site, AdminService, and SQL connection details in XML. The
collector automatically creates the website data from MECM.

No deployment counts, compliance percentages, client records, component
statuses, or alerts are maintained in XML.

## What is collected automatically

| Dashboard area | Automatic source |
| --- | --- |
| OSD deployment monitoring | `SMS_DeploymentSummary`, feature type 7 |
| Patch compliance | `SMS_DeploymentSummary`, feature type 5 |
| Application/package deployment summary | `SMS_DeploymentSummary`, feature types 1, 2, 8, and 9 |
| Application/package content by distribution point | `v_ContentDistributionReport_DP` or `v_DistributionStatus`, enriched by `v_ContentDistributionMessages` and package views |
| OSD failed task-sequence step and exit code | `v_TaskExecutionStatus`, optionally joined to `v_Advertisement` and `v_R_System` |
| Active deployments | `SMS_DeploymentSummary` where targeted clients are greater than zero |
| MECM active alerts | `SMS_Alert` where `AlertState=0` and `Enabled=true` |
| Component health | `SMS_ComponentSummarizer` using the configured tally interval |
| Site-system health | `SMS_SiteSystemSummarizer` plus a TCP probe of each XML endpoint |
| Every managed client | SQL reporting view `v_R_System` |
| Client health and reported error | `v_CH_ClientSummary`, `v_CH_EvalResults`, and `v_StateNames` |
| Failed SCCM client installations | `v_ClientDeploymentState` joined to topic type 800 in `v_StateNames` |

The browser provides workload navigation, client search, site and health
filters, 100-row paging, active-alert display, responsive mobile navigation, a
live clock, manual snapshot refresh, timed refresh, and CSV export for the
current view, every individual monitoring feature, or all monitoring data.

## Architecture

text
MECM AdminService ----\
                       > PowerShell scheduled collector --> dashboard.json
MECM reporting SQL ---/                                      |
Configured TCP probes ---------------------------------------|--> IIS static website


The browser never connects directly to SQL or the SMS Provider. A scheduled
PowerShell task uses Windows authentication, writes one JSON snapshot
atomically, and IIS serves static files with Windows Authentication.

## 1. Prerequisites

- Windows Server 2019 or later for the IIS monitor server.
- Network access from the monitor server to:
  - SMS Provider/AdminService on HTTPS TCP 443.
  - MECM SQL Server on the configured SQL port.
  - Each configured MP, DP, primary site, and SQL endpoint.
- A domain gMSA is recommended for the scheduled collector.
- The collector identity must have:
  - MECM RBAC read access through AdminService. Add it as a Configuration
    Manager administrative user, normally with the built-in **Read-only
    Analyst** role and appropriate security scopes.
  - Existing SQL read/`SELECT` access to the required reporting views.
- A valid certificate on the SMS Provider/AdminService endpoint. The collector
  does not disable certificate validation.

If the scheduled task runs as `SYSTEM`, remote systems see the IIS monitor
server's domain computer account, for example `CONTOSO\MECMMON01$`. That
computer account—not your interactive user account—must already have the
required AdminService and SQL access. A gMSA is easier to audit.

## 2. Confirm existing SQL read access

If the account that will run the scheduled task already has read access to the
MECM database, **do not grant any additional SQL permissions**. The installer
does not create SQL logins, database users, roles, or permissions, and it never
runs `Grant-Reporting-Access.example.sql`.

Confirm that the scheduled-task identity can read:

- `dbo.v_R_System`
- `dbo.v_CH_ClientSummary`
- `dbo.v_CH_EvalResults`
- `dbo.v_StateNames`
- `dbo.v_SMS_Alert` (optional fallback for active alerts)
- `dbo.v_ContentDistributionReport_DP` or `dbo.v_DistributionStatus`
- `dbo.v_ContentDistributionMessages` (optional content error text)
- `dbo.v_Package` or `dbo.v_SMSPackage` (optional friendly content name/type)
- `dbo.v_TaskExecutionStatus`
- `dbo.v_Advertisement` (optional task-sequence deployment name)
- `dbo.v_ClientDeploymentState`

Run `Queries\Verify-Reporting-Access.sql` while impersonating or signing in as
the collector identity to confirm read access.

`Queries\Grant-Reporting-Access.example.sql` is optional reference material for
a SQL administrator only when a new collector identity does not already have
access. Skip it when your account already has the required read permissions.

## 3. Verify AdminService

From the monitor server, sign in as the collector identity and test:

```powershell
Invoke-RestMethod `
  -Uri "https://smsprovider.contoso.com/AdminService/wmi/SMS_Site" `
  -UseDefaultCredentials
```

The request must return JSON without a credential prompt or certificate error.
AdminService is installed on an SMS Provider and uses HTTPS. Microsoft documents
the endpoint and validation procedure here:

- <https://learn.microsoft.com/en-us/intune/configmgr/develop/adminservice/set-up>
- <https://learn.microsoft.com/en-us/intune/configmgr/develop/adminservice/overview>

## 4. Install the IIS package

Extract the ZIP on the IIS monitor server. Open an elevated Windows PowerShell
console in the extracted folder.

If your existing domain account already has both SQL and MECM AdminService read
access, install the scheduled task under that account:

```powershell
$credential = Get-Credential "CONTOSO\YourUserName"

.\Install-MECMMonitor.ps1
  -TaskCredential $credential
  -Port 8088
  -CollectionIntervalMinutes 2

The task will use the supplied account's existing permissions. Be aware that a
personal-account password change will require updating the scheduled-task
credentials.

Recommended long-term gMSA installation:

powershell
.\Install-MECMMonitor.ps1 
  -TaskUser "CONTOSO\MECMMonitorSvc$" 
  -Port 8088 
  -CollectionIntervalMinutes 2

For a normal domain service account:

powershell
$credential = Get-Credential "CONTOSO\MECMMonitorSvc"

.\Install-MECMMonitor.ps1
  -TaskCredential $credential
  -Port 8088 `
  -CollectionIntervalMinutes 2

The installer:

- installs the required IIS features;
- deploys the static website to `C:\inetpub\MECMHealthMonitor`;
- deploys collector/config files to `C:\ProgramData\MECMHealthMonitor`;
- enables IIS Windows Authentication by default;
- creates a Domain-profile firewall rule restricted to `LocalSubnet`;
- creates the scheduled task `MECM Health Monitor - Collect`;
- creates and validates the initial snapshot.

## 5. Edit only MECMMonitor.xml

Edit:

```text
C:\ProgramData\MECMHealthMonitor\MECMMonitor.xml
```

Replace the example values and set the required entries to `Enabled="true"`.

```xml
<General>
  <Title>MECM Command Center</Title>
  <EnvironmentName>Production MECM</EnvironmentName>
  <PrimarySiteCode>ABC</PrimarySiteCode>
  <RefreshSeconds>30</RefreshSeconds>
  <StaleAfterMinutes>10</StaleAfterMinutes>
  <ClientPageSize>100</ClientPageSize>
</General>

<AdminService Enabled="true"
              ProviderFqdn="abc-smsprovider.contoso.com"
              Port="443"
              TimeoutSeconds="45"
              ComponentTallyInterval="0001128000100008"
              MaxDeployments="500"
              MaxAlerts="200" />

<Sql Enabled="true"
     Server="abc-sql01.contoso.com"
     Instance=""
     Port="1433"
     Database="CM_ABC"
     Encrypt="true"
     TrustServerCertificate="false"
     CommandTimeoutSeconds="120"
     MaxClients="0"
     MaxDetailRows="10000" />
```

`MaxClients="0"` means collect all managed clients. Set a positive value only
when you intentionally want a collection limit. `MaxDetailRows` limits each
per-DP distribution, OSD-step-error, and client-install-error query. Increase it
if a large estate needs more history; the browser shows the first 250 detailed
rows for responsiveness, while CSV export includes every collected row.

Add one enabled server entry for every endpoint you want probed:

```xml
<Infrastructure>
  <Server Enabled="true"
          Name="ABC-CM01"
          Fqdn="abc-cm01.contoso.com"
          Role="Primary Site Server"
          SiteCode="ABC"
          Port="443" />
  <Server Enabled="true"
          Name="ABC-SQL01"
          Fqdn="abc-sql01.contoso.com"
          Role="SQL Database"
          SiteCode="ABC"
          Port="1433" />
  <Server Enabled="true"
          Name="ABC-MP01"
          Fqdn="abc-mp01.contoso.com"
          Role="Management Point"
          SiteCode="ABC"
          Port="443" />
  <Server Enabled="true"
          Name="ABC-DP01"
          Fqdn="abc-dp01.contoso.com"
          Role="Distribution Point"
          SiteCode="ABC"
          Port="443" />
</Infrastructure>
```

Do not add credentials or operational values to XML.

## 6. Generate and test the first live snapshot

powershell
Start-ScheduledTask -TaskName "MECM Health Monitor - Collect"

while ((Get-ScheduledTask -TaskName "MECM Health Monitor - Collect").State -eq "Running") {
    Start-Sleep -Seconds 2
}

& "C:\ProgramData\MECMHealthMonitor\Collector\Test-MECMMonitor.ps1" 
  -ConfigPath "C:\ProgramData\MECMHealthMonitor\MECMMonitor.xml" 
  -CollectorPath "C:\ProgramData\MECMHealthMonitor\Collector\Collect-MECMHealth.ps1"

Review:

text
C:\ProgramData\MECMHealthMonitor\Logs\collector.log
C:\inetpub\MECMHealthMonitor\data\dashboard.json


Open:

text
http://MECMMON01:8088


## 7. Refresh behavior

- The scheduled task queries MECM every two minutes by default.
- The website reloads the latest JSON according to `RefreshSeconds`.
- The **Refresh** button immediately reloads the latest completed snapshot.
- **Export CSV** downloads the selected workload without querying SQL from the
  browser. Select **All monitoring data** for one combined export.
- The clock updates every second in the browser.
- A stale warning appears after `StaleAfterMinutes`.

This is near-real-time monitoring. MECM deployment and component summarizers
update on their own MECM schedules, so the dashboard cannot be newer than the
source summary.

## Distribution, OSD, and client-install error logic

The Software page shows one content-status row per application/package and
distribution point when the reporting views are available. Failed and retrying
content is sorted first. The latest available content message supplies the
error code and message; the dashboard leaves these fields blank when MECM did
not report them.

The OSD page shows failed task-sequence action records with the device,
deployment/package identifiers, failed action name, group, execution time,
action output, and exit code. Numeric error codes are displayed in decimal and
hexadecimal to make log and Microsoft-documentation searches easier.

The Client Health page adds failed client-push/client-install records from
`v_ClientDeploymentState`. It resolves the deployment state through
`v_StateNames` topic type 800 and displays an explicit error/exit code when
MECM supplies one; otherwise it displays the MECM state ID.

These are optional detailed sources. If one view is absent or unreadable, the
collector records the exact reason under **Site Systems → Collector
diagnostics** and continues generating the rest of the dashboard.

## Client error logic

Every active, non-obsolete MECM client returned by `v_R_System` is included.
The collector merges:

- the client active state;
- client-health summary state names;
- health evaluation result/error codes;
- the most recent available online/active/evaluation time.

Critical and warning clients are sorted first. Up to three distinct reported
issues are shown for each client. If MECM has no client-health summary for a
device, the dashboard displays `Unknown` with that reason rather than inventing
a healthy state.

## Troubleshooting

### AdminService returns 401 or 403

- Run the test request as the scheduled-task identity.
- Confirm the identity is a MECM administrative user with read permissions.
- Confirm Windows Integrated Authentication can be delegated/reached from the
  monitor server.
- Review `adminservice.log` and `SMS_REST_PROVIDER.log` on the SMS Provider.

### SQL login failed

- Confirm the scheduled-task identity has a SQL login and database user.
- Confirm read permissions by running `Verify-Reporting-Access.sql`.
- For a named instance, set both `Instance` and the actual SQL port.

### Certificate trust failed

Trust the issuing CA on the monitor server. Keep
`TrustServerCertificate="false"` for production. Set it to `true` only as a
temporary, approved diagnostic measure.

### Dashboard is blank

- Read `collector.log`.
- Confirm AdminService and SQL are set to `Enabled="true"`.
- Start the collection task manually.
- Confirm `dashboard.json` has a recent `generatedAtUtc`.
- Open the **Site Systems** page and review Collector diagnostics.

### A detailed table is empty

- Run `Queries\Verify-Reporting-Access.sql` as the scheduled-task service
  account.
- Review the matching SQL source under Collector diagnostics.
- Confirm `MaxDetailRows` is at least 100.
- Remember that a table with no current failures can correctly be empty.

## Update and removal

Rerunning the installer updates program and web files but preserves the installed
XML configuration. If upgrading from the old manual-data version, compare the
preserved file with the new `Config\MECMMonitor.xml`; manual sections are ignored.

To remove the monitor:

```powershell
.\Uninstall-MECMMonitor.ps1
```

Use `-RemoveData` only if you also want to delete the installed XML, collector
logs, and data directory.

## Microsoft references

- AdminService setup: <https://learn.microsoft.com/en-us/intune/configmgr/develop/adminservice/set-up>
- Deployment summary class: <https://learn.microsoft.com/en-us/intune/configmgr/develop/reference/apps/sms_deploymentsummary-server-wmi-class>
- Active alert class: <https://learn.microsoft.com/en-us/intune/configmgr/develop/reference/core/servers/manage/sms_alert-server-wmi-class>
- Component summarizer class: <https://learn.microsoft.com/en-us/intune/configmgr/develop/reference/core/servers/manage/sms_componentsummarizer-server-wmi-class>
- Client-status reporting views: <https://learn.microsoft.com/en-us/intune/configmgr/develop/core/understand/sqlviews/client-status-views-configuration-manager>
- Content-management reporting views: <https://learn.microsoft.com/en-us/intune/configmgr/develop/core/understand/sqlviews/content-management-views-configuration-manager>
- OSD reporting views: <https://learn.microsoft.com/en-us/intune/configmgr/develop/core/understand/sqlviews/operating-system-deployment-views-configuration-manager>
- Client deployment status query: <https://learn.microsoft.com/en-us/intune/configmgr/develop/core/understand/sqlviews/sample-queries-client-deployment-configuration-manager>

