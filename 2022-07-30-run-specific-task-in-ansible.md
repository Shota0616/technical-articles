ansibleで特定のタスクのみを実行したいときの方法

## ansibleで特定のタスクのみを実行する方法

ansibleを使用して、環境構築をしているとターゲットに対してこのタスクだけ実行したいというときありますよね？そんなときの方法を紹介します。

結論から書くと、`--start-at-task`オプションと`--step`オプションのあわせ技で行う。

まず`--start-at-task`これは、タスクの開始位置を指定できる。
```
# ex taskからタスクを開始する
ansible-playbook playbook.yml --start-at-task="ex task"
```

そして`--step`これは、タスクを一つ一つ、対話形式で実行できる。
```
ansible-playbook playbook.yml --step
Perform task: first task(y/n/c):
```
一番最初のタスクが「first task」だったらこんな感じになる。そして、「y」はタスクが実行、「n」はタスクをスキップ、「c」は残りのタスクすべて実行する。

これを組み合わせることによって
```
ansible-playbook playbook.yml --step --start-at-task="ex task"
```
上のオプションをつければ「ex task」から実行し「--step」でタスクを一つ一つ対話型で実行できるので、ex taskを実行したあとに`CTRL＋C`で強制終了すれば特定のタスクのみを実行できたことになる。
