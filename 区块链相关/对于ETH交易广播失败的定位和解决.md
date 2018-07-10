





---

tx_pool

1. pending : 所有的，可以进行打包的交易
2. queue : 队列中，但是还不可以打包的交易
3. all ： 所有的交易，包括pending和queue
4. priced ：一个最大堆，根据给的gas price进行的排序



## 日志分析



### 交易进入交易池

1. 检查交易交易池中是否包含这个交易，如果有抛出异常，结束

2. 检查交易是否是有效，如果有无效，抛出异常，结束

3. 检查 queue 和 pool是否已经满了，如果满了

   1. 如果是从其他节点中转过来的交易，给的price 过低，抛出异常，整个过程结束
   2. 如果给的不是从其他节点来的交易，按照price给新的交易腾地方

4. 如果查 pending队列中，是否存在同一个nonce的交易，如果存在，直接从pending队列中替换掉，然后返回

5. 交易进入队列，进入队列时，它会根据 交易 的nonce值查询时候存在同一个nonce的交易存在于队列中，如果存在，它会将其从队列中移除。**这个过程，没有日志打印** ，也就是说如果存在nonce冲突，交易被移除出queue，我们是没办法根据日志定位的。另外，如果旧的交易给的gas price 更高，那么新的交易将不会被添加进队列，整个过程会抛出异常失败

6. 由于是新的交易，并且，它没有覆盖以前的一笔交易的话，它会立即进入 pending。

   

以上就是整个交易进入交易池的流程，现在我们来分析下nonce冲突的两笔交易。

https://confluence.primeledger.cn/pages/viewpage.action?pageId=18120727 我们以哪个ID 为73193的交易为例子，当时它卡到10的交易Hash 为`0xf71454585220c0c6182efc4b171dbf790b51571e8df3e9990901087930edeaf5` 通过这个Hash。我们定位到其进入交易池的日志，如图一。 nonce冲突的另一笔交易的Hash 为`0x9246e363cfb9d66af0bd528348750302af04b829ac1755faeb2f358fc4b9c2e8`它们的nonce值都为1560。我们可以根据它的前6位去查询日志。如下图



![1531120090872](C:\Users\walke\AppData\Local\Temp\1531120090872.png)

![1531108256631](C:\Users\walke\AppData\Local\Temp\1531108256631.png)



可以看出，有效的第二笔交易，在进入交易池时，在第上面的4步，即pending队列里面查到了找到了nonce一致的一笔交易，然后将其替换掉了。代码位置在 tx_pool.go 文件中的 `(pool *TxPool) add(tx *types.Transaction, local bool) (bool, error) ` 方法,我把对应的代码片段贴上来

```go
// If the transaction is replacing an already pending one, do directly
	from, _ := types.Sender(pool.signer, tx) // already validated
	if list := pool.pending[from]; list != nil && list.Overlaps(tx) {
		// Nonce already pending, check if required price bump is met
		inserted, old := list.Add(tx, pool.config.PriceBump)
		if !inserted {
			pendingDiscardCounter.Inc(1)
			return false, ErrReplaceUnderpriced
		}
		// New transaction is better, replace old one
		if old != nil {
			pool.all.Remove(old.Hash())
			pool.priced.Removed()
			pendingReplaceCounter.Inc(1)
		}
		pool.all.Add(tx)
		pool.priced.Put(tx)
		pool.journalTx(from, tx)

		log.Trace("Pooled new executable transaction", "hash", hash, "from", from, "to", tx.To())

		// We've directly injected a replacement transaction, notify subsystems
		go pool.txFeed.Send(NewTxsEvent{types.Transactions{tx}})

		return old != nil, nil
	}
```



**分析了另外的几个交易，基本上都是这样的问题，即前一个 同nonce值 还在pending阶段，又来了一个同nonce的交易，这样前一个nonce值的交易就被从pending的队列中移除掉了，后面这笔交易被广播到整个网络，这样，前一个交易就会被整个网络移除**   



**现在问题基本在于，为什么他们会获取到同样的nonce值,下面来分析这个原因**



#### nonce获取分析

我们在拿地址对应nonce值是通过查询其pending来定位的。直接去看其源码，因为代码过多，我只摘取部分比较重要的来说

```
('block_height:', 5908684, '    lastest_nonce:', 1536, '    pending_nonce:', 1540, '    time:', '2018-07-05 15:31:21')

('block_height:', 5908684, '    lastest_nonce:', 1536, '    pending_nonce:', 1540, '    time:', '2018-07-05 15:31:41')

('block_height:', 5908685, '    lastest_nonce:', 1536, '    pending_nonce:', 1536, '    time:', '2018-07-05 15:32:01')

('block_height:', 5908688, '    lastest_nonce:', 1536, '    pending_nonce:', 1540, '    time:', '2018-07-05 15:32:21')

('block_height:', 5908689, '    lastest_nonce:', 1536, '    pending_nonce:', 1540, '    time:', '2018-07-05 15:32:41')

('block_height:', 5908692, '    lastest_nonce:', 1536, '    pending_nonce:', 1540, '    time:', '2018-07-05 15:33:01')

('block_height:', 5908693, '    lastest_nonce:', 1536, '    pending_nonce:', 1540, '    time:', '2018-07-05 15:33:21')

('block_height:', 5908695, '    lastest_nonce:', 1536, '    pending_nonce:', 1541, '    time:', '2018-07-05 15:33:42')

('block_height:', 5908697, '    lastest_nonce:', 1536, '    pending_nonce:', 1536, '    time:', '2018-07-05 15:34:02')

('block_height:', 5908699, '    lastest_nonce:', 1536, '    pending_nonce:', 1541, '    time:', '2018-07-05 15:34:22')

('block_height:', 5908699, '    lastest_nonce:', 1536, '    pending_nonce:', 1541, '    time:', '2018-07-05 15:34:42')

('block_height:', 5908702, '    lastest_nonce:', 1536, '    pending_nonce:', 1536, '    time:', '2018-07-05 15:35:02')

('block_height:', 5908703, '    lastest_nonce:', 1536, '    pending_nonce:', 1541, '    time:', '2018-07-05 15:35:21')

```



1. 通过pending 来定位，可以看到，它是去miner模块找正在pending block

```go
func (b *EthAPIBackend) StateAndHeaderByNumber(ctx context.Context, blockNr rpc.BlockNumber) (*state.StateDB, *types.Header, error) {
	// Pending state is only known by the miner
	if blockNr == rpc.PendingBlockNumber {
		block, state := b.eth.miner.Pending()
		return state, block.Header(), nil
	}
		// Otherwise resolve the block number and return its state
	header, err := b.HeaderByNumber(ctx, blockNr)
	if header == nil || err != nil {
		return nil, nil, err
	}
	stateDb, err := b.eth.BlockChain().StateAt(header.Root)
	return stateDb, header, err
```

2. 如果全节点没有在挖矿，节点的snapshot block 会在每次收到 tx 事件时对 snapshot block进行更新

   代码出处 ： `worker.go`  263行   

```go
            // 当有新的交易进来时，tx_pool会通知
		case ev := <-self.txsCh:
			// Apply transactions to the pending state if we're not mining.
			//
			// Note all transactions received may not be continuous with transactions
			// already included in the current mining block. These transactions will
			// be automatically eliminated.
			if atomic.LoadInt32(&self.mining) == 0 {
				self.currentMu.Lock()
				txs := make(map[common.Address]types.Transactions)
				for _, tx := range ev.Txs {
                      //acc 为tx 的from 地址
					acc, _ := types.Sender(self.current.signer, tx) 
					txs[acc] = append(txs[acc], tx)
				}
				txset := types.NewTransactionsByPriceAndNonce(self.current.signer, txs)
				self.current.commitTransactions(self.mux, txset, self.chain, self.coinbase)
				self.updateSnapshot()
				self.currentMu.Unlock()
			} else {
					......我们不挖矿省略。。。。
			}
```

**先说结论，nonce值并不是根据tx_pool中的地址对应的交易来给的，也不是snapshot block，而是snapshot state** 这是一个DB对象，所以，我们的nonce其实是通过DB来获取的。

----

最终！我们发现了这块日志 ： 

![1531205009081](C:\Users\walke\AppData\Local\Temp\1531205009081.png)



**对这块之日进行筛选，会发现，这些日志出现的时间，恰好和我们提笔单出问题的时间吻合！** 

下面的代码片段出自于 worker.go 中的 `commitTransactions` 方法，即tx_pool 收到交易后进行 snapshot 快照时出现的！ 

```

err, logs := env.commitTransaction(tx, bc, coinbase, env.gasPool)
		switch err {
		case core.ErrGasLimitReached:
			// Pop the current out-of-gas transaction without shifting in the next from the account
			log.Trace("Gas limit exceeded for current block", "sender", from)
			txs.Pop()
```

异常的最终来源来自于 : 

```

// SubGas deducts the given amount from the pool if enough gas is
// available and returns an error otherwise.
func (gp *GasPool) SubGas(amount uint64) error {
	if uint64(*gp) < amount {
		return ErrGasLimitReached
	}
	*(*uint64)(gp) -= amount
	return nil
}
```

异常抛出的原因在于 我们的全节点中的Gas Pool 中没有足够的Gas去执行这个transaction。这也可以解释为什么我们的nonce忽高忽低的原因。因为我们的Gas pool 是动态变化的。 

