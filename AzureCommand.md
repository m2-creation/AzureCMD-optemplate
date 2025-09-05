
# Azure VM構築手順書 (Azure CLI) 

この手順書は、Azure Cloud Shellを使用して、**既存の仮想ネットワーク（VNet）内**に新しいサブネットと仮想マシン（VM）環境を構築するためのコマンドと手順をまとめたものです。

## 0. 準備：変数の設定

コマンド内で使用する共通の値をシェル変数に設定します。`VNET_NAME` には、利用したい既存のVNet名を指定してください。

```bash
# ## --- 共通変数の設定 --- ##

# リソースグループ名 (VMやサブネットなど新規リソースを作成するリソースグループ)
RESOURCE_GROUP="my-vm-rg"

# リージョン
LOCATION="japaneast"

# 仮想マシン名
VM_NAME="my-vm"

# VMサイズ (例: Standard_D2s_v3)
# 利用可能なサイズは `az vm list-sizes --location $LOCATION --output table` で確認
VM_SIZE="Standard_D2s_v3"

# 管理者ユーザー名
ADMIN_USER="azureuser"

# 管理者パスワード (Azureのパスワード要件を満たす複雑なものにしてください)
ADMIN_PASSWORD="YourComplexPassword!123"

# --- ネットワーク関連 ---
# ★【要指定】利用する"既存"VNetの名前
VNET_NAME="existing-vnet"
# ★【要指定】既存VNetが存在するリソースグループ名
VNET_RESOURCE_GROUP="existing-vnet-rg"
# 新しく作成するサブネット名
SUBNET_NAME="my-new-subnet"
# 新しく作成するサブネットのアドレス範囲
SUBNET_PREFIX="10.0.2.0/24"

# NSG名
NSG_NAME="my-nsg"

# ルートテーブル名
ROUTE_TABLE_NAME="my-route-table"

# Log Analyticsワークスペース名
LOG_ANALYTICS_WORKSPACE_NAME="my-la-workspace"

# データ収集エンドポイント(DCE)名
DCE_NAME="my-dce"

# データ収集ルール(DCR)名
DCR_NAME="my-dcr"
```

## 1. Azureへのログインとサブスクリプションの設定

Azureにログインし、操作対象のサブスクリプションを選択します。

```bash
# Azureにログイン (Cloud Shellでは通常自動でログインされます)
az login

# テナント内のサブスクリプション一覧を表示
az account list --output table

# 操作対象のサブスクリプションを設定 (IDまたは名前を指定)
az account set --subscription "YOUR_SUBSCRIPTION_ID_OR_NAME"

# 【確認】現在の操作対象サブスクリプションを表示
az account show --output table
```

## 2. リソースグループの作成

VMやサブネットなど、今回新しく作成するリソースを格納するためのリソースグループを作成します。

```bash
az group create \
  --name $RESOURCE_GROUP \
  --location $LOCATION

# 【確認】リソースグループが作成されたことを確認
az group show --name $RESOURCE_GROUP --output table
```

## 3. OSイメージの検索

VM作成に使用するOSイメージをMarketplaceから検索し、URN（一意識別子）を取得します。

```bash
# よく使われるイメージを一覧表示して概要を把握
az vm image list --output table

# 検索した情報からイメージのURNを取得
# Linuxの例:
IMAGE_URN_LINUX="Canonical:0001-com-ubuntu-server-jammy:22_04-lts-gen2:latest"
# Windowsの例:
IMAGE_URN_WINDOWS="MicrosoftWindowsServer:WindowsServer:2022-datacenter-azure-edition:latest"

# 【確認】イメージの詳細情報を確認
az vm image show --urn $IMAGE_URN_LINUX
```

## 4. ネットワークリソースの作成 (既存VNet内にサブネットを新規作成)

既存のVNet内に新しいサブネットを作成し、そのサブネットに新規作成したNSGとルートテーブルを関連付けます。

```bash
# 4.1. 新しいサブネットを既存のVNet内に作成
az network vnet subnet create \
  --resource-group $VNET_RESOURCE_GROUP \
  --vnet-name $VNET_NAME \
  --name $SUBNET_NAME \
  --address-prefixes $SUBNET_PREFIX

# 4.2. ネットワークセキュリティグループ (NSG) の作成 (新規リソース用のRGに作成)
az network nsg create \
  --resource-group $RESOURCE_GROUP \
  --name $NSG_NAME

# 4.3. ルートテーブルの作成 (新規リソース用のRGに作成)
az network route-table create \
  --resource-group $RESOURCE_GROUP \
  --name $ROUTE_TABLE_NAME

# 4.4. 作成したサブネットにNSGとルートテーブルを適用（ID で関連付け）
NSG_ID=$(az network nsg show --resource-group $RESOURCE_GROUP --name $NSG_NAME --query id -o tsv)
RT_ID=$(az network route-table show --resource-group $RESOURCE_GROUP --name $ROUTE_TABLE_NAME --query id -o tsv)
az network vnet subnet update \
  --resource-group $VNET_RESOURCE_GROUP \
  --vnet-name $VNET_NAME \
  --name $SUBNET_NAME \
  --network-security-group $NSG_ID \
  --route-table $RT_ID

# サブネットのリソースID（VM 作成で利用）
SUBNET_ID=$(az network vnet subnet show --resource-group $VNET_RESOURCE_GROUP --vnet-name $VNET_NAME --name $SUBNET_NAME --query id -o tsv)

# 【確認】サブネットの詳細を表示し、NSGとルートテーブルが関連付けられていることを確認
az network vnet subnet show \
  --resource-group $VNET_RESOURCE_GROUP \
  --vnet-name $VNET_NAME \
  --name $SUBNET_NAME \
  --query '{name:name, addressPrefixes:addressPrefixes, nsg:networkSecurityGroup.id, routeTable:routeTable.id}'
```

## 5. NSGルールの設定例

特定のIPアドレスからのみSSH/RDP接続を許可するルールを追加します。`--source-address-prefixes` には、接続元のグローバルIPアドレスを指定してください。

```bash
# 例1: 特定のIPからSSH(ポート22)を許可するルール (Linux VM用)
az network nsg rule create \
  --resource-group $RESOURCE_GROUP \
  --nsg-name $NSG_NAME \
  --name AllowSshFromMyIp \
  --priority 1000 \
  --protocol Tcp \
  --destination-port-ranges 22 \
  --access Allow \
  --direction Inbound \
  --source-address-prefixes 'YOUR_GLOBAL_IP_ADDRESS'

# 例2: 特定のIPからRDP(ポート3389)を許可するルール (Windows VM用)
az network nsg rule create \
  --resource-group $RESOURCE_GROUP \
  --nsg-name $NSG_NAME \
  --name AllowRdpFromMyIp \
  --priority 1010 \
  --protocol Tcp \
  --destination-port-ranges 3389 \
  --access Allow \
  --direction Inbound \
  --source-address-prefixes 'YOUR_GLOBAL_IP_ADDRESS'

# 【確認】設定されたNSGルールの一覧を表示
az network nsg rule list --resource-group $RESOURCE_GROUP --nsg-name $NSG_NAME --output table
```

## 6. 仮想マシン (VM) の作成

VMサイズを指定し、新しく作成したサブネット上にパスワード認証でVMを作成します。

### 6.1. Linux VMの作成

```bash
az vm create \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME \
  --image $IMAGE_URN_LINUX \
  --size $VM_SIZE \
  --subnet $SUBNET_ID \
  --admin-username $ADMIN_USER \
  --admin-password $ADMIN_PASSWORD \
  --public-ip-address "" \
  --nsg "" \
  --storage-sku StandardSSD_LRS
```

### 6.2. Windows VMの作成

```bash
az vm create \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME \
  --image $IMAGE_URN_WINDOWS \
  --size $VM_SIZE \
  --subnet $SUBNET_ID \
  --admin-username $ADMIN_USER \
  --admin-password $ADMIN_PASSWORD \
  --public-ip-address "" \
  --nsg "" \
  --storage-sku StandardSSD_LRS
```

```bash
# 【確認】VMの作成が完了し、ステータスが "VM running" であることを確認
az vm show \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME \
  --show-details \
  --query '{name:name, size:hardwareProfile.vmSize, powerState:powerState}' \
  --output table
```

## 7. データディスクと共有ディスクの追加

VMにデータディスクと共有ディスクをアタッチします。（このセクションは前回から変更ありません）

### 7.1. データディスクのアタッチ

```bash
# 128GBのStandard SSDをLUN 0に、キャッシュは None（推奨）でアタッチ
az vm disk attach \
  --resource-group $RESOURCE_GROUP \
  --vm-name $VM_NAME \
  --name ${VM_NAME}-datadisk1 \
  --size-gb 128 \
  --sku StandardSSD_LRS \
  --lun 0 \
  --caching None \
  --new
```

### 7.2. 共有ディスクの作成とアタッチ

```bash
# 共有ディスク用のディスクを作成 (Premium SSD, 256GB, 2台で共有可能)
SHARED_DISK_NAME="my-shared-disk"
az disk create \
  --resource-group $RESOURCE_GROUP \
  --name $SHARED_DISK_NAME \
  --size-gb 256 \
  --sku Premium_LRS \
  --max-shares 2

# 作成した共有ディスクをLUN 1にアタッチ
az vm disk attach \
  --resource-group $RESOURCE_GROUP \
  --vm-name $VM_NAME \
  --name $SHARED_DISK_NAME \
  --lun 1 \
  --caching None
```

```bash
# 【確認】VMにアタッチされているディスクの一覧と設定（LUN, Caching）を確認
az vm show \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME \
  --query 'storageProfile.dataDisks' \
  --output table
```

## 8. 監視リソースの作成と設定

Log Analytics, DCE, DCRを作成し、VMのログを収集します。（このセクションは前回から変更ありません）

```bash
# 8.1. Log Analyticsワークスペースの作成
az monitor log-analytics workspace create \
  --resource-group $RESOURCE_GROUP \
  --workspace-name $LOG_ANALYTICS_WORKSPACE_NAME \
  --location $LOCATION

# 8.2. データ収集エンドポイント (DCE) の作成
az monitor data-collection endpoint create \
  --resource-group $RESOURCE_GROUP \
  --name $DCE_NAME \
  --location $LOCATION

# 8.3. データ収集ルール (DCR) の作成 (Linux Syslogの例)
DCR_FILE="dcr.json"
WORKSPACE_ID=$(az monitor log-analytics workspace show --resource-group $RESOURCE_GROUP --workspace-name $LOG_ANALYTICS_WORKSPACE_NAME --query id -o tsv)
DCE_ID=$(az monitor data-collection endpoint show --resource-group $RESOURCE_GROUP --name $DCE_NAME --query id -o tsv)

cat <<EOF > $DCR_FILE
{
  "location": "$LOCATION",
  "properties": {
    "dataSources": { "syslog": [ { "name": "syslogDataSource", "streams": [ "Microsoft-Syslog" ], "facilityNames": ["*"], "logLevels": ["*"] } ] },
    "destinations": { "logAnalytics": [ { "workspaceResourceId": "$WORKSPACE_ID", "name": "la-destination" } ] },
    "dataFlows": [ { "streams": [ "Microsoft-Syslog" ], "destinations": [ "la-destination" ] } ],
    "dataCollectionEndpointId": "$DCE_ID"
  }
}
EOF

az monitor data-collection-rule create \
  --resource-group $RESOURCE_GROUP \
  --name $DCR_NAME \
  --rule-file $DCR_FILE

# 8.4. VMとDCRの関連付け
VM_ID=$(az vm show --resource-group $RESOURCE_GROUP --name $VM_NAME --query id -o tsv)
DCR_ID=$(az monitor data-collection-rule show --resource-group $RESOURCE_GROUP --name $DCR_NAME --query id -o tsv)
az monitor data-collection-rule association create \
  --name "dcr-association" \
  --rule-id $DCR_ID \
  --resource $VM_ID

# 【確認】VMにDCRが正しく関連付けられたことを確認
az monitor data-collection-rule association list --resource $VM_ID --output table
```

---
