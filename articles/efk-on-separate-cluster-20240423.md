---
title: "Fluentd+ElasticSearch+KibanaでKubernetes上のログを収集して可視化する"
emoji: "🤛🏻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

# はじめに
LKE(Linode Kubernetes Engine)上に構築されているwebアプリケーションのログを外部ストレージに保存し、他所にデプロイされたElasticSearch+Kibanaで可視化するまでのプロセスと注意点をまとめます。

# EFK(**E**lasticSearch+**F**luentd+**K**ibana)スタックについて