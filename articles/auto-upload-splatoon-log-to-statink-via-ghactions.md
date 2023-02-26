---
title: "Github Actionsを使ってSplatoon3のバトル戦績をstat.inkにアップロードするいちばん簡単な方法"
emoji: "🦑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["スプラトゥーン","splatoon3","statink","cloudshell","githubactions"]
published: true
---

## stat.inkとは

スプラトゥーンの勝敗データを収集して保存・解析するサイト。 \
[https://stat.ink/](https://stat.ink/)

ユーザーはスプラトゥーン3（1と2も対応している）での自分のバトル戦績をアップロードしあって、様々なデータ分析に利用することができる。

自分の戦績が公開されてしまうことが嫌ではないならば、ゲーム内やイカリング3で確認できるバトル戦績の50件を超えた分も保存しておけるので、自分のバトルを振り返るのに非常に役に立つ。

stat.inkのデータは多くの人（たとえば [Splatoon 統計課 (@splatoon_stat) / Twitter](https://twitter.com/splatoon_stat) など）によって分析されているので、それを見るのも楽しい。


## 方針

上記のようにメリットいっぱいのstat.inkではあるが、[s3s](https://github.com/frozenpandaman/s3s)（stat.inkでも紹介されているeliさんによる戦績アップロード用のプログラム）公式のセットアップガイドではPythonのインストールが必要だと書かれていたり、そもそも定期的に実行する環境を作成・維持するのにコストがかかったりと、導入のハードルが高いように見える。

有用な分析のためには多くの人がstat.inkにデータをアップロードする必要があるので、この記事では手軽に実現できて**無料**、かつ**自動**、かつ**メンテナンスフリー**な方法を紹介したい。
タイトルの"いちばん簡単"は盛った。

具体的には、汚してもいい一時的な環境としてのCloud Shell（無料）上でDockerコンテナ化されたs3s（無料）を実行してconfig.txtを作成し、GitHubのPublicリポジトリ上でGitHub Actions（無料）を使って定期的にs3sを実行することを目指す。


## 準備するもの

**ニンテンドーアカウント**
* バトル戦績をアップロードしたいスプラトゥーン3のデータがあるもの

**stat.inkアカウント**
* アカウント作成後、[[プロフィールと設定](https://stat.ink/profile)]からAPIキーを取得しておく

**GitHubアカウント**
* Publicリポジトリを作成してGitHub Actionsを設定する
* Zennを読んでいるような人はGitHubアカウントくらい持っているはず

**Googleアカウント**
* ブラウザ上で完結できるDocker実行環境として[Cloud Shell](https://cloud.google.com/shell/docs/launching-cloud-shell?hl=ja)を利用する
* 同じくZennを読んでいるような人はGoogleアカウントも持っているはず


## 手順1：config.txtを作成

config.txtはs3sを実行するために必要な情報が書かれているファイル。

s3sを実行することで作成されるが、自動化するのはGitHub Actions上になるので手元の環境を汚さずに、かつ手軽に入手したい。

そこで、issei_mさんによってメンテナンスされている[Dockerコンテナ化されたs3s](https://hub.docker.com/r/isseim/s3s)を利用する。


### コンテナ上のs3sを起動する

Google Cloud の Cloud Shellにアクセスする。 \
[https://shell.cloud.google.com/](https://shell.cloud.google.com/)

ちなみに、Cloud Shellはブラウザ上で操作できるターミナルとエディタが入った無料で使えるVMであり、今回は気兼ねなく利用できる環境として選択しているが、Docker Hubからコンテナイメージを持ってきて実行できる環境であればなんでもいい。

Cloud Shell画面下部のターミナル上で以下のコマンドを実行する。


```shell
docker run -it --rm --name s3s-for-config isseim/s3s -M
```


Cloud ShellにはDockerがプリインストールされているので、これだけでconfig.txtの作成が開始される。

config.txtの作成にはいくつかの対話的なやり取りが発生する。


### APIキーの入力

まずはstat.inkのAPIキーを設定する。

[[プロフィールと設定](https://stat.ink/profile)]から取得してコピペすればOK。


```shell
Generating new config file.
s3s v0.3.1
stat.ink API key: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```



### ロケール設定

次は地域設定をする。

どういう用途なのかはわからないが、stat.ink作者の相沢陽菜さんが[集計する際に見ている](https://twitter.com/fetus_hina/status/1612004471869149184)ようなので、スプラトゥーン3のプレイ言語と合わせておくほうがよさそう。
https://twitter.com/fetus_hina/status/1612004471869149184?conversation=none

日本語でプレイしているので `ja-JP` を入力する。


```shell
Default locale is en-US. Press Enter to accept, or enter your own (see readme for list).
ja-JP
```



### ニンテンドーアカウントから各種トークン取得

次はニンテンドーアカウントからリンクURLを取得する。

ターミナル上で表示される `https://accounts.nintendo.com/connect/1.0.0/authorize?~~~` という形式のURLにアクセスする。

```shell
Make sure you have read the "Token generation" section of the readme before proceeding. To manually input your tokens instead, enter "skip" at the prompt below.

Navigate to this URL in your browser:
https://accounts.nintendo.com/connect/1.0.0/authorize?~~~
Log in, right click the "Select this account" button, copy the link address, and paste it below:
```
:::message

「トークン作成のためimink APIに送信されるデータに個人を特定できる情報は含まれない（意訳）」とされているが、心配な場合は別途手動で外部へのデータ送信なしに作成することも可能
[https://github.com/frozenpandaman/s3s#token-generation-](https://github.com/frozenpandaman/s3s#token-generation-)


この記事ではs3sを信頼して、データ送信を気にせずに進める

:::

Cloud Shellを利用しているならば、このURLはそのままマウスカーソルでクリックできるのでコピペする必要はない。

表示されたニンテンドーアカウントの選択画面で、スプラトゥーン3のデータを持っているアカウントの右にある「この人にする」のリンクアドレスを取得（PCであれば右クリックから「リンクのアドレスをコピー」など）して、ターミナルに貼り付ける。

![連携アカウントの選択画面](/images/auto-upload-splatoon-log-to-statink-via-ghactions/nintendo-form.webp)
*あくまでリンクを取得するだけでクリックはしない*

`xxxx://auth#session_state=xxxx&session_token_code=xxxx&state=xxxx` という形式のかなり長い文字列を貼り付けることになる。


```shell
Make sure you have read the "Token generation" section of the readme before proceeding. To manually input your tokens instead, enter "skip" at the prompt below.

Navigate to this URL in your browser:
https://accounts.nintendo.com/connect/1.0.0/authorize?~~~
Log in, right click the "Select this account" button, copy the link address, and paste it below:
xxxx://auth#session_state=xxxx&session_token_code=xxxx&state=xxxx
```


すると以下のように表示されて300秒のカウントダウンが始まる。 \
この画面はそのままにしておき、ターミナル上で別のタブを開いて次の作業を行う。


```shell
Wrote session_token to config.txt.
Attempting to generate new gtoken and bulletToken...
Wrote tokens for <ニンテンドーアカウントの「おなまえ」> to config.txt.

Validating your tokens... done.

Waiting for new battles/jobs... (checking every 5 minutes)
Press Ctrl+C to exit. 300
```


:::message alert

ここで表示されている `<ニンテンドーアカウントの「おなまえ」>` はGitHub Actionsの実行ログに残ることになる。
[ニンテンドーアカウントのプロフィール編集画面](https://accounts.nintendo.com/profile/edit)から、「おなまえ」をGitHub上で公開されてもいい名前に変更しておくこと。

:::


### config.txtをコンテナから抽出

Cloud Shellのターミナル上部にある [+] アイコンから別のタブを開く。


![Cloud Shellのタブ欄](/images/auto-upload-splatoon-log-to-statink-via-ghactions/cloudshell-tab.webp)
*タブ形式でターミナルをいくつも立ち上げることができる*


新しく開かれたターミナルで以下のコマンドを実行すると、Cloud Shellのホームディレクトリにconfig.txtが作成される。 \
**コマンド末尾にドットがある**ので忘れずに含めること。


```shell
docker cp s3s-for-config:/opt/s3s/config.txt .
```


ここでCloud Shell画面上部のエディタからホームディレクトリを開いて、画面左側のファイル一覧にある `config.txt` を選択すると以下のような内容が確認できる。


```json
{
    "api_key": "xxxx",
    "acc_loc": "ja-JP|JP",
    "gtoken": "xxxx",
    "bullettoken": "xxxx",
    "session_token": "xxxx",
    "f_gen": "https://api.imink.app/f"
}
```


これらの情報を使ってGitHub Actionsを設定していく。

:::message

これらの情報を手元に控えたら、もう使わないので[Cloud Shell上のデータは削除](https://cloud.google.com/shell/docs/resetting-cloud-shell?hl=ja#reset)して構わない

:::

## 手順2：GitHub Actionsを設定

GitHub Actionsを利用してs3sを実行するように設定する。

GitHub Actionsは[Publicリポジトリとしてコードを公開した状態であれば無料で実行できる](https://docs.github.com/ja/actions/learn-github-actions/usage-limits-billing-and-administration)ので、今回はこれを利用する。


### Secretsの設定

config.txtの内容は機密情報でありyamlファイルに直接記入したくないため、リポジトリのSecretsに登録する。

リポジトリの [Setting]>[Security]>[Secrets and Variables]>[Actions] から、画面右上の [New repository secret] ボタンをクリックする。


![リポジトリシークレットの一覧画面](/images/auto-upload-splatoon-log-to-statink-via-ghactions/repo-secrets-list.webp)
*登録したあとはこうなる*


Nameとして `API_KEY` などのconfig.txtのKey文字列を入力し、Secretとして対応するValue文字列を入力する。

今回の例ではKey文字列は大文字に書き換えたが特に意味はない。


![Secret登録画面](/images/auto-upload-splatoon-log-to-statink-via-ghactions/create-repo-secret.webp)
*言うまでもないが、xxxx部分は置き換える必要がある*



### yamlファイルの作成

最近のGitHubはブラウザ上からファイルを追加できる。

[Code] タブの右上から [Add file]>[Create new file] を選択する。

画面上部のファイル名欄は「/」を入力すると自動的にディレクトリを作成してくれるので、ここでディレクトリとファイル名を指定する。

ファイル名はなんでもいいので `s3s-to-statink.yml` とした。


```
.github/workflows/s3s-to-statink.yml
```


ファイルの中身は以下の通り。


```yaml
name: Splatoon3 Battlelog Uplorder

on:
  schedule:
    - cron: '0 0-23/2 * * *'
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest
    env:
      API_KEY: ${{ secrets.API_KEY }}
      ACC_LOC: ${{ secrets.ACC_LOC }}
      GTOKEN: ${{ secrets.GTOKEN }}
      BULLETTOKEN: ${{ secrets.BULLETTOKEN }}
      SESSION_TOKEN: ${{ secrets.SESSION_TOKEN }}
      F_GEN: ${{ secrets.F_GEN }}

    name: Splatoon3 Battlelog Uploader
    steps:
      - name: Set up Python 3.11
        uses: actions/setup-python@v4
        with:
          python-version: 3.11

      - name: Checkout frozenpandaman/s3s
        uses: actions/checkout@v3
        with:
          repository: 'frozenpandaman/s3s'
          path: s3s

      - name: Generate config.txt
        working-directory: s3s
        run: |
          echo '{"api_key": "${{ env.API_KEY }}", "acc_loc": "${{ env.ACC_LOC }}", "gtoken": "${{ env.GTOKEN }}", "bullettoken": "${{ env.BULLETTOKEN }}", "session_token": "${{ env.SESSION_TOKEN }}", "f_gen": "${{ env.F_GEN }}" }' > config.txt
          cat config.txt
      - name: Install s3s requirements
        working-directory: s3s
        run: |
          pip install -r requirements.txt
      - name: Run s3s
        working-directory: s3s
        run: |
          python3 s3s.py -r
```


コードは[参考にしたサイト](https://unyacat.net/2022/12/29/gh-actions-splatoon-battlelog-uploder/)のものから以下の点を変更した。



* **cronの指定**：ゲーム内のステージ更新（日本時間奇数時）と同期するようにした
* **env変数のシークレット化**：このリポジトリを公開しても問題ないようにした
* **s3sのパラメータ**：`-nsr` を削除してサーモンランのデータも含むようにした

以上で設定が完了。

スプラトゥーン3の戦績がstat.inkに自動的にアップロードされるようになった。


### 実行の確認

このまま待っていても自動実行されるが、2時間待つのは大変なので手動で実行して確認する。

GitHubの [Actions] タブから [Splatoon3 Battlelog Uplorder]（上記yamlファイル内で指定した名前）を開く。

yamlファイルで `workflow_dispatch` を指定してあるので、[Run workflow]>[Run workflow] から手動で実行できる。


![GitHub Actionsの実行ログ画面](/images/auto-upload-splatoon-log-to-statink-via-ghactions/run-workflow.webp)
*それぞれの実行ログから詳細を確認することもできる*


手動実行後しばらく待てば完了する。

画像のようにグリーンチェックマーク✅が付けばOK。

:::message

Github Actionsを停止したくなったら[リポジトリを削除](https://docs.github.com/ja/repositories/creating-and-managing-repositories/deleting-a-repository)するか、[ワークフローを無効化](https://docs.github.com/ja/actions/managing-workflow-runs/disabling-and-enabling-a-workflow)すること

:::

## 自動実行されるようになったら

なにも考えずにプレイすれば、自動的にデータがアップロードされstat.inkに貢献できる。

GitHub Actionsではs3sの最新版を毎回取得しているので、メンテナンスも必要ない（はず）。

たまにstat.inkの自分のページをみて「このブキ好きだけど負けまくってんな〜」とか考えたりするのも楽しい。

*ほな、カイサン！！！*




## 謝辞

コンテナ化されたs3sからconfig.txtを取得するアイデアとコードはこちらから拝借しました。 \
[AWS Fargate を使ってスプラトゥーン3の戦績を stat.ink に定期保存できるようにした](https://engineering.mobalab.net/2022/11/30/sync-s3s-results-regulary-using-fargate/)
https://engineering.mobalab.net/2022/11/30/sync-s3s-results-regulary-using-fargate/

GitHub Actionsを利用してs3sを実行するというアイデアとコードはこちらから拝借しました。 \
[Splatoon3の戦績をstat.inkに自動アップロードするGitHub Actions - うにゃのすみか](https://unyacat.net/2022/12/29/gh-actions-splatoon-battlelog-uploder/)
https://unyacat.net/2022/12/29/gh-actions-splatoon-battlelog-uploder/
