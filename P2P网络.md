# P2P内网穿透原理-UDP

[参考](http://www.voidcn.com/article/p-yfadhvnw-br.html)

## NAT

NAT（Network Address Translation,网络地址转换），当在专用网内部的一些主机本来已经分配到了本地IP地址（即仅在本专用网内使用的专用地址），但现在又想和因特网上的主机通信（并不需要加密）时，可使用NAT方法。

NAT的实现依赖于NAT路由器，它至少需要一个有效的外部全球IP地址。

### NAT功能

NAT不仅能解决了lP地址不足的问题，而且还能够有效地避免来自网络外部的攻击，隐藏并保护网络内部的计算机。

1.宽带分享：这是 NAT 主机的最大功能。

2.安全防护：NAT 之内的 PC 联机到 Internet 上面时，他所显示的 IP 是 NAT 主机的公共 IP，所以 Client 端的 PC 当然就具有一定程度的安全了，外界在进行 portscan（端口扫描） 的时候，就侦测不到源Client 端的 PC 。

### NAT实现方式

+ 静态转换
+ 动态转换
+ **端口多路复用**：是指改变外出数据包的[源端口](https://baike.baidu.com/item/%E6%BA%90%E7%AB%AF%E5%8F%A3)并进行端口转换，即端口[地址转换](https://baike.baidu.com/item/%E5%9C%B0%E5%9D%80%E8%BD%AC%E6%8D%A2)（PAT，Port Address Translation).采用端口多路复用方式。内部网络的所有[主机](https://baike.baidu.com/item/%E4%B8%BB%E6%9C%BA)均可共享一个合法外部IP地址实现对Internet的访问，从而可以最大限度地节约IP地址资源。同时，又可隐藏网络内部的所有[主机](https://baike.baidu.com/item/%E4%B8%BB%E6%9C%BA)，有效避免来自internet的攻击。因此，目前网络中应用最多的就是端口多路复用方式。
+ ALG （**Application Level Gateway ）： 利用应用层的协议报文中携带地址信息。他能对应用层进行网络通信时进行相应的NAT转换。**例如：对于FTP协议的PORT/PASV命令、DNS协议的 "A" 和 "PTR" queries命令和部分ICMP消息类型等都需要相应的ALG来支持**

### NAPT

NAPT（Network Address Port Translation），即网络端口地址转换，也**就是<内部地址+内部端口>与<外部地址+外部端口>之间的转换  ** 。 NAPT普遍用于[接入设备](https://baike.baidu.com/item/%E6%8E%A5%E5%85%A5%E8%AE%BE%E5%A4%87)中，它可以将中小型的网络隐藏在一个合法的IP地址后面。NAPT也被称为“多对一”的NAT，或者叫PAT（Port Address Translations，端口[地址转换](https://baike.baidu.com/item/%E5%9C%B0%E5%9D%80%E8%BD%AC%E6%8D%A2)）、地址超载（address overloading）。

**NAPT与[动态地址](https://baike.baidu.com/item/%E5%8A%A8%E6%80%81%E5%9C%B0%E5%9D%80)NAT不同，它将内部连接映射到外部网络中的一个单独的IP地址上** ， 同时在该地址上加上一个由NAT设备选定的TCP[端口号](https://baike.baidu.com/item/%E7%AB%AF%E5%8F%A3%E5%8F%B7)。NAPT算得上是一种较流行的NAT变体，通过转换TCP或UDP协议[端口号](https://baike.baidu.com/item/%E7%AB%AF%E5%8F%A3%E5%8F%B7)以及地址来提供并发性。除了一对源和目的IP地址以外，这个表还包括一对源和目的协议[端口号](https://baike.baidu.com/item/%E7%AB%AF%E5%8F%A3%E5%8F%B7)，以及NAT盒使用的一个协议端口号。

NAPT的主要优势在于，能够使用一个全球有效IP地址获得通用性。主要缺点在于其通信仅限于TCP或UDP。当所有通信都采用TCP或UDP，NAPT允许一台内部计算机访问多台外部计算机，并允许多台内部[主机](https://baike.baidu.com/item/%E4%B8%BB%E6%9C%BA)访问同一台外部计算机，相互之间不会发生冲突。





## NAT工作原理

借助于NAT，私有（保留）地址的"内部"网络通过[路由器](https://baike.baidu.com/item/%E8%B7%AF%E7%94%B1%E5%99%A8)发送[数据包](https://baike.baidu.com/item/%E6%95%B0%E6%8D%AE%E5%8C%85)时，[私有地址](https://baike.baidu.com/item/%E7%A7%81%E6%9C%89%E5%9C%B0%E5%9D%80)被转换成合法的IP地址，一个局域网只需使用少量IP地址（甚至是1个）即可实现私有地址网络内所有计算机与Internet的通信需求。

[![Nat-工作流程1](https://gss0.bdstatic.com/-4o3dSag_xI4khGkpoWK1HF6hhy/baike/c0%3Dbaike72%2C5%2C5%2C72%2C24/sign=ae188f721e178a82da3177f2976a18e8/0dd7912397dda144810a190cb2b7d0a20cf48608.jpg)](https://baike.baidu.com/pic/nat/320024/0/0dd7912397dda144810a190cb2b7d0a20cf48608?fr=lemma&ct=single)		Nat-工作流程1

①如右图这个 client（终端） 的 gateway （网关）设定为 NAT 主机，所以当要连上 Internet 的时候，该封包就会被送到 NAT 主机，这个时候的封包 Header 之 source IP（源IP） 为 192.168.1.100 ；

②而透过这个 NAT 主机，它会将 client 的对外联机封包的 source IP ( 192.168.1.100 ) 伪装成 ppp0 ( 假设为拨接情况 )这个接口所具有的公共 IP ，因为是公共 IP 了，所以这个封包就可以连上 Internet 了，同时 NAT 主机并且会记忆这个联机的封包是由哪一个 ( 192.168.1.100 ) client 端传送来的；

[![Nat工作流程2](https://gss2.bdstatic.com/-fo3dSag_xI4khGkpoWK1HF6hhy/baike/c0%3Dbaike72%2C5%2C5%2C72%2C24/sign=9eee1eb0bc315c60579863bdecd8a076/8b82b9014a90f6030d9d2ee23912b31bb051ed62.jpg)](https://baike.baidu.com/pic/nat/320024/0/8b82b9014a90f6030d9d2ee23912b31bb051ed62?fr=lemma&ct=single)Nat工作流程2

③由 Internet 传送回来的[封包](https://baike.baidu.com/item/%E5%B0%81%E5%8C%85)，当然由 NAT[主机](https://baike.baidu.com/item/%E4%B8%BB%E6%9C%BA)来接收了，这个时候， NAT 主机会去查询原本记录的[路由](https://baike.baidu.com/item/%E8%B7%AF%E7%94%B1)信息，并将目标 IP 由 ppp0 上面的公共 IP 改回原来的 192.168.1.100 ；

④最后则由 NAT 主机将该封包传送给原先发送封包的 Client [2][ ]() 。