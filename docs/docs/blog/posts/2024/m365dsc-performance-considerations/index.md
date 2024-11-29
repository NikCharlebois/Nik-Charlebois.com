---
title: Performance Considerations for Microsoft365DSC
---

<h1 class="blog-title">Performance Considerations for Microsoft365DSC</h1>
<div class="article-date">2024-11-29</div>

<p>In this article we will cover various things you should consider when trying to improve the performance of Snapshots and Monitoring/Deployment when using <a href="https://Microsoft365DSc.com">Microsoft365DSC</a>. This is a topic to is of concern for almost every customer I get to talk to, and there are various things that users can do to improve the overall time it takes to perform the various operations using the solution.</p>

<h2 id="noteverything">Only Export What You Need<a href="#noteverything" class="anchor">⚓</a></h2>
<p>By default, doing an export of a tenant's configuration using the <em>Export-M365DSCConfiguration</em> cmdlet will include over 405 resource types as part of the capture. Most customer I talk to don't need to capture it all. Our recommendation is to make sure you only capture the components you need and are interested in exporting. The <em>Export-M365DSCConfiguration</em> cmdlet offers various ways to specify subsets of the resources you want to export:</p>

<ul>
<li><strong>Components</strong>: This parameter lets you specify the individual resource types you wish to export. For example, if you only care about Conditional Access Policies and Applications in Entra Id, you could specify the parameter as <em>-Components @('AADConditionalAccessPolicy, 'AADApplication')</em>. The <a href="https://export.microsoft365dsc.com">Microsoft365DSC Export</a> site allows you to visually select the components you want and build the associated command.</li>
<li><strong>Workloads</strong>: This parameter lets you specify the workloads you whish to export the resources for. For example, if you want to export all of the Exchange Online resources types, instead of specifying each one using the <em>Components</em> parameter, you can use the <em>-Workloads @("EXO")</em> parameter instead.</li>
<li><strong>Mode</strong>: The cmdlet include the Mode parameter which let's you specify 3 modes (Default, Full, Lite). Each one includes different set of resources that will be included as part of the export.</li>
</ul>

<h2 id="exportfilters">Use Filters<a href="#exportfilters" class="anchor">⚓</a></h2>
<p>It is one thing to only select the resource types you wish to capture, but what about if you only want certain instances of these resource types to be captured as part of your export? By using the <strong>Filter</strong> parameter with the Export-M365DSCConfiguration cmdlet, you can scope any instance type down to the specific instances you want. For example, the newly introduced <em>AADRoleManagementPolicyRule</em> resource tends to take a very long time to export all of its associated instances. What if you are only interested in capturing a specific privileged role, say Global Administrator instead of all of them? The following lines of code will allow you to achieve exactly this:</p>

``` powershell
$Filters = @{
    AADRoleManagementPolicyRule = "displayName eq 'Global Administrator' "
}

Export-M365DSCConfiguration @Global:IntegrationParams `
                            -Components 'AADRoleManagementPolicyRule' `
                            -Filters $Filters
```
<img src="/blog/posts/2024/m365dsc-performance-considerations/images/filters.png" alt="Using filters when exporting a tenant's existing configuration." />

<h2 id="controlplane">Try to Avoid Data Plane Resource Types<a href="#controlplane" class="anchor">⚓</a></h2>
<p>Whenever I do customer presentations, I always introduce this topic as the "It's not because it's avaialble that you SHOULD use it" section. Microsoft365DSC today supports over 440+ resource types, where some of them fall into what we consider being part of the <strong>control plane</strong> and some others fall into <strong>data plane</strong>. In our definition, control plane represents configuration settings that are normally only made available to the administrators via the various admin centers, whereas the data plane represent configuration settings that can also be managed by end-users. Consider the Planner resources for example, these clearly fall into the data plane. It wouldn't make sense for your organization to control these via DSC, even less to monitor them for configuration drifts. The whole point of having a Planner tasks is that it will begin in the <strong>Not Started</strong> phase with 0% completion and hopefully get to the <strong>Completed</strong> stage with 100% completion. You would never want to monitor this for drifts since end-users can interact with a task and modify it. Therefore you should always ask yourself whether or not you need to include these as part of your config and if it's worth including them in the export.</p>

<p>There are some cases that fall more into a grey zone. Teams channels is a great example of this where they can both be controlled via the Teams admin center, but as an owner of the team I can also change its settings and remove/create and delete the channels. You need to be careful before capturing those as part of your configuration and be intentional about how you intend to use it. If you are simply to try to capture a backup of resource types for compliance or reporting pusposes, that's fine but just know it will take longer to export those as resource types from the data plane tend to have many instances. We would <strong>never recommend</strong> you use DSC to monitor resources that are part of the data plane.</p>

<h2 id="skipdependencies">Skip Dependencies Validation<a href="#skipdependencies" class="anchor">⚓</a></h2>
<p>We took a design decision a long time ago to validate that the machine from which customers are running DSC have the right versions of the required dependencies installed. Because we don't control the full logic flow when DSc is being run via the Local Configuration Manager (e.g., via Start-DSCConfiguration or Test-DSCConfiguration), we opted to add a check in every Get/Set/Test/Export function for every resource. While the first call will always be the heaviest, it still takes up some cycles to have to verify this for every resource. To help reduce the overhead of the dependency validation, we introduced a global variable that can be set to explicitely tell M365DSC not to validate the dependencies. While setting this value to skip the dependency check will improve the performance of the overall process, it is mostly going to help at the initiation phase, where it will reduce the initial overhead required to initiate the first resource export. If you are considering disabling this check, you should consider running the following line of code before executing the wanted operation:</p>

``` powershell
$Global:M365DSCSkipDependenciesValidation = $true
```

<p>When running preliminary benchmarks with a single resource type (AADApplication with 37 instances), we've observed a 13% of reduction in overall completion time. Please take this with a grain of salt since as mentioned above, it mostly helps speed up the script warmup process and doesn't have much impact on the subsequent resource export.</p>

<h2 id="parallel">Parallelize Jobs as Much as Possible<a href="#parallel" class="anchor">⚓</a></h2>
<p>While this one might sound trivial, it is recommend to divider and conquer where possible. Microsoft365DSC is sequential in nature, therefore it is better to split the deployment, monitoring and snapshot jobs into several smaller parallel jobs. For example, instead of having a single process export all of the resource instance of your tenant, your should opt to split the process into smaller jobs such as having one process for each workload running in parallel. It is important to note that while this will help speed up the overall completion time, that you are still subject to the API throttling behind the scene; Microsoft365DSC doesn't escape from these restrictions.</p>

<h2 id="telemetry">Disable Telemetry<a href="#telemetry" class="anchor">⚓</a></h2>
<p>By default, Microsoft365DSC captures telemetry information to better understand how customers are using its various features and find areas of improvements for the overall solution. Customers can also choose to send this telemetry back to their own Application Insights environments for reporting purposes. For additional details on how to setup the telemetry for your agents, you can refer to the following <a href="https://microsoft365dsc.com/user-guide/get-started/telemetry/">article</a>. This telemetry capture can, in some instances, add extra overhead on the solution and disabling it can help improve the overall speed of your operations.</p>

``` powershell
Set-M365DSCTelemetryOption -Enabled $false
```

<script src="https://utteranc.es/client.js"
        repo="NikCharlebois/Nik-Charlebois.com"
        issue-term="pathname"
        label="Comment"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>