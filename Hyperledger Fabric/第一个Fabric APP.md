# 第一个Fabric APP

> 参考：
>
> 1. [名词解释](http://hyperledger-fabric.readthedocs.io/en/release/glossary.html)
> 2. [超级账本模型](http://www.jianshu.com/writer#/notebooks/10896012/notes/13433621)
> 3. ​

示例包括两个组织，每个组织包含有两个个人节点和一个单独的交易服务





## fabric-network sample

官方的示例脚本`byfn.sh`的功能是建立包括两个组织，每个组织包含有两个个人节点和一个单独的交易服务。脚本启动一个可以运行脚本的容器，使得个人节点可以加入到一个channel中，通过部署和执行chaincode来执行对应的交易行为



```
./byfn.sh -h
Usage:
  byfn.sh -m up|down|restart|generate [-c <channel name>] [-t <timeout>]
  byfn.sh -h|--help (print this message)
    -m <mode> - one of 'up', 'down', 'restart' or 'generate'
      - 'up' - bring up the network with docker-compose up
      - 'down' - clear the network with docker-compose down
      - 'restart' - restart the network
      - 'generate' - generate required certificates and genesis block
    -c <channel name> - config name to use (defaults to "mychannel")
    -t <timeout> - CLI timeout duration in microseconds (defaults to 10000)

Typically, one would first generate the required certificates and
genesis block, then bring up the network. e.g.:

  byfn.sh -m generate -c <channelname>
  byfn.sh -m up -c <channelname>
```

###### 首先 执行 ./byfn.sh -m generate

通过脚本我们洞察下这个脚本执行后都进行了什么操作

1.  generateCerts : cryptogen 通过配置文件`crypto-config.yaml`为 organization 和 organization 相关的配件生成相关的认证库文件。每个orgnazation的每个组件都会含有一个认证的根证书文件。这使得每个参与者在网络上都有一个专门的识别认证。在hyperledger 网络中，交易和通讯
    Transactions and communications within Hyperledger Fabric are signed by an entity’s private key (`keystore`), and then verified by means of a public key (`signcerts`).
2.  replacePrivateKey ：
3.  generateChannelArtifacts

对于fabric 网络中的一个实体来讲，它的名字的命名就是‘“{{.Hostname}}.{{.Domain}}”

```
OrdererOrgs:
#---------------------------------------------------------
# Orderer
# --------------------------------------------------------
- Name: Orderer
  Domain: example.com
  CA:
      Country: US
      Province: California
      Locality: San Francisco
  #   OrganizationalUnit: Hyperledger Fabric
  #   StreetAddress: address for org # default nil
  #   PostalCode: postalCode for org # default nil
  # ------------------------------------------------------
  # "Specs" - See PeerOrgs below for complete description
# -----------------------------------------------------
  Specs:
    - Hostname: orderer
# -------------------------------------------------------
# "PeerOrgs" - Definition of organizations managing peer nodes
# ------------------------------------------------------
PeerOrgs:
# -----------------------------------------------------
# Org1
# ----------------------------------------------------
- Name: Org1
  Domain: org1.example.com
```

看上面的示例,它在网络中的名字就是`orderer.example.com`.更详细的MSP[(Membership Service Providers (MSP))](https://hyperledger-fabric.readthedocs.io/en/release/msp.html)运行完`cryotogen`工具后，生成的认证文件可私匙被放置在`crypto-config`文件夹下



The `configtxgen tool` is used to create four configuration artifacts:具体是使用详见[configtxgen的使用](https://hyperledger-fabric.readthedocs.io/en/release/configtxgen.html)

configtxgen tool 用来生成4个配置文件

> - orderer `genesis block`, 生成创世交易块
> - channel `configuration transaction`,
> - and two `anchor peer transactions` - one for each Peer Org.



`genesis block` 创世交易块,这个配置块作为区块链上的首个块用来初始化区块链网络/[channel](https://hyperledger-fabric.readthedocs.io/en/release/glossary.html#channel).在Channel创建时，channel的交易文件被广播到交易节点。



在fabric网络上的每个节点都有一个自己的对应的锚节点(anchor peer)





