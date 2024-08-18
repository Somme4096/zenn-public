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

本記事では、Elastic Stackの構築および公開、fluentdによるログの収集と送信、ElasticSearchでログを取り扱う上での設定などを紹介します。

:::message
ElasticSearchとfluentdをログ収集用途で使用する上に必要なindex lifecycle management機能やrollver indexの設定関連などについては、本記事では設定ファイルに登場した名詞を軽く解説する程度にとどまります。いずれはこれらの機能を詳説する記事を執筆する予定です。
:::

## ECKとは？
**ECK**(**E**lastic **C**loud on **K**ubernetes)はElasticSearchやKibanaなどを含むElastic StackのCRD/Operatorで、KubernetesへのElastic Stackのデプロイと管理を効率化します。

具体的には、ElasticSearchやKibana、APM serverなど個々のアプリケーションのカスタムリソースを定義し、定義されたリソースのタイプに基づいてアプリケーションの管理を自動化します。
:::message
(24/07/23)現在、公式に推奨されるElastic StackのKubernetesへのデプロイ方法です。
:::

## 動作環境
システムの全体図は以下になります。

#### 収集側クラスタ（SaaSプロダクション環境）
- LKE (Linode Kubernetes Engine) Dedicated 32GB
:::message
fluentdはメモリ使用量~200mbほどの小さいアプリケーションであるため、大抵の環境で動作できます。そのため、ここでは収集側クラスタのスペックは明記しません。
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

## 前提のインストール
公式ガイドを参考にしてカスタムリソース(CRD)をインストールします。

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
オペレーターをデプロイします。
```bash
kubectl apply -f https://download.elastic.co/downloads/eck/2.13.0/operator.yaml
```
## ElasticSearchのインストール

次にElasticSearchを構築します。ここでは`default`ノードを三つ作成し、各ノードに300GiのLinode Block Storageを与えています。名前は`logging-dashboard`とします。
:::message
上記の`default`ノードはElasticSearchのアプリケーション内ノードで、Kubernetesクラスタのノードではありません。
:::
Linode Block Storage以外のストレージサービスを使用している場合は適宜該当サービスのCRIドライバーが使用可能なことを確認し、`storageClassName`を変更してください。テスト用途にローカルボリュームを使用したい場合は`local-path-provisioner`を使用できます。
@[card](https://github.com/rancher/local-path-provisioner)

```yaml:elasticsearch.yaml
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: logging-dashboard
spec:
  version: 8.14.3
  nodeSets:
  - name: default
    count: 3
    config:
      node.roles: ["master", "data"]
      node.store.allow_mmap: false
    volumeClaimTemplates:
    - metadata:
        name: elasticsearch-data # Do not change this name unless you set up a volume mount for the data path.
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 300Gi
        storageClassName: linode-block-storage-retain
```
ファイルをapplyします。
```bash
kubectl apply -f elasticsearch.yaml
```
ヘルス状態を確かめます。
```
$ kubectl get elasticsearch

NAME                HEALTH   NODES   VERSION   PHASE   AGE
logging-dashboard   green    3       8.14.3    Ready   90s
```


問題なく動作しているようです。
## Kibanaのインストール
以下のファイルでKibanaを構築します。注意点として、`metadata.name`と`elasticsearchRef`の値はelasticsearchの構成ファイルの`metadata.name`の値と同じに設定してください。`spec.count`で同時に出現するKibanaのpodの数を指定できます。
```yaml:kibana.yaml
apiVersion: kibana.k8s.elastic.co/v1
kind: Kibana
metadata:
  name: logging-dashboard
spec:
  version: 8.14.3
  count: 2
  elasticsearchRef:
    name: logging-dashboard
```
ファイルをapplyします。
```bash
kubectl apply -f kibana.yaml
```
ヘルス状態を確かめます。
```
$ kubectl get kibana

NAME                HEALTH   NODES   VERSION   AGE
logging-dashboard   green    1       8.14.3    90s
```

インストールは特に問題なく完了したようです。試しに、KibanaのServiceを取得して`port-forwarding`で動作確認してみます。
```
$ kubectl get svc

NAME                                 TYPE           CLUSTER-IP       EXTERNAL-IP     PORT(S)          AGE
kubernetes                           ClusterIP      10.128.0.1       <none>          443/TCP          116d
logging-dashboard-es-default         ClusterIP      None             <none>          9200/TCP         60m
logging-dashboard-es-http            ClusterIP      10.128.5.221     <none>          9200/TCP         60m
logging-dashboard-es-internal-http   ClusterIP      10.128.238.88    <none>          9200/TCP         60m
logging-dashboard-es-transport       ClusterIP      None             <none>          9300/TCP         60m
logging-dashboard-kb-http            ClusterIP      10.128.185.177   <none>          5601/TCP         60m
```


KibanaのServiceである`logging-dashboard-kb-http`が見つかりました。
次のコマンドでKibanaに接続してみます。
:::message
注:kibanaのServiceの名称は`kibanaの構成ファイルに記述されているmetadata.nameの値`-kb-httpのような形式になっています。
:::
```
kubectl port-forward service/logging-dashboard-kb-http 5601
```
ブラウザで開いてみます。
@[card](https://localhost:5601)
問題なく表示できました。
![](/images/efk-on-separate-cluster/image.png)
## ElasticSearchとKibanaを公開する

先ほど`kubectl port-forwarding`でKibanaの動作確認ができましたが、現状ではKibanaとElasticSearchはクラスタ外部からアクセスできません。
```
> kubectl get svc

NAME                                 TYPE           CLUSTER-IP       EXTERNAL-IP     PORT(S)          AGE
logging-dashboard-es-default         ClusterIP      None             <none>          9200/TCP         41h
logging-dashboard-es-http            ClusterIP      10.128.5.221     <none>          9200/TCP         41h
logging-dashboard-es-internal-http   ClusterIP      10.128.238.88    <none>          9200/TCP         41h
logging-dashboard-es-transport       ClusterIP      None             <none>          9300/TCP         41h
logging-dashboard-kb-http            ClusterIP      10.128.185.177   <none>          5601/TCP         41h
```

ElasticSearchとKibanaのServiceである`logging-dashboard-kb-http`および`logging-dashboard-es-http`はタイプがクラスタの内部通信に使われる`ClusterIP`であり、このタイプのServiceは外部からアクセスできる公開IPアドレスが割り当てられていません。`Ingress`や`LoadBalancer`などを用いて適切に外部トラフィックを該当の内部Serviceにルーティングする必要があります。

### ElasticSearchをLoadBalancerで公開する
fluentdはElasticSearchへログを送信するゆえ、ElasticSearchは膨大な量のトラフィックを持続的に受け入れる必要があります。そこでElasticSearchはLoadBalancerを用いて公開します。

```
apiVersion: v1
kind: Service
metadata:
  name: elasticsearch-loadbalancer
spec:
  selector:
    common.k8s.elastic.co/type: elasticsearch
    elasticsearch.k8s.elastic.co/cluster-name: logging-dashboard
  type: LoadBalancer
  ports:
    - name: http
      protocol: TCP
      port: 9200
      targetPort: 9200
```


### KibanaをIngressで公開する

::::details Ingress Controllerをインストールする
`Ingress`タイプのServiceを扱うにはIngress Controllerが**必須**です。Ingress Controllerは多数ありますが、この記事ではIngress Nginx Controllerを使用します。
@[card](https://kubernetes.github.io/ingress-nginx/deploy/)
:::message
ここではhelmを使用してインストールしていますが、一般的なマニフェストを利用してインストールすることもできます。詳しくはingress-nginxの公式ドキュメントを参照してください。
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
動作確認できました。
::::

::::details cert-managerをインストールする
すでにインストールされている方は次のステップにお進みください。
@[card](https://cert-manager.io/docs/installation/)
:::message
ここではhelmを使用してインストールしていますが、yaml manifestを利用してインストールすることもできます。詳しくはcert-managerの公式ドキュメントを参照してください。
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
::::

::::details cert-managerとLet's EncryptでクラスタのTLS認証を行う

cert-managerで自動的にLet's Encrypt(HTTP01 Challenge)による証明書の発行と期限切れの証明書の更新を行えるように設定します。

`ClusterIssuer`を作成します。cert-managerのIssuerには二種類あり、ここではクラスタレベルのスコープで使用できるClusterIssuerを使います。

```yaml:clusterissuer.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: youremail@youremail.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
          serviceType: ClusterIP
```
変更すべき項目を説明します。
- `acme.email`:自分のメールアドレスを記入してください。
- `acme.server`:使用するacmeのサーバー。ここではLet's EncryptのProd(Production)サーバーを使用していますが、リクエスト回数に制限があるので、テスト環境ではLet's Encryptのstaging環境(https://acme-staging-v02.api.letsencrypt.org/directory)をお勧めします。

:::message alert
Let's Encryptのstaging環境にて発行される証明書は正式に使用できません。あくまでテスト用としてご利用ください。
:::
- `acme.solvers[0].http01.ingress.class`:使用しているIngress Controllerの値に置き換えてください。この記事では`nginx`になります。これでcert-managerはport 80にingressを配置し、acmeサーバーにHTTP-01認証をさせることができます。

パラメーターの調整が終わりましたらapplyしましょう。
```bash
kubectl apply -f clusterissuer.yaml
```
ついでに確認もしてみます。
```bash
$ kubectl get ClusterIssuer

NAME               READY   AGE
letsencrypt-prod   True    13m
```
::::

Kibana用に、以下のファイルで`ingress`Serviceを作成します。

```yaml:ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/backend-protocol: HTTPS
  name: kibana
spec:
  ingressClassName: nginx
  rules:
  - host: your domain
    http:
      paths:
      - backend:
          service:
            name: logging-dashboard-kb-http
            port:
              number: 5601
        path: /
        pathType: ImplementationSpecific
      - path: /elasticsearch
        backend:
          serviceName: logging-dashboard-es-http 
          servicePort: 9200
  tls:
  - hosts:
    - your domain
    secretName: your domain
```


- `your domain`:ドメイン名を入力してください。もし手持ちのドメインがありませんでしたら、テスト用途に`nip.io`を利用できます。先ほど取得した`ingress-nginx-controller`Serviceの`External-IP`の値を`your domain`のように当てはめることでDNSルーティングができます。詳しくは https://nip.io/ を参照してください。
- `your Ingress Controller`:使用しているIngress Controllerのclass nameに置き換えてください。ここでは`nginx`です・

## 追加設定


# 2.収集側の作業
## namespaceを作成する
namespaceを`default`と分けて`fluent`で作成します。
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
## fluentdの設定ファイル
### パーサーの設定
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

### インデックスの設定
こちらでは、ElasticSearchでログローテーションなどを構築する際に必要になる*index lifecycle management* (*ILM*)の構築に必要なFluentdの設定を記載します。以下ではElasticSearchにおけるログ

:::details

:::

## 2.3  Elastic Stackクラスタの情報を記載するSecretを作成する

fluentdがElasticSearchにログを送信する際にはElasticSearchのアカウントとパスワードをが必要になります。パスワードを直接fluentdの設定ファイルに記述するのは安全性と柔軟性に欠けますので、ここではSecretを使用して先ほど取得したパスワードとアカウントを記述します。

まず、OpaqueタイプのSecretに格納される情報は`base64`エンコードされている必要がありますので、先にアカウントとパスワードを以下のコマンドで`base64`エンコードしましょう。
```bash
echo -n 'fluentd' | base64
```
```bas2
echo -n 'mypassword' | base64
```

:::message
注：`''`は取り外さないでください。
:::
次に、先ほどのコマンドで出力されたユーザー名とパスワードのエンコード結果を、以下のファイルの該当項目に入力します。
```yaml:fluent-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: elastic-secret
  namespace: fluent
type: Opaque
data:
  username: 
  password: 
```

applyします。
```bash
kubectl apply -f fluent-secret.yaml
```

## 2.4 fluentdをデプロイする
いよいよ大詰めです。以下に変更すべきの`metadata.name`の値と同じに設定`spec.count`はkibanaの同時に出現するpodの数を指定できます。してください。の一覧を紹介します。
- `containers.env.name: FLUENT_ELASTICSEARCH_HOST`
  - 先ほど取得したElasticsearchのEXTERNAL-IPを記入してください。
- 

```yaml:fluent.yaml
---
apiVersion: v1
kind: ServiceAccoun2
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
            value: "yourelasticsearchhost"
          - name:  FLUENT_ELASTICSEARCH_PORT
            value: "9200"
          - name: FLUENT_ELASTICSEARCH_SCHEME
            value: "https"
          - name: FLUENTD_SYSTEMD_CONF
            value: disable
          # Option to configure elasticsearch plugin with self signed certs
          # ================================================================
          - name: FLUENT_ELASTICSEARCH_SSL_VERIFY
            value: "true"
          # Option to configure elasticsearch plugin with tls
          # ================================================================
          - name: FLUENT_ELASTICSEARCH_SSL_VERSION
            value: "TLSv1_2"
          # X-Pack Authentication
          # =====================
          - name: FLUENT_ELASTICSEARCH_USER
            valueFrom:
              secretKeyRef:
                name: elastic-secret
                key: username
          - name: FLUENT_ELASTICSEARCH_PASSWORD
            valueFrom:
              secretKeyRef:
                name: elastic-secret
                key: password
          - name: FLUENT_CONTAINER_TAIL_EXCLUDE_PATH
            value: /var/log/containers/fluent*
          - name: FLUENT_CONTAINER_TAIL_PATH
            value: /var/log/containers/*.log
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: tail-container-parse-conf-volume
          mountPath: /fluentd/etc/tail_container_parse.conf
          subPath: tail_container_parse.conf
        # When actual pod logs in /var/lib/docker/containers, the following lines should be used.
        - name: dockercontainerlogdirectory
          mountPath: /var/lib/docker/containers
          readOnly: true
        # When actual pod logs in /var/log/pods, the following lines should be used.
        # - name: dockercontainerlogdirectory
        #   mountPath: /var/log/pods
        #   readOnly: true
      terminationGracePeriodSeconds: 45
      volumes:
      - name: tail-container-parse-conf-volume
        configMap:
          name: tail-container-parse-conf
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

