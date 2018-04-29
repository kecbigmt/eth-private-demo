
# 手順
## 1. solcをインストール

```bash
$ brew update
$ brew upgrade
$ brew tap ethereum/ethereum
$ brew install solidity
$ solc --version
solc, the solidity compiler commandline interface
Version: 0.4.23+commit.124ca40d.Darwin.appleclang
$ which solc
/usr/local/bin/solc
```

## 2. コントラクトコードを作成

```
pragma solidity ^0.4.0;
contract SingleNumRegister {
  uint storedData;
  function set(uint x) public{
    storedData = x;
  }
  function get() public constant returns (uint retVal){
    return storedData;
  }
}
```

## 3. コントラクトコードをコンパイル
```bash
$ solc --abi --bin SingleNumRegister.sol

======= SingleNumRegister.sol:SingleNumRegister =======
Binary:
608060405234801561001057600080fd5b5060df8061001f6000396000f3006080604052600436106049576000357c0100000000000000000000000000000000000000000000000000000000900463ffffffff16806360fe47b114604e5780636d4ce63c146078575b600080fd5b348015605957600080fd5b5060766004803603810190808035906020019092919050505060a0565b005b348015608357600080fd5b50608a60aa565b6040518082815260200191505060405180910390f35b8060008190555050565b600080549050905600a165627a7a72305820b1953023a4f5bcf69d74f8d8ca317f6761bfe3fbcf025db5359423a65f0fd6370029
Contract JSON ABI
[{"constant":false,"inputs":[{"name":"x","type":"uint256"}],"name":"set","outputs":[],"payable":false,"stateMutability":"nonpayable","type":"function"},{"constant":true,"inputs":[],"name":"get","outputs":[{"name":"retVal","type":"uint256"}],"payable":false,"stateMutability":"view","type":"function"}]
```

## 4.Contractアカウントを作成
```js:Terminal+Geth
> personal.unlockAccount(eth.accounts[0])
Unlock account 0xfb6ab33c38a483e20554d9a4eabb5a6f3886fb2c
Passphrase:
true
> var bin = "0x608060405234801561001057600080fd5b5060df8061001f6000396000f3006080604052600436106049576000357c0100000000000000000000000000000000000000000000000000000000900463ffffffff16806360fe47b114604e5780636d4ce63c146078575b600080fd5b348015605957600080fd5b5060766004803603810190808035906020019092919050505060a0565b005b348015608357600080fd5b50608a60aa565b6040518082815260200191505060405180910390f35b8060008190555050565b600080549050905600a165627a7a72305820b1953023a4f5bcf69d74f8d8ca317f6761bfe3fbcf025db5359423a65f0fd6370029" // コンパイルしたBinaryの先頭に0xを付ける
> var abi = [{"constant":false,"inputs":[{"name":"x","type":"uint256"}],"name":"set","outputs":[],"payable":false,"stateMutability":"nonpayable","type":"function"},{"constant":true,"inputs":[],"name":"get","outputs":[{"name":"retVal","type":"uint256"}],"payable":false,"stateMutability":"view","type":"function"}]
> var contract = eth.contract(abi)
> var myContract = contract.new({ from: eth.accounts[0], data: bin})
```

`myContract`の中身を見てみると、`address`が`undefined`になっていて、まだ未採掘であることがわかります。

```js:Terminal+Geth
> myContract
{
  abi: [{
      constant: false,
      inputs: [{...}],
      name: "set",
      outputs: [],
      payable: false,
      stateMutability: "nonpayable",
      type: "function"
  }, {
      constant: true,
      inputs: [],
      name: "get",
      outputs: [{...}],
      payable: false,
      stateMutability: "view",
      type: "function"
  }],
  address: undefined,
  transactionHash: "0x94156d6c46c853b545876b477ce87fbac8a2885e8b87a0fd7c04e68880ba5125"
}
```



`miner.start()`で採掘者を動かして、新しく作ったContractを登録したブロックの採掘を待ちます。

`myContract`の中身を確認したときに`address`にちゃんとアドレスが入っていたら採掘が終わっているので、`miner.stop()`で採掘を止めます。

```js:Terminal+Geth
> miner.start()
> myContract
{
  abi: [{
      constant: false,
      inputs: [{...}],
      name: "set",
      outputs: [],
      payable: false,
      stateMutability: "nonpayable",
      type: "function"
  }, {
      constant: true,
      inputs: [],
      name: "get",
      outputs: [{...}],
      payable: false,
      stateMutability: "view",
      type: "function"
  }],
  address: "0x9f5df97fe660bbdb061d4ce9071942fd81d5d464",
  transactionHash: "0x94156d6c46c853b545876b477ce87fbac8a2885e8b87a0fd7c04e68880ba5125",
  allEvents: function(),
  get: function(),
  set: function()
}
> miner.stop()
```

これで、作成したスマートコントラクトを実行するContractアカウントが`0x9f5df97fe660bbdb061d4ce9071942fd81d5d464`というアドレスで登録されました。

## 5.Contractアカウントにアクセスする

```js:Terminal+Geth
> myContract.address
"0x9f5df97fe660bbdb061d4ce9071942fd81d5d464"
> myContract.abi
[{
    constant: false,
    inputs: [{
        name: "x",
        type: "uint256"
    }],
    name: "set",
    outputs: [],
    payable: false,
    stateMutability: "nonpayable",
    type: "function"
}, {
    constant: true,
    inputs: [],
    name: "get",
    outputs: [{
        name: "retVal",
        type: "uint256"
    }],
    payable: false,
    stateMutability: "view",
    type: "function"
}]
```


```js:Terminal+Geth
> personal.unlockAccount(eth.accounts[0])
Unlock account 0xfb6ab33c38a483e20554d9a4eabb5a6f3886fb2c
Passphrase:
true
> var cnt = eth.contract(myContract.abi).at(myContract.address);
> var txHash =cnt.set.sendTransaction(6, {from: eth.accounts[0] })
"0x6810ffb6d3fc5d84bf919c9e24244ad8d2e78cf056095441f0cc8b68068c0aa2"
> eth.getTransaction(txHash)
{
  blockHash: "0x0000000000000000000000000000000000000000000000000000000000000000",
  blockNumber: null,
  from: "0xfb6ab33c38a483e20554d9a4eabb5a6f3886fb2c",
  gas: 90000,
  gasPrice: 18000000000,
  hash: "0x6810ffb6d3fc5d84bf919c9e24244ad8d2e78cf056095441f0cc8b68068c0aa2",
  input: "0x60fe47b10000000000000000000000000000000000000000000000000000000000000006",
  nonce: 3,
  r: "0x1ff854141a721b9f977ed7224c5f19aea1ba86d14844779b59b63fa4fa6bad65",
  s: "0x437ab144dabbd340a081c2f25e84d8406e0a38722c2995f593d774787ee8d7b",
  to: "0x9f5df97fe660bbdb061d4ce9071942fd81d5d464",
  transactionIndex: 0,
  v: "0x1c",
  value: 0
}
```

```js:Terminal+Geth
> miner.start()
> eth.getTransaction(txHash)
{
  blockHash: "0x042131c0e2ff415944452c97eaf86867c837235ebf4f985677e36d8a1131a71f",
  blockNumber: 5104,
  from: "0xfb6ab33c38a483e20554d9a4eabb5a6f3886fb2c",
  gas: 90000,
  gasPrice: 18000000000,
  hash: "0x6810ffb6d3fc5d84bf919c9e24244ad8d2e78cf056095441f0cc8b68068c0aa2",
  input: "0x60fe47b10000000000000000000000000000000000000000000000000000000000000006",
  nonce: 3,
  r: "0x1ff854141a721b9f977ed7224c5f19aea1ba86d14844779b59b63fa4fa6bad65",
  s: "0x437ab144dabbd340a081c2f25e84d8406e0a38722c2995f593d774787ee8d7b",
  to: "0x9f5df97fe660bbdb061d4ce9071942fd81d5d464",
  transactionIndex: 0,
  v: "0x1c",
  value: 0
}
> miner.stop()
```

## 6.ブロックチェーン上のコントラクトに登録された値を読み出す
```js:Terminal+Geth
> cnt.get()
6
```
