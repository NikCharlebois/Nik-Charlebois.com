<h1>Getting Started with Microsoft365DSC</h1>
<div style="position:inherit;padding-top:15px;"><span style="float:left;padding-left:15px;"><p><strong>by Nik Charlebois</strong><br />
November 29th, 2024</p></span></div>
<br /><br /><br />

<p>In this article, we will guide you through all the steps involved in getting up and running with <a href="https://Microsoft365DSC.com">Microsoft365DSC</a>. For the purpose of this article, I created a brand new Windows Server 2025 virtual machine and this is the environment I will be using throughout this blog post. Please note that we will also be using Windows PowerShell 5.1 for all steps described below. Also, at the time of writing this article, the latest Microsoft365DSC version was <strong>1.24.1127.1</strong>.</p>

<h2>Step 1 - Install the Core Microsoft365DSC Module</h2>

<p>Open a new Windows PowerShell (version 5.1) windows as an administrator.</p>
<img src="/blog/2024/getting-started-m365dsc/images/runpowershellasadmin.png" alt="Run PowerShell as Admin" />

<p>Run the following line of PowerShell code. This will pull the Microsoft365DSC modul from the <a href="https://PowerShellGallery.com">PowerShell Gallery</a> and install it on the machine under the path <em>c:\Program Files\WindowsPowerShell\Modules\Microsoft365DSC\<version></em></p>
``` powershell
Install-Module Microsoft365DSC -Force
```

<p>If prompted to install the latest version of the nuget provider, type in <strong>Y</strong>.</p>
<img src="/blog/2024/getting-started-m365dsc/images/installnuget.png" alt="Install the latest version of the nuget provider." />

<h2>Update Dependencies</h2>
<p>Microsoft365DSC depends on about a dozen different other modules (e.g., the Exchange Online Management Shell, the Teams PowerShell Moduel, the Graph PowerShell SDK, etc.). The previous step <strong>only</strong> installed the core Microsoft365DSC PowerShell module, but di not install any of its dependencies, which are required for the module to run properly. The next step of the installation consists of downloading and installing all of these dependencies. If you are curious to understand what all these dependencies are, you can refer to the <a href="https://github.com/microsoft/Microsoft365DSC/blob/Dev/Modules/Microsoft365DSC/Dependencies/Manifest.psd1">Microsoft365DSC Dependency Manifest</a> file. In order to download and install all these dependencies, simply run the following PowerShell cmdlet.</p>

``` powershell
Update-M365DSCModule
```

<img src="/blog/2024/getting-started-m365dsc/images/installingdependencies.png" alt="Install the latest version of the nuget provider." />
<p><strong>Note:</strong> Running this command will automatically download all the dependencies from the PowerShell Gallery and install them under <em>C:\Program Files\WindowsPowerShell\Modules\</em>.</p>