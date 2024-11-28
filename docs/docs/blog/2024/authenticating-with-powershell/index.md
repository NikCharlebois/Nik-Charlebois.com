---
title: Authenticating to Microsoft 365 with PowerShell
...
date:
  created: 2024-11-28
readtime: 10
---

<h1>Authenticating to Microsoft 365 with PowerShell</h1>

<p>In today's world, administrators must have about a dozen different PowerShell modules under their tool belt. It seems as if every workload out there has its own PowerShell module to allow you to administer its related configuration settings; add to this the fact that every module has its own intricacy for how it let's you authenticate. For Entra Id and Intune we have the Microsoft Graph PowerShell SDK, for Teams, we have the Teams PowerShell module, for Exchange Online, there's the Exchange Online Management Shell, etc. To help simplify the overall authentication process, we put together the <a href="https://GitHub.com/Microsoft/MSCloudLoginAssistant">MSCloudLoginAssistant</a> module, which acts as an abstraction layer sitting on top of all these modules and provies a coherent and streamlined way to authenticate to them.</p>

<p>In today's article, I want to spend time walking down the authentication stack for each of the major workloads that are supported by <a href="https://Microsoft365DSC.com">Microsoft365DSC</a>. For each one, we will describe how to authenticate using MSCloudLoginAssistant, but we'll also double click on how this module in turns authenticate to the lower layers by calling the workloads' specific PowerShell module directly. As part of this article, we will only focus on the Service Principal with certificate thumbprint flow. We will not cover the process of configuring a new App Registration in Entra ID as part of the article as therer are plenty of literature on the topic out there. We simply assume that you created your own app registration, and uploaded a certificate's public key to it, and that you have access to install the associated private key on your mahcine.</p>

<p>For each workload we will provide the instructions to connect via the MSCloudLoginAssistant module, via its native PowerShell module and we will provide additional information about how you can leverage Microsoft365DSC to have the Local Configuration Manager (LCM) service authenticate. It is important to understand that the LCM always runs in the context of the Local System and therefore having the certificate stored in the current user's store is not sufficient in a lot of cases.</p>

<h2>Microsoft Graph (Entra Id and Intune)</h2>
<p>Both of the Entra Id and Intune workloads in Microsoft365DSC are leveraging the various <a href="https://www.powershellgallery.com/packages/Microsoft.Graph.Authentication/">Microsoft.Graph.*</a> PowerShell modules. In order to authenticate with the Microsoft Graph PowerShell module, you simply need to have the certificate stored in your current user's store.</p>

<img src="/blog/2024/authenticating-with-powershell/images/certlocaluser.png" alt="Certificate in the local user's store." />

<h3>Via MSCloudLoginAssistant</h3>

``` powershell
Connect-M365Tenant -Workload 'MicrosoftGraph' `
                   -ApplicationId '<your app id>' `
                   -TenantId '<your tenant>.onmicrosoft.com' `
                   -CertificateThumbprint '<your thumbprint>'
```
<img src="/blog/2024/authenticating-with-powershell/images/graphmscloud.png" alt="Connecting to Microsoft Graph via MSCloudLoginAssistant." />

<h3>Via Microsoft.Graph.Authentication</h3>

``` powershell
Connect-MgGraph -ClientId '<your app id>' `
                -TenantId '<your tenant>.onmicrosoft.com' `
                -CertificateThumbprint '<your thumbprint>'
```
<img src="/blog/2024/authenticating-with-powershell/images/graphwithmodule.png" alt="Connecting to Microsoft Graph via Microsoft Graph PowerShell SDK." />

<h3>Local Configuration Management</h3>
<p>Because the LCM service runs as the local system, we need to install the certificate in the Local Computer's store as well. If you omit to install the certificate in there as well, the LCM will not be able to authenticate to the Microsoft Graph.</p>
<img src="/blog/2024/authenticating-with-powershell/images/localmachine.png" alt="Installing the certificate in the Local Machine's store." />

<h2>Exchange Online</h2>
<p>Managing Exchange Online is done via the <a href="https://www.powershellgallery.com/packages/ExchangeOnlineManagement/">ExchangeOnlineManagement</a> module. It is sufficient for you to have the certificate in the current user's store to authenticate. However, in this case, the associated application will also need to be granted the <a href="https://learn.microsoft.com/en-us/powershell/exchange/app-only-auth-powershell-v2?view=exchange-ps#select-and-assign-the-api-permissions-from-the-portal">Manage as App</a> permission, and an Entra Id role sufficient to manage Exchange. Refer to the official Exchange Online documentation for additional details.</p>

<h3>Via MSCloudLoginAssistant</h3>

``` powershell
Connect-M365Tenant -Workload 'ExchangeOnline' `
                   -ApplicationId '<your app id>' `
                   -TenantId '<your tenant>.onmicrosoft.com' `
                   -CertificateThumbprint '<your thumbprint>'
```

<img src="/blog/2024/authenticating-with-powershell/images/exomscloud.png" alt="Connecting to Exchange Online via MSCloudLoginAssistant." />

<h3>Via ExchangeOnlineManagement</h3>

``` powershell
Connect-ExchangeOnline -AppId '<your app id>' `
                       -Organization '<your tenant>.onmicrosoft.com' `
                       -CertificateThumbprint '<your thumbprint>'
```
<img src="/blog/2024/authenticating-with-powershell/images/exomodule.png" alt="Connecting to Exchange Online via ExchangeOnlineManagement." />

<h2>Security and Compliance (Defender & Purview)</h2>
<p>Managing Defender and Purview is also is done via the <a href="https://www.powershellgallery.com/packages/ExchangeOnlineManagement/">ExchangeOnlineManagement</a> module. Again, it is sufficient for you to have the certificate in the current user's store to authenticate. However, in this case, the associated application will also need to be granted the <a href="https://learn.microsoft.com/en-us/powershell/exchange/app-only-auth-powershell-v2?view=exchange-ps#select-and-assign-the-api-permissions-from-the-portal">Manage as App</a> permission, and an Entra Id role sufficient to manage Defender & Purview. Refer to the official Security & Compliance center documentation for additional details.</p>

<h3>Via MSCloudLoginAssistant</h3>

``` powershell
Connect-M365Tenant -Workload 'SecurityComplianceCenter' `
                   -ApplicationId '<your app id>' `
                   -TenantId '<your tenant>.onmicrosoft.com' `
                   -CertificateThumbprint '<your thumbprint>'
```

<img src="/blog/2024/authenticating-with-powershell/images/scmscloud.png" alt="Connecting to Security and Compliance via MSCloudLoginAssistant." />

<h3>Via ExchangeOnlineManagement</h3>

``` powershell
Connect-IPPSSession -AppId '<your app id>' `
                    -Organization '<your tenant>.onmicrosoft.com' `
                    -CertificateThumbprint '<your thumbprint>'
```
<img src="/blog/2024/authenticating-with-powershell/images/scmodule.png" alt="Connecting to Security and Compliance Center via ExchangeOnlineManagement." />

<h3>Local Configuration Management</h3>
<p>LCM requires the certificate to be installed in the Local Machine's store in order to authenticate.</p>

<h2>SharePoint Online</h2>
<p>In the case of Microsoft365DSC, the authentication to SharePoint Online is done via the <a href="https://www.powershellgallery.com/packages/PnP.PowerShell/">PnP.PowerShell</a> module. It is sufficient to have the certificate u=in the current user's store to authenticate. You will also need to make sure that the app registration is granted the <strong>Sites.FullControl.All</strong> permission for the <strong>SharePoint API</strong> (not Microsoft Graph!). </p>

<h3>Via MSCloudLoginAssistant</h3>

``` powershell
Connect-M365Tenant -Workload 'PnP' `
                   -ApplicationId '<your app id>' `
                   -TenantId '<your tenant>.onmicrosoft.com' `
                   -CertificateThumbprint '<your thumbprint>'
```

<img src="/blog/2024/authenticating-with-powershell/images/pnpmscloud.png" alt="Connecting to SharePoint Online via MSCloudLoginAssistant." />

<h3>Via PnP.PowerShell</h3>

``` powershell
Connect-PnPOnline -ClientId '<your app id>' `
                  -Tenant '<your tenant>.onmicrosoft.com' `
                  -Thumbprint '<your thumbprint>' `
                  -Url 'https://<your tenant>-admin.sharepoint.com'
```
<img src="/blog/2024/authenticating-with-powershell/images/pnpmodule.png" alt="Connecting to SharePoint Online via PnP.PowerShell." />

<h3>Local Configuration Management</h3>
<p>LCM requires the certificate to be installed in the Local Machine's store in order to authenticate.</p>

<h2>Power Platforms</h2>
<p>Authentication to Power Platforms is handled by the <a href="https://www.powershellgallery.com/packages/Microsoft.PowerApps.Administration.PowerShell/">Microsoft.PowerApps.Administration.PowerShell</a>. This module currently <strong>only supports</strong> placing the certificate in the current user's store. In order to properly authenticate with the module, you will need to make sure you register your service principal as a Power App Management app. For details on how to do this, please refer to the <a href="https://learn.microsoft.com/en-us/power-platform/admin/powershell-create-service-principal#registering-an-admin-management-application">official documentation</a>.</p>

<h3>Via MSCloudLoginAssistant</h3>

``` powershell
Connect-M365Tenant -Workload 'PowerPlatforms' `
                   -ApplicationId '<your app id>' `
                   -TenantId '<your tenant>.onmicrosoft.com' `
                   -CertificateThumbprint '<your thumbprint>'
```

<img src="/blog/2024/authenticating-with-powershell/images/powermscloud.png" alt="Connecting to Power Platforms via MSCloudLoginAssistant." />

<h3>Via Microsoft.PowerApps.Administration.PowerShell</h3>

``` powershell
Add-PowerAppsAccount -ApplicationId '<your app id>' `
                     -TenantId '<your tenant>.onmicrosoft.com' `
                     -CertificateThumbprint '<your thumbprint>'
```
<img src="/blog/2024/authenticating-with-powershell/images/powermodule.png" alt="Connecting to Power Platforms via Microsoft.PowerApps.Administration.PowerShell." />

<h3>Local Configuration Management</h3>
<p>This is where things get a little more complicated. At the time of writing this article, the Power Platforms PowerShell mdule only supports looking into the current user's store and <strong>not</strong> in the local system's one. Because the LCM always runs as the Local System user, this means that we need to install the certificate in the current user's store of....the Local System user (I know, right). Currently, the best way to achieve this is to use a tool such as <a href="https://learn.microsoft.com/en-us/sysinternals/downloads/psexec">PSExec</a> to let you launch the Management Console as the Local System user and install the certificate in its store.</p>

<ul>
<ol>Download and install the PSTools under <strong>C:\tools</strong></ol>
<ol>Launch a new PowerShell console as admin.</ol>
<ol>Browse to C:\tools, and execute <strong>./Psexec.exe -i -s cmd.exe</strong></ol>
<ol>When prompted, click on <strong>Agree</strong> (only shown if it's the first time you use the tool).</ol>
<ol>In the new command prompt that appeared, type in <strong>mmc.exe</strong>.</ol>
<ol>In the contols that opened, press <strong>CTRL+M</strong> to open the add-in selection.</ol>
<ol>Select the Certificates add-in and choose the <strong>My user account</strong> option. Click on <strong>Finish</strong>, then <strong>Ok</strong>.<br/>
<img src="/blog/2024/authenticating-with-powershell/images/mycertselect.png" alt="Selecting th user's certificate store for the local system." /></ol>
<ol>Expand <strong>Certificates - Curent User</strong> and select the <strong>Personal</strong> folder.</ol>
<ol>In the Object Type panel, right-click and choose <strong>All Tasks > Import</strong><br />
<img src="/blog/2024/authenticating-with-powershell/images/import.png" alt="Importing a certificate." /></ol>
<ol>Click <strong>Next</strong> then browse to select the private key (.pfx) for your certificate. You will need to change the file type to <strong>Personal Information Exchange</strong>first.<br/>
<img src="/blog/2024/authenticating-with-powershell/images/pfximport.png" alt="Importing a private key for a given certificate." /></ol>
<ol>Click <strong>Next</strong> then provide the password for your private key.</ol>
<ol>Click <strong>Next</strong> twice and then on <strong>Finish</strong> to complete the import process.</ol>
</ul>

<h2>Teams</h2>
<p>Authentication to Microsof Teams is handled by the <a href = "https://www.powershellgallery.com/packages/MicrosoftTeams/">MicrosoftTeams</a> PowerShell module. Just like all the other modules, it supports placing the certificate in the current user's store.</p>

<h3>Via MSCloudLoginAssistant</h3>

``` powershell
Connect-M365Tenant -Workload 'MicrosoftTeams' `
                   -ApplicationId '<your app id>' `
                   -TenantId '<your tenant>.onmicrosoft.com' `
                   -CertificateThumbprint '<your thumbprint>'
```

<img src="/blog/2024/authenticating-with-powershell/images/teamsmscloud.png" alt="Connecting to Microsoft Teams via MSCloudLoginAssistant." />

<h3>Via MicrosoftTeams</h3>

``` powershell
Connect-MicrosoftTeams -ApplicationId '<your app id>' `
                       -TenantId '<your tenant>.onmicrosoft.com' `
                       -CertificateThumbprint '<your thumbprint>'
```
<img src="/blog/2024/authenticating-with-powershell/images/teamsmodule.png" alt="Connecting to Microsoft Teams via MicrosoftTeams." />

<h3>Local Configuration Management</h3>
<p>LCM requires the certificate to be installed in the Local Machine's store in order to authenticate.</p>