# 复杂合约部署的问题探究

记录智能合约[CARD](https://etherscan.io/token/0xb07ec2c28834b889b1ce527ca0f19364cd38935c#balances) 到测试环境时会失败，此文记录解决的尝试的方法等

## 原因一 ： 没有先部署依赖的Lib

### solidity 中Lib的作用

对于较为复杂的智能合约，例如这个CARD，它会通过依赖Lib的方式来使得智能合约体积更小，这不仅仅说源码中的体积更小，也指的使得部署智能合约的交易更小。防止交易size 过大，节点不接受的情况。

![1532069100987](C:\Users\walke\AppData\Local\Temp\1532069100987.png)

上面的代码片段，是在节点接受交易时进行的验证。**如果交易的体积大于32KB，节点会拒绝这笔交易。** 对于我们部署合约来说，就相当于会失败。这个时候我们就需要Lib。然后Token contract 或者其他的contract 来依赖这些Lib。这就需要我们在部署时要先部署contract 依赖的Lib，然后再部署contract。这样contract的体积就会响应的变小。

### CARD中的Lib

通过查看[CARD](https://etherscan.io/token/0xb07ec2c28834b889b1ce527ca0f19364cd38935c#balances)  我们会发现其含有多个Lib，Contract之间也有依赖。如果我们单纯的部署其中的 `CardStackToken` 这个contract 肯定是会失败的。我们知道，contract 在部署时其实就是通过一个交易带上这个contract编译好的bytecode ，发送到一个空地址而已。如果我们编译CardStackToken 这个合约后，拷贝出来后会发现它其实不是一个正确的Hex格式字符串，它包含了类似`______CstLibrary____` 这其实表示了它在这里引用了CasLIbrary 这个库，我们要做的就是先部署` CstLibrary` 这个库部署上去，拿到地址，将上面的token contract bytecode 中的引用替换成这些library地址即可！



## 原因二 : Token contract体积过大

在解决了上面的问题以后，重新部署Token 的contract 发现报错 ： `over size ` 也就是说这个合约编译后还是超出了32K，交易体积的限制，大概52K 左右。源码就是在上面。后来阅读Geth代码的提交提交历史发现上面的代码提交于 2017年7月份。 我猜测可能是因为交易被一些版本没有更新的节点最后打包的。

我在我测试环境解决的方式就是暴力的通过修改上面的源码，来解决的。



## 原因三： 测试环境上面最新区块的Gas Limit 不足

首先普及一点，对于一笔交易，全节点是如何发现它的Gas Limit是不是足够的。同样是在上面的validateTx在节点收到交易后，会根据交易的大小给出一个 `intrinsic gas = base_gas + data_size * 68 `  ，对于CardStackToken 这笔交易，测试网络给的intrinsic gas 为 331_5012 。**测试网络区块最大值在4714690 左右，接近于创世块 Gas Limit (4712388 )**，所以这笔交易最后是可以进交易池的,另外，主网目前的区块最大为800_0000左右。然而，计算这个合约的数据可以被打包进区块，但是由于Gas给的不足。这个合约部署最后还是会失败的。CARD 在主网上面的部署的交易 [CARD create tx](https://etherscan.io/tx/0x0f67691967ccc249e6990d5974f22c4789a8bd37092a276ff0a60150178b658d)  来看，这个合约部署需要 656_3329 Gas。 

下面来解释下Block 的Gas Limit的是怎么计算的。

1. 将咱们测试环境的区块Gas limit 弄上去的方法也是有的，就是参考这个公式 ： $r = (x * 1023 + y * 1.5) / 1024 $     其中`r` 为下一个区块的Gas limit , `x` 为parent gas limit , `y` 为 parent gas used。 x 必须大于等于 y

2. 像我们这样的私有节点，如果一直挖矿，又不给它交易，那么它就会一直降低Block Gas limit。参考上面的公式，它会很快的降回 创世区块的 Gas limit。

3. 我没们现在想在测试环境上面做测试，需要快速的将Gas Limit 提高到 600万 。如果想要取得最大的r值，显然在 x = y 时会得到最优解。 那么现在我们对其进行化简后积分

   $r = \int_0^n 1024.5x/1024 $  如果我们想要的值为 r = 7000000 ,x = 4712388 最后得出需要打包的区块大概为811个。

如果我们要快速的实现区块的Gas limit 的提升，我们就用上面的那个部署源码的交易来测试案例

1. 开一个县城的挖矿线程，每个交易给一个parent gas limit (通过RPC获取的实时准的Limit)
2. 开多个线程挖矿，每个交易给一个固定的Gas limit 。

经测试2的效率更高。所以我脚本中使用了方法2 。

当区块的Gas limit的达到了这个值600万+ 。脚本部署成功！





