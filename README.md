# kitchensink-config
Manifest files for OpenShift GitOps.

# 対象の名前空間の事前準備

普通の権限のユーザで構わない。

```
SOURCE_NAMESPACE=kitchensink-dev
TARGET_NAMESPACE=kitchensink-staging

oc new-project ${TARGET_NAMESPACE}

# Argo CDに実行許可を与える
oc policy add-role-to-user admin \
  system:serviceaccount:openshift-gitops:openshift-gitops-argocd-application-controller

# SOURCE_NAMESPACEで作成したイメージをpullする許可を与える
oc policy add-role-to-group system:image-puller \
  system:serviceaccounts:$(oc project -q) \
  -n ${SOURCE_NAMESPACE}

# EAPのクラスタリングが使うKUBE_PINGの許可を与える
oc policy add-role-to-user view -z default

# アプリが使うDBの作成 (blue/green共通)
oc new-app \
  postgresql-ephemeral \
  -p POSTGRESQL_USER=pguser \
  -p POSTGRESQL_PASSWORD=pguser \
  -p POSTGRESQL_DATABASE=kitchensink \
  -p POSTGRESQL_VERSION=12-el8

# デプロイエラーを防ぐため、現在指定されているタグをあらかじめ作成しておく
# タグは overlays/staging/kustomization.yaml の現在の値を見て確認する
oc tag kitchensink:latest kitchensink:<tag-blue> -n ${SOURCE_NAMESPACE}
oc tag kitchensink:latest kitchensink:<tag-green> -n ${SOURCE_NAMESPACE}

```

# Argo CDのアプリケーションの設定

以下はcluster-admin権限のユーザで行う。

まず必要なソフトウェアとして以下のOperatorをOperatorHubからインストールする。

- Red Hat OpenShift GitOps (Red Hat公式のArgo CD)
- JBoss EAP

Argo CDの管理コンソールのURLを調べる。

    oc get route openshift-gitops-server -n openshift-gitops

Argo CDの管理ユーザのパスワードを調べる。

    oc extract secret/openshift-gitops-cluster -n openshift-gitops --to=-

上記で調べた管理コンソールに admin/<上記のパスワード> でログインし、
以下の設定でアプリケーションを作成する。

| 項目             | 値                                               | 備考                                                    |
|------------------|--------------------------------------------------|---------------------------------------------------------|
| Application Name | kitchensink-application                          | openshift-gitops名前空間に作成されるApplicationの名前。 |
| Project          | default                                          | Argo CDのプロジェクト。固定でよい。                     |
| Sync Policy      | Automatic                                        | "Prune Resources"と"Self Heal"にもチェックを入れる。    |
| Repository URL   | https://github.com/onagano-rh/kitchensink-config | 自分の設定用リポジトリ (このリポジトリ)。               |
| Revision         | main                                             | 使用するブランチまたはタグ。                            |
| Path             | overlays/staging                                 | 使用するkustomization.yamlがあるディレクトリ。          |
| Cluster URL      | https://kubernetes.default.svc                   | デプロイ先のクラスタ。固定でよい。                      |
| Namespace        | (TARGET_NAMESPACEと同じ値)                       | 上記で作成したデプロイ先の名前空間。                    |
