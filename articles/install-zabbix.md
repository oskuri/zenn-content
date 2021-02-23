---
title: "Zabbix Applianceを導入してみた"
emoji: "⛳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["vsphere", "ESXi", "ホームラボ", "Zabbix"]
published: true
---

いろいろと新しい環境ができてきたので、機器類の異常にも気づけるように監視周りの整備を行いたいと思います。
今回はOSSで、近年広く使われているZabbixを導入していきます。

実は昨年に仕事でZabbixをコンテナで導入したことがあるのですが、今回は別の方法で導入したいと思います。

# Zabbixの提供形態とダウンロード
Zabbix自体はOSSで開発されており、誰でも[こちら](https://www.zabbix.com/jp/download)から自由にダウンロードが可能です。

リンクを見ていただくとわかる通り、Zabbixは大きく分けて以下の5種類+αの形で提供されています。

1. パッケージとしての提供
2. Cloud Imageとしての提供
3. Dockerコンテナとしての提供
4. Applianceとしての提供
5. ソースコードとしての提供

パッケージとしての提供ですが、Linuxサーバーをサポートしており、Zabbix社提供のリポジトリを使用することによって、おなじみのyumコマンド、dnfコマンド、aptコマンドなどからインストールが可能です。

Cloud　Imageとしての提供はそのままですが、クラウドベンダーによってはZabbixサーバーのインスタンスを、オーダーすることが可能なようで、面倒なセットアップは不要といったところでしょうか。

DockerコンテナのイメージはDocker hubで公開されており、イメージを落としてきて使用することが可能です。ただし、Zabbixは複数のコンポーネットによって動作しており、そのコンポーネントが別のコンテナとして提供されるため、使用には注意が必要です。

アプライアンスとしてすぐに使用できるような状態でも提供されています。インストーラーのisoイメージからVMware, Virtual Box, Hyper-V, KVMなどなど幅広く対応しています。また、リンクには記載がありませんが、認定パートナー社から物理筐体としてのアプライアンスを提供するサービスもあるようです。

今回はVM用に提供されているApplianceを導入するため、Zabbix 5.0 LTSの 5.0.8の.vmxをダウンロードしました。
![](https://storage.googleapis.com/zenn-user-upload/jye80kcluze8hvamuf5d1azhrb5h)

## おまけ：Zabbixのコンポーネント
Zabbixのメインサーバー機能は3のコンポーネントを組み合わせて連携させることによって提供しています。

1. Zabbixサーバーの監視を実行するサーバー本体
2. Zabbixサーバーのコンソール機能を提供するためのWEBサーバー
3. Zabbixサーバーの監視設定情報、監視結果などの各種情報が保管されるDBサーバー

WEBサーバー機能はApachまたはNGINXで提供され、DBはMySQLまたはPostgreSQLを利用することになります。


# Zabbix Applianceのインポート

先ほどダウンロードしたファイルをホストクライアントまたはvCenter経由で、起動させたいESXiがアクセスできるデータストアに配置します。
今回は以下の2ファイルをおきました。
- zabbix_appliance-5.0.8.vmx
- zabbix_appliance-5.0.8-disk1.vmdk

このvmdkをそのまま使うわけではなく、一度Convertする必要があります。
データストアは `/vmfs/volumes`にマウントされているので、ESXiにsshでログインし、アップロードしたvmdkファイルを見つけます。
見つけたら、vmkfstoolsコマンドを使ってcloneします。
```SHELL
vmkfstools -i 元のvmdkファイル名 新しく作るvmdkファイル名
```

:::details なぜそのままのvmdkファイルで使用できないのか
vmdkファイルは仮想マシン上でディスクとして認識されるファイルですが、実際に使用する際はVMname.vmdkファイルとVMname-flat.vmdkという実際にデータが書き込まれたファイルをペアで使用しています。そのため、ダウンロードしてきたvmdkファイルを、２つのvmdkファイルに分離する必要があり、上記のような工程が必要になります。
:::

<br>

また、vmxファイルの名前も新しく作る仮想マシンのディスプレイ名に合わせて修正して、vmxファイルの中身（仮想マシンの構成）も編集します。

```bash:zabbix_appliance-5.0.8.vmx
.encoding = "UTF-8"
displayname = "zabbix_appliance-5.0.8" #任意のディスプレイ名に変更
guestos = "other"
virtualhw.version = "15"
config.version = "8"
numvcpus = "4"
cpuid.coresPerSocket = "1"
memsize = "4096"
pciBridge0.present = "TRUE"
pciBridge4.present = "TRUE"
pciBridge4.virtualDev = "pcieRootPort"
pciBridge4.functions = "8"
pciBridge5.present = "TRUE"
pciBridge5.virtualDev = "pcieRootPort"
pciBridge5.functions = "8"
pciBridge6.present = "TRUE"
pciBridge6.virtualDev = "pcieRootPort"
pciBridge6.functions = "8"
pciBridge7.present = "TRUE"
pciBridge7.virtualDev = "pcieRootPort"
pciBridge7.functions = "8"
vmci0.present = "TRUE"
floppy0.present = "FALSE"
sata0:0.present = "TRUE"
sata0:0.deviceType = "disk"
sata0:0.fileName = "zabbix_appliance-5.0.8-disk1.vmdk" #vmkfstoolsで作成したvmdkファイル名に修正
sata0:0.allowguestconnectioncontrol = "false"
sata0:0.mode = "persistent"
sata0.present = "TRUE"
ethernet0.present = "TRUE"
ethernet0.virtualDev = "e1000"
ethernet0.connectionType = "nat"
ethernet0.startConnected = "TRUE"
ethernet0.addressType = "generated"
ethernet0.wakeonpcktrcv = "true"
ethernet0.allowguestconnectioncontrol = "true"
toolscripts.afterpoweron = "true"
toolscripts.afterresume = "true"
toolscripts.beforepoweroff = "true"
toolscripts.beforesuspend = "true"
```

これで一旦、vCenterから仮想マシンを登録します。
データストアのブラウジングページから、編集したvmxファイルを選択して、仮想マシンの登録を進めます。
![](https://storage.googleapis.com/zenn-user-upload/k8i64v21i50kf9qf51jz9riqi9n1)

基本的人は仮想マシンの名前と仮想マシンを動かすホストが聞かれるだけです。
登録が完了すると、仮想マシン一覧の中に現れるので最後に仮想マシンの設定の編集から、参加させるネットワークを選択して起動させます。
問題がなければ、ログイン画面が開いているためログインします。今回使用したバージョンではID/PWはroot/zabbixでログインできました。

![](https://storage.googleapis.com/zenn-user-upload/wov54tm698emjqzwl8zf2yscp4xk)

この時点で、すでにZabbixは起動している状態ですが、ブラウザからアクセスする必要があるのでIPを確認します。ちなみにこのアプライアンスはCentOS 8がベースとなっていました。

`ip a`で割り当てられているIPアドレスを確認して、ネットワーク設定を行います。
NetworkManagerが入っていないため、`/etc/sysconfig/network-scripts/ifcfg-eth0`あたりを修正する必要があります。


# Zabbixへの接続

IPが無事変更されたところで、ブラウザからZabbixサーバーのIPを入力して、Zabbixのコンソールへアクセスしてみます。

ログイン画面が無事表示されました。
![](https://storage.googleapis.com/zenn-user-upload/uoy5f0zenkq0387zz6g0ng6erxz5)

アプライアンスにCLIでログインした際に表示される認証情報を使ってログインします。
（今回のバージョンはID/PW:Admin/zabbixでした）
![](https://storage.googleapis.com/zenn-user-upload/iqnnlfjcguiqr6qarursmnlaq5kj)


これでZabbixが動くことは確認できたので、次回、各種設定を行います。
