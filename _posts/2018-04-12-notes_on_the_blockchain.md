---
layout: post
title: Notes on Blockchain
date: 2018-04-12
categories: linux blockchain
---

# Notes on the BlockChain

Lets split some notes about learning to work with blockchain.

## My First Ethereum Network

Following the example from [hackernoon](https://hackernoon.com/setup-your-own-private-proof-of-authority-ethereum-network-with-geth-9a0a3750cda8)

Here's an extract of what's important for me in the process.

### Create Nodes

```
jaci@geth:~/devnet$ mkdir node1 node2

jaci@geth:~/devnet$ geth --datadir node1/ account new
INFO [03-14|15:19:48] Maximum peer count                       ETH=25 LES=0 total=25
Your new account is locked with a password. Please give a password. Do not forget this password.
Passphrase:
Repeat passphrase:
Address: {f4ad40a428b02f7a45ce6fa9fc3f87c4cab20f0a}

jaci@geth:~/devnet$ geth --datadir node2/ account new
INFO [03-14|15:22:06] Maximum peer count                       ETH=25 LES=0 total=25
Your new account is locked with a password. Please give a password. Do not forget this password.
Passphrase:
Repeat passphrase:
Address: {cdd45d92a9b151b0eea54e70914ce5ea2dfdc249}
```
##### Note: save the passwords on ~/devnet/node{1,2}/password.txt, remember this is a testnet!

### Create Genesis file (see [hackernoon guide](https://hackernoon.com/setup-your-own-private-proof-of-authority-ethereum-network-with-geth-9a0a3750cda8))

You should have now a few files like the following

```
devnet$ tree -L 2
.
├── accounts.txt
├── boot.key
├── genesis.json
├── node1
│   ├── geth
│   ├── keystore
│   └── password.txt
└── node2
    ├── geth
    ├── keystore
    └── password.txt
```

### Initialize your nodes

```
devnet$ geth --datadir node1/ init genesis.json
devnet$ geth --datadir node2/ init genesis.json
```

### Start BootNode

Once you're done with genesis.json and initialized your nodes, you can start the BootNode 
 
``` 
jaci@geth:~/devnet$ bootnode -nodekey boot.key -verbosity 9 -addr :30310
INFO [03-14|15:27:15] UDP listener up                          self=enode://42735281a2058b0eedcebf3fa579253e654dbdf945b89ac8a44fc4a3386bd14bca3f0cb19947a97f8a4303b8851b23189b297166c369ae45a7f90cb013c52c5e@[::]:30310
```

### Start Nodes

##### Node 1

```
geth --datadir node1/ --syncmode 'full' --port 30311 --rpc --rpcaddr 'localhost' --rpcport 8501 --rpcapi 'personal,db,eth,net,web3,txpool,miner' --bootnodes 'enode://42735281a2058b0eedcebf3fa579253e654dbdf945b89ac8a44fc4a3386bd14bca3f0cb19947a97f8a4303b8851b23189b297166c369ae45a7f90cb013c52c5e@127.0.0.1:30310' --networkid 1515 --gasprice '1' -unlock '0xf4ad40a428b02f7a45ce6fa9fc3f87c4cab20f0a' --password node1/password.txt --mine
```

##### Node2

```
geth --datadir node2/ --syncmode 'full' --port 30312 --rpc --rpcaddr 'localhost' --rpcport 8502 --rpcapi 'personal,db,eth,net,web3,txpool,miner' --bootnodes 'enode://42735281a2058b0eedcebf3fa579253e654dbdf945b89ac8a44fc4a3386bd14bca3f0cb19947a97f8a4303b8851b23189b297166c369ae45a7f90cb013c52c5e@127.0.0.1:30310' --networkid 1515 --gasprice '0' --unlock '0xcdd45d92a9b151b0eea54e70914ce5ea2dfdc249' --password node2/password.txt --mine
```

### Your First Transaction

First check which is your coinbase, on each node, run the following.

```
> eth.coinbase
"0xf4ad40a428b02f7a45ce6fa9fc3f87c4cab20f0a"
```

#### Send transaction  

```
> eth.sendTransaction({'from':eth.coinbase, 'to':'0xcdd45d92a9b151b0eea54e70914ce5ea2dfdc249', 'value':web3.toWei(3, 'ether')})

"0xcb3ff42f865610e195b24402700223127d1d0caeea796847f0046e2c7cfaa071"
```

#### Get Transaction

```
> eth.getTransactionReceipt("0xcb3ff42f865610e195b24402700223127d1d0caeea796847f0046e2c7cfaa071")
{
  blockHash: "0x2d106099ed00f0ee6737a9573723a4fc5f4dc9f57cdae2042fd545cd20ca5cfb",
  blockNumber: 49,
  contractAddress: null,
  cumulativeGasUsed: 21000,
  from: "0xf4ad40a428b02f7a45ce6fa9fc3f87c4cab20f0a",
  gasUsed: 21000,
  logs: [],
  logsBloom: "0x00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000",
  status: "0x1",
  to: "0xcdd45d92a9b151b0eea54e70914ce5ea2dfdc249",
  transactionHash: "0xcb3ff42f865610e195b24402700223127d1d0caeea796847f0046e2c7cfaa071",
  transactionIndex: 0
}
```

## My first Ethereum SmartContract

Following the expample from [Greeter](https://www.ethereum.org/greeter)

### Solc on your machine

If you installed the compiler on your machine, you need to compile the contract to acquire the compiled code and Application Binary Interface.

`solc -o target --bin --abi Greeter.sol`

This will create two files, one file containing the compiled code and one file creating the Application Binary Interface in a directory called target.

## Basic Commands

```
> net.listening
true

> net.peerCount
4

> admin.peers

> admin.nodeInfo

```

### Basic Curl Requests

##### Admin Peers

```
curl -X POST --data '{"jsonrpc":"2.0","method":"admin_peers","params":[],"id":0}' http://localhost:22000 | json_pp > kkk

curl -X POST --data '{"jsonrpc":"2.0","method":"admin_peers","params":[],"id":0}' http://localhost:22000
```

##### NodeInfo

```
curl -X POST --data '{"jsonrpc":"2.0","method":"admin_nodeInfo","params":[],"id":0}' http://localhost:22000
```

##### TxPool status and content

```
curl -X POST --data '{"jsonrpc":"2.0","method":"txpool_status","params":[],"id":0}' http://localhost:22000

curl -X POST --data '{"jsonrpc":"2.0","method":"txpool_content","params":[],"id":0}' http://localhost:22000
```

##### Eth Syncinc, mining, gasPrice and blockNumber

```
curl -X POST --data '{"jsonrpc":"2.0","method":"eth_syncing","params":[],"id":1}'

curl -X POST --data '{"jsonrpc":"2.0","method":"eth_mining","params":[],"id":71}'

curl -X POST --data '{"jsonrpc":"2.0","method":"eth_gasPrice","params":[],"id":73}'

curl -X POST --data '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":83}' http://localhost:22000

```