---
title: Connect to an Azure Elastic SAN volume - Windows
description: Learn how to connect to an Azure Elastic SAN volume by using the Windows VM Extension or the manual connect script.
author: roygara
ms.service: azure-elastic-san-storage
ms.topic: how-to
ms.date: 12/02/2025
ms.author: rogarana
ms.custom: references_regions
---
# Connect to Elastic SAN volumes - Windows

This article explains how to connect a Windows client to an Azure Elastic SAN volume. For details on connecting from a Linux client, see [Connect to Elastic SAN volumes - Linux](elastic-san-connect-linux.md).

Azure Elastic SAN supports two main connection methods for VMs: 

- **Elastic SAN VM Extension**: Best for new VMs or scale sets, enabling automated, consistent SAN connectivity at deployment time via the Azure portal, or when you want minimal manual steps. 

- **Manual Connect Script**: Best for existing VMs, advanced customizations, or troubleshooting, where you need to run scripts directly. 

**Note!**: Both methods require that you have already deployed an Elastic SAN resource and configured at least one volume group and volume. The VM extension does not create SAN resources for you, it only connects VMs to existing SAN volumes. 

## Connect using the Elastic SAN VM extension  

Use the Elastic SAN VM extension when you want to automate the connection process from the Azure portal. You can use the extension in two ways: 

- **During VM creation**: Connect as part of provisioning. 

- **After VM deployment**: Connect or disconnect from an existing VM or ARM deployment script to deploy for multiple VMs at scale 


### Prerequisites
Before using the VM extension, ensure you have completed the following: 
1. [Deploy an Elastic SAN](elastic-san-create.md).
2. Create and configure at least one volume group and one volume within your Elastic SAN. 
3. Configure network access: set up either private endpoints or service endpoints so your VM can reach the Elastic SAN.
4. Obtain the Volume IQN:    
  a. The IQN (iSCSI Qualified Name) for each volume is required to establish the connection.
  b. Currently, the Azure portal does not display the volume IQN directly. You must retrieve when you create the Elastic SAN. The recommended approach is to run `az elastic-san volume show --name <volume-name> --resource-group <rg-name> --elastic-san-name <esan-name>` or use the corresponding PowerShell cmdlet to fetch the IQN.  
  c. Document and save the IQN(s) and target portal name for all volumes you plan to connect.
 
### What the VM extension configures on Windows 
When you configure the extension with a **Connect** command, it ensures that: 
- The iSCSI service is enabled and running. 
- Multipath I/O (MPIO) is installed and configured. 
- The specified Elastic SAN volumes are connected using the volume IQNs, target portal addresses, and session count(s) that you provide. For multiple volumes, there should be a 1:1 mapping between each volume and its portal address, and each volume establishes exactly the number of sessions you specify. 

When you configure the extension with a **Disconnect** command, it ensures that: 
- Disconnects the specified volumes. 
- Cleans up the corresponding entries from the persistent target database in line with the existing disconnection scripts. 

**Note!**: The extension only acts on the parameters you provide. For multiple volumes or IQNs, these should be entered as comma-separated values (e.g., volume1, volume2, volume3 and corresponding IQNs). You must also specify the session count (for connect). The extension does not infer or maintain a history of prior connections. 
  
### Install and configure the VM extension during VM creation 
Use this path when you’re creating a new VM and want it to come up already connected to Elastic SAN: 
1. Sign in to the [Azure portal](https://portal.azure.com/).
2. Fill in the required fields on the Basics, Disks, and Networking tabs as usual. 
3. Navigate to the **Extensions + applications** tab during VM creation. 
4. Select **Add** and search for the **Elastic SAN Extension for Windows** in the Marketplace tiles. 
5. Select the extension to open its configuration panel.
6. On the configuration page, provide the required connection parameters: **Volume name(s)**, **Target IQN(s)**, **Target portal adress(es)** and **Sessions per target**.
7. Proceed to the **Review+ create** screen.
8. Select **Create** to finish VM creation.

Once the VM deploys successfully, the extension will run automatically within minutes and: 
- Enable iSCSI and MPIO if needed. 
- Initiate the requested connections to your Elastic SAN volumes using the IQNs and session count you provided. 

If any validation or connection step fails (e.g., invalid resource name, unreachable volume group subnet, or session limit exceeded), the extension status in the portal will reflect the failure including an error message, and guide you through troubleshooting.

### Install and configure the VM extension on an existing VM 
Use this path when the VM is already deployed and you want to connect or disconnect Elastic SAN volumes from the portal. 
1. In the Azure portal, navigate to your Windows VM. 
2. Under Settings, select **Extensions + applications**.   
3. Select **Add** and search for the **Elastic SAN Extension for Windows** or open the existing Elastic SAN extension instance if already installed. 
4. Choose **Configure / Reconfigure**. 
5. In the extension configuration panel: 
  a. Choose the desired command, such as **Connect** or **Disconnect**.
  b. Provide the required parameters: **Volume name(s)**, **Target IQN(s)**, **Target portal adress(es)** and **Sessions per target** (for Connect only). 
  c. Proceed to apply the configuration.

The VM extension will immediately run the corresponding script on the VM and:
- Connect additional volumes later by reconfiguring the extension with more volumes. 
- Disconnect volumes by switching the command to disconnect and providing the same identifiers. 

**Note!**: Reconfiguring the VM extension does not reboot the VM and does not automatically alter existing connections unless you explicitly request a disconnect. The extension simply executes the newly requested connect or disconnect operation. 


## Manually connect to Elastic SAN volumes

This section explains how to connect to an Elastic storage area network (SAN) volume from an individual Windows client. For details on connecting from a Linux client, see [Connect to Elastic SAN volumes - Linux](elastic-san-connect-linux.md).

In this section, you configure your client environment to connect to an Elastic SAN volume and establish a connection. For best performance, ensure that your VM and your Elastic SAN are in the same zone.

When using the VM extension with Virtual Machine Scale Sets, each VM in the scale set is automatically connected to the Elastic SAN volume. If multiple VMs will access the same volume, you must use a cluster manager to coordinate shared access and maintain data consistency. For details, see [Use clustered applications on Azure Elastic SAN](elastic-san-shared-volumes.md).

### Prerequisites

- Use either the [latest Azure CLI](/cli/azure/install-azure-cli) or install the [latest Azure PowerShell module](/powershell/azure/install-azure-powershell)
- [Deploy an Elastic SAN](elastic-san-create.md)
- Either [configure private endpoints](elastic-san-configure-private-endpoints.md) or [configure service endpoints](elastic-san-configure-service-endpoints.md)


### Set up your client environment
#### Enable iSCSI Initiator
To create iSCSI connections from a Windows client, confirm the iSCSI service is running. If it's not, start the service, and set it to start automatically.

```powershell
# Confirm iSCSI is running
Get-Service -Name MSiSCSI

# If it's not running, start it
Start-Service -Name MSiSCSI

# Set it to start automatically
Set-Service -Name MSiSCSI -StartupType Automatic
```

#### Install Multipath I/O

To achieve higher IOPS and throughput to a volume and reach its maximum limits, you need to create multiple-sessions from the iSCSI initiator to the target volume based on your application's multi-threaded capabilities and performance requirements. You need Multipath I/O to aggregate these multiple paths into a single device, and to improve performance by optimally distributing I/O over all available paths based on a load balancing policy.

Install Multipath I/O, enable multipath support for iSCSI devices, and set a default load balancing policy.

```powershell
# Install Multipath-IO
Add-WindowsFeature -Name 'Multipath-IO'

# Verify if the installation was successful
Get-WindowsFeature -Name 'Multipath-IO'

# Enable multipath support for iSCSI devices
Enable-MSDSMAutomaticClaim -BusType iSCSI

# Set the default load balancing policy based on your requirements. In this example, we set it to round robin
# which should be optimal for most workloads.
mpclaim -L -M 2
```

#### Attach volumes to the client

You can use the following script to create your connections. To execute it, you require the following parameters: 
- $rgname: Resource Group Name
- $esanname: Elastic SAN Name
- $vgname: Volume Group Name
- $vol1: First Volume Name
- $vol2: Second Volume Name
and other volume names that you might require
- 32: Number of sessions to each volume

Copy the script from [here](https://github.com/Azure-Samples/azure-elastic-san/blob/main/PSH%20(Windows)%20Multi-Session%20Connect%20Scripts/ElasticSanDocScripts0523/connect.ps1) and save it as a .ps1 file, for example, connect.ps1. Then execute it with the required parameters. The following is an example of how to run the script: 

```bash
./connect.ps1 $rgname $esanname $vgname $vol1,$vol2,$vol3 32
```

Verify the number of sessions your volume has with either `iscsicli SessionList` or `mpclaim -s -d`

#### Number of sessions

You need to use 32 sessions to each target volume to achieve its maximum IOPS and/or throughput limits. Windows iSCSI initiator has a limit of maximum 256 sessions. If you need to connect more than 8 volumes to a Windows client, reduce the number of sessions to each volume.


```bash
.\connect.ps1 ` 

  -ResourceGroupName "<resource-group>" ` 

  -ElasticSanName "<esan-name>" ` 

  -VolumeGroupName "<volume-group>" ` 

  -VolumeName "<volume1>", "<volume2>" ` 

  -NumSession “<value>”

## Next steps

[Configure Elastic SAN networking](elastic-san-networking.md)


  -NumSession “<value>”
```

Verify the number of sessions your volume has with either `iscsicli SessionList` or `mpclaim -s -d`

## Next steps

[Configure Elastic SAN networking](elastic-san-networking.md)
