---
title: Troubleshooting TCM Monitor and Snapshot Job Errors with errorDetails
---

<h1 class="blog-title">Troubleshooting TCM Monitor and Snapshot Job Errors with errorDetails</h1>
<div class="article-date">2026-06-22</div>

<p>When working with Tenant Configuration Management, or TCM, there are two asynchronous experiences where administrators typically need deeper troubleshooting information: monitor runs and snapshot jobs. A monitor run can come back as <strong>failed</strong> or <strong>partiallySuccessful</strong>. A snapshot job can do the same. The default Microsoft Graph response is useful for knowing that something went wrong, but it usually does not give you enough information to know what to fix.</p>

<p>The missing piece is the <strong>errorDetails</strong> property. It is not returned by default. You must ask Microsoft Graph for it explicitly by using the <code>$select</code> query parameter.</p>

<p><img src="\blog\posts\2026\tcm-error-details\images\error-details-flow.svg" alt="Diagram showing that TCM monitor runs and snapshot jobs require $select=errorDetails to retrieve actionable troubleshooting information." /></p>

<h2>The short version</h2>

<p>If you already have the identifier of the failing object, call the object directly and select <strong>errorDetails</strong>:</p>

```http
GET https://graph.microsoft.com/v1.0/admin/configurationManagement/configurationMonitoringResults/{configurationMonitoringResultId}?$select=id,monitorId,runStatus,errorDetails
```

```http
GET https://graph.microsoft.com/v1.0/admin/configurationManagement/configurationSnapshotJobs/{configurationSnapshotJobId}?$select=id,displayName,status,errorDetails
```

<p>The Microsoft Graph documentation uses the camel-case property name <code>errorDetails</code>. If you have seen examples written as <code>?$select=errordetails</code>, I recommend keeping the documented casing in your automation to avoid surprises.</p>

<h2>Why the default response is not enough</h2>

<p>Both monitor results and snapshot jobs expose a status field. That field tells you the outcome of the operation, but not necessarily the reason behind the outcome. For example, the following response tells us that the monitor run failed:</p>

```json
{
  "id": "66fa1689-22cb-49c1-8b5a-c94822b7b13b",
  "monitorId": "<monitorId>",
  "tenantId": "<tenantId>",
  "runInitiationDateTime": "2026-06-22T12:00:36.1084955Z",
  "runCompletionDateTime": "2026-06-22T12:01:11.1084955Z",
  "runStatus": "failed",
  "driftsCount": 0
}
```

<p>That is a good health signal, but it is not yet a troubleshooting signal. If this is part of an automated monitoring pipeline, you do not want your alert to simply say "the monitor failed." You want it to say which resource type failed, which instance failed, and what Microsoft Graph reported as the error.</p>

<h2>What errorDetails contains</h2>

<p>The <strong>errorDetails</strong> collection is designed to provide exactly that missing context. For monitor runs, the documented type is <strong>errorDetail</strong>. For snapshot jobs, the documentation describes the property as the details of errors related to the reasons why the snapshot cannot complete. In practice, the important troubleshooting dimensions are:</p>

<ul>
<li><strong>resourceType:</strong> the TCM resource type that failed, such as <code>microsoft.teams.meetingpolicy</code> or <code>microsoft.exchange.transportrule</code>.</li>
<li><strong>resourceInstanceName:</strong> the specific resource instance that caused the issue, when available.</li>
<li><strong>errorMessage:</strong> the error text that points you toward the remediation.</li>
</ul>

<p><img src="\blog\posts\2026\tcm-error-details\images\error-details-anatomy.svg" alt="Graphic showing the three important fields in a TCM errorDetails response: resourceType, resourceInstanceName, and errorMessage." /></p>

<p>A selected response for a failed monitor run could look like this:</p>

```json
{
  "id": "<monitorId>",
  "monitorId": "69b6b9ba-20c9-4ffb-beef-263c07063222",
  "runStatus": "failed",
  "errorDetails": [
    {
      "resourceType": "microsoft.teams.meetingpolicy",
      "resourceInstanceName": "Global",
      "errorMessage": "Access Denied."
    }
  ]
}
```

<p>That is much more actionable. Instead of guessing whether the issue is with the monitor definition, the TCM service principal, a workload role, or a resource-specific problem, you now have a concrete starting point.</p>

<h2>Finding failed monitor runs</h2>

<p>Monitor run history is exposed through <strong>configurationMonitoringResults</strong>. The list operation supports <code>$select</code>, <code>$filter</code>, <code>$orderby</code>, and <code>$top</code>, so you can quickly find recent failed or partially successful runs.</p>

```http
GET https://graph.microsoft.com/v1.0/admin/configurationManagement/configurationMonitoringResults?$filter=runStatus eq 'failed'&$orderby=runInitiationDateTime desc&$top=10
```

<p>Once you have the failed result identifier, retrieve the error details:</p>

```http
GET https://graph.microsoft.com/v1.0/admin/configurationManagement/configurationMonitoringResults/<monitorId>?$select=id,monitorId,runInitiationDateTime,runCompletionDateTime,runStatus,driftsCount,errorDetails
```

<p>You can also filter for partially successful monitor runs:</p>

```http
GET https://graph.microsoft.com/v1.0/admin/configurationManagement/configurationMonitoringResults?$filter=runStatus eq 'partiallySuccessful'&$orderby=runInitiationDateTime desc
```

<p>This is important because a partially successful run may still contain useful drift results for some resources while one or more resources failed. Treating <strong>partiallySuccessful</strong> as "good enough" in automation can hide resource coverage gaps.</p>

<h2>Finding failed snapshot jobs</h2>

<p>Snapshot jobs are exposed through <strong>configurationSnapshotJobs</strong>. Snapshot jobs are asynchronous, so the normal workflow is to create the snapshot, poll the job until it completes, and then download the snapshot from the <strong>resourceLocation</strong> value when the job succeeds or partially succeeds.</p>

<p>To list recent failed snapshot jobs:</p>

```http
GET https://graph.microsoft.com/v1.0/admin/configurationManagement/configurationSnapshotJobs?$filter=status eq 'failed'&$orderby=createdDateTime desc&$top=10
```

<p>Then call the specific job and select <strong>errorDetails</strong>:</p>

```http
GET https://graph.microsoft.com/v1.0/admin/configurationManagement/configurationSnapshotJobs/<jobId>?$select=id,displayName,status,createdDateTime,completedDateTime,errorDetails
```

<p>For partially successful jobs, use the same pattern:</p>

```http
GET https://graph.microsoft.com/v1.0/admin/configurationManagement/configurationSnapshotJobs?$filter=status eq 'partiallySuccessful'&$orderby=createdDateTime desc
```

<p>The snapshot job status enumeration includes <strong>partiallySuccessful</strong> as an evolvable enum value. If you are building strongly typed clients, make sure your code does not break when Graph returns future enum members. For raw REST calls, this usually just means treating status values as strings and not assuming that today's list is the complete list forever.</p>

<h2>Using Microsoft Graph PowerShell</h2>

<p>If you are using the Microsoft Graph PowerShell SDK, the command names are available in the <strong>Microsoft.Graph.ConfigurationManagement</strong> module. The generated cmdlets expose the same resources as the REST API. If the SDK cmdlet in your installed version does not expose a convenient select parameter for the non-default property, you can always use <strong>Invoke-MgGraphRequest</strong> and call the REST endpoint directly.</p>

```powershell
Import-Module Microsoft.Graph.Authentication
Import-Module Microsoft.Graph.ConfigurationManagement

Connect-MgGraph -Scopes 'ConfigurationMonitoring.Read.All'

$resultId = '<resultId>'
$uri = "/v1.0/admin/configurationManagement/configurationMonitoringResults/$resultId" +
       '?$select=id,monitorId,runStatus,errorDetails'

$result = Invoke-MgGraphRequest -Method GET -Uri $uri
$result.errorDetails | Format-Table resourceType, resourceInstanceName, errorMessage -AutoSize
```

<p>For snapshot jobs, only the path changes:</p>

```powershell
$snapshotJobId = '<jobId>'
$uri = "/v1.0/admin/configurationManagement/configurationSnapshotJobs/$snapshotJobId" +
       '?$select=id,displayName,status,errorDetails'

$job = Invoke-MgGraphRequest -Method GET -Uri $uri
$job.errorDetails | Format-Table resourceType, resourceInstanceName, errorMessage -AutoSize
```

<h2>Building better automation around TCM errors</h2>

<p>The practical value of <strong>errorDetails</strong> shows up when you build it into your operational workflows. A monitor failure should not require someone to manually open Graph Explorer, re-run the query, and discover that the service principal is missing a workload role. Your automation can do that triage automatically.</p>

<p><img src="\blog\posts\2026\tcm-error-details\images\troubleshooting-loop.png" alt="Troubleshooting loop showing how to find a failed TCM run, select errorDetails, map the cause, fix the issue, and rerun." /></p>

<p>At a minimum, I recommend logging these fields whenever a monitor result or snapshot job is not fully successful:</p>

<ul>
<li><strong>Object identifier:</strong> the monitoring result ID or snapshot job ID.</li>
<li><strong>Operation status:</strong> <code>failed</code> or <code>partiallySuccessful</code>.</li>
<li><strong>Resource type:</strong> the TCM resource type that failed.</li>
<li><strong>Resource instance:</strong> the resource instance name, when present.</li>
<li><strong>Error message:</strong> the text returned by Graph.</li>
</ul>

<p>From there, you can group errors by <strong>resourceType</strong>. If every Teams-related resource fails with an access error, you are probably dealing with a missing workload role or permission. If only one resource instance fails, the issue is likely more specific to that object or its configuration. If snapshot jobs fail across every requested resource, look first at the TCM service principal setup and the tenant-level permissions.</p>

<h2>Common remediation patterns</h2>

<p>The error message is the source of truth, but most failures tend to fall into a few operational categories:</p>

<ul>
<li><strong>Missing Microsoft Graph permission:</strong> the calling application or user does not have the required <code>ConfigurationMonitoring.Read.All</code> or <code>ConfigurationMonitoring.ReadWrite.All</code> permission for the operation being performed.</li>
<li><strong>Missing TCM service principal permission:</strong> the Unified Tenant Configuration Management service principal does not have the workload permissions required to read or evaluate the selected resource type.</li>
<li><strong>Missing workload role:</strong> some workloads require the TCM service principal to hold specific Microsoft Entra roles in addition to Graph permissions.</li>
<li><strong>Unsupported or unavailable resource state:</strong> the requested resource type or instance cannot be processed in the tenant's current state.</li>
<li><strong>Transient workload issue:</strong> a backend workload error may require retrying the monitor run or snapshot job after the service recovers.</li>
</ul>

<p>The key is to avoid treating all TCM failures as equal. A failed monitor run with one Exchange transport rule error is a different operational problem than a snapshot job where every Teams resource failed due to access denied.</p>

<h2>Suggested pattern for scripts</h2>

<p>The following pseudo-flow is what I normally recommend for automation:</p>

```powershell
# 1. Find recent failed or partially successful monitor runs.
# 2. For each result, call the result directly with $select=errorDetails.
# 3. Write one log record per error detail.
# 4. Group by resourceType and errorMessage.
# 5. Route the issue to the right owner or remediation workflow.
```

<p>That same pattern applies to snapshot jobs. The only difference is the collection and status field names:</p>

<table>
<thead>
<tr>
<th>Scenario</th>
<th>Collection</th>
<th>Status field</th>
<th>Error property</th>
</tr>
</thead>
<tbody>
<tr>
<td>Monitor run</td>
<td><code>configurationMonitoringResults</code></td>
<td><code>runStatus</code></td>
<td><code>errorDetails</code></td>
</tr>
<tr>
<td>Snapshot job</td>
<td><code>configurationSnapshotJobs</code></td>
<td><code>status</code></td>
<td><code>errorDetails</code></td>
</tr>
</tbody>
</table>

<h2>Conclusion</h2>

<p>The <strong>errorDetails</strong> property is one of those small Graph API details that makes a big operational difference. Without it, a failed TCM monitor run or snapshot job is only a status value. With it, you can see the failing resource type, the affected instance, and the message that points you toward remediation.</p>

<p>If you are building automation around Tenant Configuration Management, make <code>$select=errorDetails</code> part of your failure-handling path. It will make your alerts more useful, your logs more actionable, and your troubleshooting much faster.</p>

<script src="https://utteranc.es/client.js"
        repo="NikCharlebois/Nik-Charlebois.com"
        issue-term="pathname"
        label="Comment"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>
