---
title: Application Gateway と Key Vault を統合した際によくある問題について 
date: 2024-03-12 14:10:00 
tags:
  - Network
  - Application Gateway
  - Key Vault
---

こんにちは、Azure テクニカル サポート チームの金です。
Application Gateway v2 (Standard/WAF) の TLS 終端は、リスナーに直接証明書をアップロードする、またはリスナーにて Key Vault 証明書/シークレットを参照することで実現できます。
リスナーにて Key Vault 証明書/シークレットを参照する構成では、Applocation Gateway から 4 時間毎に Key Vault に 証明書/シークレットを参照する動作を行います。Applocation Gateway を更新するタイミングでも、Key Vault へ 証明書/シークレットを参照します。

この記事では、リスナーにて Key Vault 証明書/シークレットを参照する構成で、Application Gateway に HTTPS でアクセスした際に、「ERR_SSL_UNRECOGNIZED_NAME_ALERT」といった証明書エラーが発生する、もしくは Application Gateway の停止・起動や変更が正常に完了せず、Application Gateway のプロビジョニング状態が失敗になった場合に、確認すべき設定をご紹介します。リスナーにて Key Vault 証明書/シークレットを参照する設定を行う際にも該当します。

> [!TIP]
> Application Gateway から Key Vault 証明書/シークレットを取得する際は、マネージド ID を使用しており、Key Vault 側では、ロールベースのアクセス制御、またはアクセス ポリシーで、当該マネージド ID に対して適切なアクセス権限を付与する必要があります。
> Application Gateway と Key Vault 統合の詳細については、[公式ドキュメント](https://learn.microsoft.com/ja-jp/azure/application-gateway/key-vault-certs)をご参照ください。

### Key Vault 自体、Key Vault 証明書/シークレットの存在有無
Key Vault 自体が削除された場合は、Key Vault を復元する、もしくは Key Vault を新規作成する必要があります。
Key Vault 証明書/シークレットが削除された場合は、Key Vault 証明書/シークレットを復元する、もしくは Key Vault 証明書/シークレットを新規作成する必要があります。

Key Vault 自体、Key Vault 証明書/シークレットの削除や復元する方法に関しては、[公式ドキュメント](https://learn.microsoft.com/ja-jp/azure/key-vault/general/key-vault-recovery?tabs=azure-portal)をご参照ください。

### Key Vault におけるマネージド ID のアクセス権限の確認
ロールベースのアクセス制御を利用している場合は、Key Vault の [アクセス制御 (IAM)] で対象のマネージド ID に「キー コンテナー シークレット ユーザー」ロールを付与する必要があります。
アクセスポリシーを利用している場合は、Key Vault の [アクセス ポリシー] で対象のマネージド ID に [シークレットのアクセス許可] の [取得] 権限を付与する必要があります。

### マネージド ID の存在有無
マネージド ID が削除された場合は、リスナーにおける証明書関連の設定更新のみならず、Application Gateway 停止・起動や変更が正常に完了せず、Application Gateway のプロビジョニング状態が失敗になります。その際は、以下のようなエラー内容が出力されます。

> Response: '{"error":{"code":"BadRequest","message":"Resource '/subscriptions/サブスクリプション ID/resourcegroups/リソース グループ名/providers/Microsoft.ManagedIdentity/userAssignedIdentities/マネージド ID' was not found."}}'.' で失敗しました。

解決方法：上記エラー内容に示したリソース グループに同名のマネージド ID を作成し、Key Vault 側で当該マネージド ID に対して適切な権限を付与します。

### 補足 1
Key Vault 統合で設定不備があった場合は、Azure Advisor より Key Vault エラーが出力される場合もありますので、エラー コードと解決策については、[公式ドキュメント](https://learn.microsoft.com/ja-jp/azure/application-gateway/application-gateway-key-vault-common-errors)をご参照ください。

### 補足 2
Application Gateway からプライベート エンドポイントを介して Key Vault にアクセスする場合は、以下の設定も必要となります。
- プライベート DNS ゾーン (privatelink.vaultcore.azure.net) にプライベート エンドポイントの DNS レコードが登録されている
- プライベート DNS ゾーン (privatelink.vaultcore.azure.net) に Application Gateway の VNet がリンクされている
- Application Gateway の VNet の [DNS サーバー] は、「既定 (Azure 提供)」にする
> [!TIP]
> Application Gateway の VNet の [DNS サーバー] で、カスタム DNS サーバーを指定している場合は、[公式ドキュメント](https://learn.microsoft.com/ja-jp/azure/application-gateway/key-vault-certs#verify-firewall-permissions-to-key-vault)に記載されたように、Key Vault 証明書/シークレットを正常に取得できない場合がございます。

- プライベート エンドポイントが所属するサブネットに NSG を構成しており、当該サブネットの [プライベート エンドポイント ネットワーク ポリシー] の [ネットワーク セキュリティ グループ] が有効になっている場合は、NSG の受信規則で Application Gateway のサブネットからの通信を許可する

### 補足 3
- AppService 証明書を Key Vault に格納して使用する場合は、AppService 証明書のドメイン検証が終わってから、Application Gateway のリスナー側で指定してください。そうでない場合、Key Vault 側で証明書参照の準備ができていないため、Application Gateway のプロビジョニング状態が失敗になる可能性があります。
- Application Gateway のリスナーを削除した際に、適応済みに証明書はそのまま残りますので、不要な証明書は削除することをお勧めします。削除方法は以下の通りです。
Azure ポータルより対象の Application Gateway を開き、[リソース] → [リスナー TLS 証明書] 一覧から、対象の証明書を削除します。
- Applocation Gateway に複数のリスナーがある場合、リスナー毎に異なるマネージド ID を指定せず、Applocation Gateway 単位で同一マネージド ID をご利用ください。

### お願い
上記の設定内容に不備がない場合、Applocation Gateway の観点でお問い合わせいただければと存じますが、その場合は、以下の三つのコマンドを実行し、実行したコマンドとその結果をお問い合わせ時に添付いただけるとスムーズかと存じます。

```
Get-AzKeyVault -VaultName <KeyVault 名>
Get-AzKeyVaultSecret -VaultName <KeyVault 名>
Get-AzKeyVaultSecret -VaultName <KeyVault 名> -Name
```
---
