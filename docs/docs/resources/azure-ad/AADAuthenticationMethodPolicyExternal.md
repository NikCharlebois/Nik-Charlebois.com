﻿# AADAuthenticationMethodPolicyExternal

## Parameters

| Parameter | Attribute | DataType | Description | Allowed Values |
| --- | --- | --- | --- | --- |
| **ExcludeTargets** | Write | MSFT_AADAuthenticationMethodPolicyExternalExcludeTarget[] | Displayname of the groups of users that are excluded from a policy. | |
| **IncludeTargets** | Write | MSFT_AADAuthenticationMethodPolicyExternalIncludeTarget[] | Displayname of the groups of users that are included from a policy. | |
| **OpenIdConnectSetting** | Write | MSFT_AADAuthenticationMethodPolicyExternalOpenIdConnectSetting | Open ID Connection settings used by this external authentication method. | |
| **State** | Write | String | The state of the policy. Possible values are: enabled, disabled. | `enabled`, `disabled` |
| **AppId** | Write | String | The appId for the app registration in Microsoft Entra ID representing the integration with the external provider. | |
| **DisplayName** | Key | String | The displayName of the authentication policy configuration. Read-only. | |
| **Ensure** | Write | String | Present ensures the policy exists, absent ensures it is removed. | `Present`, `Absent` |
| **Credential** | Write | PSCredential | Credentials of the Admin | |
| **ApplicationId** | Write | String | Id of the Azure Active Directory application to authenticate with. | |
| **TenantId** | Write | String | Id of the Azure Active Directory tenant used for authentication. | |
| **ApplicationSecret** | Write | PSCredential | Secret of the Azure Active Directory tenant used for authentication. | |
| **CertificateThumbprint** | Write | String | Thumbprint of the Azure Active Directory application's authentication certificate to use for authentication. | |
| **ManagedIdentity** | Write | Boolean | Managed ID being used for authentication. | |
| **AccessTokens** | Write | StringArray[] | Access token used for authentication. | |

### MSFT_AADAuthenticationMethodPolicyExternalExcludeTarget

#### Parameters

| Parameter | Attribute | DataType | Description | Allowed Values |
| --- | --- | --- | --- | --- |
| **Id** | Write | String | The object identifier of an Azure AD group. | |
| **TargetType** | Write | String | The type of the authentication method target. Possible values are: group and unknownFutureValue. | `user`, `group`, `unknownFutureValue` |

### MSFT_AADAuthenticationMethodPolicyExternalIncludeTarget

#### Parameters

| Parameter | Attribute | DataType | Description | Allowed Values |
| --- | --- | --- | --- | --- |
| **Id** | Write | String | The object identifier of an Azure AD group. | |
| **TargetType** | Write | String | The type of the authentication method target. Possible values are: group and unknownFutureValue. | `user`, `group`, `unknownFutureValue` |

### MSFT_AADAuthenticationMethodPolicyExternalOpenIdConnectSetting

#### Parameters

| Parameter | Attribute | DataType | Description | Allowed Values |
| --- | --- | --- | --- | --- |
| **ClientId** | Write | String | The Microsoft Entra ID's client ID as generated by the provider or admin to identify Microsoft Entra ID. | |
| **DiscoveryUrl** | Write | String | The host URL of the external identity provider's OIDC discovery endpoint. | |


## Description

Azure AD Authentication Method Policy External

## Permissions

### Microsoft Graph

To authenticate with the Microsoft Graph API, this resource required the following permissions:

#### Delegated permissions

- **Read**

    - Policy.ReadWrite.AuthenticationMethod, Policy.Read.All

- **Update**

    - Policy.ReadWrite.AuthenticationMethod, Policy.Read.All

#### Application permissions

- **Read**

    - Policy.ReadWrite.AuthenticationMethod, Policy.Read.All

- **Update**

    - Policy.ReadWrite.AuthenticationMethod, Policy.Read.All

## Examples

### Example 1

This example is used to test new resources and showcase the usage of new resources being worked on.
It is not meant to use as a production baseline.

```powershell
Configuration Example
{
    param(
        [Parameter()]
        [System.String]
        $ApplicationId,

        [Parameter()]
        [System.String]
        $TenantId,

        [Parameter()]
        [System.String]
        $CertificateThumbprint
    )
    Import-DscResource -ModuleName Microsoft365DSC

    node localhost
    {
        AADAuthenticationMethodPolicyExternal "AADAuthenticationMethodPolicyExternal-Cisco Duo"
        {
            ApplicationId         = $ApplicationId
            TenantId              = $TenantId
            CertificateThumbprint = $CertificateThumbprint
            AppId                 = "e35c54ff-bd24-4c52-921a-4b90a35808eb";
            DisplayName           = "Cisco Duo";
            Ensure                = "Present";
            ExcludeTargets        = @(
                MSFT_AADAuthenticationMethodPolicyExternalExcludeTarget{
                    Id = 'Design'
                    TargetType = 'group'
                }
            );
            IncludeTargets        = @(
                MSFT_AADAuthenticationMethodPolicyExternalIncludeTarget{
                    Id = 'Contoso'
                    TargetType = 'group'
                }
            );
            OpenIdConnectSetting  = MSFT_AADAuthenticationMethodPolicyExternalOpenIdConnectSetting{
                discoveryUrl = 'https://graph.microsoft.com/'
                clientId = '7698a352-4939-486e-9974-4ea5aff93f74'
            };
            State                 = "disabled";
        }
    }
}
```

### Example 2

This example is used to test new resources and showcase the usage of new resources being worked on.
It is not meant to use as a production baseline.

```powershell
Configuration Example
{
    param(
        [Parameter()]
        [System.String]
        $ApplicationId,

        [Parameter()]
        [System.String]
        $TenantId,

        [Parameter()]
        [System.String]
        $CertificateThumbprint
    )
    Import-DscResource -ModuleName Microsoft365DSC

    Node localhost
    {
        AADAuthenticationMethodPolicyExternal "AADAuthenticationMethodPolicyExternal-Cisco Duo"
        {
            ApplicationId         = $ApplicationId
            TenantId              = $TenantId
            CertificateThumbprint = $CertificateThumbprint
            AppId                 = "e35c54ff-bd24-4c52-921a-4b90a35808eb";
            DisplayName           = "Cisco Duo";
            Ensure                = "Present";
            ExcludeTargets        = @(
                MSFT_AADAuthenticationMethodPolicyExternalExcludeTarget{
                    Id = 'Design'
                    TargetType = 'group'
                }
            );
            IncludeTargets        = @(
                MSFT_AADAuthenticationMethodPolicyExternalIncludeTarget{
                    Id = 'Contoso'
                    TargetType = 'group'
                }
            );
            OpenIdConnectSetting  = MSFT_AADAuthenticationMethodPolicyExternalOpenIdConnectSetting{
                discoveryUrl = 'https://graph.microsoft.com/'
                clientId = '7698a352-4939-486e-9974-4ea5aff93f74'
            };
            State                 = "disabled";
        }
    }
}
```

### Example 3

This example is used to test new resources and showcase the usage of new resources being worked on.
It is not meant to use as a production baseline.

```powershell
Configuration Example
{
    param(
        [Parameter()]
        [System.String]
        $ApplicationId,

        [Parameter()]
        [System.String]
        $TenantId,

        [Parameter()]
        [System.String]
        $CertificateThumbprint
    )
    Import-DscResource -ModuleName Microsoft365DSC

    Node localhost
    {
        AADAuthenticationMethodPolicyExternal "AADAuthenticationMethodPolicyExternal-Cisco Duo"
        {
            ApplicationId         = $ApplicationId
            TenantId              = $TenantId
            CertificateThumbprint = $CertificateThumbprint
            DisplayName           = "Cisco Duo";
            Ensure                = "Absent";
        }
    }
}
```
