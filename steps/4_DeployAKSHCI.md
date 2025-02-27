Deploy your AKS-HCI infrastructure
==============
Overview
-----------
It's now time to deploy AKS on Azure Stack HCI. You'll first deploy the AKS on Azure Stack HCI management cluster, and then deploy a target cluster, onto which you can test deployment of a workload.

Contents
-----------
- [Overview](#overview)
- [Contents](#contents)
- [Architecture](#architecture)
- [Allow popups in Edge browser](#allow-popups-in-edge-browser)
- [Configure Windows Admin Center](#configure-windows-admin-center)
- [Deploying the AKS on Azure Stack HCI management cluster](#deploying-the-aks-on-azure-stack-hci-management-cluster)
- [Enable Azure Arc integration (Optional)](#enable-azure-arc-integration-optional)
- [Create a Kubernetes cluster (Target cluster)](#create-a-kubernetes-cluster-target-cluster)
- [Scale your Kubernetes cluster (Target cluster)](#scale-your-kubernetes-cluster-target-cluster)
- [Next Steps](#next-steps)
- [Product improvements](#product-improvements)
- [Raising issues](#raising-issues)

Architecture
-----------

From an architecture perspective, as shown earlier, this graphic showcases the different layers and interconnections between the different components:

![Architecture diagram for AKS on Azure Stack HCI in Azure](/media/nested_virt_akshci_ga.png "Architecture diagram for AKS on Azure Stack HCI in Azure")

In this section, you'll use Windows Admin Center to deploy the AKS on Azure Stack HCI, including all the key components. First, the management cluster (kubernetes virtual appliance) provides the the core orchestration mechanism and interface for deploying and managing one or more target clusters. These target, or workload clusters contain worker nodes and are where application workloads run. If you're interested in learning more about the building blocks of the Kubernetes infrastructure, you can [read more here](https://docs.microsoft.com/en-us/azure-stack/aks-hci/kubernetes-concepts "Kubernetes core concepts for Azure Kubernetes Service on Azure Stack HCI").

As mentioned earlier, Azure Stack HCI and AKS-HCI will de deployed as 2 separate environments within the same Azure VM. In a production environment, you would run AKS-HCI **on top of** Azure Stack HCI, but in this nested environment, the performance of the multiple levels of nesting can have a negative impact, so in this case, they will be deployed side by side for evaluation.

Allow popups in Edge browser
-----------
To give the optimal experience with Windows Admin Center, you should enable **Microsoft Edge** to allow popups for Windows Admin Center.

1. Still inside your **HybridHost001 VM**, double-click the **Microsoft Edge icon** on your desktop
2. Navigate to **edge://settings/content/popups**
3. In the **Allow** box, click on **Add**
4. In the **Add a site** box, enter **https://HybridHost001** (assuming you didn't change the host name at deployment time)

![Allow popups in Edge](/media/allow_popup_edge.png "Allow popups in Edge")

5. Close the **settings tab**.

Configure Windows Admin Center
-----------
The environment has already deployed Windows Admin Center, and updated all the extensions that are pre-installed, however there are some additional configuration steps that must be performed before you can use it to deploy AKS on Azure Stack HCI.

1. **Double-click the Windows Admin Center** shortcut on the desktop.
2. Once open, minimize Windows Admin Center
3. Open **File Explorer** and navigate to **C:\\**. Right-click and select **New**, **Folder** and enter **"Feed"** as the new name for the folder.
4. Navigate to the **V:\Source** folder**
5. **Right-click** the **AksHciPreview** ZIP file, select **Extract All**, then click **Extract**
6. In the newly **extracted AksHciPreview folder**, select the **NUPKG file**, right-click and select **copy**, then navigate to **C:\Feed** and paste into this folder.
8. With the NUPKG file in your C:\Feed folder, maximize Windows Admin Center.
9. Navigate to **Settings**, then **Extensions**, and finally, click on **Feeds**
10. On the Feeds page, click **Add**. In the **"Add package source"** blade, enter **"C:\Feed"** (without the quotes) and click **Add** in the bottom-right corner.

![Adding an extension feed in Windows Admin Center](/media/add_feed.png "Adding an extension feed in Windows Admin Center")

11. Click on **Available extensions** and you should now see **Azure Kubernetes Service** listed as available

![Available extensions in Windows Admin Center](/media/available_extensions.png "Available extensions in Windows Admin Center")

12. To install the extension, simply click on it, and click **Install**. Within a few moments, this will be completed. You can double-check by navigating to **Installed extensions**, where you should see Azure Kubernetes Service listed as **Installed**

With your extensions correctly deployed, in order to deploy AKS-HCI with Windows Admin Center, you need to connect your Windows Admin Center instance to Azure, which it should have been from when you registered your Azure Stack HCI 20H2 cluster.  If it hasn't been registered, follow the steps:

13. Click on **Settings** then under **Gateway** click on **Azure**.
14. Click **Register**, and in the **Get started with Azure in Windows Admin Center** blade, follow the instructions to **Copy the code** and then click on the link to configure device login.
15. When prompted for credentials, **enter your Azure credentials** for a tenant you'd like to use to register the Windows Admin Center
16. Back in Windows Admin Center, you'll notice your tenant information has been added.  You can now click **Connect** to connect Windows Admin Center to Azure

![Connecting Windows Admin Center to Azure](/media/wac_azure_connect.png "Connecting Windows Admin Center to Azure")

17. Click on **Sign in** and when prompted for credentials, **enter your Azure credentials** and you should see a popup that asks for you to accept the permissions, so click **Accept**

![Permissions for Windows Admin Center](/media/wac_azure_permissions.png "Permissions for Windows Admin Center")

18. Click on **Windows Admin Center** in the top-left corner to return to the home page

You'll notice that your HybridHost001 is already under management, so at this stage, you're ready to proceed to deploy the AKS on Azure Stack HCI management cluster onto your Windows Server 2019 Hyper-V host.

![HybridHost001 under management in Windows Admin Center](/media/akshcihost_in_wac.png "HybridHost001 under management in Windows Admin Center")

Deploying the AKS on Azure Stack HCI management cluster
-----------
The next section will walk through configuring the AKS on Azure Stack HCI management cluster, on your single node Windows Server 2019 host.

1. From the Windows Admin Center homepage, click on your **HybridHost001.hybrid.local** cluster.
2. You'll be presented with a rich array of information about your HybridHost001 cluster, of which you can feel free to explore the different options and metrics. When you're ready, on the left-hand side, scroll down and under **Extensions**, click **Azure Kubernetes Service**

![Ready to deploy AKS-HCI with Windows Admin Center](/media/aks_extension.png "Ready to deploy AKS-HCI with Windows Admin Center")

3. Click on **Set up** to start the deployment process
4. Firstly, review the prerequisites - your environment will meet all the prerequisites, so you should be fine to click **Next: System checks**
5. On the **System checks** page, enter the password for your **azureuser** account
6. Once your credentials have been validated, Windows Admin Center will begin to validate it's own configuration, and the configuration of your target nodes, which in this case, is the Windows Server 2019 Hyper-V host (HybridHost001, running in your Azure VM)

![System checks performed by Windows Admin Center](/media/wac_system_checks_single.png "System checks performed by Windows Admin Center")

You'll notice that Windows Admin Center will validate memory, storage, networking, roles and features and more. If you've followed the guide correctly, you'll find you'll pass all the checks and can proceed.

7. Once validated, click **Apply**, wait a few moments, then click **Next: Connectivity**
8. On the **Connectivity** page, read the information about **CredSSP**, then click **Enable**. Once enabled, click **Next: Host configuration**

![Enable CredSSP in Windows Admin Center](/media/aks_hostconfig_credssp.png "Enable CredSSP in Windows Admin Center")

**NOTE** - if nyou receive a WinRM error, open an **Administrative PowerShell console** and run the following command and then retry:

```powershell
Restart-Service WinRm -Force
```

9.  On the **Host configuration** page, under **Host details**, select your **V:**, and leave the other settings as default

![Host configuration in Windows Admin Center](/media/aks_hostconfig_hostdetails_single.png "Host configuration in Windows Admin Center")

10. Under **VM Networking**, ensure that **InternalNAT** is selected for the **Internet-connected virtual switch**
11. For **Enable virtual LAN identification**, leave this selected as **No**
12. For **Cloudagent IP** this is optional, so we will leave this blank
13. For **IP address allocation method** choose **DHCP**

![Host configuration in Windows Admin Center](/media/aks_hostconfig_vmnet_int.png "Host configuration in Windows Admin Center")

14.  Under **Load balancer settings**, enter the range from **192.168.0.150** to **192.168.0.250** and then click **Next:Azure registration**

![Host configuration in Windows Admin Center](/media/aks_hostconfig_lb.png "Host configuration in Windows Admin Center")

16. On the **Azure registration page**, your Azure account should be automatically populated. Use the drop-down to select your preferred subscription. **Note**, nothing will be deployed into this subscription and no charges will be incurred. If you are prompted, log into Azure with your Azure credentials. Once successfully authenticated, you should see your **Account**, then **choose your subscription**

![AKS Azure Registration in Windows Admin Center](/media/aks_azure_reg.png "AKS Azure Registration in Windows Admin Center")

15. Once you've chosen your subscription, click **Next: Review**
16. Review your choices and settings, then click **Apply**. After a few moments, you should receive a notification:

![Setting the AKS-HCI config in Windows Admin Center](/media/aks_host_mgmtconfirm.png "Setting the AKS-HCI config in Windows Admin Center")

17. Once confirmed, you can click **Next: New cluster** to start the deployment process of the management cluster.

![AKS on Azure Stack HCI management cluster deployment started in Windows Admin Center](/media/aks_deploy_started.png "AKS on Azure Stack HCI management cluster deployment started in Windows Admin Center")

**NOTE 1** - Do not close the Windows Admin Center browser at this time. Leave it open and wait for successful completion.

**NOTE 2** - You may receive a WinRM error message stating "Downloading virtual machine images and binaries for the AKS host failed" - this can be ignored, so **do not close/refresh the browser**.

18.  Upon completion you should receive a notification of success. In this case, you can see deployment of the AKS on Azure Stack HCI management cluster took just over 7 minutes.

![AKS-HCI management cluster deployment completed in Windows Admin Center](/media/aks_deploy_success.png "AKS-HCI management cluster deployment completed in Windows Admin Center")

19. Once reviewed, click **Finish**. You will then be presented with a management dashboard where you can create and manage your Kubernetes clusters.

### Updates and Cleanup ###
To learn more about **updating**, **redeploying** or **uninstalling** AKS on Azure Stack HCI with Windows Admin Center, you can [read the official documentation here.](https://docs.microsoft.com/en-us/azure-stack/aks-hci/setup "Official documentation on updating, redeploying and uninstalling AKS on Azure Stack HCI")

Enable Azure Arc integration (Optional)
-----------
In the next section, you'll be deploying target clusters to run workloads. During that process, you'll have the option to connect those clusters to [Azure Arc](https://docs.microsoft.com/en-us/azure/azure-arc/kubernetes/overview "Azure Arc enabled Kubernetes"). Now, in order to do that successfully, you need to grant some additional permissions on the Windows Admin Center Azure AD application that was created when you connected Windows Admin Center to Azure, earlier.

1. Still in Windows Admin Center, click on the **Settings** gear in the top-right corner
2. Under **Gateway**, click **Azure**. You should see your previously registered Azure AD app:

![Your Azure AD app in Windows Admin Center](/media/wac_azureadapp.png "Your Azure AD app in Windows Admin Center")

3. Click on **View in Azure** to be taken to the Azure AD app portal, where you should see information about this app, including permissions required. If you're prompted to log in, provide appropriate credentials.
4. Once logged in, under **Configured permissions**, you may see the **Microsoft.Graph (5)** listed with the status **Not granted for...**

![Your Azure AD app permissions in Windows Admin Center](/media/wac_azuread_grant.png "Your Azure AD app permissions in Windows Admin Center")

**NOTE** If you don't see Microsoft Graph listed in the API permissions, you can either [re-register Windows Admin Center at Step 13 here](#configure-windows-admin-center "re-register Windows Admin Center at step 13 here") for the permissions to appear correctly, or manually add the **Microsoft Graph Appliation.ReadWrite.All** permission.  To manually add the permission:

- Click **+ Add a permission**
- Select **Microsoft Graph**, then **Delegated permissions**
- Search for **Application.ReadWrite.All**, then if required, expand the **Application** dropdown
- Select the **checkbox** and click **Add permissions**, then follow the next step

5. Click on **Grant admin consent for __________** and when prompted to confirm permissions, click **Yes**

![Confirm Azure AD app permissions in Windows Admin Center](/media/wac_azuread_confirm.png "Confirm Azure AD app permissions in Windows Admin Center")

With that step completed, you're now ready to create your first target cluster.

Create a Kubernetes cluster (Target cluster)
-----------
With the management cluster deployed successfully, you're ready to move on to deploying Kubernetes clusters that can host your workloads. We'll then briefly walk through how to scale your Kubernetes cluster and upgrade the Kubernetes version of your cluster.

There are two ways to create a Kubernetes cluster in Windows Admin Center.

#### Option 1 ####
1. From your Windows Admin Center landing page (https://HybridHost001), click on **+Add**.
2. In the **Add or create resources blade**, in the **Kubernetes clusters (preview) tile**, click **Create new**

![Create Kubernetes cluster in Windows Admin Center](/media/create_cluster_method1.png "Create Kubernetes cluster in Windows Admin Center")

#### Option 2 ####
1. From your Windows Admin Center landing page (https://HybridHost001), click on your **HybridHost001.hybrid.local \[Gateway\]** machine.
2. Then, on the left-hand side, scroll down and under **Extensions**, click **Azure Kubernetes Service**.
3. In the central pane, click on **Add cluster**

![Create Kubernetes cluster in Windows Admin Center](/media/create_cluster_method2.png "Create Kubernetes cluster in Windows Admin Center")

Whichever option you chose, you will now be at the start of the **Create kubernetes cluster** wizard.

1. Firstly, review the prerequisites - your Azure VM environment will meet all the prerequisites, so you should be fine to click **Next: Basics**
2. On the **Basics** page, firstly, choose whether you wish to **optionally** integrate with Azure Arc for Kubernetes. You can click the link on the page to learn more about Azure Arc. If you do wish to integrate, select the **Enabled** radio button, then use the drop downs to select the **subscription** and **resource group**. Alternatively, you can create a new resource group, in a specific region, exclusively for the Azure Arc integration resource.

![Enable Arc integration with Windows Admin Center](/media/aks_basics_arc.png "Enable Arc integration with Windows Admin Center")

3. Still on the **Basics** page, under **Cluster details**, provide a **Kubernetes cluster name**, **Azure Kubernetes Service host**, which should be **HybridHost001.hybrid.local**, enter your host credentials, then select the **Kubernetes version** from the drop down.

![AKS cluster details in Windows Admin Center](/media/aks_basics_cluster_details_single.png "AKS cluster details in Windows Admin Center")

4. Under **Primary node pool**, accept the defaults, and then click **Next: Node pools**

![AKS primary node pool in Windows Admin Center](/media/aks_basics_primarynp.png "AKS primary node pool in Windows Admin Center")

5. On the **Node pools** page, click on **+Add node pool**
6. In the **Add a node pool** blade, enter the following, then click **Add**
   1. **Node pool name**: LinuxPool1
   2. **OS type**: Linux
   3. **Node size**: Standard_K8S3_v1 (6 GB Memory, 4 CPU)
   4. **Node count**: 1

![AKS node pools in Windows Admin Center](/media/aks_node_pools.png "AKS node pools in Windows Admin Center")

1. Once your **Node pools** have been defined, click **Next: Authentication**
2. For this evaluation, for **AD Authentication** click **Disabled** and then click **Next: Networking**
3.  On the **Networking** page, review the **defaults** and click **Next: Integration**
4.  On the **Integration** page, review the **Persistent storage defaults**, then click **Next: Review + Create**
5.  On the **Review + Create** page, review your chosen settings, then click **Create**

![Finalize creation of AKS cluster in Windows Admin Center](/media/aks_create.png "Finalize creation of AKS cluster in Windows Admin Center")

13. The creation process will begin and take a few minutes

![Start deployment of AKS cluster in Windows Admin Center](/media/aks_create_start.png "Start deployment of AKS cluster in Windows Admin Center")

14. Once completed, you should see a message for successful creation, then click **Finish**

![Completed deployment of AKS cluster in Windows Admin Center](/media/aks_create_complete.png "Completed deployment of AKS cluster in Windows Admin Center")

15. Back in the **Azure Kubernetes Service on Azure Stack HCI landing page**, you should now see your cluster listed

![AKS cluster in Windows Admin Center](/media/aks_dashboard.png "AKS cluster in Windows Admin Center")

16. On the dashboard, if you chose to integrate with Azure Arc, you should be able to click the **Azure instance** link to be taken to the Azure Arc view in the Azure portal.

![AKS cluster in Azure Arc](/media/aks_in_arc.png "AKS cluster in Azure Arc")

17. In addition, you may wish to download your **Kubernetes cluster kubeconfig** file in order to access this Kubernetes cluster via **kubectl** later.
18. Once you have your Kubeconfig file, you can click **Finish**


Scale your Kubernetes cluster (Target cluster)
-----------
Next, you'll scale your Kubernetes cluster to add an additional Linux worker node. As it stands, this has to be performed with **PowerShell** but will be available in Windows Admin Center in the future.

1. Open **PowerShell as Administrator** and run the following command to import the new modules, and list their functions.

```powershell
Import-Module AksHci
Get-Command -Module AksHci
```

2. Next, to check on the status of the existing clusters, run the following

```powershell
Get-AksHciCluster
```

![Output of Get-AksHciCluster](/media/get_akshcicluster_wac.png "Output of Get-AksHciCluster")

3. Next, you'll scale your Kubernetes cluster to have **2 Linux worker nodes**:

```powershell
Set-AksHciClusterNodeCount –Name akshciclus001 -linuxNodeCount 2 -windowsNodeCount 1
```

**NOTE** - You can also scale your Control Plane nodes for this particular cluster, however it has to be **scaled independently from the worker nodes** themselves. You can scale the Control Plane nodes using the command. Before you run this command however, check that you have an extra 16GB memory left of your HybridHost001 OS - if your host has been deployed with 64GB RAM, you may not have enough capacity for an additonal 2 Control Plane VMs.

```powershell
Set-AksHciClusterNodeCount –Name akshciclus001 -controlPlaneNodeCount 3
```

**NOTE** - the control plane node count should be an **odd** number, such as 1, 3, 5 etc.

4. Once these steps have been completed, you can verify the details by running the following command:

```powershell
Get-AksHciCluster
```

![Output of Get-AksHciCluster](/media/get_akshcicluster_wac2.png "Output of Get-AksHciCluster")

To access this **akshciclus001** cluster using **kubectl** (which was installed on your host as part of the overall installation process), you'll first need the **kubeconfig file**.

5. To retrieve the kubeconfig file for the akshciclus001 cluster, you'll need to run the following command from your **administrative PowerShell**:

```powershell
Get-AksHciCredential -Name akshciclus001
dir $env:USERPROFILE\.kube
```

Next Steps
-----------
In this step, you've successfully deployed the AKS on Azure Stack HCI management cluster using Windows Admin Center, optionally integrated with Azure Arc, and subsequently, deployed and scaled a Kubernetes cluster that you can move forward with to the next stage, in which you can deploy your applications.

* [**Part 5** - Explore AKS on Azure Stack HCI](/steps/5_ExploreAKSHCI.md "Explore AKS on Azure Stack HCI")

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