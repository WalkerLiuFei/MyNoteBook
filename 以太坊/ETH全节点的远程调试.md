# ETH全节点的远程调试

ETH全节点还是很浪费资源的，尤其是在同步下来所有区块链，如果你打算在本地进行全节点的Debug吗，有点不现实。这个文档

1. 编译，安装delve Debug工具 ： https://github.com/derekparker/delve 
2. 安装好之后，进入到 `project root/ cmd/geth` 目录下
3. 运行 `go build -gcflag='-N -l' ` 命令，golang 1.10 运行 `go build -gcflag='all -N -l` 命令,这一步完成之后，应当在目录下面发现一个名字叫 `geth`的可执行文件。
4. 然后运行命令 `dlv --listen=:6070 --headless=true --api-version=2 exec geth -- `  (最后的 `--` 用来区分 `geth`的参数的，不加这个分隔符，它会认为后面的参数还是 dlv的参数)。。。。。这个命令是，开启一个调试服务器，监听在本机的6070端口
5. 进入到Goland，这里我们用Goland进行代码跟踪调试，添加Go Remote选项，添加正确参数，开始Debug，如果console 出现![1531369657020](C:\Users\walke\AppData\Local\Temp\1531369657020.png)
6. **即为连接成功，enjoy you debugging **！

