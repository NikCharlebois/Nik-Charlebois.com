---
title: Converting an M365DSC Configuration into a UTCM Baseline
---

<h1 class="blog-title">Converting an M365DSC Configuration into a UTCM Baseline</h1>
<div class="article-date">2026-07-29</div>

<p>If you have been using Microsoft365DSC for a while, you probably already have a rich configuration file that represents the desired state of your Microsoft 365 tenant. With the Tenant Configuration Management APIs, also referred to as Unified Tenant Configuration Management or UTCM, that same configuration-as-code approach can now be used directly through Microsoft Graph baselines and monitors.</p>

<p>The good news is that you do not need to rewrite every resource by hand. The <a href="https://github.com/microsoft/M365DSC-to-TCM">M365DSC-to-TCM utility</a> can parse an extracted Microsoft365DSC configuration and convert the supported resources into a UTCM baseline JSON template. In this post, I will walk through the process step by step, starting with an existing M365DSC export and ending with a baseline file that can be used with the TCM monitor APIs.</p>

<h2>What the utility does</h2>

<p>The converter is a PowerShell module that exposes the <code>ConvertFrom-M365DSCToTCM</code> command. It reads a Microsoft365DSC configuration, uses the DSCParser module to convert the DSC resources into objects, maps supported M365DSC resource names to UTCM resource types, and emits a JSON baseline template.</p>

<p>For example, an M365DSC resource such as <code>AADConditionalAccessPolicy</code> is mapped to the UTCM resource type <code>microsoft.entra.conditionalaccesspolicy</code>. An Exchange transport rule becomes <code>microsoft.exchange.transportrule</code>. The supported mappings are defined in the utility's <code>M365DSCToTCMMappings.psd1</code> file, so the first validation step after conversion should always be to confirm that the resources you care about made it into the output.</p>

<h2>Step 1: Start from an extracted M365DSC configuration</h2>

<p>The input file should be the PowerShell configuration file produced by Microsoft365DSC. For example, you may already have a file such as:</p>

```text
C:\Temp\M365DSC\Contoso.ps1
```

<p>Before converting it, take a quick look at the file and make sure it is a straightforward exported configuration. The current converter is intentionally focused on exported DSC resources and does not support every possible PowerShell construct.</p>

<p>In particular, avoid using an input configuration that includes:</p>

<ul>
<li>PowerShell conditional logic such as <code>if</code> or <code>else</code>.</li>
<li>Looping logic such as <code>for</code>, <code>foreach</code>, or <code>while</code>.</li>
<li>DSC composite resources.</li>
<li>Non-string variables.</li>
<li>Null values in resource properties.</li>
</ul>

<p>If your extracted file contains post-processing logic that you added manually, I recommend keeping a clean exported copy and converting that version first. You can always add UTCM-specific cleanup or parameterization after the first conversion succeeds.</p>

<h2>Step 2: Download the converter</h2>

<p>Clone the M365DSC-to-TCM repository locally. The examples below use <code>C:\Temp</code>, but any working folder is fine.</p>

```powershell
git clone https://github.com/microsoft/M365DSC-to-TCM.git C:\Temp\M365DSC-to-TCM
```

<p>The repository contains the module manifest, the PowerShell module, the mapping file, and the base UTCM template used during conversion:</p>

```text
M365DSCToTCM.psd1
M365DSCToTCM.psm1
M365DSCToTCMMappings.psd1
TCMTemplate.json
```

<h2>Step 3: Install the required parser module</h2>

<p>The converter depends on <code>DSCParser</code> version <code>3.0.0.5</code>. Install it from the PowerShell Gallery if it is not already present on your workstation.</p>

```powershell
Install-Module DSCParser -RequiredVersion 3.0.0.5 -Scope CurrentUser
```

<p>If your environment requires a trusted repository prompt, approve the PowerShell Gallery prompt or configure the repository trust according to your organization's policy.</p>

<h2>Step 4: Import the M365DSC-to-TCM module</h2>

<p>Import the module from the folder you cloned in the previous step:</p>

```powershell
Import-Module C:\Temp\M365DSC-to-TCM\M365DSCToTCM.psd1 -Force
```

<p>You can confirm that the conversion command is available with:</p>

```powershell
Get-Command ConvertFrom-M365DSCToTCM
```

<h2>Step 5: Convert the M365DSC file to a UTCM baseline</h2>

<p>Now run the converter and write the generated JSON to disk:</p>

```powershell
$sourcePath = "C:\Temp\M365DSC\Contoso.ps1"
$targetPath = "C:\Temp\UTCM\Contoso-Baseline.json"

New-Item -ItemType Directory -Path (Split-Path $targetPath) -Force | Out-Null

$json = ConvertFrom-M365DSCToTCM -Path $sourcePath -Parameterize $true
$json | Out-File -FilePath $targetPath -Encoding utf8
```

<p>The <code>-Parameterize</code> switch is enabled by default, but I like to include it explicitly because it makes the intent clear. When parameterization is enabled, variables found in the source configuration are added to the UTCM baseline's <code>parameters</code> collection and are referenced from resource properties using UTCM expressions.</p>

<p>For example, a value that is only the variable <code>$FQDN</code> becomes:</p>

```json
"[parameters('FQDN')]"
```

<p>A value that combines text and variables can become a <code>concat</code> expression:</p>

```json
"[concat('admin@', parameters('FQDN'))]"
```

<p>If you want the converter to preserve the literal values instead, run it with parameterization disabled:</p>

```powershell
$json = ConvertFrom-M365DSCToTCM -Path $sourcePath -Parameterize $false
```

<h2>Step 6: Inspect the generated baseline</h2>

<p>Load the output file back into PowerShell and inspect the high-level structure:</p>

```powershell
$baseline = Get-Content $targetPath -Raw | ConvertFrom-Json

$baseline.displayName
$baseline.description
$baseline.parameters
$baseline.resources | Select-Object displayName, resourceType
```

<p>The generated file follows the UTCM baseline shape. At the top level, you should see metadata such as <code>displayName</code> and <code>description</code>, a <code>parameters</code> collection if variables were discovered, and a <code>resources</code> collection containing the converted configuration resources.</p>

<p>A simplified resource in the output will look similar to this:</p>

```json
{
  "displayName": "Global",
  "resourceType": "microsoft.teams.meetingpolicy",
  "properties": {
    "Identity": "Global",
    "AllowAnonymousUsersToStartMeeting": false,
    "AllowCloudRecording": true,
    "Ensure": "Present"
  }
}
```

<h2>Step 7: Check for unsupported resources</h2>

<p>The converter only emits resources that exist in its mapping file. If an M365DSC resource is not mapped to a UTCM resource type, it will not appear in the generated baseline. That is why this step is important: you want to know whether the baseline is complete enough for the scenario you are building.</p>

<p>You can compare the extracted resource names with the converter's mapping file by running the following commands from the converter folder:</p>

```powershell
Set-Location C:\Temp\M365DSC-to-TCM

$content = Get-Content $sourcePath -Raw
$parsedResources = ConvertTo-DSCObject -Content $content -IncludeCIMInstanceInfo $false
$mappings = Import-PowerShellDataFile .\M365DSCToTCMMappings.psd1

$parsedResources |
    Where-Object { -not $mappings.ContainsKey($_.ResourceName) } |
    Select-Object ResourceName -Unique
```

<p>If the command returns nothing, every resource type in the extracted file has a mapping. If it returns one or more resource names, those resource types need to be reviewed separately. Depending on the scenario, you may decide to remove them from the source configuration, wait for UTCM coverage, or manually author equivalent UTCM resources if the resource type is supported by the APIs but not yet mapped by the utility.</p>

<h2>Step 8: Customize the baseline metadata</h2>

<p>The utility starts from a sample template, so you will usually want to update the baseline name and description before using it in a monitor or storing it in source control.</p>

```powershell
$baseline = Get-Content $targetPath -Raw | ConvertFrom-Json

$baseline.displayName = "Contoso Standard Configuration Baseline"
$baseline.description = "Baseline converted from the Microsoft365DSC export for Contoso."

$baseline |
    ConvertTo-Json -Depth 99 |
    Set-Content -Path $targetPath -Encoding utf8
```

<p>This is also a good point to review whether the generated parameters have friendly descriptions. The converter creates parameter entries from variables it finds in the source configuration, but you may want to make those descriptions more meaningful before sharing the baseline with other administrators.</p>

<h2>Step 9: Validate the JSON file</h2>

<p>At a minimum, validate that the file is well-formed JSON and that every resource has the fields UTCM expects:</p>

```powershell
$baseline = Get-Content $targetPath -Raw | ConvertFrom-Json

$baseline.resources |
    Where-Object {
        [string]::IsNullOrWhiteSpace($_.displayName) -or
        [string]::IsNullOrWhiteSpace($_.resourceType) -or
        $null -eq $_.properties
    } |
    Select-Object displayName, resourceType
```

<p>If that command returns no rows, each converted resource has a display name, a UTCM resource type, and a properties object. That does not guarantee that every workload-specific property value is perfect, but it does catch the most obvious structural issues before you send the baseline to Microsoft Graph.</p>

<h2>Step 10: Use the baseline with a UTCM monitor</h2>

<p>Once you are happy with the generated baseline, the next important thing to understand is where that JSON belongs in the API request. The file produced by the converter is the content of the <code>baseline</code> property of the monitor object. It is not, by itself, the complete JSON body you send to the TCM APIs.</p>

<p>To create a monitor from the converted baseline, wrap the converted JSON inside the full monitor payload. The monitor-level object contains properties such as <code>displayName</code>, <code>description</code>, and optionally <code>parameters</code>. The converted UTCM baseline goes under the monitor object's <code>baseline</code> property.</p>

```http
POST https://graph.microsoft.com/v1.0/admin/configurationManagement/configurationMonitors
```

```json
{
  "displayName": "Contoso baseline monitor",
  "description": "Monitor created from a Microsoft365DSC-converted UTCM baseline.",
  "baseline": {
    "displayName": "Contoso Standard Configuration Baseline",
    "description": "Baseline converted from the Microsoft365DSC export for Contoso.",
    "parameters": [],
    "resources": []
  }
}
```

<p>In the example above, the <code>baseline</code> object is shortened for readability. In your real request, replace that shortened object with the full JSON object generated by the converter. In other words, do not send only the converted file as the request body. Load the converted file, assign it to the <code>baseline</code> property, and then send the complete monitor payload to Microsoft Graph.</p>

```powershell
$baseline = Get-Content "C:\Temp\UTCM\Contoso-Baseline.json" -Raw | ConvertFrom-Json

$monitorPayload = @{
    displayName = "Contoso baseline monitor"
    description = "Monitor created from a Microsoft365DSC-converted UTCM baseline."
    baseline = $baseline
}

$body = $monitorPayload | ConvertTo-Json -Depth 99
```

<p>If the converted baseline includes parameters, provide the parameter values at the monitor level. For example:</p>

```json
{
  "displayName": "Contoso baseline monitor",
  "description": "Monitor created from a parameterized converted baseline.",
  "baseline": {
    "displayName": "Contoso Standard Configuration Baseline",
    "description": "Baseline converted from the Microsoft365DSC export for Contoso.",
    "parameters": [
      {
        "displayName": "FQDN",
        "description": "The fully qualified domain name for the tenant.",
        "parameterType": "String"
      }
    ],
    "resources": []
  },
  "parameters": {
    "FQDN": "contoso.onmicrosoft.com"
  }
}
```

<p>Remember that monitor execution depends on the Unified Tenant Configuration Management service principal having the right workload permissions and roles for the resource types in your baseline. If the monitor returns <strong>failed</strong> or <strong>partiallySuccessful</strong>, query the monitor result with <code>$select=errorDetails</code> so you can see the specific resource type and error message.</p>

<h2>Putting it all together</h2>

<p>Here is the complete conversion flow in one script:</p>

```powershell
$converterPath = "C:\Temp\M365DSC-to-TCM"
$sourcePath = "C:\Temp\M365DSC\Contoso.ps1"
$targetPath = "C:\Temp\UTCM\Contoso-Baseline.json"

Install-Module DSCParser -RequiredVersion 2.0.0.14 -Scope CurrentUser
Import-Module "$converterPath\M365DSCToTCM.psd1" -Force

New-Item -ItemType Directory -Path (Split-Path $targetPath) -Force | Out-Null

$json = ConvertFrom-M365DSCToTCM -Path $sourcePath -Parameterize $true
$json | Out-File -FilePath $targetPath -Encoding utf8

$baseline = Get-Content $targetPath -Raw | ConvertFrom-Json
$baseline.displayName = "Contoso Standard Configuration Baseline"
$baseline.description = "Baseline converted from the Microsoft365DSC export for Contoso."

$baseline |
    ConvertTo-Json -Depth 99 |
    Set-Content -Path $targetPath -Encoding utf8

$baseline.resources | Select-Object displayName, resourceType
```

<p>The M365DSC-to-TCM utility gives existing Microsoft365DSC users a practical bridge into UTCM. It will not replace the need to review the generated baseline, validate resource coverage, and assign the required service principal permissions, but it dramatically reduces the amount of manual translation needed to get started.</p>

<p>My recommendation is to treat the converted file as a strong first draft: generate it from a clean M365DSC export, inspect the supported resources, refine parameters and metadata, and then use it as the baseline for a UTCM monitor. From there, your existing configuration-as-code investment can become part of the newer Microsoft Graph-based tenant configuration management workflow.</p>
