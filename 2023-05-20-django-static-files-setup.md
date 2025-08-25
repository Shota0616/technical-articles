Djangoのstaticファイルの設定について

## 概要

Djangoのstaticファイルの設定についてまとめます。staticファイルとは、cssファイル,jsファイル,画像などの動的ではないファイルのことです。

公式ドキュメントは以下になります。(ver4.2)

https://docs.djangoproject.com/en/4.2/howto/static-files/

各種STATICの変数の公式ドキュメントは以下になります。

https://docs.djangoproject.com/en/4.2/ref/settings/#static-root

## staticファイルの設定値について

他にも`STATICFILES_STORAGE`や`STATICFILES_FINDERS`もあるようですが、今回は扱いません。

| 設定 | 詳細 |
|:-----------|------------:|
| STATIC_URL | staticファイルの配信用のURL |
| STATICFILES_DIRS | staticファイルのプロジェクト内の保存先 |
| STATIC_ROOT | 本番配信用のstaticファイルの保存先 |

## STATIC_URL

`STATIC_URL`は配信用のURLで、`STATIC_ROOT`に配置されているstaticファイルを参照するためのURLです。
もし`STATIC_URL = '/static/'`と指定した場合以下のようなurlでstaticファイルにアクセスできるようになります。

```
http://static.example.com/static/...
```

## STATICFILES_DIRS

Djangoプロジェクト内のstaticファイルの保存先を指定する変数です。この変数には、複数設定することができます。
(例)
```
STATICFILES_DIRS = [
    "/home/special.polls.com/polls/static",
    "/home/polls.com/polls/static",
    "/opt/webfiles/common",
]
```
このときもし同様のファイルが存在したときは、下に書いた設定のものが優先されるようです。

staticファイルをtemplateファイルで使用するときは、以下のように記載します。
```html
<!DOCTYPE html>
<!-- staticファイルの読み込み,ここでSTATICFILES_DIRSに指定したディレクトリが読み込まれる -->
{% load static %}
<html>
  <head>
    <meta charset="utf-8">
      <title>テストtemplate</title>
      <!-- STATIC_DIR/css/style.cssを読み込む -->
      <link rel="stylesheet" href={% static "css/style.css" %}>
  </head>
  <body>
    ....
  </body>
</html>
```

## STATIC_ROOT

先ほど、`STATICFILES_DIRS`に指定したディレクトリに静的ファイルを読み込むと言いましたが、本番環境では、staticファイルはWebサーバがレスポンスするような動作になるのが基本だと思います。
そこで、`STATIC_ROOT`です。
もし、`STATIC_ROOT = '/var/www/html/static'`と指定した場合以下のコマンドを実行することで、`/var/www/html/static/`にstaticファイルがまとめられます。
```
$ python manage.py collectstatic
```

上記のようにすることで、アプリケーションが存在するディレクトリと違う場所にstaticファイルをまとめられるので、セキュリティ上も良いと思います。

