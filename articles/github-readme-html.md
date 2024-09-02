---
title: "GitHubのREADME.mdをイイ感じにHTMLに変換してプッシュする"
emoji: "🌊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [GitHub, "Markdown", "HTML"]
published: false
---

## 概要

社内向けのツールをGitHubで開発した際にREADME.mdにマニュアルを書いていたのですが、GitHubのアカウントを持っていない人にもマニュアルを配布しやすいように、README.mdをイイ感じの見た目のHTMLに変換してリポジトリにプッシュすることにしました。

その方法についてまとめます。

## 実現したいこと

今回実現したいことに関わる要件は以下の通りです。

- GitHub上のプレビューと同じように、スタイルやシンタックスハイライトが効いた見た目にする
- スタイルや画像をすべて埋め込んで、単独のHTMLファイルとして配布できるようにする
- プルリクが`master`ブランチにプッシュされた後にHTMLに変換し、`master`にプッシュする
  - ただし、`master`はGitHub Repository rulesetsで保護されている

## 結果

以下の形で実現しました。

1. [jaywcjlove/markdown-to-html-cli](https://github.com/jaywcjlove/markdown-to-html-cli/) でREADME.mdをHTMLに変換する
1. [peter-evans/create-pull-request](https://github.com/peter-evans/create-pull-request) で作成したHTMLをpushし、プルリクエストを作成する
1. GitHub CLIの`gh pr merge`でプルリクを自動マージする

実際のGitHub Actionsの`.yml`ファイルは以下のようになりました。

```yaml
name: Gerenate README.html

on:
  push:
    branches:
      - master
    paths:
      - README.md

permissions:
  contents: write
  pull-requests: write

jobs:
  convert:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Set up Node.js
        uses: actions/setup-node@v3

      - name: Install dependencies
        run: npm i -g markdown-to-html-cli

      - name: Convert .md to .html
        run: markdown-to-html --source README.md --output README.html --title title --img-base64 --no-dark-mode

      - name: Create Pull Request
        id: cpr
        uses: peter-evans/create-pull-request@v6.1.0
        with:
          commit-message: "[bot] Update README.html"
          title: "[bot] Update README.html"
          body: "Automated update of README.html from README.md by [create-pull-request](https://github.com/peter-evans/create-pull-request) GitHub action"
          labels: github-actions
          delete-branch: true

      - name: Merge Pull Request
        if: steps.cpr.outputs.pull-request-operation == 'created'
        run: gh pr merge --merge --auto ${{ steps.cpr.outputs.pull-request-number }}
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

## 解説

なぜこの方法を取ったか、それぞれ解説していきます。

### トリガー

`on: push: branches: master`だけだとプルリクの自動マージによって再度ワークフローが走って無限ループになりそうだったため、`paths: README.md`を指定しています。  
ただし、[こちら](https://github.com/ad-m/github-push-action/issues/32)の情報によると`GITHUB_TOKEN`を使用したイベントはトリガーの対象にならないようなので、`paths: README.md`は不要かもしれません。

### Markdown→HTML変換

細かい設定をしなくても

1. GitHub上のプレビューと同じように、スタイルやシンタックスハイライトが効いた見た目にする
2. スタイルや画像をすべて埋め込んで、単独のHTMLファイルとして配布できるようにする

の2要件を満たすことができるツールとして[markdown-to-html-cli](https://github.com/jaywcjlove/markdown-to-html-cli/)が最適でした。  
1.については特にオプションを指定しないままで対応でき、2.については`--img-base64`オプションを指定することで画像を埋め込むことができます。

ただし、`markdown-to-html-cli`はGitHub Actionsでも使用可能なのですが、執筆現在の`v4.1.0`ではActionsに`--img-base64`に相当するオプションがなかったため`npm install`して実行する形にしています。

#### pandocを使用していない理由

MarkdownからHTMLに変換するツールとしては`pandoc`も有名です。また、GitHubのスタイルを実現するカスタムCSSとして[github-markdown-css](https://github.com/sindresorhus/github-markdown-css)もあり、以下のようなコマンドでスタンドアロンなHTMLを生成できるため最初はこれを検討していました。

```bash
pandoc -f gfm -t html5 --embed-resources --standalone -c https://cdnjs.cloudflare.com/ajax/libs/github-markdown-css/5.6.1/github-markdown.css README.md -o README.html
```

しかし、私の環境では

- シンタックスハイライト以外のスタイルが適用されない
- 箇条書きの内部に作成したテーブルが正しくレンダリングされない

などの問題があったため、`markdown-to-html-cli`を使用することにしました。

### プルリクエスト作成

`master`ブランチはGitHub Repository rulesetsで直接のpushを禁止していたため、プルリクエストを作成して自動でマージすることにしました。  
[Deploy Keys](https://docs.github.com/ja/authentication/connecting-to-github-with-ssh/managing-deploy-keys#deploy-keys)を作成してGitHub Repository rulesetsの[Bypass list](https://docs.github.com/ja/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/creating-rulesets-for-a-repository#granting-bypass-permissions-for-your-branch-or-tag-ruleset)に追加することで直接のpushも可能だったかもしれませんが、わざわざこのためだけにDeploy Keysを作成する必要はないと判断してこの形としました。

プルリクの作成には[create-pull-request](https://github.com/peter-evans/create-pull-request)アクションを使用しました。  
`labels: github-actions`でPRにラベルを付与することで、自動生成したプルリクを識別しやすくしています。

### プルリクエストの自動マージ

GitHub ActionsのワークフローではデフォルトでGitHub CLIが使用可能なため、`gh pr merge`で自動マージする形としました。  
プルリク作成のステップで`id: cpr`を指定しているため、`${{ steps.cpr.outputs.pull-request-number }}`でプルリクの番号を取得可能になっています。

## まとめ

GitHubのREADME.mdをイイ感じにHTMLに変換してプッシュする方法についてまとめました。

色々な実現方法があると思いますが、意外とこの手順を紹介している記事は見つからなかったため、同じようなことを実現したい方がいれば参考になれば幸いです。
