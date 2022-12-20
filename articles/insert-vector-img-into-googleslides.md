---
title: "Googleスライドにベクター画像を挿入する方法"
emoji: "↗️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Googleスライド", "Googleドライブ"]
published: true
published_at: 2022-12-21 09:00
---

# TL;DR

WMF/EMF形式のファイルをGoogleドライブにアップロードする際にMIMEタイプを `application/x-msmetafile` と指定し、Google図形描画で変換するとGoogleスライドにコピペできる


# 仕組み

Google図形描画（Google Drawing）というマイナーなプロダクトがあり、これはGoogleスライドと共有できるベクターオブジェクトを扱うことができる。 \
このGoogle図形描画で、扱いたい元となるベクター画像を開くことでGoogleスライドにもコピペが可能になる。 \
だが単純にGoogle図形描画からベクター画像を開こうとしても、Googleスライドと同じく対応していないというエラーが表示されるばかりで開くことができない。

一方、Google図形描画の土台ともいえる[Googleドライブにはファイルの自動変換機能](https://support.google.com/drive/answer/2424368?hl=ja)というものがある。 \
一般的にはExcel形式などのファイルをGoogleスプレッドシート形式などに変換する際に使われる機能である。

これはGoole図形描画にも備わっているのだが、やはり単純にベクター画像をGoogleドライブにアップロードしただけでは変換機能が動作しない。 \
動作させるためには、MIMEタイプを `application/x-msmetafile` と指定しつつGoogleドライブにアップロードする必要がある。

適切にMIMEタイプを指定すると変換機能の対象として認識されるので、Google図形描画で開くことができるようになり、結果としてGoogleスライドにコピペすることができるようになる。


# 手順


## ベクター画像を用意する

残念ながらSVG形式はこの方法であっても使えない。

:::message

最後のGoogle図形描画で変換する工程で変換が終了しなかった

:::

SVG形式のファイルしかなければ変換する必要があるが、幸いこれは難しくない。 \
「SVG 変換」などで検索して出てきたツールでも使えばいいのでここでは説明を省略する。

:::message alert

当然のことながら画像を変換した場合は変換によって表示崩れなどが出ていないか確認する必要がある

:::

実際にアップロードするのはWMF形式またはEMF形式がよさそうである。 \
これは後に指定するMIMEタイプが意味しているファイル形式だからという理由になる。 \
他のファイル形式は試していないが、Google図形描画の変換機能にファイルを認識させることが目的なだけなので、指定するMIMEタイプと実際のファイル形式が違っていても動作するものがあるかもしれない。

:::message

[MIMEタイプ `application/x-msmetafile` はWMF形式を指す](https://mimetype.io/application/x-msmetafile)**らしい** \
EMF形式も含むような記述も見つかったが正直このあたりは全然調べていない

:::


## MIMEタイプ `application/x-msmetafile` を指定してアップロードする

今回の手法のいちばん重要な部分である。

一般的なアップロード方法（ブラウザへのD&D、PC版Googleドライブなど）ではこれが指定されないため、その結果として変換機能を利用することができない。 \
よって手動でこれを指定しつつアップロードすることで期待通りの動作をさせる。

アップロードにはスクリプトを使うことになるが、実行環境はどこでも構わない。 \
この記事ではローカル環境構築が必要ない方法として、Google Colaboratory（Colab）を使う方法を紹介する。

この他にも、Google Apps Script（GAS）を使って簡易的なアップローダーを作ってもいいかもしれない。 \
その場合は[Drive APIで `mimeType` から指定](https://developers.google.com/drive/api/v3/reference/files/create#request-body)すればよさそう。


### Google Colaboratory（Colab）サンプル


```python
from google.colab import auth, files
from googleapiclient.discovery import build
from googleapiclient.http import MediaFileUpload

auth.authenticate_user()

uploaded = files.upload()

service = build('drive', 'v3')
file_metadata = {
  'name': list(uploaded.keys())[0],
  'mimeType': 'application/x-msmetafile'
}
media = MediaFileUpload(list(uploaded.keys())[0], mimetype='application/x-msmetafile')
service.files().create(body=file_metadata, media_body=media, fields='id').execute()
```


@[card](https://gist.github.com/mintommm/83f9b8e9499358f9bf006399e51ba50f)


## Google図形描画ファイルとして変換する

アップロードされたベクター画像ファイルを、変換機能を通じてGoogle図形描画で開く。

これはGoolgeドライブ上のマウス操作で簡単に実行できる。 \
![Googleドライブ上での表示サンプル](/images/insert-vector-img-into-googleslides/drive-context-menu.png) \
*MIMEタイプの指定に成功しているとGoogle図形描画で開くこと（＝変換）ができるようになる*

:::message alert

ここでも画像の変換が（文字通り）発生するので表示崩れなどがないか確認する必要がある \
[テストに使った画像](https://www.w3.org/Graphics/SVG/svglogo.svg)は、WMF形式では崩れたがEMF形式なら大丈夫だった

:::


## コピペする

変換されてGoogle図形描画で開かれたオブジェクトはGoogleスライドへコピペできる。


# まとめ

準備に手間がかかるが一度スクリプトを作れば使い回せるので、時間があるときに準備しておくといいと思う。


# 蛇足：背景

例えば、Googleスライドでスライドを作成しているときロゴを貼り付けたいが解像度の高い画像が見つからず、SVG形式のベクター画像しか手に入らないことがある。 \
そうではなくともPNG形式などのラスター画像は拡大すると荒れるし、拡大に耐えうるサイズの巨大な画像を貼り付けるのは効率面でダサくて嫌だと思う人も多いのではないか。 \
今風に言うとストレージ資源を無駄に消費してエコではないという見方もできる。 \
なにより私は身軽で高効率なベクター画像が好きである。 \
省コストは正義といえる。

それなのにGoogleドキュメントときたらコピペや挿入で貼り付けようとすると対応してない形式だといってダメ出しをしてくる。 \
今どきベクター画像に対応していないなんて考えられない。 \
お前の作るオブジェクト、それこそベクター画像ではないのか。 \
自家製ならばいいが、他所様の作ったものは食べられませんとでもいうつもりか。

検索するといくつかの方法を挙げているページがヒットするが、2022年12月時点ではPowerpointファイルを通じて持ってくる方法以外は利用できなさそうな状況になっている。 \
Powerpointは持っていない。 \
というか、気軽使えるWindows環境がない。

Web版のPowerpointなら無料で使えるかもしれないが、ベクター画像のためだけにGoogleスライドの競合であるPowerpointを使うのも癪である。 \
というか、それならそもそもPowerpointを使えばいい。

そういったわけで、少しの手間をかけてでもGoogleスライドにベクター画像を貼り付ける方法が必要だった。
