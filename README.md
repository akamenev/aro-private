# Private ARO cluster setup

## Create network for an ARO cluster

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
### Create a Firewall Subnet
```bash
az network vnet subnet create \
    -g "$RESOURCEGROUP" \
    --vnet-name vnet \
    -n "AzureFirewallSubnet" \
    --address-prefixes 10.100.1.0/26 \
    >/dev/null
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
### Create a jump-host VM
```bash
VMUSERNAME=aroadmin

az vm create --name ubuntu-jump \
             --resource-group $RESOURCEGROUP \
             --ssh-key-values ~/.ssh/id_rsa.pub \
             --admin-username $VMUSERNAME \
             --image UbuntuLTS \
             --subnet "jumphost" \
             --public-ip-address jumphost-ip \
             --vnet-name vnet 
```
## Create an Azure Firewall
### Create a public IP Address
```bash
az network public-ip create -g $RESOURCEGROUP -n fw-ip --sku "Standard" --location $LOCATION
```
### Update install Azure Firewall extension
```bash
az extension install -n azure-firewall
az extension update -n azure-firewall
```

### Create Azure Firewall and configure IP Config
```bash
az network firewall create -g $RESOURCEGROUP -n aro-private -l $LOCATION
az network firewall ip-config create -g $RESOURCEGROUP -f aro-private -n fw-config --public-ip-address fw-ip --vnet-name vnet

```

### Capture Azure Firewall IPs for a later use
```bash
FWPUBLIC_IP=$(az network public-ip show -g $RESOURCEGROUP -n fw-ip --query "ipAddress" -o tsv)
FWPRIVATE_IP=$(az network firewall show -g $RESOURCEGROUP -n aro-private --query "ipConfigurations[0].privateIpAddress" -o tsv)

echo $FWPUBLIC_IP
echo $FWPRIVATE_IP
```

### Create a UDR and Routing Table for Azure Firewall
```bash
az network route-table create -g $RESOURCEGROUP --name aro-udr

az network route-table route create -g $RESOURCEGROUP --name aru-udr --route-table-name aro-udr --address-prefix 0.0.0.0/0 --next-hop-type VirtualAppliance --next-hop-ip-address $FWPRIVATE_IP
```

### Add Application Rules for Azure Firewall
Rule for OpenShift to work based on this list:
```bash
az network firewall application-rule create -g $RESOURCEGROUP -f aro-private \
 --collection-name 'OpenShift' \
 --action allow \
 --priority 100 \
 -n 'required' \
 --source-addresses '*' \
 --protocols 'http=80' 'https=443' \
 --target-fqdns 'registry.redhat.io' '*.quay.io' 'sso.redhat.com' 'cert-api.access.redhat.com' 'api.access.redhat.com' 'infogw.api.openshift.com' 'cloud.redhat.com' 'management.azure.com' 'mirror.openshift.com' '*.cloudfront.net' '*.apps.9cj4h6wn.eastus.aroapp.io' 'quay-registry.s3.amazonaws.com' 'api.openshift.com' 'art-rhcos-ci.s3.amazonaws.com'
```
Optional rules for Docker images:
```bash
az network firewall application-rule create -g $RESOURCEGROUP -f aro-private \
 --collection-name 'Docker' \
 --action allow \
 --priority 200 \
 -n 'docker' \
 --source-addresses '*' \
 --protocols 'http=80' 'https=443' \
 --target-fqdns '*cloudflare.docker.com' '*registry-1.docker.io' 'apt.dockerproject.org'
```

### Associate ARO Subnets to FW
```bash
az network vnet subnet update -g $RESOURCEGROUP --vnet-name vnet --name "$CLUSTER-master" --route-table aro-udr
az network vnet subnet update -g $RESOURCEGROUP --vnet-name vnet --name "$CLUSTER-worker" --route-table aro-udr
```

## Create an Azure Red Hat OpenShift cluster
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