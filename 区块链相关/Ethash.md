# 安全方面的一些思考

1. 在接入BTX系列的代币时，如果BTX接收充币操作，那么如果set 了lock time ，那么这笔UTXO相当于时没办法归集的，这个应该时怎么考虑？ 是在接收充币时拒绝？（那么这个充币可能就会被自动忽略），如果这个存入UTXO table，如果在lock time之前进行归集操作，那么这个归集的交易就会被全节点给拒绝掉。



-----

ETP 区块的难度校验

ETP的区块难度校验和ETH的一致，**ETH的ethash算法大致的流程如下：**

1. 有一个seed(种子数)，可以通过扫描每个知道目前这个区块的Header 来获取

2. 拿到这个seed后，通过伪随机数算法计算出一个动态大小的Cache（初始值为16MB），light client是需要保存这个Cache的。这个cache用来校验区块的有效性,注意，这个cache的大小必须是一个素数，下面是计算cache的伪代码

   ```python
   def get_cache_size(block_number):
       sz = CACHE_BYTES_INIT + CACHE_BYTES_GROWTH * (block_number // EPOCH_LENGTH)
       sz -= HASH_BYTES
       while not isprime(sz / HASH_BYTES):
           sz -= 2 * HASH_BYTES
       return sz
   ```

   

3. 通过这个Cache，可以计算出一个初始值为1GB大小的dataset (创世区块)，其中数据集中的每个项目仅依赖于缓存中的少量item，全节点和矿工存储这个dataset, dateset 的大小增长时线性的，注意这个Cache size必须还剩时素数。下面是计算dataset的伪代码

   ```python
   def get_full_size(block_number):
       sz = DATASET_BYTES_INIT + DATASET_BYTES_GROWTH * (block_number // EPOCH_LENGTH)
       sz -= MIX_BYTES
       while not isprime(sz / MIX_BYTES):
           sz -= 2 * MIX_BYTES
       return sz
   ```

   

4. 挖矿时，miner通过机抓取dataset中的一部分然后将他们一块Hash,在验证时，你只需要通过这个16MB大小的Cache重新生成 指定部分的dataset （用来Hash的那部分dataset） ，所以，轻量级钱包验证区块只需要小部分的memory

在ETH中，每30000个block调整一个dataset的大小，所以，miner的主要工作就是需要读取这个**占大内存的dataset**生成指定的小于难度的Hash值。这就是为什么Ethhash是吃内存的，不像BTC是吃GPU的计算能力的



注意以下名词 请查阅 ETH黄皮书 Append 	j  ethash

```
WORD_BYTES = 4                    # bytes in word
DATASET_BYTES_INIT = 2**30        # bytes in dataset at genesis
DATASET_BYTES_GROWTH = 2**23      # dataset growth per epoch
CACHE_BYTES_INIT = 2**24          # bytes in cache at genesis
CACHE_BYTES_GROWTH = 2**17        # cache growth per epoch
CACHE_MULTIPLIER=1024             # Size of the DAG relative to the cache
EPOCH_LENGTH = 30000              # blocks per epoch
MIX_BYTES = 128                   # width of mix
HASH_BYTES = 64                   # hash length in bytes
DATASET_PARENTS = 256             # number of parents of each dataset element
CACHE_ROUNDS = 3                  # number of rounds in cache production
ACCESSES = 64                     # number of accesses in hashimoto loop

```





**在这里我只关注，区块的有效性验证，也就是工作难度的验证** 

代码来自于ETH的C++实现 ： [cpp-ethereum](https://github.com/ethereum/cpp-ethereum) 

```c++

/**
区块验证
header 为当前区块的 header
_parent 为上一个区块的header 
**/
bool MinerAux::verifySeal(libbitcoin::chain::header& _header, libbitcoin::chain::header& _parent)
{
    Result result;
    //计算出这个seed
    h256 seedHash = HeaderAux::seedHash(_header);
    //计算出header的hash值
    h256 headerHash  = HeaderAux::hashHead(_header);
    Nonce nonce = (Nonce)_header.nonce;
    if( _header.bits != HeaderAux::calculateDifficulty(_header, _parent))
    {
        log::error(LOG_MINER) << _header.number<<" block , verify diffculty failed\n";
        return false;
    }
    DEV_GUARDED(get()->x_fulls)
    if (FullType dag = get()->m_fulls[seedHash].lock())
    {
        result = dag->compute(headerHash, nonce);

        if(result.value <= HeaderAux::boundary(_header) && (result.mixHash).hex() == ((h256)_header.mixhash).hex())
        {
            return true;
        }
        return false;
    }
    result = get()->get_light(seedHash)->compute(headerHash, nonce);
    if(result.value <= HeaderAux::boundary(_header) && (result.mixHash).hex() == ((h256)_header.mixhash).hex())
    {
        return true;
    }
    log::error(LOG_MINER) << _header.number <<" block  verified failed !\n";
    return false;
}

```

**我们需要关注下 `seedhash` 这个函数** 

```c++

/***
 计算出seed的Hash
 这个函数一共做了
 1. 拿到这个区块的高度
 2. 通过高度计算现在的阶段（每30000个高度调整一次难度）
 3. 加锁
 4. 计算seed,seed其实就是一个32位的sha256值，每一个echo 阶段的seed都是上一个echo的SHA3 256 HASH值
***/
h256 HeaderAux::seedHash(libbitcoin::chain::header& _bi)
{	
    unsigned _number = (unsigned) _bi.number;
    unsigned epoch = _number / ETHASH_EPOCH_LENGTH;
    //加锁
    Guard l(get()->x_epochs);
    if (epoch >= get()->m_seedHashes.size())
    {
        h256 ret;
        unsigned n = 0;
        if (!get()->m_seedHashes.empty())
        {
            ret = get()->m_seedHashes.back();
            n = get()->m_seedHashes.size() - 1;
        }
        get()->m_seedHashes.resize(epoch + 1);
//        cdebug << "Searching for seedHash of epoch " << epoch;
        for (; n <= epoch; ++n, ret = sha3(ret))
        {
            get()->m_seedHashes[n] = ret;
//            cdebug << "Epoch" << n << "is" << ret;
        }
    }
    return get()->m_seedHashes[epoch];
}
```

**下面是计算难度值**

有几个Constant 值需要关注“

1. adjustment ，2048分度值，每次调整难度时的分度值
2. 创世区块难度131072，也是最小难度值
3. 13 ，ETH设置每13秒打包一个区块，如果区块打包时间小于这个值，那么调大难度，如果大于这个值，调小这个难度

```c++

/**
1. 根据本次区块打包和上一个区块之间的时间差来调整下一个区块的难度
**/
u256 HeaderAux::calculateDifficulty(libbitcoin::chain::header& _bi, libbitcoin::chain::header& _parent)
{
    auto minimumDifficulty = is_testnet ? bigint(300000) : bigint(914572800);
    bigint target;

    // DO NOT MODIFY time_config in release
    static uint32_t time_config{24};
    if (!_bi.number)
    {
        throw GenesisBlockCannotBeCalculated();
    }

    if(_bi.timestamp >= _parent.timestamp + time_config)
    {
        target = _parent.bits - (_parent.bits/2048);
    } else {
        target = _parent.bits + (_parent.bits/2048);
    }
    bigint result = target;

    result = std::max<bigint>(minimumDifficulty, result);
    return u256(std::min<bigint>(result, std::numeric_limits<u256>::max()));
}

```

