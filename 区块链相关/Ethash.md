

# Ethash

-----

Ethash共识的原理以及实现

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

   

3. 通过这个Cache，可以计算出一个初始值为1GB大小的dataset (创世区块)，其中数据集中的每个项目仅依赖于缓存中的少量item，全节点和矿工存储这个dataset, dateset 的大小增长时线性的，注意这个Cache size必须还剩时素数。下面是计算dataset大小的代码

   ```python
   def get_full_size(block_number):
       sz = DATASET_BYTES_INIT + DATASET_BYTES_GROWTH * (block_number // EPOCH_LENGTH)
       sz -= MIX_BYTES
       while not isprime(sz / MIX_BYTES):
           sz -= 2 * MIX_BYTES
       return sz
   ```

   

4. 挖矿时，miner通过机抓取dataset中的一部分然后将他们一块Hash,计算出的Hash值只要等于指定的难度值，那么这个区块就是有效的！而轻量级钱包在验证时，你只需要通过这个16MB大小的cache(`每个epoch周期的值是一样的`)重新生成 指定部分的dataset （用来Hash的那部分dataset） ，所以，轻量级钱包验证区块只需要小部分的memory

在ETH中，每30000个block调整一个dataset的大小，所以，miner的主要工作就是需要读取这个**占大内存的dataset**生成指定的小于难度的Hash值。这就是为什么Ethhash是吃内存的，不像BTC是吃GPU的计算能力的POW共识算法



注意以下名词 请查阅 ETH黄皮书 append j  ethash

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

代码来自于ETH的Go实现 ： [go-ethereum](https://github.com/ethereum/go-ethereum) 

1. 通过高度获取一个cache 
2. 通过高度计算一个dataset 的 size
3. 

```go
// VerifySeal implements consensus.Engine, checking whether the given block satisfies
// the PoW difficulty requirements.
func (ethash *Ethash) VerifySeal(chain consensus.ChainReader, header *types.Header) error {
	// 区块高度
	number := header.Number.Uint64()
	//获取cache值，请看上面共识的内容
	cache := ethash.cache(number)
	size := datasetSize(number)
	
	digest, result := hashimotoLight(size, cache.cache, header.HashNoNonce().Bytes(), header.Nonce.Uint64())
	// Caches are unmapped in a finalizer. Ensure that the cache stays live
	// until after the call to hashimotoLight so it's not unmapped while being used.
	runtime.KeepAlive(cache)

	if !bytes.Equal(header.MixDigest[:], digest) {
		return errInvalidMixDigest
	}
	target := new(big.Int).Div(maxUint256, header.Difficulty)
	if new(big.Int).SetBytes(result).Cmp(target) > 0 {
		return errInvalidPoW
	}
	return nil
}

```

**我们需要关注下通过获取seed的代码** 

seedHash用来生成cache，一共32个字节，利用的是keccak256算法[KECCAK-256 != SHA3](https://crypto.stackexchange.com/questions/15727/what-are-the-key-differences-between-the-draft-sha-3-standard-and-the-keccak-sub)  ,每个epoch周期的seedhash 其实就是上一个seed的32位Hash值

```go

/**

**/
// seedHash is the seed to use for generating a verification cache and the mining
// dataset.
func seedHash(block uint64) []byte {
    //创世区块所在的epoch周期的seedhash 就是32个0
	seed := make([]byte, 32)
	if block < epochLength {
		return seed
	}
    //keccak256 一种比sha256跟
	keccak256 := makeHasher(sha3.NewKeccak256())
	for i := 0; i < int(block/epochLength); i++ {
		keccak256(seed, seed)
	}
	return seed
}

```

接下来就是生成cache了，由于Golang 实现过于，麻烦这里只上Python代码，更易于理解，总之就是

0. cache中的每64个字节（hashbytes）相当于是一个raw
1. 将seedHash填充到cache的第一个raw里面
2. cache中依次根据前一个raw的Hash值就是下一个raw的值
3. 利用一个RandMemoHash三次混淆cache

**RandMemoHash**的算法注释已经写到注释中

1. 选中一个srcRaw 这个raw 和一个随机的raw中的每个字节进行**异或计算**
2. 将这个step1的结果进行Hash 后存入dstRaw中 。dstRaw是从0开始的

因为上面的计算需要随机访问内存所以又称之为 Rand Memo Hash function

```go
func generateCache(dest []uint32, epoch uint64, seed []byte) {
    // Create a hasher to reuse between invocations
	keccak512 := makeHasher(sha3.NewKeccak512())

	// 其实就将seedHash重新Hash一下存储到cache里面的前64个字节
	keccak512(cache, seed)
	for offset := uint64(hashBytes); offset < size; offset += hashBytes {
		keccak512(cache[offset:], cache[offset-hashBytes:offset])
	}

   	// Use a low-round version of randmemohash
	temp := make([]byte, hashBytes)

	for i := 0; i < cacheRounds; i++ {
		for j := 0; j < rows; j++ {
			var (
				//从最后一个Row开始，然后是第0个Row
				srcOff = ((j - 1 + rows) % rows) * hashBytes
				//
				dstOff = j * hashBytes
				// 相当于生成一个随机数，相当于随机挑选一个row
				xorOff = (binary.LittleEndian.Uint32(cache[dstOff:]) % uint32(rows)) * hashBytes
			)
			bitutil.XORBytes(temp, cache[srcOff:srcOff+hashBytes], cache[xorOff:xorOff+hashBytes])
			keccak512(cache[dstOff:], temp)

			atomic.AddUint32(&progress, 1)
		}
	}
```



现在我们有了cache,就可以拿着这个cache计算出对应的dataset了，下面的方法是生成多一个Dataset中的Item，dataset的Item其实就是一个64字节的Hash值，具体生成的Item的个数就是 **item count =  size / 64**

1. 依次从 cache中生成 选取部分raw，根据这些raw生成的Hash值填充到mix里面
2. **这里的通过FNV Hash 进行分散式的Hash，并保存到mix中** 
3. 最后再进行一个 hash512一次生成 64字节Hash值并响应

```python
def calc_dataset_item(cache, i):
    n = len(cache)
    r = HASH_BYTES // WORD_BYTES
    # initialize the mix
    mix = copy.copy(cache[i % n])
    mix[0] ^= i
    mix = sha3_512(mix)
    # fnv it with a lot of random cache nodes based on i
    for j in range(DATASET_PARENTS):
        cache_index = fnv(i ^ j, mix[j % r])
        mix = map(fnv, mix, cache[cache_index % n])
    return sha3_512(mix)
```

FNV能快速hash大量数据并保持较小的冲突率，它的高度分散使它适用于hash一些非常相近的字符串，比如URL，hostname，文件名，text，IP地址等,类似的Bloom Fliter 也大量使用Fnv hash来进行信息摘要

```go
func fnv(a, b uint32) uint32 {
	return a*0x01000193 ^ b
}
```





```go
// generateDatasetItem combines data from 256 pseudorandomly selected cache nodes,
// and hashes that to compute a single dataset node.
func generateDatasetItem(cache []uint32, index uint32, keccak512 hasher) []byte {
	// Calculate the number of theoretical rows (we use one buffer nonetheless)
	rows := uint32(len(cache) / hashWords)

	// Initialize the mix
	mix := make([]byte, hashBytes)

	binary.LittleEndian.PutUint32(mix, cache[(index%rows)*hashWords]^index)
    //hashwords 值为16
	for i := 1; i < hashWords; i++ {
		binary.LittleEndian.PutUint32(mix[i*4:], cache[(index%rows)*hashWords+uint32(i)])
	}
	keccak512(mix, mix)

	// Convert the mix to uint32s to avoid constant bit shifting
	intMix := make([]uint32, hashWords)
	for i := 0; i < len(intMix); i++ {
		intMix[i] = binary.LittleEndian.Uint32(mix[i*4:])
	}
	// fnv it with a lot of random cache nodes based on index
	for i := uint32(0); i < datasetParents; i++ {
		parent := fnv(index^i, intMix[i%16]) % rows
		fnvHash(intMix, cache[parent*hashWords:])
	}
	// Flatten the uint32 mix into a binary one and return
	for i, val := range intMix {
		binary.LittleEndian.PutUint32(mix[i*4:], val)
	}
	keccak512(mix, mix)
	return mix
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

