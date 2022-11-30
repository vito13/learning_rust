
# 准备

## 安装系统

### 安装 ububtu 20.10 server
- 下载ubuntu-22.10-live-server-amd64.iso
- virtualbox，新建虚拟机：2cpu，4g内存，25g硬盘，网卡1使用默认“网络地址转换NAT”（装完系统后可以在虚拟机里上外网，但与宿主不可互ping，需要加装host-only网卡用于形成互ping的局域网）
- 运行安装server，不要最小安装那样会缺少必要的程序，【Mirror address】那步骤改地址改为：http://mirrors.aliyun.com/ubuntu，  并且选上安装ssh，其余默认即可
- 需要下载的都安装完后，回车重启，重启过程中有err再次回车即可，直到输入用户名密码登录，安装完毕

### 设置root密码

- 首先确保已登录普通用户。
- 输入命令：sudo passwd root，输入当前普通用户密码，设置root用户密码，再次输入确认root用户密码
- 输入命令：su root，输入root用户密码进行登录，成功进入root用户后退出root用户，输入命令exit

### 安装net-tools

```shell
    sudo apt install net-tools
```

### 调整网卡硬件

- 关闭运行的虚拟机系统，以下配置是硬件级别设置，可用于所有被虚拟的系统，如centos7和ubuntu20等

- Virtualbox工具栏->管理->全局设定

    ![全局设定](./全局设定.png)

- Virtualbox工具栏->管理->主机网络管理器

    ![主机网络管理器1](./主机网络管理器1.png)

    ![主机网络管理器2](./主机网络管理器2.png)

    此刻多了一块网卡如下

    ![系统网络设配器](./系统网络设配器.png)

- Virtualbox客户机网络设置

    网络地址转换NAT，用于上外网
    ![客户机网络设置1](./客户机网络设置1.png)

    Host-Only用于ssh，即局域网互通
    ![客户机网络设置2](./客户机网络设置2.png)

### 查看网络状态的命令

- ip a 或 ip addr
- networkctl status -a
- route -n

### 在centos中配置网络

- 修改网卡1（网络地址转换NAT）配置为如下，HWADDR可以不用，UUID可以使用uuidgen生成。保存后, service network restart，此时应可正常访问外网

``` shell

[huawei@10 ~]$ cat /etc/sysconfig/network-scripts/ifcfg-enp0s3
TYPE="Ethernet"
PROXY_METHOD="none"
BROWSER_ONLY="no"
BOOTPROTO="dhcp"
DEFROUTE="yes"
IPV4_FAILURE_FATAL="no"
IPV6INIT="yes"
IPV6_AUTOCONF="yes"
IPV6_DEFROUTE="yes"
IPV6_FAILURE_FATAL="no"
IPV6_ADDR_GEN_MODE="stable-privacy"
NAME="enp0s3"
UUID="00359752-7a18-4c26-bac6-cbb3bb484892"
DEVICE="enp0s3"
ONBOOT="yes"
HWADDR="08:00:27:C5:A0:36"

```

- 修改网卡2（Host-Only）配置为如下，HWADDR可以不用，UUID可以使用uuidgen生成，使用固定IP地址（IPADDR确保在前面创建的host-only网卡的min到max地址中即可）保存后, service network restart，此时应可以与物理机或其他虚拟机互ping

``` shell

[huawei@10 ~]$ cat /etc/sysconfig/network-scripts/ifcfg-enp0s8
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=yes
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=enp0s8
UUID=27aaf75f-90ce-4cb3-94e2-5279c86b3fcf
DEVICE=enp0s8
ONBOOT=yes
HWADDR="08:00:27:2B:24:FD"
IPADDR=192.168.56.120
NETMASK=255.255.255.0

```

### 在ubuntu中配置网络

- 修改配置文件内容为如下（addresses确保在前面创建的host-only网卡的min到max地址中即可，enp0s3是NAT的网卡，enp0s8是Host-Only网络）
- sudo netplan apply
- ip addr 查看静态ip是否设置成功
- 此时应可以与物理机或其他虚拟机互ping且可以上外网
- 如失败可以route -n看下路由表，检查enp0s8的gateway是否非[添加网卡硬件](#添加网卡硬件)的全局设定中的ip，如何改待完善。。。

``` shell

huaw@test:~$ cat /etc/netplan/00-installer-config.yaml
# This is the network config written by 'subiquity'
network:
  ethernets:
    enp0s3:
      addresses: []
      dhcp4: true
    enp0s8:
      dhcp4: false
      addresses: [192.168.56.102/24]
      nameservers:
        addresses: [192.168.56.1]
  version: 2

```

### 配置ssh

- ubuntu server安装时候如选上ssh了，则不用配置
- 手动配置则如下操作：先 sudo vi /etc/ssh/sshd_config，修改PermitRootLogin为yes后保存退出，再 sudo service ssh restart

### 扩大空余磁盘

- 先查看磁盘占用空间，找到挂载的位置（/dev/mapper/ubuntu--vg-ubuntu--lv），然后扩容，再变更文件系统尺寸。操作如下：

``` shell

huaw@test:~$ df -h
Filesystem                         Size  Used Avail Use% Mounted on
tmpfs                              393M  1.1M  392M   1% /run
/dev/mapper/ubuntu--vg-ubuntu--lv   12G  5.1G  5.7G  48% /
tmpfs                              2.0G     0  2.0G   0% /dev/shm
tmpfs                              5.0M     0  5.0M   0% /run/lock
/dev/sda2                          2.0G  136M  1.7G   8% /boot
tmpfs                              393M  4.0K  393M   1% /run/user/1000


huaw@test:~$ sudo lvresize -l +100%FREE /dev/mapper/ubuntu--vg-ubuntu--lv
  New size (5887 extents) matches existing size (5887 extents).

huaw@test:~$ sudo resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv
resize2fs 1.46.5 (30-Dec-2021)
Filesystem at /dev/mapper/ubuntu--vg-ubuntu--lv is mounted on /; on-line resizing required
old_desc_blocks = 2, new_desc_blocks = 3
The filesystem on /dev/mapper/ubuntu--vg-ubuntu--lv is now 6028288 (4k) blocks long.

huaw@test:~$ df -h
Filesystem                         Size  Used Avail Use% Mounted on
tmpfs                              393M  1.1M  392M   1% /run
/dev/mapper/ubuntu--vg-ubuntu--lv   23G  5.1G   17G  24% /
tmpfs                              2.0G     0  2.0G   0% /dev/shm
tmpfs                              5.0M     0  5.0M   0% /run/lock
/dev/sda2                          2.0G  136M  1.7G   8% /boot
tmpfs                              393M  4.0K  393M   1% /run/user/1000
```

### 修改时区

- 执行tzselect，选4 Asia，选10 China，选1 Beijing，选1 yes
- sudo vi ~/.profile，在文件尾追加如下一行

``` shell
    TZ='Asia/Shanghai'; export TZ
```

- sudo vi /etc/default/locale，在文件尾追加如下一行

``` shell
    LC_TIME=zh_CN.UTF-8
```

### 修改shell前导

- /etc/bash.bashrc与/home/huaw/.bashrc，文件末尾追加此行

``` shell
PS1='[\u@\h \W]\$ '
```

### 改源

- 在安装ubuntu server时候因为修改过Mirror address，因此也不用改了
- 如需手动修改则如下

``` shell

sudo vim /etc/apt/sources.list
:%s/cn\.archive\.ubuntu\.com/mirrors\.aliyun\.com/g

```

### 先安装必要工具

此处可以先重启一下后再操作如下

``` shell
sudo apt-get remove needrestart

sudo apt-get install -y gcc
sudo apt-get install -y gdb
sudo apt-get install -y cmake
sudo apt-get install -y python2
sudo apt-get install -y python3
sudo apt-get install -y perl
sudo apt-get install -y openjdk-18-jdk
sudo apt-get install -y openjdk-17-jdk
sudo apt-get install -y cmake
sudo apt-get install -y automake
sudo apt-get install -y git
sudo apt-get install -y pkg-config
sudo apt-get install -y openssl
sudo apt-get install -y libssl-dev
sudo apt-get install -y build-essential
sudo apt-get install -y libxml2
sudo apt-get install -y clang
sudo apt-get install -y libclang
sudo apt-get install -y protoc
sudo apt-get install -y libprotoc-dev
sudo apt-get install -y bison
sudo apt-get install -y flex
sudo apt-get install -y gettext
sudo apt-get install -y libreadline-dev
sudo apt-get install -y protobuf-compiler
sudo apt-get install -y libprotobuf-dev
sudo apt-get install -y unixodbc
```

## 安装rust

- 执行下面，然后选择1

``` shell
curl https://sh.rustup.rs -sSf | sh
```

- sudo vi ~/.profile，末尾添加如下

```shell
export PATH="$HOME/.cargo/bin:$PATH"
```

- 验证成功

``` shell
huaw@test:~$ .  ~/.profile
huaw@test:~$ rustc --version
rustc 1.65.0 (897e37553 2022-11-02)
huaw@test:~$ cargo --version
cargo 1.65.0 (4bc8f24d3 2022-10-20)
```

## 更新rust

``` shell
huaw@test:~$ rustup update
```

## 连接remote ssh失败的解决方法之一

如果连不上，有可能是因为目标虚拟机的ip之前被别的机器使用过，需要重新生成，在windows执行如下(ip为目标虚拟机ip)

```console
ssh-keygen -R 192.168.56.102
```

会更新/c/Users/huaw/.ssh/known_hosts updated，再次连接成功

## 安装vscode插件

rust-analyzer

## helloworld

- 新建“main.rs”

``` rust
fn main()
{
    println!("hello world");
}
```

- 使用“rustc main.rs”进行编译

- 运行可执行文件“./main”即可
