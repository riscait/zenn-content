---
title: "VS CodeからCloud Runのデプロイ後にはまったDRSポリシーの設定"
emoji: "😶‍🌫️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [cloudrun, vscode, gcp]
published: true
publication_name: "altiveinc"
---
# 概要

IDE向けに提供されている拡張機能である「Google Cloud Code」のVS Code版を使い、
Google Cloud Runへサービスをデプロイする。

ここまでは、以下の動画で解説されている通りに行えば素晴らしく簡単にデプロイできました🚀

https://www.youtube.com/live/l85hDDzOvuk?feature=share

しかし、今回デプロイしたサービスは、認証なしでアクセスできるように公開したかったので少しハマりました。

組織ポリシーで、認証なしで公開するサービスを制限していたためです。
しかし、今回のサービスは認証なしで公開したいものです…

Google Cloud自体詳しくないので調べながら、「DRSポリシーの設定」やタグの作成・バインドを行い無事公開へ至ることができました🎊

その過程をステップバイステップでこの記事にまとめてみました！

:::message
筆者はGoogle Cloudについて詳しくないので、間違い等がある恐れがあります。
ご指摘いただけると幸いです。
:::

# 前提

組織： `altive.co.jp`
プロジェクト： `altive-api`
サービス： `altive-api`
リージョン： `us-central1`

というサービス環境で行いました。

また、コマンド内で `{ORGANIZATION_ID}` のように波括弧で囲まれた変数は、ご自身のものに置き換えてください。
組織IDは、コンソールか以下のコマンドで確認できます。

```shell
$ gcloud organizations list

DISPLAY_NAME           ID  DIRECTORY_CUSTOMER_ID
altive.co.jp  {ORGANIZATION_ID}              {DIRECTORY_CUSTOMER_ID}
```

# 発生した問題とは

## Setting IAM policy failed
デプロイ自体は成功したものの、以下のエラーが表示されました。

```shell
Setting IAM policy failed, try "gcloud beta run services add-iam-policy-binding --region=us-central1 --member=allUsers --role=roles/run.invoker altive-api"
```

IAMポリシーの設定に失敗したようです。
試しに、デプロイ時にCloud Runにより割り当てられたURLにブラウザにアクセスしても表示できません🤔

まずは、エラーで促されているコマンドを実行してみると…

```shell
$ gcloud beta run services add-iam-policy-binding --region=us-central1 --member=allUsers --role=roles/run.invoker altive-api

ERROR: Policy modification failed. For a binding with condition, run "gcloud alpha iam policies lint-condition" to identify issues in condition.
ERROR: (gcloud.beta.run.services.add-iam-policy-binding) FAILED_PRECONDITION: One or more users named in the policy do not belong to a permitted customer,  perhaps due to an organization policy.
```

やはりポリシーの変更に失敗するようですね。

## Domain Restricted Sharing

ここでググりながら調べていくと、以下のドキュメントにたどり着きました。

https://cloud.google.com/blog/topics/developers-practitioners/how-create-public-cloud-run-services-when-domain-restricted-sharing-enforced?hl=en

`Domain Restricted Sharing` という機能が関係していそうです。

> Google Cloudプロジェクトで意図せず外部ユーザーに権限を付与してしまうことを防ぐセキュリティ機能です。
> 信頼できるドメインのユーザーにのみパーミッションが付与されるようにします。

Google Cloudにて組織ポリシーを設定した際に、このDRSポリシーをオンしたことが原因のようでした。

## 対策方法に関しては2通り
ひとつはDRSポリシーを一旦無効にし、allUsersを含むIAMポリシーを設定した後に、DRSポリシーを再度有効にする方法。

組織ポリシーは遡及的に適用されないためこの手順でも対応可能です。
(方法については[ドキュメント](https://cloud.google.com/resource-manager/docs/organization-policy/restricting-domains#forcing_access)を参照のこと。)

ただ、上記方法は推奨されず、より明示的な方法が紹介されていました。
条件付き組織ポリシーとリソースマネージャタグを使用する方法です。

特定のCloud RunサービスにDRSから除外するタグを付けることで、組織全体のDRS設定制約は持ったまま、特定のサービスに対してinvokerロールをallUsersに割り当てることができるようになります。

# 解決のため手順通りに進めてみます

まずは、DRSから除外するための「タグ」を作成します。

https://cloud.google.com/resource-manager/docs/tags/tags-creating-and-managing?hl=ja

タグはコンソールからも作成できますが、今回は `gcloud` コマンドで以下を実行してみました。

## タグキーの作成
以下のコマンドでタグキーの作成を試みるものの、エラーが表示されました。

```shell
$ gcloud resource-manager tags keys create allUsersIngress \
 --parent=organizations/{ORGANIZATION_ID}

ERROR: (gcloud.resource-manager.tags.keys.create) PERMISSION_DENIED: Permission 'resourcemanager.tagKeys.create' denied on resource '//cloudresourcemanager.googleapis.com/organizations/{ORGANIZATION_ID}' (or it may not exist).
- '@type': type.googleapis.com/google.rpc.ErrorInfo
  domain: cloudresourcemanager.googleapis.com
  metadata:
    permission: resourcemanager.tagKeys.create
    resource: organizations/{ORGANIZATION_ID}
  reason: IAM_PERMISSION_DENIED
```

タグキーを作成するために必要な権限が足りないとのこと。
（一応コンソールで確認してみたら「タグキーの作成」ボタンが非活性状態で押せませんでした）

タグを作成可能なロール（権限）の付与が必要そうです。

## 「タグ管理者」ロールを追加

`gcloud` コマンドでもロール（権限）付与できそうですが、今回はコンソールのIAMから自分に対して「タグ管理者」ロールを追加しました。

必要なロールを付与したので、再度タグキーの作成を試みてみます。

```shell
$ gcloud resource-manager tags keys create allUsersIngress \
 --parent=organizations/{ORGANIZATION_ID}

Waiting for TagKey [allUsersIngress] to be created...done.                                                                                                
createTime: '2023-07-12T01:41:58.105745Z'
etag: e75b1df4-5916-418c-8ee6-7baa14fe5348
name: tagKeys/{TAG_KEY_ID}
namespacedName: {ORGANIZATION_ID}/allUsersIngress
parent: organizations/{ORGANIZATION_ID}
shortName: allUsersIngress
updateTime: '2023-07-12T01:41:58.105745Z'
```

タグキーの作成に成功しました🙌

![](/images/deploy-to-google-cloud-run-from-visual-studio-code/tag_key.png)

## タグ値の作成

先ほど作成したタグキー `allUsersIngress` に対してタグ値 `True` を作成します。

```shell
$ gcloud resource-manager tags values create True \
 --parent={ORGANIZATION_ID}/allUsersIngress \
 --description="Allow for allUsers for internal Cloud Run services"
 
Waiting for TagValue [True] to be created...done.                                                                                                         
createTime: '2023-07-12T02:17:59.998161Z'
description: Allow for allUsers for internal Cloud Run services
etag: e75b1df4-5916-418c-8ee6-7baa14fe5348
name: tagValues/{TAG_VALUE_ID}
namespacedName: {ORGANIZATION_ID}/allUsersIngress/True
parent: tagKeys/{TAG_KEY_ID}
shortName: 'True'
updateTime: '2023-07-12T02:17:59.998161Z'
```

タグ値の作成も成功しました🙌

![](/images/deploy-to-google-cloud-run-from-visual-studio-code/tag_value.png)

## DRSポリシーの作成

以下の `drs-policy.yaml` ファイルをコピペし、組織IDなどを書き換えて保存しました。

```yaml:drs-policy.yaml
name: organizations/ORGANIZATION_ID/policies/iam.allowedPolicyMemberDomains
spec:
  rules:
  - values:
      allowedValues:
      - DIRECTORY_CUSTOMER_ID
  - allowAll: true
    condition:
      expression: resource.matchTag("ORGANIZATION_ID/allUsersIngress", "True")
      title: allowAllUsersIngress
```

`DIRECTORY_CUSTOMER_ID` は以下のコメントで確認できます。

```shell
$ gcloud organizations list
DISPLAY_NAME           ID  DIRECTORY_CUSTOMER_ID
altive.co.jp  {ORGANIZATION_ID}              {DIRECTORY_CUSTOMER_ID}
```

## DRSポリシーの更新

ポリシーを設定するために、以下のコマンドを実行します。

```shell
$ gcloud org-policies set-policy drs-policy.yaml

API [orgpolicy.googleapis.com] not enabled on project [{PROJECT_ID}]. Would you like to enable and retry (this will take a few minutes)? (y/N)?  y

Updated policy [organizations/{ORGANIZATION_ID}/policies/iam.allowedPolicyMemberDomains].
name: organizations/{ORGANIZATION_ID}/policies/iam.allowedPolicyMemberDomains
spec:
  etag: 607abefb-7171-4e67-b3e4-d9f0464034bf
  rules:
  - allowAll: true
    condition:
      expression: resource.matchTag("{ORGANIZATION_ID}/allUsersIngress", "True")
      title: allowAllUsersIngress
  - values:
      allowedValues:
      - {DIRECTORY_CUSTOMER_ID}
  updateTime: '2023-07-12T02:40:37.104153Z'
```

ポリシーを更新できました🙌

## 特定のCloud Runサービスリソースにタグ付け(タグバインディング)

それでは、値TrueのallUsersIngressタグをCloud Runサービスにタグ付けします。

とその前に…

## 「タグユーザー」ロールを追加

「タグ管理者」で足りると思いきや、別途「タグユーザー」のロールが必要でした。
こちらもコンソールか `gcloud` コマンドで付与しましょう。

## タグ付け実行

ロールを付与したら、以下のコマンドを叩いて対象のCloud Runサービスにタグを付けましょう。

```shell
$ gcloud resource-manager tags bindings create \
 --tag-value={ORGANIZATION_ID}/allUsersIngress/True \
 --parent=//run.googleapis.com/projects/{PROJECT_ID}/locations/{REGION}/services/{SERVICE} \
 --location={REGION}

done: true
response:
  '@type': type.googleapis.com/google.cloud.resourcemanager.v3.TagBinding
  name: tagBindings/%2F%2Frun.googleapis.com%2Fprojects%2F{PROJECT_ID}%2Flocations%2F{REGION}%2Fservices%2F{SERVICE}/tagValues/{TAG_VALUE_ID}
  parent: //run.googleapis.com/projects/{PROJECT_ID}/locations/{REGION}/services/{SERVICE}
  tagValue: tagValues/{TAG_VALUE_ID}
  tagValueNamespacedName: {ORGANIZATION_ID}/allUsersIngress/True
```

:::message
`{PROJECT_ID}` には、サービスが所属するプロジェクト名を指定します。
`{SERVICE}` には `altive-api` のようなCloud Runサービス名を指定します。
`{REGION}` には `us-central1` のようなリージョンを指定します。
:::

対象のCloud Runサービスを見てみると…

![](/images/deploy-to-google-cloud-run-from-visual-studio-code/tag_value.png)

`allUsersIngress: True` のタグが付与されていることが確認できました🙌

これでいよいよ準備は整いました🙌

## Cloud RunサービスのIAMポリシーを更新

```shell
$ gcloud beta run services add-iam-policy-binding --region=us-central1 --member=allUsers --role=roles/run.invoker altive-api

Updated IAM policy for service [altive-api].
bindings:
- members:
  - allUsers
  role: roles/run.invoker
etag: 68514b3a-158f-4f68-a380-2be19fda5646
version: 1
```

URLに再度アクセスしてみると…

# デプロイしたページがブラウザから閲覧できました🙌

![](/images/deploy-to-google-cloud-run-from-visual-studio-code/deployed_container_to_cloud_run.png)

まだCloud Codeで作成したテンプレートをデプロイしただけですが、嬉しいですね🙌

# （余談）カスタムドメインの設定

Cloud Runでは `https://altive-api-XXXXXXXX-uc.a.run.app` のようなURLが自動で割り当てられますが、
もちろん独自ドメインを設定することも可能です。

幸い？Google Workspaceでドメインを取得して所有権の証明済みだったので、カスタムドメインを設定するのも簡単でした！

`MANAGE CUSTOM DOMAINS` をクリックして…

![](/images/deploy-to-google-cloud-run-from-visual-studio-code/custom_domain_mapping_01.png)

↑サービスとマッピングしたいベースORサブドメインを選択して、

![](/images/deploy-to-google-cloud-run-from-visual-studio-code/custom_domain_mapping_02.png)

↑ドメインを管理しているサイトでDNSレコードを追加するだけ🚀

すぐには反映されませんが、数十分後には独自ドメインでアクセスできるようになりました🙌

# 参考リンク

https://cloud.google.com/iam/docs/overview?hl=ja