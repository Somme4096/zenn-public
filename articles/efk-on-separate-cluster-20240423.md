---
title: "ElasticSearch+fluentd+KibanaでKubernetes上のログを収集して保存&可視化する"
emoji: "🤛🏻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["kubernetes","fluentd","elasticsearch","kibana"]
published: false
---

# はじめに
この記事ではKubernetes上に構築されているwebアプリケーションのログを収集し、ElasticSearchとKibanaでそれらを可視化するまでのプロセスをまとめます。

手順としては、Kubernetesクラスタ内の各コンテナのログをfluentdコンテナで取り込み、ログ収集クラスタ上にデプロイされたElasticSearchに転送します。転送した後、外部ストレージもしくはローカルストレージにログを保存し、Kibanaによるログの視覚的に検索・解析を可能にします。

# 使用したアプリケーション
- ElasticSearch
Javaで書かれているデータ検索分析エンジン。収集したログの分析、検索処理に使われます。
- Fluentd
`td-agent`とも。ログ収集・転送ソフトウェア。Kubernetes上のアプリケーションのログを収集し、ElasticSearchへと転送、格納します。
- Kibana
データ視覚化ツール。ElasticSearchのフロントエンドとして利用します。