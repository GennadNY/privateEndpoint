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


## Configure Private Link for Azure Database for PostgreSQL - Flexible Server

Private endpoints are required to enable Private Link. This can be done using the following steps:

### Prerequisites
 * An Azure Database for PostgreSQL - Flexible Server and databases

### Azure Cloud Shell

Azure hosts Azure Cloud Shell, an interactive shell environment that you can use through your browser. You can use either Bash or PowerShell with Cloud Shell to work with Azure services. You can use the Cloud Shell preinstalled commands to run the code in this article, without having to install anything on your local environment.

To start Azure Cloud Shell:

| Option | Example/Link |
|-----------------------------------------------|---|
| Select **Try It** in the upper-right corner of a code or command block. Selecting **Try It** doesn't automatically copy the code or command to Cloud Shell. | ![Screenshot that shows an example of Try It for Azure Cloud Shell.](./media/hdi-azure-cli-try-it.png) |
| Go to [https://shell.azure.com](https://shell.azure.com), or select the **Launch Cloud Shell** button to open Cloud Shell in your browser. | [![Screenshot that shows how to launch Cloud Shell in a new window.](media/hdi-launch-cloud-shell.png)](https://shell.azure.com) |
| Select the **Cloud Shell** button on the menu bar at the upper right in the [Azure portal](https://portal.azure.com). | ![Screenshot that shows the Cloud Shell button in the Azure portal](./media/hdi-cloud-shell-menu.png) |

To use Azure Cloud Shell:

1. Start Cloud Shell.

2. Select the **Copy** button on a code block (or command block) to copy the code or command.

3. Paste the code or command into the Cloud Shell session by selecting **Ctrl**+**Shift**+**V** on Windows and Linux, or by selecting **Cmd**+**Shift**+**V** on macOS.

4. Select **Enter** to run the code or command.

If you decide to install and use Azure CLI locally instead, this quickstart requires you to use Azure CLI version 2.0.28 or later. To find your installed version, run *az --version.*

See [Install Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli) for install or upgrade info.

### Create Resource Group

Before you can create any resource, you have to create a resource group to host the Virtual Network. Create a resource group with [*az group create*](https://learn.microsoft.com/en-us/cli/azure/group?view=azure-cli-latest#az-group-create). This example creates a resource group named *myResourceGroup* in the *westeurope* location:
```azurecli-interactive
az group create --name myResourceGroup --location westeurope
```

### Create Virtual Network

Create a Virtual Network with [*az network vnet create*](https://learn.microsoft.com/en-us/cli/azure/network/vnet?view=azure-cli-latest#az-network-vnet-create). This example creates a default Virtual Network named *myVirtualNetwork* with one subnet named *mySubnet*:
```azurecli
az network vnet create \
--name myVirtualNetwork \
--resource-group myResourceGroup \
--subnet-name mySubnet
```
### Disable subnet private endpoint policies

Azure deploys resources to a subnet within a virtual network, so you need to create or update the subnet to disable private endpoint [network policies](https://learn.microsoft.com/en-us/azure/private-link/disable-private-endpoint-network-policy?tabs=network-policy-portal). Update a subnet configuration named mySubnet with [*az network vnet subnet update*](https://learn.microsoft.com/en-us/cli/azure/network/vnet/subnet?view=azure-cli-latest#az-network-vnet-subnet-update):
```azurecli
az network vnet subnet update \
--name mySubnet \
--resource-group myResourceGroup \
--vnet-name myVirtualNetwork \
--disable-private-endpoint-network-policies true
```

### Create an Azure Database for PostgreSQL - Flexible server

Create a Azure Database for PostgreSQL with the [az postgres flexible-server create](https://learn.microsoft.com/en-us/cli/azure/postgres/flexible-server?view=azure-cli-latest#az-postgres-flexible-server-create) command. Remember that the name of your PostgreSQL Server must be unique across Azure, so replace the placeholder value with your own unique values that you used above:
```azurecli
az postgres flexible-server create \
--name mydemoserver \
--resource-group myresourcegroup \
--location westeurope \
--admin-user mylogin \
--admin-password <server_admin_password> \
--sku-name Standard_B1ms \
--tier Burstable \
--version 13
```

### Create the Private Endpoint

Create a private endpoint for the PostgreSQL Flexible server in your Virtual Network:
```azurecli
az network private-endpoint create \  
    --name myPrivateEndpoint \  
    --resource-group myResourceGroup \  
    --vnet-name myVirtualNetwork  \  
    --subnet mySubnet \  
    --storage-size 128 \   
    --group-id postgresqlServer \  
    --high-availability Enabled \
    --zone 1 \
    --standby-zone 3
```

### Configure the Private DNS Zone

Create a Private DNS Zone for PostgreSQL server domain and create an association link with the Virtual Network. 
```azurecli
az network private-dns zone create --resource-group myResourceGroup \ 
   --name  "privatelink.postgres.database.azure.com" 

az network private-dns link vnet create --resource-group myResourceGroup \ 
   --zone-name  "privatelink.postgres.database.azure.com"\ 
   --name MyDNSLink \ 
   --virtual-network myVirtualNetwork \ 
   --registration-enabled false

#Query for the network interface ID  

networkInterfaceId=$(az network private-endpoint show --name myPrivateEndpoint --resource-group myResourceGroup --query 'networkInterfaces[0].id' -o tsv)


az resource show --ids $networkInterfaceId --api-version 2019-04-01 -o json
# Copy the content for privateIPAddress and FQDN matching the Azure database for PostgreSQL name

#Create DNS records 

az network private-dns record-set a create --name myserver --zone-name privatelink.postgres.database.azure.com --resource-group myResourceGroup  
az network private-dns record-set a add-record --record-set-name myserver --zone-name privatelink.postgres.database.azure.com --resource-group myResourceGroup -a <Private IP Address>
```

> [!NOTE]
> The FQDN in the customer DNS setting does not resolve to the private IP configured. You will have to setup a DNS zone for the configured FQDN as shown [here](https://learn.microsoft.com/en-us/azure/dns/dns-operations-recordsets-portal).

> [!NOTE]
> In some cases the Azure Database for PostgreSQL and the VNet-subnet are in different subscriptions. In these cases you must ensure the following configurations:
* Make sure that both the subscription has the Microsoft.DBforPostgreSQL resource provider registered. For more information, refer to > resource providers.

### Create the VM Client

Create a VM with [az vm create](https://learn.microsoft.com/cli/azure/vm?view=azure-cli-latest#az-vm-create). When prompted, provide a password to be used as the sign-in credentials for the VM. This example creates a VM named *myVm*:
```azurecli
az vm create \
  --resource-group myResourceGroup \
  --name myVm \
  --image Win2019Datacenter
```

### Connect to VM from Internet
