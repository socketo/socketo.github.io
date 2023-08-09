---
date: 2018-01-22T00:05:37.0000000
draft: false
title: "ハニーポットCowrieを設置した話"
tags: ["Security", "Honeypot"]
eyecatch: https://i.imgur.com/ojhKfFQ.jpg
---

※ 移行前の元ブログ記事 : [ハニーポットCowrieを設置した話 - #include <sys_socket.h>](https://socketo.hatenablog.jp/entry/2018/01/22/000537)

はい．

本投稿の内容に間違いや問題などありましたら，Twitter経由などでやんわり優しめに教えて頂ければ幸いです．(Twitter:[@sys_socket](https://twitter.com/sys_socket))

## Cowrieとは

[GitHub - micheloosterhof/cowrie: Cowrie SSH/Telnet Honeypot](https://github.com/micheloosterhof/cowrie)

> Cowrie is a medium interaction SSH and Telnet honeypot designed to log brute force attacks and the shell interaction performed by the attacker.
> Cowrie is developed by Michel Oosterhof.

攻撃者のブルートフォース攻撃とシェルの対話を記録するように設計された対話型SSH/Telnetのハニーポットです．


## 導入

ここを見てやっていきます．
[cowrie/INSTALL.md at master · micheloosterhof/cowrie · GitHub](https://github.com/micheloosterhof/cowrie)

今回はUbuntu　16.04.3 LTSで構築していきます．

必要なパッケージのインストール

``` text
$ sudo apt install git python-virtualenv libssl-dev libffi-dev build-essential libpython-dev python2.7-minimal authbind
```

インストールするパッケージで分かるのですが，Cowrieではpython-virtualenv上でPython2.7系を使用しています．
Python3系はサポートされていません．

Cowrie用のユーザを作成/cowrieユーザにスイッチ

``` text
$ sudo adduser --disabled-password cowrie
$ sudo su - cowrie
```

Cowrieリポジトリをclone

``` text
$ git clone https://github.com/micheloosterhof/cowrie.git
$ cd cowrie
```

python-virtualenvでPython環境を作成し，必要なPythonパッケージをインストールする

``` text
$ virtualenv cowrie-env
$ source cowrie-env/bin/activate
(cowrie-env) $ pip install --upgrade pip
(cowrie-env) $ pip install --upgrade -r requirements.txt
```

Cowrieの設定ファイルはリポジトリのディレクトリのcowrie.cfg.distとcowrie.cfgの2つ．
どちらのファイルも起動時に読み込まれてcowrie.cfgの方が優先される．
設定ファイルを作成．

``` text
$ cp cowrie.cfg.dist cowrie.cfg
```

ここで色々と設定を変更出来ます．

Cowrieはデフォルトではtelnetが有効になっていないので，telnetハニーポットとして利用する際は有効にする．

``` text
# ============================================================================
# Telnet Specific Options
# ============================================================================
[telnet]

# Enable Telnet support, disabled by default
enabled = true # ここをfalseからtrueに
```

通常ユーザに戻り，22と23ポートへの通信をCowrieが待ち構えているポート2222と2223へリダイレクトする設定をする

``` text
$ sudo iptables -t nat -A PREROUTING -p tcp --dport 22 -j REDIRECT --to-port 2222
$ sudo iptables -t nat -A PREROUTING -p tcp --dport 23 -j REDIRECT --to-port 2223
```

iptabelsを管理するiptables-persistentをインストール

iptablesの設定を永続化する

``` text
$ sudo apt install iptables-persistent
```

iptablesの確認

``` text
$ cat /etc/iptables/rules.v4
```

user cowrieでCowrieを起動

``` text
$ sudo su - cowrie
$ ~/cowrie/bin/cowrie start
```

Cowrieの動作を確認

``` text
$ netstat -antp | grep -e 2222 -e 2223
```

Cowrieに登録されているダミーユーザ/パスワードの確認

``` text
$ cat cowrie/data/userdb.txt
```

別の端末からSSH接続をして動いているか確かめる．

userdb.txtに存在するダミーユーザでログインしてみる．

この接続ログも記録されるので後々ログを見る際に注意をする．

``` text
$ ssh root@{ipaddress}
```

## ログの確認

palylogを使うことで攻撃者が侵入した後のコマンド内容を再生することが出来る．

``` text
$ ~/cowrie/bin/playlog ~/cowrie/log/tty/XXXXXXXX-XXXXXX-XXXXXXXXXXXX-XX.log
```

ログにあたりがついていて適当に見る分にはこれがいいが，良い感じにシュッと見たい．

個人的にはELK Stackを使いたかったが，ELK Stackを使うとなると結構ハードウェア的にスペックが必要になりそうで今回は断念．

[Logging Cowrie logs to the ELK stack - Fernando Domínguez](http://blog.fernandodominguez.me/logging-cowrie-logs-to-the-elk-stack/)

そういうわけで今回は，ある程度LogViewerとして軽量でシンプルそうなCowrie Log Viewerを使用

[GitHub - mindphluxnet/cowrie-logviewer: A simple log viewer for cowrie](https://github.com/mindphluxnet/cowrie-logviewer)

Cowrieサーバにログインし，cowrieユーザにスイッチ，cowrie-logviewerをインストール

``` text
$ sudo su - cowrie
$ cd ~/../cowrie
$ git clone https://github.com/mindphluxnet/cowrie-logviewer.git
$ cd cowrie-logviewer
$ source ~/cowrie/cowrie-env/bin/activate
$ pip install -r requirements.txt
$ chmod +x cowrie-logviewer.py
```

MaxMind GeoLite 2 Country databaseの設定

``` text
$ mkdir maxmind
$ cd maxmind
$ wget http://geolite.maxmind.com/download/geoip/database/GeoLite2-Country.mmdb.gz
$ gunzip GeoLite2-Country.mmdb.gz
$ rm GeoLite2-Country.mmdb.gz
```

Cowrie Log Viewerの起動

``` text
$ python cowrie-logviewer.py
```

ポートフォワーディングを行いブラウザでそのアドレスを開くとCowrie Log Viewerのダッシュボードにアクセスできる．

こんな感じで見れる．

![](https://i.imgur.com/ojhKfFQ.jpg)

ダッシュボード

![](https://i.imgur.com/0PBmXFM.jpg)
Attackers by country

ELK Stackにしたいな…いつかやります

## 運用の際の注意

ハニーポットを設置するサーバは，個人の自宅にある物理サーバかVPSという選択肢があるようです．

VPSを利用する際は，利用規約に注意が必要です．

サーバ上でハニーポットを運用するのが利用可能かどうかは以下の[@blackle0pard_](https://twitter.com/blackle0pard_)さんのブログがとても参考になります．

* [くろひょうのぶろぐ - ハニーポットの運用が規約違反でないか調べてみた](https://blackle0pard.net/v9bnm7/)

## まとめ

SSH/TelnetハニーポットのCowrieの構築をしてきました．

わりと簡単に構築出来て良い感じでした．

今回設置するにあたり調べた時に[T-Pot](http://dtag-dev-sec.github.io/mediator/feature/2016/10/31/t-pot-16.10.html)良い感じだなと思ったのですが，やっぱりある程度ハードウェア的なスペックが必要なよう．

ここ最近Honeypot Advent Calendar 2017](https://adventar.org/calendars/2263)や[morihi-soc](https://twitter.com/morihi_soc)さんのブログ[www.morihi-soc.net](http://www.morihi-soc.net/)など色々と読んでいたのですが，[ハニーポット運用楽しそうだなという軽いモチベーションからはじめましたがよろしくやっていきたいなという感じです．

とりあえず今回設置したCowrieを観察しつつ色々と手を出していきたいなという気持ちです．

### 参考

* [ハニーポットCowrieをCentOS7に入れてみた。 - Shi0shishi0](http://ecoha0630.hatenablog.com/entry/2016/02/07/125110)
* [cowrieを設置した // 題未定 //](https://www.lapis-zero09.xyz/25th.html)
* [Amazon LinuxにハニーポットCowrieを入れてみる - kopug memo](http://kopug.hatenablog.com/entry/2016/12/30/183349)