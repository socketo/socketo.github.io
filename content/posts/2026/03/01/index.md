---
title: "ノーコードツールMake.comを使ってInoreaderの記事をNotionデータベースへ登録する"
date: 2026-03-01T14:40:25+09:00
draft: false
tags: ["Notion", "ノーコードツール", "Inoreader", "自動化"]
---

{{< figure src="./images/01.png" alt="Head" width="600" >}}

---

# TL;DR

Inoreaderの「あとで読む」に追加した記事を、Make.com経由でNotionデータベースへ自動登録する仕組みを構築した。Webhookによるリアルタイム連携に加え、過去のアーカイブ全件はPythonスクリプトでInoreader APIを叩いて一括移行した。

# 1. モチベーション

数年前からRSSリーダとしてInoreaderを使っている。X(旧Twitter)をあまり能動的に見なくなってから情報収集としてRSSリーダをすごい使っている。
また、自分用備忘録はローカルのテキストファイルから始まり、Evernoteやらhack.mdやらObsidianなど様々なものを経由してNotionに落ち着いた。

Inoreaderで記録しておきたかった記事を「あとで読む」または「アーカイブ」、Webサイトを見ていた時に記録しておきたいNotion Web ClipperでNotionのデータベースに保存。
みたいな感じのことをしているとどこに何があるのかわからなくなる。多分よくある話。

Notionは自分用の備忘録やログ、データベースとしても機能させているので、なるべくここに集約したい。

# 2. システム構成図

## 概要
Inoreader の「あとで読む」に保存した記事を、Make.com を経由して Notion の My Tech Links データベースに自動登録するシステムを構築した。タグ付けは Notion AI の自動プロパティ入力機能で対応。

```
Inoreader「あとで読む」に追加
↓  Webhook（リアルタイム発火）
Make.com がデータ受信
↓
Web Clip DB に自動登録
↓
Notion AI が自動タグ付け
```

## 利用サービス一覧
| **サービス名** | **プラン** | 
| --- | --- |
| Inoreader | Pro |
| Notion | Plus(+AI課金) |
| Make.com | Free |

## Inoreader

Inoreaderのプラン
{{< figure src="./images/02.png" alt="Inoreader Pricing" width="600" >}}

https://www.inoreader.com/ja/pricing

| **項目** | **内容** |
| --- | --- |
| プラン | Pro（Webhookルール機能に必要） |
| トリガー | 「あとで読む」リストに記事を追加したとき |
| ルール名 | Save to Notion |
| アクション | Webhook をトリガー → Make.com の Webhook URL を指定 |

※ Supporter プランではルール機能（Webhook）が使えないため Pro にアップグレードが必要

## Notion

Notion Plusを契約しているのでそのまま利用。
最近NotionAIも使ってみているのでそれも使っている。

## Make.com
https://www.make.com/en

Make.comはノーコードツールプラットフォームで、Difyのように生成AI関連のアプリケーションと組み合わせる感じで最近話題になったりしている気がする。（気がする）

これについては自分のやりたいことではFreeの範囲で行けたのでFreeで利用。

| **項目** | **内容** |
| --- | --- |
| プラン | Free（無料枠で運用可能） |
| モジュール① | Inoreader: Watch Webhook Triggers |
| モジュール② | Notion: Create a Database Item (Legacy) |


# 3. 実装
## 3.1. リアルタイム自動連携
### Inoreaderの準備
Inoreaderの左パネルの「Automation」へアクセスし、ルールを作成する。

https://www.inoreader.com/rules

{{< figure src="./images/03.png" alt="Create Rules" width="600" >}}


### Make.comの準備
Make.comにログインする。

パネルの左の「Scenario」をクリックし、「Create scenario」をクリックして新しいシナリオを作成する。

{{< figure src="./images/05.png" alt="Create Scenario" width="600" >}}

Search apps or modulesから「inoreader」と検索し、「Watch Webhook Triggers」 を選択

{{< figure src="./images/06.png" alt="Create Scenario inoreader 01" width="600" >}}

Webhookを作成する。
1. Webhookを新規作成
2. Webhook名を入力
    - 「Webhook name」に任意のタイトルを入力(例: inoreader-star)
    - 「Connection」の隣の「Add」をクリック
3. Inoreaderアカウントを認証
4. 発行されたWebhookURLをコピー
    - 例 : `https://hook.eu2.make.com/xxxxxx`

{{< figure src="./images/07.png" alt="Create Scenario inoreader 02" width="600" >}}

### InoreaderのAutomationに入力
Inoreaderの準備で作成したAutomationのルールに以下のように入力する。

```
ルール名 : 任意のタイトル(例: Save to Notion)
いつ : 発火の条件(例: 「あとで読む」リストに新しく記事が追加されたとき)
条件 : 条件の追加(今回は空白)
アクション : 実行されるアクション(例: Webhookをトリガー <Webhook URL>)
```

設定が完了するとAutomationが追加される。

{{< figure src="./images/08.png" alt="Inoreader Settings Done" width="600" >}}


### Make.comでInoreaderのWebhookのテスト

make.comに戻って、以下の手順で確認。
1. シナリオの左下「Run Once」クリック(make.comがデータ待受状態になる)
2. Inoreaderで任意の記事を「あとで読む」に追加
3. make.comでデータが受信できているか確認する

{{< figure src="./images/09.png" alt="received inoreader" width="600" >}}


### Make.comでNotionモジュールを追加

1. キャンバス上の+(Inoreaderモジュールの右側)をクリック
2. 検索ボックスでNotionを入力
3. Notionを選択
4. Notionから「Create Database Item(Legacy)」を選択

{{< figure src="./images/10.png" alt="add notion module" width="600" >}}

Notionのアカウントを認証し、データ投入先のデータベースを指定する。

その際、個人用のNotionなので Connection TypeはNotion Publicを指定。

以下の項目を設定しSave。
```
Enter a Database ID : 登録先のDBを指定
Database ID : 登録先のDBのIDを指定
Fields : 
  タイトル : 1. Title
  URL : 1. Alternate[1]: Href
　タグ : (指定せず)
```

{{< figure src="./images/11.png" alt="notion setting" width="600" >}}


これで一通りの設定が完了。

「Run onece」の右隣にある「Immediately as data arrives」を有効にして定常実行の有効化(Scheduling)する。

完了

## 3.2. 過去のアーカイブ全件インポート

過去にInoreaderで大量に「あとで見る」をしていたものもNotionに移行させたかったので、Pythonスクリプトを書いてAPI直接操作を実施した。

InoreaderのAPIは以下のページで公開されているので参考にしながらいい感じにやればできた。

https://www.inoreader.com/developers/



# 4. 躓きポイント
- 過去のInoreaderのデータをNotionに移行させる作業がOAuthのstateパラメータが必須で、認可コードの有効期限が短くてAuthTokenの取得に苦戦した
- 元のNotionのデータベースの設計がぐちゃぐちゃだったのでそれを正す方がだるかった

# 5. 感想
ノーコードツールをほぼほぼ初めて使った気がするが、すんなりできてよかった。

あとから気づいたが、ZapierでInoreader Notion Integrationとなるものがありこれでよかった感。

Zapier Inoreader Notion Integration - Quick Connect - Zapier : https://zapier.com/apps/inoreader/integrations/notion
