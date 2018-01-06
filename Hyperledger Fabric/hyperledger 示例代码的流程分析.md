> 通过这个文章记录下对示例脚本分析和踩到的坑

## 示例代码的流程分析
脚本` byfn.sh`帮我们封装了启动一个hyperledger fabric 服务服务启动的流程分为三个主要的命令
1. generate ： 为各个网络中的各个节点生成证书和密匙。
2. up： 启动服务，其实启动的就是docker中的几个容器组成的服务，通过docker compose启动。启动后你可以通过滚动日志查看交易
3. down：  关闭容器，移除加密材料和四个配置信息，从Docker仓库删除chaincode镜像.
### generate过程

`cryptogen generate --config=./crypto-config.yaml` 这是第一个命令。

`Cryptogen`消费一个包含网络拓扑的`crypto-config.yaml`，并允许我们为组织和属于这些组织的组件生成一组证书和密钥。 加密生成器 `cryptogen generate --config=./crypto-config.yaml`  运行完这个命令后，生成的证书和密匙被保存到名为crypto-config的文件夹中。

在这个示例fabric网络中我们一共两个组织，peer和order.网络实体的命名约定如下：”{{.Hostname}}.{{.Domain}}”。







### 配置交易生成器

`configtxgen tool`用于创建4个配置工作： order的genesis block(创世区块), channel的`channel configuration transaction`, *以及两个`anchor peer transactions`一个对应一个Peer组织。

channel transaction文件在Channel创建的时候广播给Order。

`anchor peer transactions`，正如名称所示，指定了每个组织在此channel上的[`Anchor peer`](http://hyperledger-fabric.readthedocs.io/en/latest/glossary.html#anchor-peer)。




## 踩到的坑
1. docker pull 镜像时 连不上docker hub。问题同 [issue 1317](https://github.com/docker/for-mac/issues/1317),通过手动修改docker 容器 通用配置的DNS，将其改成 8.8.8.8.问题解决cpn
2. api verison 过旧 ，通过 docker-copose 启动服务时，提示我API version过旧 问题描述同 [issue 5103]，通过修改docker compose 下面的各个yaml 配置文件的version 版本号，version 2修改为 2.1。 问题解决
3. 我通过docker拉取的最新的镜像是 1.0.5版本的，在我做的时候 **hyperledger/fabric-tools:latest这个镜像一直pull不下来。具体未知** 然后我通过修改 `docker-compose-cli.yaml` 文件中的fabric-tools的tag 将其改成`docker 
   image ls`命令由的那个 版本，问题解决
4. 没有正确的将host的硬盘driver挂载到容器中，通过log解决，简单
5. 一开始执行 **docker  exec -it cli bash**后， 后，命令提示没有找到`cli`容器 [解决来源](https://stackoverflow.com/questions/43955722/cli-container-not-running-hyperledger-1-0-not-able-to-start-no-tls-network) 注释掉`docker-compose-cli.yaml`的`command: /bin/bash -c './scripts/script.sh ${CHANNEL_NAME} ${DELAY}; sleep $TIMEOUT'`这条代码，重启启动。问题解决！
