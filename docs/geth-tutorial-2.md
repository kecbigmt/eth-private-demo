**Ethereumのプライベートネットワークで送金してみる（Geth on macOS）**

# About
## 前回までのあらすじ
[前回](https://qiita.com/kecy/items/de7337c6fcb6505cc963)は以下のことをしました。
①GethというCUIクライアントをインストールして、
②自分のプライベートネットワークで自分のブロックチェーンを作って、
③採掘してみる

※前回の記事はこちら
[Ethereumのプライベートネットワークを作って採掘ごっこしてみる（Geth on macOS）](https://qiita.com/kecy/items/de7337c6fcb6505cc963)


## 今回やること
今回は、
①バックグラウンドで採掘しながら、
②etherをほかのアカウントに送金して、
③そのトランザクションの中身をざっと眺めてみる
という内容です。

今回も引き続き、オープンソースのドキュメント「[Ethereum入門](https://book.ethereum-jp.net/)」のチュートリアルの一部をがっつり参考にしています。

# 注意点
この記事で説明しているのは、あくまで**プライベートネットワーク上での実験**です。パブリックの取引や採掘には一切関与しないため、採掘して得たetherをパブリックのアカウントに送金することはできないし、ご自身のパブリックのether残高にアクセスすることもありません。

# 環境・知識
## 筆者の環境
* macOS 10.13.4
* Homebrew 1.6.2

## 必要な知識
* ブロックチェーンに対するざっくりとした理解
* Terminalの基本的な操作
* 基本的なUNIXコマンド
* JavaScriptの簡単な知識（変数代入とか、オブジェクトとか）
* [前回](https://qiita.com/kecy/items/de7337c6fcb6505cc963)までの知識

# 手順
1. Gethの起動と採掘の実行
2. 残高確認とアカウントのアンロック
3. 送金の実行
4. トランザクションの確認

## 1. Gethの起動と採掘の実行
ここは[前回](https://qiita.com/kecy/items/de7337c6fcb6505cc963)のおさらいです。
まずは、TerminalでGethを起動します。

```bash:Terminal
$ cd eth-private-demo # 前回作ったデータディレクトリに移動
$ geth --networkid "15" --nodiscover --datadir "./" console 2>> ./geth_err.log
Welcome to the Geth JavaScript console!

instance: Geth/v1.8.4-stable/darwin-amd64/go1.10.1
coinbase: 0xfb6ab33c38a483e20554d9a4eabb5a6f3886fb2c
at block: 4962 (Sat, 28 Apr 2018 10:07:00 JST)
 datadir: /Users/kecy/dev/src/github.com/kecbigmt/eth-private-demo
 modules: admin:1.0 debug:1.0 eth:1.0 miner:1.0 net:1.0 personal:1.0 rpc:1.0 txpool:1.0 web3:1.0

>
```

次に、採掘を実行します。

今回の趣旨はetherの送金ですが、ブロックチェーンでは送金のトランザクションを採掘者に記録・検証してもらわなければいけません。
自分のプライベートネットワークで発生したトランザクションを記録・検証してくれる採掘者は、もちろん自分で用意しておく必要があります。

前回同様、2番目のアカウントを採掘報酬の受取先に指定してから、`miner.start()`を実行します。

```js:Terminal+Geth
> miner.setEtherbase(eth.accounts[1]) // 2番目のアカウントを採掘報酬の受取先に指定
true
> miner.start() // 採掘の開始
null
> eth.mining // 採掘中かどうか確認
true
```

これで採掘が開始されました。これをバックグラウンドで走らせつつ、送金処理をやってみます。

## 2. 残高確認とアカウントのアンロック

まずは、アカウントの残高を確認しておきます。

```js:Terminal+Geth
> web3.fromWei(eth.getBalance(eth.accounts[0]), "ether") // 1番目のアカウント`foo`の残高を確認
0.001242
> web3.fromWei(eth.getBalance(eth.accounts[1]), "ether") // 2番目のアカウント`bar`の残高を確認
25489.998758
```

はい。2番目のアカウント`bar`にはたんまりetherが入ってて、1番目のアカウント`foo`には少しだけ入っています。
（`foo`に少しだけ入っているのは、このあいだうっかり`foo`を報酬受取先にして採掘をしてしまったからです。本当は0にしたかった）

現在のレートで換算すると、19億円近くの資産を抱える`bar`と、92円の小銭しか持たない`foo`。哀れに感じた`bar`は、100 etherを`foo`に恵んであげることにしました。
※ETH/JPY 74,619（2018/4/28現在, bitFlyer）

まずは、2番目のアカウント`bar`のロックを解除します。[前回](https://qiita.com/kecy/items/de7337c6fcb6505cc963)のアカウント作成時に指定したパスワードが必要になります。

```js:Terminal+Geth
> personal.unlockAccount(eth.accounts[1]) // ロックを解除（パスワードを要求されるので入力）
Unlock account 0x0efc6673f35b9b9489f539ea965609eb5ae587d0
Passphrase:
true
```

## 3. 送金の実行
※ほんの少しだけJavaScriptの知識がでてきます

`eth.sendTransaction()`で送金を実行します。引数として、送金元（`from`）・送金先（`to`）・送金額（`value`）をオブジェクトに入れて渡す必要があります。
また、この関数の返り値はトランザクションIDになるので、それを`txHash`という変数に代入しておきます。
矢継ぎ早に`eth.getTransaction(txHash)`を実行して、未完了のトランザクションの情報を確認してみましょう。
（のんびりしているとバックグラウンドで実行されている採掘プロセスに拾われて、送金処理が完了してしまいます）

```js:Terminal+Geth
> txHash = eth.sendTransaction({from: eth.accounts[1], to: eth.accounts[0], value: web3.toWei(100, "ether")})
"0x3ff59587aca8b933dc73e5971e8d9203284bf79af90d75e602897d09eb703d6d"
> eth.getTransaction(txHash)
{
  blockHash: "0x0000000000000000000000000000000000000000000000000000000000000000",
  blockNumber: null,
  from: "0x0efc6673f35b9b9489f539ea965609eb5ae587d0",
  gas: 90000,
  gasPrice: 18000000000,
  hash: "0x3ff59587aca8b933dc73e5971e8d9203284bf79af90d75e602897d09eb703d6d",
  input: "0x",
  nonce: 0,
  r: "0x17aeffde51d3786af2773397e4476ecd6358ef0eb592e78c0a3e462f74d0d6b7",
  s: "0x7ed0afd16fd3cbc75eb5ac9b824f59d1bd948e72a4b94714dd9ae09b5ae6a0ee",
  to: "0xfb6ab33c38a483e20554d9a4eabb5a6f3886fb2c",
  transactionIndex: 0,
  v: "0x1b",
  value: 100000000000000000000
}
```

`from`、`to`にはさっき指定した送金元のアドレスと送金先のアドレスが入っていて、`value`にはWei換算された送金額が記録されていることがわかります。

`blockNumber`が`null`、`blockHash`が`0x00`（16進数のゼロ）になっているなど、まだ採掘が完了していないこともわかります。

## 4.トランザクションの確認

バックグラウンドで採掘プロセスが走ったままであれば、10数秒以内に採掘が完了するはずです。
もう一度`eth.getTransaction(txHash)`でトランザクションの情報を確認してみましょう。

```js:Terminal+Geth
> eth.getTransaction(txHash)

{
  blockHash: "0x5ff998808329c3a9514027bf37c7604b37c30de8da652f3cc35fb753e46fe37e",
  blockNumber: 5099,
  from: "0x0efc6673f35b9b9489f539ea965609eb5ae587d0",
  gas: 90000,
  gasPrice: 18000000000,
  hash: "0x3ff59587aca8b933dc73e5971e8d9203284bf79af90d75e602897d09eb703d6d",
  input: "0x",
  nonce: 0,
  r: "0x17aeffde51d3786af2773397e4476ecd6358ef0eb592e78c0a3e462f74d0d6b7",
  s: "0x7ed0afd16fd3cbc75eb5ac9b824f59d1bd948e72a4b94714dd9ae09b5ae6a0ee",
  to: "0xfb6ab33c38a483e20554d9a4eabb5a6f3886fb2c",
  transactionIndex: 0,
  v: "0x1b",
  value: 100000000000000000000
}
```

`blockNumber`や`blockHash`がしっかり埋まっているので、採掘が完了していることがわかります。

`web3.fromWei(eth.getBalance(アドレス), "ether")`でether換算の残高も確認してみましょう。

```js:Terminal+Geth
> web3.fromWei(eth.getBalance(eth.accounts[0]), "ether") // 1番目のアカウント`foo`の残高を確認
100.001242
> web3.fromWei(eth.getBalance(eth.accounts[1]), "ether") // 2番目のアカウント`bar`の残高を確認
25399.998758
```

めでたく、`foo`のアカウントに100 etherが送金されていることがわかります。これで`foo`は現在のレートで74,619,000円分のetherを恵んでもらうことができました。
※ETH/JPY 74,619（2018/4/28現在, bitFlyer）

`bar`のアカウントからは90 etherだけ減っています。ぴったり100 etherではないのは、

- 送金時に手数料を上乗せして残高から差し引かれた
- 採掘報酬を受け取った（このノードの採掘報酬の受取先は`bar`のアドレスに指定されているため）

という理由が挙げられます。

送金手数料の計算や、トランザクション情報の詳細については次回詳しく見ていきたいなと思います。

# References
- Ethereum入門: https://book.ethereum-jp.net/
