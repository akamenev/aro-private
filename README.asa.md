LOCATION=japaneast
RESOURCEGROUP=aro-v4-private
CLUSTER=aroprivate

az group create -g "$RESOURCEGROUP" -l $LOCATION

az network vnet create \
  -g "$RESOURCEGROUP" \
  -n vnet \
  --address-prefixes 10.0.0.0/8

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

az network vnet subnet update \
  -g "$RESOURCEGROUP" \
  --vnet-name vnet \
  -n "$CLUSTER-master" \
  --disable-private-link-service-network-policies true

az network vnet subnet create \
    -g "$RESOURCEGROUP" \
    --vnet-name vnet \
    -n "AzureFirewallSubnet" \
    --address-prefixes 10.100.1.0/26


az network vnet subnet create \
    -g "$RESOURCEGROUP" \
    --vnet-name vnet \
    -n "jumphost" \
    --address-prefixes 10.30.1.0/24 \
    --service-endpoints Microsoft.ContainerRegistry


VMUSERNAME=aroadmin

az vm create --name ubuntu-jump \
             --resource-group $RESOURCEGROUP \
             --ssh-key-values ~/.ssh/id_rsa.pub \
             --admin-username $VMUSERNAME \
             --image UbuntuLTS \
             --subnet "jumphost" \
             --public-ip-address jumphost-ip \
             --vnet-name vnet



PULL_SECRET="{"auths":{"cloud.openshift.com":{"auth":"b3BlbnNoaWZ0LXJlbGVhc2UtZGV2K2FzYW1zMXJmZm1mNnF2ZHN3bm9rdTZnbGZsbHZraHhlOkg0MFVLR01IU005VDdITkdCVUlaRTMwQzZEMEVaMlBCUk40UkVPQjBBT1I5RlFDVjY2OTJCNlRXSTFGVEVVMTk=","email":"shiasa@microsoft.com"},"quay.io":{"auth":"b3BlbnNoaWZ0LXJlbGVhc2UtZGV2K2FzYW1zMXJmZm1mNnF2ZHN3bm9rdTZnbGZsbHZraHhlOkg0MFVLR01IU005VDdITkdCVUlaRTMwQzZEMEVaMlBCUk40UkVPQjBBT1I5RlFDVjY2OTJCNlRXSTFGVEVVMTk=","email":"shiasa@microsoft.com"},"registry.connect.redhat.com":{"auth":"NTI1MjcwMTZ8dWhjLTFSZmZNRjZxdmRTd25va1U2Z2xGbGxWS0hYZTpleUpoYkdjaU9pSlNVelV4TWlKOS5leUp6ZFdJaU9pSTVZamN4WVRGak56a3pOakkwT0dWaE9XSmpOamsyWWpGall6RXhOV1ZqWXlKOS51a28wU3FGTEVIQ1dIWkZZaDRtRzY4enp5bFQzTzFCc0JCTW9KSjlUWld1SDF5X09iVGNQT29leVd3MHVRX0oya3RuWTNHeHZSTmhjbGlUX2JYSEItbHF6bDVEdXJEZlR3SFFYUDFYQi0ta2ZkcnFwQkhKSFJEN09NY3BvWDZMNzVZYUNjWkVrbVZzR0VxcWxvd19EMmN6TGM5SUVFVHBUOTRldDViSHV5RUhpa3lmN25wbVVoT2RWUnVjNEVRMWttX0ZxRzVTV2lnTXZIcTVFZDQzb0gybUZMTjZydjdOalI5YUpZOWI5MHFmQzN2Q0hRMmE0ZG9ESmxMOW1qdl91UHpvSE1BMEF5bnl4bjNTaEp2N2N1b1d5RkxaSVpsMloxTEh4ejNrZm9LX05MMTZxWWRpWU5fLW9DQXpFMUxMZ2xWaEVyQ2JGbHhkTFZKaHE2TjRVZmJkamo1QmFEWjNldGFtS0M3MUhZNW9QX0ljTVFYb3ViTFhjZFRySXZmVlNMZC1KR2hJSzcwcUFHMlV4T0owZE13STdYdjhBVktidjJSV3pCRGlyRTU2cGVEVVphQVFXNmZkMkhwNGRKQkNOZmU1YW9kZzQ4MndVOEh5TUdCZjhqRlY1LWswWEtUR1RSSVdmd2lSSzFqX3IwdWpYVzF4ZEdfSXg1MXVReG5iNk82QjhnX1JfNi0zZVZKWVc3WW1MTWR6NURpZFRrY3pCVnZ2eVlrRFMyRTlodUg5UzlNYzNiLU1XWmZqbVVGa0d1QW0tM255ck91Y2lnbVlUVWQ5eWs4NXVKbDJDQmNGaGlhaHRGdENOaWlsYnRLWUFlZVhvVFc2aGNxbWpFMHBBNDREWHRQQkVHOEVFUFg3RUxVTmhNb0RiTGpPS2FlbHp3aUVsSWxiRnUtZw==","email":"shiasa@microsoft.com"},"registry.redhat.io":{"auth":"NTI1MjcwMTZ8dWhjLTFSZmZNRjZxdmRTd25va1U2Z2xGbGxWS0hYZTpleUpoYkdjaU9pSlNVelV4TWlKOS5leUp6ZFdJaU9pSTVZamN4WVRGak56a3pOakkwT0dWaE9XSmpOamsyWWpGall6RXhOV1ZqWXlKOS51a28wU3FGTEVIQ1dIWkZZaDRtRzY4enp5bFQzTzFCc0JCTW9KSjlUWld1SDF5X09iVGNQT29leVd3MHVRX0oya3RuWTNHeHZSTmhjbGlUX2JYSEItbHF6bDVEdXJEZlR3SFFYUDFYQi0ta2ZkcnFwQkhKSFJEN09NY3BvWDZMNzVZYUNjWkVrbVZzR0VxcWxvd19EMmN6TGM5SUVFVHBUOTRldDViSHV5RUhpa3lmN25wbVVoT2RWUnVjNEVRMWttX0ZxRzVTV2lnTXZIcTVFZDQzb0gybUZMTjZydjdOalI5YUpZOWI5MHFmQzN2Q0hRMmE0ZG9ESmxMOW1qdl91UHpvSE1BMEF5bnl4bjNTaEp2N2N1b1d5RkxaSVpsMloxTEh4ejNrZm9LX05MMTZxWWRpWU5fLW9DQXpFMUxMZ2xWaEVyQ2JGbHhkTFZKaHE2TjRVZmJkamo1QmFEWjNldGFtS0M3MUhZNW9QX0ljTVFYb3ViTFhjZFRySXZmVlNMZC1KR2hJSzcwcUFHMlV4T0owZE13STdYdjhBVktidjJSV3pCRGlyRTU2cGVEVVphQVFXNmZkMkhwNGRKQkNOZmU1YW9kZzQ4MndVOEh5TUdCZjhqRlY1LWswWEtUR1RSSVdmd2lSSzFqX3IwdWpYVzF4ZEdfSXg1MXVReG5iNk82QjhnX1JfNi0zZVZKWVc3WW1MTWR6NURpZFRrY3pCVnZ2eVlrRFMyRTlodUg5UzlNYzNiLU1XWmZqbVVGa0d1QW0tM255ck91Y2lnbVlUVWQ5eWs4NXVKbDJDQmNGaGlhaHRGdENOaWlsYnRLWUFlZVhvVFc2aGNxbWpFMHBBNDREWHRQQkVHOEVFUFg3RUxVTmhNb0RiTGpPS2FlbHp3aUVsSWxiRnUtZw==","email":"shiasa@microsoft.com"}}}"


az aro create \
  -g "$RESOURCEGROUP" \
  -n "$CLUSTER" \
  --vnet vnet \
  --master-subnet "$CLUSTER-master" \
  --worker-subnet "$CLUSTER-worker" \
  --apiserver-visibility Private \
  --ingress-visibility Private 

az network public-ip create -g $RESOURCEGROUP -n fw-ip --sku "Standard" --location $LOCATION

az extension add -n azure-firewall
az extension update -n azure-firewall

az network firewall create -g $RESOURCEGROUP -n aro-private -l $LOCATION
az network firewall ip-config create -g $RESOURCEGROUP -f aro-private -n fw-config --public-ip-address fw-ip --vnet-name vnet

FWPUBLIC_IP=$(az network public-ip show -g $RESOURCEGROUP -n fw-ip --query "ipAddress" -o tsv)
FWPRIVATE_IP=$(az network firewall show -g $RESOURCEGROUP -n aro-private --query "ipConfigurations[0].privateIpAddress" -o tsv)

echo $FWPUBLIC_IP
echo $FWPRIVATE_IP

az network route-table create -g $RESOURCEGROUP --name aro-udr

az network route-table route create -g $RESOURCEGROUP --name aro-udr --route-table-name aro-udr --address-prefix 0.0.0.0/0 --next-hop-type VirtualAppliance --next-hop-ip-address $FWPRIVATE_IP


az network firewall application-rule create -g $RESOURCEGROUP -f aro-private \
 --collection-name 'OpenShift' \
 --action allow \
 --priority 100 \
 -n 'required' \
 --source-addresses '*' \
 --protocols 'http=80' 'https=443' \
 --target-fqdns 'registry.redhat.io' '*.quay.io' 'sso.redhat.com' 'management.azure.com' 'mirror.openshift.com' 'api.openshift.com' 'quay.io' '*.blob.core.windows.net' 'gcs.prod.monitoring.core.windows.net' 'registry.access.redhat.com' 'login.microsoftonline.com' '*.servicebus.windows.net' '*.table.core.windows.net' 'grafana.com'


az network firewall application-rule create -g $RESOURCEGROUP -f aro-private \
 --collection-name 'Docker' \
 --action allow \
 --priority 200 \
 -n 'docker' \
 --source-addresses '*' \
 --protocols 'http=80' 'https=443' \
 --target-fqdns '*cloudflare.docker.com' '*registry-1.docker.io' 'apt.dockerproject.org' 'auth.docker.io'

az network vnet subnet update -g $RESOURCEGROUP --vnet-name vnet --name "$CLUSTER-master" --route-table aro-udr
az network vnet subnet update -g $RESOURCEGROUP --vnet-name vnet --name "$CLUSTER-worker" --route-table aro-udr


ssh  aroadmin@52.185.161.192

#Install Azure-cli
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
#Install jq
sudo apt install jq -y
#Install aro extension
az extension add -n aro --index https://az.aroapp.io/stable

LOCATION=japaneast
RESOURCEGROUP=aro-v4-private
CLUSTER=aroprivate

# Login to Azure
az login

#Get the cluster credentials
ARO_PASSWORD=$(az aro list-credentials -n $CLUSTER -g $RESOURCEGROUP -o json | jq -r '.kubeadminPassword')
ARO_USERNAME=$(az aro list-credentials -n $CLUSTER -g $RESOURCEGROUP -o json | jq -r '.kubeadminUsername')

ARO_URL=$(az aro show -n $CLUSTER -g $RESOURCEGROUP -o json | jq -r '.apiserverProfile.url')

wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/openshift-client-linux.tar.gz
tar xvf openshift-client-linux.tar.gz
sudo mv oc /usr/local/bin/
sudo mv kubectl /usr/local/bin/


oc login $ARO_URL -u $ARO_USERNAME -p $ARO_PASSWORD



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

oc exec -it centos -- /bin/bash
curl microsoft.com


oc apply -f https://raw.githubusercontent.com/akamenev/aro-private/master/welcome-app-deployment.yaml



APP_ROUTE=$(oc get route welcome-app -o json | jq -r .spec.host)
curl $APP_ROUTE
