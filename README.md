# 『ゼロから作る Deep Learning 2』


## 1章 ニューラルネットワークの復習

前作『ゼロから作る Deep Learning』の内容の復習。

前作の内容をサマライズした内容は省略するとして、差分や、要復習だった部分を書く。


### 1.3.4 計算グラフ

* 分岐ノード
    * 逆伝播のときは、上流からの勾配の和になる
* Repeatノード
    * 分岐ノードの一般化。逆伝播のときは上流からの勾配の総和を取る
* Sumノード
    * Repeatノードと順伝播と逆伝播が逆になっているノード
* MatMulノード
    * 結論は分かったが、導出過程の説明が理解が浅いように思うので、再度確認が必要

### 1.3.5.1 Sigmoid 関数の微分の導出過程

* 詳細は「付録 A」参照
* 重みの更新のステップ
    * 訓練データの中からランダムに複数のデータを選び出す
    * 誤差逆伝播方式により、各重みパラメータに関する損失関数の勾配を求める
    * 勾配を使って重みパラメータを更新する
    * 上記3ステップを必要な回数繰り返す

### 1.4.3 学習用のソースコード

一般的な学習の構成

* ハイパーパラメータの設定
* データの読み込み、モデルとオプティマイザの生成
* データのシャッフル
* 勾配を求め、パラメータを更新
* 定期的に学習経過を出力


### 1.4.4 Trainer クラス

* `1.4.3 学習用のソースコード` をクラスにまとめたもの
* `ch01.train_custom_loop` を `common.trainer` にまとめ
* `fit()` を学習用のインターフェースと持たす


### 1.5.1 ビット精度

経験的に精度を落とすことなく、以下の事項が知られている

* (64ビットではなく)32ビット浮動小数の利用で充分
* 推論に限って言えば16ビット浮動小数で充分
* しかし、H/W側が提供しているのが32ビット演算なので、基本的に32ビットを使う
* 保存のときだけ容量削減のため16ビットで実施する


### 1.5.2 GPU (CuPy)

* numpy 互換インターフェースのGPU利用版
* 切り替えについては `common.config`、`common.np`、`common.layers`あたりを参照。
* 4章からのコードで使えるとのことだが、対応GPUを持っていないので試せないと思う。


### 1.6 章のまとめ

ニューラルネットワークの実装では以下を作るとよい

* 構成要素の `Layer` クラス (I/Fとして `forward` と `backward` を持つ)
* 学習のための `Trainer` クラス (I/Fとして `fit` を持つ)
    

## 2章 自然言語と単語の分散表現

### 2.1 自然言語処理とは

本書で扱うのは

* シソーラスによる手法 (2.2 と 付録B)
* カウントベースの方法 (2.3)
* 推論ベースの手法 (word2vec) (次章)

シソーラスとカウントベースはやったことがあるので、概ねパスするかも。


### 2.2 シソーラスによる手法

コードを伴う部分は WordNet のコーパスを使って付録Bでやるとのことだが、パス


### 2.3 カウントベースの方法

一旦英文が対象


#### 2.3.3 分布仮説 (distributional hypothesis)

* 「単語の意味は、周囲の単語によって形成される」という考え方
* (例) "drink" の近くには飲み物が来やすい
* 「コンテキスト」：注目する単語に対して、その周囲に存在する単語を指す
* 「ウィンドウサイズ」：注目する単語を中心にして、周囲の語をどの程度見るか


#### 2.3.4 共起行列 (cooccurrence matrix)

* ウィンドウサイズ内で出現した単語を行列にしたもの
* コーパスの単語が縦横に入り、値は出現回数になる


#### 2.3.5 ベクトル間の類似度

* コサイン類似度
* ゼロ除算を避けるためにごく小さい値(1e-8)を分母に足す
* 上記や浮動小数演算により、誤差が生じるので UnitTest のさいには `assertAlmostEquel` を使う


#### 2.3.6 類似単語のランキング表示

* コサイン類似度の単語ランキングを作成する


#### 2.4.2 次元削減

* 次元削減：ベクトルの必要な情報を残したまま、データ量を削減すること
* 疎なベクトル：ベクトル中の多くの要素が0であるベクトルのこと
* 特異値分解(SVD)：任意の行列を3つの行列の積へと変換


#### 2.4.5 PTBデータセットでの評価

PTBデータセットで共起行列を作って正の相互情報量を出し、SVDで単語ベクトルを次元圧縮する。


### 2.6 まとめ

カウントベースの単語の類似度を測るための手法について学んだ。

次の章で簡易的な `word2vec` を実装する。


## 3章 word2vec

#### 3.1.1 カウントベース手法の問題点

* 計算量が多いこと(例：SVDは n x n 行列に対して計算量がO(n^3)かかる)
* コーパス全体のデータを使わないといけない


#### 3.1.2 推論ベースの手法の概要

* コンテキストを入力。出力を各単語の出現確率とするようなモデルを作成する
* コンテキストをベクトル表現できれば(one-hot表現が紹介されている)計算可能になりそう


### 3.2 シンプルな word2vec

word2vecには

* continuous bag-of-words(CBOW)
* skip-gram

の2モデルがあり、まずはCBOWの方を先に説明する(skip-gramは3.5.2)

#### 3.2.1 CBOWモデルの推論処理

* 入力層が複数
* 中間層が1層(全結合による変換処理の値が平均されたもの)
* 出力層(スコア算出層で、Softmaxをまだかけていない)

ものから構成される。


#### 3.2.3 word2vecの重みと分散表現

入力と出力側の両方に重みがあるがどちらを使うか、という問題があるが、word2vec(特にskip-gramモデル)の場合は入力側の重みだけを使う。


### 3.3 学習データの準備

#### 3.3.1 コンテキストとターゲット

* word2vecで用いるニューラルネットワークの
    * 入力：コンテキスト。正解を囲む単語
    * 正解ラベル：コンテキストに囲まれた中央の単語

とする


### 3.4 CBOW モデルの実装

CBOWモデルの実装。 

optimizer として特段説明なしに Adam が出てくるが、構造自体はこれまでのニューラルネットの実装と同じ。


## 環境セットアップ

### 要件

* インタプリタ
    * Python 3系 (venvで環境切り出し)
* ライブラリ
    * NumPy
    * Matplotlib
    * CuPy (オプション。非NVIDIA GPUなのでインストールできない)

    
### 構築手順

### インタプリタとvenvの設定

PyCharm を使うのでその設定

* 本体メニューの「PyCharm」
* 「Preferences」
* 「Project Interpreter」
* 「Add...」
* 「New environment」

で Base Interpreter で Python 3.x 系を選択してプロジェクトルートに `venv` ディレクトリに virtualenv の設定を配置。

### ライブラリ

* インタプリタの設定にモジュールのインストールリストが出ているので
* `numpy` と `matplotlib` を選択してインストール

環境が非NVIDIA GPUなのでCuPyはインストールに失敗するのでインストールしない。


### ライブラリのバージョンの指定

`pip freeze` の出力を保存。

```
$ pip freeze > requirements.txt
```




## 参考

* 本紙
    * [oreilly-japan/deep-learning-from-scratch-2](https://github.com/oreilly-japan/deep-learning-from-scratch-2) 
