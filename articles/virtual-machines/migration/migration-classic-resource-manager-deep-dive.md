---
title: Platform-supported migration tool.
description: Technical deep dive on platform-supported migration of resources from the classic deployment model to Azure Resource Manager.
author: oriwolman
manager: vashan
ms.service: azure-virtual-machines
ms.subservice: classic-to-arm-migration
ms.topic: concept-article
ms.date: 1/25/2023
ms.author: oriwolman
ms.custom: compute-evergreen, devx-track-arm-template
---

# Technical deep dive on platform-supported migration from classic to Azure Resource Manager

**Applies to:** :heavy_check_mark: Linux VMs :heavy_check_mark: Windows VMs

> [!IMPORTANT]
> Today, about 90% of IaaS VMs are using [Azure Resource Manager](https://azure.microsoft.com/features/resource-manager/). As of February 28, 2020, classic VMs have been deprecated and will be fully retired on September 6, 2023. [Learn more]( https://aka.ms/classicvmretirement) about this deprecation and [how it affects you](../classic-vm-deprecation.md#how-does-this-affect-me).

Let's take a deep-dive on migrating from the Azure classic deployment model to the Azure Resource Manager deployment model. We look at resources at a resource and feature level to help you understand how the Azure platform migrates resources between the two deployment models. For more information, please read the service announcement article: [Platform-supported migration of IaaS resources from classic to Azure Resource Manager](migration-classic-resource-manager-overview.md).


## Migrate IaaS resources from the classic deployment model to Azure Resource Manager
First, it's important to understand the difference between data-plane and management-plane operations on the infrastructure as a service (IaaS) resources.

* *Management/control plane* describes the calls that come into the management/control plane or the API for modifying resources. For example, operations like creating a VM, restarting a VM, and updating a virtual network with a new subnet manage the running resources. They don't directly affect connecting to the VMs.
* *Data plane* (application) describes the runtime of the application itself, and involves interaction with instances that don’t go through the Azure API. For example, accessing your website, or pulling data from a running SQL Server instance or a MongoDB server, are data plane or application interactions. Other examples include copying a blob from a storage account, and accessing a public IP address to use Remote Desktop Protocol (RDP) or Secure Shell (SSH) into the virtual machine. These operations keep the application running across compute, networking, and storage.

The data plane is the same between the classic deployment model and Resource Manager stacks. The difference is that during the migration process, Microsoft translates the representation of the resources from the classic deployment model to that in the Resource Manager stack. As a result, you need to use new tools, APIs, and SDKs to manage your resources in the Resource Manager stack.

![Screenshot that shows the difference between management/control plane and data plane.](../media/virtual-machines-windows-migration-classic-resource-manager/data-control-plane.png)


> [!NOTE]
> In some migration scenarios, the Azure platform stops, deallocates, and restarts your virtual machines. This causes a brief data-plane downtime.
>

## The migration experience
Before you start the migration:

* Ensure that the resources that you want to migrate don't use any unsupported features or configurations. Usually the platform detects these issues and generates an error.
* If you have VMs that aren't in a virtual network, they're stopped and deallocated as part of the prepare operation. If you don't want to lose the public IP address, consider reserving the IP address before triggering the prepare operation. If the VMs are in a virtual network, they aren't stopped and deallocated.
* Plan your migration during non-business hours to accommodate for any unexpected failures that might happen during migration.
* Download the current configuration of your VMs by using PowerShell, command-line interface (CLI) commands, or REST APIs to make it easier for validation after the prepare step is complete.
* Update your automation and operationalization scripts to handle the Resource Manager deployment model, before you start the migration. You can optionally do GET operations when the resources are in the prepared state.
* Evaluate the Azure role-based access control (Azure RBAC) policies that are configured on the IaaS resources in the classic deployment model, and plan for after the migration is complete.

The migration workflow is as follows:

![Screenshot that shows the migration workflow.](../media/migration-classic-resource-manager/migration-workflow.png)

> [!NOTE]
> The operations described in the following sections are all idempotent. If you have a problem other than an unsupported feature or a configuration error, retry the prepare, abort, or commit operation. Azure tries the action again.
>
>

### Validate
The validate operation is the first step in the migration process. The goal of this step is to analyze the state of the resources you want to migrate in the classic deployment model. The operation evaluates whether the resources are capable of migration (success or failure).

You select the virtual network or a cloud service (if it’s not in a virtual network) that you want to validate for migration. If the resource isn't capable of migration, Azure lists the reasons why.

#### Checks not done in the validate operation

The validate operation only analyzes the state of the resources in the classic deployment model. It can check for all failures and unsupported scenarios due to various configurations in the classic deployment model. It isn't possible to check for all issues that the Azure Resource Manager stack might impose on the resources during migration. These issues are only checked when the resources undergo transformation in the next step of migration (the prepare operation). The following table lists all the issues not checked in the validate operation:


|Networking checks not in the validate operation|
|-|
|A virtual network having both ER and VPN gateways.|
|A virtual network gateway connection in a disconnected state.|
|All ER circuits are pre-migrated to Azure Resource Manager stack.|
|Azure Resource Manager quota checks for networking resources. For example: static public IP, dynamic public IPs, load balancer, network security groups, route tables, and network interfaces. |
| All load balancer rules are valid across deployment and the virtual network. |
| Conflicting private IPs between stop-deallocated VMs in the same virtual network. |

### Prepare
The prepare operation is the second step in the migration process. The goal of this step is to simulate the transformation of the IaaS resources from the classic deployment model to Resource Manager resources. Further, the prepare operation presents this side-by-side for you to visualize.

> [!NOTE] 
> Your resources in the classic deployment model aren't modified during this step. It's a safe step to run if you're trying out migration. 

You select the virtual network or the cloud service (if it’s not a virtual network) that you want to prepare for migration.

* If the resource isn't capable of migration, Azure stops the migration process and lists the reason why the prepare operation failed.
* If the resource is capable of migration, Azure locks down the management-plane operations for the resources under migration. For example, you aren't able to add a data disk to a VM under migration.

Azure then starts the migration of metadata from the classic deployment model to Resource Manager for the migrating resources.

After the prepare operation is complete, you have the option of visualizing the resources in both the classic deployment model and Resource Manager. For every cloud service in the classic deployment model, the Azure platform creates a resource group name that has the pattern `cloud-service-name>-Migrated`.

> [!NOTE]
> It isn't possible to select the name of a resource group created for migrated resources (that is, "-Migrated"). After migration is complete, however, you can use the move feature of Azure Resource Manager to move resources to any resource group you want. For more information, see [Move resources to new resource group or subscription](/azure/azure-resource-manager/management/move-resource-group-and-subscription).

The following two screenshots show the result after a successful prepare operation. The first one shows a resource group that contains the original cloud service. The second one shows the new "-Migrated" resource group that contains the equivalent Azure Resource Manager resources.

![Screenshot that shows original cloud service](../media/migration-classic-resource-manager/portal-classic.png)

![Screenshot that shows Azure Resource Manager resources in the prepare operation](../media/migration-classic-resource-manager/portal-arm.png)

Here's a behind-the-scenes look at your resources after the completion of the prepare phase. Note that the resource in the data plane is the same. It's represented in both the management plane (classic deployment model) and the control plane (Resource Manager).

![Diagram of the prepare phase](../media/migration-classic-resource-manager/behind-the-scenes-prepare.png)

> [!NOTE]
> VMs that aren't in a virtual network in the classic deployment model are stopped and deallocated in this phase of migration.
>

### Check (manual or scripted)
In the check step, you have the option to use the configuration that you downloaded earlier to validate that the migration looks correct. Alternatively, you can sign in to the portal, and spot check the properties and resources to validate that metadata migration looks good.

If you're migrating a virtual network, most configuration of virtual machines isn't restarted. For applications on those VMs, you can validate that the application is still running.

You can test your monitoring and operational scripts to see if the VMs are working as expected, and if your updated scripts work correctly. Only GET operations are supported when the resources are in the prepared state.

There's no set window of time before which you need to commit the migration. You can take as much time as you want in this state. However, the management plane is locked for these resources until you either abort or commit.

If you see any issues, you can always abort the migration and go back to the classic deployment model. After you go back, Azure opens the management-plane operations on the resources, so that you can resume normal operations on those VMs in the classic deployment model.

### Abort
This is an optional step if you want to revert your changes to the classic deployment model and stop the migration. This operation deletes the Resource Manager metadata (created in the prepare step) for your resources. 

![Screenshot that shows the abort step.](../media/migration-classic-resource-manager/behind-the-scenes-abort.png)


> [!NOTE]
> This operation can't be done after you have triggered the commit operation.     
>

### Commit
After you finish the validation, you can commit the migration. Resources don't appear anymore in the classic deployment model, and are available only in the Resource Manager deployment model. The migrated resources can be managed only in the new portal.

> [!NOTE]
> This is an idempotent operation. If it fails, retry the operation. If it continues to fail, create a support ticket or create a forum on [Microsoft Q&A](/answers/index.html)
>
>

![Screenshot that shows the commit step.](../media/migration-classic-resource-manager/behind-the-scenes-commit.png)

## Migration flowchart

Here's a flowchart that shows how to proceed with migration:

![Screenshot that shows the migration steps.](../media/migration-classic-resource-manager/migration-flow.png)

## Translation of the classic deployment model to Resource Manager resources
You can find the classic deployment model and Resource Manager representations of the resources in the following table. Other features and resources aren't currently supported.

| Classic representation | Resource Manager representation | Notes |
| --- | --- | --- |
| Cloud service name (Hosted Service Name) |DNS name |During migration, a new resource group is created for every cloud service with the naming pattern `<cloudservicename>-migrated`. This resource group contains all your resources. The cloud service name becomes a DNS name that is associated with the public IP address. |
| Virtual machine |Virtual machine |VM-specific properties are migrated unchanged. Certain osProfile information, like computer name, isn't stored in the classic deployment model, and remains empty after migration. |
| Disk resources attached to VM |Implicit disks attached to VM |Disks aren't modeled as top-level resources in the Resource Manager deployment model. They're migrated as implicit disks under the VM. Only disks that are attached to a VM are currently supported. Resource Manager VMs can now use storage accounts in the classic deployment model, which allows the disks to be easily migrated without any updates. |
| VM extensions |VM extensions |All the resource extensions, except XML extensions, are migrated from the classic deployment model. |
| Virtual machine certificates |Certificates in Azure Key Vault |If a cloud service contains service certificates, the migration creates a new Azure key vault per cloud service, and moves the certificates into the key vault. The VMs are updated to reference the certificates from the key vault. <br><br> Don't delete the key vault. This can cause the VM to go into a failed state. |
| WinRM configuration |WinRM configuration under osProfile |Windows Remote Management configuration is moved unchanged, as part of the migration. |
| Availability-set property |Availability-set resource | Availability-set specification is a property on the VM in the classic deployment model. Availability sets become a top-level resource as part of the migration. The following configurations aren't supported: multiple availability sets per cloud service, or one or more availability sets along with VMs that aren't in any availability set in a cloud service. |
| Network configuration on a VM |Primary network interface |Network configuration on a VM is represented as the primary network interface resource after migration. For VMs that aren't in a virtual network, the internal IP address changes during migration. |
| Multiple network interfaces on a VM |Network interfaces |If a VM has multiple network interfaces associated with it, each network interface becomes a top-level resource as part of the migration, along with all the properties. |
| Load-balanced endpoint set |Load balancer |In the classic deployment model, the platform assigned an implicit load balancer for every cloud service. During migration, a new load-balancer resource is created, and the load-balancing endpoint set becomes load-balancer rules. |
| Inbound NAT rules |Inbound NAT rules |Input endpoints defined on the VM are converted to inbound network address translation rules under the load balancer during the migration. |
| VIP address |Public IP address with DNS name |The virtual IP address becomes a public IP address, and is associated with the load balancer. A virtual IP can only be migrated if there's an input endpoint assigned to it. To retain the IP, you can [convert it to Reserved IP](/previous-versions/azure/virtual-network/virtual-networks-reserved-public-ip#reserve-the-ip-address-of-an-existing-cloud-service) before migration. There will be downtime of about 60 seconds during this change.|
| Virtual network |Virtual network |The virtual network is migrated, with all its properties, to the Resource Manager deployment model. A new resource group is created with the name `-migrated`. |
| Reserved IPs |Public IP address with static allocation method |Reserved IPs associated with the load balancer are migrated, along with the migration of the cloud service or the virtual machine. Unassociated reserved IPs can be migrated using [Move-AzureReservedIP](/powershell/module/servicemanagement/azure/move-azurereservedip).  |
| Public IP address per VM |Public IP address with dynamic allocation method |The public IP address associated with the VM is converted as a public IP address resource, with the allocation method set to dynamic. |
| NSGs |NSGs |Network security groups associated with a virtual machine or subnet are cloned as part of the migration to the Resource Manager deployment model. The NSG in the classic deployment model isn't removed during the migration. However, the management-plane operations for the NSG are blocked when the migration is in progress. Unassociated NSGs can be migrated using [Move-AzureNetworkSecurityGroup](/powershell/module/servicemanagement/azure/move-azurenetworksecuritygroup).|
| DNS servers |DNS servers |DNS servers associated with a virtual network or the VM are migrated as part of the corresponding resource migration, along with all the properties. |
| UDRs |UDRs |User-defined routes associated with a subnet are cloned as part of the migration to the Resource Manager deployment model. The UDR in the classic deployment model isn't removed during the migration. The management-plane operations for the UDR are blocked when the migration is in progress. Unassociated UDRs can be migrated using [Move-AzureRouteTable](/powershell/module/servicemanagement/azure/Move-AzureRouteTable). |
| IP forwarding property on a VM's network configuration |IP forwarding property on the NIC |The IP forwarding property on a VM is converted to a property on the network interface during the migration. |
| Load balancer with multiple IPs |Load balancer with multiple public IP resources |Every public IP associated with the load balancer is converted to a public IP resource, and associated with the load balancer after migration. |
| Internal DNS names on the VM |Internal DNS names on the NIC |During migration, the internal DNS suffixes for the VMs are migrated to a read-only property named “InternalDomainNameSuffix” on the NIC. The suffix remains unchanged after migration, and VM resolution should continue to work as previously. |
| Virtual network gateway |Virtual network gateway |Virtual network gateway properties are migrated unchanged. The VIP associated with the gateway doesn't change either. |
| Local network site |Local network gateway |Local network site properties are migrated unchanged to a new resource called a local network gateway. This represents on-premises address prefixes and the remote gateway IP. |
| Connections references |Connection |Connectivity references between the gateway and the local network site in network configuration is represented by a new resource called Connection. All properties of connectivity reference in network configuration files are copied unchanged to the Connection resource. Connectivity between virtual networks in the classic deployment model is achieved by creating two IPsec tunnels to local network sites representing the virtual networks. This is transformed to the virtual-network-to-virtual-network connection type in the Resource Manager model, without requiring local network gateways. |

## Changes to your automation and tooling after migration
As part of migrating your resources from the classic deployment model to the Resource Manager deployment model, you must update your existing automation or tooling to ensure that it continues to work after the migration.


## Next steps

* [Overview of platform-supported migration of IaaS resources from classic to Azure Resource Manager](migration-classic-resource-manager-overview.md)
* [Planning for migration of IaaS resources from classic to Azure Resource Manager](migration-classic-resource-manager-plan.md)
* [Use PowerShell to migrate IaaS resources from classic to Azure Resource Manager](migration-classic-resource-manager-ps.md)
* [Use CLI to migrate IaaS resources from classic to Azure Resource Manager](migration-classic-resource-manager-cli.md)
* [VPN Gateway classic to Resource Manager migration](/azure/vpn-gateway/vpn-gateway-classic-resource-manager-migration)
* [Migrate ExpressRoute circuits and associated virtual networks from the classic to the Resource Manager deployment model](/azure/expressroute/expressroute-migration-classic-resource-manager)
* [Community tools for assisting with migration of IaaS resources from classic to Azure Resource Manager](migration-classic-resource-manager-community-tools.md)
* [Review most common migration errors](migration-classic-resource-manager-errors.md)
* [Review the most frequently asked questions about migrating IaaS resources from classic to Azure Resource Manager](migration-classic-resource-manager-faq.yml)
