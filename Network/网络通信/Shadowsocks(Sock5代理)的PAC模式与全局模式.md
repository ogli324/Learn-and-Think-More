# Shadowsocks(Sock5代理)的PAC模式与全局模式

src:https://blog.csdn.net/weixin_39643613/article/details/109670741



# VPN
VPN，即虚拟专用网络

虚拟专用网络的功能是：在公用网络上建立专用网络，进行加密通讯。在企业网络中有广泛应用。VPN网关通过对数据包的加密和数据包目标地址的转换实现远程访问。VPN有多种分类方式，主要是按协议进行分类。VPN可通过服务器、硬件、软件等多种方式实现。

VPN顾名思义，虚拟专网，你接入VPN就是接入了一个专有网络，那么你访问网络都是从这个专有网络的出口出去，好比你在家，你家路由器后面的网络设备是在同一个网络，而VPN则是让你的设备进入了另一个网络。同时你的IP地址也变成了由VPN分配的一个IP地址。通常是一个私网地址。你和VPN服务器之间的通信是否加密取决于连接VPN的具体方式/协议。

# Shadowsocks
Shadowsocks，即Sock5代理

采用socks协议的代理服务器就是SOCKS服务器，是一种通用的代理服务器。Socks是个电路级的底层网关，是DavidKoblas在1990年开发的，此后就一直作为Internet RFC标准的开放标准。Socks 不要求应用程序遵循特定的操作系统平台，Socks 代理与应用层代理、 HTTP 层代理不同，Socks 代理只是简单地传递数据包，而不必关心是何种应用协议（比如FTP、HTTP和NNTP请求）。所以，Socks代理比其他应用层代理要快得多。

Sock5代理服务器则是把你的网络数据请求通过一条连接你和代理服务器之间的通道，由服务器转发到目的地。你没有加入任何新的网络，只是http/socks数据经过代理服务器的转发送出，并从代理服务器接收回应。你与代理服务器通信过程不会被额外处理，如果你用https，那本身就是加密的。

# Shadowsocks全局模式与VPN的区别
VPN控制的是你电脑的整个网络，只要需要连接到互联网的流量都会经过vpn。

你的IP会被更换为VPN的IP。连接VPN只需要知道IP和账号密码。

Shadowsocks的全局模式，是设置你的系统代理的代理服务器，使你的所有http/socks数据经过代理服务器的转发送出。而只有支持socks5或者使用系统代理的软件才能使用Shadowsocks（一般的浏览器都是默认使用系统代理）。

经过代理服务器的IP会被更换。连接Shadowsocks需要知道IP、端口、账号密码和加密方式。但是Shadowsocks因为可以自由换端口，所以定期换端口就可以有效避免IP被封！

# Shadowsocks全局模式与PAC模式的区别
上面已经解释了Shadowsocks的全局模式，而PAC模式就是会在你连接网站的时候读取PAC文件里的规则，来确定你访问的网站有没有被墙，如果符合，那就会使用代理服务器连接网站，而PAC列表一般都是从GFWList更新的。GFWList定期会更新被墙的网站（不过一般挺慢的）。

简单地说，在全局模式下，所有网站默认走代理。而PAC模式是只有被墙的才会走代理，推荐PAC模式，如果PAC模式无法访问一些网站，就换全局模式试试，一般是因为PAC更新不及时（也可能是GFWList更新不及时）导致的。

# 其他
还有，说一下Chrome不需要Proxy SwitchyOmega和Proxy SwitchySharp插件，这两个插件的作用就是，快速切换代理，判断网站需不需要使用某个代理的（ss已经有pac模式了，所以不需要这个）。如果你只用shadowsocks的话，就不需要这个插件了！

个人认为现在的Shadowsocks加密技术是很不错的，推荐使用Shadowsocks（IOS除外），想要做到类VPN的方式（控制整个网络）可以搭配Proxifier使用，教程看这里。

上面仅代表个人的浅层观点，目前墙针对VPN的Feng Suo越来越严重，比较有名的OpenVPN已经被第一个集火了！而PPTP也逐渐不稳定了起来，可能今天能用，明天就不能用了。L2TP和IPSe0c目前还好，加密方式还无法被效率破解。