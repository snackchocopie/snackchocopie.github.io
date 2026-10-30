---
title: 【初心者向け】AWS Lightsail + OpenVPNで自分専用VPNサーバーを構築する
lang: ja-JP
permalink: /jp/openvpn-server-install
author: ramen
date: 2026-10-30 16:50:00 +0900
categories: [OpenVPN]
tags: [vpn, linux, aws, lightsail]
canonical_url: https://qiita.com/snackchocopie/items/50f08dd766ca7c61439b
---

![logo.jpeg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4523380/36bfbdf0-6c3c-4ff6-98d8-574dc2ec2e68.jpeg)

# AWS LightsailでOpenVPNサーバーを構築してみる（基礎編）

今回は、簡単に再現できるVPNサービス構築の手順を実演します。

使用するサーバーはAWSの**Lightsail**、VPNサーバーは**OpenVPN**を使用します。  
基礎的な実演のため、詳細なセキュリティ設定などは今回の記事では扱いません。

## Lightsailとは

Lightsailは、AWSが提供するシンプルなVPS（仮想プライベートサーバー）サービスです。  
月額固定料金で利用でき、プランごとに一定のデータ転送量が含まれています。  
（含まれた転送量を超えると追加料金が発生します。）

![1.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4523380/3458e2c5-78d5-4f62-a64d-fd5bd8e4cf56.png)

*AWS Lightsailのメイン画面*

現在、すでに1台のサーバーを利用していますが、それとは別に新しいサーバーを作成していきます。


## サーバーの作成
![2.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4523380/0072d828-6c80-4ec6-b15b-6a4938828b71.png)

*プラットフォームはOperating System、BlueprintはUbuntu 24を選択*

画像のように、プラットフォームは**Operating System**をクリックし、Blueprintは**Ubuntu 24**を選択します。

Blueprintとは、AWS内であらかじめ用意されたテンプレートのようなものだと考えてください。


__リージョンを選択するよう赤色でマークしたリンクをクリックします。__


![3.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4523380/65002cff-35bb-4c7e-99a8-82c74d0b172c.png)

ここでは、米国リージョンの中から**Oregon**リージョンを選択します。

![4.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4523380/1f219bef-3b50-4495-8efe-bab0d54b1099.png)

ネットワークタイプは、IPv4とIPv6の両方を利用できる**Dual-Stack**を選択します。

IPv4アドレスが不要な場合は、**IPv6-only**を選択しても問題ありません。

参考までに、IPv4アドレスとは私たちが普段よく見る `123.123.123.123` のような、4つの数字ブロックで区切られたアドレスです。  
一方、IPv6はそれよりもはるかに長く複雑な `2001:0db8:85a3:0000:0000:8a2e:0370:7334` のような形式のアドレスです。

![5.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4523380/5d4d836e-beb2-485b-8044-18e9c5755d88.png)

インスタンス（サーバーの呼称単位）の名前は、識別しやすい一意の名前を付けます。

スナップショットは、万が一の障害に備えたバックアップイメージです。  
必要に応じてチェックしてください。今回は簡単な実演のため、チェックしません。

**タグ**は、AWSサービス内で作成したサーバーを識別・管理するためのラベルのようなものです。  
今回は国を区別する目的だけで使用するため、`country` タグキーのみを設定します。

![6.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4523380/a17fd11c-a56f-4032-b14b-1c08689636b8.png)

インスタンスが作成されると、自動的にパブリックIPv4アドレスとIPv6アドレスが割り当てられます。  

また、Amazonのプライベートネットワークで使用されるプライベートIPv4アドレスも1つ生成されます。  
（現時点では「こういうものが存在する」程度に理解しておけば十分です。）

本記事は単純な実演用のため公開しても特に問題ありませんが、実際の運用では外部に公開しないことを強く推奨します。  
（DDoS攻撃のターゲットになる可能性があるためです。）


## IPv4アドレスを固定する（アタッチ）

IPv6アドレスと異なり、IPv4アドレスはサーバーを停止・起動するたびに変わることがあります。  
これはIPv4アドレスが枯渇しつつあるアドレス体系のため、AWS内でも体系的に管理されているからです。

IPv4を所有（固定）するには、このIPを固定する必要があります。  
Lightsailではこれを **静的IP（Static IP）** と呼びます。

![7.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4523380/53b4a03b-2154-4a1f-9174-72d5afe8e0e6.png)
![8.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4523380/2507d6aa-bd06-4a27-a7e8-f0c9dc580b24.png)

*NetworkingタブでAttach Static IPを選択*


**Attach Static IP**をクリックし、希望する名前を指定して確認を押します。

## ネットワークトラフィックを許可する

![9.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4523380/82255c72-19af-4441-9e41-38f0aff1fa2d.png)

**AWS（Lightsail）** はデフォルトで、**80番（HTTPで使用するポート）** と **22番（SSHリモート接続）** のTCPポート以外は、すべてのポートを遮断します。

一般的に私たちが利用するインターネットトラフィックの多くはUDPを使用しており、今回使用するOpenVPNもデフォルトでUDP経由で通信します（変更は可能です）。

Networkingタブの下部にあるFirewallルール設定画面で、UDPプロトコルとOpenVPNと通信するUDPポートを開放します。

なお、上記は実演用のためすべてを開放しています。  
実際のサービスではすべてのポートを開放するのは避け、Firewallルールは最小限の権限を基本に厳格に運用する必要がある点を覚えておいてください。

![10.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4523380/a2e025a8-0832-4bb0-be9b-4b2483108954.png)

OpenVPNで使用するデフォルトのUDPポートは**1194**のため、これを許可します。

また、許可するIPv4およびIPv6のソースIPも、以下のようにすべて開放します。
 
*UDP 1194ポートとすべてのソースIPを許可*

## サーバーへのリモート接続用SSHキーの発行

![22.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4523380/d3ba9e64-37f6-4392-ad48-4ac47e95ef7e.png)

上記画像のセクションのように、まず**SSHキー**をダウンロードします。  
（キーの管理は厳重に行ってください！）

## サーバーに接続してOpenVPNサーバーをインストール

![11.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4523380/1e94c329-f934-41f7-ad82-aa46b65bcdac.png)

別途SSH接続プログラムをインストールすることなく、ブラウザ経由でAWSのSSH環境を使用してサーバーに接続します。

Connectタブに戻り、**Connect using SSH**ボタンを押してサーバーにアクセスします。



## オープンソース OpenVPN-Install を利用した簡単インストール

本格的にOpenVPNサーバーをインストールしていきます。

OpenVPNをゼロから手動でインストールしようとすると、複雑な環境設定によりトラブルシューティングに時間を取られることになります。  
これを最初から避けるため、使いやすいオープンソースを活用しましょう。

今回使用するオープンソースプロジェクトは、**angristan氏の openvpn-install** です。  
https://github.com/angristan/openvpn-install

最近は優れたオープンソースが非常に多く、目的のソリューションがあれば積極的にオープンソースを活用する方が良いと考えています。

### パッケージマネージャーを更新する。

```shell
$ sudo apt update
```
### ファイルのダウンロードおよび実行権限の付与

```shell
$ curl -O https://raw.githubusercontent.com/angristan/openvpn-install/master/openvpn-install.sh
$ chmod +x openvpn-install.sh
```

`chmod +x` コマンドは、該当ファイルに実行権限を付与する意味です。

Linuxでは、curlなどで外部からダウンロードしたファイルはセキュリティ上の理由でデフォルトで実行権限が外されているため、実行権限を付与しましょう。  
実行権限がないと実行できません。


## openvpn-install.sh インタラクティブ実行
![14.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4523380/3651e8ea-477b-4d26-aa48-973886eb4288.png)

上記の画像のように、実行権限（`x`）が付与されるとファイルの色が緑色で表示されます。

該当ファイルを以下のコマンドで実行します。

```shell
$ sudo ./openvpn-install.sh interactive
```

上記のコマンドを実行すると、以下のように対話型でインストールが進行します。

順次表示される質問は、ほとんどの場合デフォルト値のままEnterキーを押して進めて問題ありません。  
私も基本的にデフォルト値で設定しました。

以下が私の回答内容です。

```shell
What IP version should clients use to connect to this server?
> 1 (IPv4)

It seems this server is behind NAT. What is its public IPv4 address or hostname?
> 35.88.182.243

What IP versions should VPN clients use?
> 3 (Dual-stack (IPv4 + IPv6))

IPv4 VPN subnet:
> 1 (Default: 10.8.0.0/24)

IPv6 VPN subnet:
> 1 (Default: fd42:42:42:42::/112)

Route client internet traffic through the VPN? [y/n]
> y

Allow VPN clients to access each other? [y/n]
> n

Allow VPN clients to access the server's local network? [y/n]
> n

What port do you want OpenVPN to listen to?
> 1 (Default: 1194)

What protocol do you want OpenVPN to use?
> 1 (UDP)

What DNS resolvers do you want to use with the VPN?
> 3 (Cloudflare)

Allow multiple devices per client? [y/n]
> n

Do you want to customize the tunnel MTU?
> 1 (Default: 1500)

Choose the authentication mode:
> 1 (PKI)

Do you want to customize encryption settings? [y/n]
> n

Tell me a name for the client.
> tester

How many days should the client certificate be valid for?
> 3650

Do you want to protect the configuration file with a password?
> 2 (Use a password for the client)
```

ちなみに**Route client internet traffic through the VPN?** は、必ず **Y** を選択してください。

**N** にすると、すべてのインターネットトラフィックがVPNサーバーを経由しなくなるため、他国のサービス利用に制限が生じる可能性があります。


## VPNサーバー接続のための事前準備

対話型モードの回答がすべて終わると、スクリプト内でPKIの生成や必要な一連の処理が自動的に行われます。

最終的に、下記の画像のように`.ovpn`ファイルが生成されます。  
このファイルがOpenVPNサーバーに接続するための重要な設定ファイルです。

このファイルを、私たちが使用しているローカルコンピュータにダウンロードする必要があります。  
そのために**SCP**というプロトコルを利用します。SCPはTCP 22番ポートのSSH通信を活用したファイル転送プロトコルです。

![20.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4523380/147888bb-45b1-4bd8-99c8-61c9f37bd63f.png)
![21.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4523380/02a4a558-983f-4a96-a216-6ac431d3674e.png)

```powershell
scp -i C:\Users\me\aws\sshkey.pem ubuntu@35.88.182.243:/home/ubuntu/tester.ovpn C:\Users\me\aws\tester.ovpn
```

**Windows 11** では、デフォルトでSSHおよびSCPがインストールされています。  
もし入っていない場合は、別途インストールしてください。

上記のコマンドを、自分のPCの **PowerShell** または **コマンドプロンプト（CMD）** で実行します。

もちろん、該当する例の **サーバーIP**、**ユーザー名**、**キー名**はすべて私が使用しているサンプルですので、ご自身の情報をご入力ください。


## OpenVPNクライアントで接続をテストする

![24.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4523380/55efb6ac-6eeb-45ff-ad3e-3f6f76b29809.png)

本格的に **OpenVPNクライアント** をインストールして接続をテストしてみましょう。

OpenVPNクライアントは、OpenVPN公式が提供するクライアントアプリケーションです。  
他のサードパーティ製アプリではなく、こちらの公式クライアントの使用を推奨します。

アプリをインストールしたら、先ほどダウンロードした`.ovpn`ファイルをインポートして接続してみましょう。

![25.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4523380/652fa2d3-50c6-45ae-8076-d7132d36eb59.png)
![26.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4523380/2018b3f7-97b9-4204-b21b-c9d5b801c20f.png)

無事に接続が完了したことを確認できます。

また、[mylocation.org](https://mylocation.org) で位置情報を確認したところ、無事にOregonリージョンとして表示されることを確認できます。

