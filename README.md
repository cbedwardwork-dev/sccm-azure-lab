# Configuration Manager Current Branch lab in Azure (patched)

A fork of the Azure Quickstart template
[`application-workloads/sccm/sccm-currentbranch`](https://github.com/Azure/azure-quickstart-templates/tree/master/application-workloads/sccm/sccm-currentbranch),
fixed so it deploys end to end in 2026, including into subscriptions under Azure Landing Zone policies.

[![Deploy To Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FOWNER%2FREPO%2Fmain%2Fazuredeploy.json)

> After pushing, replace `OWNER` and `REPO` in the button link above with your GitHub user/org and repo name.
> The repo must be **public** (or see "Private repo" below), because the VMs download the DSC scripts from it.

## What it builds

* AD domain controller (DC01)
* Standalone primary site (PS01) with SQL Server 2019, ADK and ConfigMgr, upgraded to the latest current branch release
* Remote site system (DPMP01) with management point and distribution point
* Windows 11 client(s)

No VM has a public IP. Connect with **Azure Bastion (Developer SKU)**: VM → Connect → Bastion.
Sign in as **`<domain>\<adminUsername>`** (e.g. `contoso\CBAdmin`), not the local account.

## Changes from the upstream template

### `azuredeploy.json`

| Problem upstream | Fix |
|---|---|
| Default sizes `Standard_B2s` / `Standard_B2ms` (B-series v1) are retirement-announced and blocked for new subscriptions | Defaults `Standard_D2ads_v5` (DC, DP/MP, client) and `Standard_D4ads_v5` (site server + SQL). Both have a temp disk |
| Basic-SKU dynamic public IP per VM (Basic public IPs are retired) | Public IPs removed |
| NSG allowed RDP 3389 from `*` (blocked by ALZ *Deny-MgmtPorts-Internet*) | No inbound rules; Bastion Developer reaches VMs via the platform's built-in allow rule |
| NSG attached to the subnet after VNet creation (blocked by ALZ *Deny-Subnet-Without-Nsg*) | NSG associated inline in the subnet definition, VNet depends on NSG |
| No outbound internet once public IPs are gone (new VNets default to private subnets) | Subnet sets `defaultOutboundAccess: true` (VNet API 2024-05-01) |
| Client image `win11-23h2-ent` (Enterprise 23H2 servicing ended Nov 2026) | `win11-24h2-ent` |

### DSC scripts (`DSC/` and `DSC/DCConfiguration.zip`)

| Problem upstream | Fix |
|---|---|
| Installs ODBC Driver 18 **18.1.x** and accepts anything ≥ 18.1.2.1. ConfigMgr 2503+ requires **≥ 18.4.1.1**, so the in-console upgrade fails its prerequisite check and the build stops before DP/MP/client | `DownloadAndInstallODBC` installs the VC++ 2015-2022 x64 runtime, then the ODBC 18 driver from the link the 2509 prereq checker uses; `Test()` requires ≥ 18.4.1.1 and rejects 18.6.1.x (known ConfigMgr issues) |
| Update with prerequisite **warnings** sits at `PREREQ_WARNING` forever and the script waits forever | Sets `SetIgnorePrereqWarning(2)` on the update package before installing, and re-kicks `Install-CMSiteUpdate` if it is still parked on `PREREQ_WARNING` |
| `SetCustomPagingFile` failure fails the whole VM extension | Wrapped in try/catch; on failure it restores the system-managed page file and marks itself compliant, so DSC doesn't reboot-loop |
| Re-running the `ScriptWorkFlow` scheduled task re-runs ConfigMgr setup on an installed site | Setup is skipped if the site is already installed; DP, MP and client steps are skipped if already `Completed` in `C:\Windows\Temp\ProvisionScript\PS1.json` |

`DCConfiguration.zip` is what the VMs actually download. The loose `.ps1` files in `DSC/` are the same content, kept for readability.
**If you edit a script, rebuild the zip** (see below), or your change won't be deployed.

## Deploying

### Option 1: Deploy to Azure button (recommended)
Uses `_artifactsLocation = [deployment().properties.templateLink.uri]`, so the DSC zip is found automatically next to `azuredeploy.json`. Leave **_artifacts Location Sas Token** blank.

### Option 2: Custom template upload / CLI with a local file
There's no template link in this case, so set `_artifactsLocation` explicitly:

```powershell
New-AzResourceGroupDeployment -ResourceGroupName rg-sccmlab -TemplateFile .\azuredeploy.json `
  -prefix sccm01 -adminUsername CBAdmin -_artifactsLocation 'https://raw.githubusercontent.com/OWNER/REPO/main/'
```

(Note the trailing `/`.)

### Private repo
Raw GitHub URLs for private repos need a token the DSC extension can't use. Instead, copy `DSC/DCConfiguration.zip` to a blob container (keeping the `DSC/` folder name), set `_artifactsLocation` to the container URL with a trailing `/`, and put a container SAS token (starting with `?`) in `_artifactsLocationSasToken`.

### Before you deploy
* **Quota:** standalone with 1 client = 4 VMs = **10 vCPU** of *Standard DASv5 Family* in the target region.
* **Region:** any region with Dadsv5 capacity; check with `Get-AzComputeResourceSku` or the portal size picker.
* Build time is roughly 2–3 hours. Progress logs on PS01: `C:\Windows\Temp\ProvisionScript\*.txt`, status in `PS1.json`.

## Rebuilding `DCConfiguration.zip` after editing scripts

From the repo root in PowerShell:

```powershell
Push-Location .\DSC
Compress-Archive -Path *.ps1, TemplateHelpDSC -DestinationPath .\DCConfiguration.zip -Force
Pop-Location
```

## Known limitations

* **This patched version hasn't been deployed end to end yet.** The fixes come from a real Standalone build of the upstream template (where each problem was hit and fixed by hand), but the first run of this fork is its real test. Watch the logs on PS01 on that first build.
* The Hierarchy configuration has the same fixes applied but hasn't been tested at all.
* `defaultOutboundAccess` is Microsoft's legacy outbound method. If your org blocks it, add a NAT gateway to the subnet.
* The landing-zone fixes cover the two policies hit so far (*Deny-Subnet-Without-Nsg*, *Deny-MgmtPorts-Internet*). Other org policies, such as required tags or allowed locations, may still need tweaks.
