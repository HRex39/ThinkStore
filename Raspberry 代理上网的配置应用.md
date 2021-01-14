# Raspberry 代理上网的配置应用
@[TOC](For those impatient)
# 前言
这是一篇面向Linux命令行版本的代理服务器客户端配置指南。
由于我们需要代理上网，所以首先说说硬件的连接方式。

 - 硬件
   - 树莓派3B x 1
   - Thinkpad T480 x 1
 - 软件
   - Raspberry Buster
   - Windows10

T480使用无线网卡连接互联网，在

> win10 -> 网络和Internet设置 -> WLAN -> 更改适配器选项

中设定无线网卡共享网络给以太网，使用网线连接树莓派的RJ45网口，给树莓派上电，这样连接就完成了上网的工作。
![树莓派的简单接线](https://img-blog.csdnimg.cn/20210114194948990.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NzA0Nzk5OQ==,size_16,color_FFFFFF,t_70#pic_center)

# 需要的依赖和准备工作
现在默认大家已经来到了ssh或者远程连接的位置。
如果在这方面有所疑惑可以查询[树莓派的SSH连接](https://www.cnblogs.com/little-kwy/p/10340317.html)和[
Windows远程桌面连接树莓派](https://www.cnblogs.com/feynxd/p/11364669.html)。
## Git
### 安装Git及换源(if necessary)
树莓派换源教程请[点击](https://blog.csdn.net/qq_31720305/article/details/107519699)：
```bash
sudo apt-get install git
```
Git对于大家来说应该是必备且不陌生的东西了。
## SSR
### 下载安装SSR

 - 下载项目文件

```bash
cd ~/Desktop
git clone -b manyuser https://github.com/shadowsocksr-rm/shadowsocksr.git ssr
```
此时可以看到在桌面上出现了ssr的文件夹。
```bash
cd ssr
ls
```
在其中可以找到名为"config.json"的文件，这个文件就是我们需要去配置的文件
### 配置Config.json
 - 配置config.json
在这里笔者给出一个样例，大家可以依此进行参考配置
至于配置什么，这需要由你的服务器来决定
```bash
sudo nano ~/Desktop/ssr/config.json
```

```bash
{
    "server": "X.X.X.X",	**需要读者自行更改**
    "server_ipv6": "::",	**需要读者自行更改**
    "server_port": XXXX,	**需要读者自行更改**
    "local_address": "127.0.0.1",	
    "local_port": 1080,		

    "password": "XXXXXX",	**需要读者自行更改**
    "method": "XXXXXX",		**需要读者自行更改**
    "protocol": "XXXXXX",	**需要读者自行更改**
    "protocol_param": "",	
    "obfs": "XXXXXX",		**需要读者自行更改**
    "obfs_param": "",

    "timeout": 300,
    "udp_timeout": 60,
    "dns_ipv6": false,
    "connect_verbose_info": 1,
    "redirect": "",
    "fast_open": false
}

```
至此，config的配置结束。
如果想了解更加清晰的配置问题，请[点击](http://goldsudo.com/develop/shadowsocks/ssr%E6%9C%8D%E5%8A%A1%E7%AB%AF%E9%85%8D%E7%BD%AE%E8%AF%B4%E6%98%8E%E5%8C%85%E6%8B%AC%E5%A4%9A%E7%94%A8%E6%88%B7%E9%85%8D%E7%BD%AE)。
### 开启SSR

```bash
cd ~/Desktop/ssr/shadowsocks
python3 local.py
```
如果此时，你的整体配置被local.py这个程序接受了，(**注意：这并不代表你的配置成功了**)，那么你将会得到这样一行信息：

```bash
2021-01-14 20:24:13 INFO     local.py:54 starting local at 127.0.0.1:1080
```
至于该如何验证是否成功，我们将在下面进行验证。
## Proxychains
### 安装与配置Proxychains
```bash
sudo apt-get install proxychains
sudo nano /etc/proxychains.conf
```
打开了proxychains.conf后，将最后一行更改为本地的代理信息，比如跟上面保持一致：

```bash
socks5 127.0.0.1 1080
```
### 验证是否代理成功
1、保持ssr的开启，即

```bash
cd ~/Desktop/ssr/shadowsocks
python3 local.py
```
**(要是不在路径中实操似乎会有些问题)**

2、另外开启一个终端，验证使用proxychains时是否可以上网，比如：

```bash
proxychains curl baidu.com
```
如果此时连接正常，你将会得到以下信息：

```bash
pi@DNSPi:~/Desktop $ proxychains curl baidu.com
ProxyChains-3.1 (http://proxychains.sf.net)
|DNS-request| baidu.com
|S-chain|-<>-127.0.0.1:1080-<><>-4.2.2.2:53-<><>-OK
|DNS-response| baidu.com is 220.181.38.148
|S-chain|-<>-127.0.0.1:1080-<><>-220.181.38.148:80-<><>-OK
<html>
<meta http-equiv="refresh" content="0;url=http://www.baidu.com/">
</html>
```
当然，如果你的服务器可以连接外网，也可以试试：

```bash
proxychains curl google.com
```
你将会得到这样的信息：

```bash
pi@DNSPi:~/Desktop $ proxychains curl google.com
ProxyChains-3.1 (http://proxychains.sf.net)
|DNS-request| google.com
|S-chain|-<>-127.0.0.1:1080-<><>-4.2.2.2:53-<><>-OK
|DNS-response| google.com is 172.217.24.78
|S-chain|-<>-127.0.0.1:1080-<><>-172.217.24.78:80-<><>-OK
<HTML><HEAD><meta http-equiv="content-type" content="text/html;charset=utf-8">
<TITLE>301 Moved</TITLE></HEAD><BODY>
<H1>301 Moved</H1>
The document has moved
<A HREF="http://www.google.com/">here</A>.
</BODY></HTML>
```
至此，Proxychains的安装与配置结束。
当然，如果你此时顺利的根据流程走了下来，你就知道代理上网应当如何应用了。
# 使用

```bash
1. cd ~/Desktop/ssr/shadowsocks
2. python3 local.py
另外开启一个终端
3. use "proxychains" in bash
```
## 单一命令行使用
在想使用代理但是软件就是不走代理的命令时，加上proxychains，就能自动走代理，比如：

```bash
proxychains git clone xxxxxxxxx
```
## 整个Terminal使用
```bash
proxychains bash
```
之后就可以在该终端的所有命令中都使用代理了
## 浏览器使用
~~不会吧，不会吧，不会真有人用树莓派刷网页看视频吧（手动狗头）~~ 
至于浏览器的配置问题，请[查阅此](http://www.wangchao.info/1316.html)。
由于笔者只需要命令行的功能作为使用，故未考虑实际浏览器的使用，现在的命令行版本已经满足了笔者的要求。
# 致谢及引用
[1] [Linux使用SSR客户端](https://mikoto10032.github.io/post/%E7%A8%8B%E5%BA%8F%E5%91%98%E9%82%A3%E4%BA%9B%E4%BA%8B/linux%E4%BD%BF%E7%94%A8ssr%E5%AE%A2%E6%88%B7%E7%AB%AF/)
[2] [安装shadowsocks-python并启用chacha20加密](https://blog.phpgao.com/shadowsocks_chacha20.html)
[3] [用proxychains无脑设置Linux代理](https://www.chenxublog.com/2020/06/12/proxychains-proxy.html)
# 故事
笔者想要用Raspberry做一个软路由，同时下载一个Pi-hole准备尝试一下广告过滤器，但事与愿违，可以科学上网的笔记本本应该做一个本地代理，给树莓派提供科学上网的“动力”，但怎么尝试……都以失败告终……
没法度，只能尝试着给树莓派部署一下SSR使其能够科学上网，虽然这么操作略显繁琐，但最终还是解决了问题并使笔者提前预习了计算机网络的相关知识（虽然课就是选不上）……
希望这篇配置应用能对你们有所帮助。
 
