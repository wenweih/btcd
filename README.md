## 修复开源项目 btcd 实现比特币获取区块 RPC 的问题
[btcd](https://github.com/btcsuite/btcd) 提供的 RPC 接口中，获取区块和交易详情接口 [GetBlockVerboseTx](https://github.com/btcsuite/btcd/blob/master/rpcclient/chain.go#L177) 存在一个 BUG, 具体可以查看 issue :

**[[RPC] getblock command has been changed](https://github.com/btcsuite/btcd/issues/1096)**

getblock 接口详情如下
```shell
getblock "blockhash" ( verbosity )

If verbosity is 0, returns a string that is serialized, hex-encoded data for block 'hash'.
If verbosity is 1, returns an Object with information about block <hash>.
If verbosity is 2, returns an Object with information about block <hash> and information about each transaction.

Arguments:
1. "blockhash"          (string, required) The block hash
2. verbosity              (numeric, optional, default=1) 0 for hex encoded data, 1 for a json object, and 2 for json object with transaction data
```
我没有在 btcd 提 pull request, 故把修改过的文件提交到这个 repo, 使用 btcd 时直接覆盖对应文件即可。

[Command: getblock](https://bitcoin.org/en/developer-reference#getblock)
## 修改的代码
在 btcd 中，获取区块和区块中所有交易详情的 RPC 方法是：
```golang
func (c *Client) GetBlockVerboseTx(blockHash *chainhash.Hash) (*btcjson.GetBlockVerboseResult, error) {
	return c.GetBlockVerboseTxAsync(blockHash).Receive()
}
```
接着我们再看看 ```GetBlockVerboseTxAsync``` 方法：
```golang
func (c *Client) GetBlockVerboseTxAsync(blockHash *chainhash.Hash) FutureGetBlockVerboseResult {
	hash := ""
	if blockHash != nil {
		hash = blockHash.String()
	}

	cmd := btcjson.NewGetBlockCmd(hash, btcjson.Bool(true), btcjson.Bool(true))
	return c.sendCmd(cmd)
}
```
其中 ```NewGetBlockCmd``` 函数接受三个参数，后面两个为布尔类型，显然是不对的，getblock 接口格式为 ```getblock "blockhash" ( verbosity )```

因为我只是做了小修改再加上对 btcd 源码不怎么熟悉，为了不引入新的 bug ，我在这里只添加代码，不删除，不修改 ...
```golang
func (c *Client) GetBlockVerboseTxM(blockHash *chainhash.Hash) (*btcjson.GetBlockVerboseResult, error) {
	return c.GetBlockVerboseTxAsyncM(blockHash).Receive()
}

...

func (c *Client) GetBlockVerboseTxAsyncM(blockHash *chainhash.Hash) FutureGetBlockVerboseResult {
	hash := ""
	if blockHash != nil {
		hash = blockHash.String()
	}

	cmd := btcjson.NewGetBlockCmdM(hash, 2)
	return c.sendCmd(cmd)
}
```
在 ```rpcclient/chain.go``` 修改，不删除原来的方法，追加修改的两个方法。接下来继续看看 ```NewGetBlockCmdM``` 方法，在 ```btcjson/chainsvrcmds.go``` 文件中：
```golang
func NewGetBlockCmdM(hash string, verbose int) *GetBlockCmdM {
	return &GetBlockCmdM{
		Hash:    hash,
		Verbose: &verbose,
	}
}

...
// GetBlockCmdM defines the getblock JSON-RPC command.
type GetBlockCmdM struct {
	Hash    string
	Verbose *int `jsonrpcdefault:"1"`
}
...
```
，通过 ```getblock "blockhash" 2``` 在比特币节点的返回数据中，Unmarshal 后的结构体 ```GetBlockVerboseResult``` ```Tx``` 字段类型应该改为 ```TxRawResult```，在 ```btcjson/chainsvrresults.go``` 文件中:
```going
...
type GetBlockVerboseResult struct {
	Hash          string        `json:"hash"`
	Confirmations uint64        `json:"confirmations"`
	StrippedSize  int32         `json:"strippedsize"`
	Size          int32         `json:"size"`
	Weight        int32         `json:"weight"`
	Height        int64         `json:"height"`
	Version       int32         `json:"version"`
	VersionHex    string        `json:"versionHex"`
	MerkleRoot    string        `json:"merkleroot"`
	Tx            []TxRawResult `json:"tx,omitempty"`
	RawTx         []TxRawResult `json:"rawtx,omitempty"`
	Time          int64         `json:"time"`
	Nonce         uint32        `json:"nonce"`
	Bits          string        `json:"bits"`
	Difficulty    float64       `json:"difficulty"`
	PreviousHash  string        `json:"previousblockhash"`
	NextHash      string        `json:"nextblockhash,omitempty"`
}
```
最后，在 ```btcjson/chainsvrcmds.go``` 文件中修改 getblock 命令和修改后的方法绑定:
```golang
MustRegisterCmd("getblock", (*GetBlockCmdM)(nil), flags)
```
## fix verbocity in getblock command for btcd
When i call getblock by btcd rpcclient, the generated rpc command looks like this:
```json
{
  "jsonrpc":"1.0",
  "method":"getblock",
  "params":
    ["0000000000000000015ed7a2934f8ecb39eda15459376fff64284e1a87688d5d",true,true],
  "id":1
}
```
the right generation sendCmd should look like this:
```json
{
  "jsonrpc":"1.0",
  "method":"getblock",
  "params":["0000000000000000015ed7a2934f8ecb39eda15459376fff64284e1a87688d5d",2],
  "id":1
}
```
Original "getblock" command accept 2 arguments:
1. blockhash
2. verbosity (numeric, optional, default=1) 0 for hex encoded data, 1 for a json object, and 2 for json object with transaction data

```shell
getblock "blockhash" ( verbosity )

If verbosity is 0, returns a string that is serialized, hex-encoded data for block 'hash'.
If verbosity is 1, returns an Object with information about block <hash>.
If verbosity is 2, returns an Object with information about block <hash> and information about each transaction.

Arguments:
1. "blockhash"          (string, required) The block hash
2. verbosity              (numeric, optional, default=1) 0 for hex encoded data, 1 for a json object, and 2 for json object with transaction data
```
