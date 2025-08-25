Apple公式「container」でMac上にLinux環境！Dockerに代わる新時代コンテナを試してみた

# 概要

先日のAppleのWWDC25にてMaC上でLinuxを実行できる「Containerizationフレームワーク」を発表されました。以下が実際に公開されたリポジトリになります。OCI準拠なので既存のDockerイマージを使用できるみたいです。
今回は実際に導入方法について実際にやりながら手順を共有できればと思います。

https://github.com/apple/container?tab=readme-ov-file


# システム要件

- チップ `Apple Silicon`
- OS `macOS 26 Bata1` or `macOS 15`

:::note info
contanierはApple SiliconのMacのみ対応しています。
:::

システム要件に合っているかは以下のようなコマンドか「Appleマーク > このMac」についてなどで調べられます。
```
uname -m
sw_vers
```

# インストール手順

#### **1\. インストーラをダウンロード**

以下のGitHubリリースページからインストーラをダウンロードします。今回は`container-0.1.0-installer-signed.pkg`をダウンロードします。(2025年6月12日現在の最新は`0.1.0`です。)

https://github.com/apple/container/releases

![スクリーンショット 2025-06-12 13.59.56.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/22c160fe-087b-4fa9-a07a-92d70ad8509a.png)

#### **2\. インストール実行**

先ほどダウンロードした`container-0.1.0-installer-signed.pkg`をダブルクリックして実行します。
すると`/usr/local/bin/container`にバイナリがインストールされます。
インストールされていることも確認できました。
```
isoda@shota-mac ~ % container --version
container CLI version 0.1.0 (build: release, commit: 0fd8692)
```

#### **3\. 初期設定**

まず、containerを使用する前にサービスを開始する必要があります。以下の通り`container system start`を実行するとLinuxカーネルのインストールを促されます。今回はおすすめされているものをインストールしようと思います。
```
isoda@shota-mac ~ % container system start
Verifying apiserver is running...
Installing base container filesystem...
No default kernel configured.                                                              
Install the recommended default kernel from [https://github.com/kata-containers/kata-containers/releases/download/3.17.0/kata-static-3.17.0-arm64.tar.xz]? [Y/n]: Y
Installing kernel...
```
次に全てのコンテナ一覧を表示するコマンドでアプリケーションが動作しているか確認してみます。
```
isoda@shota-mac ~ % container list --all
ID  IMAGE  OS  ARCH  STATE  ADDR
isoda@shota-mac ~ % 
```
まだ、コンテナを作成していないので空ですね。


# 実際に使用してみる

簡単なPython Webサーバ用のDockerfileを作成してイメージを構築してみます。
以下の通り、ディレクトリ作成し
```
mkdir web-test
cd web-test
```
Dockerfileを作成してみます。
```dockerfile
FROM docker.io/python:alpine
WORKDIR /content
RUN apk add curl
RUN echo '<!DOCTYPE html><html><head><title>Hello</title></head><body><h1>Hello, world!</h1></body></html>' > index.html
CMD ["python3", "-m", "http.server", "80", "--bind", "0.0.0.0"]
```
それでは、buildしてみます。
```
container build --tag web-test --file Dockerfile .
```
すると以下の通りイメージが表示されることがわかります。
```
isoda@shota-mac web-test % container image list
NAME      TAG     DIGEST
python    alpine  b4d299311845147e7e47c970...
web-test  latest  b26b694b1a3d0ab48dbc9c30...
```
では、実際に先ほど作成したイメージを使用してWebサーバを実行してみます。
```
container run --name my-web-server --detach --rm web-test
```
コンテナが作成されているか確認してみましょう。無事にできていそうです！
```
isoda@shota-mac web-test % container list --all
ID             IMAGE                                               OS     ARCH   STATE    ADDR
my-web-server  web-test:latest                                     linux  arm64  running  192.168.64.3
buildkit       ghcr.io/apple/container-builder-shim/builder:0.1.0  linux  arm64  running  192.168.64.2
```
では、ブラウザから確認してみましょう。コンテナのIPアドレスは`ADDR`の部分に表示されていて、今回は`192.168.64.3`ですね。
http://192.168.64.3/

![スクリーンショット 2025-06-12 14.39.28.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/53b204a2-edc5-44df-bb86-e7c836df36de.png)

コンテナの作成ができたので、コンテナを停止し後片付けしておきます。
```
isoda@shota-mac web-test % container stop my-web-server
my-web-server
isoda@shota-mac web-test % container list --all        
ID        IMAGE                                               OS     ARCH   STATE    ADDR
buildkit  ghcr.io/apple/container-builder-shim/builder:0.1.0  linux  arm64  running  192.168.64.2
isoda@shota-mac web-test % 
```
完全に停止したい場合は以下のコマンドを実行します。
```
container system stop
```



# アンインストール手順

containerをアンインストールするには以下の通りスクリプトが用意されています。`container`リポジトリをcloneしてきて実際にアンインストールしてみます。
```
git clone git@github.com:apple/container.git
cd container/scripts
uninstall-container.sh -d
```
(実行結果)
```
isoda@shota-mac scripts % uninstall-container.sh -d                                 
Password:
Removed `container` tool and helpers
Removing `container` user data
```
アンインストールのスクリプト実行時に`-k`オプションで実行すればユーザデータを保持でき、再インストール時に使用可能になるみたいです。


# 基本的な操作方法

ここからは基本的な操作方法についてまとめていきます。細かいサブコマンドなどは`--help`で確認可能なので、本記事では割愛します。

#### システム操作系

```bash
# help
container system --help

# システムの起動
container system start

# システムの停止
container system stop

# システムの再起動
container system restart
```

#### コンテナ操作系

```bash
# containerコマンドのhelp
container --help

# 実行中のコンテナ一覧
container list --all

# コンテナの作成
container create [イメージ名]

# コンテナの開始
container start [コンテナID]

# コンテナの作成と開始
# --detachでバックグラウンド実行
# --memory, --cpusでリソース制限
container run --name [コンテナ名称] --detach --memory [メモリ上限] --cpus [CPU数] [イメージ名]

# コンテナの停止
container stop [コンテナID]

# コンテナの削除
container delete [コンテナID]

# コンテナの詳細表示
container inspect [コンテナID]

# 対話シェル
container exec --tty --interactive [コンテナID] bash

# コンテナのログ確認
container logs [コンテナID]

# コンテナの強制終了
container kill [コンテナID]
```



#### イメージ操作系

```bash
# buildコマンドのhelp
container build --help

# イメージのbuild
container build --tag [タグ名] --file Dockerfile .

# imagesコマンドのhelp
container images --help

# イメージのpull
container images pull [イメージ名]

# イメージの一覧
container images list

# イメージの詳細
container images inspect [イメージ名]

# イメージのpush
container images push [イメージ名]

# イメージの削除
container images rm [イメージ名]

# イメージのタグ付け
container images tag [イメージ名:変更前タグ] [イメージ名:変更後タグ]
```

#### レジストリの設定

イメージのデフォルトレジストリは`Docker Hub`となっています。
```bash
# registryコマンドのhelp
container registry --help

# デフォルトのレジストリの確認
container registry default inspect

# レジストリにログイン(Docker Hubの場合)
container registry login docker.io -u [ユーザ名]

# レジストリログアウト(Docker Hubの場合)
container registry logout docker.io
```

#### builder関連

コンテナを初めて実行する際にbuilderが起動します。これは軽量の仮想マシンで実行されるので、大規模なビルドの場合はリソースなどの制限を増やす必要があり、それらの設定が可能になっています。
```
buildkit  ghcr.io/apple/container-builder-shim/builder:0.1.0  linux  arm64  running  192.168.64.2
```
↑自動的に作成されるbuilder
```bash
# builderの開始
container builder start --cpus [CPU数] --memory [メモリ上限]

# builderの状態確認
container builder status

# builderの停止
container builder stop

# builderの削除
container builder delete
```
