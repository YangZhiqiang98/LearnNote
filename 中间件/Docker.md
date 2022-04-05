 [TOC]

## 一、什么是 Docker

Docker 是基于 Go 语言实现的开源容器项目。Docker 的构想是要实现 “Build，Ship and Run Any App，Anywhere”，即通过对应用的封装、分发、部署、运行生命周期进行管理，达到应用组件级别的 “一次封装，到处运行”，简单来说，Docker 是一个开源的应用容器引擎，让开发者可以打包他们的应用以及依赖包到一个可移植的容器中，然后发布到任何流行的 Linux 机器上，也可以实现虚拟化。

能做什么：

- **更快的交付和部署**：使用 Docker，开发人员可以使用镜像来快速构建一套标准的开发环境，只有是开发测试的代码，就可以确保在生产环境无缝运行。
- **虚拟化更加轻量级**：传统的虚拟机都是先虚拟出一个操作系统，然后在操作系统上完成各种各样的配置。docker 则是一种操作系统级别的虚拟技术，它运行在操作系统之上的用户空间，所有的容器都共用一个系统内核甚至公共库，容器引擎**提供了进程级别的隔离**，让每个容器运行在单独的系统之上，但是又能够共享很多底层资源。
- **程序可移植**：Docker 容器几乎可以在任意的平台上运行。
- **更简单的更新管理**：使用 Dockerfile，只需要简单的配置修改，就可以替代以往大量的更新工作。**所有的修改都以增量的方式被分发和更新，从而实现自动化并且高效的容器管理**。



### 1.1 Docker 与虚拟机的比较

| 特性     | 容器               | 虚拟机           |
| -------- | ------------------ | ---------------- |
| 启动速度 | 秒级               | 分钟级           |
| 性能     | 接近原生           | 较弱             |
| 内存代价 | 很小               | 较多             |
| 磁盘使用 | 一般为 MB          | 一般为 GB        |
| 运行密度 | 单机支持上千个容器 | 一般不多于几十个 |
| 隔离性   | 安全隔离           | 完全隔离         |
| 迁移性   | 优秀               | 一般             |

### 1.2 Docker 与虚拟化

> ​	在计算机技术中心，虚拟化是一种资源管理技术，是将计算机的各种实体资源，如服务器、网络、内存及存储等，予以抽象、转换后呈现出来，打破实体结构间不可切割的障碍，使用户可以用比原来的组态更好的方式来应用这些资源。

可见，虚拟化的核心是对资源的抽象，目标往往是为了在同一个主机上同时运行多个系统或应用，从而提高系统资源的利用率，并且带来降低成本、方便管理和容错容灾的好处。

#### 1.2.1 虚拟化技术分类

- 硬件虚拟化
- 软件虚拟化
  - 应用虚拟化（模拟设备）
  - 平台虚拟化（虚拟机技术）
    - 完全虚拟化。虚拟机模拟完整的底层硬件环境和特权指令的执行过程，客户操作系统无须进行修改。如 VMware Workstation、VirtualBox 等。
    - 硬件辅助虚拟化。利用硬件（主要是 CPU）辅助支持处理敏感指令来实现完全虚拟化的功能，客户操作系统无需修改，例如 VMware Workstation，KVM。
    - 部分虚拟化。只针对部分硬件资源进行虚拟化，客户操作系统需要进行修改。
    - 超虚拟化。部分硬件接口以软件的形式提供给客户机操作系统，客户操作系统需要进行修改。
    - **操作作系统级虚拟化**。内核通过创建多个虚拟的操作系统实例（内核和库）来隔离不同的进程。（**容器相关技术在这个范畴**）



![Docker 和传统虚拟化方式的不同之处](https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/Docker%20%E5%92%8C%E4%BC%A0%E7%BB%9F%E8%99%9A%E6%8B%9F%E5%8C%96%E6%96%B9%E5%BC%8F%E7%9A%84%E4%B8%8D%E5%90%8C%E4%B9%8B%E5%A4%84.png)

传统方式是在硬件层面实现虚拟化，需要有额外的虚拟机管理应用和虚拟机操作系统层。Docker 容器是在操作系统层面上实现虚拟化，直接复用本地主机的操作系统，因此更加轻量级。

## 二、核心概念

Docker 三大核心概念：镜像、容器和仓库。

### 2.1、Docker 镜像

Docker 镜像类似于虚拟机镜像，可以将它理解为一个只读的模板。

#### 2.1.1镜像与容器的关系

镜像是容器运行的基础，容器是镜像运行后的形态。总体来说，镜像是一个包含程序运行必要的环境和代码的只读文件，它采用分层的文件系统，将每一层的改变以读写层de形式增加到原来的只读文件上。

#### 2.1.2 镜像的体系结构

镜像的最底层是一个启动文件系统（bootfs）镜像，bootfs 的上层镜像叫做根镜像。一般来说，根镜像是一个操作系统，例如 Ubuntu、CentOS 等。用户的镜像必须构建于根镜像之上，在根镜像之上，用户可以构建出各种各样的其它镜像。

#### 2.1.3 镜像的写时复制机制

通过 `docker run` 命令指定一个容器创建镜像时，实际上是在该镜像之上创建一个空的可读写的文件系统层级，可以将这个文件系统层级当成一个临时的镜像来对待，而命令中所指的模板镜像则可以称为父镜像。父镜像的内容都是以只读的方式挂载进来的，容器会读取共享父镜像的内容，用户所做的所有修改都是在文件系统中，不会对父镜像造成任何影响。当然用户可以通过其它一些手段是修改持久化到父镜像中。



### 2.2 Docker 容器

Docker 容器类似于一个轻量级的沙箱，Docker 利用容器来运行和隔离应用。

容器是从镜像创建的应用运行实例，它可以启动、开始、停止、删除，而这些容器都是彼此相互隔离，互不可见的。

> 镜像自身是只读的。容器从镜像启动的时候，会在镜像的最上层创建一个可写层。

### 2.3 Docker 仓库

Docker 仓库是 Docker 集中存放镜像文件的场所。

区分仓库和注册服务器（Registry）：注册服务器是存放仓库的具体服务器，一个注册服务器上可以有多个仓库，而每个仓库下面可以有多个镜像。从这方面来说，仓库可以被认为是一个具体的项目或目录。

官方仓库：[Docker Hub](https://hub.docker.com/search?q=&type=image&image_filter=official)

第三方仓库：腾讯云、网易云、阿里云等。



## 三、使用 Docker 镜像

Docker 运行容器前需要本地存在对应的镜像，如果镜像不存在，Docker 会尝试先从默认镜像仓库下载镜像（默认使用 Docker Hub 公共注册服务器中的仓库），用户也可以通过配置使用自定义的镜像仓库。

### 3.1 获取镜像

命令：[` docker [image] pull [OPTIONS] NAME[:TAG]`](https://docs.docker.com/engine/reference/commandline/pull/)

- NAME：镜像仓库名称（用来区分镜像）
- TAG：镜像的标签（往往用来表示版本信息），如果不显示指定，默认会选择 latest 标签。

可选选项：

| Name, shorthand           | Default | Description                      |
| ------------------------- | ------- | -------------------------------- |
| `--all-tags` , `-a`       |         | 是否获取仓库中的所有镜像         |
| `--disable-content-trust` | `true`  | 取消镜像的内容校验，默认为真     |
| `--platform`              |         | 如果服务器支持多平台，则设置平台 |
| `--quiet` , `-q`          |         | 不打印详细输出                   |

例子：获取一个 Ubuntu 18.04 系统的基础镜像

```shell
$ docker pull ubuntu:18.04
18.04: Pulling from library/ubuntu
e4ca327ec0e7: Pull complete 
Digest: sha256:9bc830af2bef73276515a29aa896eedfa7bdf4bdbc5c1063b4c457a4bbb8cd79
Status: Downloaded newer image for ubuntu:18.04
docker.io/library/ubuntu:18.04
```

Docker 镜像一般由若干层（layer）组成，在上面的例子中，只有一层：`e4ca327ec0e7`，这些层会被镜像重复利用，即**当不同的镜像包括相同的层时，本地仅存储了层的一份内容**。

默认镜像会从 Docker Hub 上下载，用户也可下载时指定仓库注册服务器（registry）。

```shell
 docker pull myregistry.local:5000/testing/test-image
```



### 3.2 查看镜像信息

#### 3.2.1 使用 images 命令列出镜像

使用 [`docker images `](https://docs.docker.com/engine/reference/commandline/images/) 或者 [`docker image ls`](https://docs.docker.com/engine/reference/commandline/image_ls/) 命令可以列出已有镜像的基本信息。

```shell
[root@VM_16_10_centos ~]# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
ubuntu              latest              fb52e22af1b0        3 weeks ago         72.8MB
ubuntu              18.04               54919e10a95d        3 weeks ago         63.1MB
[root@VM_16_10_centos ~]# docker image ls
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
ubuntu              latest              fb52e22af1b0        3 weeks ago         72.8MB
ubuntu              18.04               54919e10a95d        3 weeks ago         63.1MB
```

在列出的信息中，可以看到几个字段信息：

- 来自于哪个仓库，比如 ubuntu 表示 ubuntu 系列的基础镜像；
- 镜像的标签信息，标签只是标记了并不能标识镜像内容；
- 镜像的 ID（唯一标识镜像）：如果两个镜像的 ID 相同，说明它们实际上指向了同一个镜像，只是具有不同的标签名称而已；
- 创建时间，说明镜像最后的更新时间；
- 镜像大小，优秀的镜像往往体积都较小；

在使用镜像 ID 时，一般可以使用该 ID 的前若干个字符组成的可区分串来替代完整的 ID；镜像大小信息只是表示了该镜像的逻辑体积大小，实际上由于相同的镜像层本地只会存储一份，物理上占用的存储空间会小于各镜像逻辑体积之和。

images 子命令主要支持如下选项：

| Name, shorthand   | Default | Description                        |
| ----------------- | ------- | ---------------------------------- |
| `--all` , `-a`    |         | 列出所有（包括临时文件）镜像文件   |
| `--digests`       |         | 列出镜像的数字摘要值               |
| `--filter` , `-f` |         | 过滤列出的镜像                     |
| `--format`        |         | 控制输出格式                       |
| `--no-trunc`      |         | 对输出结果中太长的部分是否进行截断 |
| `--quiet` , `-q`  |         | 仅输出 ID 信息                     |

####  3.2.2 使用 tag 命令添加镜像标签

`docker tag` 命令用来为本地镜像任意添加新的标签。

命令：[` docker tag SOURCE_IMAGE[:TAG] TARGET_IMAGE[:TAG]`](https://docs.docker.com/engine/reference/commandline/tag/)

```shell
[root@VM_16_10_centos ~]# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
ubuntu              latest              fb52e22af1b0        3 weeks ago         72.8MB

[root@VM_16_10_centos ~]# docker tag ubuntu:latest myubuntu:latest
[root@VM_16_10_centos ~]# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
myubuntu            latest              fb52e22af1b0        3 weeks ago         72.8MB
ubuntu              latest              fb52e22af1b0        3 weeks ago         72.8MB
```

之后，用户就可以直接使用新的标签来表示镜像了。

#### 3.2.3 使用 inspect 命令查看详细信息

`docker [image] inspect` 命令可以获取该镜像的详细信息，包括制作者、各层的数字摘要等。

命令： [`docker inspect [OPTIONS] NAME|ID [NAME|ID...]`](https://docs.docker.com/engine/reference/commandline/inspect/)

```shell
[root@VM_16_10_centos ~]# docker inspect ubuntu:latest
[
    {
        "Id": "sha256:fb52e22af1b01869e23e75089c368a1130fa538946d0411d47f964f8b1076180",
        "RepoTags": [
            "myubuntu:latest",
            "ubuntu:latest"
        ],
        "RepoDigests": [
            "ubuntu@sha256:9d6a8699fb5c9c39cf08a0871bd6219f0400981c570894cd8cbea30d3424a31f"
        ],
        "Parent": "",
        "Comment": "",
        "Created": "2021-08-31T01:20:56.191693866Z",
        "Container": "18d45584c4556d1d5a49733809341f0b2897434405d1297c27a371d419e1e9e9",
        "ContainerConfig": {
            "Hostname": "18d45584c455",
            "Domainname": "",
            "User": "",
            "AttachStdin": false,
            "AttachStdout": false,
            "AttachStderr": false,
            "Tty": false,
            "OpenStdin": false,
            "StdinOnce": false,
            "Env": [
                "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
            ],
            "Cmd": [
                "/bin/sh",
                "-c",
                "#(nop) ",
                "CMD [\"bash\"]"
            ],
            "Image": "sha256:97952717dbcb6750d0cef7e77a6188d9d4a7864122fd45d2434b5267b8e54e5d",
            "Volumes": null,
            "WorkingDir": "",
            "Entrypoint": null,
            "OnBuild": null,
            "Labels": {}
        },
        "DockerVersion": "20.10.7",
        "Author": "",
        "Config": {
            "Hostname": "",
            "Domainname": "",
            "User": "",
            "AttachStdin": false,
            "AttachStdout": false,
            "AttachStderr": false,
            "Tty": false,
            "OpenStdin": false,
            "StdinOnce": false,
            "Env": [
                "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
            ],
            "Cmd": [
                "bash"
            ],
            "Image": "sha256:97952717dbcb6750d0cef7e77a6188d9d4a7864122fd45d2434b5267b8e54e5d",
            "Volumes": null,
            "WorkingDir": "",
            "Entrypoint": null,
            "OnBuild": null,
            "Labels": null
        },
        "Architecture": "amd64",
        "Os": "linux",
        "Size": 72776725,
        "VirtualSize": 72776725,
        "GraphDriver": {
            "Data": {
                "MergedDir": "/var/lib/docker/overlay2/8792f889974f0fe5476dd3f19c82cdfeb65c88955acebf97c803c99c8a742ac8/merged",
                "UpperDir": "/var/lib/docker/overlay2/8792f889974f0fe5476dd3f19c82cdfeb65c88955acebf97c803c99c8a742ac8/diff",
                "WorkDir": "/var/lib/docker/overlay2/8792f889974f0fe5476dd3f19c82cdfeb65c88955acebf97c803c99c8a742ac8/work"
            },
            "Name": "overlay2"
        },
        "RootFS": {
            "Type": "layers",
            "Layers": [
                "sha256:4942a1abcbfa1c325b1d7ed93d3cf6020f555be706672308a4a4a6b6d631d2e7"
            ]
        },
        "Metadata": {
            "LastTagTime": "2021-09-21T23:04:23.286497043+08:00"
        }
    }
]

```

上面返回的是一个 JSON 格式的消息，如果我们只要其中一项内容时，可以通过 `-f` 指定。

```shell
[root@VM_16_10_centos ~]# docker inspect -f {{".Metadata"}} fb52e
{2021-09-21 23:04:23.286497043 +0800 CST}
```

#### 3.2.4 使用 history 命令查看镜像历史

`docker history` 将列出各层的创建信息。

命令：[` docker history [OPTIONS] IMAGE`](https://docs.docker.com/engine/reference/commandline/history/)

```shell
[root@VM_16_10_centos ~]# docker history ubuntu:latest
IMAGE               CREATED             CREATED BY                                      SIZE                COMMENT
fb52e22af1b0        3 weeks ago         /bin/sh -c #(nop)  CMD ["bash"]                 0B                  
<missing>           3 weeks ago         /bin/sh -c #(nop) ADD file:d2abf27fe2e8b0b5f…   72.8MB         
```

注意：过长的命令被自动截断了，可以使用 `--no-trunc` 来输出完整的命令。

### 3.3 搜索镜像

使用 `docker search` 命令可以搜索 Docker Hub 官方仓库中的镜像。

命令：[`docker search [OPTIONS] TERM`](https://docs.docker.com/engine/reference/commandline/search/)

支持如下选项：

| Name, shorthand   | Default | Description      |
| ----------------- | ------- | ---------------- |
| `--filter` , `-f` |         | 过滤输出内容     |
| `--format`        |         | 格式化输出内容   |
| `--limit`         | `25`    | 限制输出结果个数 |
| `--no-trunc`      |         | 不截断输出结果   |

如：

```shell
[root@VM_16_10_centos ~]# docker search --filter=is-official=true nginx
NAME                DESCRIPTION                STARS               OFFICIAL            AUTOMATED
nginx               Official build of Nginx.   15513               [OK]       

[root@VM_16_10_centos ~]# docker search --filter=stars=50 tomcat
NAME                DESCRIPTION                                     STARS               OFFICIAL            AUTOMATED
tomcat              Apache Tomcat is an open source implementati…   3130                [OK]                
tomee               Apache TomEE is an all-Apache Java EE certif…   92                  [OK]                
dordoka/tomcat      Ubuntu 14.04, Oracle JDK 8 and Tomcat 8 base…   58                                      [OK]
```

返回包括镜像名称、描述、收藏数、是否官方创建、是否自动创建等。默认的输出结果将按照星级评价进行排序。

### 3.4 删除和清理镜像

#### 3.4.1 删除镜像

使用 `docker rmi` 或 `docker image rm` 命令可以删除镜像。

命令：[` docker rmi [OPTIONS] IMAGE [IMAGE...]`](https://docs.docker.com/engine/reference/commandline/rmi/)

- IMAGE：为标签或 ID。

支持选项：

| Name, shorthand  | Default | Description                    |
| ---------------- | ------- | ------------------------------ |
| `--force` , `-f` |         | 强制删除镜像，即使有容器依赖它 |
| `--no-prune`     |         | 不要清理未带标签的父镜像       |



```shell
[root@VM_16_10_centos ~]# docker image ls
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
myubuntu            latest              fb52e22af1b0        3 weeks ago         72.8MB
ubuntu              latest              fb52e22af1b0        3 weeks ago         72.8MB
ubuntu              18.04               54919e10a95d        3 weeks ago         63.1MB

[root@VM_16_10_centos ~]# docker rmi myubuntu:latest
Untagged: myubuntu:latest
[root@VM_16_10_centos ~]# docker image ls
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
ubuntu              latest              fb52e22af1b0        3 weeks ago         72.8MB
ubuntu              18.04               54919e10a95d        3 weeks ago         63.1MB


[root@VM_16_10_centos ~]# docker rmi -f fb52e22
Untagged: myubuntu:latest
Untagged: ubuntu:latest
Untagged: ubuntu@sha256:9d6a8699fb5c9c39cf08a0871bd6219f0400981c570894cd8cbea30d3424a31f
Deleted: sha256:fb52e22af1b01869e23e75089c368a1130fa538946d0411d47f964f8b1076180
Deleted: sha256:4942a1abcbfa1c325b1d7ed93d3cf6020f555be706672308a4a4a6b6d631d2e7
```

- 使用标签删除镜像

当同一个镜像拥有多个标签时，`docker rmi` 命令只是删除了该镜像多个标签中指定的标签而已，并不影响镜像文件。但当镜像只剩下一个标签时，再使用 `docker rmi` 命令会彻底删除镜像。

- 使用 镜像 ID 来删除镜像

使用镜像 ID 来删除镜像，会先尝试删除所有指向该镜像的标签，然后删除该镜像本身。当有该镜像创建的容器存在时，镜像文件默认是无法被删除的。如果强制删除，可以使用 `-f` 参数。

#### 3.4.1 清理镜像

使用 Docker 一段时间后，系统中可能会遗留一些临时的镜像文件，以及一些没有被使用的镜像，可以通过 `docker image prune` 命令来进行清理。

命令：[` docker image prune [OPTIONS]`](https://docs.docker.com/engine/reference/commandline/image_prune/)

支持选项：

| Name, shorthand  | Default | Description                      |
| ---------------- | ------- | -------------------------------- |
| `--all` , `-a`   |         | 删除所有无用镜像，不光是临时镜像 |
| `--filter`       |         | 只清理符合给定过滤器的镜像       |
| `--force` , `-f` |         | 强制删除镜像，而不进行提示确认   |

如下命令会自动清理临时的遗留镜像文件层，最后会提示释放的存储空间：

```shell
[root@VM_16_10_centos ~]# docker image prune -f
Total reclaimed space: 0B
```

### 3.5 创建镜像

创建镜像的方法有三种：基于已有镜像的容器创建、基于本地模板导入、基于 Dockerfile 创建。

#### 3.5.1 基于已有容器创建

该方法主要是使用 `docker [container] commit` 命令；

命令： [`docker [container] commit [OPTIONS] CONTAINER [REPOSITORY[:TAG]]`](https://docs.docker.com/engine/reference/commandline/container_commit/)

支持选项：

| Name, shorthand    | Default | Description                    |
| ------------------ | ------- | ------------------------------ |
| `--author` , `-a`  |         | 作者信息                       |
| `--change` , `-c`  |         | 提交的时候执行 Dockerfile 指令 |
| `--message` , `-m` |         | 提交信息                       |
| `--pause` , `-p`   | `true`  | 提交时暂停容器运行             |

首先，启动一个镜像，并在其中进行修改操作。然后可以提交一个新的镜像。

```shell
[root@VM_16_10_centos ~]# docker run -it ubuntu:18.04 /bin/bash
root@16961eedf943:/# touch test
root@16961eedf943:/# exit
exit
[root@VM_16_10_centos ~]# docker commit -m "add a new file" -a "docker new" 16961eedf943 test:0.1
sha256:cc4078d3de5a74b46505a8579b5cbe14a5709fe637890dde7ce9fff3576caad8
[root@VM_16_10_centos ~]# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
test                0.1                 cc4078d3de5a        5 seconds ago       63.1MB
ubuntu              18.04               54919e10a95d        3 weeks ago         63.1MB

```



#### 3.5.2 基于本地模板导入

用户也可以直接从一个操作系统模板文件导入一个镜像，主要使用 `docker [container] import` 命令。

命令：`docker [image] import [OPTIONS] file|URL|- [REPOSITORY[:TAG]]`



#### 3.5.3 基于 Dockerfile 创建

基于 Dockerfile 创建时最常见的方式。Dockerfile 是一个文本文件，利用给定的指令描述基于某个父镜像创建新镜像的过程。



###  3.6 存出和载入镜像

#### 3.6.1 存储镜像

如果要导出镜像到本地文件，可以使用 `docker [image] save` 命令。 

命令：[` docker [image] save [OPTIONS] IMAGE [IMAGE...]`](https://docs.docker.com/engine/reference/commandline/save/)

支持选项：

| Name, shorthand   | Default | Description  |
| ----------------- | ------- | ------------ |
| `--output` , `-o` |         | 指定写入文件 |

```shell
[root@VM_16_10_centos ~]# docker save -o /tmp/ubuntu_18.04.tar ubuntu:18.04
[root@VM_16_10_centos ~]# cd /tmp
[root@VM_16_10_centos tmp]# ls
hsperfdata_root    ubuntu_18.04.tar
```

#### 3.6.2 载入镜像

可以使用 `docker [image] load` 将导出的 tar 文件再导入到本地镜像库。支持 -i、-input string 选项，从指定文件中读入镜像内容。

命令：[`docker [image] load [OPTIONS]`](https://docs.docker.com/engine/reference/commandline/load/)

支持选项：

| Name, shorthand  | Default | Description                                  |
| ---------------- | ------- | -------------------------------------------- |
| `--input` , `-i` |         | Read from tar archive file, instead of STDIN |
| `--quiet` , `-q` |         | Suppress the load output                     |

```shell
$ docker load -i ubuntu_18.04.tar
或者
$ docker load < ubuntu_18.04.tar
```

### 3.7 上传镜像

命令：[` docker[image] push [OPTIONS] NAME[:TAG]`](https://docs.docker.com/engine/reference/commandline/push/)

支持选项：

| Name, shorthand           | Default | Description                              |
| ------------------------- | ------- | ---------------------------------------- |
| `--all-tags` , `-a`       |         | Push all tagged images in the repository |
| `--disable-content-trust` | `true`  | Skip image signing                       |
| `--quiet` , `-q`          |         | Suppress verbose output                  |

该命令默认将镜像上传到 Docker Hub 官方仓库，

##  四、操作 Docker 容器

容器是 Docker 的另一个核心概念。简单来说，容器是镜像的一个运行实例。所不同的是，镜像是静态的只读文件，而容器是运行时需要的可写文件层，同时，容器中的应用进程处于运行状态。

### 4.1 创建容器

#### 4.1.1 新建容器

命令：[`docker [container] create [OPTIONS] IMAGE [COMMAND] [ARG...]`](https://docs.docker.com/engine/reference/commandline/create/)

如：使用 `docker [container] create ` 命令新建的容器处于停止状态，可以使用 `docker [container] start` 命令来启动它。

```shell
[root@VM_16_10_centos tmp]# docker create -it ubuntu:latest
819453ec892b0abdbbe8d796dae5f457f660317f109a3c6e5a1009fda803a4c7
[root@VM_16_10_centos tmp]# docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
819453ec892b        ubuntu:latest       "bash"              7 seconds ago       Created                                 blissful_williams
[root@VM_16_10_centos tmp]# docker start 819453
819453
[root@VM_16_10_centos tmp]# docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED              STATUS              PORTS               NAMES
819453ec892b        ubuntu:latest       "bash"              About a minute ago   Up 1 second                             blissful_williams
```

主要选项：

- 与容器运行模式相关

| 选项                                                        | 说明                                                         |
| ----------------------------------------------------------- | ------------------------------------------------------------ |
| `--attach` , `-a`                                           | 是否绑定到标准输入、输出和错误                               |
| **`--detach` , `-d`**                                       | **是否在后台运行容器，默认为否**                             |
| `--detach-keys=""`                                          | 从 attach 模式退出的快捷键                                   |
| `--entrypoint=""`                                           | 镜像存在入口命令时，覆盖为新的命令                           |
| **`--expose=[]`**                                           | **指定容器会暴露出来的端口或端口范围**                       |
| `--group-add=[]`                                            | 运行容器的用户组                                             |
| **`--interactive=true|false,-i`**                           | **保持标准输入打开，默认为false**                            |
| `--ipc`                                                     | 容器 IPC 命名空间，可以为其他容器或主机                      |
| `--isolation="default"`                                     | 容器使用的隔离机制                                           |
| `--log-driver="json-file"`                                  | 指定容器使用的日志驱动类型，可以为 json-file、syslog、journald、gelf 、fluentd、awslogs等。[详见](https://docs.docker.com/config/containers/logging/configure/#supported-logging-drivers) |
| `--log-opt=[]`                                              | 传递给日志驱动的选项，[详见](https://docs.docker.com/config/containers/logging/log_tags/) |
| `--net="bridge"`                                            | 指定容器网络模式，包括 bridge、none、其他容器内网络、host 的网络或者某个现有网络等 |
| `--publish,-p`                                              | 指定如何映射到本地主机端口，如L-p 11234-11234；1234-1234     |
| `--publish-all=true|false`,`-P`                             | 通过 NAT 机制将容器标记暴露的端口自动映射到本地主机的临时端口 |
| `--pid=host`                                                | 容器的 PID 命名空间                                          |
| `--userns=""`                                               | 启用 userns-remap 时配置用户命名空间的模式                   |
| `--uts=host`                                                | 容器 UTS 命名空间                                            |
| `--restart="no"`                                            | 容器的重启策略，包括 no、on-failure[:max-retry]、always、unless-stopped 等。 |
| **`--rm=true|false`**                                       | **容器退出后是否自动删除，不能跟 -d 同时使用**               |
| **`--tty=true|false , -t`**                                 | **是否分配一个伪终端，默认为 false**                         |
| `--tmpfs=[]`                                                | 挂在临时系统文件系统到容器                                   |
| **`-v| --volume [=[[HOST-DIR:] CONTAINER-DIR[:OPTIONS]]]`** | **挂载主机上的文件卷到容器内**                               |
| `--volume-driver=""`                                        | 挂载文件卷的驱动类型                                         |
| `--volumes-from=[]`                                         | 从其他容器挂在卷                                             |
| `--workdir=""` , `-w`                                       | 容器内的默认工作目录                                         |



- 与容器环境和配置相关的选项

| 选项                          | 说明                                                         |
| ----------------------------- | ------------------------------------------------------------ |
| `--add-host=[]`               | 在容器内添加一个主机名到 IP 地址的映射关系（host:ip,通过 /etc/hosts 文件） |
| `--device=[]`                 | 映射物理机上的设备到容器内                                   |
| `--dns-search=[]`             | DNS 搜索域                                                   |
| `--dns-opt=[]`                | 自定义的 DNS 选项                                            |
| `--dns=[]`                    | 自定义的 DNS 服务器                                          |
| `--env=[]` , `-e`             | 指定容器内的环境变量                                         |
| `--env-file=[]`               | 从文件中读取环境变量到容器内                                 |
| `--hostname=""` , `-h`        | 指定容器内的主机名                                           |
| `--ip`                        | 指定容器内的 IPv4 地址                                       |
| `--ip6`                       | 指定容器内的 IPv6 地址                                       |
| `--link=[<name or id>:alisa]` | 链接到其他容器                                               |
| `--link-local-ip=[]`          | 容器的本地链接地址列表                                       |
| `--mac-address`               | 指定容器的 Mac 地址                                          |
| `--name`                      | 指定容器的别名                                               |

- 与容器资源限制和安全保护相关的选项

| 选项                                         | 说明                                                         |
| -------------------------------------------- | ------------------------------------------------------------ |
| `--blkio-weight=10~1000`                     | 容器读写块设备的 I/O 性能权重，默认为 0                      |
| `--blkio-weight-device=[DEVICE_BANE:WEIGHT]` | 指定各个块设备的  I/O 性能权重                               |
| `--cap-add=[]`                               | 增加容器的 Linux 指定安全能力                                |
| `--cap-drop=[]`                              | 移除容器的 Linux 指定安全能力                                |
| `--cgroup-parent=""`                         | 容器 cgroups 限制的创建路径                                  |
| `--cgroupns`                                 | [**API 1.41+**](https://docs.docker.com/engine/api/v1.41/) Cgroup namespace to use (host\|private) 'host' |
| `--cidfile=""`                               | 指定容器的进程 ID 号写到文件                                 |
| `--cpu-count`                                | CPU count (Windows only)                                     |
| `--cpu-percent`                              | CPU percent (Windows only)                                   |
| `--cpu-period=0`                             | 限制容器在 CFS 调度器下的 CPU 占用时间片                     |
| `--cpu-quota=0`                              | 限制容器在 CFS 调度器下的 CPU 配额                           |
| `--cpu-rt-period`                            | [**API 1.25+**](https://docs.docker.com/engine/api/v1.25/) Limit CPU real-time period in microseconds |
| `--cpu-rt-runtime`                           | [**API 1.25+**](https://docs.docker.com/engine/api/v1.25/) Limit CPU real-time runtime in microseconds |
| `--cpu-shares` , `-c`                        | CPU shares (relative weight)                                 |
| `--cpus`                                     | [**API 1.25+**](https://docs.docker.com/engine/api/v1.25/) Number of CPUs |
| `--cpuset-cpus`                              | 限制容器能使用那些 CPU 核心 (0-3, 0,1)                       |
| `--cpuset-mems`                              | MEMs in which to allow execution (0-3, 0,1)                  |
| `--device-read-bps`                          | Limit read rate (bytes per second) from a device；挂载设备的读吞吐率（以 bps 为单位）限制 |
| `--device-read-iops`                         | Limit read rate (IO per second) from a device；挂载设备的读速率（以 每秒 i/o 次数为单位）限制 |
| `--device-write-bps`                         | Limit write rate (bytes per second) to a device；；挂载设备的写吞吐率（以 bps 为单位）限制 |
| `--device-write-iops`                        | Limit write rate (IO per second) to a device；挂载设备的写速率（以 每秒 i/o 次数为单位）限制 |
| `--gpus`                                     | [**API 1.40+**](https://docs.docker.com/engine/api/v1.40/) GPU devices to add to the container ('all' to pass all GPUs) |
| `--health-cmd`                               | 指定检查容器健康状态的命令                                   |
| `--health-interval`                          | Time between running the check (ms\|s\|m\|h) (default 0s)    |
| `--health-retries`                           | 健康检查失败重试次数，超过则认为不健康                       |
| `--health-start-period`                      | [**API 1.29+**](https://docs.docker.com/engine/api/v1.29/) Start period for the container to initialize before starting health-retries countdown (ms\|s\|m\|h) (default 0s) |
| `--health-timeout`                           | Maximum time to allow one check to run (ms\|s\|m\|h) (default 0s) |
| `--init`                                     | 在容器中执行一个 init 进程，来负责响应信号和处理僵尸状态子进程 |
| `--interactive` , `-i`                       | Keep STDIN open even if not attached                         |
| `--io-maxbandwidth`                          | Maximum IO bandwidth limit for the system drive (Windows only) |
| `--io-maxiops`                               | Maximum IOps limit for the system drive (Windows only)       |
| `--kernel-memory`                            | Kernel memory limit                                          |
| `--label` , `-l`                             | 以键值对方式指定容器的标签信息                               |
| `--label-file`                               | 从文件中读取标签信息                                         |
| `--memory` , `-m`                            | 限制容器内应用使用的内存                                     |
| `--memory-reservation`                       | 当系统中内存过低时，容器会被强制限制内存到给定值，默认情况下等于内存限制值 |
| `--memory-swap="LIMIT"`                      | 限制容器使用内存和交换区的大小                               |
| `--memory-swappiness`                        | 调整容器的内存交换区参数 (0 to 100)                          |
| `--no-healthcheck`                           | 是否禁用健康检查                                             |
| `--oom-kill-disable=true|false`              | 内存好近时是否杀死容器                                       |
| `--oom-score-adj`                            | 调整容器的内存耗尽参数 (-1000 to 1000)                       |
| `--pids-limit`                               | 限制容器的 pid 个数 (set -1 for unlimited)                   |
| `--privileged`                               | 是否给容器最高权限                                           |
| `--read-only`                                | 是否让容器内的文件系统只读                                   |
| `--security-opt`                             | 指定一些安全参数，包括权限、安全能力等                       |
| `--shm-size`                                 | Size of /dev/shm                                             |
| `--sig-proxy`                                | 是否代理收到的信号给应用，默认为 true,不能代理 SIGCHLD、SIGSTOP 和 SIGKILL 信号 |
| `--stop-signal=SIGTERM`                      | 指定停止容器内的系统信号                                     |
| `--stop-timeout`                             | [**API 1.25+**](https://docs.docker.com/engine/api/v1.25/) Timeout (in seconds) to stop a container |
| `--storage-opt`                              | Storage driver options for the container                     |
| `--sysctl`                                   | Sysctl options                                               |
| `--ulimit`                                   | 通过 ulimit 来限制最大文件数、最大进程数等                   |
| `--user` , `-u`                              | 指定容器内执行命令的用户信息 (format: <name\|uid>[:<group\|gid>]) |
| `--userns`                                   | 指定用户命名空间                                             |
| `--uts`                                      | UTS namespace to use                                         |



#### 4.1.2 启动容器

使用 `docker [container] start` 命令来启动一个已经创建的容器。

#### 4.1.3 新建并启动容器

命令：[`docker run [OPTIONS] IMAGE [COMMAND] [ARG...]`](https://docs.docker.com/engine/reference/commandline/run/)

该命令等价于先执行 `docker [container] create` 命令，再执行 `docker [container] start` 命令。

利用 `docker [container] run`  来创建并启动容器时，Docker 在后台运行的标准操作包括：

- 检查本地是否存在指定的镜像，不存在就从公有仓库下载；
- 利用镜像创建一个容器，并启动该容器；
- 分配一个文件系统给容器，并在只读的镜像层外面挂载一层可读写层；
- 从宿主主机配置的网桥皆苦中桥接一个虚拟接口到容器中去；
- 从网桥的地址池配置一个 IP 地址给容器；
- 执行用户指定的应用程序；
- 执行完毕后容器被自动终止；

#### 4.1.4  查看容器输出

命令：[` docker logs [OPTIONS] CONTAINER`](https://docs.docker.com/engine/reference/commandline/logs/)

支持选项：

| Name, shorthand       | Default | Description              |
| --------------------- | ------- | ------------------------ |
| `--details`           |         | 打印详细信息             |
| `--follow` , `-f`     |         | 持续保持输出             |
| `--since`             |         | 输出从某个时间开始的日志 |
| `--tail` , `-n`       | `all`   | 输出最近的若干日志       |
| `--timestamps` , `-t` |         | 显示时间戳信息           |
| `--until`             |         | 输出某个时间之前的日志   |

### 4.2 停止容器

#### 4.2.1 暂停容器

命令： [` docker [container] pause CONTAINER [CONTAINER...]`](https://docs.docker.com/engine/reference/commandline/pause/)

```shell
[root@VM_16_10_centos ~]# docker pause e04bd9
e04bd9
[root@VM_16_10_centos ~]# docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS                   PORTS               NAMES
e04bd9d537e4        ubuntu:latest       "bash"              14 minutes ago      Up 13 minutes (Paused)                       upbeat_allen
```

处于 paused 状态的容器，可以用 [` docker unpause CONTAINER [CONTAINER...]`](https://docs.docker.com/engine/reference/commandline/unpause/) 命令来恢复到运行状态。

#### 4.2.2 终止容器

命令： [`docker[container] stop [-t|--time=[=10]] CONTAINER [CONTAINER...]`](https://docs.docker.com/engine/reference/commandline/stop/)

该命令会首先向容器发送 SIGTERM 信号，等待一段超时时间后（默认为 10 秒），再发送 SIGKILL 信号来终止容器。

终止容器后，执行 `docker container prune` 会自动清除所有处于停止状态的容器。

`docker [container]  restart` 命令会将一个运行态的容器先终止，然后再重新启动。

### 4.3 进入容器

在使用 `-d` 参数时，容器启动后会进入后台，用户无法看到容器中的信息，也无法进行操作。

如果要进入容器进行操作，推荐使用 `attach` 或 `exec` 命令

#### 4.3.1 attach 命令

attach 是 Docker 自带的命令，

命令格式为：[` docker attach [OPTIONS] CONTAINER`](https://docs.docker.com/engine/reference/commandline/attach/)

支持选项：

| Name, shorthand | Default | Description                                            |
| --------------- | ------- | ------------------------------------------------------ |
| `--detach-keys` |         | 指定退出 attach 模式的快捷键序列，默认是 CTRL-p,CTRL-q |
| `--no-stdin`    |         | 是否关闭标准输入，默认是保持打开                       |
| `--sig-proxy`   | `true`  | Proxy all received signals to the process              |



```shell
[root@VM_16_10_centos ~]# docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS                 PORTS               NAMES
e04bd9d537e4        ubuntu:latest       "bash"              13 hours ago        Up 13 hours (Paused)                       upbeat_allen
819453ec892b        ubuntu:latest       "bash"              22 hours ago        Up 22 hours                                blissful_williams
550c810bad4a        ubuntu:latest       "bash"              22 hours ago        Up 22 hours                                friendly_jepsen
[root@VM_16_10_centos ~]# docker attach upbeat_allen
You cannot attach to a paused container, unpause it first
[root@VM_16_10_centos ~]# docker attach friendly_jepsen 
root@550c810bad4a:/# 
```

然而使用 attach 命令并不是很方便。多个窗口同时 attach 到同一个容器的时候，所有的窗口都会同步显示；当某个窗口因命令阻塞时，其他窗口也无法执行操作。

如果容器已经关闭或者容器是一个后台容器，则该命令就无用武之地。

#### 4.3.2 exec 命令

推荐使用此命令对容器执行操作。

如果容器在后台启动，则可以使用 `docker exec` 在容器内执行命令。不同于 `docker attach` ,使用 `docker exec` 即使用户从终端退出，容器也不会停止执行，而使用 `docker attach` 时，如果用户从终端退出，则容器会停止运行。

命令基本格式：[` docker exec [OPTIONS] CONTAINER COMMAND [ARG...]`](https://docs.docker.com/engine/reference/commandline/exec/)

支持选项：

| Name, shorthand        | Description                                                |
| ---------------------- | ---------------------------------------------------------- |
| `--detach` , `-d`      | 在容器中后台执行命令                                       |
| `--detach-keys`        | 指定将容器切回后台的按键                                   |
| `--env` , `-e`         | 指定环境变量列表                                           |
| `--env-file`           | 从文件中读取环境变量列表                                   |
| `--interactive` , `-i` | 打开标准输入接受用户输入命令，默认值为false                |
| `--privileged`         | 是否给执行命令以高权限                                     |
| `--tty` , `-t`         | 分配伪终端                                                 |
| `--user` , `-u`        | 执行命令的用户名或 ID (format: <name\|uid>[:<group\|gid>]) |
| `--workdir` , `-w`     | 指定容器中的工作目录                                       |



```shell
[root@VM_16_10_centos ~]# docker exec -it 819453 /bin/bash
root@819453ec892b:/# ps -ef 
UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  0 Sep22 pts/0    00:00:00 bash
root         8     0  0 12:14 pts/1    00:00:00 /bin/bash
root        15     8  0 12:14 pts/1    00:00:00 ps -ef
root@819453ec892b:/# w
 12:14:51 up 482 days,  2:03,  0 users,  load average: 0.65, 0.45, 0.35
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
root@819453ec892b:/# 
```

### 4.4 删除容器

可以使用 `docker [container] rm`  命令来删除**处于终止或退出状态**的容器。

命令格式：[` docker rm [OPTIONS] CONTAINER [CONTAINER...]`](https://docs.docker.com/engine/reference/commandline/rm/)

支持选项：

| Name, shorthand    | Description                        |
| ------------------ | ---------------------------------- |
| `--force` , `-f`   | 是否强行终止并删除一个运行中的容器 |
| `--link` , `-l`    | 删除容器的连接，但保留容器         |
| `--volumes` , `-v` | 删除容器挂载的数据卷               |



默认情况下，`docker rm` 命令只能删除已经处于终止或退出状态的容器，并不能删除还处于运行状态的容器，此时，可以使用 `-f` 参数选项。

```shell
[root@VM_16_10_centos ~]# docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS                     PORTS               NAMES
e04bd9d537e4        ubuntu:latest       "bash"              13 hours ago        Up 13 hours (Paused)                           upbeat_allen
819453ec892b        ubuntu:latest       "bash"              22 hours ago        Up 22 hours                                    blissful_williams
550c810bad4a        ubuntu:latest       "bash"              22 hours ago        Exited (0) 4 minutes ago                       friendly_jepsen
[root@VM_16_10_centos ~]# docker rm 550c81
550c81
[root@VM_16_10_centos ~]# docker rm -f 819453
819453
```



### 4.5 导入和导出容器

#### 4.5.1 导出容器

导出容器是指，导出一个已经创建的容器到一个文件，不管这个容器此时是否处于运行状态。

命令：[`docker export [OPTIONS] CONTAINER`](https://docs.docker.com/engine/reference/commandline/export/)

支持选项：

| Name, shorthand   | Description                |
| ----------------- | -------------------------- |
| `--output` , `-o` | 写入到文件，而不是标准输出 |

其中，可以通过 `-o` 选项来指定导出的 tar 文件名，也可以直接通过重定向来实现。

```shell
[root@VM_16_10_centos ~]# docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS                 PORTS               NAMES
e04bd9d537e4        ubuntu:latest       "bash"              13 hours ago        Up 13 hours (Paused)                       upbeat_allen
[root@VM_16_10_centos ~]# docker export -o test.tar e04
[root@VM_16_10_centos ~]# ls
consul  nacos  redis-5.0.7  test.tar
[root@VM_16_10_centos ~]# docker export e04 > test1.tar
[root@VM_16_10_centos ~]# ls
consul  nacos  redis-5.0.7  test1.tar  test.tar
```



#### 4.5.2 导入容器

导出的文件可以通过 `docker [container] import` 命令导入变成镜像。

命令：[`docker import [OPTIONS] file|URL|- [REPOSITORY[:TAG]]`](https://docs.docker.com/engine/reference/commandline/import/)

支持选项：

| Name, shorthand    | Description                                                  |
| ------------------ | ------------------------------------------------------------ |
| `--change` , `-c`  | 导入同时执行对容器进行修改的 Dockerfile 文件                 |
| `--message` , `-m` | Set commit message for imported image                        |
| `--platform`       | [**API 1.32+**](https://docs.docker.com/engine/api/v1.32/) Set platform if server is multi-platform capable |



> 实际上，即可以使用 `docker load` 命令来导入 **镜像存储文件** 到本地镜像库，也可以使用 `docker [container] import` 来导入一个 **容器快照** 到本地镜像库。
>
> 这两者的区别是：
>
> - 容器快照文件将丢弃所有的历史记录和元数据信息（即仅保存容器当时的快照状态），因此，从容器快照文件导入时可以重新指定标签等元数据信息。
> - 镜像存储文件将保存完整记录，体积更大。



### 4.6 查看容器

#### 4.6.1 查看容器详情

命令：[`docker inspect [OPTIONS] NAME|ID [NAME|ID...]`](https://docs.docker.com/engine/reference/commandline/inspect/)

支持选项：

| Name, shorthand   | Description                                       |
| ----------------- | ------------------------------------------------- |
| `--format` , `-f` | Format the output using the given Go template     |
| `--size` , `-s`   | Display total file sizes if the type is container |
| `--type`          | Return JSON for specified type                    |

使用 `format` 参数可以只查看用户关心的数据。

```shell
[root@VM_16_10_centos ~]# docker inspect -f='{{.State.Running}}' e04
true
```



#### 4.6.2  查看容器内进程

命令：[` docker top CONTAINER [ps OPTIONS]`](https://docs.docker.com/engine/reference/commandline/top/)

```shell
[root@VM_16_10_centos ~]# docker top e04
UID                 PID                 PPID                C                   STIME               TTY                 TIME                CMD
root                14420               14403               0                   07:02               ?                   00:00:00            bash

```

#### 4.6.3 查看统计信息

`docker stats` 用来查看 CPU、内存、存储、网络等使用情况的统计信息。

命令：[`docker stats [OPTIONS] [CONTAINER...]`](https://docs.docker.com/engine/reference/commandline/stats/)

支持选项：

| Name, shorthand | Description                            |
| --------------- | -------------------------------------- |
| `--all` , `-a`  | 输出所有容器统计信息，默认仅在运行中的 |
| `--format`      | 格式化输出信息                         |
| `--no-stream`   | 不持续输出，默认自动更新持续实时结果   |
| `--no-trunc`    | 不截断输出信息                         |



```shell
CONTAINER ID        NAME                CPU %               MEM USAGE / LIMIT   MEM %               NET I/O             BLOCK I/O           PIDS
e04bd9d537e4        upbeat_allen        0.00%               540KiB / 1.795GiB   0.03%               656B / 0B           0B / 0B             1
```



### 4.7 其他容器命令

#### 4.7.1 复制文件

container cp 命令支持在容器和主机之间复制文件。

命令格式： [` docker cp [OPTIONS] CONTAINER:SRC_PATH DEST_PATH|-`](https://docs.docker.com/engine/reference/commandline/cp/)

支持选项：

| Name, shorthand        | Description                                                  |
| ---------------------- | ------------------------------------------------------------ |
| `--archive` , `-a`     | 打包模式，复制文件会带有原始的 uid/gid 信息                  |
| `--follow-link` , `-L` | 跟随软链接。当原路径为软链接时，默认只复制链接信息，使用该选项会复制链接的目标内容 |

#### 4.7.2 查看变更

container diff 查看容器内文件系统的变更。

命令格式：[` docker diff CONTAINER`](https://docs.docker.com/engine/reference/commandline/diff/)

#### 4.7.3 查看端口映射

container port 命令可以查看容器的端口映射情况。

命令格式：[` docker port CONTAINER [PRIVATE_PORT[/PROTO]]`](https://docs.docker.com/engine/reference/commandline/port/)

#### 4.7.4 更新配置

container update 命令可以更新容器的一些运行是配置，主要是一些资源限制份额。

命令格式：[`docker update [OPTIONS] CONTAINER [CONTAINER...]`](https://docs.docker.com/engine/reference/commandline/update/)

支持选项：

| Name, shorthand        | Description                                                  |
| ---------------------- | ------------------------------------------------------------ |
| `--blkio-weight`       | 更新块 IO 限制，10~1000，默认为0.代表无限制；                |
| `--cpu-period`         | 限制 CPU 调度器 CFS (Completely Fair Scheduler) 使用时间，单位为微秒，最小1000； |
| `--cpu-quota`          | 限制 CPU 调度器 CFS (Completely Fair Scheduler) 配额，单位为微秒，最小1000； |
| `--cpu-rt-period`      | 限制 CPU 调度器的实时周期，单位为微秒；                      |
| `--cpu-rt-runtime`     | 限制 CPU 调度器的实时，单位为微秒；                          |
| `--cpu-shares` , `-c`  | 限制 CPU 使用份额                                            |
| `--cpus`               | 限制 CPU 个数                                                |
| `--cpuset-cpus`        | 允许使用的 CPU 核，如0-3, 0,1                                |
| `--cpuset-mems`        | 允许使用的内存块， 如 0-3, 0,1                               |
| `--kernel-memory`      | 限制使用的内核内存                                           |
| `--memory` , `-m`      | 限制使用的内存                                               |
| `--memory-reservation` | 内存软限制                                                   |
| `--memory-swap`        | 内存加上缓存区的限制。-1 表示对缓冲区无限制；                |
| `--pids-limit`         | [**API 1.40+**](https://docs.docker.com/engine/api/v1.40/) Tune container pids limit (set -1 for unlimited) |
| `--restart`            | 容器退出后的重启策略；                                       |

#### 4.7.5 查看容器日志

交互型容器查看日志很方便，但是对于后台性容器，如果要查看日志，则可以使用 docker 提供的 `docker logs` 命令来查看。

命令格式：[`docker logs [OPTIONS] CONTAINER`](https://docs.docker.com/engine/reference/commandline/logs/)

支持选项：

| Name, shorthand       | Default | Description                                                  |
| --------------------- | ------- | ------------------------------------------------------------ |
| `--details`           |         | Show extra details provided to logs                          |
| `--follow` , `-f`     |         | 实时日志打印                                                 |
| `--since`             |         | Show logs since timestamp (e.g. 2013-01-02T13:23:37Z) or relative (e.g. 42m for 42 minutes) |
| `--tail` , `-n`       | `all`   | 控制日志的输出行数                                           |
| `--timestamps` , `-t` |         | 显示日志的输出时间                                           |
| `--until`             |         | [**API 1.35+**](https://docs.docker.com/engine/api/v1.35/) Show logs before a timestamp (e.g. 2013-01-02T13:23:37Z) or relative (e.g. 42m for 42 minutes) |



## 五、Docker 数据管理

容器中管理数据主要有两种方式：

- **数据卷（Data Volumes）：容器内数据直接映射到本地主机环境。**
- **数据卷容器（Data  Volume Containers）：使用特定容器维护数据卷。**

### 5.1 数据卷

**数据卷（Data Volumes）是一个可供容器使用的特殊目录，它将主机操作系统目录直接映射进容器**。类似于 Linux 中的 mount 行为。

特性：

- 数据卷可以在容器之间共享和重用，容器间传递数据将变得高效与方便。
- 对数据卷内数据的修改会**立马生效**，无论是容器内操作还是本地操作。
- 对数据卷的更新不会影响镜像，解耦开应用和数据。
- 卷会一直存在，**直到没有容器使用**，可以安全卸载它。

#### 5.1.1创建数据卷

命令格式：[` docker volume create [OPTIONS] [VOLUME]`](https://docs.docker.com/engine/reference/commandline/volume_create/)

支持选项：

| Name, shorthand   | Default | Description                 |
| ----------------- | ------- | --------------------------- |
| `--driver` , `-d` | `local` | Specify volume driver name  |
| `--label`         |         | Set metadata for a volume   |
| `--name`          |         | Specify volume name         |
| `--opt` , `-o`    |         | Set driver specific options |

如下命令可以在本地创建一个数据卷，创建后，可查看 `/var/lib/docker/volumes` 路径下，发现所创建的数据卷。

```shell
[root@VM_16_10_centos ~]# docker volume create -d local test
test
[root@VM_16_10_centos ~]# ls /var/lib/docker/volumes
02fe4abe7a6ca2abdcf1a530c52e4f75125e53fa53844a6b2eefec93dfe94f12  f3aebd2ac2a2730b872f4ec5d5df7eeb9ded551286f66bd1537c37d250538fc5  
test
```



相关命令：

| Command                                                      | Description                                         |
| ------------------------------------------------------------ | --------------------------------------------------- |
| [docker volume create](https://docs.docker.com/engine/reference/commandline/volume_create/) | Create a volume                                     |
| [docker volume inspect](https://docs.docker.com/engine/reference/commandline/volume_inspect/) | Display detailed information on one or more volumes |
| [docker volume ls](https://docs.docker.com/engine/reference/commandline/volume_ls/) | List volumes                                        |
| [docker volume prune](https://docs.docker.com/engine/reference/commandline/volume_prune/) | Remove all unused local volumes                     |
| [docker volume rm](https://docs.docker.com/engine/reference/commandline/volume_rm/) | Remove one or more volumes                          |



#### 5.1.2 绑定数据卷

除了使用 volume 子命令来管理数据卷外，还可以在创建容器时将主机本地任意路径挂载到容器内作为数据卷，这种形式创建的数据卷称为绑定数据卷。

在用 `docker [container] run ` 命令的时候，可以使用 `-mount` 选项使用数据卷。

`-mount` 选项支持三种类型的数据卷，包括：

- volume：普通数据卷，映射到主机 `/var/lib/docker/volumes` 路径下；
- bind：绑定数据卷，映射到主机指定路径下；
- tmpfs：临时数据卷，只存在于内存中。



```shell
 docker run --read-only --mount type=volume,target=/icanwrite busybox touch /icanwrite/here
 docker run -t -i --mount type=bind,src=/data,dst=/data busybox sh
```

更多使用 `-v` 选项。

```shell
 docker  run  -v `pwd`:`pwd` -i -t  ubuntu pwd
```



### 5.2 数据卷容器

数据卷容器是一个**专门用来挂在数据卷的容器**。该容器主要是提供给其他容器引用和使用。所谓的数据卷容器，就是一个普通的容器。

1、创建数据卷容器 dbdata,并在其中创建一个数据卷挂载到 /dbdata。

```shell
[root@VM_16_10_centos _data]# docker run -it -v /dbdata --name dbdata ubuntu
root@91cab02c1287:/# ls
bin  boot  dbdata  dev  etc  home  lib  lib32  lib64  libx32  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var

```



2、然后可以在其他容器中使用 `--volumes-from` 来挂载 dbdata 容器中的数据卷。

```shell
[root@VM_16_10_centos _data]# docker run -it --volumes-from dbdata --name db1 ubuntu
root@7020432a1de4:/# la
.dockerenv  bin  boot  dbdata  dev  etc  home  lib  lib32  lib64  libx32  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
root@7020432a1de4:/# cd dbdata/
root@7020432a1de4:/dbdata# touch test
root@7020432a1de4:/dbdata# ls
test
root@7020432a1de4:/dbdata# exit
[root@VM_16_10_centos _data]# docker exec -it dbdata /bin/bash
root@91cab02c1287:/# ls
bin  boot  dbdata  dev  etc  home  lib  lib32  lib64  libx32  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
root@91cab02c1287:/# cd dbdata/
root@91cab02c1287:/dbdata# ls
test

```

可见， dbdata 和 db1 都挂载同一个数据卷到相同的 /dbdata 目录，三个容器任何一方在该目录下的写入，其他容器都可以看到。



可以多次使用 `--volumes-from` 参数来从多个容器挂载多个数据卷，还可以从其他已经挂载了容器卷的容器来挂载数据卷；

> 使用 `--volumes-from` 参数所挂载数据卷的容器自身并不需要保持在运行状态。



如果删除了挂载的容器（包括 dbdata、db1） ，数据卷并不会自动删除。如果要删除一个数据卷，必须在删除最后一个还挂载着它的容器时显示使用 `docker rm -v` 命令来指定同时删除关联的容器。

利用数据卷容器可以实现数据的备份与恢复。

### 5.4 利用数据卷容器来迁移数据（备份、恢复）

#### 5.4.1 备份

使用下面的命令来备份 dbdata 数据卷容器内的数据卷：

```shell
[root@VM_16_10_centos _data]# docker run --volumes-from dbdata -v $(pwd):/backup --name worker ubuntu tar cvf /backup/backup.tar /dbdata
tar: Removing leading `/' from member names
/dbdata/
/dbdata/test
```

命令解释：

- 使用 `--volumes-from dbdata` 参数来让 worker 容器挂载 dbdata 容器的数据卷（即 dbdata 数据卷）；
- 使用 `-v $(pwd):/backup` 参数来挂载本地的当前目录到 worker 容器的 /backup 目录；
- worker 容器启动后，使用 `tar cvf /backup/backup.tar /dbdata`  命令将 /dbdata 下内容备份为容器内的 /backup/backup.tar, 由于已经设置将当前目录映射到容器的 /backup  目录，因此可以在当前目录中看到。



#### 5.4.2 恢复

如果要恢复数据到一个容器，可以按照下面的操作。

首先创建一个带有数据卷的容器，这个容器就是要使用恢复的数据的容器。这里创建一个 nginx 容器，如下：

```dockerfile
docker run -itd -p 8088:8088 -v /usr/share/nginx/html/ --name nginx3 nginx
```

数据恢复需要一个临时容器，如下：

```dockerfile
docker run --volumes-from nginx3 -v $(pwd):/backup nginx tar xvf /backup/backup.tar
```

命令解释：

1、首先还是使用 `--volumes-from` 参数连接上备份容器。

2、然后将当前目录映射到容器的 /backup 目录下。

3、然后执行解压操作，将 backup.tar 文件解压。解压文件位置描述是一个容器内的地址，但是该地址已经映射到宿主机中的当前目录了，因此这里要解压缩的文件实际上就是宿主机当前目录下的文件。



## 六、端口映射和容器互联

Docker 除了通过网络访问外，还有两个很方便的功能来满足服务访问的基本需求：

- 允许映射容器内应用的服务端口到本地宿主主机。
- 互联机制实现多个容器间通过容器名来快速访问。

### 6.1 端口映射实现容器访问

1、 从外部访问容器应用

在启动容器的时候，如果不指定对应参数，在容器外部是无法通过网络来访问容器内的网络应用和服务的。

当容器运行一些网络应用，可以通过 `-P` 或 `-p` 参数来指定端口映射。

- 当使用 `-P`（大写）标记时，Docker 会随机映射一个 4900~49900 的端口到内部容器开放的网络端口。
- `-p`(小写) 则可以指定要映射的端口，并且在一个指定端口上只可以绑定一个容器。支持的格式有 IP:HostPort:ContainerPort | IP：：ContainerPort | HostPort:ContainerPort。

2、映射所有接口地址

使用 `HostPort:ContainerPort` 将本地端口映射到容器的端口，多次使用 `-p` 可以绑定多个端口

3、映射到指定地址的指定端口

使用 `IP:HostPort:ContainerPort` 格式指定映射使用一个特定的地址。

4、映射到指定地址的任意端口

使用 `IP::ContainerPort` 绑定 IP 的任意端口到容器的端口。还可以使用 udp 标记来指定 udp 端口。

5、查看映射端口配置

使用 `docker port` 来查看当前映射的端口配置，也可以查看到绑定的地址。



### 6.3 互联机制实现便捷互访

容器的互联是一种让多个容器中的应用进行快速交互的方式。它会在源和接收容器之间创建连接关系，接收容器可以通过容器名快速访问到源容器，而不用指定具体的 IP 地址。

1、 自定义容器命名

连接系统依据容器的名称来执行。通过 `--name` 来指定。

2、  容器互联

使用 `--link` 参数可以让容器之间安全地进行交互。

`--link` 的参数格式为 `--link name:alias` ，其中 name 是要链接的容器的名称，alias 是别名。



## 七、使用 Dockfile 创建镜像

### 7.1 基本结构

Dockerfile 由一行行命令语句组成，并且支持以 # 开头的注释行。

一般而言，Dockerfile 主题内容分为四部分：基础镜像信息、维护者信息、镜像操作指令和容器启动时执行指令。



### 7.2 [指令说明](https://docs.docker.com/engine/reference/builder/)

**配置指令**：

| 指令        | 说明                               |
| ----------- | ---------------------------------- |
| ARG         | 定义创建镜像过程中使用的变量       |
| FROM        | 指定所创建镜像的基础镜像           |
| LABEL       | 为生成的镜像添加元数据标签信息     |
| EXPOSE      | 声明镜像内服务监听的端口           |
| ENTRYPOINT  | 指定镜像的默认入口命令             |
| VOLUME      | 创建一个数据卷挂载点               |
| USER        | 指定运行容器时的用户名或 UID       |
| WORKDIR     | 配置工作目录                       |
| ONBUILD     | 创建子镜像时指定自动执行的操作指令 |
| STOPSIGNAL  | 指定退出的信号值                   |
| HEALTHCHECK | 配置所启动容器如何进行健康检查     |
| SHELL       | 指定默认 shell 类型                |

**操作指令**：

| 指令 | 说明                         |
| ---- | ---------------------------- |
| RUN  | 运行指定命令                 |
| CMD  | 启动容器是指定默认执行的命令 |
| ADD  | 添加内容到镜像               |
| COPY | 复制内容到镜像               |

#### 7.2.1 配置指令

##### 1. [ARG](https://docs.docker.com/engine/reference/builder/#arg)

定义创建镜像过程中使用的变量。是唯一允许在 FROM 指令前使用的指令。

格式为 `ARG <name>=[=<default value>]`。

在执行 docker build 时，可以通过 `--build-arg [=]` 来为变量赋值。当镜像编译成功后， ARG 指定的变量将不再存在（ENV 指定的变量将在镜像中保留）。

Docker 内置了一些镜像创建变量，用户可以直接使用而无须声明，包括（不区分大小写）HTTP_PORXY、HTTPS_PROXY、FTP_PROXY、NO_PROXY。

##### 2. [FROM](https://docs.docker.com/engine/reference/builder/#from)

指定所创建镜像的基础镜像。

格式为 `FROM <iamge> [As <name>]` 或 `FROM <image>:<tag> [AS <name>]` 或 `FROM <image>@<digest> [AS <name>]`。

**任何 Dockerfile 中第一条指令必须为 FROM 指令**。并且，如果在同一个 Dockerfile 中创建多个镜像时，可以使用多个 FROM 指令（每个镜像一次）。

```dockerfile
ARG  CODE_VERSION=latest
FROM base:${CODE_VERSION}
CMD  /code/run-app
```

##### 3. [LABEL](https://docs.docker.com/engine/reference/builder/#label)

LABEL 指令可以为生成的镜像添加元数据标签信息。

格式为 `LABEL <key>=<value> <key>=<value> <key>=<value> ...`

```dockerfile
LABEL
Learn more about the "LABEL" Dockerfile command.
 "com.example.vendor"="ACME Incorporated"
LABEL com.example.label-with-value="foo"
LABEL version="1.0"
LABEL description="This text illustrates \
that label-values can span multiple lines."
```

##### 4. [EXPOSE](https://docs.docker.com/engine/reference/builder/#expose)

声明镜像内服务监听的端口。

格式为 `EXPOSE <port> [<port>/<protocol>...]`



```dockerfile
EXPOSE 80/tcp
EXPOSE 80/udp
```

注意：该指令只是起到**声明作用**，并不会自动完成端口映射。



##### 5. [ENV](https://docs.docker.com/engine/reference/builder/#env)

指定环境变量，在镜像生成过程中会被后续 RUN 指令使用，在镜像启动的容器中也会存在。

命令格式为：`ENV <key>=<value> ...`

```dockerfile
ENV MY_NAME="John Doe"
ENV MY_DOG=Rex\ The\ Dog
ENV MY_CAT=fluffy
```

注意当一条 ENV 指令中同时为多个环境变量赋值并且值也是从环境变量读取时，会为变量都赋值都厚再更新。

如下指令，最终结果为 key1=values1 key2=value2

```dockerfile
ENV key1=value2
ENV key1=value1 keys=${key1}
```



##### 6. [ENTRYPOINT](https://docs.docker.com/engine/reference/builder/#entrypoint)

指定镜像的默认入口命令，该入口命令会在启动容器时作为根命令执行，所有传入值作为该命令的参数。

支持两种格式：

- `ENTRYPOINT ["executable", "param1", "param2"]`

- `ENTRYPOINT command param1 param2`

此时， CMD 指令指定值将作为根命令的参数。

每个 Dockerfile 中只能有一个 ENTRYPOINT，当指定多个时，只有最后一个起效。

在运行时，可以被 --entrypoint 参数覆盖掉。



##### 7. [VOLUME](https://docs.docker.com/engine/reference/builder/#volume)

创建一个数据卷挂载点。

格式为：`VOLUME ["/data"]`

运行容器时可以从本地主机或其他容器挂在数据卷，一般用来存放数据库和需要保持的数据等。

##### 8. [USER](https://docs.docker.com/engine/reference/builder/#user)

指定运行容器时的用户名或 UID，后续的 RUN 等指令也会使用指定的用户身份。

格式为：

- `USER <user>[:<group>]`
- `USER <UID>[:<GID>]`

当服务不需要管理员权限时，可以通过该命令指定运行用户，并且可以在 Dockerfile 中创建所需要的用户。

例如：

```dockerfile
RUn groupadd -r postgres && useradd --no-log-init -r -g postgres postgres
```

##### 9. [WORKDIR](https://docs.docker.com/engine/reference/builder/#workdir)

为后续的 RUN、CMD、ENTRYPOINT 指令配置工作目录。

格式：`WORKDIR /path/to/workdir`

可以使用多个WORKDIR 命令，后续命令如果参数时相对路径，则会基于之前命令指定的路径。

```dockerfile
WORKDIR /a
WORKDIR b
WORKDIR c
RUN pwd
```

则最终路径为/a/b/c。因此，为了避免出错，推荐使用绝对路径。

##### 10. [ONBUILD](https://docs.docker.com/engine/reference/builder/#onbuild)

指定当基于所生成镜像创建子镜像时，自动执行的操作指令。

格式：`ONBUILD <INSTRUCTION>`

例如，使用如下的 Dockerfile 创建父镜像 ParentImage,指定 ONBUILD 指令：

```dockerfile
#Dockerfile for ParentImage
[...]
ONBUILD ADD . /app/src
ONBUILD RUN /user/local/bin/python-build --dir /app/src
[...]
```

使用 docker build 命令创建子镜像 ChildImage 时（FROM ParentImage）, 会首先执行 ParentImage 中配置的 ONBUILD 指令。

```dockerfile
# Dockerfile for ChildImage
FROM ParentImage
```

等于在 ChildImage 的 Dockerfile 中添加了如下指令：

```dockerfile
# Automatically run the following wehn building ChildImage
ADD . /app/src
RUn /user/local/bin/python-build --dir /app/src
```

由于 ONBUILD 指令是隐式执行的，推荐在使用它的镜像标签中进行标注，例如 ruby:2.1-onbuild。

ONBUILD指令在创建专门用于自动编译、检查等操作的基础镜像时，十分有用。

##### 11. [STOPSIGNAL](/user/local/bin/python-build --dir /app/src)

指定所创建镜像启动的容器接收退出的信号值：

```dockerfile
STOPSIGNAL signal
```

##### 12. [HEALTHCHECK](https://docs.docker.com/engine/reference/builder/#healthcheck)

配置所启动容器如何进行健康检查（如何判断健康与否）。

格式有两种：

- `HEALTHCHECK [OPTIONS] CMD command` ：根据所执行命令返回值是否为 0 来判断。
- `HEALTHCHECK NONE` ：禁止基础镜像中的健康检查。

OPTIONS 支持如下参数：

- `--interval=DURATION` (default: `30s`)：过多久检查一次；
- `--timeout=DURATION` (default: `30s`)：每次检查等待结果的超时；
- `--start-period=DURATION` (default: `0s`)：检查开始的时间延迟；
- `--retries=N` (default: `3`)：如果失败了，重试几次才最终确定失败；

##### 13. [SHELL](https://docs.docker.com/engine/reference/builder/#shell)

指定其他命令使用 shell 时的默认 shell 类型：

`SHELL ["executable", "parameters"]`

默认值为 ["/bin/sh","-c"]。

> 对于 Windows 系统， Shell 路径中使用了 “\” 作为分隔符，建议在 Dockerfile 开头添加 #escape=' 来指定转义符。 



#### 7.2.2 操作指令

##### 1. [RUN](https://docs.docker.com/engine/reference/builder/#run)

运行指定命令。

格式：

- `RUN <command>` (*shell* form, the command is run in a shell, which by default is `/bin/sh -c` on Linux or `cmd /S /C` on Windows)
- `RUN ["executable", "param1", "param2"]` (*exec* form)：被解析为 JSON 数组。

指定使用其他终端可以通过第二种方式实现，如 RUN ["/bin/bash"，‘“-c”,"echo hello"]。

每条 RUN 指令将在当前镜像基础上执行指定命令，并提交为新的镜像层。当命令较长时，可以使用 \ 来换行。

##### 2.[CMD](https://docs.docker.com/engine/reference/builder/#cmd)

CMD 指令用来指定启动容器时模式执行的命令。

支持三种格式：

- `CMD ["executable","param1","param2"]` (*exec* form, this is the preferred form)
- `CMD ["param1","param2"]` (as *default parameters to ENTRYPOINT*)
- `CMD command param1 param2` (*shell* form)

每个 Dockerfile 只能有一条 CMD 命令。如果指定了多条命令，只有最后一条会被执行。

如果用户启动容器时手动指定了运行的命令（作为 run 命令的参数），则会覆盖掉 CMD指定的命令。

##### 3. [ADD](https://docs.docker.com/engine/reference/builder/#add)

添加内容到镜像。

格式为：

```dockerfile
ADD [--chown=<user>:<group>] <src>... <dest>
ADD [--chown=<user>:<group>] ["<src>",... "<dest>"]
```

该命令将复制指定的 <src> 路径下内容到容器中的 <dest> 路径下。

其中 <src> 可以是 Dockerfile 所在目录的一个相对路径（文件或目录）；也可以是一个 URL；还可以是一个 tar 文件（自动解压为目录）<dest> 可以是镜像内绝对路径，或者相对于工作目录（WORKDIR）的相对路径。

##### 4. [COPY](https://docs.docker.com/engine/reference/builder/#copy)

复制内容到镜像。

格式为：

```dockerfile
COPY [--chown=<user>:<group>] <src>... <dest>
COPY [--chown=<user>:<group>] ["<src>",... "<dest>"]
```

复制本地主机的 <src> （为 Dockerfile 所在目录的相对路径，文件或目录）下内容到镜像中的 <dest>。目标路径不存在时，会自动创建。



### 7.3 创建镜像

编写完成 Dockerfile 之后，可以通过 [docker [image] build](https://docs.docker.com/engine/reference/commandline/build/) 命令来创建镜像。

基本格式为：

```dockerfile
 docker build [OPTIONS] PATH | URL | -
```

该命令将读取指定路径下（包括子目录）的 Dockerfile ，并将该路径下所有数据作为上下文（Context）发送给 Docker 服务端。Docker 服务端在校验 Dockerfile 格式通过后，逐条执行其中定义的指令，碰到 ADD、COPY 和 RUN 指令会生成一层新的镜像。最终如果创建镜像成功，会返回最终镜像的 ID。

#### 7.3.1 [命令选项](https://docs.docker.com/engine/reference/commandline/build/#options)

| 选项                      | 说明                                                         |
| ------------------------- | ------------------------------------------------------------ |
| `--add-host`              | 添加自定义的主机名到 IP 的映射                               |
| `--build-arg`             | 添加创建时的变量                                             |
| `--cache-from`            | 使用指定镜像作为缓存源                                       |
| `--cgroup-parent`         | 继承的上层 cgroup                                            |
| `--compress`              | 使用 gzip 来压缩创建上下文数据                               |
| `--cpu-period`            | 分配的 CFS 调度器时长                                        |
| `--cpu-quota`             | CFS 调度器总份额                                             |
| `--cpu-shares` , `-c`     | CPU 权重                                                     |
| `--cpuset-cpus`           | 多 CPU 允许使用的 CPU                                        |
| `--cpuset-mems`           | 多 CPU 允许使用的内存                                        |
| `--disable-content-trust` | 不进行镜像校验，默认为 true                                  |
| `--file` , `-f`           | Dockerfile 名称 (Default is 'PATH/Dockerfile')               |
| `--force-rm`              | Always remove intermediate containers                        |
| `--iidfile`               | 将镜像 ID 写入到文件                                         |
| `--isolation`             | 容器隔离机制                                                 |
| `--label`                 | 配置镜像的元数据                                             |
| `--memory` , `-m`         | 限制使用的内存量                                             |
| `--memory-swap`           | 限制内存和缓存的总量: '-1' 不限制                            |
| `--network`               | 指定 RUN 命令时的网络模式                                    |
| `--no-cache`              | 创建镜像时不使用缓存                                         |
| `--output` , `-o`         | 输出路径 (format: type=local,dest=path)                      |
| `--platform`              | 指定平台类型                                                 |
| `--progress`              | Set type of progress output (auto, plain, tty). Use plain to show container output |
| `--pull`                  | 总是尝试获取镜像的最新版本                                   |
| `--quiet` , `-q`          | Suppress the build output and print image ID on success，default is true |
| `--rm`                    | Remove intermediate containers after a successful build      |
| `--secret`                | [**API 1.39+**](https://docs.docker.com/engine/api/v1.39/) Secret file to expose to the build (only if BuildKit enabled): id=mysecret,src=/local/secret |
| `--security-opt`          | Security options                                             |
| `--shm-size`              | Size of /dev/shm                                             |
| `--squash`                | [**experimental (daemon)**](https://docs.docker.com/engine/reference/commandline/dockerd/#daemon-configuration-file)[**API 1.25+**](https://docs.docker.com/engine/api/v1.25/) Squash newly built layers into a single new layer |
| `--ssh`                   | [**API 1.39+**](https://docs.docker.com/engine/api/v1.39/) SSH agent socket or keys to expose to the build (only if BuildKit enabled) (format: default\|<id>[=<socket>\|<key>[,<key>]]) |
| `--stream`                | 持续获取创建的上下文                                         |
| `--tag` , `-t`            | 指定镜像的标签列表                                           |
| `--target`                | 指定创建的目标阶段                                           |
| `--ulimit`                | 指定 ulimit 的配置                                           |



#### 7.3.2 选择父镜像

大部分情况下，生成新的镜像都需要通过 FROM 指令来指定父镜像。父镜像是生成镜像的基础，会直接影响到所生成镜像的大小和功能。

用户可以选择两种镜像作为父镜像，一种是所谓的基础镜像（baseimage），另外一种是普通的镜像（往往由第三方创建，基于基础镜像）。

基础镜像比较特殊，其 Dockerfile 中往往不存在 FROM 指令，或者基于 scratch 镜像（FROM scratch），这意味着其在整个镜像树中处于根的位置。



#### 7.3.3 使用 .dockerignore 文件

可以通过 .dockerignore 文件（每行添加一条匹配模式）来让 Docker 忽略匹配路径或文件，在创建镜像时不将无关数据发送到服务端。

```dockerfile
# .dockerignore 文件中可以定义忽略模式
*/temp*
*/*/temp*
tmp?
-*
Dockerfile
!README.md
```

- dockerignore 文件中模式语法支持 Golang 风格的路径正则格式；
- “ * ” 表示任意多个字符；
- “ ？“ 表示单个字符；
- “ ！” 表示不匹配

#### 7.3.4 多步骤创建（Multi-stage build）

自 17.05 版本开始， Docker 支持多步骤镜像创建，可以精简最终生成的镜像大小。

对于需要变异的应用（C、Go 或 Java 语言等），通常情况下至少需要准备两个环境的 Docker 镜像。

- 编译环境镜像：包括完整的编译引擎、依赖库等，往往比较庞大。作用是编译应用为二进制文件。
- 运行环境镜像：利用编译好的二进制文件，运行应用，由于不需要编译环境，体积比较小。


 使用多步骤创建，可以在保证最终生成的运行环境镜像保持精简的情况下，使用单一的 Dockerfile ，降低维护复杂度。



