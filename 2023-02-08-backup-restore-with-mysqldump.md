djangoで素のSQLを実行する

## 概要

djangoを使用していて複雑なクエリでDBからデータを取得したとき、ORMだけではどうにもできない時があります。そんなときのためにSQLを直接実行する方法を紹介します。
素のSQLを実行する方法としては、以下の方法が考えられます。
* extra
* raw
* cursor(これは、モデルを迂回してSQLでUPDATEなどもできます)

:::note warn
警告
これは最終手段なので、ORMで取得できるならそうした方がいいです。SQLインジェクション攻撃の種になる可能性があります。ユーザーが制御できるパラメータはエスケープする必要があります。
:::

以下は、例で使用するモデルです。
```python
class MyUser(AbstractBaseUser):
    email = models.EmailField(max_length=255, unique=True)
    first_name = models.CharField(max_length=30)
    last_name = models.CharField(max_length=30)
```

## extra

以下構文になります。先程あげた3つの中では一番素のSQLからは遠い存在です。（一番マシだと思います。）

```python
extra(select=None, where=None, params=None, tables=None, order_by=None, select_params=None)
```

**例**（あくまで例なので、下のように簡単なものでは、extraを使用するべきではありません。）
```python
queryset = MyUser.objects.extra(where = ['id in (1, 2, 3, 4) AND email = %s'], params = ['test@mail.local'])  
# printすると以下のように取得できていることが確認できます。
print(queryset)
<QuerySet [<MyUser: test@mail.local>]>     
```

## raw
SELECTだったら`raw`で実装すると良いと思います。UPDATE文は流せませんが、SELECTなら可能です。（取得はできる）
UPDATEまで、SQLで流したい場合は後述の`cursor`を使用するべきです。
※rawは`RawQuerySet`で返ってくるので、注意が必要です。

```python
raw(raw_query, params=(), translations=None)
```

**例**
```python
rawqueryset = MyUser.objects.raw('SELECT * FROM user_myuser WHERE id in (1, 2, 3)')
# こちらもprintで取得結果が確認できます。RawQuerySetで取得している事がわかります。
print(rawqueryset)
<RawQuerySet: SELECT * FROM user_myuser WHERE id in (1, 2, 3)>
```
rawを使用して`QuerySet`で取得する方法も記載しておきます。（selectでprimary keyを指定してfilterでquerysetを取得しています。）
```python
from django.db.models.expressions import RawSQL

queryset = MyUser.objects.filter(id__in=RawSQL('SELECT id FROM user_myuser WHERE id in (1, 2, 3)', []))
# これならquerysetで取得できます。
print(queryset)
<QuerySet [<MyUser: test@mail.local>]>
```

## cursor

rawでも要求を満たせないときは`cursor`を使用します。これは、モデルにマップできないクエリを扱ったり、UPDATE、INSERT、DELETEも直接実行することができます。
※cursorでは、dict型ではなくlist型で返されるので、dictで取得したいときは、別途関数を用意する必要があります。

**例**
```python
from django.db import connection

cursor = connection.cursor()
cursor.execute("UPDATE user_myuser SET email = 'testcursor@mail.local' WHERE email = 'test@mail.local'")

# selectなどの結果を取得したいときは、以下の様にします。
cursor.execute("SELECT * FROM user_myuser WHERE id in (1, 2, 3)")
list = cursor.fetchone()
# printしてみるとlistで取れている事がわかります。
print(list)
(1, 'test@mail.local', 'テスト', '太郎')
```

dictで取得したいときは、関数を定義する必要があります。以下が例になります。
```python
def dictfetchall(cursor):
    columns = [col[0] for col in cursor.description]
    return [
        dict(zip(columns, row))
        for row in cursor.fetchall()
    ]

# 上記の関数を定義してlistの時と同様に取得します。
from django.db import connection

cursor = connection.cursor()
cursor.execute("SELECT * FROM user_myuser WHERE id in (1, 2, 3)")
dictfetchall(cursor)
# 上記実行するとdictで取得できていることがわかります。
[{'id': 1, 'email': 'testcursor@mail.local', 'first_name': 'テスト', 'last_name': '太郎'}]
```

## まとめ

こんな感じでdjangoでも素のSQLを実行することができます。何度も言うようですが、**素のSQLを実行するのは、最終手段です。** 基本はDjango ORMのツールを利用してください。もし今回のような手段でデータ操作を行う場合は、SQLインジェクションにくれぐれも注意してください。
また、記事内で誤記ありましたらご指摘よろしくお願いします。

[公式ドキュメント](https://docs.djangoproject.com/en/4.1/topics/db/sql/)
