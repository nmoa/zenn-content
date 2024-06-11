---
title: "プロキシ環境下でサーバを構築するための設定メモ"
emoji: "📑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Linux", "GitHub", "Docker", "proxy"]
published: true
---

企業内などプロキシサーバを経由してインターネットに接続する必要がある環境下でサーバを構築する際にハマったので、その解決策をメモしておく。

サーバはUbuntuを想定。

## プロキシ設定

環境変数`http_proxy`、`https_proxy`にサーバURLを設定する。

```bash:~/.profile
export http_proxy=http://proxy.example.com:8080
export https_proxy=http://proxy.example.com:8080
```

### 接続確認

`example.com`にリクエストを送信し、応答が返ることを確認する。

- HTTP

    ```
    $ curl -I http://example.com
    HTTP/1.1 200 OK
    Accept-Ranges: bytes
    Age: 436224
    Content-Type: text/html; charset=UTF-8
    ...(省略)...
    ```

- HTTPS

    ```
    $ curl -I https://example.com
    HTTP/1.1 200 Connection established

    HTTP/2 200
    accept-ranges: bytes
    age: 210559
    cache-control: max-age=604800
    content-type: text/html; charset=UTF-8
    ...(省略)...
    ```

## `sudo`

`sudo`コマンドには環境変数が引き継がれずプロキシ設定が効かないため、`http_proxy`, `https_proxy`が引き継がれるように設定を行う。

以下のコマンドで`/etc/sudoers`を編集し、

```bash
sudo visudo
```

末尾に下記の設定を追加する。

```bash:/etc/sudoers
Defaults env_keep += "http_proxy https_proxy"
```

Ubuntuの場合、デフォルトで以下のように設定がコメントアウトされているので、コメントアウトを解除しても良い。

```bash:/etc/sudoers
# This preserves proxy settings from user environments of root
# equivalent users (group sudo)
#Defaults:%sudo env_keep += "http_proxy https_proxy ftp_proxy all_proxy no_proxy"
```

## `apt`

`/etc/apt/apt.conf.d/`に適当な名前のファイル（ここでは`01proxy`）を作成し、プロキシ設定を記述する。

```bash:/etc/apt/apt.conf.d/01proxy
Acquire::http::Proxy "http://proxy.example.com:8080";
Acquire::https::Proxy "http://proxy.example.com:8080";
```

## GitHubへのSSH接続をプロキシ経由で行う

プロキシを設定したらGitHubからのHTTPSでのクローンは特別な設定なしに行えるようになるが、`public`でないリポジトリからのクローンはPersonal Access Tokenを使用する必要があり面倒なので、SSHで接続できるようにする。

1. `corkscrew`をインストールする。`corkscrew`はSSH接続をHTTPプロキシでトンネリングするツール。

    ```bash
    sudo apt install corkscrew
    ```

1. `~/.ssh/config`に以下の設定を追加する。

    ```bash:~/.ssh/config
    Host github.com
      User git
      Hostname ssh.github.com
      Port 443
      ProxyCommand corkscrew proxy.example.com 8080 ssh.github.com 443
    ```

1. GitHubにSSH接続する。

    ```bash
    $ ssh -T git@github.com
    Hi xxxx! You've successfully authenticated, but GitHub does not provide shell access.
    ```

参考 : [Proxy下でSSHを使いたい!!! #Linux - Qiita](https://qiita.com/isso0424/items/e54e528e90950b5f0c38)

## docker

dockerイメージのpull時、イメージのビルド時、コンテナの実行時でそれぞれプロキシ設定が必要。

### pull時

まずは`/etc/systemd/system/docker.service.d/http-proxy.conf`を作成する。

```bash
sudo mkdir -p /etc/systemd/system/docker.service.d
sudo vim /etc/systemd/system/docker.service.d/http-proxy.conf
```

`http-proxy.conf`にプロキシ設定を記述する。

```bash:/etc/systemd/system/docker.service.d/http-proxy.conf
[Service]
Environment="http_proxy=http://proxy.example.com:8080"
Environment="https_proxy=http://proxy.example.com:8080"
```

設定ファイルを作成したら、以下のコマンドで設定を反映させる。

```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```

参考 : [Configure the daemon with systemd | Docker Docs](https://docs.docker.com/config/daemon/systemd/#httphttps-proxy)

### イメージのビルド時

`Dockerfile`に基づいてイメージをビルドする際は、`--build-arg`オプションを使用してプロキシ設定を渡す。

```bash
docker build \
  --build-arg http_proxy=http://proxy.example.com:8080 \
  --build-arg https_proxy=http://proxy.example.com:8080 \
  -t my-image .
```

[Configure Docker to use a proxy server | Docker Docs](https://docs.docker.com/network/proxy/#proxy-as-environment-variable-for-builds) にもあるように、`Doockerfile`内で`ENV`を使用するとイメージにプロキシ設定が含まれてしまうため、`--build-arg`オプションを使用した方が良い。

### コンテナの実行時

`docker run`コマンドを実行する際に`-e`オプションを使用してプロキシ設定を渡す。

```bash
docker run -e http_proxy=http://proxy.example.com:8080 -e https_proxy=http://proxy.example.com:8080 my-image
```

`~/.docker/config.json`にプロキシ設定を記述する方法もあるが、[Docker のプロキシ設定は割と地獄](https://zenn.dev/wsuzume/articles/f9935b47ce0b55)の記事にもあるようにすべてのコンテナ実行に共有されてしまうこと、`sudo`のときは設定ファイルが違うことを考えると、実行時にオプションとして渡したほうがスッキリしそう。
