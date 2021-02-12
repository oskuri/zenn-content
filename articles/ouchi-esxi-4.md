---
title: "おうちラボ環境をつくってみる#4"
emoji: "🛁"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["esxi","vsphere","ホームラボ","vCenter"]
published: true
---

この記事は [おうちラボ環境をつくってみる#3]※未作成 の続きです。
:::message
スクリーンショットの有無と資金の関係(死活問題)で先にvCenterのデプロイ編をアップします...
:::


HypervisorであるESXiがセットアップできため、今度はvCenterをセットアップしていきます。
vCenterはWindows上に導入することもできますが、アプライアンスとしても提供されており、今回はそのアプライアンス(以下VCSA)を導入していきます。

# VCSAのデプロイ
VCSAは作業用PCからネットワーク越しにインストールしていきます。VMUG経由でインストールイメージをダウンロードしてきます。（割愛）
今回使用したのは VMware-VCSA-all-7.0.1-17327517.iso でした。こちらを作業用PCにマウントします。

ISOイメージの中身は、ReadMeが諸々とインストールメディアであるアプリ・exeファイルがOSごとに別れて配置されています。
![](https://storage.googleapis.com/zenn-user-upload/2zozl0oj8m5rsdf0sbfond7jlp1g)

WindowsとMacから、どちらもGUIでインストールする物を試しましたが、どちらも導入方法は同じだったように思います。今回はMacでキャプチャしています。

<br />

まず初めに、最近のMacは開発元が不明のアプリケーションの実行に対して厳しく、プログラムの実行許可が次々と確認されてうまく進められることができないため、一時的に[全てのアプリケーションに対する実行を許可](https://pc-karuma.net/macos-sierra-allow-apps-from-anywhere/)します。せっかちさんのためにコマンドだけ載せておきます。
```
sudo spctl --master-disable
```

:::message alert
上記コマンドの実行については自己責任でお願いします。
:::

<br />

インストーラを起動すると、メニューがいくつか現れるので新規インストールを選択。
![](https://storage.googleapis.com/zenn-user-upload/ihw0vym7ljq0j5un0x5neb9lbpni)
<br />

初めに使用許諾についての同意が聞かれ、同意するとVCSAのデプロイ先を聞かれるので、対象のESXiのIPまたはホスト名とデプロイ可能な権限を持つユーザーIDの認証情報を入力します。
![](https://storage.googleapis.com/zenn-user-upload/xx2n57flaocj9b2xi4q9cfs9orc1)
<br />

vCenterのホスト名、rootのパスワードを聞かれるので入力します。
![](https://storage.googleapis.com/zenn-user-upload/21s3lwlmkz3w8hqrir7r1m2ecto2)
<br />

vCenterサーバーのデプロイサイズは、自身の環境に合わせて選択します。ホームラボなら一番小さいサイズにしかならないと思いますが。。
![](https://storage.googleapis.com/zenn-user-upload/rcl0j5h79c221paldojwm3r8drop)
<br />

VCSAをデプロイするデータストアを選択します。デプロイには結構時間がかかるので、サクッと終わらせたい方は書き込み速度の早いデータストアを選択することをお勧めします。iSCSIのデータストアにデプロイしたらえらい時間かかりました...
![](https://storage.googleapis.com/zenn-user-upload/2z3yo22e3hxzv5s8fjoevmg7qotv)
<br />

最後にVCSAのネットワーク設定を行います。ここで注意ですがデプロイ後にVCSAのネットワーク設定を弄ったところ、vCenterにWebクライアントでアクセスできなくなる事象が3度ほど発生し、私は何度もデプロイのし直しを行いました...
[VMのナレッジ](https://docs.vmware.com/jp/VMware-vSphere/6.7/com.vmware.vsphere.vcsa.doc/GUID-56C3BA9A-234E-4D81-A4BC-E2A37892A854.html)を見る限りはネットワーク設定は変更可能に見えるのですが...
DNS・デフォゲ等間違えないように入力しましょう。ちなみに私はドメイン参加させる際に名前解決ができなかったのを直したくて、DNSを修正したかったのでした。
![](https://storage.googleapis.com/zenn-user-upload/f9w7ef544cjdgkdmkuje63ttap99)
<br />

最後の確認まで完了すれば、デプロイが始まるので待ちます。
![](https://storage.googleapis.com/zenn-user-upload/iia4sxup4m3j7k3iam2hlr0exzqy)
<br />

デプロイが終わると、もう少しvCenterの設定が続きます。
![](https://storage.googleapis.com/zenn-user-upload/qdutcd7vksqqniieff7w153batd0)


# vCenterの設定
また案内にしたがって各種設定を入れていきます。

はじめてvCenterを導入する際は新規でSSOドメインを作成する必要があるようで、適当に作成します。
![](https://storage.googleapis.com/zenn-user-upload/wk0xjg7fsg8zihi67okum027y91u)


設定変はスクリーンショット不足で残念ながらこれだけしか残っていませんでしたが、きっと大した設定項目はなかったのでしょう。
設定が完了したら、セットアップがバックグラウンドで走るので完了するまで待ちます。

正常に完了すると、vCenterにログインできるようになります。
ネットワーク設定のパートで指定したIPアドレスへブラウザでアクセスします。
https://<vCenterのIP>/ui/
![](https://storage.googleapis.com/zenn-user-upload/m19dz9dzkmb47jxgw1ymk34te20x)

お疲れ様でした。
