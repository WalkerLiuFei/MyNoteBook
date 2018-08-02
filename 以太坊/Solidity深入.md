会被认为会更改区块链状态的行为

1. Writing to state variables.
2. [Emitting events](https://solidity.readthedocs.io/en/v0.4.24/contracts.html#events).
3. [Creating other contracts](https://solidity.readthedocs.io/en/v0.4.24/control-structures.html#creating-contracts).
4. Using `selfdestruct`.
5. Sending Ether via calls.
6. Calling any function not marked `view` or `pure`.
7. Using low-level calls.
8. Using inline assembly that contains certain opcodes.

----



1. web3的call其实只是"读"，并不影响block chain **call 的传参是怎样的？**

   

2. transact() 其实就是发送一笔交易，例如下面的解释

   ```
     # Assume we have a Wallet contract with the following methods.
               # * Wallet.deposit()  # deposits to `msg.sender`
               # * Wallet.deposit(address to)  # deposits to the account indicated
               #   by the `to` parameter.
               # * Wallet.withdraw(address amount)
   
               >>> wallet = Wallet(address='0xDc3A9Db694BCdd55EBaE4A89B22aC6D12b3F0c24')
               # Deposit to the `web3.eth.coinbase` account.
               >>> wallet.functions.deposit().transact({'value': 12345})
               '0x5c504ed432cb51138bcf09aa5e8a410dd4a1e204ef84bfed1be16dfba1b22060'
               # Deposit to some other account using funds from `web3.eth.coinbase`.
               >>> wallet.functions.deposit(web3.eth.accounts[1]).transact({'value': 54321})
               '0xe122ba26d25a93911e241232d3ba7c76f5a6bfe9f8038b66b198977115fb1ddf'
               # Withdraw 12345 wei.
               >>> wallet.functions.withdraw(12345).transact()
               
           :param transaction: Dictionary of transaction info for web3 interface.
           Variables include ``from``, ``gas``, ``value``, ``gasPrice``, ``nonce``.
   
   ```

   对于上面的Deposit的合约函数，其实就是执行了一笔交易。最后调用的函数是 evm.go中的 `Transfer0` ，上面例子中的单位 12345 的 unit是 wei 也就是 $1/10^{18}$ ether

3. 每个明个的OP都在`jump_table.go` 和 `instructions.go` 这个文件里面，这个仔细看，你就会明白，它其实执行的就是EVM方法，另外还有`memory_table.go` 这样的文件

4. contract 的执行最后都落在了`interpreter.go` 中的`Run`方法中 ，这个方法通过上面的 jump_table 和 instructions 这种来执行对应的操作