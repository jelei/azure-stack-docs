---
title: Azure verification for VMs on Azure Stack HCI
description: Learn about the Azure Benefits feature on Azure Stack HCI.
author: sethmanheim
ms.author: sethm
ms.topic: overview
ms.custom:
  - devx-track-azurepowershell
ms.reviewer: jlei
ms.date: 11/14/2023
ms.lastreviewed: 11/14/2023
---

# Azure verification for VMs on Azure Stack HCI

Microsoft Azure offers a range of differentiated workloads and capabilities that are designed to run only on Azure. Azure Stack HCI extends many of the same benefits you get from Azure, while running on the same familiar and high-performance on-premises or edge environments.

*Azure verification for VMs* makes it possible for supported Azure-exclusive workloads to work outside of the cloud. This feature, modeled after the [IMDS attestation](/azure/virtual-machines/windows/instance-metadata-service?tabs=windows#attested-data) service in Azure, is a built-in platform attestation service that is enabled by default on Azure Stack HCI 23H2 or later. It helps to provide guarantees for these VMs to operate in other Azure environments. 

//add link to old doc

## Benefits available on Azure Stack HCI

Azure verification for VM enables you to use these benefits available only on Azure Stack HCI:

| Workload                                 | What it is                           | How to get benefits                                                                                                                                                                                                                                                                       |
|------------------------------------------|----------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Extended Security Update (ESUs)          | Get security updates at *no additional cost* for end-of-support SQL and Windows Server VMs on Azure Stack HCI. <br/> For more information, see [Free Extended Security Updates (ESU) on Azure Stack HCI](azure-benefits-esu.md). | [Legacy OS support](#legacy-os-support) needs to be turned on for older VMs running version Windows Server 2012/ Windows 7 or earlier.|
| Azure Virtual Desktop (AVD)                    | AVD session hosts can run only on Azure infrastructure. Activate your Windows multi-session VMs on Azure Stack HCI using Azure VM verification. <br/> Note: licensing requirements for AVD still apply. See [Azure Virtual Desktop pricing](/azure/virtual-desktop/azure-stack-hci-overview#pricing) | Activated automatically for VMs running version Windows 11 multi-session with 11B update (//kb version) or later |
| Windows Server Datacenter: Azure Edition | Azure Edition VMs can run only on Azure infrastructure. Activate your [Windows Server Azure Edition](/azure/automanage/automanage-windows-server-services-overview) VMs and use the latest Windows Server innovations and other exclusive features. <br/> Note: licensing requirements still apply. See ways to [license Windows Server VMs on Azure Stack HCI](/vm-activate?tabs=azure-portal)         | Activated automatically for VMs running version Windows Server: Azure Edition 2022 with 11B update (//kb version) or later |
| Azure Policy guest configuration         | Get [Azure Policy guest configuration](/azure/governance/policy/concepts/guest-configuration) at no cost. This Arc extension enables the auditing and configuration of OS settings as code for servers and VMs. | Arc agent version 1.13 or later


## Architecture

This section is optional reading, and explains more about how Azure VM verification on HCI works "under the hood."

Azure VM verification relies on a built-in platform attestation service on Azure Stack HCI. This service is modeled after the same [IMDS Attestation](/azure/virtual-machines/windows/instance-metadata-service?tabs=windows#attested-data) service that runs in Azure, and returns an almost identical payload. The main difference is that it runs on-premises, and therefore guarantees that VMs are running on Azure Stack HCI instead of Azure.

//*image needed*

:::image type="content" source="media/azure-benefits/cluster.png" alt-text="Architecture":::

1. Azure VM verification is turned on by default with Azure Stack HCI running version 23H2 or later. During server startup, HciSvc generates an Integration Service over Hyper-V sockets ([//i.e. VMBus](/virtualization/hyper-v-on-windows/reference/hyper-v-architecture)) to facilitate secure communication between VMs and servers.

  - <b> [Legacy OS support](#legacy-os-support): </b> Workloads that cannot make Win32 API calls or directly query an Integration Service will need to enable Legacy OS support . This will provide a private and non-routable REST endpoint to VMs on the same server. 
  </br> To enable this endpoint, an internal vSwitch is configured on the Azure Stack HCI server (named *AZSHCI_HOST-IMDS_DO_NOT_MODIFY*). After that, VMs must have a NIC configured (*AZSHCI_GUEST-IMDS_DO_NOT_MODIFY*) and attached to the same vSwitch.

    > [!NOTE]
    > Modifying or deleting this switch and NIC prevents Azure VM verification from working properly. If you encounter errors, try turning off and then turning back on legacy OS support.

2. Whenever Azure Stack HCI syncs with Azure, HciSvc obtains a certificate from Azure, and securely stores it within an enclave on each server.

   > [!NOTE]
   > Certificates are renewed every time the Azure Stack HCI cluster syncs with Azure, and each renewal is valid for 30 days. As long as you maintain the usual 30 day connectivity requirements for Azure Stack HCI, no user action is required to renew certificates.

3. To activate benefits, consumer workloads (e.g. Windows Server Azure Edition VMs) request attestation from servers. They try to send requests via VMBus, or they can also query the REST endpoint using the networking components configured in <b>legacy OS support</b>. Both approaches for VM-server communication are supported and can coexist on the same cluster.

   > [!NOTE]
   > When using legacy OS support, you may need to manually enable access for VMs that require the activation of benefits.

    HciSvc then returns a signed response using the Azure certificate it has stored. Consumers verify the response and activate associated benefits.


## Tutorial: Managing Azure VM verification

Azure VM verification is automatically turned on by default in Azure Stack HCI 23H2 or later, and for most workloads, no additional setup is required. The following instructions outline the prerequisites for using this feature and steps for managing benefits (optional).

> [!NOTE]
> The only workload that requires additional setup is Extended Security Updates (ESUs). To enable ESUs, you must turn on [legacy OS support](#legacy-os-support).

### Prerequisites

- [Update your Azure Stack HCI cluster](../manage/update-cluster.md): Minimum version 23H2 or later
- [Register Azure Stack HCI](../deploy/register-with-azure.md?tab=windows-admin-center#register-a-cluster): All servers must be online and registered to Azure
- [Install Hyper-V and RSAT-Hyper-V-Tools](/windows-server/virtualization/hyper-v/get-started/install-the-hyper-v-role-on-windows-server)
- Update your VMs: See [version requirements for workloads](#benefits-available-on-azure-stack-hci)
- Turn on Hyper-V Guest Service Interface: See instructions [b//elow](#1-turn-on-legacy-os-support-on-the-host-1)
- (optional) If you are using Windows Admin Center:  You will need to run at minimum version //) with Cluster Manager extension (version /////) or later.

You can manage Azure VM verification using Windows Admin Center or PowerShell, or view its status using Azure CLI or Azure portal. The following sections describe each option.

## [Windows Admin Center](#tab/wac)
### Check server status of Azure VM verification 
1. In Windows Admin Center, select **Cluster Manager** from the top drop-down menu, navigate to the cluster that you want to activate, then under **Settings**, select **Azure verification for VMs**.

2. To check Azure VM verification server status:
   - Cluster-level status: **Host status** appears as **On**.
   - Server-level satus: Under the **Server** tab in the dashboard, check that the status for every server shows as **Active** in the table.

//*image needed*

   #### Troubleshooting servers
- Under the **Server** tab, if one or more servers appear as **Expired**:
  - If the server has not synced with Azure for more than 30 days, its status will appears as **Expired** or **Inactive**. Click on **Sync with Azure** to schedule a manual sync.

### Manage benefits activated on VMs
1. To check what benefits are activated on VMs, navigate to the **VMs** tab. 

2. In the dashbord, you will see the number of VMs with:
   - **Active benefits**: These VMs have Azure-exclusive features activated via Azure VM verification. 
   - **Inactive benefits**: These VMs have Azure-exclusive features that need further action before activation.
   - **Unknown**: We can’t determine the eligible benefits for these VMs because Hyper-V data exchange is turned off. See troubleshooting below.
   - **No applicable benefits**: These VMs do not have Azure-exclusive features and hence do not require Azure VM verification.

3. In the table, you can see the **Eligible benefit** that is applicable for each VM. See the [full list of benefits available on Azure Stack HCI](#benefits-available-on-azure-stack-hci). 

//*image needed*

   #### Troubleshooting VMs
- Under the **VMs** tab, if one or more VMs appear as **Inactive benefits**:
  - If the action suggested is to **Install updates**, you may not have the minimum OS version required for the benefit. Update the VM to meet the [version requirements for workloads](#benefits-available-on-azure-stack-hci).
   - If the action suggested is to **Turn on Guest Service Interface**, select this and open the context pane to turn on [Hyper-V Guest Service Interface](/virtualization/hyper-v-on-windows/reference/integration-services#hyper-v-powershell-direct-service). This feature is required for VMs to communicate to the host via VMbus. 
   - If the action suggested is regarding **legacy OS support**, see [troubleshooting for legacy OS support](#legacy-os-support).

- Under the **VMs** tab, if one or more VMs appear as **Unknown**:
   - If you want to determine the benefits available for these VMs, you can either do so manually by checking the [full list of benefits available on Azure Stack HCI](#benefits-available-on-azure-stack-hci), or Windows Admin Center can display this information for you. To access this information through Windows Admin Center, enable [Hyper-V data exchange (KVP)](/virtualization/hyper-v-on-windows/reference/integration-services#hyper-v-data-exchange-service-kvp) for your VMs by selecting the action labeled **Turn on Hyper-V data exchange**.


## [PowerShell](#tab/azure-ps)
### Check server status of Azure VM verification 

- When Azure VM verification setup is successful, you can view the host status. Check the cluster property **IMDS Attestation** by running the following command:

   ```powershell
   Get-AzureStackHCI
   ```

- Or, to view  Azure VM verification status for servers, run the following command:

   ```powershell
   Get-AzureStackHCIAttestation [[-ComputerName] <string>]
   ```
#### Troubleshooting servers
- If Azure VM verification for one or more servers is not yet synced and renewed with Azure, it may appear as **Expired** or **Inactive**. Schedule a manual sync:

  ```powershell
  Sync-AzureStackHCI
  ```

### Manage benefits activated on VMs

- To check access to Azure VM verification for VMs, run the following command: 

   ```powershell
   Get-AzStackHCIVMAttestation
   ```
   Note: For a VM that is supported for **v2**, it can communicate with the server using VMBus. Conversely, a **v1** supported VM is configured with legacy OS support and can access Azure VM verification via REST. If a VM supports both **v1** and **v2**, the **v2** method (i.e. VMBus) will be primarily used, but it can fall back to **v1** if **v2** encounters an issue.

#### Troubleshooting VMs


- To set up access to Azure VM verification for VMs, you need to turn on [Hyper-V Guest Service Interface](/virtualization/hyper-v-on-windows/reference/integration-services#hyper-v-powershell-direct-service).
To check that Hyper-V Guest Service Interface is enabled, run:
   ```powershell
   Get-VMIntegrationService [[-VMName] <VMName>] -Name "Guest Service Interface"
   ```

- To turn on Hyper-V Guest Service Interface
   ```powershell
   Enable-VMIntegrationService [[-VMName] <VMName>] -Name "Guest Service Interface"
   ```

- To check that the VMs can access Azure VM verification on the host, run the following command on the VM:

  ```powershell
  Invoke-RestMethod -Headers @{"Metadata"="true"} -Method GET -Uri "http://169.254.169.253:80/metadata/attested/document?api-version=2018-10-01"
  ```





## [Azure portal](#tab/azureportal)
1. In your Azure Stack HCI cluster resource page, navigate to the **Configuration** tab.
2. Under the feature **Azure verification for VMs**, view the host attestation status:

//*image needed*


## [Azure CLI](#tab/azurecli)

Azure CLI is available to install in Windows, macOS and Linux environments. It can also be run in [Azure Cloud Shell](https://shell.azure.com/). This document details how to use Bash in Azure Cloud Shell. For more information, refer [Quickstart for Azure Cloud Shell](/azure/cloud-shell/quickstart).

Launch [Azure Cloud Shell](https://shell.azure.com/) and use Azure CLI to configure Azure VM verification following these steps:

1. Set up parameters from your subscription, resource group, and cluster name
    ```azurecli
    subscription="00000000-0000-0000-0000-000000000000" # Replace with your subscription ID
    resourceGroup="hcicluster-rg" # Replace with your resource group name
    clusterName="HCICluster" # Replace with your cluster name

    az account set --subscription "${subscription}"
    ```

1. ///check the query/To view Azure VM verification status on a cluster, run the following command:
    ```azurecli    
    az stack-hci cluster list \
    --resource-group "${resourceGroup}" \
    --query "[?name=='${clusterName}'].{Name:name, AzureBenefitsHostAttestation:reportedProperties.imdsAttestation}" \
    -o table
    ```

## Legacy OS support
For older VMs that lack the necessary Hyper-V functionality ([Guest Service Interface](/virtualization/hyper-v-on-windows/reference/integration-services#hyper-v-powershell-direct-service)) to communicate directly with the host, you will need to configure traditional networking components for Azure VM verification. If you have such workloads, like Extended Security Updates (ESUs), follow the instructions below to set up legacy OS support.


## [Windows Admin Center](#tab/wac)
### 1. Turn on legacy OS support on the host
1. In Windows Admin Center, select **Cluster Manager** from the top drop-down menu, navigate to the cluster that you want to activate, then under **Settings**, select **Azure verifications for VMs**.

2. In the section for **Legacy OS support**, select **Change status**. Select **On** in the context pane. Note: this also enables networking access for all existing VMs. You will have have to manually turn on legacy OS support for any new VMs that you create later.
  
3. Select **Change status** to confirm. It may take a few minutes for servers to reflect the changes.

4. When legacy OS support is successfully turned on:
    - Check that **Legacy OS support** appears as **On**.
    - Under the **Server** tab in the dashboard, check that legacy OS support for every server shows as **On** in the table.

//*image needed* 

### 2. Enable access for new VMs

You will have to turn on legacy OS networking for any new VMs that you create after the first setup. To manage access for VMs:

1. Navigate to the "VMs" tab. Any VM that requires legacy OS support access will show up as <b> Inactive </b>. Select action to <b> Set up legacy OS networking </b> for the selected VM or for all existing VMs on the cluster.

> [!NOTE]
> To successfully enable legacy OS support on Generation 1 VMs, the VM must first be powered off to enable the NIC to be added.

//*image needed* 

## [PowerShell](#tab/azure-ps)
### 1. Turn on legacy OS support on the host
- Run the following command from an elevated PowerShell window on your Azure Stack HCI cluster:

   ```powershell
   Enable-AzStackHCIAttestation
   ```

- Or, if you want to add all existing VMs on setup, you can run the following command:

   ```powershell
   Enable-AzStackHCIAttestation -AddVM
   ```
- Check that legacy OS support is turned on: 
   ```powershell
   Get-AzureStackHCIAttestation [[-ComputerName] <string>]
   ```

#### Troubleshooting servers

- To turn off and reset legacy OS support on your cluster, run the following command:

  ```powershell
  Disable-AzStackHCIAttestation -RemoveVM
  ```
   
### 2. Enable access for VMs
- To configure networking access for selected VMs, run the following command on your Azure Stack HCI cluster:

   ```powershell
   Add-AzStackHCIVMAttestation [-VMName]
   ```

- Or, to add all existing VMs, run the following command:

   ```powershell
   Add-AzStackHCIVMAttestation -AddAll
   ```

- Check VMs that have access to legacy OS support /// screenshot

   ```powershell
   Get-AzStackHCIVMAttestation
   ```
> [!NOTE]
> To successfully enable legacy OS support on Generation 1 VMs, the VM must first be powered off to enable the NIC to be added.

#### Troubleshooting VMs

- To remove access to legacy OS support for selected VMs:

  ```powershell
  Remove-AzStackHCIVMAttestation -VMName <string>
  ```

  Or, to remove access for all existing VMs: 

  ```powershell
  Remove-AzStackHCIVMAttestation -RemoveAll
  ```


## FAQ

This FAQ provides answers to some questions about using Azure Benefits.

### What Azure-exclusive workloads can I enable with Azure VM verification?

See the [full list here](#benefits-available-on-azure-stack-hci).

### Does it cost anything to turn on Azure VM verification?

No, turning on Azure VM verification incurs no extra fees.

### Can I use Azure VM verification on environments other than Azure Stack HCI?

No, Azure VM verification is a feature built into the Azure Stack HCI OS, and can only be used on Azure Stack HCI.

### I have just set up Azure VM verification on my cluster. How do I ensure that Azure VM verification stays active?

- In most cases, there is no user action required. Azure Stack HCI automatically renews Azure VM verification when it syncs with Azure.
- However, if the cluster disconnects for more than 30 days and Azure VM verification shows as **Expired**, you can manually sync using PowerShell and Windows Admin Center. For more information, see [syncing Azure Stack HCI](../faq.yml#what-happens-if-the-30-day-limit-is-exceeded).

### What happens when I deploy new VMs, or delete VMs?

- When you deploy new VMs that require Azure VM verification, they will automatically be activated provided that they have the right [prerequisites](#prerequisites). 

- However, for legacy VMs using legacy OS support, you can manually add new VMs to access Azure VM verification using Windows Admin Center or PowerShell, using the preceding [instructions](#legacy-os-support).
- You can still delete and migrate VMs as usual. The NIC **AZSHCI_GUEST-IMDS_DO_NOT_MODIFY** will still exist on the VM after migration. To clean up the NIC before migration, you can remove VMs from Azure VM verification using Windows Admin Center or PowerShell using the preceding instructions for legacy OS support, or you can migrate first and manually delete NICs afterwards.

### What happens when I add or remove servers?

- When you add a server, they will automatically be activated provided that they have the right [prerequisites](#prerequisites). 
- If you are using legacy OS support, you may need to manually enable for these servers. Run `Enable-AzStackHCIAttestation [[-ComputerName] <String>]` in PowerShell. You can still delete servers or remove them from the cluster as usual. The vSwitch **AZSHCI_HOST-IMDS_DO_NOT_MODIFY** will still exist on the server after removal from the cluster. You can leave it if you are planning to add the server back to the cluster later, or you can remove it manually.

## Next steps

- [Extended security updates (ESU) on Azure Stack HCI](azure-benefits-esu.md)
- [Azure Stack HCI overview](../overview.md)
- [Azure Stack HCI FAQ](../faq.yml)
