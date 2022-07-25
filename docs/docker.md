# Docker

## 什么是 Docker

Docker 是一个开源的容器化平台。开发人员可使用 Docker 将应用程序打包到容器中，容器是标准化的可执行组件，可以将应用程序源代码与在任何环境中运行该代码所需的操作系统 (OS) 库和依赖项相结合。容器简化了分布式应用程序的交付，并且随着组织转向云原生开发和混合多云环境而变得越来越流行。

开发人员可以在不使用 Docker 平台的情况下创建容器，但该平台可以使构建、部署和管理容器的过程变得更便捷且更安全。Docker 本质上是一个工具包，支持开发人员通过单个 API，使用简单的命令和省力的自动化技术来构建、部署、运行、更新和停止容器。

## 容器与虚拟机的区别

虚拟机是物理机的配置模拟出来的操作系统，预分配的资源一旦启动就会把资源全部耗尽。而容器不会耗尽资源，它会把宿主机的一些内核等资源和其他容器共享资源。
虚拟机的镜像往往占用空间大，而且启动速度很慢。容器占用空间少，容器启动方便且快速。

与虚拟机相比，容器的部署、配置和重启过程更快且更简单。这使得容器非常适合在持续集成和持续交付 (CI/CD) 管道中使用，并且更适合采取敏捷和 DevOps 实践的开发团队。

但是这并不意味着虚拟机可以被容器完全代替，虚拟机隔离性比较好，可以跨平台，例如，在 Linux 上能允许 Windows 应用。

Docker 本质上是利用容器虚拟化技术。最早的容器技术是 2008 年发布的 LXC (Linux Container)，但是不够好用，Docker 基于 LXC 做了很多对应用开发友好的封装和改进，真正带动了容器化技术的流行。
随着 Docker 自身的技术架构演进，放弃了使用 LXC，转而使用自研的 libcontainer 来和 Linux kernel 交互，之后为了符合 OCI (Open Container Initiative) 标准，又封装重构为现在著名的 runC。

## 为何选择使用 Docker

作为一种新兴的虚拟化方式，Docker 跟传统的虚拟化方式相比具有众多的优势。

### 更高效的利用系统资源

由于容器不需要进行硬件虚拟以及运行完整操作系统等额外开销，Docker 对系统资源的利用率更高。无论是应用执行速度、内存损耗或者文件存储速度，都要比传统虚拟机技术更高效。因此，相比虚拟机技术，一个相同配置的主机，往往可以运行更多数量的应用。

### 更快速的启动时间

传统的虚拟机技术启动应用服务往往需要数分钟，而 Docker 容器应用，由于直接运行于宿主内核，无需启动完整的操作系统，因此可以做到秒级、甚至毫秒级的启动时间。大大的节约了开发、测试、部署的时间。

### 一致的运行环境

开发过程中一个常见的问题是环境一致性问题。由于开发环境、测试环境、生产环境不一致，导致有些 bug 并未在开发过程中被发现。而 Docker 的镜像提供了除内核外完整的运行时环境，确保了应用运行环境一致性，从而不会再出现 「这段代码在我机器上没问题啊」 这类问题。

### 持续交付和部署

对开发和运维（DevOps）人员来说，最希望的就是一次创建或配置，可以在任意地方正常运行。

使用 Docker 可以通过定制应用镜像来实现持续集成、持续交付、部署。开发人员可以通过 Dockerfile 来进行镜像构建，并结合 持续集成（Continuous Integration）系统进行集成测试，而运维人员则可以直接在生产环境中快速部署该镜像，甚至结合 持续部署（Continuous Delivery/Deployment）系统进行自动部署。

而且使用 Dockerfile 使镜像构建透明化，不仅仅开发团队可以理解应用运行环境，也方便运维团队理解应用运行所需条件，帮助更好的生产环境中部署该镜像。

### 更轻松的迁移

由于 Docker 确保了执行环境的一致性，使得应用的迁移更加容易。Docker 可以在很多平台上运行，无论是物理机、虚拟机、公有云、私有云，甚至是笔记本，其运行结果是一致的。因此用户可以很轻易的将在一个平台上运行的应用，迁移到另一个平台上，而不用担心运行环境的变化导致应用无法正常运行的情况。

### 更轻松的维护和扩展

Docker 使用的分层存储以及镜像的技术，使得应用重复部分的复用更为容易，也使得应用的维护更新更加简单，基于基础镜像进一步扩展镜像也变得非常简单。此外，Docker 团队同各个开源项目团队一起维护了一大批高质量的 官方镜像，既可以直接在生产环境使用，又可以作为基础进一步定制，大大的降低了应用服务的镜像制作成本。

## Docker 工具和术语

### Dockerfile

每个 Docker 容器都从一个简单的文本文件开始，其中包含有关如何构建 Docker 容器镜像的指示信息。Dockerfile 将自动执行 Docker 镜像创建过程。它本质上是一个命令行界面 (CLI) 指令列表，Docker 引擎将运行这些指令来组装镜像。

### Docker Image

Docker 镜像包含可执行的应用程序源代码以及在将应用程序代码作为容器运行时所需的所有工具、库和依赖项。当运行 Docker 镜像时，它会成为容器的一个（或多个）实例。

虽然可以从头开始构建 Docker 镜像，但大多数开发人员都是从公共存储库中提取 Docker 镜像。可以通过一个基本镜像来创建多个 Docker 镜像，并且它们将共享一个公共堆栈。

Docker 镜像由层组成，每个层对应于该镜像的一个版本。每当开发人员对镜像进行更改时，都会创建一个新的顶层，该顶层将替换之前的顶层作为镜像的当前版本。之前的层会被保存下来，以便执行回滚或在其他项目中进行复用。

每次通过 Docker 镜像创建容器时，都会创建另一个称为“容器层”的新层。对容器所做的更改（例如添加或删除文件）仅保存到容器层中，并且仅在容器运行时才会出现。这种迭代式镜像创建过程可以提高整体效率，因为可以仅从一个基本镜像中运行多个实时的容器实例，如果这样做，这些实例将使用一个公共堆栈。

### Docker Container

Docker 容器是实时运行的 Docker 镜像实例。Docker 镜像是只读文件，而容器是实时的瞬态可执行内容。用户可以与它们进行交互，管理员可以使用 docker 命令调整其设置和条件。

### Docker Hub

Docker Hub 是 Docker 镜像的公共存储库，自称为“世界上最大的容器镜像库和社区”。它拥有超过 100,000 个来自商业软件供应商、开源项目和个人开发人员的容器镜像。它包括由 Docker, Inc. 生成的镜像、属于 Docker Trusted Registry 的经过认证的镜像以及成千上万的其他镜像。

所有 Docker Hub 用户都可以随意共享其镜像。他们还可以从 Docker 文件系统下载预定义的基础镜像，用作任何容器化项目的起点。

### Docker Daemon

Docker 守护进程是在操作系统上运行的服务，例如 Microsoft Windows、Apple MacOS 或 iOS。此服务使用来自客户机的命令为您创建和管理 Docker 镜像，充当 Docker 实现的控制中心。

### Docker Registry

Docker 镜像注册服务是一个用于 Docker 镜像的可扩展开源存储和分发系统。它使您可以使用标记进行标识来跟踪存储库中的镜像版本。可使用 git 版本控制工具来完成此操作。

## Docker 存储驱动

镜像包含操作系统完整的 root 文件系统，其体积往往是庞大的，因此在 Docker 设计时，就充分利用 Union FS 的技术，将其设计为分层存储的架构。
所以严格来说，镜像并非是像一个 ISO 那样的打包文件，镜像只是一个虚拟的概念，其实际体现并非由一个文件组成，而是由一组文件系统组成，或者说，由多层文件系统联合组成。

镜像构建时，会一层层构建，前一层是后一层的基础。每一层构建完就不会再发生改变，后一层上的任何改变只发生在自己这一层。
比如，删除前一层文件的操作，实际不是真的删除前一层的文件，而是仅在当前层标记为该文件已删除。在最终容器运行的时候，虽然不会看到这个文件，但是实际上该文件会一直跟随镜像。
因此，在构建镜像的时候，需要额外小心，每一层尽量只包含该层需要添加的东西，任何额外的东西应该在该层构建结束前清理掉。

分层存储的特征还使得镜像的复用、定制变的更为容易。甚至可以用之前构建好的镜像作为基础层，然后进一步添加新的层，以定制自己所需的内容，构建新的镜像。

Docker 最开始采用 AUFS 作为文件系统，也得益于 AUFS 分层的概念，实现了多个容器可以共享同一个镜像。
但由于 AUFS 未并入 Linux 内核，且只支持 Ubuntu，考虑到兼容性问题，在 Docker 0.7 版本中引入了存储驱动。
目前，Docker支持 AUFS、Btrfs、Device mapper、OverlayFS、ZFS、VFS 等存储驱动。
就如 Docker 官网上说的，没有单一的驱动适合所有的应用场景，要根据不同的场景选择合适的存储驱动，才能有效的提高Docker的性能。

- Copy-on-Write：写时复制，所有驱动都用到该技术，其原理是在需要写时才去复制，这个是针对已有文件的修改场景。比如基于一个镜像启动多个容器，如果为每个容器都去分配一个镜像一样的文件系统，那么将会占用大量的磁盘空间。
  而 CoW 技术可以让所有的容器共享镜像的文件系统，所有数据都从镜像中读取，只有当要对文件进行写操作时，才从镜像里把要写的文件复制到自己的文件系统进行修改，所以无论有多少个容器共享同一个镜像。
  所做的写操作都是对从镜像中复制到自己的文件系统中的复本上进行，并不会修改镜像的源文件，且多个容器操作同一个文件，会在每个容器的文件系统里生成一个复本，每个容器修改的都是自己的复本，相互隔离，相互不影响，可以有效的提高磁盘的利用率。

- Allocate-on-Demand：用时分配，是用在原本没有这个文件的场景，只有在要新写入一个文件时才分配空间，这样可以提高存储资源的利用率。比如启动一个容器，并不会为这个容器预分配一些磁盘空间，而是当有新文件写入时，才按需分配新空间。

[About storage drivers](https://docs.docker.com/storage/storagedriver/)

## Docker 网络模式

### Bridge 模式

通常在 Docker 进程启动时，会在主机上创建一个名为 docker0 的虚拟网桥，此主机上启动的 Docker 容器会连接到这个虚拟网桥上。

虚拟网桥的工作方式和物理交换机类似，这样主机上的所有容器就通过交换机连在了一个二层网络中。

从 docker0 子网中分配一个 IP 给容器使用，并设置 docker0 的 IP 地址为容器的默认网关。在主机上创建一对虚拟网卡 veth pair 设备，
Docker 将 veth pair 设备的一端放在新创建的容器中，并命名为 eth0（容器的网卡），另一端放在主机中，以 vethxxx 这样类似的名字命名，并将这个网络设备加入到 docker0 网桥中。可以通过 brctl show 命令查看。

Bridge 模式是 docker 的默认网络模式，使用`docker run -p`时，实际是在 iptables 做了 DNAT 规则，实现端口转发功能。

### Host 模式

如果启动容器的时候使用 Host 模式，那么这个容器将不会获得一个独立的 Network Namespace，而是和宿主机共用一个 Network Namespace。
容器将不会虚拟出自己的网卡，配置自己的 IP 等，而是使用宿主机的 IP 和端口。但是，容器的其他方面，如：文件系统、进程列表等还是和宿主机隔离的。

### Container 模式

Container 模式指定新创建的容器和已经存在的一个容器共享一个 Network Namespace，而不是和宿主机共享。
新创建的容器不会创建自己的网卡，配置自己的 IP，而是和一个指定的容器共享 IP、端口范围等。同样，两个容器除了网络方面，其他的如文件系统、进程列表等还是隔离的。
两个容器的进程可以通过 lo 网卡设备通信。

### None 模式

如果启动容器的时候使用 Host 模式，那么这个容器将拥有自己的 Network Namespace，但是，并不为 Docker 容器进行任何网络配置。
也就是说，这个 Docker 容器没有网卡、IP、路由等信息。需要我们自己为 Docker 容器添加网卡、配置 IP 等。

## Docker 数据持久化

TODO

## Docker 命令

### 容器生命周期命令

#### create

创建一个新的容器但不启动它。

```shell
docker create [OPTIONS] IMAGE [COMMAND] [ARG...]
```

OPTIONS：

- `-a`：指定标准输入输出内容类型，支持`STDIN/STDOUT/STDERR`
- `-d`：后台运行容器，并返回容器ID
- `-i`：以交互模式运行容器，通常与`-t`同时使用
- `-P`：随机端口映射，容器内部端口随机映射到主机的端口
- `-p`：指定端口映射，格式为：`宿主端口:容器端口`
- `-t`：为容器重新分配一个伪输入终端，通常与`-i`同时使用
- `--name`：为容器指定一个名称
- `--dns`：指定容器使用的DNS服务器，默认和宿主一致
- `--dns-search`：指定容器DNS搜索域名，默认和宿主一致
- `-h`：指定容器的`hostname`
- `-e`：设置环境变量
- `--env-file`：从指定文件读入环境变量
- `--cpuset`：绑定容器到指定CPU运行
- `-m`:设置容器使用内存最大值
- `--net`：指定容器的网络连接类型，支持`bridge/host/none/container`
- `--link`：添加链接到另一个容器
- `--expose`：开放一个端口或一组端口
- `--volume, -v`：绑定一个卷

#### run

创建一个新的容器并运行一个命令。

```shell
docker run [OPTIONS] IMAGE [COMMAND] [ARG...]
```

OPTIONS：同`create`。

#### start

启动一个或多个已经被停止的容器。

```shell
docker start [OPTIONS] CONTAINER [CONTAINER...]
```

#### stop

停止一个或多个运行中的容器。

```shell
docker stop [OPTIONS] CONTAINER [CONTAINER...]
```

#### restart

重启一个或多个容器。

```shell
docker restart [OPTIONS] CONTAINER [CONTAINER...]
```

#### kill

结束一个运行中的容器。

```shell
docker kill [OPTIONS] CONTAINER [CONTAINER...]
```

OPTIONS：

- `-s`：向容器发送一个信号

#### rm

删除一个或多个容器。

```shell
docker rm [OPTIONS] CONTAINER [CONTAINER...]
```

OPTIONS：

- `-f`：通过`SIGKILL`信号强制删除一个运行中的容器
- `-l`：移除容器间的网络连接，而非容器本身
- `-v`：删除与容器关联的卷

#### pause

暂停容器中所有的进程。

```shell
docker pause CONTAINER [CONTAINER...]
```

#### unpause

恢复容器中所有的进程。

```shell
docker unpause CONTAINER [CONTAINER...]
```

#### exec

在运行的容器中执行命令。

```shell
docker exec [OPTIONS] CONTAINER COMMAND [ARG...]
```

OPTIONS：

- `-d`：分离模式，在后台运行
- `-i`：保持`STDIN`打开即使没有连接
- `-t`：分配一个伪终端

### 容器命令

#### ps

列出容器。

```shell
docker ps [OPTIONS]
```

OPTIONS：

- `-a`：显示所有的容器，包括未运行的
- `-f`：根据条件过滤显示的内容
- `--format`：指定返回值的模板文件
- `-l`：显示最近创建的容器
- `-n`：列出最近创建的`n`个容器
- `--no-trunc`：不截断输出
- `-q`：静默模式，只显示容器编号
- `-s`：显示总的文件大小

#### inspect

获取容器的元数据。

```shell
docker inspect [OPTIONS] NAME|ID [NAME|ID...]
```

OPTIONS：

- `-f`：指定返回值的模板文件
- `-s`：显示总的文件大小
- `--type`：为指定类型返回 JSON

#### top

查看容器中运行的进程信息，支持 ps 命令参数。

```shell
docker top [OPTIONS] CONTAINER [ps OPTIONS]
```

#### attach

连接到正在运行中的容器。

```shell
docker attach [OPTIONS] CONTAINER
```

#### events

从服务器获取实时事件。

```shell
docker events [OPTIONS]
```

OPTIONS：

- `-f`：根据条件过滤事件
- `--since`：从指定的时间戳后显示所有事件
- `--until`：流水时间显示到指定的时间为止

#### logs

获取容器的日志。

```shell
docker logs [OPTIONS] CONTAINER
```

OPTIONS：

- `-f`：跟踪日志输出
- `--since`：显示某个开始时间的所有日志
- `-t`：显示时间戳
- `--tail`：仅列出最新N条容器日志

#### wait

阻塞运行直到容器停止，然后打印出它的退出代码。

```shell
docker wait [OPTIONS] CONTAINER [CONTAINER...]
```

#### export

将文件系统作为一个tar归档文件导出到STDOUT。

```shell
docker export [OPTIONS] CONTAINER
```

OPTIONS：

- `-o`：将输入内容写到文件

#### port

列出指定的容器的端口映射，或者查找将PRIVATE_PORT NAT到面向公众的端口。

```shell
docker port [OPTIONS] CONTAINER [PRIVATE_PORT[/PROTO]]
```

#### commit

从容器创建一个新的镜像。

```shell
docker commit [OPTIONS] CONTAINER [REPOSITORY[:TAG]]
```

OPTIONS：

- `-a`：提交的镜像作者
- `-c`：使用 Dockerfile 指令来创建镜像
- `-m`：提交时的说明文字
- `-p`：在 commit 时，将容器暂停

#### cp

用于容器与主机之间的数据拷贝。

```shell
docker cp [OPTIONS] CONTAINER:SRC_PATH DEST_PATH
```

```shell
docker cp [OPTIONS] SRC_PATH CONTAINER:DEST_PATH
```

OPTIONS：

- `-L`：保持源目标中的链接

#### diff

检查容器里文件结构的更改。

```shell
docker diff [OPTIONS] CONTAINER
```

### 镜像仓库命令

#### login

登录到 Docker 镜像仓库，如果未指定镜像仓库地址，默认为官方仓库 Docker Hub。

```shell
docker login [OPTIONS] [SERVER]
```

OPTIONS：

- `-u`：用户名
- `-p`：密码

#### logout

登出 Docker 镜像仓库，如果未指定镜像仓库地址，默认为官方仓库 Docker Hub。

```shell
docker logout [OPTIONS] [SERVER]
```

OPTIONS：

- `-u`：用户名
- `-p`：密码

#### pull

从镜像仓库中拉取或者更新指定镜像。

```shell
docker pull [OPTIONS] NAME[:TAG|@DIGEST]
```

OPTIONS：

- `-a`：拉取所有镜像
- `--disable-content-trust`：忽略镜像的校验，默认开启

#### push

将本地的镜像上传到镜像仓库，要先登录到镜像仓库。

```shell
docker push [OPTIONS] NAME[:TAG]
```

OPTIONS：

- `--disable-content-trust`：忽略镜像的校验，默认开启

#### search

搜索镜像。

```shell
docker search [OPTIONS] TERM
```

OPTIONS：

- `--automated`：只列出 automated 类型的镜像
- `--no-trunc`：显示完整的镜像描述
- `-f <clause>`：列出收藏数不小于指定值的镜像
    - `NAME`：镜像仓库源的名称
    - `DESCRIPTION`：镜像的描述
    - `OFFICIAL`：是否 docker 官方发布
    - `stars`：不小于指定 stars 的镜像
    - `AUTOMATED`：自动构建

### 本地镜像命令

#### images

列出本地镜像。

```shell
docker images [OPTIONS] [REPOSITORY[:TAG]]
```

OPTIONS：

- `-a`：列出本地所有的镜像（含中间映像层，默认情况下，过滤掉中间映像层）
- `--digests`：显示镜像的摘要信息
- `-f`：显示满足条件的镜像
- `--format`：指定返回值的模板文件
- `--no-trunc`：显示完整的镜像信息
- `-q`：只显示镜像ID

#### rmi

删除本地一个或多个镜像。

```shell
docker rmi [OPTIONS] IMAGE [IMAGE...]
```

OPTIONS：

- `-f`：强制删除
- `--no-prune`：不移除该镜像的过程镜像，默认移除

#### tag

标记本地镜像，将其归入某一仓库。

```shell
docker tag [OPTIONS] IMAGE[:TAG] [REGISTRYHOST/][USERNAME/]NAME[:TAG]
```

#### build

使用 Dockerfile 创建镜像。

```shell
docker build [OPTIONS] PATH | URL | -
```

OPTIONS：

- `--build-arg=[]`：设置镜像创建时的变量
- `--cpu-shares`：设置 cpu 使用权重
- `--cpu-period`：限制 CPU CFS 周期
- `--cpu-quota`：限制 CPU CFS 配额
- `--cpuset-cpus`：指定使用的 CPU ID
- `--cpuset-mems`：指定使用的内存 ID
- `--disable-content-trust`：忽略校验，默认开启
- `-f`：指定要使用的 Dockerfile 路径
- `--force-rm`：设置镜像过程中删除中间容器
- `--isolation`：使用容器隔离技术
- `--label=[]`：设置镜像使用的元数据
- `-m`：设置内存最大值
- `--memory-swap`：设置 Swap 的内存最大值
- `--no-cache`：创建镜像的过程不使用缓存
- `--pull`：尝试去更新镜像的新版本
- `--quiet, -q`：安静模式，成功后只输出镜像 ID
- `--rm`：设置镜像成功后删除中间容器
- `--shm-size`：设置/dev/shm的大小，默认值是64M
- `--ulimit`：ulimit 配置
- `--squash`：将 Dockerfile 中所有的操作压缩为一层
- `--tag, -t`：镜像的名字及标签，可以设置多个标签
- `--network`：在构建期间设置RUN指令的网络模式

#### history

查看指定镜像的创建历史。

```shell
docker history [OPTIONS] IMAGE
```

OPTIONS：

- `-H`：以可读的格式打印镜像大小和日期，默认为`true`
- `--no-trunc`：显示完整的提交记录
- `-q`：仅列出提交记录ID

#### save

将指定镜像保存成 tar 归档文件。

```shell
docker save [OPTIONS] IMAGE [IMAGE...]
```

OPTIONS：

- `-o`：输出到的文件

#### load

导入使用 save 命令导出的镜像。

```shell
docker load [OPTIONS]
```

OPTIONS：

- `--input, -i`： 指定导入的文件，代替`STDIN`
- `--quiet, -q`： 精简输出信息

#### import

从归档文件中创建镜像。

```shell
docker import [OPTIONS] file|URL|- [REPOSITORY[:TAG]]
```

OPTIONS：

- `-c`：使用 Docker 指令创建镜像
- `-m`：提交时的说明文字