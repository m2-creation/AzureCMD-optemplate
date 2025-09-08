# AzureCLI Basic Command

## 1.リソースグループ一覧取得

```powershell
# basic command
az group list -o table
```

```powershell
# name only
az group list --query "[].name" -o tsv
```

## 2.特定リソースグループに含まれるVM一覧

```powershell
# basic command
az vm list -g <RG-NAME> -o table
```

```powershell
# overview
az vm list -g <RG-name> --query "[].{name:name,location:location,os:storageProfile.osDisk.osType}" -o table
```

## 3.特定RGに含まれるストレージアカウント一覧

```powershell
# basic command
az storage account list -g <RG-NAME> -o table
```

```powershell
# name only
az storage account list -g <RG-NAME> --query "[].name" -o tsv
```

## 4.特定RGに含まれるDCE/DCR一覧

```powershell
# DCE
az monitor data-collection endpoint list -g <RG-NAME> -o table
```

```powershell
# DCR
az monitor data-collection rule list -g <RG-NAME> -o table
```

※上記どちらのコマンドも`-g`を除けばサブスクリプション全体で一覧化可能

## 5.特定RGに含まれるNSG一覧

```powershell
# basic command
az network nsg list -g <RG-NAME> -o table
```

```powershell
# name only
az network nsg list -g <RG-NAME> --query "[].name" -o tsv
```

## 6.特定サブネット内の使用中のIPアドレス一覧

```powershell
# 空きIPアドレスをざっくり確認（プレビュー）
az network vnet subnet list-available-ips -g <RG-NAME> --vnet-name <VENT-NAME> -n <SUBNET-NAME>
```

```powershell
# NICを起点に対象サブネットで使用されているプライベートIPアドレスを抽出
SUBNET_ID=$(az network vnet subnet show -g <RG-NAME> --vnet-name <VNET-NAME> -n <SUBNET-NAME> --query id -o tsv)
az network nic list \
  --query "[?ipConfigurations[?subnet.id=='${SUBNET_ID}']].ipConfigurations[].{nic:name,privateIp:privateIpAddress}" \
  -o table
```
