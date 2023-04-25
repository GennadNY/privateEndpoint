---
title: Networking Private Link Private Preview - Azure Database for PostgreSQL - Flexible Server
description: Learn about CMLNetworking Private Link Private Preview option for Azure Database for PostgreSQL.
ms.service: postgresql
ms.subservice: flexible-server
ms.topic: conceptual
ms.author: gennadNY
author: gennadNY
ms.date: 06/07/2022
---
# Azure Database for PostgreSQL Flexible Server Networking with Private Link - Preview

**Azure Private Link** allows you to create private endpoints for Azure Database for PostgreSQL - Flexible server to bring it inside your Virtual Network (VNet), in addition to already [existing networking capabilities provided by VNET injection](https://learn.microsoft.com/azure/postgresql/flexible-server/concepts-networking) currently in general availability with Azure Database for PostgreSQL - Flexible Server. With Private Link,traffic between your virtual network and the service travels the Microsoft backbone network. Exposing your service to the public internet is no longer necessary. You can create your own private link service in your virtual network and deliver it to your customers. Setup and consumption using Azure Private Link is consistent across Azure PaaS, customer-owned, and shared partner services.

Private Link is exposed to users through two  Azure resource types:

- Private Endpoints (Microsoft.Network/PrivateEndpoints)
- Private Link Services (Microsoft.Network/PrivateLinkServices)

## Private Endpoints
A **Private Endpoint** adds a network interface to a resource, providing it with a private IP address assigned from your VNET (Virtual Network). Once applied, you can communicate with this resource exclusively via the virtual network (VNET).
For a list to PaaS services that support Private Link functionality, review the Private Link [documentation](https://learn.microsoft.com/en-us/azure/private-link/). A private endpoint is a private IP address within a specific [VNet](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-networks-overview) and Subnet.

The same public service instance can be referenced by multiple private endpoints in different VNets/subnets, even if they belong to different users/subscriptions (including within differing Azure Active Directory (AAD) tenants) or if they have overlapping address spaces.

<image type="content" src="./media/azure-private-link.png" alt-text="Diagram that shows how Azure Private Link works with Private Endpoints." />


## Key Benefits of Azure Private Link

Azure Private Link provides the following benefits:

- **Privately access services on the Azure platform:** Connect your virtual network using private endpoints to all services that can be used as application components in Azure. Service providers can render their services in their own virtual network and consumers can access those services in their local virtual network. The Private Link platform will handle the connectivity between the consumer and services over the Azure backbone network.

- **On-premises and peered networks:** Access services running in Azure from on-premises over ExpressRoute private peering, VPN tunnels, and peered virtual networks using private endpoints. There's no need to configure ExpressRoute Microsoft peering or traverse the internet to reach the service. Private Link provides a secure way to migrate workloads to Azure.

- **Protection against data leakage:** A private endpoint is mapped to an instance of a PaaS resource instead of the entire service. Consumers can only connect to the specific resource. Access to any other resource in the service is blocked. This mechanism provides protection against data leakage risks.

- **Global reach: Connect privately to services running in other regions.** The consumer's virtual network could be in region A and it can connect to services behind Private Link in region B.

## Private Link and DNS

When using a private endpoint, you need to connect to the same Azure service but use the private endpoint IP address. The intimate endpoint connection requires separate DNS settings to resolve the private IP address to the resource name.
Private DNS zones provide domain name resolution within a virtual network without a custom DNS solution. You link the private DNS zones to each virtual network to provide DNS services to that network.

Private DNS zones provide separate DNS zone names for each Azure service. For example, if you configured a private DNS zone for the storage account blob service in the previous image, the DNS zones name is **privatelink.blob.core.windows.net**. Check out the Microsoft documentation here to see more of the private DNS zone names for all Azure services.
> [!NOTE]
>Private endpoint private DNS zone configurations will only automatically generate if you use the recommended naming scheme: **privatelink.postgres.database.azure.com**

## Private Link and Network Security Groups

By default, network policies are disabled for a subnet in a virtual network. To utilize network policies like User-Defined Routes and Network Security Groups support, network policy support must be enabled for the subnet. This setting is only applicable to private endpoints within the subnet. This setting affects all private endpoints within the subnet. For other resources in the subnet, access is controlled based on security rules in the network security group.

Network policies can be enabled either for Network Security Groups only, for User-Defined Routes only, or for both. For more you can see [Azure docs](https://learn.microsoft.com/azure/private-link/disable-private-endpoint-network-policy?tabs=network-policy-portal)

## Private Link combined with firewall rules

The following situations and outcomes are possible when you use Private Link in combination with firewall rules:

* If you don't configure any firewall rules, then by default, no traffic will be able to access the Azure Database for PostgreSQL Single server.

* If you configure public traffic or a service endpoint and you create private endpoints, then different types of incoming traffic are authorized by the corresponding type of firewall rule.

* If you don't configure any public traffic or service endpoint and you create private endpoints, then the Azure Database for PostgreSQL Single server is accessible only through the private endpoints. If you don't configure public traffic or a service endpoint, after all approved private endpoints are rejected or deleted, no traffic will be able to access the Azure Database for PostgreSQL Single server. 


## Configure Private Link for Azure Database for PostgreSQL - Flexible Server (Preview) via Portal

Private endpoints are required to enable Private Link. This can be done using the following steps:

### Prerequisites
 * An Azure Database for PostgreSQL - Flexible Server and databases

### Sign in Azure

Sign in to the [Azure portal](https://portal.azure.com).

### Create an Azure VM

In this section, you will create virtual network and the subnet to host the VM that is used to access your Private Link resource (a PostgreSQL Flexible server in Azure).

#### Create Virtual Network

In this section, you will create a Virtual Network and the subnet to host the VM that is used to access your Private Link resource.
1. On the upper-left side of the screen, select **Create a resource** > **Networking** > **Virtual network**.
2. In **Create virtual network**, enter or select this information:

| **Setting** | **Value**|
|---------|------|
|Name     |  Enter *MyVirtualNetwork*    |
|Address space| Enter *10.1.0.0/16*      |
|Subscription | Select your subscription|
|Resource group | Select **Create new**, enter *myResourceGroup*, then select **OK**.|
|Location| Select **West Europe**.|
|Subnet - Name| Enter *mySubnet*|
|Subnet - Address range| Enter *10.1.0.0/24*|

3. Leave the rest as default and select *Create*.

#### Create Virtual Machine

1. On the upper-left side of the screen in the Azure portal, select **Create a resource** > **Compute** > **Virtual Machine**.
2. In **Create a virtual machine - Basics**, enter or select this information:

| **Setting** | **Value**|
|---------|------|
|PROJECT DETAILS 
|Subscription| Select your subscription|
|Resource group | Select myResourceGroup. You created this in the previous section|
|INSTANCE DETAILS|
|Virtual machine name| Enter *myVm* |
|Region | Select **West Europe**|
|Availability options| Leave the default **No infrastructure redundancy required**|
|Image| Select **Windows Server 2019 Datacenter**|
|Size| Leave the default **Standard DS1 v2** |
|ADMINISTRATOR ACCOUNT|
|Username| Enter a username of your choosing|
|Pasword| Enter a password of your choosing. The password must be at least 12 characters long and meet the [defined complexity requirements](https://learn.microsoft.com/en-us/azure/virtual-machines/windows/faq?toc=%2Fazure%2Fvirtual-network%2Ftoc.json#what-are-the-password-requirements-when-creating-a-vm-)| 
|Confirm Password| Reenter password|
|INBOUND PORT RULES|
|Public inbound port| Leave the default None|
| SAVE MONEY|
|Already have a Windows license?| Leave the default **No**.

3. Select **Next: Disks**.
4. In **Create a virtual machine - Disks**, leave the defaults and select **Next: Networking**.
5. In **Create a virtual machine - Networking**, select this information:

| **Setting** | **Value**|
|---------|------|
|Virtual network | Leave the default *MyVirtualNetwork*|
|Address space | Leave the default **10.1.0.0/24** |
|Subnet| Leave the default **mySubnet (10.1.0.0/24)** |
|Public IP| Leave the default **(new) myVm-ip** |
|Public inbound ports| Select **Allow selected ports** |
|Select inbound ports| Select **HTTP** and **RDP**| 
6. Select **Review + create**. You're taken to the **Review + create** page where Azure validates your configuration.
7. When you see the **Validation passed** message, select **Create**.


#### Create an Azure Database for PostgreSQL - Flexible Server
In this section, you will create an Azure Database for PostgreSQL - Flexible Server in Azure.

To create an Azure Database for PostgreSQL server, take the following steps:

1. Select Create a resource (+) in the upper-left corner of the portal.

2. Select Databases > Azure Database for PostgreSQL.

3. Select the Flexible server deployment option.

4. Fill out the Basics form with the following information

| **Setting** | **Value**|
|---------|------|
|Subscription| Select your subscription|
|Resource group| Select **myResourceGroup**. You created this in the previous section|
|Server name| Enter *myserver*. If this name is taken, create a unique name|
|Admin username |Enter an administrator name of your choosing|
|Password|Enter a password of your choosing. The password must be at least 8 characters long and meet the defined requirements|
|Location|Select an Azure region where you want to want your PostgreSQL Server to reside|
|Version|Select the database version of the PostgreSQL server that is required|
|Compute + Storage|elect the pricing tier that is needed for the server based on the workload|

5. Select **OK**.
6. Select **Review + create**. You're taken to the **Review + create** page where Azure validates your configuration.
7. When you see the Validation passed message, select **Create**.

#### Connect to a VM using Remote Desktop (RDP)

After you've created Azure virtual machine called *myVm*, connect to it from the internet as follows:

1. In the portal's search bar, enter *myVm*.

2. Select the **Connect** button. After selecting the Connect button, Connect to virtual machine opens.

3. Select Download RDP File. Azure creates a Remote Desktop Protocol (.rdp) file and downloads it to your computer.

4. Open the downloaded.rdp file.

    - If prompted, select **Connect**.

    - Enter the username and password you specified when creating the VM.
5. Select OK.
6. You may receive a certificate warning during the sign-in process. If you receive a certificate warning, select Yes or Continue.
7. Once the VM desktop appears, minimize it to go back to your local desktop.

#### Access the PostgreSQL server privately from the VM

1. In the Remote Desktop of *myVM*, open PowerShell.

2. Enter *nslookup mydemopostgresserver.privatelink.postgres.database.azure.com*.

3. You'll receive a message similar to this:

  ```azurepowershell
  Server:  UnKnown
Address:  168.63.129.16
Non-authoritative answer:
Name:    mydemopostgresserver.privatelink.postgres.database.azure.com
Address:  10.1.3.4
```

4. Test the private link connection for the PostgreSQL server using any available client. In the example below I have used [Azure Data studio](https://learn.microsoft.com/sql/azure-data-studio/download-azure-data-studio?view=sql-server-ver16&tabs=redhat-install%2Credhat-uninstall) to do the operation.

5. In New connection, enter or select this information:

| **Setting** | **Value**|
|---------|------|
|Server type |Select PostgreSQL|
|Server name |Select mydemopostgresserver.privatelink.postgres.database.azure.com|
|User name |Enter username as username@servername which is provided during the PostgreSQL server creation.|
|Password |Enter a password provided during the PostgreSQL server creation|
|SSL|Select Required|
6. Select **Connect**.
7. Browse databases from left menu
8. (Optionally) Create or query information from the postgreSQL server
9. Close the remote desktop connection to myVm
