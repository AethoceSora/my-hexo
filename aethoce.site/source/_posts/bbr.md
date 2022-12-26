---
title: 如何在服务端启用BBR拥塞控制算法
cover: https://s2.loli.net/2022/12/26/zN9BEclKUFpyDtw.png
tags: Devops
date: 2022/12/26 1:09:26
---



2016 年底，Google 发表了一篇优化 TCP 传输算法的文章，极大的提高了 TCP 的 throughput（吞吐量），并且已经集成到 Linux 4.9 内核中。这个算法就是BBR(Bottleneck Bandwidth and Round-trip propagation time)。



作为一个软件源镜像站，启用BBR将显著提升服务器的带宽利用率。但由于服务器所用的CentOS内核版本过低，我们需要先升级系统内核。

### 更新内核包

1. 执行以下命令，查看当前 Kernel 版本。

   ```shell
   uname -r
   ```

2. 执行以下命令，更新软件包。

   ```shell
   yum update -y
   ```

3. 执行以下命令，导入 ELRepo 公钥。

   ```shell
   rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
   ```

4. 执行以下命令，安装 ELRepo 的 yum 源。

   ```shell
   yum install https://www.elrepo.org/elrepo-release-7.0-4.el7.elrepo.noarch.rpm
   ```

### 安装新内核

1. 执行以下命令，查看 ELRepo 仓库下当前系统支持的内核包。

   ```shell
   yum --disablerepo="*" --enablerepo="elrepo-kernel" list available
   ```

2. 执行以下命令，安装最新的主线稳定内核。

   ```shell
   yum --enablerepo=elrepo-kernel install kernel-ml
   ```

### 更改 grub 配置

1. 执行以下命令，打开 `/etc/default/grub` 文件

   ```shell
   nano /etc/default/grub
   ```

2. 将 `GRUB_DEFAULT=saved` 修改为 `GRUB_DEFAULT=0`。

3. 执行以下命令，重新生成 Kernel 配置。

   ```shell
   grub2-mkconfig -o /boot/grub2/grub.cfg
   ```

4. 重启机器

   ```shell
   reboot
   ```

### 开启BBR

1. 执行以下命令，编辑`/etc/sysctl.conf`文件。

   ```shell
   vim /etc/sysctl.conf
   ```

2.  切换至编辑模式，添加如下内容。

   ```shell
   net.core.default_qdisc=fq
   net.ipv4.tcp_congestion_control=bbr
   ```

3. 执行以下命令，从`/etc/sysctl.conf`配置文件中加载内核参数设置。

   ```shell
   sysctl -p
   ```

4. 依次执行以下命令，验证是否成功开启了 BBR。

   ```shell
   sysctl net.ipv4.tcp_congestion_control
   # 显示如下内容即可：
   # net.ipv4.tcp_congestion_control = bbr
   ```

   ```shell
   sysctl net.ipv4.tcp_available_congestion_control
   # 显示如下内容即可：
   # net.ipv4.tcp_available_congestion_control = reno cubic bbr
   ```

5. 执行以下命令，查看内核模块是否加载。

   ```shell
   lsmod | grep bbr
   ```