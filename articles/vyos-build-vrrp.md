---
title: "VyOSでVRRPを構成してみた"
emoji: "🏥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [vyos, network, vrrp]
published: true
---

こんばんは。[前回](https://zenn.dev/payapayao/articles/vyos-build-first)は仮想環境向けに仮想ルーター VyOS を導入しました。
今後、仮想基盤の重要なルーターとして稼働させることを想定し、ルーターをVRRPで冗長化しておきたいと思います。

# VRRPとは
VRRPは標準化されたプロトコルで、VRRPを構成したルーターたちに仮想IPアドレス(VIP)を付与し、ルーター間の死活状況によってどのルーターがVIPを持つかが決定されます。
詳しくはネットワークの勉強のお供を参考下さい。
https://www.infraexpert.com/study/fhrpz06.html

# VyOSでのVRRPの設定方法
どこかの技術ブログを参考に設定するつもりですが、導入済みのVyOSとのバージョン違いで設定方法が変更になっていたため、細かいことが考えれていない可能性が高いですがサンプルでコマンドを書いておきます。


## 設定の前提
今回は↓のeth0に対して192.168.1.0/24のIPが、それぞれ1つずつ付与された状態のVyOS2台でVIP192.168.1.254を持つように設定していきます。
```
vyos@VyOS-1:~$ show interfaces
Codes: S - State, L - Link, u - Up, D - Down, A - Admin Down
Interface        IP Address                        S/L  Description
---------        ----------                        ---  -----------
eth0             192.168.1.253                     u/u  
lo               127.0.0.1/8                       u/u  
                 ::1/128

vyos@VyOS-2:~$ show interfaces
Codes: S - State, L - Link, u - Up, D - Down, A - Admin Down
Interface        IP Address                        S/L  Description
---------        ----------                        ---  -----------
eth0             192.168.1.252                     u/u  
lo               127.0.0.1/8                       u/u  
                 ::1/128
```

## 設定方法
testというVRID10のVRRPグループを作成し、インターフェースを指定、VIPを192.168.1.254/24に設定する上3行を両方のVyOSに対して実行し、最後のルーターの優先順位を決定するためのpriorityオプションの値を1-255の中の数字で、優先させたいルーター側の数字が高くなるように数字を設定します。
今回はVyOS-1を優先させるように、VyOS-01のpriorityを200、VyOS-02のpriorityを100としました。
```
set high-availability vrrp group test vrid 10
set high-availability vrrp group test interface eth0
set high-availability vrrp group test virtual-address 192.168.1.254/24
set high-availability vrrp group test priority (1-255)
```

## VRRPの状態
上記の必要最低限の設定が完了したら、両VyOSでのVRRPとinterfaceの状態を確認してみます。
```
vyos@VyOS-1:~$ show vrrp
Name    Interface      VRID  State      Priority  Last Transition
------  -----------  ------  -------  ----------  -----------------
test    eth0             10  MASTER          200  14s

vyos@VyOS-2:~$ show vrrp
Name    Interface      VRID  State      Priority  Last Transition
------  -----------  ------  -------  ----------  -----------------
test    eth0             10  BACKUP          100  25s

vyos@VyOS-1:~$ show int
Codes: S - State, L - Link, u - Up, D - Down, A - Admin Down
Interface        IP Address                        S/L  Description
---------        ----------                        ---  -----------
eth0             192.168.1.253/24                  u/u                         
                 192.168.1.254/24                       
lo               127.0.0.1/8                       u/u  
                 ::1/128

vyos@VyOS-2:~$ show int
Codes: S - State, L - Link, u - Up, D - Down, A - Admin Down
Interface        IP Address                        S/L  Description
---------        ----------                        ---  -----------
eth0             192.168.1.252/24                  u/u  
lo               127.0.0.1/8                       u/u  
                 ::1/128             
```

show vrrpの結果でstateがMASTERとなっているVyOS-1のインターフェースeth0に、VIPである192.168.1.254/24が設定されています。
例えばこの状態からeth0をネットワークから切り離す（VMでNWを切断させる）と、VyOS-2のshow vrrpの結果が変化しVyOS-2のeth0に192.168.1.254/24が割り当てされることが確認できます。


# 参考サイト
同一バージョンを使用した例を紹介しているサイトがなかったため、公式のドキュメントを参考にしています。
今回出てこなかったオプション（preemption)などはこちらを参照下さい。
https://docs.vyos.io/en/equuleus/configuration/highavailability/index.html?highlight=vrrp#high-availability
