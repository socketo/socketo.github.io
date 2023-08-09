---
title: "Hello Hugo"
date: 2023-04-25T22:00:26+09:00
tags: [ others ]
draft: false
eyecatch: 
---

いつの頃からか、自分のはてなブログ [#include <sys/socket.h>](https://socketo.hatenablog.jp/)の冒頭に「この広告は、90日以上更新していないブログに表示しています。」と表示がされるようになった。

3年以上、個人の技術ブログを書いていないのは如何なものかと思い重い腰を上げた次第である。

## Hugoのインストール

Homebrewでhugoをインストールし、ブログを作成する。

```
$ brew install hugo
$ hugo new site hugo_blog
```

`hubo_blog`ディレクトリが作成されたので、テーマの追加。今回はシンプルで良さそうな[Hugo PaperMod](https://github.com/adityatelange/hugo-PaperMod)を使用する。

```
$ git init .
$ git submodule add https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
```

configにブログのbaseURLの修正やテーマの追加をする。

``` config.toml
baseURL = 'https://socketo.github.io'
title = 'socketo.github.io'
languageCode = 'ja'
theme = 'PaperMod'
author = 'socketo'
copyright = '© Copyright socketo - All rights reserved.'
```

テーマの導入まで完了をしているので、実際に動かして確認をする。

```
$ hugo serve
```

起動し、 `localhost:1313` にアクセスする。

![](/images/hello_hugo.png)


## 記事の作成

この記事を作成する。
```
$ hugo new posts/hello-hugo.md
```

![](/images/hello_hugo_post.png)

任意の記事の内容を書く。

### 記事のビルド

```
$ hugo
```

## さいごに

ブログを書いていきたい。