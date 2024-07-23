---
title: "Elastic Cloud on Kubernetes (ECK)でSaaSのアプリケーションログ監視環境を構築した話をなるべく詳しく書く"
emoji: "🔭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["kubernetes","fluentd","elasticsearch","kibana"]
published: false
---

# はじめに
KubernetesでB2B SaaSプロダクトを運用するにあたり、一時間で十数万条に上る膨大な量のアプリケーションログを効果的に収集して検索や分析を行う手段が必要でした。

そこで、ログ収集エージェントのfluentdをプロダクトのKubernetesクラスタに配置し、エージェントが収集したアプリケーションログを監視および可視化するために、ECKを用いて新たにElastic Stackを Linode Kubernetes Engine 上に構築しました。

本記事では、fluentdによるログの収集およびElasticSearchへの送信、Elastic Stackの構築およびKibanaダッシュボードのIngressによる公開と簡略なSSL認証までカバーします。

## ECKとは？
**ECK**(**E**lastic **C**loud on **K**ubernetes)はElasticSearchやKibanaなどを含むElastic StackのCRD/Operatorで、KubernetesへのElastic Stackのデプロイと管理を効率化します。

具体的には、ElasticSearchやKibana、APM serverなど個々のアプリケーションのカスタムリソースを定義し、定義されたリソースのタイプに基づいてアプリケーションの管理を自動化します。
:::message
(24/07/23)現在、公式に推奨されるElastic StackのKubernetesへのデプロイ方法です。
:::

## 動作環境
#### 収集側クラスタ（SaaSプロダクション）
- LKE (Linode Kubernetes Engine) Dedicated 32GB
:::message
実際にデプロイする収集用エージェントのfluentdはメモリ使用量~200mbほどの小さいアプリケーションであるため、大抵の環境で動作できます。そのため、ここでは収集側クラスタのスペックは明記しません。
:::
#### Elastic Stack用クラスタ
- LKE (Linode Kubernetes Engine) Dedicated 8GB plan x 1
  - 2 CPU
  - 8GB RAM
  - 160GB Storage
  - 5TB Transfer
  - 40/5Gbps Network In/Out
- Linode Block Storage（従量課金制）
  - 受信したログを保存するために使用

:::message alert
LKEを利用している場合、Dedicated 8GB以下のプランはリソース不足でElastic Stackが起動できません。可能であれば、Dedicated 8GB x3台の構成が望ましいと思われます。
:::
# 1.Elastic Stackの構築
:::message
(**24/07/23**)現在、最新のバージョンである8.14.3をデプロイします。サポートされているKubernetesバージョンなどはこちら(https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-quickstart.html)を参照してください。
:::

## 1.1 前提のインストール
まずは公式ガイドを参考して、カスタムリソース(CRD)をインストールします。

```bash
kubectl create -f https://download.elastic.co/downloads/eck/2.13.0/crds.yaml
```
```
customresourcedefinition.apiextensions.k8s.io/agents.agent.k8s.elastic.co created
customresourcedefinition.apiextensions.k8s.io/apmservers.apm.k8s.elastic.co created
customresourcedefinition.apiextensions.k8s.io/beats.beat.k8s.elastic.co created
customresourcedefinition.apiextensions.k8s.io/elasticmapsservers.maps.k8s.elastic.co created
customresourcedefinition.apiextensions.k8s.io/elasticsearches.elasticsearch.k8s.elastic.co created
customresourcedefinition.apiextensions.k8s.io/enterprisesearches.enterprisesearch.k8s.elastic.co created
customresourcedefinition.apiextensions.k8s.io/kibanas.kibana.k8s.elastic.co created
customresourcedefinition.apiextensions.k8s.io/logstashes.logstash.k8s.elastic.co created
```
次に、OperatorとRBACルールをインストールします。
```bash
kubectl apply -f https://download.elastic.co/downloads/eck/2.13.0/operator.yaml
```
## 1.2 ElasticSearchとKibanaのインストール

## 1.3 Ingress Controllerを使用してKibanaを公開する

先ほど`kubectl port-forwarding`でKibanaの動作確認ができましたが、現在の状態では外部からアクセスできません。というのも、`kubectl get svc`で確認すると、
```
NAME                                 TYPE           CLUSTER-IP       EXTERNAL-IP     PORT(S)          AGE
logging-dashboard-es-default         ClusterIP      None             <none>          9200/TCP         41h
logging-dashboard-es-http            ClusterIP      10.128.5.221     <none>          9200/TCP         41h
logging-dashboard-es-internal-http   ClusterIP      10.128.238.88    <none>          9200/TCP         41h
logging-dashboard-es-transport       ClusterIP      None             <none>          9300/TCP         41h
logging-dashboard-kb-http            ClusterIP      10.128.185.177   <none>          5601/TCP         41h
```
外部からアクセスしたいKibanaのServiceである`logging-dashboard-kb-http`はタイプがクラスタの内部通信に使われる`ClusterIP`で、`EXTERNAL-IP`が空欄になっていていることから公開IPアドレスが割り当てられていないことがわかります。

このServiceの外部アクセスを可能するには`LoadBalancer`タイプのServiceを新たに作る手法もありますが、今回は`Ingress`および`Ingress-Controller`を用いて適切に外部トラフィックを内部Serviceにルーティングする方法を取ります。

同時に、cert-managerを利用してTLS証明書の発行および管理を自動的に行うシステムの構築を行います。一般的なWebサービスはHTTPSプロトコルを使用します。当然、Kibanaもその例外ではなく、必然的にTLS署名が必要となります。そこで署名や管理を自動で行えるcert-managerは必要不可欠ですし、この記事でも軽く使い方を説明します。

### 1.3.1 Ingress Controllerをインストールする
`Ingress`のServiceを扱うにはIngress Controllerが必要です。Ingress Controllerは多数公開されていますが、この場合はIngress Nginx Controller(https://kubernetes.github.io/ingress-nginx/deploy/)を使用します。
:::message
ここではhelmを使用してインストールしていますが、他の方法でインストールすることも可能です。詳しくは公式ドキュメントをご参照ください。
:::
以下のコマンドでインストールします。
```
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```
これで、`ingress-nginx`namespaceにIngress Nginx Controllerがインストールされました。
`kubectl get svc -n ingress-nginx`で確認してみましょう。

```
NAME                                 TYPE           CLUSTER-IP       EXTERNAL-IP    PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.128.229.175   xx.xx.xxx.xx   80:30350/TCP,443:31867/TCP   43h
ingress-nginx-controller-admission   ClusterIP      10.128.142.155   <none>         443/TCP                      43h
```
`ingress-nginx-controller`の`EXTERNAL-IP`の値は後ほど使用します。

### 1.3.2 cert-managerとLet's EncryptでKibanaのTLS認証を行う

CA機関によるTLS認証に必要な証明書の発行は高額な費用がかかる場合があります。

ですが、現在はLet's Encryptが無料、オープンかつ自動化された証明書の発行サービスを提供しているので、cert-managerで自動的にLet's Encrypt(HTTP01 Challenge)による証明書の発行と期限切れの証明書の更新を行えるように設定します。

まずはcert-managerをインストールします。すでにインストールされている方は次のステップにお進みください。

:::message
ここではhelmを使用してインストールしていますが、`yaml`マニフェストを利用してインストールすることもできます。詳しくはcert-managerの公式ドキュメントを参照してください。
:::

```bash
$ helm repo add jetstack https://charts.jetstack.io --force-update
>> "jetstack" has been added to your repositories

$ helm repo update
>> Successfully got an update from the "jetstack" chart repository

$ helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.15.1 \
  --set crds.enabled=true
>> cert-manager v1.15.1 has been deployed successfully!
```
次に、`ClusterIssuer`を作成します。深入りはしませんが、cert-managerのIssuerには二種類あり、ここではクラスタレベルのスコープで使用できるClusterIssuerを使います。

注意すべきパラメーターを簡単に説明します。
- `acme.email`:自分のメールアドレスを記入してください。
- `acme.server`:使用するacmeのサーバー。ここではLet's EncryptのProd(Production)サーバーを使用していますが、リクエスト回数に制限があるので、テスト環境ではLet's EncryptのTest環境(https://acme-staging-v02.api.letsencrypt.org/directory)をお勧めします。

:::message alert
Let's Encryptのテスト環境にて発行される証明書は発行側が完全な情報を記述していないので正式に使用できません。あくまでテスト用としてご利用ください。
:::
- `acme.solvers[0].http01.ingress.class`:使用しているIngress Controllerの値に置き換えてください。ここでは`nginx`になります。これでcert-managerはport 80にingressを配置し、acmeサーバーにHTTP-01認証をさせることができます。

パラメーターの調整が終わりましたらapplyしましょう。
```bash
kubectl apply -f clusterissuer.yaml
```
ついでに確認もしてみます。
```bash
$ kubectl get ClusterIssuer

>>letsencrypt-prod   True    42h
```

# KibanaをIngressで公開する
最後の手順です。以下のファイルで`ingress`を作成します。

注意すべきパラメーターを以下に説明します。
- `your domain`:ドメイン名を入力してください。もし手持ちのドメインがありませんでしたら、テスト用途に`nip.io`を利用できます。先ほど取得した`ingress-nginx-controller`Serviceの`External-IP`の値を`xx.xx.xxx.xx.nip.io`のように当てはめることでDNSルーティングができます。詳しくは https://nip.io/ を参照してください。
- `your ingress controller`:使用しているIngress Controllerのclass nameに置き換えてください。ここでは`nginx`です・

# 2.収集側の作業
## 2.1 `namespace`を作成する
他のプロダクトが`default`にデプロイされているため、namespaceを`default`と分けて`fluent`で作成します。
``` yaml:fluent-namespace.yaml
kind: Namespace
apiVersion: v1
metadata:
  name: fluent
```
applyします。
```bash
kubectl apply -f fluent-namespace.yaml
```
## 2.2 fluentdの設定ファイルを変更するための`ConfigMap`を作成する
fluentdはデフォルトで`var/log/containers/`以下のファイルに対してjson用パーサーを使用します。これはdockerのログがjson形式であるためです。
ですが、もしcontainerdやcri-oを使用している場合、ログの形式はdockerのそれと全く異なります。よって、専用のcriパーサー(https://github.com/fluent/fluent-plugin-parser-cri)を使用する必要があります。

どのパーサーを使用するべきなのか不確かでしたら、`kubectl exec -it yourpodname -- sh`でpodに接続し、`/var/log/`以下のファイルを覗いてみてください。もしログの形式が

```
2020-10-10T00:10:00.333333333Z stdout F Hello Fluentd
```
のようになっていたら、以下を参考にfluentdの設定を変更してください。もしjson形式でしたらこの手順は必要ありません。

`ConfigMap`用のマニフェストファイルを作成します。
```yaml:fluent-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: tail-container-parse-conf
  namespace: fluent
data:
  tail_container_parse.conf: |
```
applyします。
```
kubectl apply -f fluent-configmap.yaml
```
:::message
追記：この`ConfigMap`が変更しているのは後ほど使用するfluentd-kubernetes-daemonset(https://github.com/fluent/fluentd-kubernetes-daemonset)イメージの`/fluentd/etc/`以下にある`tail_container_parse.conf`です。こちら(https://github.com/fluent/fluentd-kubernetes-daemonset/tree/master/docker-image/v1.17/debian-elasticsearch8/conf)ですべての設定ファイルを閲覧できます。
:::

## 2.3 Elastic Stackクラスタの情報を記載するSecretを作成する
fluentdがElasticSearchにログを送信する際にはElasticSearchのアカウントとパスワードをが必要になります。パスワードを直接fluentdの設定ファイルに記述するのは安全性と柔軟性に欠けますので、ここではSecretを使用して先ほど取得したパスワードとアカウントを記述します。

まず、OpaqueタイプのSecretに格納される情報は`base64`エンコードされている必要がありますので、先にアカウントとパスワードを以下のコマンドで`base64`エンコードしましょう。
```bash
echo -n 'fluentd' | base64
```
```bash
echo -n 'mypassword' | base64
```

:::message
注：`''`は取り外さないでください。
:::
次に、先ほどのコマンドで出力されたアカウントとパスワードのエンコード結果を、以下のファイルの該当項目に入力します。

applyします。
```bash
kubectl apply -f fluent-secret.yaml
```

## 2.4 fluentdをデプロイする
いよいよ大詰めです。以下に変更すべき設定の一覧を紹介します。
- `containers.env.name: FLUENT_ELASTICSEARCH_HOST`
  - 先ほど取得したElasticsearchのEXTERNAL-IPを記入してください。
- 

```yaml:fluent.yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: fluentd
  namespace: fluent
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: fluentd
rules:
- apiGroups:
  - ""
  resources:
  - pods
  - namespaces
  verbs:
  - get
  - list
  - watch

---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: fluentd
roleRef:
  kind: ClusterRole
  name: fluentd
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: fluentd
  namespace: fluent
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: fluent
  labels:
    k8s-app: fluentd-logging
    version: v1
spec:
  selector:
    matchLabels:
      k8s-app: fluentd-logging
      version: v1
  template:
    metadata:
      labels:
        k8s-app: fluentd-logging
        version: v1
    spec:
      serviceAccount: fluentd
      serviceAccountName: fluentd
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        effect: NoSchedule
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: fluentd
        image: fluent/fluentd-kubernetes-daemonset:v1.17-debian-elasticsearch8-1
        env:
          - name: K8S_NODE_NAME
            valueFrom:
              fieldRef:
                fieldPath: spec.nodeName
          - name:  FLUENT_ELASTICSEARCH_HOST
            value: "45.79.244.31"
          - name:  FLUENT_ELASTICSEARCH_PORT
            value: "9200"
          - name: FLUENT_ELASTICSEARCH_SCHEME
            value: "https"
          - name: FLUENTD_SYSTEMD_CONF
            value: disable
          # Option to configure elasticsearch plugin with self signed certs
          # ================================================================
          - name: FLUENT_ELASTICSEARCH_SSL_VERIFY
            value: "false"
          # Option to configure elasticsearch plugin with tls
          # ================================================================
          - name: FLUENT_ELASTICSEARCH_SSL_VERSION
            value: "TLSv1_2"
          # X-Pack Authentication
          # =====================
          - name: FLUENT_ELASTICSEARCH_USER
            value: "elastic"
          - name: FLUENT_ELASTICSEARCH_PASSWORD
            value: "k29mg54NteJvmJsf"
          - name: FLUENT_CONTAINER_TAIL_EXCLUDE_PATH
            value: /var/log/containers/fluent*
          - name: FLUENT_CONTAINER_TAIL_PARSER_TYPE
            value: /^(?<time>.+) (?<stream>stdout|stderr) [^ ]* (?<log>.*)$/
          - name: FLUENT_CONTAINER_TAIL_PATH
            value: /var/log/containers/abconvert*.log
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        # - name: containers-conf-volume
        #   mountPath: /fluentd/etc/kubernetes/containers.conf
        #   subPath: containers.conf
        # When actual pod logs in /var/lib/docker/containers, the following lines should be used.
        - name: dockercontainerlogdirectory
          mountPath: /var/lib/docker/containers
          readOnly: true
        # When actual pod logs in /var/log/pods, the following lines should be used.
        # - name: dockercontainerlogdirectory
        #   mountPath: /var/log/pods
        #   readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:
      # - name: containers-conf-volume
      #   configMap:
      #     name: containers-conf
      # - name: nodejs-conf-volume
      #   configMap:
      #     name: nodejs-conf
      # - name: tail-container-parse-conf-volume
      #   configMap:
      #     name: tail-container-parse-conf
      - name: varlog
        hostPath:
          path: /var/log
      # When actual pod logs in /var/lib/docker/containers, the following lines should be used.
      - name: dockercontainerlogdirectory
        hostPath:
          path: /var/lib/docker/containers
      # When actual pod logs in /var/log/pods, the following lines should be used.
      # - name: dockercontainerlogdirectory
      #   hostPath:
      #     path: /var/log/pods
```

