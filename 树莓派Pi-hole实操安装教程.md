# 树莓派Pi-hole实操安装教程
@[TOC](目录)
# 前言
Pi-hole 是一款开源且免费的 DNS 沉洞服务器（DNS sinkhole），能够在不安装任何客户端侧软件的前提下为设备提供网络内容屏蔽服务，非常轻量易用。搭配上家中吃灰已久的树莓派，我们就能够轻松打造属于自己的广告屏蔽助手。[^1]
但我们该如何真正的配置它呢？在实操中会出现很多的问题，下面将为大家带来自己的一点操作流程以及体会。

## 速览
本文主要参考教程为[此网站](https://sspai.com/post/58183)，并在此网站的基础上为大家排个雷。大家可以边看它边操作，出现的问题可以来对照解决。
## 材料

 - 树莓派3B x 1
 - 笔记本 x 1
 - 家用路由器 x 1

# 安装
首先我们需要将树莓派接入路由器，有线或无线均可，然后远程登录到树莓派上：

（现在默认大家已经来到了ssh或者远程连接的位置。 如果在这方面有所疑惑可以查询[树莓派的SSH连接](https://www.cnblogs.com/little-kwy/p/10340317.html)和 [Windows远程桌面连接树莓派](http://cnblogs.com/feynxd/p/11364669.html)。

## 方法1（适合在港澳台以及国外的读者）
Pi-hole 提供了一键安装脚本：

```bash
curl -sSL https://install.pi-hole.net | bash
```
当然我们也可以试试，因为这网络啊实在是太太太难成功了……
## 方法2 （适合中国大陆的读者）

 

 - 首先提升通过修改hosts或者别的什么途径来提高你连接境外服务器的速度。（请读者[自行查阅](https://github.com/HRex39/ThinkStore/blob/master/Raspberry%20%E4%BB%A3%E7%90%86%E4%B8%8A%E7%BD%91%E7%9A%84%E9%85%8D%E7%BD%AE%E5%BA%94%E7%94%A8.md)）
 - 做完了第一步后，你会发现……这样下载的网络永远会被FTL给卡断，所以我们查阅Github的官方文档，得到了以下的解决方案：
 
 ### 1. 手动编译FTL

  - **Debian / Ubuntu / Raspbian，下载依赖**
```bash
sudo apt install build-essential libgmp-dev m4 cmake libidn11-dev libreadline-dev
```

- **从源文件编译 libnettle**
```bash
wget https://ftp.gnu.org/gnu/nettle/nettle-3.6.tar.gz
tar -xzf nettle-3.6.tar.gz
cd nettle-3.6
./configure --libdir=/usr/local/lib
make -j $(nproc)
sudo make install
```
- **Clone FTL**

```bash
git clone https://github.com/pi-hole/FTL.git && cd FTL
```
- **编译 FTL**

```bash
./build.sh
```
注意：这里可能会等很长一段时间，不要担心，你的树莓派没有坏掉，只是……慢……
你可以选择：~~砸掉买台新的~~ 、~~泡杯茶~~ 、~~打几把游戏~~ ，然后等待他慢慢的成功
- **安装 FTL并重启服务**

```bash
./build.sh install
sudo service pihole-FTL restart
```
注：如果一切顺利，此时你应该已经成功下载好了FTL了

### 2. 下载basic-install.sh的脚本文件
```bash
wget -O basic-install.sh https://install.pi-hole.net
```

### 3. 在脚本文件中，修改`FTLinstall()`

 - 在`basic-install.sh`中，按下`ctrl+F`，输入`FTLinstall()`，搜索到该函数，在函数的第一行添加`return 0`
 如：
 

```bash
FTLinstall() {
    return 0 # 这里是我添加的return代码



    # Local, named variables
    local latesttag
    local str="Downloading and Installing FTL"
    printf "  %b %s..." "${INFO}" "${str}"

    # Move into the temp ftl directory
    pushd "$(mktemp -d)" > /dev/null || { printf "Unable to make temporary directory for FTL binary download\\n"; return 1; }

    # Always replace pihole-FTL.service
    install -T -m 0755 "${PI_HOLE_LOCAL_REPO}/advanced/Templates/pihole-FTL.service" "/etc/init.d/pihole-FTL"

    local ftlBranch
    local url

    if [[ -f "/etc/pihole/ftlbranch" ]];then
        ftlBranch=$(</etc/pihole/ftlbranch)
    else
        ftlBranch="master"
    fi

    local binary
    binary="${1}"

    # Determine which version of FTL to download
    if [[ "${ftlBranch}" == "master" ]];then
        url="https://github.com/pi-hole/ftl/releases/latest/download"
    else
        url="https://ftl.pi-hole.net/${ftlBranch}"
    fi

    # If the download worked,
    if curl -sSL --fail "${url}/${binary}" -o "${binary}"; then
        # get sha1 of the binary we just downloaded for verification.
        curl -sSL --fail "${url}/${binary}.sha1" -o "${binary}.sha1"

        # If we downloaded binary file (as opposed to text),
        if sha1sum --status --quiet -c "${binary}".sha1; then
            printf "transferred... "

            # Before stopping FTL, we download the macvendor database
            curl -sSL "https://ftl.pi-hole.net/macvendor.db" -o "${PI_HOLE_CONFIG_DIR}/macvendor.db" || true
            chmod 644 "${PI_HOLE_CONFIG_DIR}/macvendor.db"
            chown pihole:pihole "${PI_HOLE_CONFIG_DIR}/macvendor.db"

            # Stop pihole-FTL service if available
            stop_service pihole-FTL &> /dev/null

            # Install the new version with the correct permissions
            install -T -m 0755 "${binary}" /usr/bin/pihole-FTL

            # Move back into the original directory the user was in
            popd > /dev/null || { printf "Unable to return to original directory after FTL binary download.\\n"; return 1; }

            # Installed the FTL service
            printf "%b  %b %s\\n" "${OVER}" "${TICK}" "${str}"
            return 0
        # Otherwise,
        else
            # the download failed, so just go back to the original directory
            popd > /dev/null || { printf "Unable to return to original directory after FTL binary download.\\n"; return 1; }
            printf "%b  %b %s\\n" "${OVER}" "${CROSS}" "${str}"
            printf "  %bError: Download of %s/%s failed (checksum error)%b\\n" "${COL_LIGHT_RED}" "${url}" "${binary}" "${COL_NC}"
            return 1
        fi
    # Otherwise,
    else
        popd > /dev/null || { printf "Unable to return to original directory after FTL binary download.\\n"; return 1; }
        printf "%b  %b %s\\n" "${OVER}" "${CROSS}" "${str}"
        # The URL could not be found
        printf "  %bError: URL %s/%s not found%b\\n" "${COL_LIGHT_RED}" "${url}" "${binary}" "${COL_NC}"
        return 1
    fi
}
```

 - 再次运行下载程序，这次应该就成了
 

```bash
sudo bash basic-install.sh
```


# 引用
[^1]: [基于树莓派的全能广告屏蔽助手 —— Pi-hole](https://sspai.com/post/58183)
