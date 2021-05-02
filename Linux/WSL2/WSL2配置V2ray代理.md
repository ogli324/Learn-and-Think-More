# WSL2配置V2ray代理

这里是指`win10`下面开了`v2ray`后，怎么配置`WSL2`的`http/https`代理，为什么不配置`socks`代理，因为`unbuntu`下的`apt`及`pip`不支持，要寻找使用`socks`代理`apt-get`的方法。



另外，主机连接的是`WiFi`，要找`win10`在同一WLAN 下的 `IP`。



`v2ray`：

![image-20210502234125649](images/image-20210502234125649.png)



`win10`下`powershell`里面`ipconfig`：

```

无线局域网适配器 WLAN:

   连接特定的 DNS 后缀 . . . . . . . : cmcc.10086.cn
   本地链接 IPv6 地址. . . . . . . . : fe80::395f:8fe6:6c28:b624%2
   IPv4 地址 . . . . . . . . . . . . : 192.168.10.14
   子网掩码  . . . . . . . . . . . . : 255.255.255.0
   默认网关. . . . . . . . . . . . . : 192.168.10.1
```

找到`192.168.10.14`后在`wsl2`里面`vim ~/.bashrc`，在最后加上：

```
export ALL_PROXY="http://192.168.10.14:10809"
export HTTP_PROXY=$ALL_PROXY
export http_proxy=$ALL_PROXY
export HTTPS_PROXY=$ALL_PROXY
export https_proxy=$ALL_PROXY
```

win10防火墙问题自行搜索解决，能相互`ping`通差不多就可以。