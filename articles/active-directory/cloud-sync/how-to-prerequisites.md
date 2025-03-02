---
title: 'Prerequisites for Azure AD Connect cloud sync in Azure AD'
description: This article describes the prerequisites and hardware requirements you need for cloud sync.
services: active-directory
author: billmath
manager: karenhoran
ms.service: active-directory
ms.workload: identity
ms.topic: how-to
ms.date: 10/19/2021
ms.subservice: hybrid
ms.author: billmath
ms.collection: M365-identity-device-management
---

# Prerequisites for Azure AD Connect cloud sync
This article provides guidance on how to choose and use Azure Active Directory (Azure AD) Connect cloud sync as your identity solution.

## Cloud provisioning agent requirements
You need the following to use Azure AD Connect cloud sync:

- Domain Administrator or Enterprise Administrator credentials to create the Azure AD Connect Cloud Sync gMSA (group Managed Service Account) to run the agent service.	
- A hybrid identity administrator account for your Azure AD tenant that is not a guest user.
- An on-premises server for the provisioning agent with Windows 2016 or later.  This server should be a tier 0 server based on the [Active Directory administrative tier model](/windows-server/identity/securing-privileged-access/securing-privileged-access-reference-material).  Installing the agent on a domain controller is supported.
- High availability refers to the Azure AD Connect cloud sync's ability to operate continuously without failure for a long time.  By having multiple active agents installed and running, Azure AD Connect cloud sync can continue to function even if one agent should fail.  Microsoft recommends having 3 active agents installed for high availability.
- On-premises firewall configurations.

## Group Managed Service Accounts
A group Managed Service Account is a managed domain account that provides automatic password management, simplified service principal name (SPN) management,the ability to delegate the management to other administrators, and also extends this functionality over multiple servers.  Azure AD Connect Cloud Sync supports and uses a gMSA for running the agent.  You will be prompted for administrative credentials during setup, in order to create this account.  The account will appear as (domain\provAgentgMSA$).  For more information on a gMSA, see [Group Managed Service Accounts](/windows-server/security/group-managed-service-accounts/group-managed-service-accounts-overview) 

### Prerequisites for gMSA:
1.	The Active Directory schema in the gMSA domain's forest needs to be updated to Windows Server 2012 or later.
2.	[PowerShell RSAT modules](/windows-server/remote/remote-server-administration-tools) on a domain controller
3.	At least one domain controller in the domain must be running Windows Server 2012 or later.
4.	A domain joined server where the agent is being installed needs to be either Windows Server 2016 or later.

### Custom gMSA account
If you are creating a custom gMSA account, you need to ensure that the account has the following permissions.

|Type |Name |Access |Applies To| 
|-----|-----|-----|-----|
|Allow |gMSA Account |Read all properties |Descendant device objects| 
|Allow |gMSA Account|Read all properties |Descendant InetOrgPerson objects| 
|Allow |gMSA Account |Read all properties |Descendant Computer objects| 
|Allow |gMSA Account |Read all properties |Descendant foreignSecurityPrincipal objects| 
|Allow |gMSA Account |Full control |Descendant Group objects| 
|Allow |gMSA Account |Read all properties |Descendant User objects| 
|Allow |gMSA Account |Read all properties |Descendant Contact objects| 
|Allow |gMSA Account |Create/delete User objects|This object and all descendant objects| 

For steps on how to upgrade an existing agent to use a gMSA account see [Group Managed Service Accounts](how-to-install.md#group-managed-service-accounts).

#### Create gMSA account with PowerShell
You can use the following PowerShell script to create a custom gMSA account.  Then you can use the [cloud sync gMSA cmdlets](how-to-gmsa-cmdlets.md) to apply more granular permissions.

```powershell
# Filename:    1_SetupgMSA.ps1
# Description: Creates and installs a custom gMSA account for use with Azure AD Connect cloud sync.
#
# DISCLAIMER:
# Copyright (c) Microsoft Corporation. All rights reserved. This 
# script is made available to you without any express, implied or 
# statutory warranty, not even the implied warranty of 
# merchantability or fitness for a particular purpose, or the 
# warranty of title or non-infringement. The entire risk of the 
# use or the results from the use of this script remains with you.
#
#
#
#
# Declare variables
$Name = 'provAPP1gMSA'
$Description = "Azure AD Cloud Sync service account for APP1 server"
$Server = "APP1.contoso.com"
$Principal = Get-ADGroup 'Domain Computers'

# Create service account in Active Directory
New-ADServiceAccount -Name $Name `
-Description $Description `
-DNSHostName $Server `
-ManagedPasswordIntervalInDays 30 `
-PrincipalsAllowedToRetrieveManagedPassword $Principal `
-Enabled $True `
-PassThru

# Install the new service account on Azure AD Cloud Sync server
Install-ADServiceAccount -Identity $Name
```

For additional information on the cmdlets above, see [Getting Started with Group Managed Service Accounts](/previous-versions/windows/it-pro/windows-server-2012-R2-and-2012/jj128431(v=ws.11)?redirectedfrom=MSDN).

### In the Azure Active Directory admin center

1. Create a cloud-only hybrid identity administrator account on your Azure AD tenant. This way, you can manage the configuration of your tenant if your on-premises services fail or become unavailable. Learn about how to [add a cloud-only hybrid identity administrator account](../fundamentals/add-users-azure-active-directory.md). Finishing this step is critical to ensure that you don't get locked out of your tenant.
1. Add one or more [custom domain names](../fundamentals/add-custom-domain.md) to your Azure AD tenant. Your users can sign in with one of these domain names.

### In your directory in Active Directory

Run the [IdFix tool](/office365/enterprise/prepare-directory-attributes-for-synch-with-idfix) to prepare the directory attributes for synchronization.

### In your on-premises environment

1. Identify a domain-joined host server running Windows Server 2016 or greater with a minimum of 4-GB RAM and .NET 4.7.1+ runtime.

2. The PowerShell execution policy on the local server must be set to Undefined or RemoteSigned.

3. If there's a firewall between your servers and Azure AD, configure the following items:
    - Ensure that agents can make *outbound* requests to Azure AD over the following ports:

      | Port number | How it's used |
      | --- | --- |
      | **80** | Downloads the certificate revocation lists (CRLs) while validating the TLS/SSL certificate.  |
      | **443** | Handles all outbound communication with the service. |
      | **8080** (optional) | Agents report their status every 10 minutes over port 8080, if port 443 is unavailable. This status is displayed in the Azure AD portal. |

    - If your firewall enforces rules according to the originating users, open these ports for traffic from Windows services that run as a network service.
    - If your firewall or proxy allows you to specify safe suffixes, add connections to \*.msappproxy.net and \*.servicebus.windows.net. If not, allow access to the [Azure datacenter IP ranges](https://www.microsoft.com/download/details.aspx?id=41653), which are updated weekly.
    - Your agents need access to login.windows.net and login.microsoftonline.com for initial registration. Open your firewall for those URLs as well.
    - For certificate validation, unblock the following URLs: mscrl.microsoft.com:80, crl.microsoft.com:80, ocsp.msocsp.com:80, and www\.microsoft.com:80. These URLs are used for certificate validation with other Microsoft products, so you might already have these URLs unblocked.

    >[!NOTE]
    > Installing the cloud provisioning agent on Windows Server Core is not supported.

### Additional requirements

- [Microsoft .NET Framework 4.7.1](https://dotnet.microsoft.com/download/dotnet-framework/net471) 

#### TLS requirements

> [!NOTE]
> Transport Layer Security (TLS) is a protocol that provides for secure communications. Changing the TLS settings affects the entire forest. For more information, see [Update to enable TLS 1.1 and TLS 1.2 as default secure protocols in WinHTTP in Windows](https://support.microsoft.com/help/3140245/update-to-enable-tls-1-1-and-tls-1-2-as-default-secure-protocols-in-wi).

The Windows server that hosts the Azure AD Connect cloud provisioning agent must have TLS 1.2 enabled before you install it.

To enable TLS 1.2, follow these steps.

1. Set the following registry keys:

    ```
    [HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.2]
    [HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.2\Client] "DisabledByDefault"=dword:00000000 "Enabled"=dword:00000001
    [HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.2\Server] "DisabledByDefault"=dword:00000000 "Enabled"=dword:00000001
    [HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\.NETFramework\v4.0.30319] "SchUseStrongCrypto"=dword:00000001
    ```

1. Restart the server.
## NTLM requirement

You should not enable NTLM on the Windows Server that is running the Azure AD Connect Provisioning Agent and if it is enabled you should make sure you disable it. 

## Known limitations

The following are known limitations:

### Delta Synchronization

- Group scope filtering for delta sync does not support more than 50,000 members.
- When you delete a group that's used as part of a group scoping filter, users who are members of the group, don't get deleted. 
- When you rename the OU or group that's in scope, delta sync will not remove the users.

### Provisioning Logs
- Provisioning logs do not clearly differentiate between create and update operations.  You may see a create operation for an update and an update operation for a create.

### Group re-naming or OU re-naming
- If you rename a group or OU in AD that's in scope for a given configuration, the cloud sync job will not be able to recognize the name change in AD. The job won't go into quarantine and will remain healthy.

### Scoping filter
When using OU scoping filter
- You can only sync up to 59 separate OUs for a given configuration. 
- Nested OUs are supported (that is, you **can** sync an OU that has 130 nested OUs, but you **cannot** sync 60 separate OUs in the same configuration). 

### Password Hash Sync
- Using password hash sync with InetOrgPerson is not supported.


## Next steps 

- [What is provisioning?](what-is-provisioning.md)
- [What is Azure AD Connect cloud sync?](what-is-cloud-sync.md)
