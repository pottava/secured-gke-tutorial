# セキュアな GKE クラスタ 作成

<walkthrough-watcher-constant key="region" value="asia-northeast1"></walkthrough-watcher-constant>
<walkthrough-watcher-constant key="zone" value="asia-northeast1-c"></walkthrough-watcher-constant>
<walkthrough-watcher-constant key="vpc-dmz" value="dmz"></walkthrough-watcher-constant>
<walkthrough-watcher-constant key="subnet-dmz" value="dmz"></walkthrough-watcher-constant>
<walkthrough-watcher-constant key="vpc-internal" value="internal"></walkthrough-watcher-constant>
<walkthrough-watcher-constant key="subnet-internal" value="internal"></walkthrough-watcher-constant>
<walkthrough-watcher-constant key="firewall-iap" value="allow-ingress-from-iap"></walkthrough-watcher-constant>
<walkthrough-watcher-constant key="firewall-bastion" value="allow-ingress-from-bastions"></walkthrough-watcher-constant>
<walkthrough-watcher-constant key="svc-account" value="app-permissions"></walkthrough-watcher-constant>
<walkthrough-watcher-constant key="bastion-linux" value="bastion-linux"></walkthrough-watcher-constant>
<walkthrough-watcher-constant key="network-tag" value="dmz-bastions"></walkthrough-watcher-constant>
<walkthrough-watcher-constant key="gke-cluster" value="secured-cluster"></walkthrough-watcher-constant>

## 始めましょう

VPC 内のプライベート接続を始め、よりセキュアな GKE を利用するための手順です。

各手順においては、複数の選択肢が併記されているステップもあります。要件に応じて、より適切なコマンドを選びながらチュートリアルを進めてみてください。

**所要時間**: 約 30 分

**前提条件**: GCP アカウント

**[開始]** ボタンをクリックして次のステップに進みます。

## プロジェクトの設定

この手順の中で実際にリソースを構築する対象のプロジェクトを選択してください。

<walkthrough-project-billing-setup permissions="compute.instances.create"></walkthrough-project-billing-setup>

## CLI の設定

本チュートリアルをコマンドラインからも適切に実行できるよう、デフォルト設定を実施します。

```bash
gcloud config set compute/region {{region}}
gcloud config set compute/zone {{zone}}
```

チュートリアルで利用するいくつかの API を有効化します。

```bash
gcloud services enable container.googleapis.com containerregistry.googleapis.com cloudbuild.googleapis.com
```

## VPC ネットワークへの移動

画面の左上にある <walkthrough-spotlight-pointer spotlightId="console-nav-menu">ナビゲーション メニュー</walkthrough-spotlight-pointer> を開きます。

**VPC ネットワーク** セクションを開きましょう。

<walkthrough-menu-navigation sectionId="VIRTUAL_NETWORK_SECTION"></walkthrough-menu-navigation>

Compute Engine API の API が無効の場合は有効化しましょう。

## VPC の作成

[Private Google Access](https://cloud.google.com/vpc/docs/private-access-options#pga) を有効にした **DMZ 用 VPC** を作成します。

コンソールからの作成する場合

<walkthrough-spotlight-pointer cssSelector="a[cfciamcheck='compute.networks.create']">VPC ネットワークを作成</walkthrough-spotlight-pointer> をクリックしてください。

* VPC の名前: `{{vpc-dmz}}`
* カスタムサブネットの名前: `{{subnet-dmz}}`
* カスタムサブネットのリージョン: `{{region}}`
* IP アドレス範囲: `10.128.0.0/16`
* 限定公開の Google アクセス: `オン`

まで入力できたら

* <walkthrough-spotlight-pointer cssSelector="button[type='submit']">作成</walkthrough-spotlight-pointer> をクリックします。

CLI から作成するには、以下のコマンドを実行します。

```bash
gcloud compute networks create {{vpc-dmz}} --subnet-mode=custom --bgp-routing-mode=regional
```

```bash
gcloud compute networks subnets create {{subnet-dmz}} --range=10.128.0.0/16 --network={{vpc-dmz}} --region={{region}} --enable-private-ip-google-access
```

## VPC の作成

同様に **プライベートな VPC** を作成します。

<walkthrough-spotlight-pointer cssSelector="a[cfciamcheck='compute.networks.create']">VPC ネットワークを作成</walkthrough-spotlight-pointer> をクリックしてください。

* VPC の名前: `{{vpc-internal}}`
* カスタムサブネットの名前: `{{subnet-internal}}`
* カスタムサブネットのリージョン: `{{region}}`
* IP アドレス範囲: `10.129.0.0/16`
* 限定公開の Google アクセス: `オン`

まで入力できたら

* <walkthrough-spotlight-pointer cssSelector="button[type='submit']">作成</walkthrough-spotlight-pointer> をクリックします。

CLI から作成するには、以下のコマンドを実行します。

```bash
gcloud compute networks create {{vpc-internal}} --subnet-mode=custom --bgp-routing-mode=regional
```

```bash
gcloud compute networks subnets create {{subnet-internal}} --range=10.19.0.0/16 --network={{vpc-internal}} --region={{region}} --enable-private-ip-google-access
```

## ファイアウォールの設定

DMZ に対して [Identity-Aware Proxy](https://cloud.google.com/iap) からの接続を許可します。

左側のメニューから <walkthrough-spotlight-pointer cssSelector="#cfctest-section-nav-item-firewall">ファイアウォール</walkthrough-spotlight-pointer> を開き、
<walkthrough-spotlight-pointer cssSelector="button[cfciamcheck='compute.firewalls.create']">ファイアウォール ルールを作成</walkthrough-spotlight-pointer> をクリックします。

* 名前: `{{firewall-iap}}`
* ネットネットワーク: `{{vpc-dmz}}`
* ターゲット: `ネットワーク上のすべてのインスタンス`
* ソースフィルタ: `IP 範囲`
* ソース IP の範囲: `35.235.240.0/20`
* 指定したプロトコルとポート: `TCP: 22, 443, 3389`

まで入力できたら <walkthrough-spotlight-pointer cssSelector="button[type='submit']">作成</walkthrough-spotlight-pointer> をクリックします。
もしくは CLI から以下のコマンドを実行します。

```bash
gcloud compute firewall-rules create {{firewall-iap}} --network={{vpc-dmz}} --direction=INGRESS --priority=1000 --action=ALLOW --rules=tcp:22,tcp:443,tcp:3389 --source-ranges=35.235.240.0/20
```

## ファイアウォールの設定

プライベートな VPC に対して DMZ 上の踏み台サーバからの接続を許可します。

<walkthrough-spotlight-pointer cssSelector="button[cfciamcheck='compute.firewalls.create']">ファイアウォール ルールを作成</walkthrough-spotlight-pointer> をクリックします。

* 名前: `{{firewall-bastion}}`
* ネットネットワーク: `{{vpc-internal}}`
* ターゲット: `ネットワーク上のすべてのインスタンス`
* ソースフィルタ: `ソースタグ`
* ソースタグ: `{{network-tag}}`
* 指定したプロトコルとポート: `TCP: 22, 443, 3389`

まで入力できたら <walkthrough-spotlight-pointer cssSelector="button[type='submit']">作成</walkthrough-spotlight-pointer> をクリックします。
もしくは CLI から以下のコマンドを実行します。

```bash
gcloud compute firewall-rules create {{firewall-bastion}} --network={{vpc-internal}} --direction=INGRESS --priority=1000 --action=ALLOW --rules=tcp:22,tcp:443,tcp:3389 --source-tags=dmz-bastions
```

## ハイブリッド接続への移動

**dmz** VPC と **internal** VPC を Cloud VPN で接続します。

画面の左上にある <walkthrough-spotlight-pointer spotlightId="console-nav-menu">ナビゲーション メニュー</walkthrough-spotlight-pointer> を開きます。

**ハイブリッド接続** セクションを開きましょう。

<walkthrough-menu-navigation sectionId="INTERCONNECT_SECTION"></walkthrough-menu-navigation>

## クラウド ルーターの設定

左側のメニューから <walkthrough-spotlight-pointer cssSelector="#cfctest-section-nav-item-router">クラウドルーター</walkthrough-spotlight-pointer> を開きます。

<walkthrough-spotlight-pointer cssSelector="a[cfciamcheck='compute.routers.create']">ルーターを作成</walkthrough-spotlight-pointer> をクリックし、

* 名前: `{{vpc-dmz}}-router`
* ネットワーク: `{{vpc-dmz}}`
* リージョン: `{{region}}`
* Google ASN: `65001`

まで入力できたら、<walkthrough-spotlight-pointer cssSelector="button[type='submit']">作成</walkthrough-spotlight-pointer> をクリックします。同様に以下の設定値で {{vpc-internal}} 用にも作成します。

* 名前: `{{vpc-internal}}-router`
* ネットワーク: `{{vpc-internal}}`
* リージョン: `{{region}}`
* Google ASN: `65002`

CLI から作成するには、以下のコマンドを実行します。

```bash
gcloud compute routers create {{vpc-dmz}}-router --network={{vpc-dmz}} --region={{region}} --asn=65001
gcloud compute routers create {{vpc-internal}}-router --network={{vpc-internal}} --region={{region}} --asn=65002
```

## VPN ゲートウェイの設定

左側のメニューから <walkthrough-spotlight-pointer cssSelector="#cfctest-section-nav-item-vpn">VPN</walkthrough-spotlight-pointer> を開きます。

<walkthrough-spotlight-pointer cssSelector="gs-zero-state-button[permission='compute.targetVpnGateways.create'] span">VPN 接続を作成</walkthrough-spotlight-pointer> をクリックします。

* VPN オプション: `高可用性 (HA) VPN`

<walkthrough-spotlight-pointer cssSelector="button[type='submit']">続行</walkthrough-spotlight-pointer> をクリックし、

* 名前: `{{vpc-dmz}}-vpn-gateway`
* VPC ネットワーク: `{{vpc-dmz}}`
* リージョン: `{{region}}`

と入力できたら、<walkthrough-spotlight-pointer cssSelector="button[type='submit']">作成して続行</walkthrough-spotlight-pointer> をクリックします。同様に対向の VPN ゲートウェイを以下の設定で作成します。

* 名前: `{{vpc-internal}}-vpn-gateway`
* VPC ネットワーク: `{{vpc-internal}}`
* リージョン: `{{region}}`

または CLI から以下のコマンドを実行します。

```bash
gcloud compute vpn-gateways create {{vpc-dmz}}-vpn-gateway --network={{vpc-dmz}} --region={{region}}
gcloud compute vpn-gateways create {{vpc-internal}}-vpn-gateway --network={{vpc-internal}} --region={{region}}
```

## VPN トンネルの設定

[事前共有鍵を生成](https://cloud.google.com/network-connectivity/docs/vpn/how-to/generating-pre-shared-key?hl=ja)し、

```bash
shared_key=$(head -c 24 /dev/urandom | base64)
echo ${shared_key}
```

CLI から以下のコマンドを実行します。

```bash
gcloud compute vpn-tunnels create {{vpc-dmz}}-vpn-tunnel0 --vpn-gateway={{vpc-dmz}}-vpn-gateway --interface=0 --shared-secret=${shared_key} --peer-gcp-gateway={{vpc-internal}}-vpn-gateway --ike-version=2 --router={{vpc-dmz}}-router --router-region={{region}}
gcloud compute vpn-tunnels create {{vpc-dmz}}-vpn-tunnel1 --vpn-gateway={{vpc-dmz}}-vpn-gateway --interface=1 --shared-secret=${shared_key} --peer-gcp-gateway={{vpc-internal}}-vpn-gateway --ike-version=2 --router={{vpc-dmz}}-router --router-region={{region}}
gcloud compute vpn-tunnels create {{vpc-internal}}-vpn-tunnel0 --vpn-gateway={{vpc-internal}}-vpn-gateway --interface=0 --shared-secret=${shared_key} --peer-gcp-gateway={{vpc-dmz}}-vpn-gateway --ike-version=2 --router={{vpc-internal}}-router --router-region={{region}}
gcloud compute vpn-tunnels create {{vpc-internal}}-vpn-tunnel1 --vpn-gateway={{vpc-internal}}-vpn-gateway --interface=1 --shared-secret=${shared_key} --peer-gcp-gateway={{vpc-dmz}}-vpn-gateway --ike-version=2 --router={{vpc-internal}}-router --router-region={{region}}
```

## BGP セッションの構成

4 つのトンネルに対し、それぞれ [BGP セッションを確立](https://cloud.google.com/network-connectivity/docs/vpn/how-to/creating-ha-vpn?hl=ja#creating-ha-gw-peer-tunnel) します。
VPN コンソールの画面をリロードし、すべての接続が正常であることを確認しましょう。

## セキュリティへの移動

画面の左上にある <walkthrough-spotlight-pointer spotlightId="console-nav-menu">ナビゲーション メニュー</walkthrough-spotlight-pointer> を開きます。

**セキュリティ** セクションを開きましょう。

<walkthrough-menu-navigation sectionId="SECURITY_SECTION"></walkthrough-menu-navigation>

## Identity-Aware Proxy (IAP) の設定

左側のメニューから <walkthrough-spotlight-pointer cssSelector="#cfctest-section-nav-item-security_gatekeeper">Identity-Aware Proxy</walkthrough-spotlight-pointer> を開き、`API を有効にする` をクリックします。

## Access Context (IAP への接続条件) の設定

左側のメニューから <walkthrough-spotlight-pointer cssSelector="#cfctest-section-nav-item-security_access_context_manager">Access Context Manager</walkthrough-spotlight-pointer> を開き、設定対象の組織を選択します。

一覧画面が表示されたら <walkthrough-spotlight-pointer cssSelector="#_3rif_new-level-button">新規</walkthrough-spotlight-pointer> をクリックします。

**新しいアクセスレベル** として

* IP サブネットワーク
* 使用可能なリージョン
* デバイス ポリシー

などを設定し、 <walkthrough-spotlight-pointer cssSelector="#_3rif_save-button">保存</walkthrough-spotlight-pointer> をクリックします。

## VPC Service Controls の設定

左側のメニューから <walkthrough-spotlight-pointer cssSelector="#cfctest-section-nav-item-security_vpc_service_controls">VPC Service Controls</walkthrough-spotlight-pointer> を開き、設定対象の組織を選択します。

*新しい境界* をクリックし

* 境界名: `default_perimeter`
* プロジェクト: `{{project-id}}`
* 制限付きサービス: `Google Cloud Storage API, Google Compute Engine API, Google Container Registry API, Google Kubernetes Engine API, Stackdriver Logging API`
* VPC のアクセス可能なサービス: `すべての制限付きサービス`
* アクセスレベル: `この前の手順で作成したアクセスレベル`

入力できたら `境界を作成` をクリックします。

## サンプル アプリケーションの作成

作業ディレクトリを作り、Node.js アプリケーションを配置します。

```
mkdir sample-hello && cd $_
cat << EOF > server.js
const http = require('http');
const os = require('os');
const handler = function(request, response) {
  response.writeHead(200);
  response.end("<h1>Hello World! == " + os.hostname() + "</h1>\n");
}
http.createServer(handler).listen(8080);
EOF
```

サーバーをローカルで起動してみます。

```bash
docker run --name node -d -v $(pwd):/work -p 8080:8080 node:12.18.4-slim node /work/server.js
```

[ウェブでプレビュー](https://cloud.google.com/shell/docs/using-web-preview?hl=ja) してみて問題がなければ、コンテナを停止します。

```bash
docker rm -f node
```

## Docker イメージの作成

Dockerfile を作成します。

```
cat << EOF > Dockerfile
FROM node:12.18.4-slim
EXPOSE 8080
COPY server.js /server.js
CMD node /server.js
EOF
```

ビルドしましょう。

```bash
docker build -t asia.gcr.io/{{project-id}}/hello:v1 .
```

GCR へログインします。

```bash
gcloud auth print-access-token | docker login -u oauth2accesstoken --password-stdin asia.gcr.io
```

イメージをプッシュし、正しく保存されたか確認してみましょう。

```bash
docker push asia.gcr.io/{{project-id}}/hello:v1
gcloud container images list-tags asia.gcr.io/{{project-id}}/hello
```

## Kubernetes Engine への移動

画面の左上にある <walkthrough-spotlight-pointer spotlightId="console-nav-menu">ナビゲーション メニュー</walkthrough-spotlight-pointer> を開きます。

**Kubernetes Engine** セクションを開きましょう。

<walkthrough-menu-navigation sectionId="KUBERNETES_SECTION"></walkthrough-menu-navigation>

## GKE クラスタの作成

<walkthrough-spotlight-pointer cssSelector="gs-zero-state-button[permission='container.clusters.create'] a">クラスタを作成</walkthrough-spotlight-pointer> クリックします。

* 名前: `{{gke-cluster}}`
* ゾーン: `{{zone}}`
* マスターのバージョン: `リリース チャンネル: Regular チャンネル`
* ノードプール
  * `ノード数:2, 自動スケーリング: 1~3, サージ アップグレード: 最大 1, オフライン上限 0`
  * ノード: `OS: COS, マシンタイプ: e2-small`
  * セキュリティ: `シールド オプション: セキュアブートを有効`
* ネットワーキング
  * 限定公開クラスタ
  * 外部 IP アドレスを使用したマスターへのアクセス: `オフ`
  * マスター IP 範囲: `10.127.0.0/28`
  * ネットワーク: `{{subnet-dmz}}`
  * ノードのサブネット: `{{subnet-dmz}}`
* セキュリティ
  * シールドされた GKE ノード: `有効`
  * Workload Identity: `有効`

を設定し、 <walkthrough-spotlight-pointer spotlightId="gke-cluster-create-button">作成</walkthrough-spotlight-pointer> をクリックします。
もしくは CLI から以下のコマンドを実行します。

```bash
gcloud beta container clusters create "{{gke-cluster}}" --zone "{{zone}}" --release-channel "regular" --machine-type "e2-small" --image-type "COS" --disk-type "pd-standard" --disk-size "100" --num-nodes "2" --enable-stackdriver-kubernetes --enable-private-nodes --enable-private-endpoint --enable-master-authorized-networks --master-ipv4-cidr "10.127.0.0/28" --no-enable-basic-auth --no-issue-client-certificate --enable-ip-alias --network "projects/{{project-id}}/global/networks/{{vpc-dmz}}" --subnetwork "projects/{{project-id}}/regions/{{region}}/subnetworks/{{subnet-dmz}}" --default-max-pods-per-node "110" --enable-autoscaling --min-nodes "1" --max-nodes "3" --addons HttpLoadBalancing --enable-autoupgrade --enable-autorepair --max-surge-upgrade 1 --max-unavailable-upgrade 0 --workload-pool "{{project-id}}.svc.id.goog" --enable-shielded-nodes --shielded-secure-boot --metadata disable-legacy-endpoints=true
```

## Compute Engine への移動

画面の左上にある <walkthrough-spotlight-pointer spotlightId="console-nav-menu">ナビゲーション メニュー</walkthrough-spotlight-pointer> を開きます。

**Compute Engine** セクションを開きましょう。

<walkthrough-menu-navigation sectionId="COMPUTE_SECTION"></walkthrough-menu-navigation>

## 踏み台サーバーの作成

<walkthrough-spotlight-pointer spotlightId="gce-zero-new-vm">作成</walkthrough-spotlight-pointer> クリックします。

* 名前: `{{bastion-linux}}`
* ゾーン: `{{zone}}`
* マシンシリーズ: `N1`
* マシンタイプ: `g1-small`
* ブートディスク: `ディスクサイズ: 100GB`
* 管理: `メタデータ: enable-oslogin=TRUE`
* ネットワーキング: `ネットワーク タグ: {{network-tag}}, ネットワーク インターフェイス > ネットワーク: {{subnet-dmz}}`

を設定し、 <walkthrough-spotlight-pointer spotlightId="gce-submit-button">作成</walkthrough-spotlight-pointer> をクリックします。
もしくは CLI から以下のコマンドを実行します。

```bash
gcloud compute instances create {{bastion-linux}} --machine-type=g1-small --subnet={{subnet-dmz}} --zone={{zone}} --network-tier=STANDARD --metadata=enable-oslogin=TRUE --boot-disk-size=100GB --boot-disk-type=pd-standard --boot-disk-device-name={{bastion-linux}} --tags={{network-tag}}
```

## 踏み台からの GKE 操作

kubectl での操作ができるよう kubeconfig を取得します。

```bash
KUBECONFIG=./kubecfg gcloud container clusters get-credentials "{{gke-cluster}}" --zone={{zone}} --internal-ip
```

kubeconfig を転送しつつ、踏み台へ SSH 接続します。

```bash
gcloud compute scp kubecfg {{bastion-linux}}:~ --zone={{zone}} --tunnel-through-iap
gcloud compute ssh {{bastion-linux}} --zone={{zone}} --tunnel-through-iap
```

踏み台に設定置き場をつくり、gclould コマンドをインストールします。

```bash
mkdir -p $HOME/gcloud
```

```
cat << EOF >> ~/.bashrc
gcloud() {
  sudo docker run --rm -it -v \$(pwd):\$(pwd) -v \$HOME:\$HOME \\
    -v \$HOME/gcloud:/.config/gcloud \\
    -v /etc/passwd:/etc/passwd:ro -v /etc/group:/etc/group:ro \\
    -v /mnt:/mnt -w \$(pwd) -u \$(id -u):\$(id -g) \\
    gcr.io/google.com/cloudsdktool/cloud-sdk:311.0.0-alpine \\
    gcloud "\$@";
}
EOF
source ~/.bashrc
```

`project_number` を先程の番号に変換しつつ、VM に設定されたサービスアカウントで認証を通します。

```bash
gcloud config set account "<project_number>-compute@developer.gserviceaccount.com"
gcloud config set project {{project-id}}
gcloud auth list
curl -H "Metadata-Flavor: Google" "http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/token"
```

## IAM と管理への移動

画面の左上にある <walkthrough-spotlight-pointer spotlightId="console-nav-menu">ナビゲーション メニュー</walkthrough-spotlight-pointer> を開きます。

**IAM と管理** セクションを開きましょう。

<walkthrough-menu-navigation sectionId="IAM_ADMIN_SECTION"></walkthrough-menu-navigation>

## サービスアカウントの作成

左側のメニューから <walkthrough-spotlight-pointer cssSelector="#cfctest-section-nav-item-serviceaccounts">サービスアカウント</walkthrough-spotlight-pointer> を開きます。

<walkthrough-spotlight-pointer cssSelector="button[cfciamcheck='iam.serviceAccounts.create']">サービスアカウントを作成</walkthrough-spotlight-pointer> をクリックし

* サービス アカウント名: `{{svc-account}}`
* サービス アカウントの説明: `Allow GKE apps to access GCS`

を入力したら *作成* をクリックします。もしくは CLI から以下のコマンドを実行します。

```bash
gcloud iam service-accounts create {{svc-account}} --display-name="{{svc-account}}" --description="Allow GKE apps to access GCS"
```

## サービスアカウントへの権限付与

左側のメニューから <walkthrough-spotlight-pointer cssSelector="#cfctest-section-nav-item-iam">IAM</walkthrough-spotlight-pointer> を開きます。

<walkthrough-spotlight-pointer spotlightId="iam-add-member">追加</walkthrough-spotlight-pointer> をクリックし

* 新しいメンバー: `{{svc-account}}@{{project-id}}.iam.gserviceaccount.com`
* ロール: `Storage オブジェクト作成者`

を入力したら *作成* をクリックします。もしくは CLI から以下のコマンドを実行します。

```bash
gcloud projects add-iam-policy-binding "{{project-id}}" --member "serviceAccount:$( gcloud iam service-accounts list \
        --filter="displayName:{{svc-account}}" --format 'value(email)' )" --role roles/storage.objectCreator
```

## 


## これで終わりです

<walkthrough-conclusion-trophy></walkthrough-conclusion-trophy>

すべて完了しました。

**作業後は忘れずにクリーンアップする**: テスト プロジェクトを作成した場合は、不要な料金の発生を避けるためにプロジェクトを削除してください。

```bash
gcloud projects delete {{project-id}}
```
