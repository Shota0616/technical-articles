elasticsearch，kibana，td-agentでログデータの可視化において型がstringになってしまう問題について

# 問題
td-agentでログを取得してkibanaで可視化しようとするときに、ちゃんと設定できていないと取得したログすべてがstring型になってしまいvisualizeなどで可視化できない可能性がある。

# 解決方法
そんなときの解決方法を解説する。
大まかに方法は以下の2つがある。
* elasticsearch（es）のmapping設定
* td-agentのtype設定

## esのmapping設定
自分的には、こちらを推奨している。（なんとなく型情報の管理がしやすそう）
まずesの前提知識として、以下を覚えておいてほしい。
* 既存indexのmappingは削除、変更はできない。（追加は可能）
* mapping情報のないfieldをもつドキュメントがindexに登録されたときは、esのdynamic mappingによってよしなにfieldの型を判断してくれる。（つまり勝手に型情報が定義されちゃう）
* index templateを設定しておけば新しく作られるindexには自動的にmapping情報を登録してくれる。
* 既存のindex patternも削除、変更はできない。（追加は可能）あと、indexを再作成したらindex patternも再作成する必要がある。

具体的にindexにmappingを登録する方法は、いくつか存在するが今回は、kibana上のdev-toolsのコンソールから操作を行うことを想定する。（esのapi叩くときの文法とそんなに変わらないと思う。）
```
PUT index/_mapping
{
"properties": {
  "field name":{
    "integer"
  ],
  "field name":{
    "float"
  ],
}
}
```
※index：indexの名称
※field name：任意のtd-agentから送信されてくるデータのfield名称

実際は、この作業だけじゃうまく行かない事がある。ていうかほぼうまく行かないと思うので、上の方に記載した、esの前提知識をよく見てほしいこれが結構重要になってくる。それでもできなかったらtd-agent側で設定を変更する。

ちなみにコンソールに以下を打ち込むとindexのmapping情報を取得もできる
```
GET index/_mapping
```

## td-agentのtype設定
これは、td-agent側で送信するログfield情報をあらかじめ定義してesに突っ込み、esのdynamic mappingでよしなに型を定義してもらおう的な感じです。（パスは確か/etc/td-agent/td-agent.conf）

設定変更する場所は、td-agentの設定ファイルの<source>の部分に以下のtypesというパラメータを追加する。
```
<source>
  type tail
  path /var/log/nginx/access.log
  pos_file /var/log/buffer/nginx/access.pos
  tag nginx.access
  format /^(?<remote>[^ ]*) (?<host>[^ ]*) (?<user>[^ ]*) \[(?<time>[^\]]*)\] "(?<method>\S+)(?: +(?<path>[^\"]*?)(?: +\S*)?)?" (?<code>[^ ]*) (?<size>[^ ]*)(?: "(?<referer>[^\"]*)" "(?<agent>[^\"]*)"(?:\s+(?<http_x_forwarded_for>[^ ]+))?)?$/
  time_format %d/%b/%Y:%H:%M:%S %z
  types code:integer,size:integer    #これを追加する
</source>
```

ちなみに上のformatの部分を変更することで、様々なログファイルの切り分けが可能になる。ここの正規表現は、普通の正規表現と違って複雑なので、後日記事にするかもしれない。

# まとめ
esは結構な厄介者です。検証環境作ってあげて色々といじってあげるのが、一番勉強になる気がします。時間があればes,kibanaの検証環境の作成方法なども記事にします。
