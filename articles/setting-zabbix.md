---
title: "Zabbixの諸設定"
emoji: "👀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Zabbix"]
published: true
---

こちらの記事でZabbix ApplianceをVM基盤にデプロイしたので、こちらの諸設定を行っていきます。

# VMware toolsのインストール

今回はopen-vm-toolsをインストールします。

```
yum install open-vm-tools
```

# タイムゾーンの設定

時刻同期はデフォルトで行われているので、タイムゾーンを日本に変更します。
```
timedatectl set-timezone Asia/Tokyo
```

# Zabbixの日本語対応

英語に苦手意識がある人間としては、こういうところを日本語化しないで頑張っていきたいところですが、今回は**あくまで**記事のネタのために日本語化します。

言語設定はZabbixのユーザーごとに設定が可能です。
設定は、左ペインの[Administration]-[Users]で対象のユーザーを選択します。

![](https://storage.googleapis.com/zenn-user-upload/v8fb3gnrs59ox8x7o0bmsomdjo12)

Languageから任意の言語を選択します。
![](https://storage.googleapis.com/zenn-user-upload/gjif23df7hg6c53u2b1pw4liusj0)

ここで、Zabbix Applianceでは初期導入の言語はen_GBとen_USの２つのみで、Japaneseも他言語と同様グレーアウトされており選択ができませんでした。
なのでここでyumを使って日本語のパッケージをインストールします。

```
yum install zabbix-web-japanese
```

インストールが完了すると、ページの再読み込みでJapaneseが選択できるようになるので、最後にUpdateボタンを押して反映させます。
ちなみにコンテナを日本語化させるのは結構めんどくさかったので、パッケージを入れただけでできるのはむちゃくちゃ楽ちんですね。。

# メディアタイプの作成

今回のZabbixはホームラボ環境の障害をお手軽に検知するための監視システムで、通知もラフにiPhoneのプッシュ通知とかで受け取りたいなと思っていました。
Zabbixは通知方法をメディアタイプと定義されている中から選択して通知することが可能です。無難にメールやSMSなどありますが、SlackやServiceNowなどよく使われそうなサービスとの連携に使えるテンプレートを親切に用意してくれています。

![](https://storage.googleapis.com/zenn-user-upload/w98cz5ns4cve1gcey2ke7jiyh818)

ざっと目を通して、自分が個人的にのみ利用しているものがなかったので今回はIFTTTのWebhook経由でPush通知が出せるように、メディアタイプを作ることにします。

## IFTTTのAppletを作成する

まずはIFTTTでAppletを作成します。IFTTTの細かい使い方は割愛しますが、IF を Webhookに、ThenをNotificationに設定します。

![](https://storage.googleapis.com/zenn-user-upload/zb0l65ori8qawv8hd38hvmpdsq19)

Webhookの設定内容については、EventNameだけ控えておきます。NotificationはとりあえずデフォルトのままでOK。
![](https://storage.googleapis.com/zenn-user-upload/co4h90h3l4mhelolut4vnfytwcjq)

## IFTTTでの通知テスト

一旦IFTTT上でテストします。[こちら](https://ifttt.com/maker_webhooks)のURLの先にあるDocumentationボタンを押して、テスト用のHTTP通信が発行できるページを開きます。

![](https://storage.googleapis.com/zenn-user-upload/d3n4gan4dk6woanq4vdwndefhm6n)
この中で、とりあえず赤枠のところだけ、↑で控えておいたEventNameを入力してTest itのボタンを押します。実際に自分用のWebhook URLに対してPOSTリクエストが発行されるため、正しく設定が行えていれば、IFTTTのインストールされたスマートフォン等から通知が行われるはずです。

![](https://storage.googleapis.com/zenn-user-upload/o5qir2d19b3bysldae074l14tt7z)

## ZabbixでのHTTPリクエスト発行

今度はZabbixのメディアタイプの方を動作確認ができるレベルで作ります。
（細かいメッセージ等は実際にアクションの設定をする時の方が、メッセージが具体的に現れてわかりやすくなると思うので後回しです）

左ペインの[管理]-メディアタイプ の画面右上からメディアタイプの作成を選択します。
メディアタイプの名前はお好みで設定し、タイプはWebhookを選択。初めから用意されているパラメータは一旦そのままで（Proxyは使う予定がないので削除してしまいました）、URLにIFTTTのWebhook URLを入れます。
![](https://storage.googleapis.com/zenn-user-upload/zta0hjao6l0zqd4xuu58inlsg9lz)

最後にスクリプトの項目のところで、必要なHTTPリクエストが発行できるようにJavaScriptを適当に書き書きします。
必要最低限のことしか書いていませんが参考までに載せておきます。

```JavaScript
var req = new CurlHttpRequest();
var param = JSON.parse(value);
req.AddHeader('Content-Type: application/x-www-form-urlencoded');

req.Post(
  param.URL,
  'value1='+param.Subject+'&value2='+param.Message
);

return JSON.stringify({
  'return_message': {
    'message': 'webhook test message'
  }
});
```

これで一度メディアタイプを保存して、実際にこのメディアタイプを使ってIFTTT宛にPOSTしてみます。メディアタイプの一覧から先ほど作成したメディアタイプの一番右のテストを押してPOPを開きます。とりあえずURLだけ埋まってることを確認して、テストボタンを押します。

![](https://storage.googleapis.com/zenn-user-upload/tv6xmgb7z64zyuiayigrch0tg1tw)

実際にZabbixサーバーからIFTTTのWebhook URL宛にHTTPリクエストが行われるため、IFTTTから無事通知があれば成功です！



# その他

Zabbixは全体でこれを設定しておけ、みたいなのがほとんどないイメージです。
もし、こんな設定しておいたほうがいいよ、というようなものがあれば教えて下さい🐶
