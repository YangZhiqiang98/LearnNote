#### 环境准备

1、[VMware Workstation 16 Pro](https://www.vmware.com/products/workstation-pro/workstation-pro-evaluation.html)

2、[CentOS-7-x86_64-DVD-2009.iso](http://mirrors.aliyun.com/centos/7.9.2009/isos/x86_64/CentOS-7-x86_64-DVD-2009.iso)

3、[SSH 连接改工具 finashell](http://www.hostbuf.com/)



#### 一、安装 centos

[VMware安装CentOS7超详细版_Xiao J.的博客-CSDN博客_vmware安装centos7](https://blog.csdn.net/tsundere_x/article/details/104263100)



如果网络无法打开，则检查本机上的下面两个服务是否打开。

![image-20220405093506875](https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/20220405093521.png)

#### 二、配置 centos

##### 1、更换为国内源

```shell
mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
yum clean all
yum makecache
```

##### 2、安装 JDK

[【Linux】CentOS7下安装JDK详细过程 - Angel挤一挤 - 博客园 (cnblogs.com)](https://www.cnblogs.com/sxdcgaq8080/p/7492426.html)

##### 3、虚拟机IP配置

1) 输入命令 `ip addr` 或者 `ifconfig` 查看虚拟机 IP 地址。（下图采用 [finashell](http://www.hostbuf.com/) 进行连接）

![img](https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/20220405101131.png)

2) 修改网卡配置，修改为静态 IP

- 进入目录 `cd /etc/sysconfig/network-scripts/` ，`ls` 查看网卡

![](https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/20220405101605.png)

- 修改 `ifcfg-ens33`

  ![](https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/20220405102535.png)

  BOOTPROTO 为 hdcp，改为 static
  ONBOOT 为 yes，可能有的是 no，一定要改为 yes。

  再在最后添加如下：（看自己网路地址进行更改）

  ```shell
  # 网关
  GATEWAY=192.168.153.2
  # 静态IP
  IPADDR=192.168.153.132
  NETMASK=255.255.255.0
  # 配置DNS服务器
  DNS1=8.8.8.8
  DNS2=114.114.114.114
  ```

  保存退出，输入 `service network restart` 重启网卡：

  ![](https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/20220405102849.png)

3) 规划集群，预创建 3 台虚拟机，则 IP 规划如下：

```shell
192.168.153.132 master
192.168.153.133 slave1
192.168.153.134 slave2
```

修改 hosts 文件， `vi /etc/hosts`：

![](https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/20220405103320.png)

保存退出。



4) 关闭防火墙：

```shell
systemctl stop firewalld
systemctl disable firewalld
```

> 启动： `systemctl start firewalld`
> 关闭： `systemctl stop firewalld`
> 查看状态： `systemctl status firewalld`
> 开机禁用 ： `systemctl disable firewalld`
> 开机启用 ： `systemctl enable firewalld`



至此，单个虚拟机配置完毕。



###### 参考文章

1、[CenOS7 Hadoop集群搭建（二）：Hadoop集群搭建_机智的小狐狸的博客-CSDN博客](https://blog.csdn.net/Zz_xiaohuli_zZ/article/details/86151317)

2、[使用两台Centos7系统搭建Hadoop-3.1.4完全分布式集群-51CTO.COM](https://os.51cto.com/article/656790.html)



##### 4、克隆副本

1、在关机状态下，右键 master - 管理 - 克隆，按照如下进行克隆。

![](https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/20220405104211.png)

![](https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/20220405104222.png)

![](https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/20220405104234.png)

![](https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/20220405104244.png)

![](https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/20220405104314.png)

2、克隆完毕后，不要启动，点击 `编辑虚拟机设置`

![](https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/20220405104711.png)

选择 `网络适配器`-`高级`

![](https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/20220405104749.png)

点击 `生成`，生成 mac 地址，防止 mac 地址重复造成网络冲突。

![](https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/20220405104843.png)



重复上述两步，克隆 slave2。



3、启动 master、slave1、slave2。

1）修改网卡配置，修改静态 IP。

```ABAP
192.168.153.132 master
192.168.153.133 slave1
192.168.153.134 slave2
```

`vi /etc/sysconfig/network-scripts/ifcfg-ens33` 修改后保存退出

重启网络服务：`service network restart`



2）修改主机名

执行 `vi /etc/sysconfig/network`

增加以下内容:

```shell
NETWORKING=yes
HOSTNAME=master
```

##### 5、配置 SSH 免密登录

1、查看 `/root/.ssh` 是否是一个目录，如果不存在此目录，通过 ssh 连接其他主机即可自动产生。如果没有产生，则可以删除原来的非目录的 .ssh，然后新建 .ssh 目录，然后进入目录。



![](https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/20220405111814.png)



2、执行 `ssh-keygen -t rsa` ，一路回车即可，生成公钥。



![](https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/20220405112210.png)



向本机复制公钥 `cat id_rsa.pub >> authorized_keys`

![](https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/20220405112459.png)



3、登录其他主机，执行 `ssh-keygen -t rsa`，将其他主机的公钥文件内容都拷贝到 master 主机上的 authorized_keys 文件中，命令如下：

```shell
ssh-copy-id -i master #登录 slave1,将公钥拷贝到 master 的 authorized_keys 中

ssh-copy-id -i master #登录 slave2,将公钥拷贝到 master 的 authorized_keys 中
```

查看 authorized_keys：

![](https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/20220405113005.png)



4、将授权文件分配到其他主机上，登录 master，将授权文件分发到 slave1,slave2。（当前目录：`/root/.ssh`）

```shell
scp authorized_keys root@slave1:$PWD
scp authorized_keys root@slave2:$PWD
```



5、测试

![](https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/20220405113452.png)

至此，ssh 免密登录配置完成。

###### 参考文章

1、[CenOS7 Hadoop集群搭建（二）：Hadoop集群搭建_机智的小狐狸的博客-CSDN博客](https://blog.csdn.net/Zz_xiaohuli_zZ/article/details/86151317)

2、[使用两台Centos7系统搭建Hadoop-3.1.4完全分布式集群-51CTO.COM](https://os.51cto.com/article/656790.html)

3、[CentOS7虚拟机实现Hadoop3.0分布式集群安装_咖喱姬姬的博客-CSDN博客](https://blog.csdn.net/qq_34885184/article/details/107840940)

4、[CentOS7设置集群环境SSH免密访问 - 猫不夜行 - 博客园 (cnblogs.com)](https://www.cnblogs.com/MWCloud/p/11348601.html)