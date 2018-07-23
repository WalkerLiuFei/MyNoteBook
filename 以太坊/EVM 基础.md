# EVM 基础

EVM的基本,EVM是一个智能合约的一个独立的运行系统，它不需要连接网络或者访问文件系统，甚至，它也不允许访问其他的智能合约

## Account

两种类型的Account 共享同一个地址

1. **External accounts** that are controlled by public-private key pairs 
2. **contract accounts** which are controlled by the code stored together with the account. 

Every account has a persistent key-value store mapping 256-bit words to 256-bit words called **storage**. 

Furthermore, every account has a **balance** in Ether (in “Wei” to be exact) which can be modified by sending transactions that include Ether.  

## Transactions

交易可以包含Binary data（payload） and Ether

如果目标账户包含 Code，那个Code会将交易中的Payload 作为输入来执行那些Code

如果交易的目标地址是一个zero-account ，这笔交易会创建一个新的合约，**合约的地址是从 交易发送方和交易的nonce值来生成的。这个创作合约的交易做以bytecode的形式被EVM接收并执行**

 The output of this execution is permanently stored as the code of the contract. 这就意味着，当你要新建一个合约是，你不需要发送合约的源码，but in fact code that returns that code when executed. 

While a contract is being created, its code is still empty. Because of that, you should not call back into the contract under construction until its constructor has finished executing.  

## storage ，memory and stack

每个account有一个持久化的空间，是一个256-bit to 256 -bit的键值对。



The second memory area is called **memory**, of which a contract obtains a freshly cleared instance for each message call 

Memory 的访问的代价是以2次方来扩张的。



The EVM is not a register machine but a stack machine, so all computations are performed on an area called the **stack**. It has a maximum size of 1024 elements and contains words of 256 bits 

### Instruction Set

The instruction set of the EVM is kept minimal in order to avoid incorrect implementations which could cause consensus problems. 

所有的指令动作都是最基本的数据类型，也就是256位宽度的。



### Message Calls

合约可以通过messag的方式调用或者发送以太币到其他非智能合约的地址。

As already said, the called contract (which can be the same as the caller) will receive a freshly cleared instance of memory and has access to the call payload - which will be provided in a separate area called the **calldata**. After it has finished execution, it can return data which will be stored at a location in the caller’s memory preallocated by the caller. 