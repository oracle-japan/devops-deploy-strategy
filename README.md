# OCI DevOps Deploy Strategy

Sample Code for Blue-Green Deployment and Canary Release in OCI DevOps

## BLUE-GREEN Demo Environment

### OKE

以下チュートリアルを参照して、Kubernetesクラスタを構築します。

本でも環境では、3 Node としています。

[Oracle Container Engine for Kubernetes(OKE)をプロビジョニングしよう](https://oracle-japan.github.io/ocitutorials/cloud-native/oke-for-commons/)

1. OKEクラスターのプロビジョニング
2. CLI実行環境(Cloud Shell)の準備

### OCIR

以下チュートリアルを参照して、OCIRにリポジトリを作成します。

[Oracle Cloud Infrastructure(OCI) DevOpsことはじめ-OKE編-](https://oracle-japan.github.io/ocitutorials/cloud-native/devops-for-beginners-oke/)

3. アーティファクト
  - 3-1. OCIRのセットアップ

＜パラメータ＞  
コンパートメント：対象とするコンパートメント  
リポジトリ名：blue-green  
アクセス：パブリック

### Ingress Controller Setup

サンプルコードを取得します。

```
git clone https://github.com/oracle-japan/devops-deploy-strategy.git
```

ディレクトリの内容を確認します。

```
ls devops-deploy-strategy
```
```
deploy-strategy-bg  deploy-strategy-canary  README.md
```

`deploy-strategy-bg` ディレクトリに移動して、ingress controller をセットアップします。

```
cd deploy-strategy-bg
```
```
kubectl apply -f ingress-controller.yaml 
```
```
namespace/ingress-nginx created
serviceaccount/ingress-nginx created
serviceaccount/ingress-nginx-admission created
role.rbac.authorization.k8s.io/ingress-nginx created
role.rbac.authorization.k8s.io/ingress-nginx-admission created
clusterrole.rbac.authorization.k8s.io/ingress-nginx created
clusterrole.rbac.authorization.k8s.io/ingress-nginx-admission created
rolebinding.rbac.authorization.k8s.io/ingress-nginx created
rolebinding.rbac.authorization.k8s.io/ingress-nginx-admission created
clusterrolebinding.rbac.authorization.k8s.io/ingress-nginx created
clusterrolebinding.rbac.authorization.k8s.io/ingress-nginx-admission created
configmap/ingress-nginx-controller created
service/ingress-nginx-controller created
service/ingress-nginx-controller-admission created
deployment.apps/ingress-nginx-controller created
job.batch/ingress-nginx-admission-create created
job.batch/ingress-nginx-admission-patch created
ingressclass.networking.k8s.io/nginx created
validatingwebhookconfiguration.admissionregistration.k8s.io/ingress-nginx-admission created
```

3個の `ingress-nginx-controller` のステータスが `Running` であることを確認します。

```
kubectl get pods -n ingress-nginx
```
```
NAME                                        READY   STATUS      RESTARTS   AGE
ingress-nginx-admission-create-4jptb        0/1     Completed   0          43m
ingress-nginx-admission-patch-5pch2         0/1     Completed   1          43m
ingress-nginx-controller-8574b6d7c9-4nskl   1/1     Running     0          43m
ingress-nginx-controller-8574b6d7c9-6k5zp   1/1     Running     0          43m
ingress-nginx-controller-8574b6d7c9-hpf62   1/1     Running     0          43m
```

LoadBalancer `ingress-nginx-controller` の EXTERNAL-IP が表示されていることを確認します。

```
kubectl get services -n ingress-nginx
```
```
NAME                                 TYPE           CLUSTER-IP      EXTERNAL-IP       PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.96.107.235   132.xxx.xxx.xxx   80:32487/TCP,443:32238/TCP   44m
ingress-nginx-controller-admission   ClusterIP      10.96.101.214   <none>            443/TCP                      44m
```

### Namespace Create

対象のKubernetesクラスタに、BLUEとGREEN用のNamespaceを作成します。

```
kubectl create ns blue
```
```
namespace/blue created
```

```
kubectl create ns green
```
```
namespace/green created
```

blueとgreenのNamespaceができていること確認します。

```
kubectl get ns
```
```
NAME              STATUS   AGE
blue              Active   11m
default           Active   5h58m
green             Active   11m
ingress-nginx     Active   153m
kube-node-lease   Active   5h58m
kube-public       Active   5h58m
kube-system       Active   5h58m
```

### OCI DevOps Setup

#### 1. プロジェクトの作成

OCI DevOps環境の事前準備はこちらを参照して作成します。

[Oracle Cloud Infrastructure(OCI) DevOps事前準備](https://oracle-japan.github.io/ocitutorials/cloud-native/devops-for-commons/)

ここでは、以下の名前で作成します。

トピック名：devops-deploy-strategy
プロジェクト名：blue-green

#### 2. 環境の作成

作成したプロジェクトを選択して、「環境」を選択します。  
左メニューから「環境」を選択します。

![環境の作成](./images/01.png)

「環境の作成」ボタンをクリックします。

![環境の作成](./images/02.png)

以下の設定を行い、「次」ボタンをクリックします。

- 環境タイプ：Oracle Kubernetesエンジン  
- 名前：blue-green-cluster

![環境の作成](./images/03.png)

![環境の作成](./images/04.png)

以下の設定を行って、「環境の作成」ボタンをクリックします。

- リージョン：対象クラスタのリージョン  
- コンパートメント：対象のコンパートメント
- クラスタ：対象のクラスタ（ここでは cluster1）

![環境の作成](./images/05.png)

![環境の作成](./images/06.png)

パンくずリストから「blue-green」を選択します。

![環境の作成](./images/07.png)

#### 3. コード・リポジトリの作成

左メニューから「コード・リポジトリ」を選択します。

![コード・リポジトリの作成](./images/08.png)

「リポジトリの作成」ボタンをクリックします。

![コード・リポジトリの作成](./images/09.png)

以下の設定を行い、「リポジトリの作成」ボタンをクリックします。

- リポジトリ名：strategy-blue-green  
- デフォルト・ブランチ・オプション：main

![コード・リポジトリの作成](./images/10.png)

![コード・リポジトリの作成](./images/09.png)

「HTTPS」ボタンをクリックして、コマンドの「コピー」テキストをクリック後、コマンドを実行してクローンします。

![コード・リポジトリの作成](./images/11.png)

クローンしたディレクトリにサンプルコードをコピーします。

```
cp -pR devops-deploy-strategy/deploy-strategy-bg/* ./strategy-blue-green/
```
```
ls strategy-blue-green/
```
```
blue-green-app.yaml  build_spec.yaml  content.html  Dockerfile  ingress-controller.yaml
```

作成したリポジトリにサンプルコードをプッシュします。

```
cd strategy-blue-green
```
```
git add -A .
```
```
git commit -m "first commit"
```
```
[main 46179ca] first commit
 5 files changed, 785 insertions(+)
 create mode 100644 Dockerfile
 create mode 100644 blue-green-app.yaml
 create mode 100644 build_spec.yaml
 create mode 100644 content.html
 create mode 100644 ingress-controller.yaml
```

```
git branch -M main
```
```
git push -u origin main
```
```
Enumerating objects: 8, done.
Counting objects: 100% (8/8), done.
Delta compression using up to 2 threads
Compressing objects: 100% (7/7), done.
Writing objects: 100% (7/7), 3.79 KiB | 1.26 MiB/s, done.
Total 7 (delta 0), reused 0 (delta 0), pack-reused 0
To https://devops.scmservice.uk-london-1.oci.oraclecloud.com/namespaces/orasejapan/projects/blue-green/repositories/strategy-blue-green
   dc00f99..46179ca  main -> main
Branch 'main' set up to track remote branch 'main' from 'origin'.
```

リポジトリを確認すると、サンプルコードが表示されています。

![コード・リポジトリの作成](./images/12.png)

#### 4. アーティファクトの作成

Kubernetesクラスタにデプロイする際に必要となるマニフェストファイルをアーティファクトレジストリにアップロードします。最初にアーティファクトレジストリにリポジトリを作成します。

アップロードするマニフェストファイル内にある、コンテナイメージのパスを変更します。

```
vim blue-green-app.yaml 
```
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: helloworld
spec:
  selector:
    matchLabels:
      app: helloworld
  replicas: 3
  template:
    metadata:
      labels:
        app: helloworld
    spec:
      containers:
        - name: helloworld
          # enter the path to your image, be sure to include the correct region prefix
          image: <your-region>.ocir.io/<your-object-storage-namespace>/blue-green/blue-green-app:${BUILDRUN_HASH} #対象となるパスに変更
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: helloworld
spec:
  type: ClusterIP
  ports:
    - port: 80
      targetPort: 80
  selector:
    app: helloworld
---
# main-ingress.yaml with networking.k8s.io/v1 version
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: helloworld-ing
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  rules: 
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: helloworld
                port:
                  number: 80
```

ハンバーガーメニューから「開発者サービス」-「アーティファクト・レジストリ」を選択します。

![アーティファクトの作成](./images/13.png)

「リポジトリの作成」ボタンをクリックします。

![アーティファクトの作成](./images/09.png)

以下の設定を行い、「作成」ボタンをクリックします。

- 名前：artifact-repository-bg
- コンパートメント：対象となるコンパートメント
- 不変アーティファクト：チェックを外す

![アーティファクトの作成](./images/14.png)

![アーティファクトの作成](./images/15.png)

マニフェストファイルをアップロードします。「アーティファクトのアップロード」ボタンをクリックします。

![アーティファクトの作成](./images/16.png)

以下の設定を行い、「コピー」テキストをクリックします。

- アーティファクト・パス：blue-green-app.yaml
- バージョン：1
- Uplad method：Cloud Shell

![アーティファクトの作成](./images/17.png)

コピーした内容をペーストして、「./＜file-name＞」の箇所を「./blue-grenn-app.yaml」に変更してコマンドを実行します。

```
oci artifacts generic artifact upload-by-path \
>   --repository-id ocid1.artifactrepository.oc1.uk-london-1.0.amaaaaaassl65iqaj6s4wnf3k5tvtanp3impmofawxr6k6ps3bc7upjlxfla \
>   --artifact-path blue-green-app.yaml \
>   --artifact-version 1 \
>   --content-body ./blue-green-app.yaml
```
以下の結果が返ってくればアップロード完了です。
```
{
  "data": {
    "artifact-path": "blue-green-app.yaml",
    "compartment-id": "ocid1.compartment.oc1..aaaaaaaagp3t6j5zjzg3hrau4i3rtcxvl64cxbcdgpid5smp6avpnfecegqa",
    "defined-tags": {},
    "display-name": "blue-green-app.yaml:1",
    "freeform-tags": {},
    "id": "ocid1.genericartifact.oc1.uk-london-1.0.amaaaaaassl65iqakquhxcpokv7k3sfdq53hx2dxyzhtnd6vaxgrkilrzxeq",
    "lifecycle-state": "AVAILABLE",
    "repository-id": "ocid1.artifactrepository.oc1.uk-london-1.0.amaaaaaassl65iqaj6s4wnf3k5tvtanp3impmofawxr6k6ps3bc7upjlxfla",
    "sha256": "de9f5af3b99b54c0aa05a473af504644972f9cb1a0ed0c283214e4debd7a930d",
    "size-in-bytes": 1167,
    "time-created": "2023-01-26T05:32:22.234000+00:00",
    "version": "1"
  }
}
```

「閉じる」ボタンをクリックします。

![アーティファクトの作成](./images/18.png)

ブラウザを更新するとアップロードしたアーティファクトがリストされます。

![アーティファクトの作成](./images/19.png)

OCI DevOpsの「blue-green」プロジェクト画面に戻って、アーティファクトに対象のOCIRとアーティファクトレジストリのリポジトリを登録します。

OCI DevOpsの左目メニューから「アーティファクト」を選択します。

![アーティファクトの作成](./images/20.png)

「アーティファクトの追加」ボタンをクリックします。

![アーティファクトの作成](./images/21.png)

以下の設定を行い、「追加」ボタンをクリックします。

- 名前：ocir-repogitory-bg
- タイプ：コンテナ・イメージ・リポジトリ
- コンテナ・レジストリのイメージへの完全修飾パスを入力してください：マニフェストファイルに指定したリポジトリを設定してください

![アーティファクトの作成](./images/22.png)

![アーティファクトの作成](./images/23.png)

「アーティファクトの追加」ボタンをクリックします。

![アーティファクトの作成](./images/21.png)

以下の設定を行い、「選択」ボタンをクリックします。

- 名前：artifact-repogitory-bg
- タイプ：Kubernetesマニフェスト

![アーティファクトの作成](./images/24.png)

以下の設定を行い、「選択」ボタンをクリックします。

- リージョン：対象となるリージョン
- コンパートメント：対象となるコンパートメント
- アーティファクト・レジストリ・リポジトリ：artifact-repository-bg

![アーティファクトの作成](./images/25.png)

![アーティファクトの作成](./images/26.png)

「アーティファクト」の「選択」ボタンをクリックします。

![アーティファクトの作成](./images/27.png)

「blue-green-app.yaml:v1」にチェックを入れて、「選択」ボタンをクリックします。

![アーティファクトの作成](./images/28.png)

![アーティファクトの作成](./images/26.png)

最後に「追加」ボタンをクリックします。

![アーティファクトの作成](./images/23.png)

登録したOCIRとアーティファクト・レジストリのリポジトリがリストされていることを確認します。

![アーティファクトの作成](./images/29.png)

パンくずリストから「blue-green」を選択します。

![アーティファクトの作成](./images/30.png)

#### 5. デプロイメント・パイプラインの作成

デプロイメント・パイプラインを作成します。  
左メニューから「デプロイメント・パイプライン」を選択します。

![デプロイメント・パイプラインの作成](./images/31.png)

「パイプラインの作成」ボタンをクリックします。

![デプロイメント・パイプラインの作成](./images/32.png)

以下の設定を行い、「パイプラインの作成」ボタンをクリックします。

- パイプライン名：deploy-to-oke

![デプロイメント・パイプラインの作成](./images/33.png)

![デプロイメント・パイプラインの作成](./images/32.png)

「ステージの追加」ボタンをクリックします。

![デプロイメント・パイプラインの作成](./images/34.png)

「ブルー/グリーン戦略」を選択して、「次」ボタンをクリックします。

![デプロイメント・パイプラインの作成](./images/35.png)

![デプロイメント・パイプラインの作成](./images/04.png)

以下の設定を行い、「アーティファクトの選択」ボタンをクリックします。

- デプロイメント・タイプ：OKE
- ステージ名：deploy-strategy-bg
- 環境：blue-green-cluster
- ネームスペースA：blue
- ネームスペースB：green
- NGINXイングレス名：helloworld-ing

![デプロイメント・パイプラインの作成](./images/36.png)

「artifact-repository-bg」を選択して、「変更の保存」ボタンをクリックします。

![デプロイメント・パイプラインの作成](./images/37.png)

![デプロイメント・パイプラインの作成](./images/38.png)

「次」ボタンをクリックします。

![デプロイメント・パイプラインの作成](./images/04.png)

以下の設定を行い、「追加」ボタンをクリックします。

- 承認制御：有効
- ステージ名：confirm

![デプロイメント・パイプラインの作成](./images/39.png)

![デプロイメント・パイプラインの作成](./images/23.png)

完了と表示されることを確認して、「閉じる」ボタンをクリックします。

![デプロイメント・パイプラインの作成](./images/40.png)

![デプロイメント・パイプラインの作成](./images/41.png)

パンくずリストから「blue-green」を選択します。

![デプロイメント・パイプラインの作成](./images/42.png)

#### 6. ビルド・パイプラインの作成

ビルド・パイプラインを作成します。  
左メニューから「ビルド・パイプライン」を選択します。

![ビルド・パイプラインの作成](./images/43.png)

「ビルド・パイプラインの作成」ボタンをクリックします。

![ビルド・パイプラインの作成](./images/44.png)

以下の設定を行い、「作成」ボタンをクリックします。

- 名前：image-build-ship

![ビルド・パイプラインの作成](./images/45.png)

![ビルド・パイプラインの作成](./images/15.png)

「ステージの追加」ボタンをクリックします。

![ビルド・パイプラインの作成](./images/34.png)

「マネージド・ビルド」を選択して、「次」ボタンをクリックします。

![ビルド・パイプラインの作成](./images/46.png)

![ビルド・パイプラインの作成](./images/04.png)

以下の設定を行い、「選択」ボタンをクリックします。

- ステージ名：container-image-build
- ビルド仕様ファイル・パス：build_spec.yaml

![ビルド・パイプラインの作成](./images/47.png)

以下の設定を行い、「選択」ボタンをクリックします。

- ソース：接続タイプ：OCIコード・リポジトリ
- strategy-blue-grenn にチェックを入れる
- ブランチの選択：main

![ビルド・パイプラインの作成](./images/48.png)

![ビルド・パイプラインの作成](./images/26.png)

「追加」ボタンをクリックします。

![ビルド・パイプラインの作成](./images/23.png)

「+」をクリックして、「ステージの追加」を選択します。

![ビルド・パイプラインの作成](./images/49.png)

「アーティファクトの配信」を選択して、「次」ボタンをクリックします。

![ビルド・パイプラインの作成](./images/50.png)

![ビルド・パイプラインの作成](./images/04.png)

以下の設定を行い、「アーティファクトの選択」ボタンをクリックします。

- ステージ名：container-image-ship

![ビルド・パイプラインの作成](./images/51.png)

「ocir-repository-bg」にチェックを入れて、「追加」ボタンをクリックします。

![ビルド・パイプラインの作成](./images/52.png)

![ビルド・パイプラインの作成](./images/53.png)

「ビルド構成/結果アーティファクト名」に「bg_image」と設定を行い、「追加」ボタンをクリックします。

![ビルド・パイプラインの作成](./images/54.png)

![ビルド・パイプラインの作成](./images/23.png)

「+」をクリックして、「ステージの追加」を選択します。

![ビルド・パイプラインの作成](./images/55.png)

「デプロイメントのトリガー」を選択して、「次」ボタンをクリックします。

![ビルド・パイプラインの作成](./images/56.png)

![ビルド・パイプラインの作成](./images/04.png)

以下の設定を行い、「デプロイメント・パイプラインの選択」ボタンをクリックします。

- ステージ名：connect-to-deployment-pipeline

![ビルド・パイプラインの作成](./images/57.png)

「deploy-to-oke」にチェックを入れて、「変更の保存」ボタンをクリックします。

![ビルド・パイプラインの作成](./images/58.png)

![ビルド・パイプラインの作成](./images/38.png)

最後に「追加」ボタンをクリックします。

![ビルド・パイプラインの作成](./images/23.png)

パンくずリストから「blue-green」を選択します。

![ビルド・パイプラインの作成](./images/59.png)

#### 7. トリガーの作成

ソースコード更新後のプッシュをトリガーにBlue-Greenデプロイを実行する設定を行います。  
左メニューから「トリガー」を選択します。

![トリガーの作成](./images/60.png)

「トリガーの作成」ボタンをクリックします。

![トリガーの作成](./images/61.png)

以下の設定を行い、「選択」ボタンをクリックします。

- 名前：deploy-strategy-bg
- ソース接続：OCIコード・リポジトリ

![トリガーの作成](./images/62.png)

「strategy-blue-grenn」にチェックを入れて、「変更の保存」ボタンをクリックします。

![トリガーの作成](./images/63.png)

![トリガーの作成](./images/38.png)

「アクションの追加」ボタンをクリックします。

![トリガーの作成](./images/64.png)

「プッシュ」にチェックを入れて、「選択」ボタンをクリックします。

![トリガーの作成](./images/65.png)

「image-build-ship」にチェックを入れて、「選択」ボタンをクリックします。

![トリガーの作成](./images/66.png)

![トリガーの作成](./images/26.png)

「アクションの追加」ボタンをクリックします。

![トリガーの作成](./images/67.png)

最後に、「作成」ボタンをクリックします。

![トリガーの作成](./images/15.png)

以上で完了です。

### Blue-Green Deploy Execute

実際にブルー/グリーンデプロイを実行するために、ソースコードを変更します。

```
cd strategy-blue-green
```
```
vim content.html
```
```
<!DOCTYPE html>
<html lang="ja">
<head>

<meta charset="UTF-8">

<title>Blue-Green</title>

</head>
<body>

<h1>Hello, Blue deploy first!!</h1> #「Blue deploy first!!」に変更

</body>
</html>
```

コード・リポジトリにプッシュします。

```
git add -A .
```
```
git commit -m "first commit"
```
```
[main 03ffea8] first commit
 5 files changed, 785 insertions(+)
 create mode 100644 Dockerfile
 create mode 100644 blue-green-app.yaml
 create mode 100644 build_spec.yaml
 create mode 100644 content.html
 create mode 100644 ingress-controller.yaml
```
```
git branch -M main
```
```
git push -u origin main
```
```
Enumerating objects: 8, done.
Counting objects: 100% (8/8), done.
Delta compression using up to 2 threads
Compressing objects: 100% (7/7), done.
Writing objects: 100% (7/7), 3.79 KiB | 1.89 MiB/s, done.
Total 7 (delta 0), reused 0 (delta 0), pack-reused 0
To https://devops.scmservice.uk-london-1.oci.oraclecloud.com/namespaces/orasejapan/projects/blue-green/repositories/strategy-blue-green
   aca6740..03ffea8  main -> main
Branch 'main' set up to track remote branch 'main' from 'origin'.
```

プッシュ処理をトリガーにビルド・パイプラインが実行されます。  
ビルド・パイプラインの成功後、デプロイメント・パイプラインが実行されます。

![ブルー/グリーンデプロイの実行](./images/68.png)

デプロイメント・パイプラインでは、承認処理をする必要があります。  
ハンバーガーメニューをクリックして、「承認」を選択します。

![ブルー/グリーンデプロイの実行](./images/69.png)

「OK」と入力して、「承認」ボタンをクリックします。

![ブルー/グリーンデプロイの実行](./images/70.png)

デプロイメント・パイプラインが成功したことを確認します。

![ブルー/グリーンデプロイの実行](./images/71.png)

Kubernetesクラスタのblue namespaceにデプロイされていることを確認します。
最初のデプロイは、blue namespaceに行われます。次のデプロイは、green namespaceとなります。その後は、blueからgreenと繰り返される仕様です。

```
kubectl get pods -n blue
```
```
NAME                          READY   STATUS    RESTARTS   AGE
helloworld-5d66bf56f4-b69sz   1/1     Running   0          13m
helloworld-5d66bf56f4-hvvnk   1/1     Running   0          13m
helloworld-5d66bf56f4-pdfbb   1/1     Running   0          13m
```

Ingressに割り当てられたEXTERNAL-IPを確認します。

```
kubectl get ingress -n blue
```
```
NAME             CLASS    HOSTS   ADDRESS           PORTS   AGE
helloworld-ing   <none>   *       132.xxx.xxx.xxx   80      14m
```

ブラウザからアクセスします。

```
http://132.xxx.xxx.xxx/content.html
```

![ブルー/グリーンデプロイの実行](./images/72.png)

再度デプロイを実行して、green namespaceにデプロイされることを確認します。  
ソースコードを変更します。

```
vim content.html
```
```
<!DOCTYPE html>
<html lang="ja">
<head>

<meta charset="UTF-8">

<title>Blue-Green</title>

</head>
<body>

<h1>Hello, Green deploy first!!</h1> #「Green deploy first!!」に変更

</body>
</html>
```

コード・リポジトリにプッシュします。

```
git add -A .
```
```
git commit -m "second commit"
```
```
[main db77643] second commit
 1 file changed, 1 insertion(+), 1 deletion(-)
```
```
git branch -M main
```
```
git push -u origin main
```
```
Enumerating objects: 5, done.
Counting objects: 100% (5/5), done.
Delta compression using up to 2 threads
Compressing objects: 100% (3/3), done.
Writing objects: 100% (3/3), 292 bytes | 292.00 KiB/s, done.
Total 3 (delta 2), reused 0 (delta 0), pack-reused 0
remote: Resolving deltas: 100% (2/2)
To https://devops.scmservice.uk-london-1.oci.oraclecloud.com/namespaces/orasejapan/projects/blue-green/repositories/strategy-blue-green
   5f6b809..db77643  main -> main
Branch 'main' set up to track remote branch 'main' from 'origin'.
```

このプッシュ処理をトリガーに、ビルド・パイプライン、デプロイメント・パイプラインが実行されて、デプロイされます。デプロイメント・パイプラインでは一回目と同様に承認処理を手動で実行します。

```
kubectl get pods -n green
```
```
NAME                          READY   STATUS    RESTARTS   AGE
helloworld-7cd5f9b688-g4sln   1/1     Running   0          5m24s
helloworld-7cd5f9b688-mphp7   1/1     Running   0          5m24s
helloworld-7cd5f9b688-xwfhv   1/1     Running   0          5m24s
```

Ingressに割り当てられたEXTERNAL-IPを確認します。  
※ngress Controllerによりblue namespaceと同じEXTERNAL-IPとなります。

```
kubectl get ingress -n green
```
```
NAME             CLASS    HOSTS   ADDRESS           PORTS   AGE
helloworld-ing   <none>   *       132.xxx.xxx.xxx   80      14m
```

ブラウザからアクセスします。

```
http://132.xxx.xxx.xxx/content.html
```

![ブルー/グリーンデプロイの実行](./images/73.png)

## CANARY Demo Environment

ディレクトリを移動します。

```
cd ../deploy-strategy-canary/
```
```
ls
```
```
build_spec.yaml  canary-app.yaml  content.html  Dockerfile  ingress-controller.yaml
```

事前に取得したサンプルコードをプッシュするために、作成したリポジトリをクローンします。

```
git clone https://devops.scmservice.uk-london-1.oci.oraclecloud.com/namespaces/orasejapan/projects/blue-green/repositories/strategy-blue-green
```

### OKE

以下チュートリアルを参照して、Kubernetesクラスタを構築します。

本でも環境では、3 Node としています。

[Oracle Container Engine for Kubernetes(OKE)をプロビジョニングしよう](https://oracle-japan.github.io/ocitutorials/cloud-native/oke-for-commons/)

1. OKEクラスターのプロビジョニング
2. CLI実行環境(Cloud Shell)の準備

### OCIR

以下チュートリアルを参照して、OCIRにリポジトリを作成します。

[Oracle Cloud Infrastructure(OCI) DevOpsことはじめ-OKE編-](https://oracle-japan.github.io/ocitutorials/cloud-native/devops-for-beginners-oke/)

3. アーティファクト
  - 3-1. OCIRのセットアップ

＜パラメータ＞  
コンパートメント：対象とするコンパートメント  
リポジトリ名：canary  
アクセス：パブリック

### Ingress Controller Setup

サンプルコードを取得します。

```
git clone https://github.com/oracle-japan/devops-deploy-strategy.git
```

ディレクトリの内容を確認します。

```
ls devops-deploy-strategy
```
```
deploy-strategy-bg  deploy-strategy-canary  README.md
```

`deploy-strategy-canary` ディレクトリに移動して、ingress controller をセットアップします。

```
cd deploy-strategy-canary
```
```
kubectl apply -f ingress-controller.yaml 
```
```
namespace/ingress-nginx created
serviceaccount/ingress-nginx created
serviceaccount/ingress-nginx-admission created
role.rbac.authorization.k8s.io/ingress-nginx created
role.rbac.authorization.k8s.io/ingress-nginx-admission created
clusterrole.rbac.authorization.k8s.io/ingress-nginx created
clusterrole.rbac.authorization.k8s.io/ingress-nginx-admission created
rolebinding.rbac.authorization.k8s.io/ingress-nginx created
rolebinding.rbac.authorization.k8s.io/ingress-nginx-admission created
clusterrolebinding.rbac.authorization.k8s.io/ingress-nginx created
clusterrolebinding.rbac.authorization.k8s.io/ingress-nginx-admission created
configmap/ingress-nginx-controller created
service/ingress-nginx-controller created
service/ingress-nginx-controller-admission created
deployment.apps/ingress-nginx-controller created
job.batch/ingress-nginx-admission-create created
job.batch/ingress-nginx-admission-patch created
ingressclass.networking.k8s.io/nginx created
validatingwebhookconfiguration.admissionregistration.k8s.io/ingress-nginx-admission created
```

3個の `ingress-nginx-controller` のステータスが `Running` であることを確認します。

```
kubectl get pods -n ingress-nginx
```
```
NAME                                        READY   STATUS      RESTARTS   AGE
ingress-nginx-admission-create-4jptb        0/1     Completed   0          43m
ingress-nginx-admission-patch-5pch2         0/1     Completed   1          43m
ingress-nginx-controller-8574b6d7c9-4nskl   1/1     Running     0          43m
ingress-nginx-controller-8574b6d7c9-6k5zp   1/1     Running     0          43m
ingress-nginx-controller-8574b6d7c9-hpf62   1/1     Running     0          43m
```

LoadBalancer `ingress-nginx-controller` の EXTERNAL-IP が表示されていることを確認します。

```
kubectl get services -n ingress-nginx
```
```
NAME                                 TYPE           CLUSTER-IP      EXTERNAL-IP       PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.96.107.235   132.xxx.xxx.xxx   80:32487/TCP,443:32238/TCP   44m
ingress-nginx-controller-admission   ClusterIP      10.96.101.214   <none>            443/TCP                      44m
```

### Namespace Create

対象のKubernetesクラスタに、CANARY用のNamespaceを作成します。

```
kubectl create ns canary
```
```
namespace/green created
```

canaryのNamespaceができていること確認します。

```
kubectl get ns
```
```
NAME              STATUS   AGE
blue              Active   11m
canary            Active   10s
default           Active   5h58m
green             Active   11m
ingress-nginx     Active   153m
kube-node-lease   Active   5h58m
kube-public       Active   5h58m
kube-system       Active   5h58m
```

### OCI DevOps Setup