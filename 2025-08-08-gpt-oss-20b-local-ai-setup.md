ローカルPCで動くAI「gpt-oss-20b」とは？インストールから実行まで完全ガイド

:::note info
この記事はAIの力を使用せず、手動で書いています。
:::

# 概要

2025年8月5日にOpenAI社が、低コストで強力な実世界パフォーマンスを実現する2つの最先端のオープンウェイト言語モデルの<font color="red">**gpt-oss-120b**</font>と<font color="red">**gpt-oss-20b**</font>をリリースしました。

https://openai.com/ja-JP/index/introducing-gpt-oss/

> gpt-oss-120b モデルは、単一の80GB GPU で効率的に稼働しながら

とあるので、gpt-oss-120bは最強PCを持っている方でないと試せなそうですが、、、

> gpt-oss-20b モデルは、一般的なベンチマークで OpenAI o3‑mini と同様の結果を出し、わずか16 GBのメモリを搭載したエッジデバイスで実行でき

gpt-oss-20bでは、当方の所持しているM4 Macでも動作させられそうです！！しかも、o3-miniと同等とのこと！！

ということで、実際にインストールから実行まで最速で試してみます。

# 実行環境

* CPU : Apple M4
* メモリ : 16 GB
* OS : MacOS15.5

# 手順


gpt-oss-20bを使用するために、LM Studioをインストールします。
これは、ローカルでLLMを簡単に実行・対話できるGUIツールで、gpt-ossのようなOSSのモデルを誰でも試せることを目的としたアプリケーションです。

https://lmstudio.ai/


1\. LM Studioのインストーラーをダウンロードする

![スクリーンショット 2025-08-08 1.06.44.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/30db31e2-8fb1-439e-a504-b3fedb6746ad.png)

2\. ダウンロードした`LM-Studio-0.3.22-2-arm64.dmg`を実行



3\. LM StudioをApplicationsに移動してインストール

![スクリーンショット 2025-08-08 1.10.11.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/5d52a384-194e-48e3-b460-535ed45808f2.png)

4\. 起動すると初期設定が必要なので、レベルは`User`とします。

![スクリーンショット 2025-08-08 1.13.31.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/b3cf34e2-3cb8-4bbf-ac1a-79c8b016e504.png)

5\. 続いて、`gpt-oss-20b`のモデルをダウンロードしていきます。（12.11GBもある。。）

![スクリーンショット 2025-08-08 1.15.10.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/9ceb3aa0-ddb8-4871-84eb-7935caf3750a.png)

6\. ダンロードが完了したらチャットを開始できます！

![スクリーンショット 2025-08-08 1.20.00.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/0a32a348-46f4-4dd4-b564-b5049e373b22.png)

7\. Chatの画面にいくと、画面上部にどのモデルを読み込むか選択することができるので、ここで`gpt-oss-20b`を選択してください。

![スクリーンショット 2025-08-08 1.29.57.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/db0e46f3-32fe-4177-9d4a-7d50a5feace2.png)

これで、ローカルでLLMが使用できるようになりました！！
ためしに、以下のプロンプトでgpt-oss-20bのことについて聞いてみました。
```
gpt-oss-20bについて簡潔にまとめて
```

↓ 以下出力です！


**GPT‑Oss‑20B（OpenAI 社が公開した 20 億パラメータの言語モデル）**

| 項目 | 内容 |
|------|------|
| **概要** | 2023年頃にリリースされた、Transformer ベースの大規模言語モデル。GPT‑4 の前段階で、OpenAI が公開した「OSS（Open Source Software）」バージョン。 |
| **パラメータ数** | 約 20 億（2.0B）パラメータ。|
| **アーキテクチャ** | GPT‑3 と同様の自己回帰型 Transformer。LayerNorm、残差接続、位置エンコーディングは標準的。 |
| **学習データ** | 2021 年までに公開されているインターネット上のテキスト（WebText、Common Crawl 等）。 |
| **性能** | ベンチマークでは GPT‑3 の「davinci」レベルと同等程度で、自然言語生成・推論タスクで高い品質を示す。|
| **用途** | チャットボット、文章要約、コード生成、質問応答など、多様な NLP タスクに利用可能。 |
| **ライセンス/アクセス** | GitHub 上の OSS 版として公開されており、MIT ライセンスで自由使用・改変が可能。ただし、商用利用には別途契約が必要になる場合も。|
| **主要な違い（GPT‑4 との比較）** | • パラメータ数は大幅に少ない<br>• 推論速度は速い（リソース効率良好）<br>• 一部高度な推論・マルチモーダル機能は欠如 |

簡単に言うと、GPT‑Oss‑20B は「中規模だが実用的な大規模言語モデル」で、オープンソースとして手軽に導入できる点が特徴です。

</details>



# トラブルシューティング

メモリに余裕のない環境だと以下のエラーが出てしまうことがあるので、解消方法も書いておきます。

```
Model loading aborted due to insufficient system resources. Overloading the system will likely cause it to freeze. If you believe this is a mistake, you can try to change the model loading guardrails in the settings.
```
![スクリーンショット 2025-08-08 1.24.01.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/69f11eec-8af3-483d-952e-4f1a13072864.png)

これは、`Model Loading guardrails`によってメモリ制限に引っかかっているので、マシンのメモリを解放してやるか`Model Loading guardrails`の設定を変更してやります。
`Model Loading guardrails`の設定をOFFにすれば読み込みには引っ掛からなくなると思いますが、PCがフリーズする可能性もあるので自己責任でお願いします。

ちなみにUserの状態だとSettingsを開いてもguardrailsの設定が表示されないので、`Power User`に変更後にSettingsを確認お願いします。

![スクリーンショット 2025-08-08 1.47.23.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/324442c9-c99a-41a0-afd2-fd9b5a147b97.png)


