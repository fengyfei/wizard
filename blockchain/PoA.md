# PoA 详解

## 1. PoA 理论介绍

### 1.1 以太坊中 PoA 产生的背景

如果你想用以太坊搭建一个联盟/私有链, 并要求该链交易成本更低甚至没有, 交易延时更低,并发更高, 还拥有完全的控制权(意味着被攻击概率更低). 目前以太坊采用 PoW 或后续的 casper 能否满足要求?

- 首先, PoW 存在 **51% 攻击问题**, 恶意挖矿者超过全网算力的 51% 后基本上就能完全控制整个网络. 由于链无法被更改, 已上链的数据也无法更改, 但恶意挖矿者也可以做一些 DoS 攻击阻止合法交易上链,考虑到具有相同创世块的旷工都能加入你的网络, 潜在的安全隐患会长期存在.
- 其次, PoW **大量的电力资源消耗**也是需要作为后续成本考虑. PoS 可以解决部分 PoW 问题, 比如节约电力,在一定程度上保护了51％的攻击(恶意旷工会被惩罚), 但从控制权和安全考虑还有欠缺, 因为 PoS 还是允许任何符合条件的旷工加入。

在已经运行的测试网络 Ropsten 中, 由于 PoW 设定的难度较低,恶意旷工滥用较低的 PoW 难度并将 gaslimit 扩大到90亿（正常是470万），发送大量的交易瘫痪了整个网络。而在此之前，攻击者也尝试了多次非常长的重组(reorgs)，导致不同客户端之间的分叉，甚至不同的版本。

这些攻击的根本原因是 PoW 网络的安全性依赖于背后的算力。而从零开始重新启动一个新的 testnet 将不会解决任何问题，因为攻击者可以一次又一次地进行相同的攻击。 Parity 团队决定采取紧急解决方案，回滚大量的块，并设置不允许 gaslimit 超过某一阈值的软分叉规则。

虽然 Parity 的解决方案可能在短期内有效, 但是这不是优雅的：Ethereum 本身应该具有动态 gaslimit 限制; 也不可移植：其他客户端需要自己实现新的软分叉逻辑; 并与同步模式不兼容, 也不支持轻客户端; 尽管并不完美，但是Parity 的解决方案仍然可行。 一个更长期的替代解决方案是使用 PoA 共识,相对简单并容易实现.

### 1.2. PoA 的特点

- PoA 是依靠预设好的授权节点(signers)，负责产生 block.
- 可以由已授权的 signer 选举(投票超过50%)加入新的 signer。
- 即使存在恶意 signer,他最多只能攻击连续块(数量是 `(SIGNER_COUNT / 2) + 1)` 中的1个,期间可以由其他signer 投票踢出该恶意 signer。
- 可指定产生 block 的时间。

### 1.3. PoA需要解决的问题

1. 如何控制挖矿频率,即出块时间
2. 如何验证某个块的有效性
3. 如何动态调整授权签名者(signers)列表,并全网动态同步
4. 如何在 signers 之间分配挖矿的负载或者叫做挖矿的机会

对应的解决办法如下:

1. 协议规定采用固定的 block 出块时间, 区块头中的时间戳间隔为 15s
2. 先看看 block 同步的方法,从中来分析 PoA 中验证 block 的解决办法

有两种同步 blockchain 的方法

1. 经典方法是从创世块开始挨个执行所有交易。 这是经过验证的，但是在 Ethereum 的复杂网络中，计算量非常大。
2. 另一个是仅下载区块头并验证其有效性，之后可以从网络下载任意的近期状态对最近的区块头进行检查。

显然第二种方法更好. 由于 PoA 方案的块可以仅由可信任的签名者来创建, 因此，客户端看到的每个块（或块头）可以与可信任签名者列表进行匹配。 要验证该块的有效性就必须得到该块对应的签名者列表, 如果签名者在该列表中带包该块有效. 这里的挑战是如何维护并及时更改的授权签名者列表？ 存储在智能合约中?不可行, 因为在快速轻量级同步期间无法访问状态。

因此, **授权签名者列表必须完全包含在块头中** 。那么需要改变块头的结构, 引入新的字段来满足投票机制吗? 这也不可行：改变这样的核心数据结构将是开发者的噩梦。

所以授权签名者名单必须完全符合当前的数据模型, 不能改变区块头中的字段，而是 **复用当前可用的字段: Extra 字段. **

**Extra** 是可变长数组, 对它的修改是 `非侵入` 操作, 比如 RLP,hash 操作都支持可变长数据. Extra 中包含所有签名者列表和当前节点的签名者对该区块头的签名数据(可以恢复出来签名者的地址).

1. 更新一个动态的签名者列表的方法是复用区块头中的 **Coinbase 和 Nonce 字段** ，以创建投票方案：

- 常规的块中这两个字段置为0
- 如果签名者希望对授权签名者列表进行更改，则将：
  - **Coinbase** 设置为被投票的签名者
  - 将 **Nonce** 设置为 **0** 或 **0xff ... f** 投票,代表 **添加或移除**
  - 任何同步的客户端都可以在块处理过程中“统计”投票，并通过投票结果来维护授权签名者列表。

为了避免一个无限的时间来统计投票，我们设置一个投票窗口, 为一个 epoch,长度是30000个block。每个 epoch的起始清空所有历史的投票, 并作为签名者列表的检查点. 这允许客户端仅基于检查点哈希进行同步，而不必重播在链路上完成的所有投票。

1. 目前的方案是在所有 signer 之间轮询出块, 并通过算法保证同一个 signer 只能签名 `(SIGNER_COUNT / 2) + 1)` 个 block 中第一个.

综上, PoA 的工作流程如下:

1. 在创世块中指定一组初始授权的 signers, **所有地址** 保存在创世块 Extra 字段中
2. 启动挖矿后, 该组 signers 开始对生成的 block 进行 **签名并广播**
3. **签名结果** 保存在区块头的 Extra 字段中
4. Extra 中更新当前高度已授权的 **所有 signers 的地址** ,因为有新加入或踢出的 signer
5. 每一高度都有一个 signer 处于 IN-TURN 状态, 其他 signer 处于 OUT-OF-TURN 状态,  IN-TURN 的 signer 签名的 block 会**立即广播** , OUT-OF-TURN 的 signer 签名的 block 会 **延时**一点随机时间后再广播, 保证 IN-TURN的签名 block 有更高的优先级上链
6. 如果需要加入一个新的 signer, signer 通过 API 接口发起一个 proposal, 该 proposal 通过复用区块头  **Coinbase (新 signer 地址)和 Nonce("0xffffffffffffffff")** 字段广播给其他节点. 所有已授权的 signers 对该新的 signer 进行"加入"投票, 如果赞成票超过 signers 总数的50%, 表示同意加入
7. 如果需要踢出一个旧的 signer, 所有已授权的 signers 对该旧的 signer 进行"踢出"投票, 如果赞成票超过signers 总数的50%, 表示同意踢出

#### 1.4 signer 对区块头进行签名

1. Extra 的长度至少65字节以上(签名结果是65字节,即 R, S, V, V是0或1)
2. 对 blockHeader中所有字段除了 Extra 的 **后65字节** 外进行 **RLP 编码**
3. 对编码后的数据进行 `Keccak256` **hash**
4. 签名后的数据(65字节)保存到 Extra 的 **后65字节** 中

#### 1.5 授权策略

以下建议的策略将减少网络流量和分叉

- 如果签名者被允许签署一个块（在授权列表中，但最近没有签名）。
  - 计算下一个块的最优签名时间（父块时间+ BLOCK_PERIOD）。
  - 如果签名人是 in-turn，立即进行签名和广播block。
  - 如果签名者是 out-of-turn，延迟 `rand(SIGNER_COUNT * 500ms)` 后再签名并广播

#### 1.6 级联投票

当移除一个授权的签名者时,可能会导致其他移除前的投票成立. 例: ABCD 4 个 signer, AB加入E,此时不成立(没有超过50%), 如果 ABC 移除 D, 会自动导致加入 E 的投票成立(2/3的投票比例)

#### 1.7 投票策略

因为 blockchain 可能会小范围重组(small reorgs), 常规的投票机制(cast-and-forget, 投票和忘记)可能不是最佳的，因为包含单个投票的 block 可能不会在最终的链上,会因为已有最新的 block 而被抛弃。

一个简单但有效的办法是对 signers 配置"提议( proposal )".例如 "add 0x...", "drop 0x...", 有多个并发的提议时, 签名代码"随机"选择一个提议注入到该签名者签名的 block 中,这样多个并发的提议和重组( reorgs )都可以保存在链上.

该列表可能在一定数量的 block/epoch 之后过期，提案通过并不意味着它不会被重新调用，因此在提议通过时不应立即丢弃。

- 加入和踢除新的 signer 的投票都是立即生效的,参与下一次投票计数
- 加入和踢除都需要 **超过当前 signer 总数的50%** 的 signer 进行投票
- 可以踢除自己(也需要超过50%投票)
- 可以并行投票(A,B交叉对C,D进行投票), 只要最终投票数操作50%
- 进入一个新的 epoch, 所有之前的 pending 投票都作废, 重新开始统计投票

#### 1.8 投票场景举例

- ABC, C踢除B, A踢除C, B踢除C, A踢除B, 结果是剩下 AB
- ABCD, AB先分别踢除CD, C踢除D, 即使C投给自己留下的票, 结果是剩下 AB
- ABCDE, ABC先分别加入F(成功,ABCDEF), BCDE踢除F(成功,ABCDE), DE 加入F(失败,ABCDE), BCD踢除A(成功, BCDE), B加入F(此时BDE加入F,满足超过50%投票), 结果是剩下 BCDEF

### 1.9 PoA中的攻击及防御

- 恶意签名者(Malicious signer). 恶意用户被添加到签名者列表中，或签名者密钥/机器遭到入侵. 解决方案是，N个授权签名人的列表，任一签名者只能对每K个 block 签名其中的1个。这样尽量减少损害，其余的矿工可以投票踢出恶意用户。
- 审查签名者(Censoring signer). 如果一个签名者（或一组签名者）试图检查 block 中其他 signer 的提议(特别是投票踢出他们), 为了解决这个问题，我们将签名者的允许的挖矿频率限制在1/(N/2)。如果他不想被踢出出去, 就必须控制超过50%的signers.
- "垃圾邮件"签名者(Spamming signer). 这些 signer 在每个他们签名的 block 中都注入一个新的投票提议.由于节点需要统计所有投票以创建授权签名者列表, 久而久之之后会产生大量垃圾的无用的投票, 导致系统运行变慢.通过 epoch 的机制,每次进入新的epoch都会丢弃旧的投票
- 并发块(Concurrent blocks). 如果授权签名者的数量为N，我们允许每个签名者签名是1/K，那么在任何时候，至少N-K个签名者都可以成功签名一个 block。为了避免这些 block 竞争( **分叉** )，每个签名者生成一个新 block 时都会加一点随机延时。这确保了很难发生分叉。


## 2. PoA共识引擎算法实现分析


节点：普通的以太坊节点，没有区块生成的权利。

矿工：具有区块生成权利的以太坊节点

委员会：所有矿工的集合

###  投票方法

所有投票都是在委员生成新区块的过程中完成，具体流程如下：

- 1）委员生成新区块时，先为该区块初始化一个 header。（prepare 方法，consensus/clique/clique.go）
- 2）从 proposals 中随机获取一个投票，将被投票的节点地址写入 header.coinbase，将提名是添加还是删除写入 header.Nonce (添加：0xffffffffffffffff 删除：0)，若该委员生成的这个区块最终被写入区块链，则header 中的投票也被写入区块链。（ prepare 方法，consensus/clique/clique.go）
- 3）委员在生成新区块时，会创建新的 snapshot，新的 snapshot 是由上一 checkponitinterval 时间点存储到数据库中的快照加入当前时间点和 checkpointinterval 时间点之间所有的 headers 数据组成。添加 header 过程中，若该 header 的 number 是 Epoch 时间点，则会将 snap 中的 Votes 和 Tally 两个集合清零。（apply方法，consensus/clique/snapshot.go）
- 4）新的 snapshot 添加 header 过程中，会检查每一个 header 中存储的投票，若该投票 snap.Votes 中已经存在，则将 snap.Votes 和 snap.Tally 两个集合的该投票删除。（apply 方法，consensus/clique/snapshot.）将每一个 header 中有效的提名写入新snapshot的snap.Votes和snap.Tally集合。（apply方法，consensus/clique/snapshot.go）
- 5）判断snap.Tally集合中某个被提名的节点，提名的次数是否大于snap.Signers的1/2,即是否有超过一半的委员对该节点进行投票，若超过，则投票成功，该节点会被添加到委员会或者从委员会中删除。（apply方法，consensus/clique/snapshot.go）

**注释：snapshot 快照中的记录的委员会，即 Signers 集合，初始化时来源于创世块 header 中的 Extra**


### clique 中一些概念和定义

- **EPOCH_LENGTH** : epoch 长度是30000个 block, 每次进入新的epoch,前面的投票都被清空,重新开始记录,这里的投票是指加入或移除signer
- **CheckpointInterval**：为常量1024（consensus/clique/clique.go中定义），即每当区块链的高度为1024的整数倍时，到达checkpointInterval时间点。
- **BLOCK_PERIOD** : 出块时间, 默认是15s
- **UNCLE_HASH** : 总是 `Keccak256(RLP([]))` ,因为没有uncle
- **SIGNER_COUNT** : 每个block都有一个signers的数量
- **SIGNER_LIMIT** : 等于 `(SIGNER_COUNT / 2) + 1`
  * 每个singer只能签名连续SIGNER_LIMIT个block中的1个, 比如有5个signer:ABCDE, 对4个block进行签名, 不允许签名者为ABAC, 因为A在连续3个block中签名了2次
- **NONCE_AUTH** : 表示投票类型是加入新的signer; 值= `0xffffffffffffffff`
- **NONCE_DROP** : 表示投票类型是踢除旧的的signer; 值= `0x0000000000000000`
- **EXTRA_VANITY** : 代表block头中Extra字段中的保留字段长度: 32字节
- **EXTRA_SEAL** : 代表block头中Extra字段中的存储签名数据的长度: 65字节
- **IN-TURN/OUT-OF-TURN** : 每个block都有一个in-turn的signer, 其他signers是out-of-turn, in-turn的signer的权重大一些, 出块的时间会快一点, 这样可以保证该高度的block被in-turn的signer挖到的概率很大.

clique中最重要的两个数据结构:

- 共识引擎的结构:

```go
type Clique struct {
	config *params.CliqueConfig // 系统配置参数
	db ethdb.Database // 数据库: 用于存取检查点快照
	recents *lru.ARCCache //保存最近block的快照, 加速reorgs
	signatures *lru.ARCCache //保存最近block的签名, 加速挖矿
	proposals map[common.Address]bool //当前signer提出的proposals列表
	signer common.Address // signer地址
	signFn SignerFn // 签名函数
	lock sync.RWMutex // 读写锁
}
```

用户通过RPC接口，调用Propose(address common.Address, auth bool)方法（consensus/clique/api.go）,进行投票，address表示要投票的节点的地址，auth表示要从将该地址加入委员会，还是从委员会中删除。

Propose 方法将 address 和 auth 两个输入参数写入到 clique.proposals 集合中。

任何一个委员会的委员，可以在任意时刻进行投票，投票包括两种，即加入委员会和从委员会中删除。

- snapshot的结构:

```go
type Snapshot struct {
	config *params.CliqueConfig // 系统配置参数
	sigcache *lru.ARCCache // 保存最近block的签名缓存,加速ecrecover
	Number uint64 // 创建快照时的block号,即生成快照时的区块链高度
	Hash common.Hash // 创建快照时的block hash
	Signers map[common.Address]struct{} // 此刻的授权的signers
	Recents map[uint64]common.Address // 最近的一组signers, key=blockNumber
	Votes []*Vote // 按时间顺序排列的投票列表
	Tally map[common.Address]Tally // 当前的投票计数，以避免重新计算，其中的Tally是该节点被投票的次数
}
```

Snapshot是一个快照，矿工程序在区块链高度为CheckpointInterval的整数倍时，会对当前相关数据和状态形成快照，并存储到数据库中。

除了这两个结构, 对block头的部分字段进行了复用定义, ethereum的block头定义:

```go
type Header struct {
	ParentHash common.Hash
	UncleHash common.Hash
	Coinbase common.Address
	Root common.Hash
	TxHash common.Hash
	ReceiptHash common.Hash
	Bloom Bloom
	Difficulty *big.Int
	Number *big.Int
	GasLimit *big.Int
	GasUsed *big.Int
	Time *big.Int
	Extra []byte
	MixDigest common.Hash
	Nonce BlockNonce
}
```

- 创世块中的Extra字段包括:
  - 32字节的前缀(extraVanity)
  - 所有signer的地址
  - 65字节的后缀(extraSeal): 保存signer的签名
- 其他block的Extra字段只包括extraVanity和extraSeal
- Time字段表示产生block的时间间隔是:blockPeriod(15s)
- Nonce字段表示进行一个投票: 添加( nonceAuthVote: `0xffffffffffffffff` )或者移除( nonceDropVote: `0x0000000000000000` )一个signer
- Coinbase字段存放被投票的地址
  - 举个栗子: signerA的一个投票:加入signerB, 那么Coinbase存放B的地址
- Difficulty字段的值: 2-是 **本block的签名者** (in turn), 1- **非本block的签名者** (out of turn)

POA共识算法中，委员会中的每一个矿工都会持续的生成新的区块，对于同一个Number的区块，不通的矿工生成该块时优先级不同。

优先级计算方法:

- Number:要生成的区块的块号
- Signers：snapshot中记录的委员会集合，并根据矿工的地址进行了升序排列
- Offset：矿工在Signers集合中的位置
- 若：(number % uint64(len(signers))) == uint64(offset)，则优先级最高，header. Difficulty =2;否则,header.Difficulty = 1

  总结如下：

| 序号   | 字段         | POW        | POA                        |
| ---- | ---------- | ---------- | -------------------------- |
| 1    | Coinbase   | 挖矿奖励地址     | 被提名为矿工的节点地址                |
| 2    | Nonce      | 随机数        | 提名分类，添加或删除                 |
| 3    | Extra      | 其他数据       | 在Epoch时间点，存储当前委员会集合Signers |
| 4    | Difficulty | 挖矿难度       | 优先级                        |
| 5    | Time       | 产生block的时间 | 产生block的时间间隔               |

下面对比较重要的函数详细分析实现流程

#### 共识引擎 clique 的初始化

在 `Ethereum.StartMining` 中,如果Ethereum.engine配置为clique.Clique, 根据当前节点的矿工地址(默认是acounts[0]), 配置clique的 **签名者** : `clique.Authorize(eb, wallet.SignHash)` ,其中 **签名函数** 是SignHash,对给定的hash进行签名.

```go
// New creates a Clique proof-of-authority consensus engine with the initial
// signers set to the ones provided by the user.
func New(config *params.CliqueConfig, db ethdb.Database) *Clique {
	// Set any missing consensus parameters to their defaults
	conf := *config
	if conf.Epoch == 0 {
		conf.Epoch = epochLength
	}
	// Allocate the snapshot caches and create the engine
	recents, _ := lru.NewARC(inmemorySnapshots)
	signatures, _ := lru.NewARC(inmemorySignatures)

	return &Clique{
		config:     &conf,
		db:         db,
		recents:    recents,
		signatures: signatures,
		proposals:  make(map[common.Address]bool),
	}
}
```

#### Clique.Prepare(chain , header)

Prepare是共识引擎接口之一. 该函数配置header中共识相关的参数(Cionbase, Difficulty, Extra, MixDigest, Time)

- 对于非epoch的block( `number % Epoch != 0` ):

1. 得到Clique.proposals中的投票数据(例:A加入C, B踢除D)
2. 根据snapshot的signers分析投票数否有效(例: C原先没有在signers中, 加入投票有效, D原先在signers中,踢除投票有效)
3. 从被投票的地址列表(C,D)中, **随机选择一个地址** ,作为该header的Coinbase,设置Nonce为加入( `0xffffffffffffffff` )或者踢除( `0x0000000000000000` )
4. `Clique.signer` 如果是本轮的签名者(in-turn), 设置header.Difficulty = diffInTurn(2), 否则就是diffNoTurn(1)
5. 配置header.Extra的数据为[ `extraVanity` + `snap中的全部signers` + `extraSeal` ]
6. MixDigest需要配置为nil
7. 配置时间戳:Time为父块的时间+15s

```go
// Prepare implements consensus.Engine, preparing all the consensus fields of the
// header for running the transactions on top.
func (c *Clique) Prepare(chain consensus.ChainReader, header *types.Header) error {
	// If the block isn't a checkpoint, cast a random vote (good enough for now)
	header.Coinbase = common.Address{}
	header.Nonce = types.BlockNonce{}

	number := header.Number.Uint64()
	// Assemble the voting snapshot to check which votes make sense
	snap, err := c.snapshot(chain, number-1, header.ParentHash, nil)
	if err != nil {
		return err
	}
	if number%c.config.Epoch != 0 {
		c.lock.RLock()

		// Gather all the proposals that make sense voting on
		addresses := make([]common.Address, 0, len(c.proposals))
		for address, authorize := range c.proposals {
			if snap.validVote(address, authorize) {
				addresses = append(addresses, address)
			}
		}
		// If there's pending proposals, cast a vote on them
		if len(addresses) > 0 {
			header.Coinbase = addresses[rand.Intn(len(addresses))]
			if c.proposals[header.Coinbase] {
				copy(header.Nonce[:], nonceAuthVote)
			} else {
				copy(header.Nonce[:], nonceDropVote)
			}
		}
		c.lock.RUnlock()
	}
	// Set the correct difficulty
	header.Difficulty = CalcDifficulty(snap, c.signer)

	// Ensure the extra data has all it's components
	if len(header.Extra) < extraVanity {
		header.Extra = append(header.Extra, bytes.Repeat([]byte{0x00}, extraVanity-len(header.Extra))...)
	}
	header.Extra = header.Extra[:extraVanity]

	if number%c.config.Epoch == 0 {
		for _, signer := range snap.signers() {
			header.Extra = append(header.Extra, signer[:]...)
		}
	}
	header.Extra = append(header.Extra, make([]byte, extraSeal)...)

	// Mix digest is reserved for now, set to empty
	header.MixDigest = common.Hash{}

	// Ensure the timestamp has the correct delay
	parent := chain.GetHeader(header.ParentHash, number-1)
	if parent == nil {
		return consensus.ErrUnknownAncestor
	}
	header.Time = new(big.Int).Add(parent.Time, new(big.Int).SetUint64(c.config.Period))
	if header.Time.Int64() < time.Now().Unix() {
		header.Time = big.NewInt(time.Now().Unix())
	}
	return nil
}
```

#### 获取给定时间点的一个快照 Clique.snapshot

- 先查找 Clique.recents 中是否有缓存, 有的话就返回该 snapshot

- 在查找持久化存储中是否有缓存, 有的话就返回该 snapshot

- 如果是创世块

  1. 从 Extra 中取出所有的 signers
  2. `newSnapshot(Clique.config, Clique.signatures, 0, genesis.Hash(), signers)`

  - signatures 是最近的签名快照
  - signers 是所有的初始 signers

  把 snapshot 加入到 Clique.recents 中, 并持久化到 db 中

- 其他普通块

  - 沿着父块 hash 一直往回找是否有 snapshot, 如果没找到就记录该区块头
  - 如果找到最近的 snapshot, 将前面记录的 headers 都 `applay` 到该 snapshot 上
  - 保存该最新的 snapshot 到缓存 Clique.recents 中, 并持久化到 db 中

```go
// snapshot retrieves the authorization snapshot at a given point in time.
func (c *Clique) snapshot(chain consensus.ChainReader, number uint64, hash common.Hash, parents []*types.Header) (*Snapshot, error) {
	// Search for a snapshot in memory or on disk for checkpoints
	var (
		headers []*types.Header
		snap    *Snapshot
	)
	for snap == nil {
		// If an in-memory snapshot was found, use that
		if s, ok := c.recents.Get(hash); ok {
			snap = s.(*Snapshot)
			break
		}
		// If an on-disk checkpoint snapshot can be found, use that
		if number%checkpointInterval == 0 {
			if s, err := loadSnapshot(c.config, c.signatures, c.db, hash); err == nil {
				log.Trace("Loaded voting snapshot form disk", "number", number, "hash", hash)
				snap = s
				break
			}
		}
		// If we're at block zero, make a snapshot
		if number == 0 {
			genesis := chain.GetHeaderByNumber(0)
			if err := c.VerifyHeader(chain, genesis, false); err != nil {
				return nil, err
			}
			signers := make([]common.Address, (len(genesis.Extra)-extraVanity-extraSeal)/common.AddressLength)
			for i := 0; i < len(signers); i++ {
				copy(signers[i][:], genesis.Extra[extraVanity+i*common.AddressLength:])
			}
			snap = newSnapshot(c.config, c.signatures, 0, genesis.Hash(), signers)
			if err := snap.store(c.db); err != nil {
				return nil, err
			}
			log.Trace("Stored genesis voting snapshot to disk")
			break
		}
		// No snapshot for this header, gather the header and move backward
		var header *types.Header
		if len(parents) > 0 {
			// If we have explicit parents, pick from there (enforced)
			header = parents[len(parents)-1]
			if header.Hash() != hash || header.Number.Uint64() != number {
				return nil, consensus.ErrUnknownAncestor
			}
			parents = parents[:len(parents)-1]
		} else {
			// No explicit parents (or no more left), reach out to the database
			header = chain.GetHeader(hash, number)
			if header == nil {
				return nil, consensus.ErrUnknownAncestor
			}
		}
		headers = append(headers, header)
		number, hash = number-1, header.ParentHash
	}
	// Previous snapshot found, apply any pending headers on top of it
	for i := 0; i < len(headers)/2; i++ {
		headers[i], headers[len(headers)-1-i] = headers[len(headers)-1-i], headers[i]
	}
	snap, err := snap.apply(headers)
	if err != nil {
		return nil, err
	}
	c.recents.Add(snap.Hash, snap)

	// If we've generated a new checkpoint snapshot, save to disk
	if snap.Number%checkpointInterval == 0 && len(headers) > 0 {
		if err = snap.store(c.db); err != nil {
			return nil, err
		}
		log.Trace("Stored voting snapshot to disk", "number", snap.Number, "hash", snap.Hash)
	}
	return snap, err
}
```

#### Snapshot.apply(headers)

创建一个新的授权signers的快照, 将从上一个snapshot开始的区块头中的proposals更新到最新的snapshot上

1. 对入参headers进行完整性检查: 因为可能传入多个区块头, **block号必须连续**
2. 遍历所有的header, 如果block号刚好处于epoch的起始(number%Epoch == 0),将snapshot中的Votes和Tally复位( **丢弃历史全部数据** )
3. 对于每一个header,从签名中恢复得到 **signer**
4. 如果该signer在snap.Recents中, 说明 **最近已经有过签名** , 不允许再次签名, 直接 **返回** 结束
5. 记录该signer最近已经有过签名，且是该block的签名者: `snap.Recents[number] = signer`
6. 统计header.Coinbase的投票数,如果超过signers总数的50%，执行加入或移除操作
7. 删除snap.Recents中的一个signer记录: key=number- (uint64(len(snap.Signers)/2 + 1)), 表示释放该signer,下次可以对block进行签名了
8. 清空被移除的Coinbase的投票
9. 移除snap.Votes中该Conibase的所有投票记录
10. 移除snap.Tally中该Conibase的所有投票数记录

```go
// apply creates a new authorization snapshot by applying the given headers to
// the original one.
func (s *Snapshot) apply(headers []*types.Header) (*Snapshot, error) {
	// Allow passing in no headers for cleaner code
	if len(headers) == 0 {
		return s, nil
	}
	// Sanity check that the headers can be applied
	for i := 0; i < len(headers)-1; i++ {
		if headers[i+1].Number.Uint64() != headers[i].Number.Uint64()+1 {
			return nil, errInvalidVotingChain
		}
	}
	if headers[0].Number.Uint64() != s.Number+1 {
		return nil, errInvalidVotingChain
	}
	// Iterate through the headers and create a new snapshot
	snap := s.copy()

	for _, header := range headers {
		// Remove any votes on checkpoint blocks
		number := header.Number.Uint64()
		if number%s.config.Epoch == 0 {
			snap.Votes = nil
			snap.Tally = make(map[common.Address]Tally)
		}
		// Delete the oldest signer from the recent list to allow it signing again
		if limit := uint64(len(snap.Signers)/2 + 1); number >= limit {
			delete(snap.Recents, number-limit)
		}
		// Resolve the authorization key and check against signers
		signer, err := ecrecover(header, s.sigcache)
		if err != nil {
			return nil, err
		}
		if _, ok := snap.Signers[signer]; !ok {
			return nil, errUnauthorized
		}
		for _, recent := range snap.Recents {
			if recent == signer {
				return nil, errUnauthorized
			}
		}
		snap.Recents[number] = signer

		// Header authorized, discard any previous votes from the signer
		for i, vote := range snap.Votes {
			if vote.Signer == signer && vote.Address == header.Coinbase {
				// Uncast the vote from the cached tally
				snap.uncast(vote.Address, vote.Authorize)

				// Uncast the vote from the chronological list
				snap.Votes = append(snap.Votes[:i], snap.Votes[i+1:]...)
				break // only one vote allowed
			}
		}
		// Tally up the new vote from the signer
		var authorize bool
		switch {
		case bytes.Equal(header.Nonce[:], nonceAuthVote):
			authorize = true
		case bytes.Equal(header.Nonce[:], nonceDropVote):
			authorize = false
		default:
			return nil, errInvalidVote
		}
		if snap.cast(header.Coinbase, authorize) {
			snap.Votes = append(snap.Votes, &Vote{
				Signer:    signer,
				Block:     number,
				Address:   header.Coinbase,
				Authorize: authorize,
			})
		}
		// If the vote passed, update the list of signers
		if tally := snap.Tally[header.Coinbase]; tally.Votes > len(snap.Signers)/2 {
			if tally.Authorize {
				snap.Signers[header.Coinbase] = struct{}{}
			} else {
				delete(snap.Signers, header.Coinbase)

				// Signer list shrunk, delete any leftover recent caches
				if limit := uint64(len(snap.Signers)/2 + 1); number >= limit {
					delete(snap.Recents, number-limit)
				}
				// Discard any previous votes the deauthorized signer cast
				for i := 0; i < len(snap.Votes); i++ {
					if snap.Votes[i].Signer == header.Coinbase {
						// Uncast the vote from the cached tally
						snap.uncast(snap.Votes[i].Address, snap.Votes[i].Authorize)

						// Uncast the vote from the chronological list
						snap.Votes = append(snap.Votes[:i], snap.Votes[i+1:]...)

						i--
					}
				}
			}
			// Discard any previous votes around the just changed account
			for i := 0; i < len(snap.Votes); i++ {
				if snap.Votes[i].Address == header.Coinbase {
					snap.Votes = append(snap.Votes[:i], snap.Votes[i+1:]...)
					i--
				}
			}
			delete(snap.Tally, header.Coinbase)
		}
	}
	snap.Number += uint64(len(headers))
	snap.Hash = headers[len(headers)-1].Hash()

	return snap, nil
}
```

#### Clique.Seal(chain, block , stop)

Seal也是共识引擎接口之一. 该函数用clique.signer对block的进行签名. 在pow算法中, 该函数进行hash运算来解"难题".

- 如果signer没有在snapshot的signers中,不允许对block进行签名
- 如果不是本block的签名者,延时一定的时间(随机)后再签名, 如果是本block的签名者, 立即签名.
- 签名结果放在Extra的extraSeal的65字节中

```go
// Seal implements consensus.Engine, attempting to create a sealed block using
// the local signing credentials.
func (c *Clique) Seal(chain consensus.ChainReader, block *types.Block, stop <-chan struct{}) (*types.Block, error) {
	header := block.Header()

	// Sealing the genesis block is not supported
	number := header.Number.Uint64()
	if number == 0 {
		return nil, errUnknownBlock
	}
	// For 0-period chains, refuse to seal empty blocks (no reward but would spin sealing)
	if c.config.Period == 0 && len(block.Transactions()) == 0 {
		return nil, errWaitTransactions
	}
	// Don't hold the signer fields for the entire sealing procedure
	c.lock.RLock()
	signer, signFn := c.signer, c.signFn
	c.lock.RUnlock()

	// Bail out if we're unauthorized to sign a block
	snap, err := c.snapshot(chain, number-1, header.ParentHash, nil)
	if err != nil {
		return nil, err
	}
	if _, authorized := snap.Signers[signer]; !authorized {
		return nil, errUnauthorized
	}
	// If we're amongst the recent signers, wait for the next block
	for seen, recent := range snap.Recents {
		if recent == signer {
			// Signer is among recents, only wait if the current block doesn't shift it out
			if limit := uint64(len(snap.Signers)/2 + 1); number < limit || seen > number-limit {
				log.Info("Signed recently, must wait for others")
				<-stop
				return nil, nil
			}
		}
	}
	// Sweet, the protocol permits us to sign the block, wait for our time
	delay := time.Unix(header.Time.Int64(), 0).Sub(time.Now()) // nolint: gosimple
	if header.Difficulty.Cmp(diffNoTurn) == 0 {
		// It's not our turn explicitly to sign, delay it a bit
		wiggle := time.Duration(len(snap.Signers)/2+1) * wiggleTime
		delay += time.Duration(rand.Int63n(int64(wiggle)))

		log.Trace("Out-of-turn signing requested", "wiggle", common.PrettyDuration(wiggle))
	}
	log.Trace("Waiting for slot to sign and propagate", "delay", common.PrettyDuration(delay))

	select {
	case <-stop:
		return nil, nil
	case <-time.After(delay):
	}
	// Sign all the things!
	sighash, err := signFn(accounts.Account{Address: signer}, sigHash(header).Bytes())
	if err != nil {
		return nil, err
	}
	copy(header.Extra[len(header.Extra)-extraSeal:], sighash)

	return block.WithSeal(header), nil
}
```

#### Clique.VerifySeal(chain, header)

VerifySeal也是共识引擎接口之一.

1. 从header的签名中恢复账户地址,改地址要求在snapshot的signers中
2. 检查header中的Difficulty是否匹配(in turn或out of turn)

```go
// VerifySeal implements consensus.Engine, checking whether the signature contained
// in the header satisfies the consensus protocol requirements.
func (c *Clique) VerifySeal(chain consensus.ChainReader, header *types.Header) error {
	return c.verifySeal(chain, header, nil)
}

// verifySeal checks whether the signature contained in the header satisfies the
// consensus protocol requirements. The method accepts an optional list of parent
// headers that aren't yet part of the local blockchain to generate the snapshots
// from.
func (c *Clique) verifySeal(chain consensus.ChainReader, header *types.Header, parents []*types.Header) error {
	// Verifying the genesis block is not supported
	number := header.Number.Uint64()
	if number == 0 {
		return errUnknownBlock
	}
	// Retrieve the snapshot needed to verify this header and cache it
	snap, err := c.snapshot(chain, number-1, header.ParentHash, parents)
	if err != nil {
		return err
	}

	// Resolve the authorization key and check against signers
	signer, err := ecrecover(header, c.signatures)
	if err != nil {
		return err
	}
	if _, ok := snap.Signers[signer]; !ok {
		return errUnauthorized
	}
	for seen, recent := range snap.Recents {
		if recent == signer {
			// Signer is among recents, only fail if the current block doesn't shift it out
			if limit := uint64(len(snap.Signers)/2 + 1); seen > number-limit {
				return errUnauthorized
			}
		}
	}
	// Ensure that the difficulty corresponds to the turn-ness of the signer
	inturn := snap.inturn(header.Number.Uint64(), signer)
	if inturn && header.Difficulty.Cmp(diffInTurn) != 0 {
		return errInvalidDifficulty
	}
	if !inturn && header.Difficulty.Cmp(diffNoTurn) != 0 {
		return errInvalidDifficulty
	}
	return nil
}

```

#### Clique.Finalize

Finalize也是共识引擎接口之一. 该函数生成一个block, 没有叔块处理,也没有奖励机制

1. `header.Root` : 状态根保持原状
2. `header.UncleHash` : 为nil
3. `types.NewBlock(header, txs, nil, receipts)` : 封装并返回最终的block

```go
// Finalize implements consensus.Engine, ensuring no uncles are set, nor block
// rewards given, and returns the final block.
func (c *Clique) Finalize(chain consensus.ChainReader, header *types.Header, state *state.StateDB, txs []*types.Transaction, uncles []*types.Header, receipts []*types.Receipt) (*types.Block, error) {
	// No block rewards in PoA, so the state remains as is and uncles are dropped
	header.Root = state.IntermediateRoot(chain.Config().IsEIP158(header.Number))
	header.UncleHash = types.CalcUncleHash(nil)

	// Assemble and return the final block for sealing
	return types.NewBlock(header, txs, nil, receipts), nil
}
```

#### API.Propose(addr, auth)

添加一个proposal: 调用者对addr的投票, auth表示加入还是踢出

```go
// Propose injects a new authorization proposal that the signer will attempt to
// push through.
func (api *API) Propose(address common.Address, auth bool) {
	api.clique.lock.Lock()
	defer api.clique.lock.Unlock()

	api.clique.proposals[address] = auth
}
```

#### API.Discard(addr)

删除一个proposal

```go
// Discard drops a currently running proposal, stopping the signer from casting
// further votes (either for or against).
func (api *API) Discard(address common.Address) {
	api.clique.lock.Lock()
	defer api.clique.lock.Unlock()

	delete(api.clique.proposals, address)
}
```

