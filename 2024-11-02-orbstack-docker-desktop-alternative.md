Docker Desktopの代替として注目されているOrbStackについてまとめてみた


# OrbStackとは
OrbStackは、軽量で高パフォーマンスな仮想化プラットフォームで、主にmacOS向けに提供されています。DockerコンテナやLinux仮想マシンを高速で動作させることができ、特にAppleシリコン（M1/M2）Macでの利用に最適化されています。Docker Desktopに代わるツールとして注目されており、リソース効率が良く、システムの負荷が抑えられるのが特徴です。

<br>

**「なぜOrbStackを選ぶか？」**
- ⚡️ **超高速** : 2秒で起動、最適化されたネットワークとファイルシステム、高速なx86エミュレーション。
- 💨 **超軽量** : 低CPUとディスク使用量、バッテリーに優しく、少ないメモリでも動作、ネイティブのSwiftアプリ。

- 🍰 **シンプルで手間いらず** : 自動でドメイン名とマイグレーションを設定、CLIでコンテナ・イメージ・ボリュームファイルにアクセス、VPNとSSHをサポート。

- ⚙️ **パワフルな機能** : Dockerコンテナ、Kubernetes、Linuxディストリビューションを実行可能。メニューバーからコンテナを素早く管理し、ボリュームやイメージファイルを探索できます。

<br>

https://orbstack.dev/

## 1. OrbStack と Docker Desktop の違い

macでdocker使用するとなったら大体の人はDocker Desktopとなると思うので比べてみます。

| 特徴              | Orbstack                          | Docker Desktop                   |
|------------------|----------------------------------|----------------------------------|
| 対応OS           | macOS (一部Linux対応予定)        | macOS、Windows、Linux            |
| パフォーマンス    | 軽量・高速                        | やや重め                          |
| リソース消費     | 少なめ                            | 比較的多め                        |
| ネイティブな統合 | ネイティブでmacOSと統合          | macOS・Windows統合だが負荷高め    |


OrbStackはDocker Desktopに比べて、メモリやCPUの使用効率が良く、特にAppleシリコンMacでパフォーマンスが高いとされています。

### 料金体系

- Docker Desktopでは以下の条件に一致する企業・団体、及び個人では無償で使用することができます。非営利の教育目的なども無償利用の対象となります。

:::note info
以下どちらも含むとき
・従業員が 251 人未満 
・年間収入が 1,000万米ドル 未満
:::
※2024年11月4日現在

- OrbStackでは以下の条件に一致する企業・団体、及び個人では無償で使用することができます。非営利の教育目的なども無償利用の対象となります。

:::note info
以下どちらも含むとき
・フリーランサーとして、営利団体、非営利団体、政府機関などで専門的に使用していない
・年間収入が 1万米ドル 未満
:::
※2024年11月4日現在

どちらも有償プランについては公式サイトを参照お願いします。（プランの詳細については頻繁に変更される可能性があるので）

https://www.docker.com/ja-jp/pricing/

https://orbstack.dev/pricing

## 2. OrbStackのシステムリソースの使用

- **CPU**
OrbStackにおけるCPUの使用については、アイドル時は約0.1%しか使用せず、コンテナとマシンを実行するとCUPがオンデマンドで使用されます。CPUの使用量を設定で制限することが可能です。
- **メモリ**
OrbStackにおけるメモリ使用については、オンデマンドで使用されるため必要に応じて増加していきます。動的にメモリ割り当てが行われ未使用のメモリは自動的に解放されます。メモリの使用量も設定で制御することが可能です。
- **ストレージ**
OrbStackの新規インストールでは、10M未満のディスク領域しか使用されません。動的なディスク管理により、オーバーヘッドがほとんどなく、必要に応じてディスク使用量が追加されます。




## 3. OrbStackのベンチマーク

ベンチマークについては、公式ページにDocker Desktop比較がありましたので、そちらを参照お願いします。

https://orbstack.dev/#benchmarks

軽く上記の内容を解説すると、以下のようになります。公式のベンチマークだけ見るとかなり軽くなっていることが分かります。

| 比較項目              | Orbstack                          | Docker Desktop                   |
|------------------|----------------------------------|----------------------------------|
| 開発環境のプロビジョニングにかかる時間 | 17分 | 45分 |
| arm64 および amd64 イメージのビルド時間 | 7分 | 19分 |
| Traefik と Grafana を使用したバックグラウンド電力使用量 | 27 mW | 123 mW |
| バックグラウンド電力使用量 | 82 mW | 137 mW |


## 4. OrbStackのデメリット

OrbStackにもデメリットはあり、主に以下のような内容になります。

- **LinuxやWindowsのサポート不足**: 現在はmacOS専用のため、LinuxやWindows環境での利用はサポートされていません。
- **Docker Composeの互換性問題**: Docker Desktopと比べて一部のDocker Compose機能に互換性がない場合があります。
- **一部のDocker拡張が非対応**: Docker Desktopで提供される一部のエクステンションや機能がサポートされていない可能性があります。


## 5. OrbStackのインストール方法

OrbStackのインストールは公式サイトからインストーラをダウンロードし実行するだけで完了します。

1. [OrbStack公式サイト](https://orbstack.dev) にアクセスします。
2. 「Download」ボタンをクリックしてインストーラをダウンロードします。
3. ダウンロードしたインストーラを開き、指示に従ってインストールを進めます。
4. インストール後は、ターミナルから`orb`コマンドを使用してコンテナやVMの管理が可能です。

OrbStackはDockerと同様にCLIから操作できるため、通常のDockerワークフローを活用できます。


## 6. OrbStackでのNginxコンテナ構築例

OrbStackを使ってNginxコンテナを構築し、アクセスするまでの手順を説明します。

### 必要なファイル

1\. **`Dockerfile`**  
Nginxイメージをカスタマイズしたい場合は、以下の`Dockerfile`を作成します。

```dockerfile
# Nginxをベースイメージとして指定
FROM nginx:latest

# カスタムの設定ファイルを追加
COPY ./default.conf /etc/nginx/conf.d/default.conf
```

2\. **`default.conf`**
カスタムのNginx設定ファイルを作成します。この設定で、Nginxがリクエストを受け付けるポートや静的ファイルのルートディレクトリを指定します。

```nginx
server {
    listen 80;
    server_name localhost;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }
}
```

3\. **`docker-compose.yml`**

```docker
version: '3'
services:
  nginx:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8080:80" # ローカルの8080ポートからNginxの80ポートにマッピング
    volumes:
      - ./html:/usr/share/nginx/html # 静的ファイルのマウント
```

4\. **`html/index.html`**

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Welcome to Orbstack Nginx</title>
</head>
<body>
    <h1>Hello from Orbstack!</h1>
</body>
</html>
```

### 実行手順

1\. 必要なファイル（Dockerfile, default.conf, docker-compose.yml, html/index.html）を作成したら、Orbstack上で以下のコマンドを実行してコンテナを起動します。
```bash
docker compose up -d
```

2\. コンテナが起動したら、ブラウザで http://localhost:8080 にアクセスして、Nginxのページが表示されるか確認します。

3\. 起動中のコンテナを確認する場合は、以下のコマンドを使用します。
```bash
docker ps -a
```
4\. コンテナを停止する場合は以下のコマンドを実行します。
```bash
docker compose down
```

# まとめ

OrbStackは、macOSユーザーにとって効率的で高速なDocker代替環境だと思います。Appleシリコン向けに最適化されており、軽量でリソース効率が高く、Docker Desktopと比較してメリットが多くあります。
Docker Desktopが重くて困っている方は是非試してみてください。
