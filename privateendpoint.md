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

## Use Cases for Private Link with Azure Database for PostgreSQL - Flexible Server in Private Preview

Clients can connect to the private endpoint from the same VNet, [peered VNet](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-network-peering-overview) in same region or across regions, or via [VNet-to-VNet connection](https://learn.microsoft.com/en-us/azure/vpn-gateway/vpn-gateway-howto-vnet-vnet-resource-manager-portal) across regions. Additionally, clients can connect from on-premises using ExpressRoute, private peering, or VPN tunneling. Below is a simplified diagram showing the common use cases.

<image type="content" src="./media/show-private-link-overview.png" alt-text="Diagram that shows how Azure Private Link works with Private Endpoints." />

### Limitations and Supported Features for Private Link Private Preview with Azure Database for PostgreSQL - Flexible Server

In the private preview of Private Endpoint for PostgreSQL flexible server, we only support single-instance, non-HA servers. Most features will work, but anything that communicates to other servers/instances we've had to redesign because we've changed the backend architecture for even better security.

Some of the features that do not work yet may be made available during private preview with normal service updates. Most (if not all) will be available with public preview in a couple of months.

Cross Feature Availability Matrix

| **Feature**   | **Availability** | **Notes**  |
| --------- | ------------ | --------- |
| High Availability (HA)    | No   |    |
| Read Replica   | No   |    |
| Geo Read Replica   | No   |    |
| Point in Time Restore (PITR)   | No   |    |
| Allowing also public/internet access with firewall rules   | No   |    |
|Major Version Upgrade (MVU)   | Yes   |  Works as designed  |
| Azure Active Directory Authentocation (AAD Auth)   | Yes   | Works as designed   |
| Connection pooling with PGBouncer   | Yes   |  Works as designed  |
| Private Endpoint DNS  | Yes   |  Works as designed and [documented](https://learn.microsoft.com/en-us/azure/private-link/private-endpoint-dns) |

### Connecting from an Azure VM in Peered Virtual Network (VNet)

Configure [VNet peering](https://learn.microsoft.com/en-us/azure/virtual-network/tutorial-connect-virtual-networks-powershell) to establish connectivity to the Azure Database for PostgreSQL - Flexible server from an Azure VM in a peered VNet.

### Connecting from an Azure VM in VNet-to-VNet environment

Configure [VNet-to-VNet VPN gateway](https://learn.microsoft.com/en-us/azure/vpn-gateway/vpn-gateway-howto-vnet-vnet-resource-manager-portal) connection to establish connectivity to a Azure Database for PostgreSQL - Flexible server from an Azure VM in a different region or subscription.


### Connecting from an on-premises environment over VPN

To establish connectivity from an on-premises environment to the Azure Database for PostgreSQL - Flexible server, choose and implement one of the options:
 - [Point-to-Site Connection](https://learn.microsoft.com/en-us/azure/vpn-gateway/vpn-gateway-howto-point-to-site-rm-ps)
 - [Site-to-Site VPN Connection](https://learn.microsoft.com/en-us/azure/vpn-gateway/vpn-gateway-create-site-to-site-rm-powershell)
 - [ExpressRoute Circuit](https://learn.microsoft.com/en-us/azure/expressroute/expressroute-howto-linkvnet-portal-resource-manager)


## Network Security and Private Link

When you use private endpoints, traffic is secured to a private-link resource. The platform validates network connections, allowing only those that reach the specified private-link resource. To access more subresources within the same Azure service, more private endpoints with corresponding targets are required. In the case of Azure Storage, for instance, you would need separate private endpoints to access the file and blob subresources.

Private endpoints provide a privately accessible IP address for the Azure service, but do not necessarily restrict public network access to it. All other Azure services require additional [access controls](https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-ip-filtering), however. These controls provide an extra network security layer to your resources, providing protection that helps prevent access to the Azure service associated with the private-link resource.

Private endpoints support network policies. Network policies enable support for Network Security Groups (NSG), User Defined Routes (UDR), and Application Security Groups (ASG). For more information about enabling network policies for a private endpoint, see [Manage network policies for private endpoints](https://learn.microsoft.com/en-us/azure/private-link/disable-private-endpoint-network-policy?tabs=network-policy-portal). To use an ASG with a private endpoint, see [Configure an application security group (ASG) with a private endpoint](https://learn.microsoft.com/en-us/azure/private-link/configure-asg-private-endpoint?tabs=portal).
## Private Link and DNS

When using a private endpoint, you need to connect to the same Azure service but use the private endpoint IP address. The intimate endpoint connection requires separate DNS settings to resolve the private IP address to the resource name.
Private DNS zones provide domain name resolution within a virtual network without a custom DNS solution. You link the private DNS zones to each virtual network to provide DNS services to that network.

Private DNS zones provide separate DNS zone names for each Azure service. For example, if you configured a private DNS zone for the storage account blob service in the previous image, the DNS zones name is **privatelink.blob.core.windows.net**. Check out the Microsoft documentation here to see more of the private DNS zone names for all Azure services.
> [!NOTE]
>Private endpoint private DNS zone configurations will only automatically generate if you use the recommended naming scheme: **privatelink.postgres.database.azure.com**

## Private Link and Network Security Groups

By default, network policies are disabled for a subnet in a virtual network. To utilize network policies like User-Defined Routes and Network Security Groups support, network policy support must be enabled for the subnet. This setting is only applicable to private endpoints within the subnet. This setting affects all private endpoints within the subnet. For other resources in the subnet, access is controlled based on security rules in the network security group.

Network policies can be enabled either for Network Security Groups only, for User-Defined Routes only, or for both. For more you can see [Azure docs](https://learn.microsoft.com/azure/private-link/disable-private-endpoint-network-policy?tabs=network-policy-portal)

Limitations to Network Security Groups (NSG) and Private Endpoints are listed [here](https://learn.microsoft.com/en-us/azure/private-link/private-endpoint-overview#limitations)


## Private Link combined with firewall rules

The following situations and outcomes are possible when you use Private Link in combination with firewall rules:

* If you don't configure any firewall rules, then by default, no traffic will be able to access the Azure Database for PostgreSQL Flexible server.

* If you configure public traffic or a service endpoint and you create private endpoints, then different types of incoming traffic are authorized by the corresponding type of firewall rule.

* If you don't configure any public traffic or service endpoint and you create private endpoints, then the Azure Database for PostgreSQL Flexible server is accessible only through the private endpoints. If you don't configure public traffic or a service endpoint, after all approved private endpoints are rejected or deleted, no traffic will be able to access the Azure Database for PostgreSQL Flexible server. 


## Configure Private Link for Azure Database for PostgreSQL - Flexible Server (Preview) via Portal

Private endpoints are required to enable Private Link. This can be done using the following steps:

### Prerequisites
 * An Azure Database for PostgreSQL - Flexible Server and databases

### Sign in Azure

Sign in to the [Azure portal](https://portal.azure.com).


### Create Virtual Network

In this section, you will create a Virtual Network in the location\region where you will create your Private Endpoint later in the tutorial, and the subnet to host the VM that is used to access your Private Link resource.
1. On the upper-left side of the screen, select **Create a resource** > **Networking** > **Virtual network**.
2. In **Create virtual network**, enter or select this information:

| **Setting** | **Value**|
|---------|------|
|Name     |  Enter *MyVirtualNetwork*    |
|Address space| Enter *10.1.0.0/16*      |
|Subscription | Select your subscription|
|Resource group | Select **Create new**, enter *myResourceGroup*, then select **OK**.|
|Location| Select location\region , example  **West Europe**.|
|Subnet - Name| Enter *mySubnet*|
|Subnet - Address range| Enter *10.1.0.0/24*|

3. Leave the rest as default and select *Create*.

### Create an Azure Database for PostgreSQL - Flexible Server with Private Endpoint
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
|Location|Select an Azure region where you want to want your PostgreSQL Server to reside, example  **West Europe**|
|Version|Select the database version of the PostgreSQL server that is required|
|Compute + Storage|Select the pricing tier that is needed for the server based on the workload|

5. Select **Next:Networking**
6. Keep **"Public Access"** checkbox checked as Connectivity method
7. Click **"Add Private Endpoint"** in Private Endpoint section

<image type="content" src="./media/private-endpoint-summary.png" alt-text="Diagram that shows how Azure Private Link works with Private Endpoints." />

8. In Create Private Endpoint Screen enter following:

| **Setting** | **Value**|
|---------|------|
|Subscription| Select your subscription|
|Resource group| Select **myResourceGroup**. You created this in the previous section|
|Location|Select an Azure region where you created your VNET above, example  **West Europe**|
|Name|Name of Private Endpoint|
|Target sub-resource|postgresqlServer|
|NETWORKING|
|Virtual Network|  Enter **MyVirtualNetwork**. You created this in the previous section |
|Subnet|Enter **mySubnet**.You created this in the previous section |
|PRIVATE DNS INTEGRATION]
|Integrate with Private DNS Zone| **Yes**|
|Private DNS Zone| Pick **(New)privatelink.postgresql.database.azure.com**. This will create new private DNS zone.|


9. Select **OK**.
10. Select **Review + create**. You're taken to the **Review + create** page where Azure validates your configuration.
11. Networking section of the **Review + Create** page will list your Private Endpoint information
<image type="content" src="./media/create-summary-page.png" alt-text="Review page showing networking section" />

12. When you see the Validation passed message, select **Create**.



### Approval Process for Private Endpoint

With separation of duties, common in many enterprises today, creation of cloud networking infrastructure, such as Azure Private Link services, are done by network administrator, whereas database servers are commonly created and managed by database administrator (DBA).
Once the network administrator creates the private endpoint (PE), the PostgreSQL database administrator (DBA) can manage the private endpoint Connection (PEC) to Azure Database for PostgreSQL. 
1. Navigate to the Azure Database for PostgreSQL - Flexible Server resource in the Azure portal.
    - Select Networking in the left pane
    - Shows a list of all private endpoint Connections (PECs)
    - Corresponding private endpoint (PE) created
    - Select an individual PEC from the list by selecting it
    - The PostgreSQL server admin can choose to approve or reject a PEC and optionally add a short text response
    - After approval or rejection, the list will reflect the appropriate state along with the response text
### Access the PostgreSQL server privately from the VM on the same network

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
|User name |Enter username  which is provided during the PostgreSQL Flexible server creation.|
|Password |Enter a password provided during the PostgreSQL server creation|
|SSL|Select Required|
6. Select **Connect**.
7. Browse databases from left menu
8. (Optionally) Create or query information from the postgreSQL server
9. Close the remote desktop connection to myVm

## Support


For any questions and feedback, please reach out to [FlexPEPreview@microsoft.com](mailto:flexpepreview@microsoft.com)