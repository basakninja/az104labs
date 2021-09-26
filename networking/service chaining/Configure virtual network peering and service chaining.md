# 1. Create a virtual network and subnets
#### 1. Create resource group that will be contain hub-spoke vnets.
```bash
az group create \
--name $RESOURCE_GROUP_VNETS \
--location eastus
```
#### 2. In Azure CLI, run the foolowing command to create hub virtual network and hub subnet.
```bash
az network vnet create \
--name $VNET_HUB \
--resource-group $RESOURCE_GROUP_VNETS \
--address-prefixes 10.0.0.0/16 \
--subnet-name $SNET_0_VNET_HUB_0 \
--subnet-prefixes 10.0.0.0/24 \
--location $HUB_LOCATION
```
#### 3. In Azure CLI, run the foolowing command to create spoke-1 virtual network and spoke-1 subnet.
```bash
az network vnet create \
--name $VNET_SPOKE_1 \
--resource-group $RESOURCE_GROUP_VNETS \
--address-prefixes 10.1.0.0/16 \
--subnet-name $SNET_0_VNET_SPOKE_1 \
--subnet-prefixes 10.1.0.0/24 \
--location $SPOKE_1_LOCATION
```
#### 4. In Azure CLI, run the foolowing command to create spoke-2 virtual network and spoke-2 subnet.
```bash
az network vnet create \
--name $VNET_SPOKE_2 \
--resource-group $RESOURCE_GROUP_VNETS \
--address-prefixes 10.2.0.0/16 \
--subnet-name $SNET_0_VNET_SPOKE_2 \
--subnet-prefixes 10.2.0.0/24 \
--location $SPOKE_2_LOCATION
```
#### 5. Run the following command to get IDs of the subnets.
```bash
SNET_0_VNET_HUB_0_ID="$(az network vnet subnet show --resource-group $RESOURCE_GROUP_VNETS --vnet-name $VNET_HUB --name $SNET_0_VNET_HUB_0 --query id --output tsv)"
SNET_0_VNET_SPOKE_1_ID="$(az network vnet subnet show --resource-group $RESOURCE_GROUP_VNETS --vnet-name $VNET_SPOKE_1 --name $SNET_0_VNET_SPOKE_1 --query id --output tsv)"
SNET_0_VNET_SPOKE_2_ID="$(az network vnet subnet show --resource-group $RESOURCE_GROUP_VNETS --vnet-name $VNET_SPOKE_2 --name $SNET_0_VNET_SPOKE_2 --query id --output tsv)"
```
# 2. Create peers, route tables and custom routes.
#### 1. Create peer two virtual networks.
> (to successfully peer two virtual networks this command must be called twice with the values for --vnet-name and --remote-vnet reversed): 
```bash```
#### 1. Run the following command to create peer VNET_HUB and VNET_SPOKE_1.
```bash
az network vnet peering create \
--name $PEER_HUB_SPOKE_1 \
--remote-vnet $VNET_SPOKE_1 \
--resource-group $RESOURCE_GROUP_VNETS \
--vnet-name $VNET_HUB \
--allow-forwarded-traffic \
--allow-gateway-transit \
--allow-vnet-access

az network vnet peering create \
--name $PEER_SPOKE_1_HUB \
--remote-vnet $VNET_HUB \
--resource-group $RESOURCE_GROUP_VNETS \
--vnet-name $VNET_SPOKE_1 \
--allow-forwarded-traffic \
--allow-gateway-transit \
--allow-vnet-access
```
#### 2. Run the following command to create peer VNET_HUB and VNET_SPOKE_2.
```bash
az network vnet peering create \
--name $PEER_HUB_SPOKE_2 \
--remote-vnet $VNET_SPOKE_2 \
--resource-group $RESOURCE_GROUP_VNETS \
--vnet-name $VNET_HUB \
--allow-forwarded-traffic \
--allow-gateway-transit \
--allow-vnet-access

az network vnet peering create \
--name $PEER_SPOKE_2_HUB \
--remote-vnet $VNET_HUB \
--resource-group $RESOURCE_GROUP_VNETS \
--vnet-name $VNET_SPOKE_2 \
--allow-forwarded-traffic \
--allow-gateway-transit \
--allow-vnet-access
```
#### 3. create route table
```bash
az network route-table create \
--name $ROUTE_TABLE_SPOKE_1 \
--resource-group $RESOURCE_GROUP_VNETS \
--disable-bgp-route-propagation false \
--location $SPOKE_1_LOCATION
```
#### 4. add custom route
```bash
az network route-table route create \
--route-table-name $ROUTE_TABLE_SPOKE_1 \
--resource-group $RESOURCE_GROUP_VNETS \
--name udr-to-spoke-2 \
--address-prefix 10.2.0.0/24 \
--next-hop-type VirtualAppliance \
--next-hop-ip-address 10.0.0.4
```
#### 5. create route table
```bash
az network route-table create \
--name $ROUTE_TABLE_SPOKE_2 \
--resource-group $RESOURCE_GROUP_VNETS \
--disable-bgp-route-propagation false \
--location $SPOKE_2_LOCATION
```
#### 6. add custom route
```bash
az network route-table route create \
--route-table-name $ROUTE_TABLE_SPOKE_2 \
--resource-group $RESOURCE_GROUP_VNETS \
--name udr-to-spoke-1 \
--address-prefix 10.1.0.0/24 \
--next-hop-type VirtualAppliance \
--next-hop-ip-address 10.0.0.4
```
#### 7.  Associate route table with the subnet.
```bash
az network vnet subnet update \
--ids $SNET_0_VNET_SPOKE_1_ID \
--route-table $ROUTE_TABLE_SPOKE_1
```
#### 8. Associate route table with the subnet.
```bash
az network vnet subnet update \
--ids $SNET_0_VNET_SPOKE_2_ID \
--route-table $ROUTE_TABLE_SPOKE_2
```

# 3. Create an NVA
>A network virtual appliance (NVA) is a virtual appliance primarily focused on network functions virtualization. A typical network virtual appliance involves various layers four to seven functions like firewall, WAN optimizer, application delivery controllers, routers, load balancers, IDS/IPS, proxies, SD-WAN edge, and more.
#### 1. Create resource group for virtual networks 
```bash
az group create \
--name $RESOURCE_GROUP_VMS \
--location eastus
```
#### 2. Run the following command to deploy a network virtual appliance (NVA) to the hub network 
```bash
az vm create \
--name $VM_NVA_NAME \
--resource-group $RESOURCE_GROUP_VMS \
--image $IMAGE \
--admin-username $USER \
--admin-password $PASSWORD \
--location $HUB_LOCATION \
--subnet $SNET_0_VNET_HUB_0_ID
```
## Enable IP forwarding for the Azure network interface

#### 1. Run the following command to get the ID of the NVA network interface.
```bash
VM_NVA_NIC_ID="$(az vm show \
--name $VM_NVA_NAME \
--resource-group $RESOURCE_GROUP_VMS \
--query networkProfile.networkInterfaces[].id \
--output tsv)"
```
#### 2. Run the following command to enable IP forwarding for the network interface.
```bash
az network nic update \
--ids $VM_NVA_NIC_ID \
--ip-forwarding true
```

## Enable IP forwarding in the appliance

#### 1. Run the following command to get public IP address of the NVA virtual machine
```bash
VM_NVA_PUBLIC_IP="$(az vm list-ip-addresses \
--resource-group $RESOURCE_GROUP_VMS \
--name $VM_NVA_NAME \
--query "[].virtualMachine.network.publicIpAddresses[*].ipAddress" \
--output tsv)"

echo $VM_NVA_PUBLIC_IP
```
#### 2. Run the following command to enable IP forwarding within the NVA
```bash
ssh -t -o StrictHostKeyChecking=no $USER@$VM_NVA_PUBLIC_IP \
'sudo sysctl -w net.ipv4.ip_forward=1; exit;'
```

# 4. Deploy virtual machines in different spokes virtual networks
#### 1. Run the following command to deploy a Linux VM to the first spoke network **`vnet-spoke-001`**.  
```bash
az vm create \
--name $VM_1_NAME \
--resource-group $RESOURCE_GROUP_VMS \
--image $IMAGE \
--admin-username $USER \
--admin-password $PASSWORD \
--location $SPOKE_1_LOCATION \
--subnet $SNET_0_VNET_SPOKE_1_ID
```
#### 1. Run the following command to deploy a Linux VM to the second spoke network **`vnet-spoke-002`**.  
```bash
az vm create \
--name $VM_2_NAME \
--resource-group $RESOURCE_GROUP_VMS \
--image $IMAGE \
--admin-username $USER \
--admin-password $PASSWORD \
--location $SPOKE_2_LOCATION \
--subnet $SNET_0_VNET_SPOKE_2_ID
```
# 5. Check connectivity
#### 1. Run the following command to get public IP address of the VM_1.
```bash
VM_1_NAME_PUBLIC_IP="$(az vm list-ip-addresses \
--resource-group $RESOURCE_GROUP_VMS \
--name $VM_1_NAME \
--query "[].virtualMachine.network.publicIpAddresses[*].ipAddress" \
--output tsv)"
```
#### 2. Run the following command to get public IP address of the VM_2.
```bash
VM_2_NAME_PUBLIC_IP="$(az vm list-ip-addresses \
--resource-group $RESOURCE_GROUP_VMS \
--name $VM_2_NAME \
--query "[].virtualMachine.network.publicIpAddresses[*].ipAddress" \
--output tsv)"
```
#### 3. Run the following command to test connectivity.
```bash
ssh -t -o StrictHostKeyChecking=no $USER@$VM_1_NAME_PUBLIC_IP \
'ping 10.2.0.4 -c 5'
```
```bash
ssh -t -o StrictHostKeyChecking=no $USER@$VM_2_NAME_PUBLIC_IP \
'ping 10.1.0.4 -c 5'
```

# Parameters
```bash
RESOURCE_GROUP_VNETS=rg-vnets-eastus
RESOURCE_GROUP_VMS=rg-vms-eastus
VNET_HUB=vnet-hub
VNET_SPOKE_1=vnet-spoke-001
VNET_SPOKE_2=vnet-spoke-002
SNET_0_VNET_HUB_0=snet-0-vnet-hub
SNET_0_VNET_SPOKE_1=snet-0-vnet-spoke-1
SNET_0_VNET_SPOKE_2=snet-0-vnet-spoke-2
HUB_LOCATION=eastus
SPOKE_1_LOCATION=westus
SPOKE_2_LOCATION=centralus
PEER_HUB_SPOKE_1=peer-hub-spoke-001
PEER_SPOKE_1_HUB=peer-spoke-001-hub
PEER_HUB_SPOKE_2=peer-hub-spoke-002
PEER_SPOKE_2_HUB=peer-spoke-002-hub
ROUTE_TABLE_SPOKE_1=rt-spoke-1
ROUTE_TABLE_SPOKE_2=rt-spoke-2
VM_NVA_NAME=vm-nva
VM_1_NAME=vm-spoke-001
VM_2_NAME=vm-spoke-002
IMAGE=UbuntuLTS
USER=azureuser
PASSWORD=azureuser123!
```