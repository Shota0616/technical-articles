GitHubで特定コミット,ブランチの差分を確認する方法

## 概要

GitHubのGUI上で、特定コミット同士の差分を確認する方法を備忘録として残しておきます。特定ブランチ同士の差分の確認方法も合わせて記載しておきます。

## 結論

特定コミット同士の差分確認方法
```
https://github.com/[ユーザー名]/[リポジトリ名]/compare/[コミットID①]...[コミットID②]
```

特定ブランチ同士の差分確認方法
```
https://github.com/[ユーザー名]/[リポジトリ名]/compare/[ブランチ名①]...[ブランチ名②]
```

## 特定コミット同士の差分確認方法の詳細


例として、私のgithubのリポジトリで確認してみます。コミットIDが`99c7f45`と`f7da013`の差分が確認できるようになっていると思います。
https://github.com/Shota0616/docker_django/compare/99c7f45...f7da013

コミットIDは以下の赤枠の部分を押下すると
![スクリーンショット 2023-05-20 12.09.43.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/ab32926d-3947-17c1-8374-dde7cf527469.png)
下の画面に遷移して赤枠の中がコミットIDになります。
![スクリーンショット 2023-05-20 12.09.55.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/3041a4fd-8e62-dc0c-12a7-42bea4fd0a68.png)


## 特定ブランチ同士の差分確認方法の詳細

こちらも例として私のリポジトリで確認してみます。ちょっと下の例だと差分無いですが、差分があるブランチ同士だと差分が出るはずです。
https://github.com/Shota0616/docker_django/compare/main...feature

