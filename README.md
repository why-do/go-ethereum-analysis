## Go Ethereum Code Analysis

一.以太坊逻辑分层

应用层、合约层、激励层、共识层、网络层、数据层

二.分层信息

1.数据层：存储以太坊区块链的所有数据，本质是一数据；

2.网络层：P2P网络，在以太坊中网络层使用kad；

3.共识层：规定通过何种方式实现交易记录的过程；

4.激励层：以太坊采用POW(与比特币不同)，实现挖矿奖励机制；

5.合约层：平台层，通过合约层开发任意的DAPP，实现完整的区块链系统，在以太坊中，合约主要包括EVM和智能合约；

6.应用层：以太坊最上层。主要使用truffle和web3.js技术。

三.以太坊源码

```
account				实现的是以太坊账户管理
bmt					二进制的Merkle树的实现
build				实现编译和构建相关脚本与配置
cmd					命令行工具
common				提供公共的工具类
consensus			以太坊共识算法类
core				以太坊核心的数据结构和算法（区块、虚拟机、区块链...）
crypto				以太坊加密算法的实现
eth					以太坊协议
ethClient			以太坊RPC客户端
ethdb				以太坊数据库：包含持久化数据库与测试使用的内存数据库
ethStatus			以太坊网络状态的报告
event				以太坊事件
les					以太坊轻量级协议
light				为以太坊轻量级客户端提供索引功能
log					提供日志功能
miner				以太坊挖矿
mobile				移动端相关管理
node				以太坊各种类型的节点
p2p					以太坊主要的网络协议
rlp					以太坊系列化处理
swarm				以太坊swarm网络处理
tests				测试
trie				以太坊主要数据结构帕特里夏树的实现
whisper				提供whisper节点协议，主要用于以太坊DAPP之间的通信
```



区块与区块链

1.区块（Block）

1.所有与账户相关的活动都会以交易的格式存储到Block中，每个block中都有一个交易列表

2.交易执行结构，日志记录

3.不同的区块ParentHash进行链接



```
// 区块结构
type Block struct {
	header       *Header
	uncles       []*Header
	transactions Transactions

	// caches
	// 哈希与size的缓存
	// atomic:原子操作
	hash atomic.Value
	size atomic.Value

	// Td is used by package core to store the total difficulty
	// of the chain up to and including the block.
	// 挖矿的总难度
	td *big.Int

	// These fields are used by package eth to track
	// inter-peer block relay.
	ReceivedAt   time.Time
	ReceivedFrom interface{}
}

// 区块头
type Header struct {
	ParentHash  common.Hash    `json:"parentHash"       gencodec:"required"`
	UncleHash   common.Hash    `json:"sha3Uncles"       gencodec:"required"`
	Coinbase    common.Address `json:"miner"            gencodec:"required"`
	// Merkle根节点的哈希值
	Root        common.Hash    `json:"stateRoot"        gencodec:"required"`
	TxHash      common.Hash    `json:"transactionsRoot" gencodec:"required"`
	ReceiptHash common.Hash    `json:"receiptsRoot"     gencodec:"required"`
	// 布隆过滤器，快速判断一个log对象是否在一组已知的log集合中
	Bloom       Bloom          `json:"logsBloom"        gencodec:"required"`
	Difficulty  *big.Int       `json:"difficulty"       gencodec:"required"`
	Number      *big.Int       `json:"number"           gencodec:"required"`
	GasLimit    *big.Int       `json:"gasLimit"         gencodec:"required"`
	GasUsed     *big.Int       `json:"gasUsed"          gencodec:"required"`
	Time        *big.Int       `json:"timestamp"        gencodec:"required"`
	Extra       []byte         `json:"extraData"        gencodec:"required"`
	// 以太坊共识算法ethash与比特币共识pow所不同的一个关键变量
	MixDigest   common.Hash    `json:"mixHash"          gencodec:"required"`
	Nonce       BlockNonce     `json:"nonce"            gencodec:"required"`
}

// 存储以太坊区块链的交易信息
type Body struct {
	Transactions []*Transaction
	// uncle设计的目的就是为了抵消以太坊网络中计算能力强的节点破坏去中心化的原则
	Uncles       []*Header
}


// 生成区块头的哈希
func (h *Header) Hash() common.Hash {
	return rlpHash(h)
}

// 区块头的RLP hash值，rlp是一种编码规则
func rlpHash(x interface{}) (h common.Hash) {
	hw := sha3.NewKeccak256()
	rlp.Encode(hw, x)
	hw.Sum(h[:0])
	return h
}

```



新建区块

core/types/block.go

```
// 新建区块
func NewBlock(header *Header, txs []*Transaction, uncles []*Header, receipts []*Receipt) *Block {
	b := &Block{header: CopyHeader(header), td: new(big.Int)}

	// TODO: panic if len(txs) != len(receipts)
	if len(txs) == 0 {
		b.header.TxHash = EmptyRootHash
	} else {
		b.header.TxHash = DeriveSha(Transactions(txs))
		b.transactions = make(Transactions, len(txs))
		copy(b.transactions, txs)
	}

	if len(receipts) == 0 {
		b.header.ReceiptHash = EmptyRootHash
	} else {
		b.header.ReceiptHash = DeriveSha(Receipts(receipts))
		b.header.Bloom = CreateBloom(receipts)
	}

	if len(uncles) == 0 {
		b.header.UncleHash = EmptyUncleHash
	} else {
		b.header.UncleHash = CalcUncleHash(uncles)
		b.uncles = make([]*Header, len(uncles))
		for i := range uncles {
			b.uncles[i] = CopyHeader(uncles[i])
		}
	}

	return b
}
```



区块链

core/blockchain.go:主要是对区块链状态进行维护，包括区块的验证、插入及状态查询

```
func NewBlockChain(chainDb ethdb.Database, config *params.ChainConfig, engine consensus.Engine, vmConfig vm.Config) (*BlockChain, error) {
	bodyCache, _ := lru.New(bodyCacheLimit)
	bodyRLPCache, _ := lru.New(bodyCacheLimit)
	blockCache, _ := lru.New(blockCacheLimit)
	futureBlocks, _ := lru.New(maxFutureBlocks)
	badBlocks, _ := lru.New(badBlockLimit)

	bc := &BlockChain{
		config:       config,
		chainDb:      chainDb,
		stateCache:   state.NewDatabase(chainDb),
		quit:         make(chan struct{}),
		bodyCache:    bodyCache,
		bodyRLPCache: bodyRLPCache,
		blockCache:   blockCache,
		futureBlocks: futureBlocks,
		engine:       engine,
		vmConfig:     vmConfig,
		badBlocks:    badBlocks,
	}
	bc.SetValidator(NewBlockValidator(config, bc, engine))
	bc.SetProcessor(NewStateProcessor(config, bc, engine))

	var err error
	bc.hc, err = NewHeaderChain(chainDb, config, engine, bc.getProcInterrupt)
	if err != nil {
		return nil, err
	}
	// 获取创世区块
	bc.genesisBlock = bc.GetBlockByNumber(0)
	if bc.genesisBlock == nil {
		return nil, ErrNoGenesis
	}
	// 获取最新状态
	if err := bc.loadLastState(); err != nil {
		return nil, err
	}
	// Check the current state of the block hashes and make sure that we do not have any of the bad blocks in our chain
	// 检查是否有坏区块
	for hash := range BadHashes {
		if header := bc.GetHeaderByHash(hash); header != nil {
			// get the canonical block corresponding to the offending header's number
			headerByNumber := bc.GetHeaderByNumber(header.Number.Uint64())
			// make sure the headerByNumber (if present) is in our current canonical chain
			if headerByNumber != nil && headerByNumber.Hash() == header.Hash() {
				log.Error("Found bad hash, rewinding chain", "number", header.Number, "hash", header.ParentHash)
				bc.SetHead(header.Number.Uint64() - 1)
				log.Error("Chain rewind was successful, resuming normal operation")
			}
		}
	}
	// Take ownership of this particular state
	go bc.update()
	return bc, nil
}

```



创世区块生成分析

入口cmd/geth/chaincmd.go:initGenesis

生成创世区块之后，整个区块链配置完成



挖矿

以太坊中的共识ethash

1.eth与bitcoin一样，采用的都是基于工作量的POW共识来产生新的区块，与比特币不同的是，eth采用可以抵御ASIC对挖矿工作的垄断，就是ethash。

2.ASIC：专门为特定的用途设计的电子电路芯片，在比特币中就叫矿机芯片

3.与CPU、GPU相比，ASIC算力能够高出数万倍

4.在比特币中，单从挖矿来说，已经不是一个去中心化思想的区块链了

5.ethash：专门针对比特币中算力集中的情况所设计的一种算法

6.ethash在bitcoin的工作量证明的基础上增加了内存消耗的步骤

7.在ethash中，除了进行比特币中的POW，还添加了历史交易数据的混淆运算。矿工节点在运算该算法的时候还需要去访问内存中的历史交易信息（内存消耗的来源）

8.在以太坊中，会有一个以特定算法生成的大小1GB左右的数据集合，矿工节点在挖矿时需要把这1GB的数据全部加载到内存中去

挖矿代码入口：miner/agent.go -> Start()



共识

概念：区块链中的各个节点对下一个区块的内容形成一致的认识

源码：

​		测试网络采用Clique

​		公网采用ethash

ethash解析：



僵尸工厂

https://cryptozombies.io/

僵尸工厂概述

