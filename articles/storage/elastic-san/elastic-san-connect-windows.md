---
title: Connect to an Azure Elastic SAN volume - Windows
description: Learn how to connect to an Azure Elastic SAN volume from an individual Windows client using iSCSI and ensure optimal performance.
author: roygara
ms.service: azure-elastic-san-storage
ms.topic: how-to
ms.date: 06/18/2025
ms.author: rogarana
ms.custom: references_regions
# Customer intent: "As a Windows client user, I want to connect to an Azure Elastic SAN volume using iSCSI, so that I can ensure optimal performance and manage multiple I/O sessions efficiently."
---
# Connect to an Azure Elastic SAN volume - Windows

This article explains how to connect a Windows client to an Azure Elastic SAN volume.

There are two connection options:

- **Elastic SAN VM extension** – Use during VM or VM scale set deployment to automatically configure connectivity. This option reduces onboarding steps and is recommended when deploying new VMs at scale.
- **Manual connect script** – Use after a VM is deployed to configure connectivity with specific parameters. This option provides flexibility for existing VMs and advanced configurations.

Choose the option that best fits your deployment scenario. Both approaches require a deployed Elastic SAN resource and configured volume groups.

## Connect during VM deployment using the Elastic SAN VM extension

Elastic SAN VM extension enables automatic volume connectivity during virtual machine provisioning. It installs and configures itself without requiring manual setup, streamlining deployment workflows. This is especially beneficial for storage-intensive workloads.
The extension is integrated into the Azure Marketplace and is available as a selectable tile during VM setup. You can also reconfigure the extension after deployment to initiate new connection or disconnection requests.

### Prerequisites

- An existing Elastic SAN resource with configured volume groups.  
- Permissions to deploy VMs and install extensions in your subscription.  

### Use the VM extension during deployment

1. In the Azure portal, start the VM creation process.  
2. On the **Extensions** tab, select the **Elastic SAN VM extension** from the Marketplace.  
3. Provide the required parameters:  
   - Elastic SAN name  
   - Volume group name  
   - Number of sessions  
   - Connection mode (for example, read/write)  
4. Complete VM creation. The extension automatically configures the iSCSI initiator and establishes connectivity with the Elastic SAN volumes.  

### Modify or reconfigure the extension after deployment

1. In the Azure portal, go to the **Extensions + applications** blade of the VM or VM scale set.  
2. Select the **Elastic SAN extension**.  
3. Update configuration parameters and save changes. The extension applies the new settings to manage connections.  

### Supported scenarios

The extension supports both single VM deployments and VM Scale Sets. For single VMs, it connects Elastic SAN volumes automatically during provisioning. For VM Scale Sets, applying the extension at the scale set level ensures uniform volume connectivity across all instances.


## Connect manually to Elastic SAN volumes - Windows

This section explains how to connect to an Elastic SAN volume from an individual Windows client. For details on connecting from a Linux client, see [Connect to Elastic SAN volumes - Linux](elastic-san-connect-linux.md).

In this section, you configure your volume group to allow connections from your subnet and then you configure your client environment to connect to an Elastic SAN volume and establish an iSCSI connection. For best performance, ensure that your VM and your Elastic SAN are in the same zone.

You must use a cluster manager when connecting an individual Elastic SAN volume to multiple clients. For details, see [Use clustered applications on Azure Elastic SAN](elastic-san-shared-volumes.md).

## Prerequisites

- Use either the [latest Azure CLI](/cli/azure/install-azure-cli) or install the [latest Azure PowerShell module](/powershell/azure/install-azure-powershell)
- [Deploy an Elastic SAN](elastic-san-create.md)
- Either [configure private endpoints](elastic-san-configure-private-endpoints.md) or [configure service endpoints](elastic-san-configure-service-endpoints.md)

## Enable iSCSI Initiator

To create iSCSI connections from a Windows client, confirm the iSCSI service is running. If it's not, start the service, and set it to start automatically.

```powershell
# Confirm iSCSI is running
Get-Service -Name MSiSCSI

# If it's not running, start it
Start-Service -Name MSiSCSI

# Set it to start automatically
Set-Service -Name MSiSCSI -StartupType Automatic
```

## Install Multipath I/O

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

## Attach volumes to the client

Use the following script to create your connections. To execute it, collect or determine the following parameters: 
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

### Set session number

Before you run the script, determine how many sessions your volume needs. To be able to reach a volume's highest IOPS and throughput capacities, you'll need 32 sessions. But, because Windows iSCSI initiator has a maximum of 256 sessions, you may need to use fewer than 32 sessions if you're connecting more than eight volumes to a Windows client.

> [!NOTE]
> Use the `-NumSession` parameter to set the number of sessions. The parameter accepts values from 1 to 32, and has a default value of 32.


```bash
.\connect.ps1 ` 

  -ResourceGroupName "<resource-group>" ` 

  -ElasticSanName "<esan-name>" ` 

  -VolumeGroupName "<volume-group>" ` 

  -VolumeName "<volume1>", "<volume2>" ` 

  -NumSession “<value>”
```

Verify the number of sessions your volume has with either `iscsicli SessionList` or `mpclaim -s -d`



## Next steps

[Configure Elastic SAN networking](elastic-san-networking.md)


