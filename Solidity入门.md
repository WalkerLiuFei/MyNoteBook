# Solidity入门

[intell-solidity插件](https://solidity.readthedocs.io/en/latest/solidity-in-depth.html)

[官方文档reference](https://solidity.readthedocs.io/en/latest/introduction-to-smart-contracts.html#)

## 安装Truffle

[Truffle 官方开发文档](http://truffleframework.com/docs/)



**注意： 在windows 环境中，可以使用git bash环境或者其他linux命令行环境**

```bash
 mkdir myproject
 cd myproject
 truffle init 
 
```

新建完成后，有四个目录，分别为：

- `contracts/`: Solidity 智能合约文件存放的目录  [Solidity contracts](http://truffleframework.com/docs/getting_started/contracts)
- `migrations/`: 智能合约部署的文件 [scriptable deployment files](http://truffleframework.com/docs/getting_started/migrations#migration-files)
- `test/`: 测试文件存放目录 [testing your application and contracts](http://truffleframework.com/docs/getting_started/testing)
- `truffle.js`: Truffle 工程的配置文件

### 安装Ganache

Ganache是一个模拟以太网络的工具[下载地址](http://truffleframework.com/ganache/)

### 开发流程

总结下我的流程，我用intelliJ Idea作为编辑器。因为实在Windows环境下面，所以运行所有的truffle命令都要到git bash命令环境下。

1. truffle init 创建工程

2. 编写智能合约代码并放到contracts目录下

3. **注意不要删除init后生成的Migrations.sol** 智能合约文件

4. 运行`truffle compile`命令编译工程

5. 通过 `truffle  develope`命令启动一个以太坊的测试网络

6. 修改truffle.js 配置文件添加以下配置

   ```javascript
   module.exports = {
     // See <http://truffleframework.com/docs/advanced/configuration>
     // to customize your Truffle configuration!
      networks: {
            development: {
                  host: "localhost",
                  port: 9545,
                  network_id: "*" // Match any network id
            }
      }
   };

   ```

   ​

7. 运行`truffle console` 命令，开始交互

### trffle 使用

#### DEPLOYER API

**deployer.deploy **函数可以接收多个参数 `DEPLOYER.DEPLOY(CONTRACT, ARGS…, OPTIONS)`

```javascript
// 部署一个没有任何参数的合约A
deployer.deploy(A);

// Deploy a single contract with constructor arguments
deployer.deploy(A, arg1, arg2, ...);

// Don't deploy this contract if it has already been deployed
deployer.deploy(A, {overwrite: false});

// Set a gas price and from address for the deployment
deployer.deploy(A, {gas: 4612388, from: "0x...."});

// Deploy multiple contracts, some with arguments and some without.
// This is quicker than writing three `deployer.deploy()` statements as the deployer
// can perform the deployment as a single batched request.
deployer.deploy([
  [A, arg1, arg2, ...],
  B,
  [C, arg1]
]);

// External dependency example:
//
// For this example, our dependency provides an address when we're deploying to the
// live network, but not for any other networks like testing and development.
// When we're deploying to the live network we want it to use that address, but in
// testing and development we need to deploy a version of our own. Instead of writing
// a bunch of conditionals, we can simply use the `overwrite` key.
deployer.deploy(SomeDependency, {overwrite: false});
```

#### DEPLOYER.LINK(LIBRARY, DESTINATIONS)

将一个已经部署的库和一个或者多个合约绑定，以供它们使用

```javascript
// Deploy library LibA, then link LibA to contract B, then deploy B.
deployer.deploy(LibA);
deployer.link(LibA, B);
deployer.deploy(B);

// Link LibA to many contracts
deployer.link(LibA, [B, C, D]);
```

如果指定的合约没有用到绑定的lib那么deployer会自动忽略掉绑定

#### DEPLOYER.THEN(FUNCTION() {…})

deployer 其实相当于一个promise链，通过链式调用 。完成部署 。。理解下面这段小程序的含义

```
var a, b;
deployer.then(function() {
  // Create a new version of A
  return A.new();
}).then(function(instance) {
  a = instance;
  // Get the deployed instance of B
  return B.deployed():
}).then(function(instance) {
  b = instance;
  // Set the new instance of A's address on B via B's setA() function.
  return b.setA(a.address);
});
```



### Truffle Test

truffle test 支持两种模式的测试

1. Javascript :  从外部环境中测试你的合约代码，就像你的APP一样
2. solidity : 在裸露的情况下测试您的合约。