Zabbixでnginxのログからデータ可視化

## Zabbixでログデータ取得して可視化

zabbixでは、簡単にログ監視の設定を行えます。主な使い方と言ったらerrorログを検出して通知するみたいな使い方でしょうか。しかし、ログからレスポンスタイムなどを取得し、データの可視化なども可能です。

ログの取得には、以下のキーを使用します。logとlogrtの２つがありますが、違いは、ファイルパスに正規表現を使用できるかできないかの違いです。

```
logrt[file_regexp,<regexp>,<encoding>,<maxlines>,<mode>,<output>,<maxdelay>,<options>,<persistent_dir>]
log[file,<regexp>,<encoding>,<maxlines>,<mode>,<output>,<maxdelay>,<options>,<persistent_dir>]
```

上のようにたくさんパラメータがありますが、今回使用するのは、「file」「regexp」「mode」「output」
file：監視するファイル
regexp：正規表現でログの抽出する部分を指定
mode：「all：すべてのログファイルを対象にする」か「skip：アイテムを設定したあとのログファイルのみを対象にする」
output：regexpで抽出する箇所を選択する。「\1」で1番最初の「()」を「\2」で2番目の「()」の中身を抽出する。

※regexpで使用できる正規表現はバージョンによっては、違うので注意。ここで詰まった。。自分の使用しているzabbixのバージョンのドキュメント確認してみてください。
https://www.zabbix.com/documentation/5.0/en/manual/regular_expressions


## nginxのログを抜き出す

nginxのログ
```
192.168.0.1 - - [1/Jun/2022:10:44:05 +0900] "GET / HTTP/2.0" 500 21 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/102.0.0.0 Safari/537.36" "-" "TLSv1.2" "ECDHE-RSA-AES256-GCM-SHA384" 0.232 502 80
```
上記のログだとレスポンスタイムは「0.232」の部分。今回は、これを抜き出したい。ちなみにレスポンスタイムがどこの部分かは、/etc/nginx/nginx.confに記載があります。

上のログの場合は、以下のようなキーでレスポンスタイムを取得できる。値を抽出したあとのグラフ化とかは、他の記事見てください。
```
log[/var/log/nginx/nginx.log,\s(\d+\.\d{3})\s,,,skip,\1]
```
このような要領で、色々なログのデータからグラフ化とかできるようになります。

## まとめ

すごく雑な説明ですけど、zabbixでもログのデータを抽出できることがわかります。今回これを説明しておいてなんですけど、ログデータの可視化ならelasticsearch，kibana，td-agentを使用したほうがいいです。
