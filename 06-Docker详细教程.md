# 阶段6 Docker 容器——零基础实操教程

> 本教程面向完全没有接触过容器的同学。每一个知识点都按"**是什么 → 为什么用 → 怎么用**"三层来讲，并且给出**可以直接复制粘贴运行**的完整命令、参数解释和预期结果。请务必跟着敲一遍，不要只看。

---

## 目录

- [第1章 为什么要用容器](#第1章-为什么要用容器)
- [第2章 Docker 核心概念](#第2章-docker-核心概念)
- [第3章 安装 docker-ce 并配置加速器](#第3章-安装-docker-ce-并配置加速器)
- [第4章 常用命令详解](#第4章-常用命令详解)
- [第5章 镜像分层与写时复制原理](#第5章-镜像分层与写时复制原理)
- [第6章 Dockerfile 详解](#第6章-dockerfile-详解)
- [第7章 数据卷 volume](#第7章-数据卷-volume)
- [第8章 Docker 网络](#第8章-docker-网络)
- [第9章 docker-compose 编排](#第9章-docker-compose-编排)
- [第10章 完整项目实战](#第10章-完整项目实战)
- [本章项目验收](#本章项目验收)

---

## 第1章 为什么要用容器

### 1.1 传统部署的痛点（是什么）

在没有容器之前，一个应用要上线，运维同学通常要这样做：

1. 买服务器或申请虚拟机；
2. 在服务器上装操作系统、装依赖（JDK、Python、Node、数据库……）；
3. 把代码传上去，配置环境变量、修改配置文件；
4. 反复调试"为什么在我电脑上能跑，服务器上跑不起来"。

这个过程有几个经典的坑：

| 痛点 | 具体表现 |
| --- | --- |
| 环境不一致 | 开发用 macOS，测试用 Windows，线上用 Linux，同一个代码三种结果 |
| 依赖冲突 | 两个应用一个要 JDK8 一个要 JDK11，装在同一台机器上互相打架 |
| 部署繁琐 | 每台新机器都要手动重装一遍依赖，费时且容易出错 |
| 资源浪费 | 为了隔离，一台机器装多个虚拟机，每个都要跑一整套操作系统，内存 CPU 被操作系统吃掉了大半 |

**容器的解决方案**：把"应用 + 它运行所需的一切依赖（库、配置、运行时）"打包成一个标准化的镜像，无论在谁的机器上，只要能跑 Docker，这个镜像跑出来的结果就一模一样。

### 1.2 容器和虚拟机的区别（配对比表）

这是面试和实操中最高频的问题。先看对比表：

| 对比项 | 虚拟机（VM） | 容器（Container） |
| --- | --- | --- |
| 隔离级别 | 硬件级隔离（Hypervisor 层） | 进程级隔离（共享宿主机内核） |
| 操作系统 | 每个 VM 有完整独立 Guest OS | 共享宿主机内核，只打包应用层 |
| 启动速度 | 分钟级（要启动完整 OS） | 秒级甚至毫秒级（只是启动进程） |
| 镜像大小 | 几 GB ~ 几十 GB | 几十 MB ~ 几百 MB |
| 资源占用 | 高（每个 VM 都要分走内存 CPU） | 低（只占用应用实际需要的量） |
| 性能损耗 | 有（虚拟化硬件转发） | 几乎无（直接调用内核） |
| 可移植性 | 较差（依赖虚拟化平台格式） | 极强（镜像统一标准，随处可跑） |
| 典型场景 | 需要不同内核、强隔离、多系统共存 | 微服务、快速扩缩容、CI/CD |

**一句话总结**：虚拟机是"一栋楼里给每户人家盖独立的一整套水电系统"；容器是"一栋楼里共用总水管总电路，每家只装自己需要的开关和插座"。容器轻、快、省，但隔离强度不如虚拟机。

> 补充：Linux 容器依赖 Linux 内核特性（namespace 做隔离、cgroup 做资源限制）。所以在 Windows/macOS 上跑 Docker 时，Docker Desktop 会在背后自动起一个 Linux 虚拟机来承载容器。

### 1.3 动手练习（第1章）

1. 用自己的话向同事解释：为什么"在我电脑上能跑"到了服务器就挂？
2. 默写容器与虚拟机的三个关键区别（隔离级别、启动速度、资源占用）。

---

## 第2章 Docker 核心概念

### 2.1 三大核心对象（是什么 + 为什么用）

| 概念 | 是什么 | 类比（生活化） |
| --- | --- | --- |
| 镜像（Image） | 一个只读的、打包了应用及其运行环境的模板 | 一个"系统安装包/光盘" |
| 容器（Container） | 镜像运行起来之后的一个实例，可读可写 | 用光盘装好、正在运行的一台"电脑" |
| 仓库（Repository） | 存放镜像的地方，按名字分类 | 软件下载站上的某个"软件栏目" |
| Registry | 仓库的服务器（承载多个仓库） | 整个"软件下载站" |

它们之间的关系链条：

```
Registry（如 Docker Hub、阿里云）
   └── Repository（如 library/nginx）
          └── Image:tag（如 nginx:1.25）
                 └── run 之后得到 Container
```

### 2.2 镜像（Image）

- **是什么**：由多层只读文件系统叠加而成的模板，包含了运行一个应用所需的全部内容：文件系统、依赖库、环境变量、默认启动命令。
- **为什么用**：它是"一次构建、到处运行"的载体。构建一次，可以在任何装了 Docker 的机器上跑出一模一样的结果。
- **怎么用**：通过 `docker pull` 拉取、`docker build` 构建、`docker images` 查看。

### 2.3 容器（Container）

- **是什么**：镜像的一次运行实例。在镜像的只读层之上叠加一层**可写层**，应用运行时产生的所有变化都写在这层里。
- **为什么用**：同一个镜像可以启动 N 个互不干扰的容器，像"同一个安装包装出很多台电脑"。
- **怎么用**：`docker run` 启动，`docker ps` 查看，`docker stop/start/rm` 管理生命周期。

### 2.4 仓库与 Registry

- **是什么**：Registry 是存镜像的服务器，Docker Hub（`docker.io`）是最知名的公共 Registry；仓库（Repository）是 Registry 里按名字归类的镜像集合，例如 `nginx`、`mysql`、`myapp/web`。
- **为什么用**：镜像要能分发、共享、备份，就需要一个集中存放的地方。
- **怎么用**：`docker pull 仓库名:标签` 拉取，`docker push 仓库名:标签` 推送（推送到自己的私有 Registry 时需要先登录）。

### 2.5 动手练习（第2章）

1. 在纸上画出 `Registry → Repository → Image → Container` 的关系图，并各举一个真实例子。
2. 说出 `nginx:1.25` 里，`nginx` 是什么、`1.25` 是什么。

---

## 第3章 安装 docker-ce 并配置加速器

> 本节以 **CentOS 7 / CentOS 8（兼容 RHEL 系列）** 为例，使用阿里云 yum 源安装社区版 docker-ce。请用 root 用户或 `sudo` 执行。

### 3.1 卸载旧版本（如机器上装过）

```bash
# 卸载可能存在的旧版 docker（没有装过也没关系，命令会提示"未安装"）
sudo yum remove -y docker \
    docker-client \
    docker-client-latest \
    docker-common \
    docker-latest \
    docker-latest-logrotate \
    docker-logrotate \
    docker-engine
```

- **参数解释**：`remove -y` 表示卸载并自动回答"yes"；后面跟的是旧版 Docker 相关包名。
- **作用/原理**：清理历史残留，避免新旧版本冲突。

### 3.2 安装必要依赖

```bash
sudo yum install -y yum-utils
```

- **作用/原理**：`yum-utils` 提供了 `yum-config-manager` 命令，后面添加阿里云源时要用到。

### 3.3 添加阿里云 yum 源（配置国内镜像加速器的前置）

```bash
# 添加阿里云 Docker CE 官方仓库
sudo yum-config-manager \
    --add-repo \
    https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

- **参数解释**：`--add-repo <url>` 表示向 yum 配置中加入一个新的仓库文件，URL 指向阿里云镜像站上的 docker-ce.repo。
- **作用/原理**：把 docker-ce 的软件源指向阿里云，下载速度快且稳定，避免访问国外 docker.com 超时。

> 说明：这里配置的"阿里云 yum 源"解决的是 **安装 docker 软件本身** 的下载加速；后面 3.6 的"镜像加速器"解决的是 **拉取镜像（如 nginx）** 的下载加速，两者不是一回事，别混淆。

### 3.4 安装 docker-ce 及配套组件

```bash
sudo yum install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

- **字段解释**：
  - `docker-ce`：Docker 引擎本体；
  - `docker-ce-cli`：命令行工具（我们敲的 `docker` 命令）；
  - `containerd.io`：底层容器运行时；
  - `docker-buildx-plugin`：构建镜像的增强插件；
  - `docker-compose-plugin`：compose v2 插件（安装它之后直接用 `docker compose` 命令）。
- **作用/原理**：一次性装齐引擎、命令行、运行时和构建/编排插件。

预期结果：最后几行类似 `Complete!`，表示安装成功。

### 3.5 启动 Docker 并设置开机自启

```bash
# 启动 docker 守护进程
sudo systemctl start docker

# 设置开机自启动
sudo systemctl enable docker

# 查看运行状态（看到 active (running) 即成功）
sudo systemctl status docker
```

- **参数解释**：`systemctl start` 立即启动服务；`systemctl enable` 写入开机自启；`status` 查看状态。
- **作用/原理**：Docker 是"守护进程（daemon）"架构，`docker` 命令只是客户端，背后必须有一个常驻后台的服务进程在运行。`start` 就是拉起这个守护进程。

验证是否安装成功：

```bash
sudo docker version
sudo docker info
```

- **作用/原理**：`docker version` 显示客户端和服务端版本（两边都有版本号才说明守护进程正常）；`docker info` 显示引擎的详细配置（存储驱动、运行时等）。

### 3.6 配置国内镜像加速器

由于默认从 Docker Hub（国外）拉镜像很慢甚至超时，需要配置镜像加速器。

```bash
# 创建/覆盖 daemon.json 配置文件
sudo tee /etc/docker/daemon.json > /dev/null <<-'EOF'
{
  "registry-mirrors": [
    "https://docker.m.daocloud.io",
    "https://dockerproxy.com",
    "https://mirror.ccs.tencentyun.com"
  ]
}
EOF
```

- **命令解释**：`tee /etc/docker/daemon.json > /dev/null` 把下面 `EOF` 之间的内容写入配置文件；`<<-'EOF'` 是 heredoc 写法，`EOF` 是结束标记。
- **字段解释**：`registry-mirrors` 是一个数组，存放多个加速器地址，拉镜像时会按顺序尝试。上面的地址示例（DaoCloud、dockerproxy、腾讯云）以当前可用的为准，可按需替换成自己阿里云账号分配的个人加速地址（见第10章）。
- **作用/原理**：Docker 拉取镜像时先查这个 mirror 列表，命中就走国内 CDN，速度快很多。

使配置生效：

```bash
sudo systemctl daemon-reload          # 重新加载 systemd 配置
sudo systemctl restart docker         # 重启 docker 使 daemon.json 生效
```

验证加速器是否生效：

```bash
sudo docker info | grep -A5 "Registry Mirrors"
```

预期结果类似：

```
 Registry Mirrors:
  https://docker.m.daocloud.io/
  https://dockerproxy.com/
  https://mirror.ccs.tencentyun.com/
```

### 3.7 动手练习（第3章）

1. 完整执行一遍 3.1~3.6，最终 `docker version` 能同时看到 Client 和 Server。
2. 用 `docker info` 找到 `Registry Mirrors` 段落，确认加速器已写入。
3. 执行 `sudo docker run hello-world`，观察它是否自动拉取并打印成功信息（这是官方最经典的验证命令）。

---

## 第4章 常用命令详解

> 本章命令均给出**完整可运行示例**，建议逐个敲一遍，对照预期结果理解。

### 4.1 docker pull —— 拉取镜像

```bash
docker pull nginx:1.25
```

- **参数解释**：`nginx` 是仓库名，`:1.25` 是标签（tag，通常是版本号）。省略标签时默认是 `latest`。
- **作用/原理**：从 Registry 下载镜像到本地。镜像按层下载，每层有独立的下载进度条；已经存在的层会显示 `Already exists` 跳过。

预期结果：出现多层 `Pull complete`，最后 `Status: Downloaded newer image for nginx:1.25`。

### 4.2 docker images —— 查看本地镜像

```bash
docker images
```

预期结果（表格形式）：

| REPOSITORY | TAG | IMAGE ID | CREATED | SIZE |
| --- | --- | --- | --- | --- |
| nginx | 1.25 | 61395b4c586d | 2 weeks ago | 187 MB |
| hello-world | latest | d2c94e258dcb | 8 months ago | 13.3 kB |

- **字段解释**：`REPOSITORY` 仓库名、`TAG` 标签、`IMAGE ID` 镜像唯一标识、`SIZE` 镜像大小。
- **作用/原理**：列出本地所有镜像，方便确认拉取是否成功、查 IMAGE ID 用于后续删除。

### 4.3 docker run —— 启动容器（最核心命令）

`docker run` 参数非常多，先把最常用的记牢：

```bash
docker run -d \
  --name mynginx \
  -p 8080:80 \
  -v /data/html:/usr/share/nginx/html \
  -e TZ=Asia/Shanghai \
  --restart=always \
  nginx:1.25
```

- **参数逐个解释**：
  - `-d`：detach，后台运行（不加 -d 会占用前台终端，Ctrl+C 可能直接停掉容器）。
  - `--name mynginx`：给容器起个名字，方便后续 `docker ps`/`stop`/`logs` 定位；不起名 Docker 会随机起名。
  - `-p 8080:80`：端口映射，`宿主机端口:容器内端口`。访问宿主机 8080 就转发到容器内 nginx 监听的 80。
  - `-v /data/html:/usr/share/nginx/html`：挂载卷，`宿主机目录:容器内目录`。宿主机目录内容会同步到容器里。
  - `-e TZ=Asia/Shanghai`：`-e` 设置环境变量，这里把容器时区设为上海。
  - `--restart=always`：重启策略，容器退出或宿主机重启后自动拉起，常用于常驻服务。
  - 最后 `nginx:1.25` 是镜像名。

预期结果：输出一行 64 位十六进制的**容器 ID**，例如 `7f3c9a...`。

再给一个**前台交互式**的例子（进入容器操作时常用）：

```bash
docker run -it --rm alpine:latest sh
```

- **参数解释**：`-i` 保持标准输入打开、`-t` 分配一个伪终端（`-it` 合起来就是"给我一个可以敲命令的终端"）；`--rm` 退出后自动删除容器；`sh` 是进入容器后要运行的命令。
- **作用/原理**：拉一个极小的 Linux（alpine 只有几 MB），进入其 shell 体验"容器就是一个进程环境"。输入 `exit` 退出。

### 4.4 docker ps —— 查看容器

```bash
docker ps          # 只看正在运行的
docker ps -a       # 看所有容器（含已停止的）
```

预期结果（`docker ps -a`）：

| CONTAINER ID | IMAGE | COMMAND | CREATED | STATUS | PORTS | NAMES |
| --- | --- | --- | --- | --- | --- | --- |
| 7f3c9a2b... | nginx:1.25 | "/docker-entrypoint.…" | 2 min | Up 2 min | 0.0.0.0:8080->80/tcp | mynginx |
| a1b2c3d4... | alpine:latest | "sh" | 5 min | Exited (0) | | sharp_galileo |

- **字段解释**：`STATUS` 显示 `Up`（运行中）或 `Exited (0)`（已退出，0 是正常退出码）；`PORTS` 显示端口映射关系；`NAMES` 容器名。
- **作用/原理**：`-a` 显示所有（否则只显示运行中的），排查"容器怎么不见了"时一定要加 `-a`。

### 4.5 docker logs —— 查看日志

```bash
docker logs mynginx            # 查看全部日志
docker logs -f mynginx         # 实时跟踪日志（Ctrl+C 退出跟踪，不停止容器）
docker logs --tail 50 mynginx  # 只看最后 50 行
docker logs --since 5m mynginx # 只看最近 5 分钟的日志
```

- **参数解释**：`-f` follow 跟随输出；`--tail N` 只显示末尾 N 行；`--since` 按时间过滤。
- **作用/原理**：容器把标准输出/标准错误写进日志，`logs` 把它们取出来，是排查应用问题的第一入口。

### 4.6 docker exec -it —— 进入运行中的容器

```bash
docker exec -it mynginx /bin/bash
```

- **参数解释**：`exec` 在**已运行**的容器里再启动一个进程；`-it` 分配交互终端；`/bin/bash` 要执行的 shell。
- **作用/原理**：`run` 是启动新容器，`exec` 是进入已存在的容器。区别很重要：一个启动、一个进入。进入后可以 `ls /`、`cat /etc/os-release` 查看容器内部。
- **注意**：nginx 官方镜像默认有 `/bin/bash`；但 alpine 这类精简镜像只有 `/bin/sh`，此时应写成 `docker exec -it xxx /bin/sh`。

退出：输入 `exit`（只退出这个 exec 的终端，**不会**停止容器）。

### 4.7 start / stop / restart —— 启停容器

```bash
docker stop mynginx     # 优雅停止：先发 SIGTERM，等 10 秒超时再强制杀
docker start mynginx    # 启动一个已停止（已存在）的容器
docker restart mynginx  # 重启 = stop + start
```

- **作用/原理**：`stop` 与 `start` 针对**已存在**的容器，不创建不删除；`run` 每次都会新建容器。所以日常是"第一次用 run 创建，之后用 start/stop 管理"。

### 4.8 docker rm / rmi —— 删除容器和镜像

```bash
docker rm mynginx                 # 删除一个已停止的容器
docker rm -f mynginx              # 强制删除（即使正在运行）
docker rm $(docker ps -aq)        # 一次性删除所有容器（-q 只输出 ID）
docker rmi nginx:1.25             # 删除镜像（先确保没有容器在使用它）
docker rmi -f 61395b4c586d        # 按 IMAGE ID 强制删除镜像
```

- **字段解释**：`rm` 删容器，`rmi`（remove image）删镜像；`-q` 只列 ID；`-f` 强制。
- **作用/原理**：清理资源。注意顺序：镜像被容器占用时无法直接删除，要先删容器。

### 4.9 docker inspect —— 查看详细元数据

```bash
docker inspect mynginx
docker inspect -f '{{.NetworkSettings.IPAddress}}' mynginx   # 只取容器 IP
docker inspect -f '{{.Mounts}}' mynginx                       # 只看挂载信息
```

- **参数解释**：`-f` 用 Go 模板格式抽取指定字段。
- **作用/原理**：输出一个巨大的 JSON，包含网络、挂载、环境变量、重启策略等全部细节，排查疑难问题时的"万能钥匙"。

### 4.10 docker cp —— 宿主机与容器互传文件

```bash
docker cp /root/index.html mynginx:/usr/share/nginx/html/   # 宿主机 → 容器
docker cp mynginx:/etc/nginx/nginx.conf /root/             # 容器 → 宿主机
```

- **参数解释**：`docker cp 源 目标`，源和目标都可以是 `容器名:路径` 或宿主机路径。
- **作用/原理**：在运行期临时拷贝文件用，比如传一个测试页面进去看效果。

### 4.11 docker commit —— 把容器保存为新镜像

```bash
# 先进入容器做点改动（比如装个 vim）
docker exec -it mynginx /bin/bash
apt-get update && apt-get install -y vim
exit

# 把改动提交成新镜像
docker commit -m "add vim" -a "ops" mynginx mynginx:vim
```

- **参数解释**：`-m` 提交说明、`-a` 作者；`mynginx` 源容器名，`mynginx:vim` 新镜像名:标签。
- **作用/原理**：把容器的可写层"固化"成一个新镜像。**不推荐在生产用**（过程不可复现、无法审计），正确做法是用 Dockerfile（第6章），但它是理解"容器可写层"的好工具。

### 4.12 docker save / load —— 镜像的导出与导入

```bash
docker save -o nginx-1.25.tar nginx:1.25   # 把镜像打成 tar 文件
docker load -i nginx-1.25.tar              # 从 tar 文件导入镜像
```

- **参数解释**：`save -o 文件 镜像` 导出；`load -i 文件` 导入。
- **作用/原理**：在**不能联网**的内网机器间传输镜像时用，相当于"把镜像打包成文件再拷过去"。与 `export/import`（针对容器）不同，`save/load` 针对镜像且保留分层和历史。

### 4.13 docker tag / push —— 打标签与推送

```bash
docker tag nginx:1.25 myrepo.com/library/nginx:1.25   # 复制一个带新名字的标签
docker push myrepo.com/library/nginx:1.25             # 推送到指定 Registry
```

- **参数解释**：`tag 源镜像 目标镜像名`；`push 镜像名`。
- **作用/原理**：`tag` 只是给同一镜像多贴一个名字标签（不复制数据）；`push` 上传到 Registry，通常需要先 `docker login`。第10章会完整演示推送到阿里云。

### 4.14 动手练习（第4章）

1. 用 `docker run -d --name web -p 9090:80 nginx` 启动一个容器，浏览器访问 `http://服务器IP:9090` 应看到 nginx 欢迎页。
2. 用 `docker exec -it web /bin/bash` 进入容器，`echo hello > /usr/share/nginx/html/index.html`，刷新页面观察变化，理解"容器内部文件系统可写"。
3. 用 `docker cp` 把该 index.html 拷回宿主机，再用 `docker inspect -f '{{.NetworkSettings.IPAddress}}' web` 查看容器 IP。
4. 依次练习 `stop` → `start` → `restart` → `logs -f` → `rm -f`，最后 `docker ps -a` 确认已清理。

---

## 第5章 镜像分层与写时复制原理

### 5.1 镜像分层（是什么 + 为什么用）

Docker 镜像不是一个完整的大文件，而是由**多个只读层（layer）**叠加而成的。

以 Dockerfile 为例，几乎每条指令都会产生一层：

```dockerfile
FROM ubuntu:22.04        # 第1层：基础系统
RUN apt-get update       # 第2层：更新源
RUN apt-get install nginx  # 第3层：装 nginx
COPY app/ /app/          # 第4层：拷贝应用代码
```

用一张示意图理解（自上而下）：

```
┌──────────────────────────┐
│ 容器可写层（Container Layer）  ← 应用运行时的新增/修改都在这里
├──────────────────────────┤
│ 只读层4：COPY app/           │
├──────────────────────────┤
│ 只读层3：RUN apt install     │
├──────────────────────────┤
│ 只读层2：RUN apt update      │
├──────────────────────────┤
│ 只读层1：FROM ubuntu         │
└──────────────────────────┘
```

**为什么用分层**：

1. **复用**：多个镜像共享相同的基础层，只保存一份。例如 10 个应用都基于 ubuntu:22.04，那个基础层只占一次磁盘。
2. **加速构建与传输**：没变化的层不需要重新构建、不需要重新下载（这就是拉取时 `Already exists` 的原因）。
3. **版本管理**：每次修改只新增/替换变化的那一层，而不是整个镜像重来。

验证分层：

```bash
docker history nginx:1.25
```

预期结果：按从新到旧列出每一层（`<missing>` 表示该层已合并或来源不可见），能看到 `IMAGE`、`CREATED BY`、`SIZE` 三列，直观感受"一个镜像由一串层组成"。

### 5.2 写时复制（Copy-on-Write，CoW）

**是什么**：容器启动时，Docker 在所有只读层之上加一个**可写层（Container Layer）**。应用要读取文件时，直接读下面只读层的内容；只有当应用要**修改**某个文件时，才把该文件**复制**到可写层，并在副本上修改——下面的原始只读层永远不动。

**为什么用**：

- 同一镜像启动的 N 个容器，每个都只拥有自己独立的可写层，互不影响；
- 只读层可以安全地被所有容器共享，磁盘和内存都省；
- 容器删除后，可写层随之销毁，数据不留痕（所以持久数据必须靠第7章的数据卷）。

**举例说明**：

1. 容器读 `/usr/share/nginx/html/index.html` → 直接从只读层读，不做复制；
2. 容器往该文件写入新内容 → Docker 先把该文件复制到可写层，再在副本上写；
3. 删除容器 → 可写层销毁，镜像里的原始文件依然完好。

这就是为什么"改容器里的文件，镜像不变"的根本原理。

### 5.3 动手练习（第5章）

1. `docker history nginx:1.25` 观察分层，数一数大概有多少层。
2. 启动两个来自同一镜像的容器，在 A 容器里改文件，进 B 容器确认文件未变，体会"可写层独立"。
3. 用 `docker commit` 把 A 容器提交成新镜像，再 `docker history` 对比，看是不是多出了一层。

---

## 第6章 Dockerfile 详解

### 6.1 Dockerfile 是什么、为什么用

- **是什么**：一个纯文本文件，里面写着一系列**构建镜像的指令**，告诉 Docker 如何一步步从基础镜像做出目标镜像。
- **为什么用**：相比 `docker commit` 手改容器，Dockerfile 把构建过程**代码化**，可复现、可审计、可进版本库（git），是 CI/CD 的基石。
- **怎么用**：写一个名为 `Dockerfile` 的文件，然后 `docker build -t 镜像名 .` 构建。

### 6.2 指令逐个详解

#### FROM —— 指定基础镜像

```dockerfile
FROM ubuntu:22.04
```

- **作用**：指定构建的起点，每个 Dockerfile 第一条有效指令必须是 `FROM`。
- **原理**：它决定了镜像的"地基"（操作系统和已有环境）。

#### RUN —— 构建时执行命令

```dockerfile
RUN apt-get update && apt-get install -y nginx
```

- **作用**：在**构建镜像阶段**执行 shell 命令，结果固化到新的层里。
- **原理**：每条 RUN 都会新增一层，所以多个命令尽量用 `&&` 串成一条，减少层数。
- **为什么用 `&&`**：如果写成两条 RUN，会多一层且如果中间某步失败，后面的层还在，镜像不干净。

#### COPY —— 拷贝宿主机文件到镜像

```dockerfile
COPY ./app /usr/share/nginx/html
```

- **作用**：把构建上下文（通常是 Dockerfile 所在目录）里的文件拷贝进镜像。
- **原理**：`COPY 源路径 目标路径`。源路径是宿主机相对路径，目标是镜像内绝对路径。

#### ADD —— 拷贝（功能比 COPY 更强的兼容指令）

```dockerfile
ADD https://example.com/file.tar.gz /opt/
ADD ./app.tar.gz /opt/
```

- **作用**：除了像 COPY 一样拷贝文件，还支持两点：源可以是 **URL**；源是本地 **tar 压缩包时自动解压**。
- **原理**：官方建议"能用 COPY 就用 COPY，只有需要 URL 下载或自动解压时才用 ADD"，因为 ADD 的隐式行为容易造成意外。

#### WORKDIR —— 设置工作目录

```dockerfile
WORKDIR /app
```

- **作用**：为后面的 RUN/CMD/ENTRYPOINT/COPY 等指令设置默认工作目录（相当于 `cd`）。
- **原理**：目录不存在时会自动创建；用绝对路径更稳妥。后续 RUN 里就不用每条都写完整路径了。

#### EXPOSE —— 声明容器监听端口

```dockerfile
EXPOSE 80
```

- **作用**：声明容器运行时监听的端口。
- **原理**：**EXPOSE 只是文档性声明，并不会真的把端口映射到宿主机**。真正对外暴露要靠 `docker run -p` 或 compose 的 `ports`。它主要起"自文档"作用，也影响 `docker run -P`（随机映射）。

#### ENV —— 设置环境变量

```dockerfile
ENV NODE_ENV=production \
    APP_PORT=3000
```

- **作用**：定义环境变量，构建期和运行期都可用。
- **原理**：`ENV 键=值`，可一次写多个。运行容器时可用 `docker run -e` 覆盖。

#### ARG —— 构建参数

```dockerfile
ARG VERSION=1.0
RUN echo "building version $VERSION"
```

- **作用**：定义**只在构建期**存在的变量，可通过 `docker build --build-arg VERSION=2.0` 传入。
- **原理**：ARG 与 ENV 的区别——ARG 构建完就没了，不会进入最终镜像；ENV 会一直留在镜像里。适合放"构建时需要的临时值"。

#### CMD —— 容器启动时的默认命令

```dockerfile
CMD ["nginx", "-g", "daemon off;"]
```

- **作用**：指定容器启动时默认执行的命令。
- **原理**：`docker run 镜像 其他命令` 时，`其他命令` 会**覆盖** CMD。一个 Dockerfile 只有最后一条 CMD 生效。

#### ENTRYPOINT —— 容器入口命令

```dockerfile
ENTRYPOINT ["nginx"]
CMD ["-g", "daemon off;"]
```

- **作用**：指定容器的主命令（入口）。
- **原理**：`docker run 镜像 参数` 时，这些参数会**追加**到 ENTRYPOINT 后面，而不是覆盖。所以 ENTRYPOINT + CMD 常配合使用。

### 6.3 CMD 与 ENTRYPOINT 的区别（重点，配对比表）

| 对比项 | CMD | ENTRYPOINT |
| --- | --- | --- |
| 定位 | 默认命令 | 固定入口命令 |
| 能否被 `docker run` 参数覆盖 | 会被完全覆盖 | 不会被覆盖，参数只会**追加** |
| 常见用法 | 给镜像一个默认行为，方便用户替换 | 固定应用入口，让用户只能传参 |
| 配合形式 | 单独用 | 常与 CMD 配合，CMD 充当默认参数 |

**两种写法的对比演示**（以运行 nginx 为例）：

写法一：只用 CMD

```dockerfile
FROM nginx
CMD ["nginx", "-g", "daemon off;"]
```

此时执行 `docker run mynginx` 正常启动 nginx；但如果执行 `docker run mynginx echo hi`，则 CMD 被覆盖，容器只执行 `echo hi`，nginx 没启动。

写法二：ENTRYPOINT + CMD

```dockerfile
FROM nginx
ENTRYPOINT ["nginx"]
CMD ["-g", "daemon off;"]
```

此时执行 `docker run mynginx`，实际执行 `nginx -g 'daemon off;'`；执行 `docker run mynginx -t`，实际执行 `nginx -t`（-t 追加到了 nginx 后面），入口始终是 nginx。

> **记忆口诀**：CMD 是"可以被推翻的默认值"，ENTRYPOINT 是"锁死的入口"。要做一个"像命令一样使用"的镜像（例如数据备份工具），用 ENTRYPOINT 更合适。

### 6.4 完整示例：构建一个自定义网页镜像（逐行解释）

先准备目录和文件：

```bash
mkdir -p ~/myweb && cd ~/myweb
# 创建 index.html
cat > index.html <<'EOF'
<!DOCTYPE html>
<html><head><meta charset="utf-8"><title>我的应用</title></head>
<body><h1>Hello Docker! 这是我自己打包的应用</h1></body></html>
EOF
```

创建 Dockerfile：

```dockerfile
# 1. 基础镜像：用官方 nginx 最新稳定版
FROM nginx:1.25

# 2. 维护者信息（可选，仅为文档）
LABEL maintainer="ops@example.com"

# 3. 设置时区等环境变量
ENV TZ=Asia/Shanghai

# 4. 设置工作目录
WORKDIR /usr/share/nginx/html

# 5. 把本地的 index.html 拷贝进镜像的网页目录
COPY ./index.html /usr/share/nginx/html/index.html

# 6. 声明容器监听的端口（文档性质）
EXPOSE 80

# 7. 容器启动时前台运行 nginx（注意 daemon off 让进程不后台化，否则容器会立刻退出）
CMD ["nginx", "-g", "daemon off;"]
```

**逐行解释**：

1. `FROM nginx:1.25`：以官方 nginx 1.25 为地基，省去自己装系统装 nginx 的步骤。
2. `LABEL`：给镜像写点元信息，纯文档作用。
3. `ENV TZ=Asia/Shanghai`：设置环境变量，让容器内日志时间是北京时间。
4. `WORKDIR`：切到网页根目录，后续命令的相对路径都基于这里。
5. `COPY`：把宿主机当前目录的 index.html 放进镜像，覆盖默认首页。
6. `EXPOSE 80`：声明监听 80 端口（真正映射靠 -p）。
7. `CMD`：容器启动时执行 `nginx -g "daemon off;"`。为什么要 `daemon off`？因为容器要求"前台进程存活则容器存活"，nginx 默认是后台守护进程，一旦后台化，主进程退出，容器就会立刻停止；`daemon off` 让它前台运行，容器就不会退出。

构建并运行：

```bash
docker build -t myweb:1.0 .
docker run -d --name myweb -p 8081:80 myweb:1.0
```

- `docker build -t myweb:1.0 .`：`-t` 指定镜像名:标签；最后的 `.` 是**构建上下文**（当前目录，Docker 会把这个目录里的东西发给守护进程用于 COPY）。
- 构建过程能看到每一条指令逐步执行（`Step 1/7 ...`），这就是分层构建的直观体现。

验证：浏览器访问 `http://服务器IP:8081`，看到"Hello Docker! 这是我自己打包的应用"。

### 6.5 动手练习（第6章）

1. 照着 6.4 从零构建并运行 myweb 镜像，浏览器看到自定义页面。
2. 把 Dockerfile 里最后一行改成只留 `CMD ["nginx","-g","daemon off;"]` 不变，再新增一行 `ENTRYPOINT ["nginx"]` 并调整，体会两种方式的运行效果。
3. 尝试用 `docker build --build-arg` 传入一个 ARG，验证 ARG 只在构建期可见。

---

## 第7章 数据卷 volume

### 7.1 为什么需要数据卷

容器删掉后，它的可写层（第5章）也会一起消失——也就是说，**容器里产生的数据默认是"易失"的**。数据库数据、用户上传的文件不能放在容器可写层里。数据卷就是解决"数据持久化 + 宿主机与容器共享数据"的机制。

### 7.2 三种挂载方式对比（是什么 + 区别）

| 方式 | 写法 | 数据位置 | 谁来管理 | 适用场景 |
| --- | --- | --- | --- | --- |
| 匿名卷 | `-v /容器内路径` | Docker 自动分配宿主机目录 | Docker | 临时数据，不在乎位置 |
| 命名卷（named volume） | `-v 卷名:/容器内路径` | Docker 管理的卷目录 | Docker | **生产推荐**，持久化数据 |
| bind mount | `-v /宿主机绝对路径:/容器内路径` | 用户指定宿主机目录 | 用户 | 开发调试、挂配置文件 |

### 7.3 命名卷（named volume）

```bash
# 创建命名卷
docker volume create webdata

# 查看所有卷
docker volume ls

# 查看卷详情（能看到宿主机上的实际挂载点 Mountpoint）
docker volume inspect webdata

# 用卷启动容器：把宿主机卷 webdata 挂到容器 /usr/share/nginx/html
docker run -d --name web -p 8082:80 -v webdata:/usr/share/nginx/html nginx
```

- **参数解释**：`-v webdata:/usr/share/nginx/html` 中 `webdata` 是卷名，冒号后是容器内路径。
- **作用/原理**：Docker 在宿主机统一目录（通常是 `/var/lib/docker/volumes/webdata/_data`）创建一个目录，并与容器内路径双向同步。容器删了，卷还在，数据不丢。
- **验证**：`docker rm -f web` 后 `docker volume ls`，卷依然存在。

### 7.4 bind mount（绑定挂载）

```bash
mkdir -p /data/web
docker run -d --name web -p 8083:80 -v /data/web:/usr/share/nginx/html nginx
```

- **作用/原理**：直接把宿主机 `/data/web` 目录挂进容器。适合开发时"宿主机改代码，容器立即生效"的场景。
- **对比命名卷**：bind mount 路径由你掌控、便于查看和备份；命名卷由 Docker 管理、可移植性好、有独立的 `volume` 子命令体系。

### 7.5 验证数据卷的持久化效果（前后对比实验）

```bash
# 1. 启动容器并写入文件
docker run -d --name db -v webdata:/data alpine sleep 3600
docker exec db sh -c "echo persistent > /data/test.txt"

# 2. 删除容器（数据卷不会跟着删）
docker rm -f db

# 3. 用新容器挂载同一个卷，文件还在
docker run --rm -v webdata:/data alpine cat /data/test.txt
```

预期结果：最后一条命令输出 `persistent`，证明**数据已经脱离容器生命周期而存活**。

### 7.6 动手练习（第7章）

1. 创建一个命名卷，挂载到 nginx 网页目录，写入文件后删除容器，再挂同一个卷验证数据仍在。
2. 用 bind mount 把宿主机某目录挂进容器，在宿主机修改文件，进容器确认同步。
3. 执行 `docker volume prune` 清理未使用的卷，观察提示。

---

## 第8章 Docker 网络

### 8.1 Docker 网络模型概览（是什么）

Docker 为每个容器创建独立的网络命名空间，并提供了几种网络驱动。查看现有网络：

```bash
docker network ls
```

预期结果（默认有三种）：

| NETWORK ID | NAME | DRIVER | SCOPE |
| --- | --- | --- | --- |
| ... | bridge | bridge | local |
| ... | host | host | local |
| ... | none | null | local |

### 8.2 三种默认网络对比

| 网络 | 特点 | 通信能力 | 典型用途 |
| --- | --- | --- | --- |
| bridge | 默认，容器连到一个虚拟网桥 docker0 | 容器间可通过 IP 互通；宿主机通过 -p 访问 | 默认单机网络 |
| host | 容器直接使用宿主机网络栈 | 完全共享宿主机网络 | 追求极致网络性能时 |
| none | 容器无网络 | 完全隔离 | 高度安全隔离的场景 |

**bridge（默认）示例**：

```bash
docker run -d --name c1 nginx
docker inspect -f '{{.NetworkSettings.IPAddress}}' c1   # 查看容器 IP，一般是 172.17.x.x
```

- **作用/原理**：容器被接到 docker0 网桥，各自获得一个 172.17.0.0/16 网段的 IP，同一网桥上的容器天然可互访。
- **缺点**：默认 bridge 上容器之间**只能用 IP 互访**，无法用容器名互相解析。

**host 示例**：

```bash
docker run -d --network host --name c2 nginx
```

- **作用/原理**：容器直接用宿主机的网络，不需要 `-p` 映射，容器监听的 80 直接就是宿主机 80。缺点：端口可能冲突，隔离性差。

**none 示例**：

```bash
docker run -d --network none --name c3 alpine sleep 3600
```

- **作用/原理**：容器只有回环网卡，无法与外界通信，用于需要完全断网的场景。

### 8.3 自定义网络（重点：容器间通信）

**为什么用自定义网络**：默认 bridge 只能用 IP 互访，而 IP 会变；自定义网络提供**内置 DNS**，容器之间可以**直接用容器名互相访问**，这才是容器编排（尤其 compose）的基础。

```bash
# 1. 创建自定义 bridge 网络
docker network create mynet

# 2. 在两个容器上使用同一个自定义网络
docker run -d --name web --network mynet nginx
docker run -d --name app --network mynet alpine sleep 3600

# 3. 在 app 容器里用"容器名"直接访问 web（关键：web 能被解析为 IP）
docker exec app ping -c 2 web
```

预期结果：`ping` 成功，输出 `64 bytes from web.mynet (172.18.0.x)`，证明 `web` 这个名字被 DNS 解析成了 IP。

- **参数解释**：`docker network create mynet` 创建网络；`--network mynet` 把容器接入该网络。
- **作用/原理**：自定义网络内嵌 DNS 服务器（127.0.0.11），容器加入网络时自动注册名字，实现"名字互通"。

查看网络详情与断开：

```bash
docker network inspect mynet            # 查看网络里的容器和 IP
docker network disconnect mynet web     # 把 web 从网络摘除
docker network rm mynet                 # 删除网络（需先清空容器）
```

### 8.4 动手练习（第8章）

1. 用 `docker network create` 建一个网络，跑两个容器，验证容器名互通。
2. 对比实验：在**默认 bridge** 上跑两个容器，尝试用容器名 ping，观察失败，理解自定义网络的 DNS 优势。
3. 分别用 `--network host` 和 `--network none` 启动容器，观察其网络差异（`docker exec` 进去 `ip addr` 查看）。

---

## 第9章 docker-compose 编排

### 9.1 compose 是什么、为什么用

- **是什么**：一个用 YAML 文件一次性定义并管理**多个容器**的工具。把 `docker run` 那一长串参数写成声明式的 `docker-compose.yml`。
- **为什么用**：一个 LNMP 应用要 nginx、php、mysql 三个容器，还要配网络、卷、依赖关系。靠一条条 `docker run` 太繁琐、难维护，compose 一条 `docker compose up` 全部搞定，且可复用、可进版本库。
- **怎么用**：写 `docker-compose.yml`，用 `docker compose up -d` 启动。

### 9.2 安装 docker-compose

**方式一（推荐，已装 plugin 的情况）**：在第3章我们已安装 `docker-compose-plugin`，直接用 `docker compose`（注意中间是空格）命令即可。

```bash
docker compose version    # 验证，输出 Docker Compose version v2.x.x
```

**方式二（老式独立二进制，CentOS7 离线/兼容用）**：

```bash
# 从国内镜像下载独立版 compose
sudo curl -L "https://mirrors.aliyun.com/docker-toolbox/linux/compose/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
docker-compose version    # 老命令是 docker-compose（中间是连字符）
```

- **作用/原理**：compose v2 已作为 docker 插件集成（`docker compose`），v1 是独立二进制（`docker-compose`），功能等价，新环境优先 v2。

### 9.3 docker-compose.yml 写法（逐字段解释）

先看一个最简示例（单个 nginx）：

```yaml
version: "3.8"            # 文件格式版本（compose v2 中可省略）
services:                 # 服务列表，每个服务对应一个容器
  web:                    # 服务名（也是容器名，同时是网络里的主机名）
    image: nginx:1.25     # 使用哪个镜像
    container_name: web   # 容器名（可选，不写则自动生成）
    ports:                # 端口映射列表（等价于 -p）
      - "8080:80"
    volumes:              # 卷挂载列表（等价于 -v）
      - ./html:/usr/share/nginx/html
      - webdata:/data
    environment:          # 环境变量（等价于 -e）
      TZ: Asia/Shanghai
    restart: always       # 重启策略（等价于 --restart）
    networks:             # 加入哪个网络
      - mynet
volumes:                  # 顶层声明命名卷（否则会报卷不存在）
  webdata:
networks:                 # 顶层声明自定义网络
  mynet:
```

**字段说明**：

| 字段 | 等价 docker run 参数 | 作用 |
| --- | --- | --- |
| `image` | 镜像名 | 指定镜像 |
| `build` | （替代 image，用 Dockerfile 构建） | 用 Dockerfile 现场构建 |
| `ports` | `-p` | 端口映射 |
| `volumes` | `-v` | 数据卷挂载 |
| `environment` | `-e` | 环境变量 |
| `restart` | `--restart` | 重启策略 |
| `depends_on` | 无（compose 特有） | 控制启动顺序 |
| `networks` | `--network` | 加入网络 |

### 9.4 常用命令详解

```bash
docker compose up -d          # 启动全部服务（-d 后台），首次会自动创建网络/卷
docker compose up -d --build  # 启动并重新构建镜像（改了 Dockerfile 用这个）
docker compose down           # 停止并删除容器/网络（默认不删卷）
docker compose down -v        # 连同命名卷一起删除（慎用，会清数据）
docker compose ps             # 查看本 compose 项目的容器状态
docker compose logs -f web    # 跟踪某个服务的日志
docker compose restart mysql  # 重启单个服务
docker compose exec web bash  # 进入某个服务的容器
docker compose stop           # 停止（不删除容器）
docker compose start          # 启动（已停止的容器）
```

- **作用/原理**：`up` 是"创建并启动"，`start/stop` 是"启停已存在的"；`down` 是"拆除整个项目"；`-d` 后台、`-v` 连卷一起删。所有这些命令都要在 `docker-compose.yml` 所在目录执行。

### 9.5 动手练习（第9章）

1. 写一个单服务 `docker-compose.yml`（nginx），用 `docker compose up -d` 启动，浏览器验证。
2. 用 `docker compose ps`、`docker compose logs -f` 观察运行状态和日志。
3. 用 `docker compose down` 拆除，再用 `docker compose up -d` 重建，体会"一条命令管理"的便利。

---

## 第10章 完整项目实战

> 本章把前面所有知识串起来，完成三个任务：
> 1. 写 Dockerfile 打包一个应用；
> 2. 用 docker-compose 编排 LNMP（nginx + php-fpm + mysql）；
> 3. 把镜像推到阿里云容器镜像服务（ACR）。

### 10.1 任务一：用 Dockerfile 打包一个 PHP 应用

**第1步**：准备项目目录和代码。

```bash
mkdir -p ~/lnmp/app && cd ~/lnmp/app
cat > index.php <<'EOF'
<?php
$host = getenv('DB_HOST') ?: 'mysql';
$user = getenv('DB_USER') ?: 'root';
$pass = getenv('DB_PASS') ?: 'root123';
try {
    $pdo = new PDO("mysql:host=$host;dbname=test", $user, $pass);
    $pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
    echo "<h1>LNMP 部署成功！数据库连接正常。</h1>";
    $r = $pdo->query('SELECT VERSION() as v')->fetch();
    echo "<p>MySQL 版本: " . $r['v'] . "</p>";
} catch (Exception $e) {
    echo "<h1>LNMP 部署成功！但数据库连接失败：</h1><p>" . $e->getMessage() . "</p>";
}
?>
```

- **代码解释**：页面从环境变量读数据库连接信息（`DB_HOST/DB_USER/DB_PASS`），尝试连 MySQL 并打印版本；连不上也把错误显示出来，方便排查。用环境变量而不是硬编码，符合"配置与代码分离"的最佳实践。

**第2步**：编写 Dockerfile。

```dockerfile
# 基于官方 php-fpm 镜像（已内置 PHP-FPM 和 PDO MySQL 扩展的常用基础）
FROM php:8.2-fpm

# 安装 pdo_mysql 扩展（php 官方镜像默认不带 MySQL 驱动，需手动装）
RUN docker-php-ext-install pdo_mysql

# 设置工作目录
WORKDIR /var/www/html

# 拷贝应用代码到容器
COPY ./index.php /var/www/html/index.php

# 声明端口（php-fpm 默认 9000）
EXPOSE 9000

# 启动 php-fpm（官方镜像默认就是这个命令，写出来只为显式说明）
CMD ["php-fpm"]
```

**逐行解释**：

1. `FROM php:8.2-fpm`：选 php-fpm 变体（专为配合 nginx 设计，nginx 处理静态文件、把 PHP 请求转给 php-fpm）。
2. `RUN docker-php-ext-install pdo_mysql`：`docker-php-ext-install` 是官方镜像提供的脚本，一键编译安装 PHP 扩展；这里装 MySQL 驱动，否则 `new PDO("mysql:...")` 会失败。
3. `WORKDIR`：切到网页根目录。
4. `COPY`：放进应用代码。
5. `EXPOSE 9000`：声明 php-fpm 端口。
6. `CMD ["php-fpm"]`：前台启动 php-fpm。

**第3步**：构建镜像。

```bash
docker build -t myphp:1.0 .
docker images | grep myphp
```

预期结果：`myphp  1.0  <IMAGE_ID>  ...  xxx MB`，构建成功。

### 10.2 任务二：用 docker-compose 编排 LNMP

**整体架构**：

```
浏览器
   │ 访问 宿主机:80
   ▼
nginx 容器 (端口映射 80:80)
   │ 通过自定义网络，把 *.php 请求 fastcgi 转发到 php:9000
   ▼
php 容器 (php-fpm:9000)
   │ 通过自定义网络，PDO 连接 mysql:3306
   ▼
mysql 容器 (3306，数据存命名卷 dbdata)
```

**第1步**：创建 nginx 配置文件。

```bash
mkdir -p ~/lnmp/nginx && cd ~/lnmp/nginx
cat > default.conf <<'EOF'
server {
    listen 80;
    server_name localhost;
    root /var/www/html;
    index index.php index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    location ~ \.php$ {
        include fastcgi_params;
        fastcgi_pass php:9000;      # 关键：php 是 compose 里的服务名，靠 DNS 解析
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_index index.php;
    }
}
EOF
```

- **配置解释**：`fastcgi_pass php:9000` 里写的是**服务名** `php` 而不是 IP，正是第8章"自定义网络容器名互通"的实战应用。nginx 会通过 compose 内置网络把 PHP 请求转发给 php 容器。

**第2步**：编写 docker-compose.yml。

```bash
cd ~/lnmp
cat > docker-compose.yml <<'EOF'
version: "3.8"

services:
  nginx:
    image: nginx:1.25
    container_name: nginx
    ports:
      - "80:80"
    volumes:
      - ./app:/var/www/html                # 应用代码（与 php 共享）
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf   # nginx 配置
    depends_on:
      - php
    networks:
      - lnmp

  php:
    build: ./app                           # 用 app 目录里的 Dockerfile 构建
    container_name: php
    volumes:
      - ./app:/var/www/html                # 同一份代码，nginx 和 php 都要读到
    environment:
      DB_HOST: mysql                       # 通过服务名连接 mysql
      DB_USER: root
      DB_PASS: root123
    depends_on:
      - mysql
    networks:
      - lnmp

  mysql:
    image: mysql:8.0
    container_name: mysql
    environment:
      MYSQL_ROOT_PASSWORD: root123         # 设置 root 密码
      MYSQL_DATABASE: test                 # 初始化时自动建库 test
    volumes:
      - dbdata:/var/lib/mysql              # 数据持久化到命名卷
    ports:
      - "3306:3306"                        # 可选：暴露给宿主机便于调试
    networks:
      - lnmp

volumes:
  dbdata:

networks:
  lnmp:
    driver: bridge
EOF
```

**逐段解释**：

- `nginx`：挂载了两样东西——应用代码目录和自定义配置文件；`depends_on: php` 保证 php 先启动。
- `php`：用 `build: ./app` 现场构建（第10.1的 Dockerfile）；挂载同一份 `./app` 代码目录，保证 nginx 和 php 看到的是同一份文件；通过环境变量把数据库地址 `DB_HOST: mysql` 传进去。
- `mysql`：`MYSQL_ROOT_PASSWORD` 和 `MYSQL_DATABASE` 是官方镜像的初始化变量，首次启动会自动建库设密码；`dbdata` 命名卷持久化数据；`depends_on` 建立启动顺序。
- 顶层 `volumes:` 和 `networks:` 声明资源，服务里才能引用。

**第3步**：启动并验证。

```bash
docker compose up -d          # 启动（首次会先 build php 镜像）
docker compose ps             # 查看三个容器状态
docker compose logs -f php    # 看 php 是否正常，Ctrl+C 退出
docker compose logs mysql     # 看 mysql 初始化日志
```

预期结果：三个容器 `Up`，`docker compose ps` 显示 nginx、php、mysql 都在运行。

### 10.3 任务三：推送到阿里云容器镜像服务（ACR）

**第1步**：登录阿里云控制台，开通"容器镜像服务 ACR"。

1. 打开阿里云控制台 → 搜索"容器镜像服务" → 首次进入按提示开通（个人版免费）。
2. 在左侧"实例列表"中进入**个人版实例**，记住你的**公网/专有网络地址**，形如 `registry.cn-hangzhou.aliyuncs.com`。
3. 在"命名空间"里创建一个命名空间，例如 `myops`。
4. 在"镜像仓库"里"创建镜像仓库"，地域选和实例一致，命名空间选 `myops`，仓库名填 `myphp`，创建后得到完整地址：`registry.cn-hangzhou.aliyuncs.com/myops/myphp`。

**第2步**：设置固定密码。

在控制台"访问凭证"（或"个人实例→访问凭证"）里设置一个**固定登录密码**，登录用户名就是你阿里云账号（或 RAM 子账号）。

**第3步**：在服务器上登录 ACR。

```bash
docker login --username=你的阿里云账号 registry.cn-hangzhou.aliyuncs.com
# 按提示输入上面设置的固定密码，看到 Login Succeeded 即成功
```

- **参数解释**：`--username` 指定用户名，最后一个参数是 Registry 地址。
- **作用/原理**：登录后凭证会被保存（`~/.docker/config.json`），后续 push 无需重复输入。

**第4步**：打标签并推送。

```bash
# 给本地镜像打上"仓库全名"标签（tag 只是加个名字，不复制数据）
docker tag myphp:1.0 registry.cn-hangzhou.aliyuncs.com/myops/myphp:1.0

# 推送到阿里云
docker push registry.cn-hangzhou.aliyuncs.com/myops/myphp:1.0
```

预期结果：出现一层层 `Pushed` / `Layer already exists`，最后 `1.0: digest: sha256:... size: ...` 表示推送成功。

**第5步**：验证拉取（可选，换台机器或删掉本地镜像再拉）。

```bash
docker rmi registry.cn-hangzhou.aliyuncs.com/myops/myphp:1.0
docker pull registry.cn-hangzhou.aliyuncs.com/myops/myphp:1.0
```

- **作用/原理**：从阿里云私有仓库把镜像拉回来，验证整个"构建→推送→拉取"链路畅通。

---

## 本章项目验收

> 目标：**从零容器化部署 LNMP，并能通过浏览器访问**。请按顺序完整执行，每一步都有验证点。这是本阶段最重要的一次实操，务必独立完成。

### 验收前置检查

```bash
docker version          # 确认 Client/Server 都正常
docker network ls       # 能看到默认网络
```

### 验收步骤

**步骤1：创建项目目录结构**

```bash
mkdir -p ~/lnmp/{app,nginx}
cd ~/lnmp
tree . || find .          # 确认结构：app 目录放代码和 Dockerfile，nginx 放配置
```

**步骤2：编写应用代码 `app/index.php`**

内容同第10.1（读取环境变量连 MySQL，成功打印版本，失败打印错误）。用 `cat > app/index.php <<'EOF' ... EOF` 写入。

**步骤3：编写 `app/Dockerfile`**

内容同第10.1（`FROM php:8.2-fpm` + 装 pdo_mysql + 拷贝代码）。

**步骤4：编写 `nginx/default.conf`**

内容同第10.2（`fastcgi_pass php:9000`）。

**步骤5：编写根目录 `docker-compose.yml`**

内容同第10.2（nginx + php + mysql 三服务，命名卷 dbdata，自定义网络 lnmp）。

**步骤6：一键启动**

```bash
cd ~/lnmp
docker compose up -d --build
docker compose ps
```

**步骤7：验证容器状态**

```bash
docker compose ps          # 三个容器都是 Up
docker compose logs mysql  # 看到 "ready for connections"
docker compose logs php    # 无报错
```

**步骤8：浏览器访问验收（关键）**

在浏览器打开 `http://服务器IP/`（若本机则 `http://127.0.0.1/`），预期看到：

```
LNMP 部署成功！数据库连接正常。
MySQL 版本: 8.0.x
```

**步骤9：验证数据持久化**

```bash
docker compose restart mysql       # 重启 mysql 容器
# 再次刷新浏览器，页面仍然能连上数据库，说明数据/服务正常
```

**步骤10：验证数据卷不随容器删除**

```bash
docker compose down                # 停并删容器（不删卷）
docker volume ls | grep dbdata     # 命名卷 dbdata 仍在
docker compose up -d               # 重新拉起，浏览器依旧正常
```

### 验收标准清单

| 序号 | 验收项 | 通过标准 |
| --- | --- | --- |
| 1 | docker 环境 | `docker version` 两端正常，加速器已配 |
| 2 | 镜像构建 | `docker compose up` 能成功 build 出 php 镜像 |
| 3 | 三容器运行 | `docker compose ps` 三个服务均为 Up |
| 4 | 网络互通 | nginx 通过服务名 `php` 转发成功 |
| 5 | 数据库连接 | 页面显示"数据库连接正常"和 MySQL 版本 |
| 6 | 数据持久化 | `down` 后再 `up`，数据仍在 |
| 7 | 镜像推送 | ACR 控制台能看到推送的镜像 |

**全部通过即视为本阶段合格**。恭喜你，你已经完整掌握了 Docker 的核心用法：从概念、安装、常用命令、镜像原理、Dockerfile、数据卷、网络，到 compose 编排和镜像推送，并能独立容器化部署一个 LNMP 应用。
