<!-- # Private ARO cluster setup -->
# プライベートAROクラスターのセットアップ

この手順は、Azure Firewallを介してEgress トラフィックをルーティングすることでARO クラスターを保護するためのものです。

<!-- The goal is t<!-- TOC -->

- プライベートAROクラスターのセットアップ
    - 構成前
    - 構成後
    - ARO クラスターのネットワーク作成
        - ARO拡張機能を更新/インストール
        - 変数設定
        - リソースグループの作成
        - 仮想ネットワークの作成
        - サブネット作成
        - ネットワークポリシーの変更
        - Firewall サブネットの作成
    - クラスタ管理用VMの作成
        - クラスタ管理用サブネットの作成
        - クラスタ管理用VMのデプロイ
    - Azure Red Hat OpenShiftクラスタの作成
    - Azure Firewallの作成
        - パブリックIPアドレスの作成
        - Azure Firewall拡張機能のインストール
        - Azure Firewallの作成
        - UDRの作成
        - Azureファイアウォールのルール設定
        - Azure FirewallにAROサブネットを関連付け
    - 動作確認
        - クラスタ管理用VMの環境設定
        - AROクラスタへのログイン
        - 外部接続性のテスト
        - サンプルアプリケーションによる稼働確認
        - サービスの公開

<!-- /TOC -->

## 構成前
![Before](images/aroprivate.png)

## 構成後
![After](images/arofw.png)

<!-- ## Create network for an ARO cluster -->
## ARO クラスターのネットワーク作成

<!-- ### Update/install the aro extension -->
### ARO拡張機能を更新/インストール
まず、次のコマンドを実行してARO拡張機能のインストールを行います。

```bash
az extension add -n aro --index https://az.aroapp.io/stable
az extension update -n aro
```

<!-- ### Set the variables -->
### 変数設定

リソースを作成するロケーションやリソースグループ、クラスタ名を変数にセットします。

```bash
LOCATION=japaneast
RESOURCEGROUP=aro-v4-private
CLUSTER=aroprivate
```

<!-- ### Create a resource group -->
### リソースグループの作成

次のコマンドを実行し、リソースグループを作成します。
```bash
az group create -g "$RESOURCEGROUP" -l $LOCATION
```

<!-- ### Create the virtual network -->
### 仮想ネットワークの作成
AROクラスタを構成する仮想ネットワークを作成します。

```bash
az network vnet create \
  -g "$RESOURCEGROUP" \
  -n vnet \
  --address-prefixes 10.0.0.0/8
```

<!-- ### Add two empty subnets to your virtual network -->
### サブネット作成
仮想ネットワークに2つの空のサブネットを追加します。
1つはマスターノードが配置されるサブネットで、もう一つはワーカーノードが配置されるサブネットです。

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

<!-- ### Disable network policies for Private Link Service on your virtual network and subnets. This is a requirement for the ARO service to access and manage the cluster. -->
### ネットワークポリシーの変更
仮想ネットワークとサブネットのプライベートリンクサービスのネットワークポリシーを無効にします。これは、AROサービスがクラスターにアクセスして管理するための要件です。

```bash
az network vnet subnet update \
  -g "$RESOURCEGROUP" \
  --vnet-name vnet \
  -n "$CLUSTER-master" \
  --disable-private-link-service-network-policies true
```

<!-- ### Create a Firewall Subnet -->
### Firewall サブネットの作成

次のコマンドを実行し、Azure Firewallをデプロイするサブネットを作成します。

```bash
az network vnet subnet create \
    -g "$RESOURCEGROUP" \
    --vnet-name vnet \
    -n "AzureFirewallSubnet" \
    --address-prefixes 10.100.1.0/26
```

<!-- ## Create a jump-host VM -->
## クラスタ管理用VMの作成

<!-- ### Create a jump-subnet -->
### クラスタ管理用サブネットの作成

次のコマンドを実行し、クラスタ管理のための仮想マシンをデプロイするサブネットを作成します。

```bash
az network vnet subnet create \
  -g "$RESOURCEGROUP" \
  --vnet-name vnet \
  -n "jumphost" \
  --address-prefixes 10.30.1.0/24 \
  --service-endpoints Microsoft.ContainerRegistry
```

<!-- ### Create a jump-host VM -->
### クラスタ管理用VMのデプロイ

AROクラスタを管理するための仮想マシンを作成します。次の例ではUbuntuのマシンをJumphostサブネットに作成し、このマシン経由でクラスタを操作します。

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

<!-- ## Create an Azure Red Hat OpenShift cluster -->
## Azure Red Hat OpenShiftクラスタの作成

<!-- Get a `pull-secret` value from Red Hat Customer Portal and store it in a variables: -->

Red Hat pull-secretを使用すると、クラスターは追加のコンテンツと共に Red Hatコンテナー レジストリにアクセスできます。 

[Red Hat OpenShift クラスター マネージャー ポータル](https://cloud.redhat.com/openshift/install/azure/aro-provisioned)に移動し、ログインします。
Red Hatアカウントにログインするか、新しいRed Hat アカウントを作成し、使用条件に同意する必要があります。

[Download pull secret]を選択し、ダウンロードします。
このpull-secret.txt ファイルは安全な場所に保管してください。このファイルは、クラスターを作成するたびに使用します。

取得した値は変数`PULL_SECRET`に設定します。

```bash
PULL_SECRET='<your_pull_secret>'
```

次のコマンドを実行し、AROクラスタを作成します。クラスタの作成には約35分かかります。

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

<!-- ## Create an Azure Firewall -->
## Azure Firewallの作成

<!-- ### Create a public IP Address -->
### パブリックIPアドレスの作成

次のコマンド実行し、Azure Firewallに割り当てるパブリックIPアドレスを作成します。

```bash
az network public-ip create \
    -g $RESOURCEGROUP \
    -l $LOCATION \
    -n fw-ip \
    --sku "Standard" 
```
<!-- ### Update install Azure Firewall extension -->
### Azure Firewall拡張機能のインストール

次のコマンドを実行し、Azure Firewal拡張機能をインストールします。
```bash
az extension add -n azure-firewall
az extension update -n azure-firewall
```

<!-- ### Create Azure Firewall and configure IP Config -->
### Azure Firewallの作成

次のコマンドを実行し、Azure Firewallを作成し、IP構成を行います。

```bash
az network firewall create \
    -g $RESOURCEGROUP \
    -n aro-private \
    -l $LOCATION

az network firewall ip-config create \
    -g $RESOURCEGROUP \
    -f aro-private \
    -n fw-config \
    --public-ip-address fw-ip \
    --vnet-name vnet
```

次に、Azure FirewallのパブリックIPアドレスとプライベートIPアドレスを変数に設定します。

```bash
FWPUBLIC_IP=$(az network public-ip show -g $RESOURCEGROUP -n fw-ip --query "ipAddress" -o tsv)
FWPRIVATE_IP=$(az network firewall show -g $RESOURCEGROUP -n aro-private --query "ipConfigurations[0].privateIpAddress" -o tsv)

echo $FWPUBLIC_IP
echo $FWPRIVATE_IP
```

<!-- ### Create a UDR and Routing Table for Azure Firewall -->
### UDRの作成
Azure FirewallのUDRとルーティングテーブルを作成します。

```bash
az network route-table create \
    -g $RESOURCEGROUP \
    -n aro-udr

az network route-table route create \
    -g $RESOURCEGROUP \
    -n aro-udr \
    --route-table-name aro-udr \
    --address-prefix 0.0.0.0/0 \
    --next-hop-type VirtualAppliance \
    --next-hop-ip-address $FWPRIVATE_IP
```

<!-- ### Add Application Rules for Azure Firewall -->
### Azureファイアウォールのルール設定

Azure Firewallのアプリケーションルールを追加します。

+ OpenShift動作させるためのルールの[list](https://docs.openshift.com/container-platform/4.3/installing/install_config/configuring-firewall.html#configuring-firewall_configuring-firewall)はこちらです。
+ ただし、今後変更になる場合があります。

```bash
az network firewall application-rule create \
    -g $RESOURCEGROUP \
    -f aro-private \
    --collection-name 'OpenShift' \
    --action allow \
    --priority 100 \
    -n 'required' \
    --source-addresses '*' \
    --protocols 'http=80' 'https=443' \
    --target-fqdns 'registry.redhat.io' '*.quay.io' 'sso.redhat.com' 'management.azure.com' 'mirror.openshift.com' 'api.openshift.com' 'quay.io' '*.blob.core.windows.net' 'gcs.prod.monitoring.core.windows.net' 'registry.access.redhat.com' 'login.microsoftonline.com' '*.servicebus.windows.net' '*.table.core.windows.net' 'grafana.com'
```

[**オプション**] Dockerを使用するための設定はこちらです。

```bash
az network firewall application-rule create \
    -g $RESOURCEGROUP \
    -f aro-private \
    --collection-name 'Docker' \
    --action allow \
    --priority 200 \
    -n 'docker' \
    --source-addresses '*' \
    --protocols 'http=80' 'https=443' \
    --target-fqdns '*cloudflare.docker.com' '*registry-1.docker.io' 'apt.dockerproject.org' 'auth.docker.io'
```

<!-- ### Associate ARO Subnets to FW -->
### Azure FirewallにAROサブネットを関連付け
次のコマンドを実行し、Azure FirewallにAROサブネットを関連付けます。

```bash
az network vnet subnet update \
    -g $RESOURCEGROUP \
    --vnet-name vnet \
    -n "$CLUSTER-master" \
    --route-table aro-udr

az network vnet subnet update \
    -g $RESOURCEGROUP \
    --vnet-name vnet \
    -n "$CLUSTER-worker" \
    --route-table aro-udr
```

<!-- ## Test the configuration -->
## 動作確認
<!-- These steps works only if you added rules for Docker images.  -->
これらの手順は、Dockerイメージのルールを追加した場合にのみ動作します。

<!-- ### Configure the jumpbox -->
### クラスタ管理用VMの環境設定

<!-- Log into a jumpbox VM and install `azure-cli`, `oc-cli`, and `jq` utils. For the installation of openshift-cli check the Red Hat customer portal. -->

クラスタ管理用VMにログインし、`azure-cli`, `oc-cli`, `jq` utils をインストールします。なお、openshift-cli のインストールについては、Red Hat カスタマーポータルを確認してください。

次のコマンドを実行し、クラスタ管理用VMのIPアドレスを取得します。
```bash
JUMPIP=$(az vm list-ip-addresses --query "[?virtualMachine.name=='ubuntu-jump'].virtualMachine.{PublicIPAddresses:network.publicIpAddresses[0].ipAddress}" -o tsv)
```

ローカルPCからクラスタ管理用VMにSSHで接続します。
```bash
ssh aroadmin@$JUMPIP
Welcome to Ubuntu 18.04.4 LTS (GNU/Linux 5.3.0-1022-azure x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage


To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

aroadmin@ubuntu-jump:~$

```

接続したVM上で次のコマンドを実行します。



```bash
#Install Azure-cli
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

#Install jq
sudo apt install jq -y

#Install aro extension
az extension add -n aro --index https://az.aroapp.io/stable

#Install oc/kubectl
wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/openshift-client-linux.tar.gz
tar zxvf openshift-client-linux.tar.gz
sudo mv oc /usr/local/bin/ 
sudo mv kubectl /usr/local/bin/ 
```

<!-- ### Log into the ARO cluster -->
### AROクラスタへのログイン


```bash
# Initialize variables
LOCATION=japaneast
RESOURCEGROUP=aro-v4-private
CLUSTER=aroprivate

# Login to Azure
az login

#Get the cluster credentials
ARO_PASSWORD=$(az aro list-credentials -n $CLUSTER -g $RESOURCEGROUP -o json | jq -r '.kubeadminPassword')
ARO_USERNAME=$(az aro list-credentials -n $CLUSTER -g $RESOURCEGROUP -o json | jq -r '.kubeadminUsername')
```

AROのAPI Serverのエンドポイントを取得します。
```bash
ARO_URL=$(az aro show -n $CLUSTER -g $RESOURCEGROUP -o json | jq -r '.apiserverProfile.url')
```

次に`oc login`コマンドでクラスタにログインします。

```bash
./oc login $ARO_URL \
    -u $ARO_USERNAME \
    -p $ARO_PASSWORD
```

<!-- ### Run CentOS to test outside connectivity -->
### 外部接続性のテスト
次のコマンドを実行し、AROクラスタ上でCentOSが動くテスト用のPodを起動します。

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

Podが起動したら、その中で外部接続できるかをテストします。

```bash
oc exec -it centos -- /bin/bash
curl microsoft.com
```

<!-- ### Run a test app and expose it internally via Route -->
### サンプルアプリケーションによる稼働確認
AROクラスタ上でサンプルアプリを実行し、Route経由で内部的に公開します。

次のコマンドを実行し、サンプルアプリ`welcome-app service`の`Deployment`リソースを作成します。

```bash
oc apply -f https://raw.githubusercontent.com/akamenev/aro-private/master/welcome-app-deployment.yaml
```
`welcome-app service`の`Service`リソースを作成します。

```bash
oc apply -f https://raw.githubusercontent.com/akamenev/aro-private/master/welcome-app-clusterip-svc.yaml
```
`welcome-app service`の`Service`を`Route`経由で公開します。

```bash
oc expose service welcome-app
```
次のコマンドを実行し、サンプルアプリの動作確認を行います。

```bash
APP_ROUTE=$(oc get route welcome-app -o json | jq -r .spec.host)
curl $APP_ROUTE
```

<!-- ### Expose the app externally with an Internal Load Balancer and Azure Firewall -->
### サービスの公開

内部ロードバランサーとAzure Firewallを使用してアプリを外部に公開します。

次のコマンドを実行し、内部ロードバランサーを作成します。

```bash
oc apply -f https://raw.githubusercontent.com/akamenev/aro-private/master/welcome-app-internallb-svc.yaml
```
Internal IPアドレスを取得します。

```bash
INTERNAL_APP_IP=$(oc get svc welcome-app-internal -o json | jq -r '.status.loadBalancer.ingress[0].ip')
```

Azure FirewallでDNATルールを構成します。

```bash
# Get a Firewall's public ip
FWPUBLIC_IP=$(az network public-ip show -g $RESOURCEGROUP -n fw-ip --query "ipAddress" -o tsv)
echo $FWPUBLIC_IP

# Add a firewall extension
az extension add -n azure-firewall

# Configure a rule
az network firewall nat-rule create \
    -g $RESOURCEGROUP -f aro-private \
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

ブラウザでFirewallのパブリックIPアドレスにアクセスするか、curlコマンドを実行してサンプルアプリの動作確認を行います。

```bash
curl $FWPUBLIC_IP
```
