---
title: Introducing the Tenant Configuration Management APIs
---

<h1 class="blog-title">Introducing the Tenant Configuration Management APIs</h1>
<div class="article-date">2026-01-27</div>

<p>On Tuesday, January 27th, Microsoft released the public preview of their Tenant Configuration Management APIs. Those are a set of APIs built on Microsoft Graph which provide features allowing customers to manage their Microsoft 365 tenant's configuration as code. At the time of writing this article, they provide two main features: the ability to backup an existing tenant's configuration into a JSON configuration template, and the ability to continuously monitor your tenants for unwanted configuration drifts. Over the next few months, they will also release the ability to deploy configuration changes (via JSON configuration templates) and to automatically remediate to detected configuration drifts. The preview currently only supports the Workloads listed below. Support for additional workloads such as SharePoint Online, Copilot, etc. will be provided over time.</p>
<ul>
<li>Defender (limited coverage)</li>
<li>Entra ID</li>
<li>Exchange Online</li>
<li>Intune</li>
<li>Purview</li>
<li>Teams</li>
</ul>

<p>While they are currently only available in commercial tenants, Microsoft has plans to make those available in Government and other sovereign clouds in the near future. The fact that the APIs hit public preview today, means that every commercial Microsoft 365 tenant is now able to leverage them and call into the endpoints without any pre-requesites (e.g., the need to onboard a tenant to the preview).</p>

<h2>Authentication</h2>
<p>There are two ways you can interact with the new TCM APIs at the moment: with a user account or via a service principal. If you are trying to interact with a user account, that user will require one of the available default Entra ID Privileged roles. Custom Entra ID roles are not supported for interacting with the APIS. To get more information on what are the available privileged roles in Entra ID, please refer to the <a href="https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/privileged-roles-permissions?tabs=admin-center">Privileged Roles</a> article on the Microsoft site.</p>

<p>The second way to interact with the APIs is via a service principal managed by your organization and created via an app registration. In order for that service principal to be able to interact with the TCM APIs, it will require the <strong>ConfiguratonMonitoring.Read.All</strong> or the <strong>ConfigurationMonitoring.ReadWrite.All</strong></p> Microsoft Graph permission. It doesn't matter if you are trying to interact with the Monitors or Snapshots APIs, the same permissions apply (even if the name refers to monitoring only).</p>

<img src="\blog\posts\2026\introducing-tcm-apis\images\tcm-intro-blog1.png" alt="Granting the ConfiguratonMonitoring.ReadWrite.All permission to a service principal." />

<h2>Snapshot</h2>
<p>Probably the first thing most customers will want to do when trying the TCM APIs for the first time, is extract the current configuration of an existing tenant they own and are familiar with. Getting a snapshot is also a great way to get started with a baseline configuration JSON template without having to write everything from scratch. The process of generating a snapshot of a tenant involves creating an asynchronous job known as a snapshot job. When creating this job, you will need to do a POST call to the APIs and provide the list of the configuration resources you wish to capture as part of the snapshot. A configuration resource, in the context of TCM, represents the macro setting type. For example, an Entra ID Conditional Access Policy, an Exchange Online Transport Rule, a Teams Meeting Policy, those are all coonfiguration resource types. For a full list of the resources supported by TCM and additional information about how to include them in your configuration files, you can refer to the official Microsoft documentation: <a href="https://learn.microsoft.com/en-us/graph/api/resources/configurationbaseline">Configuration Baseline Overview</a>.</p>

<p>The snapshot jobs run on behalf of the Unified Tenant Configuration Management service principal. This service principal is owned and managed by Microsoft, but doesn't come by default in your tenants. In order to add this service principal to a tenant, follow the steps described in the following Microsoft documentation: <a href="">Authentication for the Tenant Configuration Management APIs</a>.</p>

<p>For the purpose of this article, we will assume that we are trying to take a snapshot of our Entra Id Conditional Access Policies, our Entra Id Group Lifecycle Policy, our Exchange Online Transport Rules and our Teams Meeting Policies. In order for the snapshot job to capture the requested resources, it will need the right permissions assigned to its associated Unified Tenant Configuration Management service principal. To make it easier for users to determine what permissions are required based on the resources types we are trying to interact with, I've put together a simple PowerShell module that will accept a list of resource types or an actual JSON configuration template and will output the list of required permissions and roles to be assigned. To learn more about this module, head to <a href="">TCM.Utility on GitHub</a>.</p>

<p>In our case, by running the following line of PowerShell:</p>

```powershell
$result = Test-TCMConfigurationTemplate -ResourceNames @(
            'microsoft.entra.conditionalaccesspolicy', 
            'microsoft.entra.grouplifecyclepolicy', 
            'microsoft.exchange.transportrule',
            'microsoft.teams.meetingpolicy')
```

<p>Since Snapshot is a read-only operation, we can determine that the following Read permissions and roles need to be assigned to the Unified Tenant Configuration Management service principal:</p>
<ul>
<li><strong>Graph:</strong>
<ul>
<li>Agreement.Read.All
<li>Application.Read.All</li>
<li>CustomSecAttributeDefinition.Read.All</li>
<li>Directory.Read.All</li>
<li>Group.Read.All</li>
<li>Organization.Read.All</li>
<li>Policy.Read.All</li>
<li>RoleManagement.Read.Directory</li>
<li>User.Read.All</li>
</ul>
<li><strong>Office 365 Exchange Online:</strong>
<ul>
<li>Exchange.ManageAsApp</li>
</ul>
</li>
</ul>

<p>With this information handy, we now need to grant the required permissions to the service principal. Luckily, my utility module also provides an easy way for organization to assign the require API permissions. By running the following line of PowerShell, you can automate the permissions granting process, where <em>$info</em> is the permission information obtained above:</p>

```powershell
Add-TCMServicePrincipalPermissions -Permissions $info.Read.Permissions
```

<img src="\blog\posts\2026\introducing-tcm-apis\images\tcm-intro-blog2.png" alt="Permissions grant on the Unified Tenant Configuration Management service principal." />

<p>For workloads other than Entra Id and Intune, it is also required to assign Entra Id roles to the UTCM service principal. The list of these required roles is also provided by the utility module:</p>

```powershell
$info.Read.Roles
```

<p>In our example, the above line of PowerShell will return two Entra Id roles that need to be assigned: Security Reader and Teams Reader. Off course uber roles such as Global Reader would also do the trick, but the utility will inform you on the least required roles and permissions. In order to grant these roles, navigate to the Entra Portal and grant each one to the Unified Tenant Configuration Management service principal.</p>

<img src="\blog\posts\2026\introducing-tcm-apis\images\tcm-intro-blog3.png" alt="Granting roles to the Unified Tenant Configuration Management service principal." />

<p>If you forget to assign permissions or roles to the UTCM service principal, your snapshot job will return a status of <strong>Partially Successful</strong> with errors such as the following being reported: <em>microsoft.teams.meetingpolicy: Error exporting resource [microsoft.teams.meetingpolicy]. exceptionMessage(s):Error during Export: {"code":"Forbidden","message":"Access Denied.","action":"Provide different credential or request access."}</em></p>

<p>Now that all roles and permissions have been properly granted, we are ready to proceed with our snapshot operation. In order to initiate the snapshot job, we need to make the following POST call to the APIs:</p>

```json
POST https://graph.microsoft.com/beta/admin/configurationManagement/configurationSnapshots/createSnapshot
{
     
    "displayName": "Snapshot Blog Post", 
    "description": "Snapshot Description", 
    "resources": 
    [
            "microsoft.entra.conditionalaccesspolicy", 
            "microsoft.entra.grouplifecyclepolicy", 
            "microsoft.exchange.transportrule",
            "microsoft.teams.meetingpolicy"
    ] 
}
```

<p>The response will look something like the following:</p>

```json
{
    "@odata.context": "https://graph.microsoft.com/beta/$metadata#microsoft.graph.configurationSnapshotJob",
    "id": "d265ab29-2e00-4107-a365-720b3c2e7e99",
    "displayName": "Snapshot Blog Post",
    "description": "Snapshot Description",
    "tenantId": "xxxxxxxxxxxxx",
    "status": "notStarted",
    "resources": [
        "microsoft.entra.conditionalaccesspolicy",
        "microsoft.entra.grouplifecyclepolicy",
        "microsoft.exchange.transportrule",
        "microsoft.teams.meetingpolicy"
    ],
    "createdDateTime": "2026-01-27T22:09:05.7819107Z",
    "completedDateTime": "0001-01-01T00:00:00Z",
    "resourceLocation": "",
    "createdBy": {
        "user": {
            "id": "74b56a93-a473-40eb-a011-555555550555",
            "displayName": "MOD Administrator"
        },
        "application": {
            "id": null,
            "displayName": null
        }
    }
}
```

<p>At the moment, you need to poll the Snapshot Jobs API to figure out when the job has completed. In the future, Microsoft is planning on implementing some Graph notifications mechanisms to allow for a Push model instead of a Poll one. To get the status of your job, user will need to ping the follow endpoint, passing in the id of the job received from the previous API's response:</p>

```json
GET https://graph.microsoft.com/beta/admin/configurationManagement/configurationSnapshotJobs('d265ab29-2e00-4107-a365-720b3c2e7e99')
{
    "@odata.context": "https://graph.microsoft.com/beta/$metadata#admin/configurationManagement/configurationSnapshotJobs/$entity",
    "@microsoft.graph.tips": "This request only returns a subset of the resource's properties. Your app will need to use $select to return non-default properties. To find out what other properties are available for this resource see https://learn.microsoft.com/graph/api/resources/configurationSnapshotJob",
    "id": "d265ab29-2e00-4107-a365-720b3c2e7e99",
    "displayName": "Snapshot Blog Post",
    "description": "Snapshot Description",
    "tenantId": "xxxxxxxxxxxxxxx",
    "status": "succeeded",
    "resources": [
        "microsoft.entra.conditionalaccesspolicy",
        "microsoft.entra.grouplifecyclepolicy",
        "microsoft.exchange.transportrule",
        "microsoft.teams.meetingpolicy"
    ],
    "createdDateTime": "2026-01-27T22:09:05.7819107Z",
    "completedDateTime": "2026-01-27T22:10:08.8365374Z",
    "resourceLocation": "https://graph.microsoft.com/beta/admin/configurationManagement/configurationSnapshots('3b05c24c-db31-4d15-b386-947768ed66b0')",
    "createdBy": {
        "user": {
            "id": "74b56a93-a473-40eb-a011-555555550555",
            "displayName": "MOD Administrator"
        },
        "application": {
            "id": null,
            "displayName": null
        }
    }
}
```

<p>Once the job's status gets returned as <strong>succeeded</strong> (or even partiallySuccessful), you will be provided a value for the resourceLocation value. That value represents the URL you need to call into to retrieve the content of your snapshot. The snapshots are preserved for a maximum of 7 days after they are created.</p>

```json
GET https://graph.microsoft.com/beta/admin/configurationManagement/configurationSnapshots('3b05c24c-db31-4d15-b386-947768ed66b0')
{
    "@odata.context": "https://graph.microsoft.com/beta/$metadata#admin/configurationManagement/configurationSnapshots/$entity",
    "@microsoft.graph.tips": "Use $select to choose only the properties your app needs, as this can lead to performance improvements. For example: GET admin/configurationManagement/configurationSnapshots('<guid>')?$select=description,displayName",
    "id": "3b05c24c-db31-4d15-b386-947768ed66b0",
    "displayName": "Snapshot Blog Post",
    "description": "Snapshot Description",
    "parameters": [],
    "resources": [
        {
            "displayName": "TeamsMeetingPolicy-Global",
            "resourceType": "microsoft.teams.meetingpolicy",
            "properties": {
                "AllowEngagementReport": "Enabled",
                "AllowWatermarkForCameraVideo": true,
                "ChannelRecordingDownload": "Allow",
                "AllowCloudRecording": true,
                "AllowSharedNotes": true,
                "AllowAnonymousUsersToDialOut": false,
                "AllowWatermarkForScreenSharing": true,
                "AllowParticipantGiveRequestControl": true,
                "AllowWhiteboard": true,
                "Ensure": "Present",
                "AllowAnnotations": true,
                "AllowNetworkConfigurationSettingsLookup": false,
                "InfoShownInReportMode": "FullInformation",
                "TeamsCameraFarEndPTZMode": "Disabled",
                "EnrollUserOverride": "Disabled",
                "AllowRecordingStorageOutsideRegion": false,
                "StreamingAttendeeMode": "Disabled",
                "ExplicitRecordingConsent": "Disabled",
                "AllowNDIStreaming": false,
                "AllowChannelMeetingScheduling": true,
                "AllowAnonymousUsersToJoinMeeting": true,
                "AllowPrivateMeetNow": true,
                "QnAEngagementMode": "Enabled",
                "LiveInterpretationEnabledType": "DisabledUserOverride",
                "AllowIPAudio": true,
                "RoomPeopleNameUserOverride": "Off",
                "LiveCaptionsEnabledType": "DisabledUserOverride",
                "AllowPSTNUsersToBypassLobby": false,
                "AllowOrganizersToOverrideLobbySettings": false,
                "IPAudioMode": "EnabledOutgoingIncoming",
                "AllowExternalParticipantGiveRequestControl": false,
                "AllowMeetNow": false,
                "AllowMeetingRegistration": false,
                "WhoCanRegister": "Everyone",
                "AllowMeetingReactions": true,
                "AllowTranscription": false,
                "AllowCartCaptionsScheduling": "DisabledUserOverride",
                "AllowPrivateMeetingScheduling": true,
                "AllowMeetingCoach": true,
                "Identity": "Global",
                "SpeakerAttributionMode": "EnabledUserOverride",
                "AllowAnonymousUsersToStartMeeting": false,
                "IPVideoMode": "EnabledOutgoingIncoming",
                "DesignatedPresenterRoleMode": "EveryoneUserOverride",
                "ScreenSharingMode": "EntireScreen",
                "AutoAdmittedUsers": "EveryoneInCompany",
                "AllowPowerPointSharing": true,
                "VideoFiltersMode": "AllFilters",
                "MediaBitRateKb": 50000,
                "NewMeetingRecordingExpirationDays": 120,
                "LiveStreamingMode": "Disabled",
                "AllowBreakoutRooms": true,
                "AllowIPVideo": true,
                "PreferredMeetingProviderForIslandsMode": "TeamsAndSfb",
                "MeetingChatEnabledType": "Enabled",
                "AllowOutlookAddIn": true,
                "AllowDocumentCollaboration": "Enabled"
            }
        },
        {
            "displayName": "EXOTransportRule-Deny Sending Outbound by Default",
            "resourceType": "microsoft.exchange.transportrule",
            "properties": {
                "Name": "Deny Sending Outbound by Default",
                "FromScope": "InOrganization",
                "RemoveOMEv2": false,
                "ExceptIfAttachmentIsPasswordProtected": false,
                "Priority": 1,
                "ExceptIfHasNoClassification": false,
                "ModerateMessageByManager": false,
                "ExceptIfAttachmentProcessingLimitExceeded": false,
                "SentToScope": "NotInOrganization",
                "RecipientAddressType": "Resolved",
                "AttachmentProcessingLimitExceeded": false,
                "DeleteMessage": false,
                "RejectMessageReasonText": "Sending emails to this domain or email address is restricted.",
                "SenderAddressLocation": "HeaderOrEnvelope",
                "Enabled": true,
                "SetAuditSeverity": "Medium",
                "ExceptIfAttachmentIsUnsupported": false,
                "Comments": "This rule restricts ALL outbound email by default unless a preceding rule explicitly allows it.",
                "RuleSubType": "None",
                "HasNoClassification": false,
                "Quarantine": false,
                "AttachmentIsUnsupported": false,
                "AttachmentHasExecutableContent": false,
                "Mode": "Enforce",
                "RemoveRMSAttachmentEncryption": false,
                "Ensure": "Present",
                "StopRuleProcessing": true,
                "AttachmentIsPasswordProtected": false,
                "RemoveOME": false,
                "RuleErrorAction": "Ignore",
                "ExceptIfAttachmentHasExecutableContent": false,
                "RejectMessageEnhancedStatusCode": "5.7.1",
                "RouteMessageOutboundRequireTls": false,
                "ApplyOME": false,
                "ExceptIfRecipientDomainIs": [],
                "ExceptIfAnyOfToHeaderMemberOf": [],
                "AnyOfCcHeaderMemberOf": [],
                "BetweenMemberOf1": [],
                "RedirectMessageTo": [],
                "AttachmentPropertyContainsWords": [],
                "ExceptIfAttachmentPropertyContainsWords": [],
                "ExceptIfAttachmentExtensionMatchesWords": [],
                "AnyOfRecipientAddressMatchesPatterns": [],
                "ExceptIfAnyOfRecipientAddressMatchesPatterns": [],
                "SenderDomainIs": [],
                "ManagerAddresses": [],
                "ExceptIfBetweenMemberOf1": [],
                "ExceptIfAnyOfRecipientAddressContainsWords": [],
                "AnyOfToCcHeader": [],
                "AttachmentNameMatchesPatterns": [],
                "AnyOfCcHeader": [],
                "ExceptIfRecipientADAttributeMatchesPatterns": [],
                "SubjectMatchesPatterns": [],
                "ExceptIfRecipientInSenderList": [],
                "ExceptIfRecipientADAttributeContainsWords": [],
                "ExceptIfAttachmentContainsWords": [],
                "MessageContainsDataClassifications": [],
                "ExceptIfAttachmentNameMatchesPatterns": [],
                "ExceptIfSubjectMatchesPatterns": [],
                "HeaderContainsWords": [],
                "ExceptIfAttachmentMatchesPatterns": [],
                "ExceptIfSenderInRecipientList": [],
                "AnyOfToCcHeaderMemberOf": [],
                "RecipientAddressMatchesPatterns": [],
                "ExceptIfRecipientAddressMatchesPatterns": [],
                "ExceptIfFromMemberOf": [],
                "ExceptIfSubjectContainsWords": [],
                "SubjectContainsWords": [],
                "SentTo": [],
                "IncidentReportContent": [],
                "SenderInRecipientList": [],
                "RecipientAddressContainsWords": [],
                "AddToRecipients": [],
                "ContentCharacterSetContainsWords": [],
                "RecipientInSenderList": [],
                "RecipientADAttributeContainsWords": [],
                "AttachmentMatchesPatterns": [],
                "AttachmentExtensionMatchesWords": [],
                "CopyTo": [],
                "ExceptIfFrom": [],
                "HeaderMatchesPatterns": [],
                "BlindCopyTo": [],
                "RecipientDomainIs": [],
                "ExceptIfSubjectOrBodyContainsWords": [],
                "SenderADAttributeContainsWords": [],
                "ExceptIfAnyOfCcHeaderMemberOf": [],
                "ExceptIfRecipientAddressContainsWords": [],
                "ExceptIfContentCharacterSetContainsWords": [],
                "ExceptIfBetweenMemberOf2": [],
                "ExceptIfSentTo": [],
                "ExceptIfSentToMemberOf": [],
                "AnyOfRecipientAddressContainsWords": [],
                "ExceptIfManagerAddresses": [],
                "ExceptIfAnyOfToCcHeader": [],
                "ExceptIfHeaderContainsWords": [],
                "SubjectOrBodyMatchesPatterns": [],
                "AnyOfToHeader": [],
                "FromMemberOf": [
                    "FinanceTeam@contoso.com"
                ],
                "ExceptIfFromAddressContainsWords": [],
                "SubjectOrBodyContainsWords": [],
                "SenderIpRanges": [],
                "SentToMemberOf": [],
                "ExceptIfAnyOfToHeader": [],
                "AnyOfToHeaderMemberOf": [],
                "ExceptIfFromAddressMatchesPatterns": [],
                "SenderADAttributeMatchesPatterns": [],
                "ModerateMessageByUser": [],
                "ExceptIfHeaderMatchesPatterns": [],
                "ExceptIfSenderADAttributeContainsWords": [],
                "ExceptIfSenderDomainIs": [],
                "ExceptIfSenderIpRanges": [],
                "From": [],
                "ExceptIfAnyOfCcHeader": [],
                "AttachmentContainsWords": [],
                "ExceptIfSubjectOrBodyMatchesPatterns": [],
                "FromAddressContainsWords": [],
                "ExceptIfSenderADAttributeMatchesPatterns": [],
                "BetweenMemberOf2": [],
                "RecipientADAttributeMatchesPatterns": [],
                "ExceptIfAnyOfToCcHeaderMemberOf": [],
                "FromAddressMatchesPatterns": [],
                "ExceptIfMessageContainsDataClassifications": []
            }
        },
        {
            "displayName": "AADGroupLifecyclePolicy",
            "resourceType": "microsoft.entra.grouplifecyclepolicy",
            "properties": {
                "IsSingleInstance": "Yes",
                "GroupLifetimeInDays@odata.type": "#Int64",
                "GroupLifetimeInDays": 99,
                "Ensure": "Present",
                "ManagedGroupTypes": "Selected",
                "AlternateNotificationEmails": [
                    "Admin@M365x73318397.onmicrosoft.com"
                ]
            }
        }
        ...
}
```

<p>Tenant Configuration Management is not a file management solution, meaning that you shouldn't be using it to store your snapshots (which expire after 7 days anyway). Instead, you should download their content and store in a platform such as an Application Lifecycle Management solution such as Azure DevOPS or GitHub.</p>

<h2>Monitoring</h2>
<p>Once you have a configuration template, either obtained by a Snapshot or by writing them manually, you will want to proceed to creating a monitor with it. In the context of Tenant Configuration Management, you can think of a monitor as a container that executes on a schedule to validate that the settings defined in the configuration template are still set correctly. During the public preview, the monitors will execute every 6 hours. Microsoft will be providing other options to run the monitors more frequently over the coming months. This scheduled is not based on the time the monitor is created, but are rather getting executed at a specific time of the day.</p>

<p>Just like it was the case for the snapshot operation, when executing, the monitor will act on behalf of the Unified Tenant Configuration Management service principal. You can use my TCM-Utility PowerShell module to scan any existing configuration template and determine what the required permissions to monitor it would be, you simply need to run the following line of PowerShell code, where the Path parameter points to the local copy of the snapshot I downloaded from the previous section.</p>

```powershell
Test-TCMConfigurationTemplate -Path "C:\temp\ConfigTemplate.json"
```

<p>Monitors also have a configuration mode, which during the public preview is enforced to MonitorOnly. This mode means that the monitors will never attempt to change any configuration settings. Instead it will only perform read-only operations and report any drifts detected. Therefore, just like for the snapshot scenario, you are only required to grant the Read permissions and roles returned by my PowerShell utility.</p>

<p>After confirming that the Unified Tenant COnfiguration Management service principal has all the required permissions and roles assigned, we can call the monitor APIs to create our monitor, passing in the content of our configuration template as a parameter.</p>

```json
POST https://graph.microsoft.com/beta/admin/configurationManagement/configurationMonitors/
{
    "displayName": "Name of your monitor", 
    "description": "This is a Monitor", 
    "baseline": CONTENT OF YOUR CONFIGURATION TEMPLATE
}
```

<p>If successfully created, the monitor APIs will return a response similar to the following:</p>

```json
{
    "@odata.context": "https://graph.microsoft.com/beta/$metadata#admin/configurationManagement/configurationMonitors/$entity",
    "id": "92a07b41-f6fb-434c-9b80-d345c735d5df",
    "displayName": "Name of your monitor",
    "description": "This is a Monitor",
    "tenantId": "xxxxxxxxxxxxxxxxxxxxx",
    "status": "active",
    "monitorRunFrequencyInHours": 6,
    "mode": "monitorOnly",
    "createdDateTime": "2026-01-27T23:00:34.516085Z",
    "lastModifiedDateTime": "2026-01-27T23:00:34.5305959Z",
    "runAsUTCMServicePrincipal": true,
    "inactivationReason": null,
    "createdBy": {
        "user": {
            "id": "74b56a93-a473-40eb-a011-555555550555",
            "displayName": "MOD Administrator"
        },
        "application": {
            "id": null,
            "displayName": null
        }
    },
    "runningOnBehalfOf": {
        "user": null,
        "application": {
            "id": "03b07b79-c5bc-4b5e-9bfa-13acf4a99998",
            "displayName": "Unified Tenant Configuration Management"
        }
    },
    "lastModifiedBy": {
        "user": {
            "id": "74b56a93-a473-40eb-a011-555555550555",
            "displayName": "MOD Administrator"
        },
        "application": {
            "id": null,
            "displayName": null
        }
    },
    "parameters": {}
}
```

<p>The above payload indicates that the monitor was successfully created and is executed on the 6-hours schedule. After some time, the monitor will have executed and generated what we call a MonitorResult entry, which is a metadata slip that indicates if the monitor run was successful, how long it took to complete, and whether or not it detected any configuration drifts. To retrieve these MonitorResults, you can make a call similar to the one below, where the MonitorId is the value received from the response above.</p>

```json
GET  https://graph.microsoft.com/beta/admin/configurationManagement/configurationMonitoringResults?$filter=MonitorId eq '92a07b41-f6fb-434c-9b80-d345c735d5df'
{
    "@odata.context": "https://graph.microsoft.com/beta/$metadata#admin/configurationManagement/configurationMonitoringResults",
    "@microsoft.graph.tips": "This request only returns a subset of the resource's properties. Your app will need to use $select to return non-default properties. To find out what other properties are available for this resource see https://learn.microsoft.com/graph/api/resources/configurationMonitoringResult",
    "value": [
        {
            "id": "2fbd827d-d2c2-42fb-bb1e-358c0cc64b77",
            "monitorId": "92a07b41-f6fb-434c-9b80-d345c735d5df",
            "tenantId": "xxxxxxxxxxxx",
            "runInitiationDateTime": "2026-01-27T06:00:32.3210059Z",
            "runCompletionDateTime": "2026-01-27T06:01:24.4507329Z",
            "runStatus": "successful",
            "driftsCount": 1,
            "driftsFixed": 0,
            "runType": "monitor"
        }
    ]
}
```
<p>We can seen from the response above that the monitor did detect a drift based on the <strong>driftsCount</strong> property. The MonitorResult doesn't provide details about what the actual drift is. In order to get those details, we need to call into the configuration drift APIs as follow:</p>

```json
GET https://graph.microsoft.com/beta/admin/configurationManagement/configurationDrifts?$filter=monitorId eq '92a07b41-f6fb-434c-9b80-d345c735d5df'
{
    "@odata.context": "https://graph.microsoft.com/beta/$metadata#admin/configurationManagement/configurationDrifts",
    "@microsoft.graph.tips": "Use $select to choose only the properties your app needs, as this can lead to performance improvements. For example: GET admin/configurationManagement/configurationDrifts?$select=baselineResourceDisplayName,driftedProperties",
    "value": [
        {
            "id": "77098573-dea7-4d84-bcfb-8086ea827d13",
            "monitorId": "92a07b41-f6fb-434c-9b80-d345c735d5df",
            "tenantId": "xxxxxxxxxxxxx",
            "resourceType": "microsoft.entra.conditionalaccesspolicy",
            "baselineResourceDisplayName": "AADConditionalAccessPolicy-Multifactor authentication for Microsoft partners",
            "firstReportedDateTime": "2026-01-27T06:01:23.9659605Z",
            "status": "active",
            "resourceInstanceIdentifier": {
                "DisplayName": "Multifactor authentication for Microsoft partners"
            },
            "driftedProperties": [
                {
                    "propertyName": "ExcludeUsers",
                    "currentValue": [],
                    "desiredValue": [
                        "admin@contoso.com"
                    ]
                },
                {
                    "propertyName": "ExcludeRoles",
                    "currentValue": [],
                    "desiredValue": [
                        "Directory Synchronization Accounts",
                        "On Premises Directory Sync Account"
                    ]
                },
                {
                    "propertyName": "IncludeUsers",
                    "currentValue": [
                        "None"
                    ],
                    "desiredValue": [
                        "All"
                    ]
                },
                {
                    "propertyName": "State",
                    "currentValue": "disabled",
                    "desiredValue": "enabledForReportingButNotEnforced"
                }
            ]
        }
    ]
}
```

<p>Based on the response, we can see that the detected drift is on an Entra Id conditional access policies, named <strong>Multifactor authentication for Microsoft partners</strong>. We can also see from the above, that while the monitor reported 1 drift, there are several properties that drifted: ExcludeUsers, ExcludeRoles, IncludeUsers and State. For example, the State of the conditional access policy was detected to be disabled, whereas the desired value, as defined by the configuration template, is supposed to be set to enabledForReportingButNotEnforced. Currently, since we do not yet have the ability to have monitors automatically remediate to detected drifts, we would have to go and manually correct the detected drifts. Once the drifts have been corrected, the next monitor run should report that there are no longer any active drifts:</p>

```json
{
    "@odata.context": "https://graph.microsoft.com/beta/$metadata#admin/configurationManagement/configurationMonitoringResults",
    "@microsoft.graph.tips": "This request only returns a subset of the resource's properties. Your app will need to use $select to return non-default properties. To find out what other properties are available for this resource see https://learn.microsoft.com/graph/api/resources/configurationMonitoringResult",
    "value": [
        {
            "id": "84982abc-bdfe-4395-9f4f-c217504fa5f2",
            "monitorId": "92a07b41-f6fb-434c-9b80-d345c735d5df",
            "tenantId": "xxxxxxxxxxxxxxx",
            "runInitiationDateTime": "2026-01-28T00:00:10.5912228Z",
            "runCompletionDateTime": "2026-01-28T00:01:10.8837901Z",
            "runStatus": "successful",
            "driftsCount": 0,
            "driftsFixed": 0,
            "runType": "monitor"
        }
    ]
}
```

<h2>Conclusion</h2>
<p>As you can see from the examples provided above, while we don't yet have the ability for the Tenant Configuration Management APIs to make configuration changes to enforce a organization's secure posture, having the ability to continuously monitor your tenants for unwanted configuration drifts is a true game changer. If you have any feedback for the team or encounter any issues during preview, Microsoft recommends you contact <a href="mailto:feedback@microsoft.com">Microsoft Feedback</a>, making sure that you mention Tenant Configuration Management APIs.</p>