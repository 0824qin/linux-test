# 阶段7 Kubernetes 容器编排 —— 从零到一完整实操教程

> 本教程面向零基础小白，目标只有一个：**照着敲，就能把一个真实的 K8s 集群跑起来，并把 LNMP 应用完整迁移上去**。
> 每个知识点都按「是什么 → 为什么用 → 怎么用（含命令 + 字段解释 + 预期结果）」三层讲透，绝不只是名词清单。
> 实验环境：3 台虚拟机，Ubuntu 22.04 LTS，Kubernetes v1.28.x，containerd 运行时，Calico 网络。

---

## 目录

1. [为什么需要 Kubernetes 与核心概念](#第1章-为什么需要-kubernetes-与核心概念)
2. [集群环境准备（3 台机器 + 运行时）](#第2章-集群环境准备)
3. [kubeadm 搭建 1 主 2 从集群](#第3章-用-kubeadm-搭建-1主2从集群)
4. [核心资源详解（每个配 YAML）](#第4章-核心资源详解)
5. [kubectl 常用命令手册](#第5章-kubectl-常用命令手册)
6. [国内镜像源与拉取加速](#第6章-国内镜像源与拉取加速)
7. [Helm 包管理器](#第7章-helm-包管理器)
8. [完整项目：LNMP 迁移到 K8s](#第8章-完整项目把-lnmp-迁移到-kubernetes)
9. [本章项目验收](#第9章-本章项目验收)

---

# 第1章 为什么需要 Kubernetes 与核心概念

## 1.1 部署方式的演进（前后对比）

在理解 K8s 之前，先搞清一个问题：我们的应用是怎么被"部署"到服务器上的？一共经历了三个阶段。

### 阶段一：传统部署（裸机 / 虚拟机直接装）

直接把程序装在服务器操作系统上：

```bash
# 传统方式：在服务器上直接安装 Nginx + PHP + MySQL
apt update
apt install -y nginx php-fpm mysql-server
vim /etc/nginx/nginx.conf   # 手工改配置
systemctl start nginx
```

**问题**：
- 应用和操作系统强耦合，A 应用需要 PHP 7.4，B 应用需要 PHP 8.1，同一台机器装不下两套环境。
- 迁移、扩容要靠"重装一遍系统 + 重装一遍依赖"，费时费力。
- 资源利用率低：一台 16 核 64G 的机器可能只跑一个轻量应用。

### 阶段二：容器部署（Docker）

把应用和它的依赖一起打包成镜像，随处运行：

```bash
docker run -d --name nginx -p 80:80 nginx:1.25
```

**解决了什么**：环境隔离、一次打包到处运行、启动秒级。

**遗留问题**：
- 单台机器的 Docker 出问题，应用就挂了（无高可用）。
- 容器多了以后，靠 `docker run` 一条条手工管理几十上百个容器？根本不现实。
- 某容器挂了不会自动拉起；流量大了不会自动加实例；版本升级要手工停旧起新。

### 阶段三：容器编排（Kubernetes）

K8s 接管所有容器，你只要**声明"我要 3 个 Nginx 副本，永远保持 3 个"**，K8s 自动帮你调度、拉起、健康检查、滚动升级。

| 对比维度 | 传统部署 | Docker 单机 | Kubernetes |
|---|---|---|---|
| 环境隔离 | 无 | 有 | 有 |
| 高可用 | 手工做 | 无 | 自动 |
| 自动扩容 | 无 | 无 | 有（HPA） |
| 故障自愈 | 无 | 无 | 有（Pod 自动重启/重建） |
| 滚动升级/回滚 | 手工停服 | 手工 | 一行命令 |
| 服务发现 | 手工改 IP | 手工 | 自动（Service） |
| 配置管理 | 硬编码/散落 | 环境变量 | ConfigMap/Secret |
| 多机编排 | 无 | 无 | 核心能力 |

**一句话总结**：Docker 解决"单个应用怎么打包运行"，K8s 解决"成百上千个容器在多台机器上怎么统一管理"。

## 1.2 Kubernetes 是什么

Kubernetes（简称 K8s，因为 K 和 s 中间有 8 个字母）是 Google 开源的**容器编排平台**。它的核心工作模型是：

> **你声明期望状态（Desired State），K8s 不断把实际状态（Actual State）调向期望状态。**

```yaml
# 你声明：我要 3 个 nginx 副本
spec:
  replicas: 3
```

K8s 会持续检查：现在实际只有 2 个？那就再起 1 个。某台机器宕机导致副本变成 1 个？自动在别的机器补 2 个。这种"声明式 + 调和循环（Reconcile Loop）"是 K8s 的立身之本。

## 1.3 核心概念

### 1.3.1 集群（Cluster）

- **是什么**：一组被 K8s 统一管理的机器（物理机或虚拟机）的集合。
- **为什么用**：单个节点的算力、可靠性有限，集群把多台机器聚合成"一个资源池"，应用跑在哪台机器上由 K8s 调度，用户不关心。
- **怎么用**：你面对的不是 3 台机器，而是"一个集群"，用 `kubectl` 跟集群的控制平面对话。

### 1.3.2 节点（Node）

- **是什么**：集群里的一台真实机器。分两类：
  - **控制平面节点（Control Plane Node，旧称 Master）**：跑集群"大脑"组件（API Server、Scheduler、Controller Manager、etcd）。
  - **工作节点（Worker Node）**：真正跑业务容器的地方。
- **为什么分两类**：控制平面负责决策（这个 Pod 放哪台机器），工作节点负责执行（跑容器）。二者职责分离，控制平面挂了整个集群就"失联"，所以生产环境控制平面常做多副本高可用。
- **怎么用**：工作节点通过 `kubeadm join` 加入集群后，`kubectl get nodes` 就能看到它们。

### 1.3.3 控制平面（Control Plane）与核心组件

| 组件 | 是什么 | 为什么用 | 一句话作用 |
|---|---|---|---|
| **kube-apiserver** | 集群的唯一入口（REST API） | 所有操作（kubectl、组件间通信）都走它，统一鉴权、校验、持久化 | "前台" |
| **etcd** | 分布式键值数据库 | 存储集群所有状态数据，是整个集群的"记忆" | "数据库" |
| **kube-scheduler** | 调度器 | 决定新 Pod 放到哪台工作节点（看资源、亲和性、污点） | "排班" |
| **kube-controller-manager** | 一堆控制器的集合 | 持续把实际状态调向期望状态（副本不够就扩、节点挂了就迁移） | "监工" |
| **cloud-controller-manager** | 云厂商控制器 | 对接云负载均衡、云盘等；自建集群通常没有 | "云对接"（可选） |

### 1.3.4 Pod（最小调度单位）

- **是什么**：K8s 里**最小、最基础**的调度和管理单位，是一个或多个容器的组合，共享同一网络命名空间（同一个 IP）和存储卷。
- **为什么用**：K8s 不直接调度容器，而是调度 Pod。常见场景是一个主容器 + 若干"边车"容器（sidecar，比如日志收集、代理）。
- **怎么用**：一个 Pod 里的容器共享 `localhost` 通信、共享挂载卷。**注意**：Pod 是"易逝"的（Ephemeral），被杀掉重建后 IP、内部数据都会变——所以后面才有 Service、PV 来补位。

### 1.3.5 命名空间（Namespace）

- **是什么**：集群内部的逻辑分区，用于把资源按项目、团队、环境隔离开。
- **为什么用**：一个集群可以服务多个团队，互不干扰；权限、资源配额、命名可以按命名空间隔离。
- **怎么用**：

```bash
kubectl get namespaces
# 输出（自带四个）：
# NAME              STATUS   AGE
# default            Active   5d     # 默认命名空间，不指定时就用它
# kube-system        Active   5d     # 系统组件所在（DNS、网络插件等）
# kube-public        Active   5d     # 公共只读信息
# kube-node-lease    Active   5d     # 节点心跳租约
```

## 1.4 架构图（文字版）

```
                          ┌──────────────────────────────────────────┐
                          │            用户 / 运维                    │
                          │    kubectl 命令行 或 Web 控制台(Dashboard)│
                          └───────────────────┬──────────────────────┘
                                              │ HTTPS (6443)
                                              ▼
┌───────────────────────────────────────────────────────────────────────┐
│                      控制平面节点 (Control Plane)                      │
│                                                                       │
│   ┌──────────────┐     ┌───────────────────────────┐                  │
│   │ kube-apiserver│◄──►│          etcd             │  集群"记忆"      │
│   └──────┬───────┘     └───────────────────────────┘                  │
│          │ watch                                                       │
│   ┌──────▼────────────────┐   ┌────────────────────────┐              │
│   │ kube-controller-manager│  │     kube-scheduler      │              │
│   └───────────────────────┘   └────────────────────────┘              │
└───────────────────────────────────┬───────────────────────────────────┘
                                    │ 下发调度结果
          ┌─────────────────────────┼─────────────────────────┐
          ▼                         ▼                         ▼
┌──────────────────┐      ┌──────────────────┐      ┌──────────────────┐
│   工作节点 Node1  │      │   工作节点 Node2  │      │   工作节点 Node3  │
│  ┌────────────┐  │      │  ┌────────────┐  │      │  ┌────────────┐  │
│  │   kubelet  │  │      │  │   kubelet  │  │      │  │   kubelet  │  │
│  └─────┬──────┘  │      │  └─────┬──────┘  │      │  └─────┬──────┘  │
│  ┌─────▼──────┐  │      │  ┌─────▼──────┐  │      │  ┌─────▼──────┐  │
│  │ kube-proxy │  │      │  │ kube-proxy │  │      │  │ kube-proxy │  │
│  └────────────┘  │      │  └────────────┘  │      │  └────────────┘  │
│  ┌────────────┐  │      │  ┌────────────┐  │      │  ┌────────────┐  │
│  │  Pod  Pod  │  │      │  │  Pod  Pod  │  │      │  │  Pod  Pod  │  │
│  │ (容器们)    │  │      │  │ (容器们)    │  │      │  │ (容器们)    │  │
│  └────────────┘  │      │  └────────────┘  │      │  └────────────┘  │
│  containerd 运行时│      │  containerd 运行时│      │  containerd 运行时│
└──────────────────┘      └──────────────────┘      └──────────────────┘
```

**工作节点上的关键组件**：

| 组件 | 是什么 | 作用 |
|---|---|---|
| **kubelet** | 每台节点上的"代理" | 接收控制平面指令，确保本机 Pod 按声明运行，上报状态 |
| **kube-proxy** | 网络代理 | 实现 Service 的网络规则（iptables/IPVS），让流量能正确路由到 Pod |
| **容器运行时** | containerd / CRI-O / Docker(经 cri-dockerd) | 真正拉取镜像、启动容器 |

**一次 `kubectl apply -f nginx.yaml` 的完整链路**（理解它你就理解了架构）：

1. `kubectl` 把 YAML 发给 **kube-apiserver**。
2. apiserver 鉴权、校验后，把期望状态写入 **etcd**。
3. **scheduler** 监听到有未调度的 Pod，选一台合适的节点，写回 etcd。
4. 目标节点的 **kubelet** 监听到"本机要跑这个 Pod"，调用容器运行时拉镜像、起容器。
5. **controller-manager** 持续核对：副本数对不对、健康不健康，不符就纠正。

---

## 【动手练习 · 第1章】

1. 用自己的话在纸上写出：集群、节点、控制平面、Pod、命名空间这 5 个词各是什么，各解决什么问题。
2. 画出（或默写）上面的架构图，并在图上标注：`kubectl` 的命令先发给谁？Pod 由谁最终拉起？
3. 思考题：为什么 Pod 不能直接对用户提供稳定入口？（提示：Pod 会重建、IP 会变）——答案在第 4 章的 Service 小节揭晓。

---

# 第2章 集群环境准备

## 2.1 机器规划

本教程用 3 台 Ubuntu 22.04 虚拟机（内存 ≥ 2G，建议 4G；CPU ≥ 2 核）。IP 按你的实际环境替换。

| 角色 | 主机名 | IP（示例） | 最低配置 |
|---|---|---|---|
| 控制平面节点 | k8s-master | 192.168.56.10 | 2C / 2G |
| 工作节点 | k8s-node1 | 192.168.56.11 | 2C / 2G |
| 工作节点 | k8s-node2 | 192.168.56.12 | 2C / 2G |

> 下文命令里凡出现 `192.168.56.10` 这样的 IP，请替换成你自己机器的真实 IP。

## 2.2 系统初始化（3 台机器都要做）

以下命令**在 3 台机器上都执行一遍**（可以用 `ssh` 分别登录，或每台单独执行）。

### 2.2.1 设置主机名

```bash
# master 机器执行：
hostnamectl set-hostname k8s-master

# node1 机器执行：
hostnamectl set-hostname k8s-node1

# node2 机器执行：
hostnamectl set-hostname k8s-node2
```
- **字段/命令解释**：`hostnamectl set-hostname` 永久修改主机名（重启不丢）。`/etc/hostname` 里也会同步写入。
- **作用**：K8s 里节点以主机名标识，名字清晰便于排查。

### 2.2.2 配置 hosts（3 台都做）

```bash
cat >> /etc/hosts <<EOF
192.168.56.10 k8s-master
192.168.56.11 k8s-node1
192.168.56.12 k8s-node2
EOF
```
- **解释**：`cat >> /etc/hosts <<EOF ... EOF` 把三行追加写入 hosts 文件（`>>` 是追加，`EOF` 是 heredoc 结束标记）。
- **作用**：让各节点能通过主机名互访，避免 DNS 没配导致节点间通信失败。

### 2.2.3 关闭 swap 交换分区

```bash
# 临时关闭
swapoff -a
# 永久关闭：注释掉 /etc/fstab 里带 swap 的那一行
sed -i 's/^\(.*swap.*\)$/#\1/' /etc/fstab
# 验证：输出应为空
swapon --show
```
- **为什么**：kubelet 强制要求关闭 swap（早期版本硬性限制，避免内存换出导致性能与调度混乱）。不关会导致 `kubeadm init` 或 kubelet 启动报错 `failed to run Kubelet: running with swap on is not supported`。
- **参数解释**：`swapoff -a` 关闭所有 swap；`sed -i 's/.../.../'` 原地修改文件，把 swap 行前面加 `#` 注释掉。

### 2.2.4 配置内核参数（转发 + 网桥过滤）

```bash
cat > /etc/modules-load.d/k8s.conf <<EOF
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter

cat > /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sysctl --system
```
- **逐条解释**：
  - `overlay`、`br_netfilter`：两个内核模块。overlay 是容器镜像分层存储依赖的联合文件系统；br_netfilter 让桥接网络的包能经过 iptables 过滤（Service 转发依赖它）。
  - `net.bridge.bridge-nf-call-iptables = 1`：桥接流量也要过 iptables，否则 kube-proxy 的规则对 Pod 流量不生效。
  - `net.ipv4.ip_forward = 1`：开启 IP 转发，节点要能帮 Pod 转发跨网段流量。
  - `sysctl --system`：让上面所有 sysctl 配置立即生效。
- **作用**：这些是 K8s 网络（尤其是 Service、Pod 互通）能正常工作的前提。

### 2.2.5 关闭防火墙（实验环境）

```bash
systemctl stop ufw && systemctl disable ufw
# CentOS/Rocky 用：systemctl stop firewalld && systemctl disable firewalld
```
- **为什么**：实验环境为减少变量，直接关掉主机防火墙；生产环境应改用精准的端口放行策略（master 放 6443/2379-2380/10250 等）。

### 2.2.6 时间同步（可选但推荐）

```bash
apt install -y chrony
systemctl enable --now chrony
chronyc sources
```
- **作用**：节点间时间漂移会破坏证书校验和 etcd 一致性，务必保证时间基本一致。

## 2.3 安装容器运行时：containerd（3 台都做）

从 K8s 1.24 起，官方移除了对 Docker 的直接支持（Docker 需额外装 cri-dockerd 适配器）。本教程直接使用官方推荐且更轻量的 **containerd**。

```bash
apt update
apt install -y containerd

# 生成默认配置
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml
```

**关键修改（非常重要，不改会拉不动镜像）**：编辑 `/etc/containerd/config.toml`：

```bash
vim /etc/containerd/config.toml
```

找到这两处并修改：

```toml
# 1) 把 sandbox_image 换成国内镜像（原为 registry.k8s.io/pause:3.9 会拉不动）
[plugins."io.containerd.grpc.v1.cri"]
  sandbox_image = "registry.aliyuncs.com/google_containers/pause:3.9"

# 2) 把 SystemdCgroup 改成 true（与 kubelet 的 cgroup 驱动保持一致，否则 kubelet 会警告甚至失败）
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
  SystemdCgroup = true
```

> 用 `sed` 一键改也行：
> ```bash
> sed -i 's|registry.k8s.io/pause:3.8|registry.aliyuncs.com/google_containers/pause:3.9|g' /etc/containerd/config.toml
> sed -i 's|SystemdCgroup = false|SystemdCgroup = true|g' /etc/containerd/config.toml
> ```

重启并设置开机自启：

```bash
systemctl restart containerd
systemctl enable containerd
systemctl status containerd   # 看到 active (running) 即成功
```

- **sandbox_image（pause 镜像）是什么**：每个 Pod 启动时，会先起一个极小的 `pause` 容器作为 Pod 的"网络/命名空间锚点"，其他容器共享它的命名空间。所以 pause 镜像是必须的。
- **为什么换国内源**：`registry.k8s.io` 在国内常被墙/超时，导致 Pod 卡在 `ContainerCreating`。
- **SystemdCgroup 为什么必须 true**：cgroup 是 Linux 限制 CPU/内存的机制；kubelet 默认用 systemd 驱动，containerd 若用 cgroupfs 驱动，两者"各管一套账"，会导致资源限制失效并产生 `cgroup driver mismatch` 告警。

## 2.4 配置 Docker 镜像加速（可选，如果你用 Docker）

若你坚持用 Docker 运行时（需额外装 cri-dockerd），记得配加速器：

```bash
cat > /etc/docker/daemon.json <<EOF
{
  "registry-mirrors": [
    "https://docker.m.daocloud.io",
    "https://dockerproxy.com",
    "https://docker.nju.edu.cn"
  ],
  "exec-opts": ["native.cgroupdriver=systemd"]
}
EOF
systemctl restart docker
```
- **registry-mirrors**：国内拉取 Docker Hub 镜像的加速地址。
- **exec-opts**：同样是为了 cgroup 驱动统一为 systemd。

## 2.5 安装 kubeadm / kubelet / kubectl（3 台都做）

用阿里云的 apt 源安装（避免官方源被墙）：

```bash
apt update && apt install -y apt-transport-https ca-certificates curl gpg

# 拉取 K8s 官方签名密钥（走阿里云镜像）
curl -fsSL https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.28/deb/Release.key \
  | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# 写入 apt 源
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
  https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.28/deb/ /" \
  | tee /etc/apt/sources.list.d/kubernetes.list

apt update
apt install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl
```

- **三个组件职责**：
  - `kubeadm`：集群"安装器"，负责 `init` 和 `join`，装完集群就"功成身退"。
  - `kubelet`：常驻后台的节点代理，真正干活的那个。
  - `kubectl`：你的命令行客户端，发指令用的。
- **`apt-mark hold`**：锁定版本，防止 `apt upgrade` 时把 K8s 组件意外升级到不兼容版本。

---

## 【动手练习 · 第2章】

1. 在三台机器上完成全部初始化，并分别执行 `swapon --show`、`sysctl net.ipv4.ip_forward`、`containerd --version` 验证。
2. 检查 containerd 配置：`grep -E "sandbox_image|SystemdCgroup" /etc/containerd/config.toml`，确认镜像地址是阿里云、SystemdCgroup 是 true。
3. 执行 `kubelet --version` 和 `kubectl version --client`，记录版本号，确认三者版本一致（本教程用 v1.28）。

---

# 第3章 用 kubeadm 搭建 1主2从集群

## 3.1 初始化控制平面（只在 master 执行）

```bash
kubeadm init \
  --image-repository registry.aliyuncs.com/google_containers \
  --kubernetes-version v1.28.2 \
  --pod-network-cidr=192.168.0.0/16 \
  --service-cidr=10.96.0.0/12 \
  --apiserver-advertise-address=192.168.56.10
```

**参数逐一解释**：

| 参数 | 含义 | 为什么这样设 |
|---|---|---|
| `--image-repository` | 镜像仓库地址 | 官方 `registry.k8s.io` 拉不动，换成阿里云镜像仓库 |
| `--kubernetes-version` | 指定版本 | 与已安装的 kubeadm/kubelet 版本对齐，避免混版 |
| `--pod-network-cidr` | Pod 网段 | 必须与后面 Calico 的默认网段一致（Calico 默认 `192.168.0.0/16`） |
| `--service-cidr` | Service 网段 | 集群内 Service 分配的虚拟 IP 段，一般用默认即可 |
| `--apiserver-advertise-address` | API Server 对外 IP | 必须是本机真实 IP，其他节点靠它来 join |

**预期结果**（成功末尾会出现类似）：

```
Your Kubernetes control-plane has initialized successfully!
...
kubeadm join 192.168.56.10:6443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:xxxxxxxxxxxxxxxxxxxxxxx
```

> **重要**：把上面这段 `kubeadm join ...` 完整复制保存下来，这是让工作节点加入集群的"门票"。如果忘了保存，后面 3.5 有补发命令。

## 3.2 配置 kubectl（master 上，让当前用户能操作集群）

```bash
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

# 验证
kubectl get nodes
# 预期输出：
# NAME         STATUS     ROLES           AGE   VERSION
# k8s-master   NotReady   control-plane   2m    v1.28.2
```
- **原理**：`/etc/kubernetes/admin.conf` 是集群管理员凭证（含 API Server 地址、证书、token）。拷贝到 `~/.kube/config` 后，kubectl 才能免密连接集群。
- **注意**：此时 STATUS 是 `NotReady` 是**正常现象**——因为还没装网络插件，等 3.4 装完 Calico 就会变成 `Ready`。

## 3.3 安装网络插件 Calico（master 上）

K8s 只负责调度，**Pod 之间跨节点怎么通信，需要网络插件（CNI）来做**。这里选最常用的 Calico。

```bash
# 下载 Calico 清单并应用
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml
```

> 若 GitHub 访问慢，可先 `curl -o calico.yaml` 下载后用 `sed` 把镜像换成国内源，或直接用 `kubectl apply -f calico.yaml`。核心思想一致：把 Calico 的 DaemonSet（每个节点跑一个 calico-node）装进 `kube-system` 命名空间。

**验证 Calico 是否就绪**：

```bash
kubectl get pods -n kube-system
# 预期：所有 Pod 都是 Running，calico-node-xxx 每个节点一个
# NAME                                       READY   STATUS    RESTARTS   AGE
# calico-kube-controllers-xxx                1/1     Running   0          2m
# calico-node-xxxxx                          1/1     Running   0          2m
# coredns-xxx                                1/1     Running   0          3m
```

再过约 1 分钟，节点会变成 Ready：

```bash
kubectl get nodes
# NAME         STATUS   ROLES           AGE   VERSION
# k8s-master   Ready    control-plane   4m    v1.28.2
```

## 3.4 工作节点加入集群（node1、node2 上执行）

在 node1、node2 上分别执行 3.1 末尾保存的 join 命令：

```bash
kubeadm join 192.168.56.10:6443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:xxxxxxxxxxxxxxxxxxxxxxx
```

**参数解释**：
- `192.168.56.10:6443`：API Server 地址和端口。
- `--token`：加入集群的一次性凭证。
- `--discovery-token-ca-cert-hash`：CA 证书指纹，用于校验 API Server 身份，防止中间人攻击。

**预期结果**：

```
This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.
```

## 3.5 补发 join 命令（忘了保存门票时）

回到 master：

```bash
# 查看已生成的 token
kubeadm token list
# 若 token 过期，重新生成一个
kubeadm token create --print-join-command
```

`--print-join-command` 会直接打印一条完整的、可直接粘贴的 join 命令，把这行拿到工作节点执行即可。

## 3.6 验证集群整体状态

```bash
kubectl get nodes -o wide
# 预期：
# NAME         STATUS   ROLES           AGE   VERSION   INTERNAL-IP
# k8s-master   Ready    control-plane   10m   v1.28.2   192.168.56.10
# k8s-node1    Ready    <none>          5m    v1.28.2   192.168.56.11
# k8s-node2    Ready    <none>          5m    v1.28.2   192.168.56.12

kubectl get pods -A
# -A 表示 --all-namespaces，查看所有命名空间下的 Pod，应全部 Running
```

- **`-o wide`**：输出更多列（这里多出 INTERNAL-IP），排查时常用。
- **`-A`**：跨所有命名空间。不加则只看 default 命名空间。

## 3.7 错误排查手册

| 现象 | 原因 | 解决办法 |
|---|---|---|
| `kubeadm init` 报 `swap on is not supported` | swap 没关干净 | 重新 `swapoff -a` 并检查 `/etc/fstab` |
| 报 `[ERROR FileContent--proc-sys-net-...]: /proc/sys/net/bridge/...` | 内核模块没加载 | 重新执行 2.2.4 的 `modprobe` 和 `sysctl --system` |
| 镜像拉取卡住 / `Timed out` | 官方镜像源被墙 | 确认 `--image-repository registry.aliyuncs.com/google_containers` 已加 |
| Pod 一直 `ContainerCreating` | pause 镜像拉不动 | 检查 containerd 的 `sandbox_image` 是否已换国内源 |
| 节点一直 `NotReady` | 网络插件没装好 | `kubectl -n kube-system get pods` 看 calico 是否 Running |
| join 报 `token expired` | token 过期（默认 24h） | master 上 `kubeadm token create --print-join-command` 重新生成 |
| kubelet 起不来 | cgroup 驱动不一致 | 确认 containerd `SystemdCgroup = true` 并 `systemctl restart containerd` |
| 报 `crictl` 相关错误 | 运行时没起来 | `systemctl status containerd` 排查 |
| `kubectl get nodes` 报连接拒绝 | 没配 kubectl 凭证 | 重新执行 3.2 的拷贝 config 步骤 |

**万能排查三板斧**：

```bash
# 1. 看 Pod 详细事件（最常用）
kubectl describe pod <pod名> -n kube-system

# 2. 看容器日志
kubectl logs <pod名> -n kube-system

# 3. 看节点/系统日志
journalctl -u kubelet -n 50 --no-pager
systemctl status kubelet
```

---

## 【动手练习 · 第3章】

1. 完整跑通 1 主 2 从集群，用 `kubectl get nodes` 确认 3 台机器全部 `Ready`。
2. 故意在 node1 上执行 `swapoff` 之前先看状态，理解 swap 检查的必要性（可用 `kubeadm reset` 清理后重试，理解 reset 的作用：`kubeadm reset` 会清空本机 K8s 痕迹，让你能重新 init/join）。
3. 用 `kubectl describe node k8s-node1` 查看节点容量（Capacity 字段），说出这台机器可分配多少 CPU 和内存。

---

# 第4章 核心资源详解

> 本章是重中之重。每个资源都配一个**可直接 `kubectl apply -f` 运行的 YAML**，并逐字段解释。请跟着敲一遍，边敲边理解。

## 4.0 先理解 YAML 与"四大字段"

K8s 的 YAML 清单几乎都有这四段：

```yaml
apiVersion: v1        # 1. 资源属于哪个 API 组/版本
kind: Pod             # 2. 资源类型
metadata:             # 3. 元数据：名字、标签、命名空间
  name: my-pod
  labels:
    app: demo
spec:                 # 4. 期望状态：这个资源具体长什么样
  containers: [...]
```

- **apiVersion / kind**：决定"这是个什么资源、用哪版接口"。
- **metadata.name**：资源唯一名，同类型同命名空间下不能重名。
- **metadata.labels**：标签（键值对），是 K8s 里"关联资源"的粘合剂（Service 靠它找 Pod，Deployment 靠它管理 Pod）。
- **spec**：期望状态的核心，每种资源 spec 内容不同。

---

## 4.1 Pod

### 是什么
最小调度单位，一个或多个共享网络与存储的容器。

### 为什么用
它是"容器"在 K8s 里的外壳，提供了 sidecar 模式、共享卷、探针等能力。

### 怎么用

```yaml
# pod-nginx.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod          # Pod 名字
  labels:
    app: nginx             # 标签，后面 Service 用它来选中这个 Pod
spec:
  containers:
  - name: nginx            # 容器名
    image: nginx:1.25      # 镜像
    ports:
    - containerPort: 80    # 容器监听端口（声明用，不强制映射）
    resources:             # 资源请求与上限
      requests:
        memory: "64Mi"
        cpu: "250m"        # 250m = 0.25 核
      limits:
        memory: "128Mi"
        cpu: "500m"
```

```bash
kubectl apply -f pod-nginx.yaml
kubectl get pods -o wide
# NAME        READY   STATUS    RESTARTS   AGE   IP
# nginx-pod   1/1     Running   0          10s   192.168.x.x

# 进去看看（Pod 内是容器视角）
kubectl exec -it nginx-pod -- bash
# 在容器内： curl localhost   → 应返回 nginx 欢迎页 HTML

# 删除
kubectl delete pod nginx-pod
```

**关键字段解释**：
- `resources.requests`：调度时保证的最低资源（scheduler 按它挑节点）。
- `resources.limits`：容器最多用多少，超出会被限流（CPU）或 OOM 杀掉（内存）。
- `cpu: 250m`：m 是毫核，1000m = 1 核。

**⚠️ 裸 Pod 的局限**：单独创建 Pod 最"脆弱"——被删了不会自动重建，滚动升级做不了。所以生产里几乎总是用 Deployment 来"包一层"管理 Pod。

---

## 4.2 Deployment（重点中的重点）

### 是什么
管理 Pod 副本的高级控制器，声明"我要 N 个某版本 Pod"。

### 为什么用
提供三大能力：**副本保持（自愈）、滚动更新、版本回滚**。这是 K8s 最核心的价值之一。

### 怎么用

```yaml
# deploy-nginx.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
  labels:
    app: nginx
spec:
  replicas: 3                    # 期望副本数：3 个
  selector:                      # 选择器：必须匹配下面 template 的 labels
    matchLabels:
      app: nginx
  template:                      # Pod 模板（和裸 Pod 的 spec 几乎一样）
    metadata:
      labels:
        app: nginx               # 这个 label 必须被 selector 匹配到
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
```

```bash
kubectl apply -f deploy-nginx.yaml
kubectl get deploy
# NAME           READY   UP-TO-DATE   AVAILABLE   AGE
# nginx-deploy   3/3     3            3           20s
# READY 3/3：3 个副本就绪；UP-TO-DATE 3：已更新到最新模板的 3 个；AVAILABLE 3：可用 3 个

kubectl get pods
# 会发现有 3 个 Pod，名字是 nginx-deploy-<随机串>，且可能分布在不同节点
```

**字段解释**：
- `replicas`：期望副本数，controller 会不停调和到 3。
- `selector.matchLabels`：Deployment 靠它"认领"哪些 Pod 归自己管。
- `template`：Pod 模板，Deployment 用它批量"复印"Pod。
- **自愈验证**：`kubectl delete pod nginx-deploy-xxx` 删掉一个，几秒后会发现又自动起了一个新的（因为实际 < 期望，controller 补上）。

### 4.2.1 滚动更新

```bash
# 把镜像从 1.25 升到 1.26（可加 --record 记录历史，新版用 rollout 记录）
kubectl set image deployment/nginx-deploy nginx=nginx:1.26
# 或编辑 YAML 改 image 再 apply

# 实时观察滚动更新过程
kubectl rollout status deployment/nginx-deploy
# 预期：
# Waiting for deployment "nginx-deploy" rollout to finish: 2 of 3 updated replicas are available...
# deployment "nginx-deploy" successfully rolled out
```

**滚动更新原理**：Deployment 先创建一个新的 ReplicaSet（副本集），逐步"先起新 Pod、就绪后再停旧 Pod"，全程不中断服务。`maxSurge`（最多多出几个）和 `maxUnavailable`（最多几个不可用）控制节奏。

### 4.2.2 版本回滚

```bash
# 查看发布历史
kubectl rollout history deployment/nginx-deploy
# REVISION  CHANGE-CAUSE
# 1         <none>
# 2         <none>

# 回滚到上一个版本
kubectl rollout undo deployment/nginx-deploy

# 回滚到指定版本（比如版本 1）
kubectl rollout undo deployment/nginx-deploy --to-revision=1

# 验证镜像已回退
kubectl get deploy nginx-deploy -o jsonpath='{.spec.template.spec.containers[0].image}'
# 输出：nginx:1.25
```

- **为什么能回滚**：Deployment 每次更新都保留历史 ReplicaSet（旧版本 Pod 的"模具"），undo 就是切回旧模具。
- **`jsonpath`**：从资源 JSON 里精确取值，排查时非常实用。

---

## 4.3 ReplicaSet（RS）

### 是什么
只做一件事的控制器：**保证指定数量的 Pod 副本始终存在**。

### 为什么用
其实你几乎**不直接**创建 RS——Deployment 内部会自动为每个版本生成一个 RS。理解它才能看懂"回滚"机制。直接用它只有"副本数管理"，没有滚动更新/回滚能力。

### 怎么用

```yaml
# rs-nginx.yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-rs
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-rs
  template:
    metadata:
      labels:
        app: nginx-rs
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
```

```bash
kubectl apply -f rs-nginx.yaml
kubectl get rs
# NAME       DESIRED   CURRENT   READY   AGE
# nginx-rs   2         2         2       10s

# 看 Deployment 内部自动生成的 RS
kubectl get rs
# NAME                      DESIRED   CURRENT   READY   AGE
# nginx-deploy-xxxxxxxx     0         0         0       5m   ← 旧版本（已回滚，副本归零）
# nginx-deploy-yyyyyyyy     3         3         3       1m   ← 当前版本
```

**关系链**：`Deployment → 管理 ReplicaSet → 管理 Pod`。三者是"套娃"关系。

---

## 4.4 StatefulSet

### 是什么
管理**有状态**应用的控制器。

### 为什么用
Deployment 的 Pod 是"无差别易耗品"（名字随机、可随时替换）。但数据库这类应用需要：**稳定的网络标识（固定主机名）、稳定的持久化存储、有序的启动/停止**。StatefulSet 就是干这个的。

### 怎么用

```yaml
# sts-nginx.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"       # 必须：关联一个 Headless Service（下面解释）
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:      # 每个 Pod 自动创建独立的 PVC
  - metadata:
      name: www
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
```

配套的 Headless Service（`clusterIP: None`）：

```yaml
# headless-nginx.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  clusterIP: None            # Headless：不给虚拟 IP，DNS 直接解析到每个 Pod IP
  selector:
    app: nginx
  ports:
  - port: 80
```

```bash
kubectl apply -f headless-nginx.yaml
kubectl apply -f sts-nginx.yaml
kubectl get pods -l app=nginx
# NAME    READY   STATUS    RESTARTS   AGE
# web-0   1/1     Running   0          20s
# web-1   1/1     Running   0          15s
# web-2   1/1     Running   0          10s
```

**与 Deployment 的关键区别**：
- Pod 名是**有序固定的**：`web-0`、`web-1`、`web-2`（不是随机串），且 0→1→2 顺序启动，删除时倒序。
- 每个 Pod 有**自己的 PVC**（`www-web-0`、`www-web-1`...），Pod 重建后还是挂回同一块盘，数据不丢。
- 配合 Headless Service，每个 Pod 有稳定 DNS：`web-0.nginx.default.svc.cluster.local`。

---

## 4.5 DaemonSet

### 是什么
保证**每个节点都恰好跑一个**该 Pod 副本。

### 为什么用
需要"每台机器一份"的组件：日志采集（Filebeat/Fluentd）、监控代理（node-exporter）、网络插件（Calico 的 calico-node 本身就是 DaemonSet）。

### 怎么用

```yaml
# ds-nginx.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-nginx
spec:
  selector:
    matchLabels:
      name: node-nginx
  template:
    metadata:
      labels:
        name: node-nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
```

```bash
kubectl apply -f ds-nginx.yaml
kubectl get ds
# NAME         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR
# node-nginx   2         2         2       2            2           <none>
# DESIRED=2：因为只有 2 个工作节点（DaemonSet 默认不在控制平面节点上跑，除非加容忍）

kubectl get pods -o wide -l name=node-nginx
# 每个工作节点各一个 Pod，NODE 列对应不同节点
```

- **特点**：节点增删时，DaemonSet 自动在新增节点上补一个、随删除节点撤一个，无需手工干预。

---

## 4.6 Job 与 CronJob

### 是什么
- **Job**：跑**一次性**任务，成功即结束（Pod 退出码 0）。
- **CronJob**：按时间表（Cron 表达式）**定时**创建 Job。

### 为什么用
跑批处理、数据备份、报表生成这类"跑完就退出"的任务，不适合用常驻的 Deployment。

### 怎么用

```yaml
# job-pi.yaml：计算圆周率（跑完就退出）
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  backoffLimit: 4        # 失败最多重试 4 次
  template:
    spec:
      restartPolicy: Never   # Job 用 Never/OnFailure，不能用 Always
      containers:
      - name: pi
        image: perl:5.34
        command: ["perl", "-Mbignum=bpi", "-wle", "print bpi(200)"]
```

```bash
kubectl apply -f job-pi.yaml
kubectl get jobs
# NAME   COMPLETIONS   DURATION   AGE
# pi     1/1           8s         12s

kubectl logs -l job-name=pi   # 查看任务输出（圆周率 200 位）
```

CronJob 示例（每分钟打印一次时间）：

```yaml
# cronjob-hello.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/1 * * * *"    # Cron 表达式：每 1 分钟
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: hello
            image: busybox:1.36
            command: ["sh", "-c", "date; echo Hello from CronJob"]
```

```bash
kubectl apply -f cronjob-hello.yaml
kubectl get cronjob
# NAME    SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
# hello   */1 * * * *   False     0        <none>          5s
# 等 1 分钟后：
kubectl get jobs   # 会看到 hello-<时间戳> 的 Job 被创建
```

- **Cron 表达式 `*/1 * * * *`**：5 个字段分别是「分 时 日 月 星期」，`*/1` 表示每 1 分钟。

---

## 4.7 Service（重中之重）

### 是什么
给一组 Pod 提供**稳定访问入口**的抽象。

### 为什么用
Pod 是易逝的：重建后 IP 会变、扩容后多出一堆 IP。Service 通过 `selector` 选中一组 Pod，给它们一个**固定不变的虚拟 IP（ClusterIP）**，并做负载均衡。你永远通过 Service 访问，不必关心背后 Pod 的 IP 变化。

### 类型对比

| 类型 | 访问范围 | 使用场景 |
|---|---|---|
| **ClusterIP** | 仅集群内部 | 集群内部服务互访（默认类型） |
| **NodePort** | 集群外（节点 IP:端口） | 无云 LB 时暴露服务到外部 |
| **LoadBalancer** | 集群外（云 LB 公网 IP） | 云厂商环境（阿里云/腾讯云 SLB） |
| **ExternalName** | 集群内 → 外部 DNS | 把外部服务映射进集群 |

### 4.7.1 ClusterIP（默认）

```yaml
# svc-clusterip.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
spec:
  type: ClusterIP
  selector:
    app: nginx          # 选中所有 label app=nginx 的 Pod
  ports:
  - port: 80            # Service 对外端口
    targetPort: 80      # 转发到 Pod 的哪个端口
    protocol: TCP
```

```bash
kubectl apply -f svc-clusterip.yaml
kubectl get svc
# NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
# kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP   1d   ← 集群自带
# nginx-svc    ClusterIP   10.101.56.33    <none>        80/TCP    10s

# 集群内访问验证（在任意 Pod 里 curl 这个虚拟 IP）
kubectl run test --rm -it --image=busybox:1.36 -- sh
# 进入后： wget -qO- http://10.101.56.33   → 返回 nginx 首页
```

- **`port` vs `targetPort`**：`port` 是 Service 自己的端口，`targetPort` 是它转发到 Pod 的端口。两者可以不同（如 8080 → 80）。
- **selector 如何工作**：Service 有个 Endpoints 对象，实时记录 `app=nginx` 的那些 Pod IP；流量按 Endpoints 负载均衡分发。

### 4.7.2 NodePort

```yaml
# svc-nodeport.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080     # 30000-32767 范围，所有节点都会监听此端口
```

```bash
kubectl apply -f svc-nodeport.yaml
kubectl get svc nginx-nodeport
# NAME             TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
# nginx-nodeport   NodePort   10.99.1.2      <none>        80:30080/TCP   5s

# 在集群外（你自己电脑）浏览器访问： http://192.168.56.10:30080
# 或访问任意节点： http://192.168.56.11:30080 —— 都能通
```

- **原理**：NodePort 会在每个节点上开一个端口（30080），无论流量打到哪台节点，都会被转发到目标 Pod。NodePort 建立在 ClusterIP 之上。

### 4.7.3 LoadBalancer

```yaml
# svc-lb.yaml（仅云环境/装了 MetalLB 才有效）
apiVersion: v1
kind: Service
metadata:
  name: nginx-lb
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
```

```bash
kubectl apply -f svc-lb.yaml
kubectl get svc nginx-lb
# 云环境会分配 EXTERNAL-IP（公网/内网 SLB 地址）
# 自建裸机集群 EXTERNAL-IP 会一直 <pending>（因为没有云 LB 可调）
```

- **原理**：LoadBalancer 会自动调用云厂商 API 创建一个负载均衡器，把流量打到各节点。裸机集群想用它需装 MetalLB 等实现。

---

## 4.8 Ingress

### 是什么
集群的**七层（HTTP/HTTPS）入口网关**，按域名和路径把外部流量路由到不同 Service。

### 为什么用
NodePort 每个服务占一个端口、LoadBalancer 每个服务占一个 LB（烧钱）。Ingress 用**一个入口 + 一套路由规则**，就能把 `a.com` 转发到服务 A、`b.com/api` 转发到服务 B，还能做 TLS 终结、路径重写。

### 怎么用
Ingress 需要先装一个 **Ingress Controller**（本教程用 NGINX Ingress）：

```bash
# 安装 ingress-nginx（国内可先下载再改镜像源）
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.9.4/deploy/static/provider/cloud/deploy.yaml

# 验证
kubectl get pods -n ingress-nginx
# NAME                                       READY   STATUS    RESTARTS   AGE
# ingress-nginx-controller-xxx               1/1     Running   0          1m
```

写 Ingress 规则：

```yaml
# ingress-nginx.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
spec:
  ingressClassName: nginx        # 指定用哪个 Ingress Controller
  rules:
  - host: demo.example.com       # 域名
    http:
      paths:
      - path: /                  # 路径
        pathType: Prefix         # 前缀匹配
        backend:
          service:
            name: nginx-svc      # 转发到哪个 Service
            port:
              number: 80
```

```bash
kubectl apply -f ingress-nginx.yaml
kubectl get ingress
# NAME            CLASS   HOSTS               ADDRESS      PORTS   AGE
# nginx-ingress   nginx   demo.example.com    192.168.x.x  80      10s

# 访问：在本地 hosts 里把 demo.example.com 解析到 Ingress Controller 的 NodePort IP，
# 然后浏览器打开 http://demo.example.com 即可看到 nginx 页面
```

- **字段解释**：`host` 匹配域名，`path` 匹配路径，`backend.service` 指定目标 Service。`ingressClassName` 关联具体的 Controller 实现。

---

## 4.9 ConfigMap

### 是什么
存**非敏感配置**（键值对或整段文件）的对象。

### 为什么用
把配置从镜像里剥离：改配置不用重新打镜像、重启容器，改 ConfigMap 即可。实现"配置与代码分离"。

### 怎么用

```yaml
# cm-app.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:                      # data 下是键值对
  APP_ENV: "production"
  DB_HOST: "mysql-svc"
  nginx.conf: |            # 也可以存整段配置文件
    server {
      listen 80;
      location / { return 200 "ok"; }
    }
```

```bash
kubectl apply -f cm-app.yaml
kubectl get cm app-config -o yaml   # 查看内容

# 用法1：作为环境变量注入 Pod
kubectl run env-pod --image=busybox:1.36 --dry-run=client -o yaml \
  --env APP_ENV=production \
  --restart=Never > /dev/null   # 仅演示，实际用下面的 YAML
```

**挂载到 Pod 的完整示例**（作为文件挂载）：

```yaml
# pod-cm.yaml
apiVersion: v1
kind: Pod
metadata:
  name: cm-pod
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    envFrom:                       # 方式1：整个 ConfigMap 的所有键都变环境变量
    - configMapRef:
        name: app-config
    volumeMounts:                  # 方式2：把 ConfigMap 作为文件挂载
    - name: config-volume
      mountPath: /etc/config
  volumes:
  - name: config-volume
    configMap:
      name: app-config
```

```bash
kubectl apply -f pod-cm.yaml
kubectl exec cm-pod -- cat /etc/config/nginx.conf   # 读到配置文件内容
kubectl exec cm-pod -- env | grep APP_ENV           # 看到 APP_ENV=production
```

---

## 4.10 Secret

### 是什么
存**敏感数据**（密码、token、证书）的对象，数据用 base64 编码存储。

### 为什么用
不要把密码明文写进 YAML 或镜像。Secret 与 ConfigMap 用法几乎一样，但专为敏感数据设计（可结合 RBAC 限制谁能读、可选 etcd 加密）。

### 怎么用

```bash
# 先 base64 编码密码
echo -n 'MyPass@123' | base64
# 输出：TXlQYXNzQDEyMw==
```

```yaml
# secret-db.yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque                 # Opaque = 通用键值类型
data:
  MYSQL_ROOT_PASSWORD: TXlQYXNzQDEyMw==   # base64 编码后的值
  MYSQL_DATABASE: bG5tcA==               # "lnmp" 的 base64
```

```bash
kubectl apply -f secret-db.yaml
kubectl get secret db-secret
kubectl get secret db-secret -o jsonpath='{.data.MYSQL_ROOT_PASSWORD}' | base64 -d
# 输出：MyPass@123  （解码回明文，验证正确）
```

在 Pod 里用：

```yaml
# pod-secret.yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-pod
spec:
  containers:
  - name: mysql
    image: mysql:8.0
    env:
    - name: MYSQL_ROOT_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: MYSQL_ROOT_PASSWORD
```

- **字段解释**：`data` 存 base64 编码值；也可以用 `stringData` 直接写明文（K8s 自动帮你编码）。`valueFrom.secretKeyRef` 把 Secret 的某个键注入环境变量。
- **安全提示**：base64 只是编码不是加密，默认 Secret 在 etcd 里是明文存储的；生产环境要开 etcd 静态加密 + RBAC 授权。

---

## 4.11 PV / PVC / StorageClass（存储三件套）

### 是什么（三层模型）

| 概念 | 角色 | 类比 |
|---|---|---|
| **PV（PersistentVolume）** | 集群里的一块"真实存储" | 停车场里的"车位" |
| **PVC（PersistentVolumeClaim）** | 用户对存储的"申请单" | 停车需求（我要 1 个 5GB 车位） |
| **StorageClass** | 存储"自动供应器" | 物业（按需求自动新建车位） |

### 为什么用
Pod 是易逝的，容器内的数据在 Pod 删除后丢失。PV/PVC 把"存储"从"Pod"里独立出来，让数据持久化，Pod 重建后还能挂回同一份数据。StorageClass 实现"动态供应"，免去手工一个个建 PV。

### 怎么用（先手工建 PV/PVC，理解绑定）

```yaml
# pv-local.yaml（用本机 hostPath 模拟，仅实验用；生产用 NFS/云盘）
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-1gi
spec:
  capacity:
    storage: 1Gi          # 容量
  accessModes:
  - ReadWriteOnce         # 只允许单节点读写
  hostPath:
    path: /data/pv1       # 本机目录
  persistentVolumeReclaimPolicy: Retain   # 删除 PVC 后保留数据
```

```yaml
# pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-1gi
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi        # 申请 1Gi，会绑定到满足条件的 PV
```

```bash
kubectl apply -f pv-local.yaml
kubectl apply -f pvc.yaml
kubectl get pv
# NAME     CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM
# pv-1gi   1Gi        RWO            Retain           Bound       default/pvc-1gi
# STATUS=Bound 表示已绑定
kubectl get pvc
# NAME      STATUS   VOLUME   CAPACITY   ACCESS MODES   AGE
# pvc-1gi   Bound    pv-1gi   1Gi        RWO            10s
```

在 Pod 里挂载 PVC：

```yaml
# pod-pvc.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pvc-pod
spec:
  containers:
  - name: busybox
    image: busybox:1.36
    command: ["sh", "-c", "echo hello > /data/hello.txt && sleep 3600"]
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: pvc-1gi
```

```bash
kubectl apply -f pod-pvc.yaml
kubectl exec pvc-pod -- cat /data/hello.txt   # 输出 hello
```

### StorageClass（动态供应）

```yaml
# sc-local.yaml（local-path-provisioner 类，实验用；生产用云盘/NFS provisioner）
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: rancher.io/local-path   # 供应器，需对应已安装的 provisioner
reclaimPolicy: Delete
```

```yaml
# pvc-dynamic.yaml：PVC 引用 StorageClass，PV 自动创建
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-dynamic
spec:
  storageClassName: fast     # 引用上面的 StorageClass
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
```

```bash
kubectl apply -f sc-local.yaml
kubectl apply -f pvc-dynamic.yaml
kubectl get pvc pvc-dynamic
# 会看到动态自动创建了一个 PV 并 Bound，无需手工建 PV
```

- **原理**：PVC 带着 `storageClassName` 来申请时，StorageClass 对应的 provisioner 会自动创建 PV，再完成绑定——这就是"按需供应"。

---

## 【动手练习 · 第4章】

1. 依次跑通 4.1 Pod、4.2 Deployment（含副本、滚动更新、回滚三个操作）、4.4 StatefulSet、4.5 DaemonSet、4.6 Job/CronJob。
2. 验证 Service 三种类型：ClusterIP（集群内访问）、NodePort（外部访问）、LoadBalancer（观察 EXTERNAL-IP pending）。
3. 把 ConfigMap 的 `APP_ENV` 值改成 `test`，用 `kubectl rollout restart deployment/nginx-deploy` 重启后，验证环境变量是否更新（提示：ConfigMap 更新不会自动同步到已运行 Pod）。
4. 创建一个 Secret 并用 `kubectl get secret xxx -o yaml` 查看它的 base64 值，再解码回明文。
5. 走一遍 PV → PVC → Pod 挂载，写入文件后删除 Pod、重建，验证数据仍在。

---

# 第5章 kubectl 常用命令手册

## 5.1 命令速查表

| 命令 | 作用 | 高频场景 |
|---|---|---|
| `get` | 列出资源 | 看状态 |
| `describe` | 查看资源详情/事件 | 排查故障 |
| `create` | 创建资源 | 单条创建 |
| `apply` | 声明式应用 YAML | **最常用**（可创建可更新） |
| `delete` | 删除资源 | 清理 |
| `logs` | 查看容器日志 | 排查应用问题 |
| `exec -it` | 进入容器 | 调试 |
| `scale` | 调整副本数 | 扩容缩容 |
| `rollout` | 管理发布（历史/状态/回滚） | 发布与回滚 |
| `set image` | 更新容器镜像 | 快速升级 |
| `port-forward` | 端口转发 | 本地调试访问 |

## 5.2 逐条详解

### get

```bash
kubectl get pods                       # 当前命名空间 Pod
kubectl get pods -A                    # 所有命名空间
kubectl get pods -o wide               # 更多列（IP、节点）
kubectl get pods -l app=nginx          # 按 label 过滤
kubectl get all                        # 常用资源一网打尽
kubectl get nodes,deploy,svc,pv,pvc    # 一次查多种（逗号分隔）
kubectl get pods -w                    # watch 实时刷新（Ctrl+C 退出）
```
- `-o wide`：多显示 IP、所在节点等列。
- `-l key=value`：标签选择器，只显示带该标签的资源。
- `-w`：持续监听变化，适合看滚动更新。

### describe（排查第一选择）

```bash
kubectl describe pod nginx-pod
kubectl describe node k8s-node1
```
- **输出重点**：底部 `Events` 段落记录了资源从创建到现在发生的事件（拉镜像失败、调度失败、探针失败都会写在这），是排查的"案发现场"。

### create / apply（区别很重要）

```bash
kubectl create -f pod.yaml     # 命令式：只能创建，已存在会报错
kubectl apply  -f pod.yaml     # 声明式：创建或更新，幂等（推荐）
```
- **为什么推荐 apply**：apply 会记录上次应用的配置（last-applied-configuration），重复执行不会报错，改 YAML 后 apply 会增量更新。create 重复执行会 `AlreadyExists` 报错。

### delete

```bash
kubectl delete pod nginx-pod
kubectl delete -f deploy-nginx.yaml       # 按文件删除
kubectl delete deploy nginx-deploy        # 按类型+名称删除
kubectl delete pods --all                 # 删当前命名空间所有 Pod
kubectl delete pvc pvc-1gi --grace-period=0 --force   # 强制删除（慎用）
```

### logs

```bash
kubectl logs nginx-pod
kubectl logs nginx-pod -c nginx           # 多容器时指定容器
kubectl logs -f nginx-pod                 # 持续跟踪（tail -f 效果）
kubectl logs --tail=20 nginx-pod          # 只看最后 20 行
kubectl logs -l app=nginx                 # 看某个 label 下所有 Pod（新版支持）
```

### exec -it（进入容器）

```bash
kubectl exec -it nginx-pod -- bash
# 进去后就是一个 shell，可以 curl、cat、ps 等
kubectl exec nginx-pod -- ls /usr/share/nginx/html   # 单条命令不交互
```
- `-it`：`-i` 交互 + `-t` 分配伪终端。`--` 后面是容器内要执行的命令。

### scale（扩容/缩容）

```bash
kubectl scale deployment nginx-deploy --replicas=5    # 扩到 5 个
kubectl get pods -l app=nginx                         # 看到 5 个 Pod
kubectl scale deployment nginx-deploy --replicas=2    # 缩回 2 个
```

### rollout（发布管理）

```bash
kubectl rollout status deployment/nginx-deploy   # 实时看发布进度
kubectl rollout history deployment/nginx-deploy  # 看发布历史版本
kubectl rollout undo deployment/nginx-deploy     # 回滚上一版
kubectl rollout undo deployment/nginx-deploy --to-revision=1  # 回滚到指定版本
kubectl rollout restart deployment/nginx-deploy  # 原地重启（无镜像变更时刷新）
```

### set image（快速改镜像）

```bash
kubectl set image deployment/nginx-deploy nginx=nginx:1.27
# 格式：deployment/名称  容器名=新镜像
# 等价于改 YAML 的 image 字段再 apply
```

### port-forward（本地调试神器）

```bash
kubectl port-forward pod/nginx-pod 8080:80
# 转发后，本地浏览器访问 http://127.0.0.1:8080 就能直达 Pod 的 80 端口
# 相当于在本地和 Pod 之间打了条隧道，无需暴露 NodePort
# 也可转发 Service：
kubectl port-forward svc/nginx-svc 9090:80
```

---

## 【动手练习 · 第5章】

1. 用 `kubectl get pods -o wide` 找出任意一个 Pod 的 IP 和所在节点，再用 `kubectl describe pod <名>` 看它的 Events。
2. 用 `kubectl scale` 把副本从 3 扩到 6，再用 `kubectl get pods -w` 观察自动创建过程。
3. 用 `kubectl set image` 升级镜像，再用 `rollout status` 看进度，最后 `rollout undo` 回滚。
4. 用 `port-forward` 把某个 Service 转发到本地端口，用浏览器验证能访问。

---

# 第6章 国内镜像源与拉取加速

## 6.1 为什么要配国内源

K8s 生态的镜像主要托管在 `registry.k8s.io`（官方）和 `docker.io`（Docker Hub），这两个在国内经常**超时或无法访问**，表现为：

- `kubeadm init` 卡在拉 `kube-apiserver`、`etcd` 等镜像。
- Pod 卡在 `ContainerCreating`，`describe` 里看到 `Failed to pull image`。

## 6.2 常用国内镜像源对照

| 官方镜像 | 国内替代（阿里云） |
|---|---|
| `registry.k8s.io/kube-apiserver:v1.28.2` | `registry.aliyuncs.com/google_containers/kube-apiserver:v1.28.2` |
| `registry.k8s.io/pause:3.9` | `registry.aliyuncs.com/google_containers/pause:3.9` |
| `registry.k8s.io/coredns:v1.10.1` | `registry.aliyuncs.com/google_containers/coredns:v1.10.1` |
| `docker.io/library/nginx:1.25` | `docker.m.daocloud.io/library/nginx:1.25` |

其他常用加速地址：`docker.m.daocloud.io`、`dockerproxy.com`、`docker.nju.edu.cn`、`quay.mirrors.ustc.edu.cn`（针对 quay.io）。

## 6.3 三种加速手段

### 手段1：kubeadm init 加镜像仓库参数（K8s 组件）

```bash
kubeadm init --image-repository registry.aliyuncs.com/google_containers
```
- 只影响 kubeadm 拉取的控制平面组件镜像，业务镜像（nginx、mysql 等）不受它管。

### 手段2：containerd 配置镜像加速（业务镜像）

编辑 `/etc/containerd/config.toml`，在 `[plugins."io.containerd.grpc.v1.cri".registry.mirrors]` 下加：

```toml
[plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
  endpoint = ["https://docker.m.daocloud.io", "https://docker.nju.edu.cn"]
```

重启 containerd：

```bash
systemctl restart containerd
```

- **原理**：containerd 拉取 `docker.io` 的镜像时，会先尝试这些 mirror 端点，实现加速。
- **验证**：`kubectl run test --image=nginx:1.25 --rm -it --command -- sh`，能快速拉起来就说明加速生效。

### 手段3：手工拉取 + 重新打 tag（万能兜底）

当某个镜像实在拉不动时：

```bash
# 1. 从国内源拉
ctr -n k8s.io images pull docker.m.daocloud.io/library/nginx:1.25
# 2. 重新打回官方 tag（让 K8s 能按原名找到）
ctr -n k8s.io images tag docker.m.daocloud.io/library/nginx:1.25 docker.io/library/nginx:1.25
```

---

## 【动手练习 · 第6章】

1. 查看 containerd 当前配置的镜像加速：`grep -A3 mirrors /etc/containerd/config.toml`。
2. 拉取一个较大的镜像（如 `mysql:8.0`）对比配置加速前后的耗时。
3. 用 `ctr -n k8s.io images list` 查看节点上已有的镜像，确认 containerd 确实从国内源拉到了镜像。

---

# 第7章 Helm 包管理器

## 7.1 Helm 是什么、为什么用

- **是什么**：K8s 的"包管理器"，相当于 apt/yum 之于 Linux、npm 之于 Node。
- **为什么用**：手工 `apply` 一堆 YAML 时，每个参数都要改、版本难管、卸载要逐个 delete。Helm 把这些 YAML 打包成 **Chart**，一条命令安装/升级/卸载/回滚，还能用 `values.yaml` 统一改参数。

核心概念：

| 概念 | 是什么 |
|---|---|
| **Chart** | 一个应用的完整打包（模板 + 默认配置），类似"安装包" |
| **Release** | Chart 的一次安装实例（同一 Chart 可装多个 Release） |
| **Repository（repo）** | 存放 Chart 的仓库（类似软件源） |
| **values.yaml** | 配置参数文件，安装时覆盖 Chart 默认值 |

## 7.2 安装 Helm

```bash
# 下载并解压（版本按需，这里是 v3.14.4）
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
bash get_helm.sh
# 国内 GitHub 慢时，用国内二进制镜像：
# wget https://mirrors.huaweicloud.com/helm/v3.14.4/helm-v3.14.4-linux-amd64.tar.gz
# tar -zxvf helm-v3.14.4-linux-amd64.tar.gz
# mv linux-amd64/helm /usr/local/bin/helm

helm version
# version.BuildInfo{Version:"v3.14.4", GitCommit:"...", GoVersion:"go1.21"}
```

## 7.3 添加仓库并搜索

```bash
# 添加官方源（国内慢可用镜像源）
helm repo add bitnami https://charts.bitnami.com/bitnami
# 国内加速源（阿里云）：
helm repo add aliyun https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts

helm repo list          # 查看已添加的仓库
helm repo update        # 更新仓库索引
helm search repo nginx  # 搜索 nginx 相关 Chart
```

## 7.4 安装一个应用（helm install）

```bash
# 安装 bitnami 的 nginx Chart，Release 名叫 my-nginx
helm install my-nginx bitnami/nginx

# 用自定义 values 覆盖默认配置（如改副本数、service 类型）
cat > my-values.yaml <<EOF
replicaCount: 2
service:
  type: NodePort
EOF
helm install my-nginx bitnami/nginx -f my-values.yaml

# 查看已安装的 Release
helm list
# NAME      NAMESPACE   REVISION  UPDATED  STATUS    CHART         APP VERSION
# my-nginx  default     1         ...      deployed  nginx-15.x.x  1.25.x
```

- **`-f values.yaml`**：用自定义文件覆盖 Chart 默认参数。也可以 `--set replicaCount=3` 直接命令行改单个值。

## 7.5 升级与回滚

```bash
# 升级：改副本数为 3
helm upgrade my-nginx bitnami/nginx --set replicaCount=3

# 查看历史版本
helm history my-nginx
# REVISION  UPDATED  STATUS      CHART  DESCRIPTION
# 1         ...      superseded  ...    Install complete
# 2         ...      deployed    ...    Upgrade complete

# 回滚到第 1 版
helm rollback my-nginx 1
```

## 7.6 卸载

```bash
helm uninstall my-nginx
# release "my-nginx" uninstalled
```

---

## 【动手练习 · 第7章】

1. 安装 Helm 并添加阿里云仓库。
2. 用 `helm install` 装一个 nginx Chart，指定 `service.type=NodePort`，从外部访问验证。
3. 执行一次 `helm upgrade` 改参数，再 `helm rollback` 回滚，用 `helm history` 查看版本变化。
4. `helm uninstall` 卸载，确认资源被清理干净。

---

# 第8章 完整项目：把 LNMP 迁移到 Kubernetes

> 目标：把经典的 LNMP（Nginx + PHP-FPM + MySQL）应用完整跑在 K8s 上。
> 架构：Nginx 作为入口（Deployment + Service + Ingress）→ 转发到 PHP-FPM（Deployment + Service）→ 访问 MySQL（StatefulSet + Service + PV）。配置用 ConfigMap，密码用 Secret。

## 8.1 项目架构图

```
外部用户 → Ingress(域名) → Nginx Service → Nginx Pod(处理静态+反代)
                                              │ fastcgi_pass
                                              ▼
                                      php-fpm Service → PHP Pod(执行 PHP)
                                              │ mysqli
                                              ▼
                                      mysql Service → MySQL Pod(StatefulSet + PVC)
```

## 8.2 创建命名空间

```bash
kubectl create namespace lnmp
kubectl config set-context --current --namespace lnmp   # 后续命令默认在 lnmp 命名空间
kubectl get ns lnmp
```

## 8.3 创建 Secret（数据库密码）

```bash
# 编码密码
echo -n 'lnmp_password_2024' | base64
# 得到：bG5tcF9wYXNzd29yZF8yMDI0
```

```yaml
# 01-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
  namespace: lnmp
type: Opaque
data:
  MYSQL_ROOT_PASSWORD: bG5tcF9wYXNzd29yZF8yMDI0
  MYSQL_DATABASE: bG5tcA==      # "lnmp"
  MYSQL_USER: bG5tcA==          # "lnmp"
  MYSQL_PASSWORD: bG5tcF9wYXNzd29yZF8yMDI0
```

```bash
kubectl apply -f 01-secret.yaml
```

## 8.4 创建 ConfigMap（Nginx 配置 + PHP 站点文件）

```yaml
# 02-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: lnmp-config
  namespace: lnmp
data:
  nginx.conf: |
    server {
      listen 80;
      server_name _;
      root /usr/share/nginx/html;
      index index.php index.html;
      location ~ \.php$ {
        fastcgi_pass php-fpm-svc:9000;   # 指向 php-fpm 的 Service
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME /usr/share/nginx/html$fastcgi_script_name;
      }
    }
  index.php: |
    <?php
    $host = 'mysql-svc';
    $user = 'lnmp';
    $pass = 'lnmp_password_2024';
    $db   = 'lnmp';
    $conn = new mysqli($host, $user, $pass, $db);
    if ($conn->connect_error) {
        die("连接失败: " . $conn->connect_error);
    }
    echo "✅ K8s LNMP 运行正常！当前 MySQL 连接成功，服务器时间：" . date("Y-m-d H:i:s");
    ?>
```

**字段解释**：
- `nginx.conf`：Nginx 站点配置，`location ~ \.php$` 匹配 PHP 文件，`fastcgi_pass php-fpm-svc:9000` 把 PHP 请求转发给 php-fpm 的 Service（ClusterIP 在集群内可直接用服务名访问，这就是 Service 的价值）。
- `index.php`：测试页面，用 mysqli 连数据库，验证整条链路。

```bash
kubectl apply -f 02-configmap.yaml
```

## 8.5 部署 MySQL（StatefulSet + Service + PV）

```yaml
# 03-mysql.yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-svc
  namespace: lnmp
spec:
  clusterIP: None                # Headless，供 StatefulSet 稳定网络标识
  selector:
    app: mysql
  ports:
  - port: 3306
    targetPort: 3306
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: lnmp
spec:
  serviceName: mysql-svc
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: MYSQL_ROOT_PASSWORD
        - name: MYSQL_DATABASE
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: MYSQL_DATABASE
        - name: MYSQL_USER
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: MYSQL_USER
        - name: MYSQL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: MYSQL_PASSWORD
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: mysql-data
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: mysql-data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
```

```bash
kubectl apply -f 03-mysql.yaml
kubectl get pods -l app=mysql -w   # 等 READY 1/1（mysql 首次初始化要 30s~1min）
```

## 8.6 部署 PHP-FPM（Deployment + Service）

```yaml
# 04-phpfpm.yaml
apiVersion: v1
kind: Service
metadata:
  name: php-fpm-svc
  namespace: lnmp
spec:
  selector:
    app: php-fpm
  ports:
  - port: 9000
    targetPort: 9000
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-fpm
  namespace: lnmp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: php-fpm
  template:
    metadata:
      labels:
        app: php-fpm
    spec:
      containers:
      - name: php-fpm
        image: php:8.2-fpm
        volumeMounts:
        - name: site
          mountPath: /usr/share/nginx/html    # 挂载 index.php
      volumes:
      - name: site
        configMap:
          name: lnmp-config
          items:
          - key: index.php
            path: index.php
```

```bash
kubectl apply -f 04-phpfpm.yaml
kubectl get pods -l app=php-fpm
```

## 8.7 部署 Nginx（Deployment + Service + Ingress）

```yaml
# 05-nginx.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: lnmp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
        volumeMounts:
        - name: nginx-conf
          mountPath: /etc/nginx/conf.d/default.conf   # 覆盖默认配置
          subPath: nginx.conf
        - name: site
          mountPath: /usr/share/nginx/html            # 挂载 index.php
      volumes:
      - name: nginx-conf
        configMap:
          name: lnmp-config
          items:
          - key: nginx.conf
            path: nginx.conf
      - name: site
        configMap:
          name: lnmp-config
          items:
          - key: index.php
            path: index.php
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
  namespace: lnmp
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: lnmp-ingress
  namespace: lnmp
spec:
  ingressClassName: nginx
  rules:
  - host: lnmp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-svc
            port:
              number: 80
```

```bash
kubectl apply -f 05-nginx.yaml
kubectl get pods -l app=nginx
kubectl get svc -n lnmp
```

## 8.8 验证 LNMP 全链路

```bash
# 方式1：NodePort 直接访问（本机浏览器打开）
# http://192.168.56.10:30080/index.php
# 预期页面显示：✅ K8s LNMP 运行正常！当前 MySQL 连接成功...

# 方式2：Ingress 域名访问（先改本地 hosts，把 lnmp.example.com 指向 ingress-nginx 的 NodePort IP）
# http://lnmp.example.com/index.php

# 方式3：port-forward 快速验证
kubectl port-forward -n lnmp svc/nginx-svc 8080:80
# 浏览器访问 http://127.0.0.1:8080/index.php
```

如果页面能显示"MySQL 连接成功"，说明 Nginx → PHP-FPM → MySQL 三层全部打通。

## 8.9 滚动更新验证

```bash
# 把 PHP 从 8.2 升到 8.3
kubectl set image deployment/php-fpm php-fpm=php:8.3-fpm

# 实时看滚动更新
kubectl rollout status deployment/php-fpm
# Waiting for deployment "php-fpm" rollout to finish: 1 out of 2 new replicas...
# deployment "php-fpm" successfully rolled out

# 查看新旧 ReplicaSet 并存（这正是滚动更新不中断的原理）
kubectl get rs -l app=php-fpm
```

## 8.10 回滚验证

```bash
kubectl rollout history deployment/php-fpm
kubectl rollout undo deployment/php-fpm
kubectl rollout status deployment/php-fpm
# 验证镜像已回退到 8.2
kubectl get deploy php-fpm -o jsonpath='{.spec.template.spec.containers[0].image}'
```

## 8.11 扩容验证

```bash
kubectl scale deployment nginx --replicas=5
kubectl get pods -l app=nginx -o wide   # 5 个 Pod 分布在多个节点
# 再通过页面/port-forward 访问，服务不中断
```

---

## 【动手练习 · 第8章】

1. 完整执行 8.2 ~ 8.8，最终在浏览器看到"MySQL 连接成功"页面。
2. 故意把 index.php 里的数据库密码改错，`apply` 后 `rollout restart`，观察页面变成"连接失败"，再改回来，理解 ConfigMap 更新与重启的关系。
3. 完成一次滚动更新 + 回滚，并用 `kubectl get rs` 观察新旧 ReplicaSet 的副本数变化。

---

# 第9章 本章项目验收

> 验收标准：从零开始，对照下面的清单逐步操作，全部通过即视为掌握本章内容。

## 9.1 验收环境假设

- 3 台 Ubuntu 22.04 虚拟机，2C2G 以上，网络互通。
- 已按第 2 章完成系统初始化、containerd 安装、kubeadm 三件套安装。

## 9.2 完整验收步骤（从零到完成）

### 第 1 步：搭建集群（约 15 分钟）

```bash
# master 上：
kubeadm init \
  --image-repository registry.aliyuncs.com/google_containers \
  --kubernetes-version v1.28.2 \
  --pod-network-cidr=192.168.0.0/16 \
  --apiserver-advertise-address=<你的master IP>

mkdir -p $HOME/.kube && cp -i /etc/kubernetes/admin.conf $HOME/.kube/config \
  && chown $(id -u):$(id -g) $HOME/.kube/config

kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml

# node1、node2 上（用 init 输出的 join 命令）：
kubeadm join <master IP>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

✅ **检查点1**：`kubectl get nodes` 显示 3 台机器全部 `Ready`。

### 第 2 步：验证核心资源（约 20 分钟）

```bash
kubectl create namespace lnmp

# 依次 apply 第 4 章的示例，至少跑通：
# Pod、Deployment（3 副本）、Service(NodePort)、ConfigMap、Secret、PV/PVC
kubectl apply -f <各 yaml 文件>
```

✅ **检查点2**：`kubectl get pods,deploy,svc,pv,pvc -n lnmp` 各资源状态正常；NodePort 能从外部访问。

### 第 3 步：部署 LNMP 应用（约 20 分钟）

按第 8 章顺序 apply `01-secret.yaml` → `02-configmap.yaml` → `03-mysql.yaml` → `04-phpfpm.yaml` → `05-nginx.yaml`。

```bash
kubectl apply -f 01-secret.yaml
kubectl apply -f 02-configmap.yaml
kubectl apply -f 03-mysql.yaml
kubectl apply -f 04-phpfpm.yaml
kubectl apply -f 05-nginx.yaml

kubectl get pods -n lnmp -w   # 等全部 Running
```

✅ **检查点3**：浏览器访问 `http://<节点IP>:30080/index.php` 显示"MySQL 连接成功"。

### 第 4 步：验证滚动更新与回滚（约 10 分钟）

```bash
kubectl set image deployment/php-fpm php-fpm=php:8.3-fpm
kubectl rollout status deployment/php-fpm
kubectl rollout undo deployment/php-fpm
kubectl rollout status deployment/php-fpm
```

✅ **检查点4**：升级后服务不中断，回滚后镜像版本回到 8.2。

### 第 5 步：验证多副本 + 高可用（约 10 分钟）

```bash
kubectl scale deployment nginx --replicas=5
kubectl get pods -l app=nginx -o wide     # 5 个 Pod 分布多节点

# 故意删掉一个 Pod，观察自动重建（自愈）
kubectl delete pod <任意 nginx pod>
kubectl get pods -l app=nginx             # 数量很快恢复 5
```

✅ **检查点5**：删除 Pod 后副本数自动恢复，外部访问不中断。

## 9.3 验收标准清单

| 序号 | 验收项 | 通过标准 |
|---|---|---|
| 1 | 集群搭建 | 1 主 2 从，`kubectl get nodes` 全部 Ready |
| 2 | 网络插件 | Calico Pod 全部 Running |
| 3 | 核心资源 | Pod/Deployment/Service/ConfigMap/Secret/PV/PVC 均能创建成功 |
| 4 | LNMP 应用 | 页面显示 MySQL 连接成功 |
| 5 | 外部访问 | NodePort 或 Ingress 能从集群外访问 |
| 6 | 滚动更新 | `set image` 后 `rollout status` 成功，服务不断 |
| 7 | 版本回滚 | `rollout undo` 后镜像回到旧版本 |
| 8 | 多副本高可用 | 副本扩到 5，删 Pod 自动恢复 |

全部通过，恭喜你，已经完成 Kubernetes 从入门到实战项目的完整闭环！🎉

---

## 附录：本教程用到的镜像速查

| 用途 | 镜像 | 国内替代 |
|---|---|---|
| K8s 组件 | `registry.k8s.io/*` | `registry.aliyuncs.com/google_containers/*` |
| Nginx | `nginx:1.25` | `docker.m.daocloud.io/library/nginx:1.25` |
| PHP-FPM | `php:8.2-fpm` | `docker.m.daocloud.io/library/php:8.2-fpm` |
| MySQL | `mysql:8.0` | `docker.m.daocloud.io/library/mysql:8.0` |
| Busybox（调试） | `busybox:1.36` | `docker.m.daocloud.io/library/busybox:1.36` |
| Calico | `docker.io/calico/*` | `registry.aliyuncs.com/...` 或离线导入 |

> 提示：本教程假设你的节点能通过某种方式（镜像加速器/代理）拉到这些镜像；若个别拉不动，参照第 6 章"手段3"手工拉取后重新打 tag。
