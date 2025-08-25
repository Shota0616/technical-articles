「不審なメールのリンク」の危険性を技術的に理解する。DVWAで学ぶCSRF攻撃のメカニズム


# 概要

以前、DVWAを使用して実際にSQLインジェクションとXSSを実践してみましたが、今回はCSRF（Cross Site Request Forgery）を試してみます。
CSRFとは、webアプリケーションでログインしているユーザが意図しない悪意のあるリクエストを送り、それをサーバ側が処理してしまうというような問題のことを指します。
> ウェブサイトの中には、サービスの提供に際しログイン機能を設けているものがあります。ここで、ログインした利用者からのリクエストについて、その利用者が意図したリクエストであるかどうかを識別する仕組みを持たないウェブサイトは、外部サイトを経由した悪意のあるリクエストを受け入れてしまう場合があります。このようなウェブサイトにログインした利用者は、悪意のある人が用意した罠により、利用者が予期しない処理を実行させられてしまう可能性があります。このような問題を「CSRF（Cross-Site Request Forgeries／クロスサイト・リクエスト・フォージェリ）の脆弱性」と呼び、これを悪用した攻撃を、「CSRF攻撃」と呼びます。
出典 : https://www.ipa.go.jp/security/vuln/websecurity/csrf.html

:::note info
この記事は人力で記載しています。
:::

https://qiita.com/shota0616/items/ef7125bcf5c2721c34b0

https://qiita.com/shota0616/items/3e3ee02bc113483a2317



# 手順

### 現状のパスワード変更のリクエストを確認

まずは、アプリケーションのCSRF部分の動作の確認をしていきます。

`Test Credentials`を押下して現状のadminユーザのパスワードを確認してみましょう。

![スクリーンショット 2025-07-20 8.11.23.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/e2ec3aa0-3fcd-49b6-b0bd-9e0702af8add.png)

デフォルトのままなので、以下の通り入力すれば`Valid password for 'admin'`と表示され、adminユーザのパスワードが`password`となっていることがわかります。
```
Username: admin
Password: password
```

![スクリーンショット 2025-07-20 8.15.16.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/7b4dfb14-14a0-4831-bc41-b2513bae13ae.png)

続いて、`New password:`と`Confirm new password:`に`newpass`と入力してパスワード再設定をしてみます。

![スクリーンショット 2025-07-20 8.22.39.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/643c2cf4-2301-4f8c-b88e-f90c24f46eb9.png)

すると、以下のようなリンクに遷移して、URLのパラメータにパスワードがそのまま入ってしまっていることがわかります。
http://localhost:4280/vulnerabilities/csrf/?password_new=newpass&password_conf=newpass&Change=Change#
ブラウザの検証ツールなどを使用してどのようなリクエストを送っているか確認してみたところGETリクエストでパスワードの変更を行っていそうです。
```
Request URL       http://localhost:4280/vulnerabilities/csrf/?password_new=newpass&password_conf=newpass&Change=Change
Request Method    GET
Status Code       200 OK
Remote Address    127.0.0.1:4280
Referrer Policy   strict-origin-when-cross-origin
```

DVWAのサーバ側のログでもリクエストの確認ができました。
```nginx
192.168.97.1 - - [19/Jul/2025:22:19:19 +0000] "GET /vulnerabilities/csrf/?password_new=newpass&password_conf=newpass&Change=Change HTTP/1.1" 200 2471 "http://localhost:4280/vulnerabilities/csrf/" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/138.0.0.0 Safari/537.36"
```

再度adminユーザのパスワードが変わっているかを確認するために、`Test Credentials`で確認してみましょう。以下の通り入力すれば`Valid password for 'admin'`と表示され、adminユーザのパスワードが`newpass`となっていることがわかります。
```
Username: admin
Password: newpass
```
DBでも確認してみましたが、パスワードハッシュもちゃんと変わっていそうです。
```sql
--- before
MariaDB [dvwa]> select user,password from users where user = "admin";
+-------+----------------------------------+
| user  | password                         |
+-------+----------------------------------+
| admin | 5f4dcc3b5aa765d61d8327deb882cf99 |
+-------+----------------------------------+
1 row in set (0.004 sec)

--- after
MariaDB [dvwa]> select user,password from users where user = "admin";
+-------+----------------------------------+
| user  | password                         |
+-------+----------------------------------+
| admin | e6053eb8d35e02ae40beeeacef203c1a |
+-------+----------------------------------+
1 row in set (0.009 sec)
```

---

### 攻撃用のページを作成する

攻撃者は、先ほどのパスワードをURLパラメータに含んだリンクを悪用して、サイトやメールを作成してそのリンクを踏ませようとします。

まず、aタグを踏むと脆弱性のあるwebアプリケーションに遷移します。imgタグはユーザがページを開くだけで自動的に画像を読み込もうとするので、バックグラウンドで攻撃が実行されてしまいます。

<p class="codepen" data-height="300" data-default-tab="html,result" data-slug-hash="PwPZOwb" data-pen-title="Untitled" data-user="gluvomsj-the-sasster" style="height: 300px; box-sizing: border-box; display: flex; align-items: center; justify-content: center; border: 2px solid; margin: 1em 0; padding: 1em;">
  <span>See the Pen <a href="https://codepen.io/gluvomsj-the-sasster/pen/PwPZOwb">
  Untitled</a> by 磯田将太 (<a href="https://codepen.io/gluvomsj-the-sasster">@gluvomsj-the-sasster</a>)
  on <a href="https://codepen.io">CodePen</a>.</span>
</p>
<script async src="https://public.codepenassets.com/embed/index.js"></script>

**※ここのhtmlとjsはAIを用いて生成しました。**

1\. DVWAにログインしたまま、上記のhtmlの「ここをクリック！」の部分を押下

2\. DVWAから一旦ログアウトして、adminユーザのパスワード`newpass`でログインできるか確認してみます。**なんと`newpass`というパスワードを入力しても`Login failed`となってしまい、adminユーザのログインができなくなってしまいました。。**
![スクリーンショット 2025-07-20 8.51.26.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/bd3b55a4-b4d5-4bff-9dd1-26e36d3a7a8f.png)

3\. 続いて、攻撃者が攻撃用リンクのパスワードパラメータに設定した`hacked`でログインできるか確認してみます。すると、、ログインできてしまいました。<font color="red">**これで、攻撃者はadminユーザの乗っ取り完了です。**</font>
![スクリーンショット 2025-07-20 8.50.23.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/172ab6ce-f93f-42f5-9bcf-ec5ac9c50f7e.png)


# 基本的な対策について

https://www.ipa.go.jp/security/vuln/websecurity/csrf.html

基本的な対策については、以下のようなものがありますが、IPAのサイトにわかりやすく記載あったのでリンクしておきます。
* CSRFトークンの導入
* リクエストのRefererヘッダーのチェック
* SameSite Cookie属性
* CAPTCHAの導入
* MFA認証
