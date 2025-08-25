Dockerの基本操作メモ

# 概要

Dockerのイメージやコンテナの基本操作についてまとめます。

# イメージの操作

イメージのビルド
```
docker image build -t [イメージ名:タグ名]　[Dockerfileを配置しているディレクトリパス]
```

イメージの取得
```
docker image pull [イメージ名:タグ名]
```

イメージ一覧
```
docker image ls
```
`docker image ls`の実行例
```
docker image ls
REPOSITORY  TAG     IMAGE ID       CREATED         SIZE
nginx       latest  a6bd71f48f68   4 days ago      187MB
python      latest  caacd5b6b541   5 weeks ago     1.02GB
ubuntu      latest  e4c58958181a   7 weeks ago     77.8MB
```
表示内容詳細
| 項目名 | 内容 | 備考 |
|:-----------|:------------|:------------:|
| REPOSITORY | リポジトリ名 |  |
| TAG | タグ名 |  |
| IMAGE ID | イメージID |  |
| CREATED | イメージ作成からの経過時間 |  |
| SIZE | イメージのサイズ |  |

使用していないイメージの一括削除
```
docker image prune
```
イメージのpush（docker hubに対するpush）
```
docker image push
```

# コンテナの操作


コンテナの実行（基本形）
```
docker container run -d -it [イメージ名 or イメージID]
```
コンテナ実行時のオプション
| オプション名 | 内容 | 備考 |
|:-----------|:------------|:------------:|
| --name | 実行コンテナの名前をつける | 開発用途ではよく利用されるが、本番ではあまり使用しないかも。同じ名前のコンテナは作成できないので |
| -d | バックグラウンドで起動（デーモン化） |  |
| -p | コンテナのポート指定 | `8080:80`とか指定するとホストの8080ポートへのアクセスはコンテナの80番にフォワードする |
| -i | docker起動後にコンテナ側の標準入力をつなぎっぱなしにする |  |
| -t | 疑似端末を有効にする | -iがついていないと端末を起動しても入力できないので-itでセットで使う事が多い |
| -rm | コンテナ停止と同時にコンテナを破棄する |  |
| -v | Data Volumeの作成 | `ホスト側ディレクトリ:コンテナ側ディレクトリ`で指定する |



コンテナの一覧表示
```
docker container ls
```
オプション
| オプション名 | 内容 | 備考 |
|:-----------|:------------|:------------:|
| --filter | lsの結果を指定の条件でfiletrする | コンテナ名で抽出するときは`"name=xxx"`、イメージで抽出するときは、`"ancestor=xxx:xx"` |
| -a | 終了したコンテナを取得する |

`docker container ls`の実行例
```
docker container ls    
CONTAINER ID   IMAGE     COMMAND       CREATED         STATUS         PORTS     NAMES
df8883a1952a   ubuntu    "/bin/bash"   3 minutes ago   Up 3 minutes             elated_nash
```
表示内容詳細
| 項目名 | 内容 | 備考 |
|:-----------|:------------|:------------:|
| CONTAINER ID | コンテナに付与されるユニークID |  |
| IMAGE | コンテナ作成に使用されているイメージ |  |
| COMMAND | コンテナで実行されているプロセス |  |
| CREATED | コンテナ実行からの経過時間 |  |
| STATUS | コンテナの状態 |  |
| PORTS | ホストとコンテナのポート | `-p`で指定したものオプションなしならポートフォワードもなし |
| NAMES | コンテナの名前 | `--name`オプション使用すた場合は、`--name`で指定したもの |

コンテナの停止
```
docker container stop [コンテナ名 or コンテナID]
```
コンテナの再起動
```
docker container restart [コンテナ名 or コンテナID]
```
コンテナの破棄
```
docker container rm [コンテナ名 or コンテナID]
※基本停止中のコンテナしか破棄できないが、`-f`オプションで強制破棄できる。
```
コンテナ標準出力の取得
```
docker container logs [コンテナ名 or コンテナID]
```
実行中のコンテナでコマンド実行
```
docker container exec [コンテナ名 or コンテナID] [xxx(実行するコマンド)]

（以下のようにするとあたかもコンテナ内にSSHしたかのような使い方もできる）
docker container exec -it コンテナIDかコンテナ名 bash
```
ホスト,コンテナ間のファイルコピー
```
docker container cp [コンテナ名 or コンテナID]:[コンテナ内のコピー元] [ホスト側のコピー先]
docker container cp [ホスト側のコピー元] [コンテナ名 or コンテナID]:[コンテナ内のコピー先]
```
使用していないコンテナの一括削除
```
docker container prune
```
コンテナ単位のシステムリソース利用状況
```
docker container stats
```


# Dockerfileの記載方法

オリジナルのイメージを作成するには、Dockerfileを書く必要があるのでnginxのオリジナルイメージを作成を例に説明します。githubにおいておきました。（かなり簡易的なものです。）


https://github.com/Shota0616/nginx-dockerfile
```dockerfile
# dockerイメージのベースとなるイメージを指定、最新の安定版1.24.0をタグ指定
# https://nginx.org/en/download.html
FROM nginx:1.24.0

# コンテナに入って操作するときvim使いたいので、aptでインストールしておく
RUN apt update && RUN apt install -y vim

# ホスト側で用意したnginx設定ファイルをコンテナ側にコピー
COPY ./conf/default.conf /etc/nginx/conf.d/default.conf
# 初期のindex.htmlを送付
COPY ./index.html /usr/share/nginx/html/index.html

# 公開ポートを指定、今回はwebなので80番開けておく
EXPOSE 80

# デーモンはオフにしてフォアグラウンドで実行
CMD ["/usr/sbin/nginx", "-g", "daemon off;"]
```
上記のリポジトリをcloneしてもらったディレクトリで以下のコマンドを実行するとイメージがbuildされます。
```
# git clone
git clone git@github.com:Shota0616/nginx-dockerfile.git

# イメージのbuild
docker image build -t nginx-base .
```
Dockerfileの説明
| 項目名 | 内容 | 備考 |
|:-----------|:------------|:------------:|
| FROM | Dockerイメージのベースとなるイメージを指定 |  |
| RUN | Dockerイメージ作成時に実行されるコマンドを指定 |  |
| COPY | ホスト上のファイルやディレクトリなどをコンテナ内にコピー |  |
| EXPOSE | コンテナの開放ポートを指定 |  |
| CMD | コンテナ実行時に実行するコマンドを指定 |  |

# その他

使用していないdockerリソースの一括削除（イメージ、コンテナ、ボリューム、ネットワーク）
```
docker system prune
```

Docker Hubのログイン
```
docker login -u [ユーザーID] -p [パスワード]
```

kubernetes, kubectlのインストール
```
# linux(ubuntu)
sudo apt-get update && sudo apt-get install -y apt-transport-https gnupg2
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee -a /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubectl

# linux(centos)
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
yum install -y kubectl

# macos
brew install kubectl

# Windows
curl -LO https://dl.k8s.io/release/v1.28.4/bin/windows/amd64/kubectl.exe
```

Exitedしてしまったコンテナに入る
```
# Exitedしてしまったコンテナをコミット
docker commit [コンテナID] [適当な名前]

# コンテナに入る
docker run --rm -it [適当な名前] bash
```
