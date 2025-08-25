CSRの発行方法とSSL証明書の確認方法

# CSRの発行方法

CSR（Certificate Signing Request）とは、SSL証明書を発行するための証明書署名要求のことです。SSL証明書を発行するにはこのCSRが必要です。

1\. 秘密鍵の作成
```
# opensslが入っていない場合はインストールしてください。
# 秘密鍵はRSAの長さは2048bitで作成しました。ここは適宜変えても大丈夫です。
# 作成先のディレクトリも適時変えてください。

openssl genrsa 2048 > /tmp/ssl.key
```
2\. CSRの作成
```
# 先ほど作成した秘密鍵を使用してCSRを作成します。

openssl req -new -key /tmp/ssl.key -sha256 -out /tmp/ssl.csr
```
3\. コマンドを実行すると以下のような入力画面になります。
```
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:
State or Province Name (full name) [Some-State]:
Locality Name (eg, city) []:
Organization Name (eg, company) [Internet Widgits Pty Ltd]:
Organizational Unit Name (eg, section) []:
Common Name (e.g. server FQDN or YOUR name) []:               
Email Address []:

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
```
4\. 入力内容については、以下に記載します。
| 項目 | 内容 |
|:-----------|:----------- |
| Country Name (2 letter code) [AU] | 国名（例：JP） |
|State or Province Name (full name) [Some-State]|都道府県（例：Tokyo）|
|Locality Name (eg, city) []|市区町村（例：Minato-ku）|
|Organization Name (eg, company) [Internet Widgits Pty Ltd]|組織名（例：company）|
|Organizational Unit Name (eg, section) []|部署名（例：section）|
|Common Name (e.g. server FQDN or YOUR name) []|FQDN（例：xxx.com）|
|Email Address []|これは特に入力しなくて良いです。|
|A challenge password []|これは特に入力しなくて良いです。|
|An optional company name []|これは特に入力しなくて良いです。|

5\. 作成したCSRの確認
```
openssl req -in /tmp/ssl.csr -text -noout
```
6\. 上記コマンドを実行すると色々と表示されますが、以下確認すれば概ねOKです
```
Subject: C = JP, ST = Tokyo, L = Minato-ku, O = company, OU = section, CN = xxx.com
```

# Let's Encryptで証明書取得

証明書の取得は以下を参考にしてみてください。
Let's Encryptだと実際にCSRは作成する必要ないので、今回作成したCSRは使用しません。

https://qiita.com/shota0616/items/90a5d8eef0ed5a362530

# SSL証明書の確認

実際にサーバに導入する前に、以下のようなことを確認する必要があります。

- 証明書の内容があっているか
- 中間証明書の内容はあっているか
- 証明書と中間証明書の結合後の確認

| ファイル名 | 内容 |
|:-----------|:----------- |
|cert.pem|SSL証明書|
|chain.pem|中間証明書|
|fullchain.pem|SSL証明書と中間証明書結合後|

### 証明書の内容があっているか

subjectのCNが想定通りか確認します。また、期日も問題ないか確認しましょう。

```
openssl x509 -noout -text -in cert.pem
```
```
        Issuer: C = US, O = Let's Encrypt, CN = R3
        Validity
            Not Before: Apr  1 13:35:11 2024 GMT
            Not After : Jun 30 13:35:10 2024 GMT
        Subject: CN = xxx.com
```

### 中間証明書の内容があっているか

中間証明書のSubjectと証明書のIssuerのCNが一致していることを確認します。また、期日が変じゃないかも確認します。

```
openssl x509 -noout -text -in chain.pem
```
```
        Issuer: C = US, O = Internet Security Research Group, CN = ISRG Root X1
        Validity
            Not Before: Sep  4 00:00:00 2020 GMT
            Not After : Sep 15 16:00:00 2025 GMT
        Subject: C = US, O = Let's Encrypt, CN = R3
```


### 証明書と中間証明書の結合後の確認

証明書を結合したあとに内容が問題ないか最低限、SubjectとIssuerと期日とかは確認してください。

```
openssl crl2pkcs7 -nocrl -certfile fullchain.pem | openssl pkcs7 -print_certs -text -noout
```
