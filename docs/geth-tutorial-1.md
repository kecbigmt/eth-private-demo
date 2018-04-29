**Ethereumのプライベートネットワークを作って採掘ごっこしてみる（Geth on macOS）**

# About
分散アプリケーションプラットフォーム「Ethereum（イーサリアム）」について学習中の筆者が、
①GethというCUIクライアントをインストールして、
②自分のプライベートネットワークで自分のブロックチェーンを作って、
③採掘してみる
という内容です。

オープンソースのドキュメント「[Ethereum入門](https://book.ethereum-jp.net/)」のチュートリアルの一部を参考にしています。

この記事で作ったブロックチェーンのデータディレクトリはGitHubで公開しています。cloneして使えるかは試していませんが、参考までに。
https://github.com/kecbigmt/eth-private-demo

# 注意点
この記事で説明しているのは、あくまで**プライベートネットワーク上での実験**です。パブリックの取引や採掘には一切関与しないため、採掘して得たetherをパブリックのアカウントに送金することはできないし、ご自身のパブリックのether残高にアクセスすることもありません。

この記事がバイブルにしている「[Ethereum入門](https://book.ethereum-jp.net/)」ではプライベートネットワークについて以下のように言及しています。

> ここでプライベート・ネットワークは、自分自身のみのネットワークなので容易にEtherの採掘が可能ですし、安全性も高いネットワークです。そのため、Ethereumの動作を調べたり、分散型アプリケーション（Dapp）の開発作業など個人的な作業を行うには、プライベート・ネットワークを立ち上げてそこでいろいろ弄ってみると便利です。

# 用意する環境・知識
前置きはこれくらいにとどめて、ここからが本題です。

## 環境
* macOS 10.13.4
* Homebrew 1.6.2

## 必要な知識
* ブロックチェーンに対するざっくりとした理解
* Terminalの基本的な操作
* 基本的なUNIXコマンド

# 流れ
1. Gethのインストール
2. ブロックチェーンを創世する
3. Gethを起動
4. アカウントを作成する
5. 採掘してみる
6. 採掘されたブロックを確認する

## 1. Gethのインストール
まず、Terminalを起動して、HomebrewでGethをインストールします。

```bash:Terminal
$ brew tap ethereum/ethereum
$ brew install ethereum
```

インストールできたら、`geth --help`で動作確認しましょう。正しくインストールされていれば、CopyrightやVersionなどがバーっと出てきます。

```bash:Terminal
$ geth --help
NAME:
   geth - the go-ethereum command line interface

   Copyright 2013-2017 The go-ethereum Authors

USAGE:
   geth [options] command [command options] [arguments...]

VERSION:
   1.8.4-stable

COMMANDS:
   account           Manage accounts
   attach            Start an interactive JavaScript environment (connect to node)
   bug               opens a window to report a bug on the geth repo
# 以下省略
```

## 2. ブロックチェーンを創世する
適当な場所で、適当なディレクトリを作ってその配下に移動します。
このディレクトリはデータディレクトリと呼ばれ、このなかにブロックやノードに関する情報が記録されていきます。

続いて、vimでも外部エディタでもなんでもいいんですが、`myGenesis.json`というのを作ります。
直訳すれば`私の創世記.json`。なんと神々しいJSONファイルでしょうか。

```bash:Terminal
$ mkdir eth-private-demo
$ cd eth-private-demo
$ vim myGenesis.json
```

`myGenesis.json`に書き込むのは以下の内容。これが、このプライベートネットワークにおけるブロックチェーンの、**第0号**（の情報を示すファイル）になります。アダムとイヴ的なワクワク感がありますね。

```js:myGenesis.json
{
  "config": {
    "chainId": 15
  },
  "nonce": "0x0000000000000042",
  "timestamp": "0x0",
  "parentHash": "0x0000000000000000000000000000000000000000000000000000000000000000",
  "extraData": "",
  "gasLimit": "0x8000000",
  "difficulty": "0x4000",
  "mixhash": "0x0000000000000000000000000000000000000000000000000000000000000000",
  "coinbase": "0x3333333333333333333333333333333333333333",
  "alloc": {}
}
```

お次に、`geth --datadir ./ init ./myGenesis.json`でブロックチェーンを初期化します。これで`myGenesis.json`を祖とするブロックチェーンが出来上がりました。

```bash:Terminal
$ geth --datadir ./ init ./myGenesis.json
INFO [04-28|03:05:34] Maximum peer count                       ETH=25 LES=0 total=25
INFO [04-28|03:05:34] Allocated cache and file handles         database=/Users/kecy/dev/src/github.com/kecbigmt/eth-private-demo/geth/chaindata cache=16 handles=16
INFO [04-28|03:05:34] Writing custom genesis block
INFO [04-28|03:05:34] Persisted trie from memory database      nodes=0 size=0.00B time=9.5µs gcnodes=0 gcsize=0.00B gctime=0s livenodes=1 livesize=0.00B
INFO [04-28|03:05:34] Successfully wrote genesis state         database=chaindata                                                                  hash=7b2e8b…7e0432
INFO [04-28|03:05:34] Allocated cache and file handles         database=/Users/kecy/dev/src/github.com/kecbigmt/eth-private-demo/geth/lightchaindata cache=16 handles=16
INFO [04-28|03:05:34] Writing custom genesis block
INFO [04-28|03:05:34] Persisted trie from memory database      nodes=0 size=0.00B time=1.971µs gcnodes=0 gcsize=0.00B gctime=0s livenodes=1 livesize=0.00B
INFO [04-28|03:05:34] Successfully wrote genesis state         database=lightchaindata                                                                  hash=7b2e8b…7e0432
```

これによって、`--datedir`オプションで指定したディレクトリのなかに、ブロックチェーンを記録するためのディレクトリが作られます。現時点でのディレクトリ構成はおそらく以下のようになっていると思います。

```
eth-private-demo/
　├ geth/
　│　├ chaindata/
　│　│ ├ 000001.log
　│　│ ├ CURRENT
　│　│ ├ LOCK
　│　│ ├ LOG
　│　│ └ MANIFEST-000000
　│　│
　│　└ lightchaindata/
　│　  ├ 000001.log
　│　  ├ CURRENT
　│　  ├ LOCK
　│　  ├ LOG
　│　  └ MANIFEST-000000
　│
　├ keystore/
　└ myGenesis.json
```

## 3. Gethを起動
プライベートネットワーク上にブロックチェーンを準備できたので、いよいよGethを起動してプライベートネットワークに接続してみましょう。
`geth --networkid "15" --nodiscover --datadir "./" console 2>> ./geth_err.log`を実行します。何やら長ったらしいコマンドですが、以下のような意味があるようです。

- `--networkid "15"`: `myGenesis.json`で指定した`chainId`と同一の値を渡す
- `--datadir "./"`: ブロックチェーンのデータやログの出力先を指定する
- `--nodiscover`: 見ず知らずのノードに接続しないようにする
- `console`: Gethの起動と同時に、対話型のコンソールを立ち上げる。このコンソールからクライアントとしていろんな操作ができる

```bash:Terminal
$ geth --networkid "15" --nodiscover --datadir "./" console 2>> ./geth_err.log
Welcome to the Geth JavaScript console!

instance: Geth/v1.8.4-stable/darwin-amd64/go1.10.1
 modules: admin:1.0 debug:1.0 eth:1.0 miner:1.0 net:1.0 personal:1.0 rpc:1.0 txpool:1.0 web3:1.0

>
```

おお、`>`がプロンプト記号になっているJavaScriptコンソールが現れました。

できたてほやほやのブロックチェーンには、まだ**0番目**のブロックしか存在しませんが、とりあえず`eth.getBlock(0)`で中身を見てみましょう。

```js:Terminal+Geth
> eth.getBlock(0) // 0番目のブロックを確認
{
  difficulty: 16384,
  extraData: "0x",
  gasLimit: 134217728,
  gasUsed: 0,
  hash: "0x7b2e8be699df0d329cc74a99271ff7720e2875cd2c4dd0b419ec60d1fe7e0432",
  logsBloom: "0x00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000",
  miner: "0x3333333333333333333333333333333333333333",
  mixHash: "0x0000000000000000000000000000000000000000000000000000000000000000",
  nonce: "0x0000000000000042",
  number: 0,
  parentHash: "0x0000000000000000000000000000000000000000000000000000000000000000",
  receiptsRoot: "0x56e81f171bcc55a6ff8345e692c0f86e5b48e01b996cadc001622fb5e363b421",
  sha3Uncles: "0x1dcc4de8dec75d7aab85b567b6ccd41ad312451b948a7413f0a142fd40d49347",
  size: 507,
  stateRoot: "0x56e81f171bcc55a6ff8345e692c0f86e5b48e01b996cadc001622fb5e363b421",
  timestamp: 0,
  totalDifficulty: 16384,
  transactions: [],
  transactionsRoot: "0x56e81f171bcc55a6ff8345e692c0f86e5b48e01b996cadc001622fb5e363b421",
  uncles: []
}
```
さっき見かけたような感じのする内容が出てきました。そう、`myGenesis.json`で指定した内容です。16進数が一部10進数に変換されてたりと差異はありますが、まぎれもなく**0番目**です。`number: 0`ってあるし。

ほかのコマンドも実行してみましょう。`eth.accounts`とか。このノードに登録されているアカウントが表示されます。

```js:Terminal+Geth
> eth.accounts　// このノードのアカウントを全て取得
[]
```

まだ何も作っていないので当然すっからかんです。

## 4. アカウントを作成する
では、アカウントを作ってみましょう。これがないと採掘も送金もできません。
コマンドは、`personal.newAccount("パスワード")`。あとで送金するために、2つアカウントを作っておきましょう。

```js:Terminal+Geth
> personal.newAccount("foo") // 1番目のアカウントfooを作成
"0xfb6ab33c38a483e20554d9a4eabb5a6f3886fb2c"
> personal.newAccount("bar") // 2番目のアカウントbarを作成
"0x0efc6673f35b9b9489f539ea965609eb5ae587d0"
```
出てくるハッシュ文字列は、そのアカウントのアドレス。パスワードと一緒に控えておきましょう。

ちなみにパスワードを忘れると復元することはできないそうです。今回はお試しのプライベートネットワークなので適当に管理して大丈夫ですが、そうじゃない場合はしっかり記録しておいてくださいね。

この状態でもう一度`eth.accounts`を実行すると、いま作ったアカウントがきちんと反映されていることがわかります。

```js:Terminal+Geth
> eth.accounts // もう一度アカウントを全て取得
["0xfb6ab33c38a483e20554d9a4eabb5a6f3886fb2c", "0x0efc6673f35b9b9489f539ea965609eb5ae587d0"]
```

## 5. 採掘してみる
さて、いよいよみんな大好き採掘のお時間です。しかしその前に、採掘の報酬がどのアドレスに入るのかを確認しましょう。

各ノードで採掘したときに報酬を受け取れるアドレスは1つだけ。その受取先を確認するには、`eth.coinbase`を実行します。

```js:Terminal+Geth
> eth.coinbase // 採掘の報酬受取先アドレスを確認
"0xfb6ab33c38a483e20554d9a4eabb5a6f3886fb2c"
```

パスワードが`foo`のほうですね。`eth.accounts`の1番目です。どうせなので`bar`に切り替えてみましょう。`miner.setEtherbase(eth.accounts[1])`で、2番目の`bar`に切り替わります。

```js:Terminal+Geth
> miner.setEtherbase(eth.accounts[1]) // 2番目のアカウントbarを報酬受取先に指定
true
```

これで準備完了。`miner.start()`で採掘を開始します。この採掘の報酬は、`bar`のアドレスに入ってきます。
`null`が返ってくれば始まっているはず。

```js:Terminal+Geth
> miner.start() // 採掘スタート！
null
```

不安であれば`eth.mining`で採掘中かどうかを確認できます。また、いま何番目までのブロックまで採掘されたかを確認するには`eth.blockNumber`です。

```js:Terminal+Geth
> eth.mining // 採掘中かどうかを確認
true
> eth.blockNumber // ブロックチェーンのブロック数を確認
0
```

ただし、初回の採掘では、DAGと呼ばれるデータセットを生成する作業が行われるため、**実際の採掘までは10〜30分ほどかかります**。気長に待ちましょう。
このDAGは、Ethereumの採掘や検証で必要になります。ファイルサイズが1GBほどあり、macOSであれば`$HOME/.ethash/`以下に格納されます。

※DAGの詳細はWhite Paperを参照
https://github.com/ethereum/wiki/wiki/Ethash

`eth.blockNumber`を実行して1以上の数字が返ってきたら実際の採掘が始まっています。適当なところで`miner.stop()`を打ち込めば採掘を停止できます。

```js:Terminal+Geth
> eth.blockNumber // ブロックチェーンのブロック数を確認
4960
> miner.stop() // 採掘を停止
true
```
↑採掘初めて早々寝落ちして、起きてみたら結構な数のブロックが採掘されてた図。

## 6. 採掘されたブロックを確認する

採掘されたブロックを`eth.getBlock()`で確認してみましょう。

```js:Terminal+Geth
> eth.getBlock(100) // 100番目のブロックを確認
{
  difficulty: 137313,
  extraData: "0xd983010804846765746888676f312e31302e318664617277696e",
  gasLimit: 121724529,
  gasUsed: 0,
  hash: "0x7c06d551273c98746a5e4b0682b72de5fda367b36526c78d052c61b13762f7bc",
  logsBloom: "0x00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000",
  miner: "0x0efc6673f35b9b9489f539ea965609eb5ae587d0",
  mixHash: "0x59929902c691655868aa59cf53bfd2be9c40e8bf569ba767f94c675f7c5ea20d",
  nonce: "0x01ae8f238386d18c",
  number: 100,
  parentHash: "0x6744542fb19458390d875119a9a271751c4aeedabb5a4d320587148faf4acc53", // このブロックの親（99）のhash
  receiptsRoot: "0x56e81f171bcc55a6ff8345e692c0f86e5b48e01b996cadc001622fb5e363b421",
  sha3Uncles: "0x1dcc4de8dec75d7aab85b567b6ccd41ad312451b948a7413f0a142fd40d49347",
  size: 538,
  stateRoot: "0x6de62e0a0e88979e9f04b927141aa28b2cd804d27237aca5e935f246d4801603",
  timestamp: 1524858921,
  totalDifficulty: 13424369,
  transactions: [],
  transactionsRoot: "0x56e81f171bcc55a6ff8345e692c0f86e5b48e01b996cadc001622fb5e363b421",
  uncles: []
}
```

細かいフィールドの説明は省きますが、`miner`にはこのブロックを採掘したアカウント`bar`のアドレスが入っていて、`parentHash`にはこのブロックの前のブロック、つまり99番目のブロックのhash値が入っています。
99番目のブロックの`hash`を確認すれば、100番目の`parentHash`と一致していることがわかります。

```js:Terminal+Geth
> eth.getBlock(99) // 99番目のブロックを一部抜粋
{
  hash: "0x6744542fb19458390d875119a9a271751c4aeedabb5a4d320587148faf4acc53", // このブロックの子（100）のparentHashと一致
  nonce: "0x73169a1acb975eb4",
  number: 99,
  parentHash: "0x8898af4a904dea9591f5de39d412d772dd3b08ea11a17b851cd5e15516b7e685",
  ...
}
```

本当にブロックチェーンって連なってるんだなあって実感できます。

## 7. 採掘の報酬を確認する

`eth.getBalance()`で、アカウントの残高を確認できます。今回採掘の報酬受取先にしていたのは2番目のアカウント`bar`なので、`eth.accounts[1]`に報酬が入っていることがわかります。

```js:Terminal+Geth
> eth.getBalance(eth.accounts[0]) // 1番目のアカウントfooの残高を確認
0
> eth.getBalance(eth.accounts[1]) // 2番目のアカウントbarの残高を確認
2.481e+22
```
ただし、これはether単位ではなく、weiという単位での数字。1 ether = 1,000,000,000,000,000,000 weiなので、とんでもない数字になっています。
`web3.fromWei()`を使って、weiを他の単位に換算できます。ここではetherに換算してみましょう。

```js:Terminal+Geth
> web3.fromWei(eth.getBalance(eth.accounts[0]),"ether") // 1番目のアカウントfooの残高をether単位で確認
0
> web3.fromWei(eth.getBalance(eth.accounts[1]),"ether") // 2番目のアカウントbarの残高をether単位で確認
24810
```

# まとめ

ここまでで、以下のことを見てきました。

- EthereumのクライアントGethをインストール
- Gethでプライベートネットワークをに接続してブロックチェーンを新しく作る
- アカウントを作成して、採掘して、その結果を確認してみる

Gethという優秀なクライアントソフトが用意されているので、Ethereumに参加するだけであれば、プログラミング知識はほとんど必要ないことがわかります。

プログラミングやブロックチェーンの初心者には、とりあえずGethを使ってしばらく遊んでみることをおすすめします。プライベートネットワーク上で遊んでいる限りは何をしても実害を被ることはないはずです。

私もブロックチェーン学習中の身なので、これからいろいろ読んでみていじってみて、どんどん投稿していきます。

次回の記事はこちら
[Ethereumのプライベートネットワークで送金してみる（Geth on macOS）](https://qiita.com/kecy/items/a4a44843a4341bf8f189)

# References
- Ethereum入門: https://book.ethereum-jp.net/
- ethereum/go-ethereum
  - Repository: https://github.com/ethereum/go-ethereum
  - Installation Instructions for Mac: https://github.com/ethereum/go-ethereum/wiki/Installation-Instructions-for-Mac
- ethereum/wiki
  - Repository: https://github.com/ethereum/wiki
  - Ethash: https://github.com/ethereum/wiki/wiki/Ethash
