---
title: "GitHubのREADME.mdをイイ感じにHTMLに変換してプッシュする"
emoji: "🌊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [GitHub, "Markdown", "HTML"]
published: true
---

## 概要

社内向けのツールをGitHubで開発した際にREADME.mdにマニュアルを書いていたのですが、GitHubのアカウントを持っていない人にもマニュアルを配布しやすくするため、README.mdをイイ感じの見た目のHTMLに変換してリポジトリにプッシュすることにしました。

この記事では、その方法についてまとめます。

## 実現したいこと

今回実現したいことに関わる要件は以下の通りです：

- GitHub上のプレビューと同じように、スタイルやシンタックスハイライトが効いた見た目にする
- スタイルや画像をすべて埋め込んで、単独のHTMLファイルとして配布できるようにする
- プルリクが`master`ブランチにマージされた後にHTMLに変換し、`master`にプッシュする
  - ただし、`master`はGitHub Repository rulesetsで保護されている

## 結果

先に結果を示すと、つぎのような見た目のHTMLが生成されるようになります：

![](/images/readme-html-example.png)

（これは[nmoa/markdown-html-example](https://github.com/nmoa/markdown-html-example)のREADME.mdを変換したものです）

実現した手順は以下のとおりです：

1. [markdown-to-html-cli](https://github.com/jaywcjlove/markdown-to-html-cli/)を使用してREADME.mdをHTMLに変換する
2. [create-pull-request](https://github.com/peter-evans/create-pull-request)を使用して作成したHTMLをプッシュし、プルリクエストを作成する
3. GitHub CLIの`gh pr merge`コマンドでプルリクを自動マージする

実際のGitHub Actionsの`.yml`ファイルは以下のようになりました。

```yaml
name: Generate README.html

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

なぜこの方法を選択したのか、それぞれ解説していきます。

### トリガー

`on: push: branches: master`だけだとプルリクの自動マージによって再度ワークフローが実行されて無限ループになるのが怖かったのと、無駄にActionsを走らせないようにするために、`paths: README.md`を指定しています。  
ただし、[こちらの情報](https://github.com/ad-m/github-push-action/issues/32)によると`GITHUB_TOKEN`を使用したイベントはトリガーの対象にならないようなので、無限ループは起こらなさそうです。

### Markdown→HTML変換

細かい設定をしなくても以下の2つの要件を満たすことができるツールとして[markdown-to-html-cli](https://github.com/jaywcjlove/markdown-to-html-cli/)が最適でした：

1. GitHub上のプレビューと同じように、スタイルやシンタックスハイライトが効いた見た目にする
2. スタイルや画像をすべて埋め込んで、単独のHTMLファイルとして配布できるようにする

1については特にオプションを指定せずに対応でき、2については`--img-base64`オプションを指定することで画像を埋め込むことができます。

ただし、`markdown-to-html-cli`はGitHub Actionsでも使用可能ですが、執筆時点の`v4.1.0`ではActionsに`--img-base64`に相当するオプションがなかったため、`npm install`して実行する形にしています。

#### pandocを使用していない理由

MarkdownからHTMLに変換するツールとしては`pandoc`も有名です。また、GitHubのスタイルを実現するカスタムCSSとして[github-markdown-css](https://github.com/sindresorhus/github-markdown-css)もあり、以下のようなコマンドでスタンドアロンなHTMLを生成できるため、最初はこれを検討していました。

```bash
pandoc -f gfm -t html5 --embed-resources --standalone -c https://cdnjs.cloudflare.com/ajax/libs/github-markdown-css/5.6.1/github-markdown.css README.md -o README.html
```

しかし、以下の問題がありました：

- シンタックスハイライト以外のスタイルが適用されない
- 箇条書きの内部に作成したテーブルが正しくレンダリングされない

具体的には、以下のような見た目のHTMLとなってしまいました。

![](/images/readme-html-pandoc.png)

これらの理由から、`markdown-to-html-cli`を使用することにしました。

### プルリクエスト作成

`master`ブランチはGitHub Repository rulesetsで直接のプッシュを禁止していたため、プルリクエストを作成して自動でマージすることにしました。  
[Deploy Keys](https://docs.github.com/ja/authentication/connecting-to-github-with-ssh/managing-deploy-keys#deploy-keys)を作成してGitHub Repository rulesetsの[Bypass list](https://docs.github.com/ja/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/creating-rulesets-for-a-repository#granting-bypass-permissions-for-your-branch-or-tag-ruleset)に追加することで直接のプッシュも可能だったかもしれませんが、このためだけにDeploy Keysを作成する必要はないと判断し、この方法を選択しました。

プルリクの作成には[create-pull-request](https://github.com/peter-evans/create-pull-request)アクションを使用しました。  
`labels: github-actions`でPRにラベルを付与することで、自動生成したプルリクを識別しやすくしています。

### プルリクエストの自動マージ

GitHub ActionsのワークフローではデフォルトでGitHub CLIが使用可能なため、`gh pr merge`コマンドで自動マージする形としました。  
プルリク作成のステップで`id: cpr`を指定しているため、`${{ steps.cpr.outputs.pull-request-number }}`でプルリクの番号を取得可能になっています。

## まとめ

GitHubのREADME.mdをイイ感じにHTMLに変換してプッシュする方法についてまとめました。

様々な実現方法があると思いますが、意外とこの手順を紹介している記事は見つからなかったため、同じようなことを実現したい方の参考になれば幸いです。  
[nmoa/markdown-html-example](https://github.com/nmoa/markdown-html-example)のリポジトリも参考にしてみてください。
