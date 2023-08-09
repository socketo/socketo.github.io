---
date: 2015-06-13T20:34:08.0000000
draft: false
title: "CTF for ビギナーズ2015 博多（Attack & Defense）に参加した話"
tags: ["CTF", "event"]
eyecatch: 
---

※ 移行前の元ブログ記事 : [CTF for ビギナーズ2015 博多（Attack & Defense）に参加した話 - #include <sys_socket.h>](https://socketo.hatenablog.jp/entry/2015/06/13/203408_1)

2015-06-07，富士通株式会社 九州支社で開催されたCTF for ビギナーズ2015 博多（Attack & Defense）に参加してきました．

CTF for ビギナーズ2015 博多（Attack & Defense）: https://attack-and-defense.doorkeeper.jp/events/24249

今回のこのイベントで初めて福岡に行きました．博多の明太子や博多ラーメン，もつ鍋がとても美味しかったのでサイコーという感じ．


# 競技

今回のCTFは名前の通り，Attack＆Defenseということで私が今までやってきたjeopardy型ではなく，本当にサーバーを介しての攻防戦だった．  
各チーム与えられたサーバ上で稼働しているシステムを保守しながら，相手チームに攻撃するとのこと．  
与えられるサーバの環境は全チーム同じ環境で，まずは自分の持っているサーバの脆弱性を探りその脆弱性を直しながら，他のチームが修正する前にそこを叩く．という感じでした．

競技のルールとしては，AttackポイントとDefenseポイントの合計で決定する．  
Defenseポイントは，与えられたサーバへの定期的なヘルスチェックのスキャンでサーバ上で稼働しているシステムが稼働しているかのSLAに応じた得点．  
Attackポイントは，今回与えられたサービスがネットショッピングのwebサイトだったため，敵のサーバで稼働しているサービスであるネットショッピングの顧客データの擬似個人情報ををスコアサーバに送信すると得点が得られる．  
みたいな感じだった．

はじめ2時間ほどは大抵のチームがSLAを保ちDefenseポイントが入り続けて順位は横ばい状態だった．後半になると，あるチームが攻撃に成功し，徐々に他のチームも得点していった．  
ちなみに今回私は[@Ranats_rifle](https://twitter.com/Ranats_rifle)とIchigoMilkなる可愛らしいチーム名で参加した．  
（競技終了後にTAKESAKOさんに「女の子が来るかと思ったこのチーム名」と言われた）

# 結果

結果としては結論から言うと9位だった．

> CTF for ビギナーズ2015 博多(Attack & Defense) 終了しました。参加された方はお疲れ様でした。今回の優勝はPh//shh/binです。 [\#seccon](https://twitter.com/hashtag/seccon?src=hash) [\#ctf4b](https://twitter.com/hashtag/ctf4b?src=hash) [\#a_and_d](https://twitter.com/hashtag/a_and_d?src=hash) [pic.twitter.com/JFiED8niDe](http://t.co/JFiED8niDe)
> — SECCON CTF (@secconctf) [2015, 6月 7](https://twitter.com/secconctf/status/607480338319695874)

一回攻め込まれてそのままズルズルとやってたら終了してた．  
@Ranats_rifleに攻撃の方を任せっぱなしで自分はサーバの状態を眺めていたりしたけど，なんともAttackポイントを得られなかった．  
つらい．@Ranats_rifleの攻撃がわりといいところまで行っていたそうで，完全にまたもや自分は眺めていただけで終了した．つらい．

攻防戦が初めてだとはいえ，ここまで手足が出せない状態なのは非常にキツかったが，あまりできない体験はできた．  
攻防戦型の大会は次にSECCONの九州大会で開催するそうなのでそれまでにマッチョになってリベンジしたいと思う．

# おまけ

本場の博多ラーメン美味しかった．（一緒に行った[@Ranats_rifle](https://twitter.com/Ranats_rifle)が替え玉合わせて4玉食べててすごかった