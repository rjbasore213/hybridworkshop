Integrate Azure Stack HCI 20H2 with Azure
==============
Overview
-----------

With your Azure Stack HCI 20H2 cluster deployed successfully, you need to register this cluster to unlock full functionality.

Contents
-----------
- [Overview](#overview)
- [Contents](#contents)
- [Prerequisites for registration](#prerequisites-for-registration)
- [Complete Registration](#complete-registration)
- [Next Steps](#next-steps)
- [Product improvements](#product-improvements)
- [Raising issues](#raising-issues)

Azure Stack HCI 20H2 is delivered as an Azure service and needs to register within 30 days of installation per the Azure Online Services Terms.  With our cluster configured, we'll now register your Azure Stack HCI 20H2 cluster with **Azure Arc** for monitoring, support, billing, and hybrid services. Upon registration, an Azure Resource Manager resource is created to represent each on-premises Azure Stack HCI 20H2 cluster, effectively extending the Azure management plane to Azure Stack HCI 20H2. Information is periodically synced between the Azure resource and the on-premises cluster.  One great aspect of Azure Stack HCI 20H2, is that the Azure Arc registration is a native capability of Azure Stack HCI 20H2, so there is no agent required.

**NOTE** - After registering your Azure Stack HCI 20H2 cluster, the **first 30 days usage will be free**.

Prerequisites for registration
-----------

Firstly, **you need an Azure Stack HCI 20H2 cluster**, which we've just created, so you're good there.

Your nodes need to have **internet connectivity** in order to register and communicate with Azure.  If you've been running nested in Azure, you should have this already set up correctly, but if you're running nested on a local physical machine, make any necessary adjustments to your InternalNAT switch to allow internet connections through to your nested Azure Stack HCI 20H2 nodes.

You'll need an **Azure subscription**, along with appropriate **Azure Active Directory permissions** to complete the registration process. If you don't already have them, you'll need to ask your Azure AD administrator to grant permissions or delegate them to you.  You can learn more about this below.

### What happens when you register Azure Stack HCI 20H2? ###
When you register your Azure Stack HCI 20H2 cluster, the process creates an Azure Resource Manager (ARM) resource to represent the on-prem cluster. This resource is provisioned by an Azure resource provider (RP) and placed inside a resource group, within your chosen Azure subscription.  If these Azure concepts are new to you, you can check out an [overview of them, and more, here](https://docs.microsoft.com/en-us/azure/azure-resource-manager/management/overview "Azure Resource Manager overview").

![ARM architecture for Azure Stack HCI 20H2](/media/azure_arm.png "ARM architecture for Azure Stack HCI 20H2")

In addition to creating an Azure resource in your subscription, registering Azure Stack HCI 20H2 creates an app identity, conceptually similar to a user, in your Azure Active Directory tenant. The app identity inherits the cluster name. This identity acts on behalf on the Azure Stack HCI 20H2 cloud service, as appropriate, within your subscription.

### Understanding required Azure Active Directory permissions ###
If the user who registers Azure Stack HCI 20H2 is an Azure Active Directory global administrator or has been delegated sufficient permissions, this all happens automatically, and no additional action is required. If not, approval may be needed from your Azure Active Directory global administrator (or someone with appropriate permissions) to complete registration. Your global administrator can either explicitly grant consent to the app, or they can delegate permissions so that you can grant consent to the app.

![Azure Active Directory Permissions](/media/aad_permissions.png "Azure Active Directory Permissions")

The user who runs Register-AzStackHCI needs Azure AD permissions to:

* Create/Get/Set/Remove Azure AD applications (New/Get/Set/Remove-AzureADApplication)
* Create/Get Azure AD service principal (New/Get-New-AzureADServicePrincipal)
* Manage AD application secrets (New/Get/Remove-AzureADApplicationKeyCredential)
* Grant consent to use specific application permissions (New/Get/Remove AzureADServiceAppRoleAssignments)

There are three ways in which this can be accomplished.

#### Option 1: Allow any user to register applications ####

In Azure Active Directory, navigate to User settings > **App registrations**. Under **Users can register applications**, select **Yes**.

This will allow any user to register applications. However, the user will still require the Azure AD admin to grant consent during cluster registration. Note that this is a tenant level setting, so it may not be suitable for large enterprise customers.

#### Option 2: Assign Cloud Application Administration role ####

Assign the built-in "Cloud Application Administration" Azure AD role to the user. This will allow the user to register clusters without the need for additional AD admin consent.

#### Option 3: Create a custom AD role and consent policy ####

The most restrictive option is to create a custom AD role with a custom consent policy that delegates tenant-wide admin consent for required permissions to the Azure Stack HCI Service. When assigned this custom role, users are able to both register and grant consent without the need for additional AD admin consent.

**NOTE** - This option requires an Azure AD Premium license and uses custom AD roles and custom consent policy features which are currently in public preview.

If you choose to perform Option 3, you'll need to follow these steps on **HybridHost001**, which we'll demonstrate mainly through PowerShell.

1. Firstly, configure the appropriate AzureAD modules, then **Connect to Azure AD**, and when prompted, **log in with your appropriate credentials**

```powershell
Remove-Module AzureAD -ErrorAction SilentlyContinue -Force
Install-Module AzureADPreview -AllowClobber -Force
Connect-AzureAD
```

2. Create a **custom consent policy**:

```powershell
New-AzureADMSPermissionGrantPolicy -Id "AzSHCI-registration-consent-policy" `
    -DisplayName "Azure Stack HCI registration admin app consent policy" `
    -Description "Azure Stack HCI registration admin app consent policy"
```

3. Add a condition that includes required app permissions for Azure Stack HCI service, which carries the app ID 1322e676-dee7-41ee-a874-ac923822781c. Note that the following permissions are for the GA release of Azure Stack HCI, and will not work with Public Preview unless you have applied the [November 23, 2020 Preview Update (KB4586852)](https://docs.microsoft.com/en-us/azure-stack/hci/release-notes "November 23, 2020 Preview Update (KB4586852)") to every server in your cluster and have downloaded the Az.StackHCI module version 0.4.1 or later.

```powershell
New-AzureADMSPermissionGrantConditionSet -PolicyId "AzSHCI-registration-consent-policy" `
    -ConditionSetType "includes" -PermissionType "application" -ResourceApplication "1322e676-dee7-41ee-a874-ac923822781c" `
    -Permissions "bbe8afc9-f3ba-4955-bb5f-1cfb6960b242", "8fa5445e-80fb-4c71-a3b1-9a16a81a1966", `
    "493bd689-9082-40db-a506-11f40b68128f", "2344a320-6a09-4530-bed7-c90485b5e5e2"
```

4. Grant permissions to allow registering Azure Stack HCI, noting the custom consent policy created in Step 2:

```powershell
$displayName = "Azure Stack HCI Registration Administrator "
$description = "Custom AD role to allow registering Azure Stack HCI "
$templateId = (New-Guid).Guid
$allowedResourceAction =
@(
    "microsoft.directory/applications/createAsOwner",
    "microsoft.directory/applications/delete",
    "microsoft.directory/applications/standard/read",
    "microsoft.directory/applications/credentials/update",
    "microsoft.directory/applications/permissions/update",
    "microsoft.directory/servicePrincipals/appRoleAssignedTo/update",
    "microsoft.directory/servicePrincipals/appRoleAssignedTo/read",
    "microsoft.directory/servicePrincipals/appRoleAssignments/read",
    "microsoft.directory/servicePrincipals/createAsOwner",
    "microsoft.directory/servicePrincipals/credentials/update",
    "microsoft.directory/servicePrincipals/permissions/update",
    "microsoft.directory/servicePrincipals/standard/read",
    "microsoft.directory/servicePrincipals/managePermissionGrantsForAll.AzSHCI-registration-consent-policy"
)
$rolePermissions = @{'allowedResourceActions' = $allowedResourceAction }
```

5. Create the new custom AD role:

```powershell
$customADRole = New-AzureADMSRoleDefinition -RolePermissions $rolePermissions `
    -DisplayName $displayName -Description $description -TemplateId $templateId -IsEnabled $true
```

6. Assign the new custom AD role to the user who will register the Azure Stack HCI cluster with Azure by following [these instructions](https://docs.microsoft.com/en-us/azure/active-directory/fundamentals/active-directory-users-assign-role-azure-portal "Guidance on creating a custom Azure AD role").

Complete Registration
-----------

To complete registration, you have 2 options - you can use **Windows Admin Center**, or you can use **PowerShell**. For this lab, it's recommended to use the PowerShell approach, due to a few unpredictible erros in the lab environment, likely due to WAC installed on the domain controller.

#### Option 1 - Register using PowerShell ####
We're going to perform the registration from the **HybridHost001** machine, which we've been using with the Windows Admin Center.

1. On **HybridHost001**, open **PowerShell ISE as administrator**
2. In the file menu, click **Open** and navigate to **V:\Source** and open **Register-AzSHCI**
3. When the script file opens, select and run the following code to install the PowerShell Module for Azure Stack HCI 20H2 on that machine.

```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Force
Install-Module Az.StackHCI
```

**NOTE** - You may recieve a message that **PowerShellGet requires NuGet Provider...** - read the full message, and then click **Yes** to allow the appropriate dependencies to be installed. You may receive a second prompt to **install the modules from the PSGallery** - click **Yes to All** to proceed.

In addition, in future releases, installing the Azure PowerShell **Az** modules will include **StackHCI**, however today, you have to install this module specifically, using the command **Install-Module Az.StackHCI**

4. With the Az.StackHCI modules installed, it's now time to register your Azure Stack HCI 20H2 cluster to Azure, however first, it's worth exploring how to check existing registration status.  The following code assumes you are still in the remote PowerShell session open from the previous commands.

```powershell
Invoke-Command -ComputerName AZSHCINODE01 -ScriptBlock {
    Get-AzureStackHCI
}
```

![Check the registration status of the Azure Stack HCI 20H2 cluster](/media/reg_check.png "Check the registration status of the Azure Stack HCI 20H2 cluster")

As you can see from the result, the cluster is yet to be registered, and the cluster status identifies as **Clustered**. Azure Stack HCI 20H2 needs to register within 30 days of installation per the Azure Online Services Terms. If not clustered after 30 days, the **ClusterStatus** will show **OutOfPolicy**, and if not registered after 30 days, the **RegistrationStatus** will show **OutOfPolicy**.

5. To register the cluster, you'll first need to get your **Azure subscription ID**.  An easy way to do this is to quickly **log into https://portal.azure.com**, and in the **search box** at the top of the screen, search for **subscriptions** and then click on **Subscriptions**

![Azure Subscriptions](/media/azure_subscriptions_ga.png "Azure Subscriptions")

6. Your **subscription** should be shown in the main window.  If you have more than one subscription listed here, click the correct one, and in the new blade, copy the **Subscription ID**.

**NOTE** - If you don't see your desired subscription, in the top right-corner of the Azure portal, click on your user account, and click **Switch directory**, then select an alternative directory.  Once in the chosen directory, repeat the search for your **Subscription ID** and copy it down.

7. With your **Subscription ID** in hand, you can **register using the following Powershell commands**, from your open PowerShell window.

```powershell
$azshciNodeCreds = Get-Credential -UserName "hybrid\azureuser" -Message "Enter the hybrid\azureuser password"
Register-AzStackHCI `
    -SubscriptionId "your-subscription-ID-here" `
    -ResourceName "azshciclus" `
    -ResourceGroupName "AZSHCICLUS_RG" `
    -Region "EastUS" `
    -EnvironmentName "AzureCloud" `
    -ComputerName "AZSHCINODE01.hybrid.local" `
    –Credential $azshciNodeCreds
```

Of these commands, many are optional:

* **-ResourceName** - If not declared, the Azure Stack HCI 20H2 cluster name is used
* **-ResourceGroupName** - If not declared, the Azure Stack HCI 20H2 cluster plus the suffix "-rg" is used
* **-Region** - If not declared, "EastUS" will be used.  Additional regions are supported, with the longer term goal to integrate with Azure Arc in all Azure regions.
* **-EnvironmentName** - If not declared, "AzureCloud" will be used, but allowed values will include additional environments in the future
* **-ComputerName** - This is used when running the commands remotely against a cluster.  Just make sure you're using a domain account that has admin privilege on the nodes and cluster
* **-Credential** - This is also used for running the commands remotely against a cluster.

**Register-AzureStackHCI** runs syncronously, with progress reporting, and typically takes 1-2 minutes.  The first time you run it, it may take slightly longer, because it needs to install some dependencies, including additional Azure PowerShell modules.

8. Once dependencies have been installed, you'll receive a popup on **HybridHost001** to authenticate to Azure. Provide your **Azure credentials**.

![Login to Azure](/media/azure_login_reg.png "Login to Azure")

9. Once successfully authenticated, the registration process will begin, and will take a few moments. Once complete, you should see a message indicating success, as per below:

![Register Azure Stack HCI 20H2 with PowerShell](/media/register_azshci_ga.png "Register Azure Stack HCI 20H2 with PowerShell")

**NOTE** - if upon registering, you receive an error similar to that below, please **try a different region**.  You can still proceed to [Step 5](#next-steps) and continue with your evaluation, and it won't affect any functionality.  Just make sure you come back and register later!

```
Register-AzStackHCI : Azure Stack HCI 20H2 is not yet available in region <regionName>
```

10. Once the cluster is registered, run the following command on **HybridHost001** to check the updated status:

```powershell
Invoke-Command -ComputerName AZSHCINODE01 -ScriptBlock {
    Get-AzureStackHCI
}
```
![Check updated registration status with PowerShell](/media/registration_status.png "Check updated registration status with PowerShell")

You can see the **ConnectionStatus** and **LastConnected** time, which is usually within the last day unless the cluster is temporarily disconnected from the Internet. An Azure Stack HCI 20H2 cluster can operate fully offline for up to 30 consecutive days.

#### Option 2 - Register using Windows Admin Center ####

1. On **HybridHost001**, logged in as **hybrid\azureuser**, open the Windows Admin Center, and on the **All connections** page, select your azshciclus
2. When the cluster dashboard has loaded, in the top-right corner, you'll see the **status of the Azure registration/connection**

![Azure registration status in Windows Admin Center](/media/wac_azure_reg_dashboard_2.png "Azure registration status in Windows Admin Center")

3. You can begin the registration process by clicking **Register this cluster**
4. If you haven't already, you'll be prompted to register Windows Admin Center with an Azure tenant. Follow the instructions to **Copy the code** and then click on the link to configure device login.
5. When prompted for credentials, **enter your Azure credentials** for a tenant you'd like to register the Windows Admin Center
6. Back in Windows Admin Center, you'll notice your tenant information has been added. You can now click **Connect** to connect Windows Admin Center to Azure

![Connecting Windows Admin Center to Azure](/media/wac_azure_connect.png "Connecting Windows Admin Center to Azure")

7. Click on **Sign in** and when prompted for credentials, **enter your Azure credentials** and you should see a popup that asks for you to accept the permissions, so click **Accept**

![Permissions for Windows Admin Center](/media/wac_azure_permissions.png "Permissions for Windows Admin Center")

8. Back in Windows Admin Center, you may need to refresh the page if your 'Register this cluster' link is not active. Once active, click **Register this cluster** and you should be presented with a window requesting more information.
9.  Choose your **Azure subscription** that you'd like to use to register, along with an **Azure resource group** and **region**, then click **Register**.  This will take a few moments.

![Final step for registering Azure Stack HCI with Windows Admin Center](/media/wac_azure_register.png "Final step for registering Azure Stack HCI with Windows Admin Center")

10. Once completed, you should see updated status on the Windows Admin Center dashboard, showing that the cluster has been correctly registered.

![Azure registration status in Windows Admin Center](/media/wac_azure_reg_dashboard_3.png "Azure registration status in Windows Admin Center")

You can now proceed on to [Viewing registration details in the Azure portal](#View-registration-details-in-the-Azure-portal)

### View registration details in the Azure portal ###
With registration complete, either through Windows Admin Center, or through PowerShell, you should take some time to explore the artifacts that are created in Azure, once registration successfully completes.

1. On **HybridHost001**, open the Edge browser and **log into https://portal.azure.com** to check the resources created there. In the **search box** at the top of the screen, search for **Resource groups** and then click on **Resource groups**
2. You should see a new **Resource group** listed, with the name you specified earlier, which in our case, is **AZSHCICLUS_RG**

![Registration resource group in Azure](/media/registration_rg_ga.png "Registration resource group in Azure")

12. Click on the **AZSHCICLUS_RG** resource group, and in the central pane, you'll see that a record with the name **azshciclus** has been created inside the resource group
13. Click on the **azihciclus** record, and you'll be taken to the new Azure Stack HCI Resource Provider, which shows information about all of your clusters, including details on the currently selected cluster

![Overview of the recently registered cluster in the Azure portal](/media/azure_portal_hcicluster.png "Overview of the recently registered cluster in the Azure portal")

**NOTE** - If when you ran **Register-AzureStackHCI**, you don't have appropriate permissions in Azure Active Directory, to grant admin consent, you will need to work with your Azure Active Directory administrator to complete registration later. You can exit and leave the registration in status "**pending admin consent**," i.e. partially completed. Once consent has been granted, **simply re-run Register-AzureStackHCI** to complete registration.

### Congratulations! ###
You've now successfully registered your Azure Stack HCI 20H2 cluster!

Next Steps
-----------
In this step, you've successfully registered your Azure Stack HCI 20H2 cluster. With this complete, you can now [Deploy your AKS-HCI infrastructure](/steps/4_DeployAKSHCI.md "Deploy your AKS-HCI infrastructure")

Product improvements
-----------
If, while you work through this guide, you have an idea to make the product better, whether it's something in Azure Stack HCI, AKS on Azure Stack HCI, Windows Admin Center, or the Azure Arc integration and experience, let us know! We want to hear from you!

For **Azure Stack HCI**, [Head on over to the Azure Stack HCI 20H2 Q&A forum](https://docs.microsoft.com/en-us/answers/topics/azure-stack-hci.html "Azure Stack HCI 20H2 Q&A"), where you can share your thoughts and ideas about making the technologies better and raise an issue if you're having trouble with the technology.

For **AKS on Azure Stack HCI**, [Head on over to our AKS on Azure Stack HCI 20H2 GitHub page](https://github.com/Azure/aks-hci/issues "AKS on Azure Stack HCI GitHub"), where you can share your thoughts and ideas about making the technologies better. If however, you have an issue that you'd like some help with, read on... 

Raising issues
-----------
If you notice something is wrong with this guide, such as a step isn't working, or something just doesn't make sense - help us to make this guide better!  Raise an issue in GitHub, and we'll be sure to fix this as quickly as possible!

If you're having an issue with Azure Stack HCI 20H2 **outside** of this guide, [head on over to the Azure Stack HCI 20H2 Q&A forum](https://docs.microsoft.com/en-us/answers/topics/azure-stack-hci.html "Azure Stack HCI 20H2 Q&A"), where Microsoft experts and valuable members of the community will do their best to help you.

If you're having a problem with AKS on Azure Stack HCI **outside** of this guide, make sure you post to [our GitHub Issues page](https://github.com/Azure/aks-hci/issues "GitHub Issues"), where Microsoft experts and valuable members of the community will do their best to help you.