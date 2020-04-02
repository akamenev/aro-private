# Private ARO cluster setup

## Create an Azure Red Hat OpenShift 4.3 Cluster

### Update/install the aro extension
```bash
az extension add -n aro --index https://az.aroapp.io/preview
az extension update -n aro
```

### Set the variables
```bash
LOCATION=eastus
RESOURCEGROUP=aro-v4-private
CLUSTER=aroprivate
```

### Create a resource group
```bash
az group create -g "$RESOURCEGROUP" -l $LOCATION
```

### Create the virtual network
```bash
az network vnet create \
  -g "$RESOURCEGROUP" \
  -n vnet \
  --address-prefixes 10.0.0.0/8 \
  >/dev/null
```

### Add two empty subnets to your virtual network
```bash
  az network vnet subnet create \
    -g "$RESOURCEGROUP" \
    --vnet-name vnet \
    -n "$CLUSTER-master" \
    --address-prefixes 10.10.1.0/24 \
    --service-endpoints Microsoft.ContainerRegistry \
    >/dev/null

  az network vnet subnet create \
    -g "$RESOURCEGROUP" \
    --vnet-name vnet \
    -n "$CLUSTER-worker" \
    --address-prefixes 10.20.1.0/24 \
    --service-endpoints Microsoft.ContainerRegistry \
    >/dev/null
```

### Disable network policies for Private Link Service on your virtual network and subnets. This is a requirement for the ARO service to access and manage the cluster.
```bash
az network vnet subnet update \
  -g "$RESOURCEGROUP" \
  --vnet-name vnet \
  -n "$CLUSTER-master" \
  --disable-private-link-service-network-policies true \
  >/dev/null
```

### Create a cluster
```bash
az aro create \
  -g "$RESOURCEGROUP" \
  -n "$CLUSTER" \
  --vnet vnet \
  --master-subnet "$CLUSTER-master" \
  --worker-subnet "$CLUSTER-worker" \
  --apiserver-visibility Private \
  --ingress-visibility Private
```
## Create a jump-host VM
### Create a jump-subnet
```bash
  az network vnet subnet create \
    -g "$RESOURCEGROUP" \
    --vnet-name vnet \
    -n "jumphost" \
    --address-prefixes 10.30.1.0/24 \
    --service-endpoints Microsoft.ContainerRegistry \
    >/dev/null
```
### Create a jump-host VM with a Windows 10
```bash
VMPASSWORD=Supersecurepassword198$
VMADMIN=aroadmin
az vm create --name windows-jump \ 
             --resource-group $RESOURCEGROUP \ 
             --admin-password $VMPASSWORD \ 
             --admin-username $VMADMIN \ 
             --image MicrosoftWindowsDesktop:Windows-10:19h1-pro:18362.720.2003120536 \ 
             --subnet "jumphost" \ 
             --public-ip-address jump-ip \ 
             --vnet-name vnet 
```

