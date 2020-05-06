# Private ARO cluster setup
The goal is to secure ARO cluster by routing Egress traffic through an Azure Firewall
## Before:
![Before](images/aroprivate.png)
## After:
![After](images/arofw.png)
## Create network for an ARO cluster

### Update/install the aro extension
```bash
az extension add -n aro --index https://az.aroapp.io/stable
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
  --address-prefixes 10.0.0.0/8
```

### Add two empty subnets to your virtual network
```bash
  az network vnet subnet create \
    -g "$RESOURCEGROUP" \
    --vnet-name vnet \
    -n "$CLUSTER-master" \
    --address-prefixes 10.10.1.0/24 \
    --service-endpoints Microsoft.ContainerRegistry

  az network vnet subnet create \
    -g "$RESOURCEGROUP" \
    --vnet-name vnet \
    -n "$CLUSTER-worker" \
    --address-prefixes 10.20.1.0/24 \
    --service-endpoints Microsoft.ContainerRegistry
```

### Disable network policies for Private Link Service on your virtual network and subnets. This is a requirement for the ARO service to access and manage the cluster.
```bash
az network vnet subnet update \
  -g "$RESOURCEGROUP" \
  --vnet-name vnet \
  -n "$CLUSTER-master" \
  --disable-private-link-service-network-policies true
```
### Create a Firewall Subnet
```bash
az network vnet subnet create \
    -g "$RESOURCEGROUP" \
    --vnet-name vnet \
    -n "AzureFirewallSubnet" \
    --address-prefixes 10.100.1.0/26
```

## Create a jump-host VM
### Create a jump-subnet
```bash
  az network vnet subnet create \
    -g "$RESOURCEGROUP" \
    --vnet-name vnet \
    -n "jumphost" \
    --address-prefixes 10.30.1.0/24 \
    --service-endpoints Microsoft.ContainerRegistry
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

## Create an Azure Red Hat OpenShift cluster
Get a `pull-secret` value from Red Hat Customer Portal and store it in a variables:
```bash
PULL_SECRET='<your_pull_secret>'
```
```bash
az aro create \
  -g "$RESOURCEGROUP" \
  -n "$CLUSTER" \
  --vnet vnet \
  --master-subnet "$CLUSTER-master" \
  --worker-subnet "$CLUSTER-worker" \
  --apiserver-visibility Private \
  --ingress-visibility Private \
  --pull-secret $PULL_SECRET
```

## Create an Azure Firewall
### Create a public IP Address
```bash
az network public-ip create -g $RESOURCEGROUP -n fw-ip --sku "Standard" --location $LOCATION
```
### Update install Azure Firewall extension
```bash
az extension add -n azure-firewall
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

az network route-table route create -g $RESOURCEGROUP --name aro-udr --route-table-name aro-udr --address-prefix 0.0.0.0/0 --next-hop-type VirtualAppliance --next-hop-ip-address $FWPRIVATE_IP
```

### Add Application Rules for Azure Firewall
Rule for OpenShift to work based on this [list](https://docs.openshift.com/container-platform/4.3/installing/install_config/configuring-firewall.html#configuring-firewall_configuring-firewall):
```bash
az network firewall application-rule create -g $RESOURCEGROUP -f aro-private \
 --collection-name 'OpenShift' \
 --action allow \
 --priority 100 \
 -n 'required' \
 --source-addresses '*' \
 --protocols 'http=80' 'https=443' \
 --target-fqdns 'registry.redhat.io' '*.quay.io' 'sso.redhat.com' 'management.azure.com' 'mirror.openshift.com' 'api.openshift.com' 'quay.io' '*.blob.core.windows.net' 'gcs.prod.monitoring.core.windows.net' 'registry.access.redhat.com' 'login.microsoftonline.com' '*.servicebus.windows.net' '*.table.core.windows.net' 'grafana.com'
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
 --target-fqdns '*cloudflare.docker.com' '*registry-1.docker.io' 'apt.dockerproject.org' 'auth.docker.io'
```

### Associate ARO Subnets to FW
```bash
az network vnet subnet update -g $RESOURCEGROUP --vnet-name vnet --name "$CLUSTER-master" --route-table aro-udr
az network vnet subnet update -g $RESOURCEGROUP --vnet-name vnet --name "$CLUSTER-worker" --route-table aro-udr
```

## Test the configuration
These steps works only if you added rules for Docker images. 
### Configure the jumpbox
Log into a jumpbox VM and install `azure-cli`, `oc-cli`, and `jq` utils. For the installation of openshift-cli check the Red Hat customer portal.
```bash
#Install Azure-cli
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
#Install jq
sudo apt install jq -y
#Install aro extension
az extension add -n aro --index https://az.aroapp.io/preview
```
### Log into the ARO cluster
List cluster credentials:
```bash
# Initialize variables
LOCATION=eastus
RESOURCEGROUP=aro-v4-private
CLUSTER=aroprivate

# Login to Azure
az login
#Get the cluster credentials
ARO_PASSWORD=$(az aro list-credentials -n $CLUSTER -g $RESOURCEGROUP | jq -r '.kubeadminPassword')
ARO_USERNAME=$(az aro list-credentials -n $CLUSTER -g $RESOURCEGROUP | jq -r '.kubeadminUsername')
```
Get an API server endpoint:
```bash
ARO_URL=$(az aro show -n $CLUSTER -g $RESOURCEGROUP -o json | jq -r '.apiserverProfile.url')
```
Log in using `oc login`:
```bash
oc login $ARO_URL -u $ARO_USERNAME -p $ARO_PASSWORD
```

### Run CentOS to test outside connectivity
Create a pod
```bash
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: centos
spec:
  containers:
  - name: centos
    image: centos
    ports:
    - containerPort: 80
    command:
    - sleep
    - "3600"
EOF
```
Once the Pod is Running exec into it and test outside connectivity
```bash
oc exec -it centos -- /bin/bash
curl microsoft.com
```
### Run a test app and expose it internally via Route
Create a welcome-app deployment:
```bash
oc apply -f https://raw.githubusercontent.com/akamenev/aro-private/master/welcome-app-deployment.yaml
```
Create a welcome-app service with a ClusterIP:
```bash
oc apply -f https://raw.githubusercontent.com/akamenev/aro-private/master/welcome-app-clusterip-svc.yaml
```
Expose a welcome-app service via route:
```bash
oc expose service welcome-app
```
Check the service
```bash
APP_ROUTE=$(oc get route welcome-app -o json | jq -r .spec.host)
curl $APP_ROUTE
```

### Expose the app externally with an Internal Load Balancer and Azure Firewall
Create a welcome-app service with an Internal LB:
```bash
oc apply -f https://raw.githubusercontent.com/akamenev/aro-private/master/welcome-app-internallb-svc.yaml
```
Get the internal IP of a service:
```bash
INTERNAL_APP_IP=$(oc get svc welcome-app-internal -o json | jq -r '.status.loadBalancer.ingress[0].ip')
```
Configure a DNAT rule in Azure Firewall:
```bash
# Get a Firewall's public ip
FWPUBLIC_IP=$(az network public-ip show -g $RESOURCEGROUP -n fw-ip --query "ipAddress" -o tsv)
echo $FWPUBLIC_IP

# Add a firewall extension
az extension add -n azure-firewall

# Configure a rule
az network firewall nat-rule create -g $RESOURCEGROUP -f aro-private \
 --collection-name 'welcome-app-nat' \
 --priority 100 \
 --action 'Dnat' \
 -n 'welcome-app' \
 --source-addresses '*' \
 --destination-address $FWPUBLIC_IP \
 --destination-ports '80' \
 --translated-address $INTERNAL_APP_IP \
 --translated-port '80' \
 --protocols 'TCP'
```
Open Firewall's public IP in a browser or curl it:
```bash
curl $FWPUBLIC_IP
```