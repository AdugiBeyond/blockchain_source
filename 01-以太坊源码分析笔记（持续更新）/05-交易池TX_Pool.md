# 交易池作用

交易池中包含本地创建的交易以及网络上搜集到的交易，

主要维护两个交易队列

## 1. pending

表示已经准备好打包的交易

## 2. queue

未来打包的交易，两个队列动态调整，打包好的交易会从pending中删除，然后将queue中的交易转换到pending中。

```go
// TxPool contains all currently known transactions. Transactions
// enter the pool when they are received from the network or submitted
// locally. They exit the pool when they are included in the blockchain.
//
// The pool separates processable transactions (which can be applied to the
// current state) and future transactions. Transactions move between those
// two states over time as they are received and processed.
type TxPool struct {
	config       TxPoolConfig
	chainconfig  *params.ChainConfig
	chain        blockChain
	gasPrice     *big.Int
	txFeed       event.Feed
	scope        event.SubscriptionScope
	chainHeadCh  chan ChainHeadEvent
	chainHeadSub event.Subscription
	signer       types.Signer
	mu           sync.RWMutex

	currentState  *state.StateDB      // Current state in the blockchain head
	pendingState  *state.ManagedState // Pending state tracking virtual nonces
	currentMaxGas uint64              // Current gas limit for transaction caps

	locals  *accountSet // Set of local transaction to exempt from eviction rules
	journal *txJournal  // Journal of local transaction to back up to disk

	pending map[common.Address]*txList   // All currently processable transactions
	queue   map[common.Address]*txList   // Queued but non-processable transactions
	beats   map[common.Address]time.Time // Last heartbeat from each known account
	all     *txLookup                    // All transactions to allow lookups
	priced  *txPricedList                // All transactions sorted by price

	wg sync.WaitGroup // for shutdown sync

	homestead bool
}
```



# 概述

txpool主要用来存放当前提交的等待写入区块的交易，有远端和本地的。

txpool里面的交易分为两种，

1. 提交但是还不能执行的，放在queue里面等待能够执行(==比如说nonce太高)==。
2. 等待执行的，放在pending里面等待执行。

从txpool的测试案例来看，txpool主要功能有下面几点。

1. 交易验证的功能，包括余额不足，Gas不足，Nonce太低, value值是合法的，不能为负数。
2. 能够缓存Nonce比当前本地账号状态高的交易。 存放在queue字段。 如果是能够执行的交易存放在pending字段
3. 相同用户的相同Nonce的交易只会保留一个GasPrice最大的那个。 其他的插入不成功。
4. 如果账号没有钱了，那么queue和pending中对应账号的交易会被删除。
5. 如果账号的余额小于一些交易的额度，那么对应的交易会被删除，同时有效的交易会从pending移动到queue里面。防止被广播。
6. txPool支持一些限制PriceLimit(remove的最低GasPrice限制)，PriceBump(替换相同Nonce的交易的价格的百分比) AccountSlots(每个账户的pending的槽位的最小值) GlobalSlots(全局pending队列的最大值)AccountQueue(每个账户的queueing的槽位的最小值) GlobalQueue(全局queueing的最大值) Lifetime(在queue队列的最长等待时间)
7. 有限的资源情况下按照GasPrice的优先级进行替换。
8. 本地的交易会使用journal的功能存放在磁盘上，重启之后会重新导入。 远程的交易不会。

