# ERC20标准代币转账全流程分析

这里写了一个简单的solidity的ERC转账，只有转账和Tranfer event两个函数，因为Token实现转账是否完成，就是看的ERC20标准中的Tranfer event log是否打印成功到block chain中完成的。

这里分析了EVM在执行这种合约，从开始执行 contract tranfer 函数到 Log打印的整个汇编指令的执行情况和EVM 虚拟机栈的情况。

**可能有点难理解**



```javascript
contract simple_event {
    event Transfer(address indexed _from,
        address indexed _to,
        uint256 indexed  _value);
    function transfer(address to,uint256 amount) public payable {
        amount = amount - 10;
        emit Transfer(msg.sender, to, amount);
    }
}
```

以上是solidity 源码。可以通过 `solc --asm ` 命令来输出asm 汇编指令集



------



**注意，以下栈中的数字皆为HEX格式**

```assembly

# 初始化memory pointer
PUSH1
80

PUSH1
"80 40"

MSTORE 

# 以下是 jumpi(tag_1, lt(calldatasize, 0x4))的汇编代码，含义为
# 如果我们的调用合约的输入小于4那么久结束合约的执行（tag_1 指向的汇编代码）
PUSH1   # 将常量值 0x04 push 进stack
4

CALLDATASIZE  # 参考2 
"4 44"

LT #这里对比后结果就是Fasle，也就是说，我们是带着跳转函数的输入来的
0

PUSH1  #这里 3f 也就是tag_1的指针地址
"0 3f"

JUMPI  		  #参考 3

# 以下指令的作用为，将我们的调用合约的输入，压入虚拟栈中
PUSH1 #先压入0，load 进stack的起始地址
0

CALLDATALOAD  #load 进32个字节个输入参数到栈中
a959cbb000000000000db5b467445ac2ad2a56b5297f8786b4

PUSH29        #将 一个29字节的数 0x1000000000000000000000 压入栈，这个数用来作为被除数
"a959cbb000000000000db5b467445ac2ad2a56b5297f8786b4 10000000000000000000000000000"

SWAP1         #栈顶元素与栈顶 offet 1的元素互换位置
"10000000000000000000000000000 a959cbb000000000000db5b467445ac2ad2a56b5297f8786b4"

DIV           #通过DIV的方式,取出前4个字节
a959cbb

PUSH4         #与一下，防止溢出，因为我们只要4个字节！
"a959cbb ffffffff"

AND
a959cbb

# 下面是通过PUSH EQ的方式找要跳转的函数，因为我们的可调用执行的函数只有一个也就是transfer
# 所以这里只有一次对比，如果有多个可调用的 public函数，那么就会有多次对比跳转。
DUP1		
"a959cbb a959cbb"

PUSH4
"a959cbb a959cbb a959cbb"

EQ
"a959cbb 1"


PUSH1	  # 这里压入的0x44其实是tranfer的tag_2 值
"a959cbb 1 44"

JUMPI     # 因为栈顶的倒数第二个元素为1，所以跳转到tranfer函数处
a959cbb

JUMPDEST  # 这个指令对 执行栈和内存都没有影响，只是标识evm执行已经进入到另一个函数中
a959cbb

PUSH1     # 80 指的是函数执行结束tag_3 的地址 ,也就是 stop
"a959cbb 80"

# 以下操作是为了将input的参数除开 调用函数的那前4个字节除开的参数，压入栈内
PUSH1     
"a959cbb 80 4"

DUP1
"a959cbb 80 4 4"

CALLDATASIZE
"a959cbb 80 4 4 44"

SUB
"a959cbb 80 4 40"

DUP2
"a959cbb 80 4 40 4"

ADD
"a959cbb 80 4 44"

SWAP1
"a959cbb 80 44 4"

DUP1
"a959cbb 80 44 4 4"

DUP1
"a959cbb 80 44 4 4 4"

CALLDATALOAD
"a959cbb 80 44 4 4 db5b467445ac2ad2a56b5297f8786b4311fefbc"

PUSH20
"a959cbb 80 44 4 4 db5b467445ac2ad2a56b5297f8786b4311fefbc ffffffffffffffffffffffffffffffffffffffff"

AND
"a959cbb 80 44 4 4 db5b467445ac2ad2a56b5297f8786b4311fefbc"

SWAP1
"a959cbb 80 44 4 db5b467445ac2ad2a56b5297f8786b4311fefbc 4"

PUSH1
"a959cbb 80 44 4 db5b467445ac2ad2a56b5297f8786b4311fefbc 4 20"

ADD
"a959cbb 80 44 4 db5b467445ac2ad2a56b5297f8786b4311fefbc 24"

SWAP1
"a959cbb 80 44 4 24 db5b467445ac2ad2a56b5297f8786b4311fefbc"

SWAP3
"a959cbb 80 db5b467445ac2ad2a56b5297f8786b4311fefbc 4 24 44"

SWAP2
"a959cbb 80 db5b467445ac2ad2a56b5297f8786b4311fefbc 44 24 4"

SWAP1
"a959cbb 80 db5b467445ac2ad2a56b5297f8786b4311fefbc 44 4 24"

DUP1
"a959cbb 80 db5b467445ac2ad2a56b5297f8786b4311fefbc 44 4 24 24"

CALLDATALOAD
"a959cbb 80 db5b467445ac2ad2a56b5297f8786b4311fefbc 44 4 24 3039"

SWAP1
"a959cbb 80 db5b467445ac2ad2a56b5297f8786b4311fefbc 44 4 3039 24"

PUSH1
"a959cbb 80 db5b467445ac2ad2a56b5297f8786b4311fefbc 44 4 3039 24 20"

ADD
"a959cbb 80 db5b467445ac2ad2a56b5297f8786b4311fefbc 44 4 3039 44"

SWAP1
"a959cbb 80 db5b467445ac2ad2a56b5297f8786b4311fefbc 44 4 44 3039"

SWAP3
"a959cbb 80 db5b467445ac2ad2a56b5297f8786b4311fefbc 3039 4 44 44"

SWAP2
"a959cbb 80 db5b467445ac2ad2a56b5297f8786b4311fefbc 3039 44 44 4"

SWAP1
"a959cbb 80 db5b467445ac2ad2a56b5297f8786b4311fefbc 3039 44 4 44"

POP
"a959cbb 80 db5b467445ac2ad2a56b5297f8786b4311fefbc 3039 44 4"

POP
"a959cbb 80 db5b467445ac2ad2a56b5297f8786b4311fefbc 3039 44"

POP
"a959cbb 80 db5b467445ac2ad2a56b5297f8786b4311fefbc 3039"
# ------------------------以上所有的操作只是为了将调用函数的入参压入栈内----------------------------


PUSH1    # 将82 也就是tag_4，tag_4指的是函数（tranfer）体内的所有执行指令
"a959cbb 80 db5b467445ac2ad2a56b5297f8786b4311fefbc 3039 82"

JUMP     # 进行跳转
"a959cbb 80 db5b467445ac2ad2a56b5297f8786b4311fefbc 3039"

JUMPDEST # 不做任何操作
"a959cbb 80 db5b467445ac2ad2a56b5297f8786b4311fefbc 3039"

# 以下的PUSH DUP2 SUB SWAP POP 其实就是 amount = amount - 10 的所有指令，比较简单
PUSH 
"a959cbb 80 db5b467445ac2ad2a56b5297f8786b4311fefbc 3039 a"

DUP2
"a959cbb 80 db5b467445ac2ad2a56b5297f8786b4311fefbc 3039 a 3039"

SUB
"a959cbb 80 db5b467445ac2ad2a56b5297f8786b4311fefbc 3039 302f"

SWAP
"a959cbb 80 db5b467445ac2ad2a56b5297f8786b4311fefbc 302f 3039"

POP
"a959cbb 80 db5b467445ac2ad2a56b5297f8786b4311fefbc 302f"
# ------------------------------------------------------------------------

DUP1 # 复制栈顶，也就是 amount
"a959cbb 80 db5b467445ac2ad2a56b5297f8786b4311fefbc 302f 302f"

DUP3 # 复制 栈顶第三个元素，也就是传参 target address
"a959cbb 80 db5b467445ac2ad2a56b5297f8786b4311fefbc 302f 302f db5b467445ac2ad2a56b5297f8786b4311fefbc"

PUSH20  #PUSH 进20个字节的ff，因为 入参类型为 address，我只需要20个字节
"a959cbb 80 db5b467445ac2ad2a56b5297f8786b4311fefbc 302f 302f db5b467445ac2ad2a56b5297f8786b4311fefbc ffffffffffffffffffffffffffffffffffffffff"

AND     # 接上一步，与一下排除其他不需要的字节
"a959cbb 80 db5b467445ac2ad2a56b5297f8786b4311fefbc 302f 302f db5b467445ac2ad2a56b5297f8786b4311fefbc"


# CALLER，如果你参看我们solidity的源码的话，发现EVENT第一个参数是msg.sender
# EVM在执行我们的调用合约时，CALLER也就是发起者相当于EVM的一部分，这个CALLER地址是可以拿得到的
# CALLER 也就是我们 emit event 时的第一个元素。
# 至此，所有调用 event 元素都准备好了
CALLER  
"a959cbb 80 db5b467445ac2ad2a56b5297f8786b4311fefbc 302f 302f db5b467445ac2ad2a56b5297f8786b4311fefbc 3af6fc188e43bee9d5c1a7df767389945625d2f6"


PUSH20 # 同样的20个字节，与一下保证有效的address 类型
"a959cbb 80 db5b467445ac2ad2a56b5297f8786b4311fefbc 302f 302f db5b467445ac2ad2a56b5297f8786b4311fefbc 3af6fc188e43bee9d5c1a7df767389945625d2f6 ffffffffffffffffffffffffffffffffffffffff"


AND    # 这一步后所有需要的元素都准备好了
"a959cbb 80 db5b467445ac2ad2a56b5297f8786b4311fefbc 302f 302f db5b467445ac2ad2a56b5297f8786b4311fefbc 3af6fc188e43bee9d5c1a7df767389945625d2f6"

PUSH32  #参考4
"a959cbb 80 db5b467445ac2ad2a56b5297f8786b4311fefbc 302f 302f db5b467445ac2ad2a56b5297f8786b4311fefbc 3af6fc188e43bee9d5c1a7df767389945625d2f6 ddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef"


# 以下部分，参考5
PUSH1	
"a959cbb 80 db5b467445ac2ad2a56b5297f8786b4311fefbc 302f 302f db5b467445ac2ad2a56b5297f8786b4311fefbc 3af6fc188e43bee9d5c1a7df767389945625d2f6 ddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef 40"

MLOAD 
"a959cbb 80 db5b467445ac2ad2a56b5297f8786b4311fefbc 302f 302f db5b467445ac2ad2a56b5297f8786b4311fefbc 3af6fc188e43bee9d5c1a7df767389945625d2f6 ddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef 80"

PUSH1  # 
"a959cbb 80 db5b467445ac2ad2a56b5297f8786b4311fefbc 302f 302f db5b467445ac2ad2a56b5297f8786b4311fefbc 3af6fc188e43bee9d5c1a7df767389945625d2f6 ddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef 80 40"

MLOAD
"a959cbb 80 db5b467445ac2ad2a56b5297f8786b4311fefbc 302f 302f db5b467445ac2ad2a56b5297f8786b4311fefbc 3af6fc188e43bee9d5c1a7df767389945625d2f6 ddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef 80 80"

DUP1
"a959cbb 80 db5b467445ac2ad2a56b5297f8786b4311fefbc 302f 302f db5b467445ac2ad2a56b5297f8786b4311fefbc 3af6fc188e43bee9d5c1a7df767389945625d2f6 ddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef 80 80 80"

SWAP2
"a959cbb 80 db5b467445ac2ad2a56b5297f8786b4311fefbc 302f 302f db5b467445ac2ad2a56b5297f8786b4311fefbc 3af6fc188e43bee9d5c1a7df767389945625d2f6 ddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef 80 80 80"

SUB
"a959cbb 80 db5b467445ac2ad2a56b5297f8786b4311fefbc 302f 302f db5b467445ac2ad2a56b5297f8786b4311fefbc 3af6fc188e43bee9d5c1a7df767389945625d2f6 ddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef 80 0"

SWAP1  
"a959cbb 80 db5b467445ac2ad2a56b5297f8786b4311fefbc 302f 302f db5b467445ac2ad2a56b5297f8786b4311fefbc 3af6fc188e43bee9d5c1a7df767389945625d2f6 ddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef 0 80"

# 或许你会疑问，为什么是LOG4 而不是LOG3?那是因为 topics[0] 是event hash的值
# 执行完这个指令后，可以看到，我们的栈顶少了6个元素，分别为 
# 1. 要访问的memory 起始空间， 2. memory 元素长度 3. topics[0] Event的Hash 值
# 4. Caller address ,event 函数的第一个参数  5. target address ，token发送的参数
# 6. amount ,Token的金额
LOG4 
"a959cbb 80 db5b467445ac2ad2a56b5297f8786b4311fefbc 302f"

POP 
"a959cbb 80 db5b467445ac2ad2a56b5297f8786b4311fefbc"

POP
"a959cbb 80"

JUMP
a959cbb

JUMPDEST
a959cbb

STOP
a959cbb




```

# 参考

1. **为什么要先PUSH进0x80 和0x40? **

   参考[Layout in Memory](https://solidity.readthedocs.io/en/v0.4.21/miscellaneous.html#layout-in-memory) 

   Solidity 的Memory 分为两个个部分

   1. 0 - 0x40: scratch space for hashing methods
   2. 0x40-0x80: currently allocated memory size (aka. free memory pointer)

   对于solidity 中的object，只会放到 这部分的free memory 中

   这里 mstore(0x40,0x80)又是何意哪？这个意思是先开辟出 (0x40 + 32)字节的内存区域

   然后将 0x80这个数存在 0x40 开始的内存中。这表示0x40 - 0x80 这些内存你是可以访问，并且存放object的

2. **CallDataSize的作用**

   CallDataSize 就是用来计算我们的输入的长度的，在这里，我们调用这个合约的输入就是  4个字节的函数标加两个32字节长的参数，分别是token 转向的目标地址  address 和 token 的 数量 amount。一共为68（0x44）字节长。

3. JUMP1的作用   ， jump to label if cond is nonzero。显然这里我们的调用合约的输入是大于4的.也就是栈中的第二参数是0，所以不跳转

4. `ddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef` 这是Event 函数 `Transfer(address,address,uint256)` 的Hash值，如果你在[keccak online](https://emn178.github.io/online-tools/keccak_256.html)   上面试一下就知道了。在这里，keccak就作为Event 函数的跳转目的地。这个Hash值是在编译期完成的。

5. 这部分比较难理解，首先我们需要看一下solidity 中 event 参数 关键字 indexed的作用[What does the indexed keyword do?](https://ethereum.stackexchange.com/questions/8658/what-does-the-indexed-keyword-do)  。小结一下就是，如果一个 Event函数的参数加上了indexed 前缀，那么这个参数就会被保存区块链中，且是可以追溯的。也就是说，如果你通过一个交易调用这个 event 最后这个event log跟随这个交易保存到区块链中，那么你的输入参数可以根据这个交易Hash追溯到。如果没有带 indexed参数，那么就是追溯不到的。我们可以看一下，EVM源码是怎么处理的。这里先贴一个示例代码。**可以看到我们event 最后一个参数是没有indexed关键字的**

   ```javascript
   contract simple_event {
       event Transfer(address indexed _from,
           address indexed _to,
           uint256 _value);
       function transfer(address to,uint256 amount) public payable {
           amount = amount - 10;
           emit Transfer(msg.sender, to, amount);
       }
   }
   ```

   EVM源码中对最后一个value参数是做的什么处理哪？

   ```go
   func(pc *uint64, evm *EVM, contract *Contract, memory *Memory, stack *Stack) ([]byte, error) {
       	// size 为带有 indexed关键字的参数个数
   		topics := make([]common.Hash, size)
           /**
           	没有带indexed关键的参数，会先被存入memory，这里mstart 和 msize就是这些参数在memory中
           	存储的起始位置和长度
           **/
   		mStart, mSize := stack.pop(), stack.pop()
           //将带有indexed 关键字的参数设置进topics中
   		for i := 0; i < size; i++ {
   			topics[i] = common.BigToHash(stack.pop())
   		}
   		// 将没有带有indexed 关键字的参数设置进data中
   		d := memory.Get(mStart.Int64(), mSize.Int64())
   		evm.StateDB.AddLog(&types.Log{
   			Address: contract.Address(),
   			Topics:  topics,
   			Data:    d,
   			// This is a non-consensus field, but assigned here because
   			// core/state doesn't know the current block number.
   			BlockNumber: evm.BlockNumber.Uint64(),
   		})
   
   		evm.interpreter.intPool.put(mStart, mSize)
   		return nil, nil
   	}
   ```

   可以看到带有indexed关键字的参数和没有带有indexed关键字的参数存储的地方是不一样的。其中Data部分是不可以访问的。

   回过头看我们的 汇编代码。mload(0x40) 就是要进行访问我们free memory 空间，将里面的数据load进evm 栈中。因为我们有两个输入参数，所以同样的操作我们进行了两次，且我们的event参数都是带有indexed关键字修饰的所以中间没有其他操作。

   通过对比，我们可以看一下如果 Event时间函数最后一个amount参数没有带 indexed修饰的话汇编指令执行是怎么样的。solidity源码就是上面的。

   ```assembly
   
   DUP4  # 复制 amount 
   "a959cbb 80 db5b467445ac2ad2a56b5297f8786b4311fefbc 302f db5b467445ac2ad2a56b5297f8786b4311fefbc 3af6fc188e43bee9d5c1a7df767389945625d2f6 ddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef 302f"
    
   PUSH1 # mload(0x40) 读取free memory pointer
   "a959cbb 80 db5b467445ac2ad2a56b5297f8786b4311fefbc 302f db5b467445ac2ad2a56b5297f8786b4311fefbc 3af6fc188e43bee9d5c1a7df767389945625d2f6 ddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef 302f 40"
   
   MLOAD
   "a959cbb 80 db5b467445ac2ad2a56b5297f8786b4311fefbc 302f db5b467445ac2ad2a56b5297f8786b4311fefbc 3af6fc188e43bee9d5c1a7df767389945625d2f6 ddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef 302f 80"
   
   DUP1 # 复制 free memory pointer 
   "a959cbb 80 db5b467445ac2ad2a56b5297f8786b4311fefbc 302f db5b467445ac2ad2a56b5297f8786b4311fefbc 3af6fc188e43bee9d5c1a7df767389945625d2f6 ddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef 302f 80 80"
   
   DUP3 # 将 amount 参数存储到 0x80 + 4字节的地方
   "a959cbb 80 db5b467445ac2ad2a56b5297f8786b4311fefbc 302f db5b467445ac2ad2a56b5297f8786b4311fefbc 3af6fc188e43bee9d5c1a7df767389945625d2f6 ddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef 302f 80 80 302f"
   
   DUP2
   "a959cbb 80 db5b467445ac2ad2a56b5297f8786b4311fefbc 302f db5b467445ac2ad2a56b5297f8786b4311fefbc 3af6fc188e43bee9d5c1a7df767389945625d2f6 ddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef 302f 80 80 302f 80"
   
   MSTORE # 存储0x302f 到 free memory 的0x80 - 0x a0 区域，（占用32字节）
   "a959cbb 80 db5b467445ac2ad2a56b5297f8786b4311fefbc 302f db5b467445ac2ad2a56b5297f8786b4311fefbc 3af6fc188e43bee9d5c1a7df767389945625d2f6 ddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef 302f 80 80"
   
   PUSH1 # 因为我们参数的长度是 32（0x20） 字节的
   "a959cbb 80 db5b467445ac2ad2a56b5297f8786b4311fefbc 302f db5b467445ac2ad2a56b5297f8786b4311fefbc 3af6fc188e43bee9d5c1a7df767389945625d2f6 ddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef 302f 80 80 20"
   
   ADD
   "a959cbb 80 db5b467445ac2ad2a56b5297f8786b4311fefbc 302f db5b467445ac2ad2a56b5297f8786b4311fefbc 3af6fc188e43bee9d5c1a7df767389945625d2f6 ddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef 302f 80 a0"
   
   SWAP2
   "a959cbb 80 db5b467445ac2ad2a56b5297f8786b4311fefbc 302f db5b467445ac2ad2a56b5297f8786b4311fefbc 3af6fc188e43bee9d5c1a7df767389945625d2f6 ddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef a0 80 302f"
   
   POP  # 存储完成，将 amount 踢出 evm 栈
   "a959cbb 80 db5b467445ac2ad2a56b5297f8786b4311fefbc 302f db5b467445ac2ad2a56b5297f8786b4311fefbc 3af6fc188e43bee9d5c1a7df767389945625d2f6 ddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef a0 80"
   
   POP  # 将 0x80 踢出，开始下一个参数，也就是 target address 
   "a959cbb 80 db5b467445ac2ad2a56b5297f8786b4311fefbc 302f db5b467445ac2ad2a56b5297f8786b4311fefbc 3af6fc188e43bee9d5c1a7df767389945625d2f6 ddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef a0"
   
   # 这下面的跟我们主要分析的就一样了，因为target address的参数是带有 indexed关键字的，
   # 所以没有存储到 free memroy 的必要
   PUSH1 
   "a959cbb 80 db5b467445ac2ad2a56b5297f8786b4311fefbc 302f db5b467445ac2ad2a56b5297f8786b4311fefbc 3af6fc188e43bee9d5c1a7df767389945625d2f6 ddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef a0 40"
   
   MLOAD
   "a959cbb 80 db5b467445ac2ad2a56b5297f8786b4311fefbc 302f db5b467445ac2ad2a56b5297f8786b4311fefbc 3af6fc188e43bee9d5c1a7df767389945625d2f6 ddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef a0 80"
   
   DUP1
   "a959cbb 80 db5b467445ac2ad2a56b5297f8786b4311fefbc 302f db5b467445ac2ad2a56b5297f8786b4311fefbc 3af6fc188e43bee9d5c1a7df767389945625d2f6 ddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef a0 80 80"
   
   SWAP2
   "a959cbb 80 db5b467445ac2ad2a56b5297f8786b4311fefbc 302f db5b467445ac2ad2a56b5297f8786b4311fefbc 3af6fc188e43bee9d5c1a7df767389945625d2f6 ddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef 80 80 a0"
   
   SUB
   "a959cbb 80 db5b467445ac2ad2a56b5297f8786b4311fefbc 302f db5b467445ac2ad2a56b5297f8786b4311fefbc 3af6fc188e43bee9d5c1a7df767389945625d2f6 ddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef 80 20"
   
   SWAP1
   "a959cbb 80 db5b467445ac2ad2a56b5297f8786b4311fefbc 302f db5b467445ac2ad2a56b5297f8786b4311fefbc 3af6fc188e43bee9d5c1a7df767389945625d2f6 ddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef 20 80"
   # 这里也是和我们的区别，我们需要打印4个参数，这里只要打印3个,（最后一个amount不带 indexed）
   LOG3
   "a959cbb 80 db5b467445ac2ad2a56b5297f8786b4311fefbc 302f"
   ```

   通过上面可以到看。在执行Log指令时这里所有栈顶元素是 [0x20 0x80]，而我们主分析的 栈顶元素是 [0x0 , 0x80]。

   通过EVM处理Log的源码可以明显得知它们的区别

   

   

   

   

   

   

   