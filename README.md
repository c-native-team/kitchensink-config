# kitchensink-config
Manifest files for OpenShift GitOps.

# 対象の名前空間の事前準備

普通の権限のユーザで構わない。

```
SOURCE_NAMESPACE=myapp-dev
TARGET_NAMESPACE=myapp-staging

oc new-project ${TARGET_NAMESPACE}

oc policy add-role-to-user admin \
  system:serviceaccount:openshift-gitops:openshift-gitops-argocd-application-controller

oc policy add-role-to-group system:image-puller \
  system:serviceaccounts:$(oc project -q) \
  -n ${SOURCE_NAMESPACE}

oc new-app \
  postgresql-ephemeral \
  -p POSTGRESQL_USER=pguser \
  -p POSTGRESQL_PASSWORD=pguser \
  -p POSTGRESQL_DATABASE=kitchensink \
  -p POSTGRESQL_VERSION=12-el8

```

# Argo CDのアプリケーションの設定

cluster-admin権限のユーザで行う。

OperatorHubで"Red Hat OpenShift GitOps" (Red Hat公式のArgo CD) をインストールする。

Argo CDの管理コンソールのURLを調べる。

    oc get route openshift-gitops-server -n openshift-gitops

adminユーザのパスワードを調べる。

    oc extract secret/openshift-gitops-cluster -n openshift-gitops --to=-

管理コンソールにログインし、以下の設定でアプリケーションを作成する。

| 項目             | 値                                               | 備考                                                    |
|------------------|--------------------------------------------------|---------------------------------------------------------|
| Application Name | kitchensink-gitops-app                           | openshift-gitops名前空間に作成されるApplicationの名前。 |
| Project          | default                                          | Argo CDのプロジェクト。固定でよい。                     |
| Sync Policy      | Automatic                                        | "Prune Resources"と"Self Heal"にもチェックを入れる。    |
| Repository URL   | https://github.com/onagano-rh/kitchensink-config | 自分の設定用リポジトリ (このリポジトリ)。               |
| Revision         | main                                             | 使用するブランチまたはタグ。                            |
| Path             | application                                      | マニフェストがあるディレクトリ。                        |
| Cluster URL      | https://kubernetes.default.svc                   | デプロイ先のクラスタ。固定でよい。                      |
| Namespace        | myapp-staging                                    | 上記で作成したデプロイ先の名前空間。                    |
