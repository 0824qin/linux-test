# Linux 运维工程师 · 零基础到入行完整学习路线（2026 版）

> 适合人群：刚毕业、零基础、想转行做 Linux 运维。目标：6–9 个月从小白到能面试入行，再逐步进阶。

---

## 📚 完整详细教程目录（按学习顺序，每份都是可照敲跑通的实操文档）

| 顺序 | 环节 | 详细教程文件 |
|---|---|---|
| 1 | 服务器硬件 | 服务器硬件详解与拆装实操.md |
| 2 | 阶段 1：Linux 系统基础 | 01-Linux系统基础详细教程.md |
| 3 | 阶段 2：Shell 脚本 | 02-Shell脚本详细教程.md |
| 4 | 阶段 3：网络基础 | 03-网络基础详细教程.md |
| 5 | 阶段 4：核心服务（Nginx/MySQL/Redis） | 04-核心服务详细教程.md |
| 6 | 阶段 5：Ansible 自动化 | 05-Ansible自动化详细教程.md |
| 7 | 阶段 6：Docker | 06-Docker详细教程.md |
| 8 | 阶段 7：Kubernetes | 07-Kubernetes详细教程.md |
| 9 | 阶段 8：监控与日志 | 08-监控与日志详细教程.md |
| 10 | 阶段 9：CI/CD | 09-CICD详细教程.md |
| 11 | 阶段 10：云平台 | 10-云平台详细教程.md |
| 12 | GitHub（阶段 9 配套） | GitHub使用完整教程.md |

---

## 第 0 章 先建立正确认知

### 0.1 运维到底是干什么的？
运维（Operations）的核心任务就一句话：**保证系统稳定、高效、安全地运行，并尽可能自动化、可重复地交付和维护**。

日常工作包括：
- 部署和发布：把代码/应用部署到服务器并上线
- 监控告警：7×24 盯系统是否健康，出问题第一时间响应
- 故障处理：定位问题、恢复服务、写复盘
- 资源管理：服务器、网络、存储、云资源的申请与维护
- 自动化：用脚本/工具把重复劳动自动化
- 安全与备份：权限、防火墙、数据备份恢复

### 0.2 岗位细分
- 系统运维：管服务器、操作系统、基础服务
- 应用运维：管业务应用发布、故障、性能
- 网络运维：管网络设备、链路、带宽
- 数据库运维 DBA：管 MySQL/Redis 等
- 云计算运维：管云上资源
- DevOps / SRE：开发+运维融合，重自动化、可观测性（进阶方向）

### 0.3 什么样的人适合
不需要数学天赋，不需要科班，需要：**耐心、动手、遇到报错不慌、愿意查文档和 Google**。运维是"越老越吃香"但"必须持续学习"的岗位。

### 0.4 心态建设（最重要）
1. 不要"收藏即学会"，必须**每步动手敲一遍**。
2. 学会看报错、看日志、查官方文档，而不是死记命令。
3. 每天固定时间学，别三天打鱼。
4. 遇到问题先自己排查 30 分钟再问人，把问题描述清楚。

---

## 第 1 章 学习路线总览（时间轴）

| 阶段 | 主题 | 建议时长 | 目标 |
|---|---|---|---|
| 0 | 环境准备 | 第 1 周 | 装好虚拟机，跑起第一台 Linux |
| 0.5 | 服务器硬件 | 第 1–2 周（穿插） | 认识硬件、会拆装、懂 RAID、会用带外管理 |
| 1 | Linux 系统基础 | 第 2–6 周 | 熟练使用 Linux，会管文件/权限/服务 |
| 2 | Shell 脚本 | 第 7–9 周 | 能写自动化脚本 |
| 3 | 网络基础 | 第 10–12 周 | 理解 TCP/IP、DNS、HTTP，会排查 |
| 4 | 核心服务 | 第 13–18 周 | 会搭 Nginx/MySQL/Redis |
| 5 | 自动化 Ansible | 第 19–22 周 | 批量管理几百台机器 |
| 6 | Docker | 第 23–25 周 | 容器化 |
| 7 | Kubernetes | 第 26–32 周 | 容器编排 |
| 8 | 监控与日志 | 第 33–36 周 | Prometheus/Grafana/ELK |
| 9 | CI/CD | 第 37–40 周 | 自动化发布流水线 |
| 10 | 云平台 | 穿插进行 | 阿里云/腾讯云实战 |
| 11 | 进阶 | 入行后 | Python/Go、SRE |

---

## 第 2 章 阶段详解

### 阶段 0：环境准备（第 1 周）

**发行版选择（重要，别装错）**

CentOS 7（2024 年 6 月停止维护）和 CentOS 8（已停更）都不要再学了。企业现在主流：
- **Rocky Linux 9 / AlmaLinux 9**：RHEL 完全兼容的免费替代，国内主流，**首选**
- Ubuntu 22.04 / 24.04：互联网公司常用
- 国内政企还会用：openEuler、Anolis OS（龙蜥）、麒麟、统信

建议：**主学 Rocky Linux 9（或 AlmaLinux 9）**，因为国内企业最多；Ubuntu 了解即可（命令略有差异）。

**安装环境**
- 电脑：8G 内存以上（16G 更舒服），Windows/Mac 都行
- 虚拟机软件：VMware Workstation Pro（免费了）或 VirtualBox
- 装 2 台虚拟机：一台 Rocky 9 主用，一台做测试，后面可以加更多做集群

**动手清单**
1. 装 VMware/VirtualBox
2. 下载 Rocky 9 镜像，装好系统
3. 学会：关机/重启、快照、克隆
4. 用 Xshell / MobaXterm / Windows Terminal SSH 连上虚拟机

---

### 阶段 0.5：服务器硬件（第 1–2 周，可与阶段 1 穿插学）

运维必须懂硬件，核心是：**会选型、会装机、会换件、会排障、会用带外管理**。完整实操教程见《服务器硬件详解与拆装实操.md》，这里给骨架：

1. **认识服务器**：机架式(1U/2U)/塔式/刀片；主流品牌 DELL PowerEdge、HPE ProLiant、联想、浪潮、华为，会读型号（如 R750）。
2. **核心硬件参数**：CPU（Intel Xeon/AMD EPYC、核数/主频/TDP/插槽）、内存（**必须 ECC+REG**、多通道插法）、硬盘（SATA/SAS/NVMe、热插拔托架）。
3. **RAID（重点）**：0/1/5/6/10 的容量、容错、适用场景，**生产首选 RAID 10**；会配硬件 RAID 卡，会用 `mdadm` 做软 RAID（模拟坏盘重建）。
4. **带外管理（运维分水岭）**：IPMI / iDRAC / iLO，配管理 IP，远程开关机、挂 ISO 装系统、看硬件健康。
5. **拆装实操**：上架、换内存/硬盘/RAID 卡/电源、装 CPU（防呆+硅脂）、装光模块。
6. **查硬件命令**：`lscpu` `lsmem` `lsblk` `lspci` `dmidecode` `smartctl` `ipmitool`。

---

### 阶段 1：Linux 系统基础（第 2–6 周）⭐ 最重要

#### 1.1 目录结构（FHS）
- `/` 根，`/etc` 配置，`/var` 日志和数据，`/usr` 程序，`/home` 用户，`/tmp` 临时，`/boot` 启动，`/opt` 第三方软件，`/proc`、`/sys` 内核信息（虚拟文件系统）

#### 1.2 必会命令（每天练）
```
ls, cd, pwd, cp, mv, rm, mkdir, touch
cat, more, less, head, tail, tail -f
find, which, whereis, locate
grep, wc, sort, uniq, cut, tr
tar, gzip, zip
ps, top, htop, kill, killall
df, du, free, lsblk, mount
chmod, chown, chgrp, umask
useradd, usermod, passwd, groupadd
systemctl, service, journalctl
crontab, at
ip, ifconfig, ping, ss, netstat, nc
ssh, scp, rsync, wget, curl
```

#### 1.3 权限管理（重点中的重点）
- rwx 含义，数字 4/2/1，`chmod 755`、`chmod u+x`
- 属主/属组，`chown`
- 特殊权限 SUID/SGID/Sticky Bit（了解）
- 默认权限 umask

#### 1.4 用户与组
- `/etc/passwd`、`/etc/shadow`、`/etc/group`
- 创建用户、设密码、sudo 配置（visudo）

#### 1.5 vim 编辑器
- 三种模式：普通/插入/命令
- 常用：`i`、`Esc`、`:wq`、`:q!`、`dd`、`yy`、`p`、`/搜索`、`:%s/旧/新/g`、`u`

#### 1.6 软件包管理
- Rocky/RHEL 系：`dnf`（yum 的继任者）
  ```
  dnf search / dnf install / dnf remove / dnf update
  dnf repolist
  ```
- 配置国内 yum 源（阿里云、清华镜像）
- Ubuntu 系：`apt`
- 源码编译安装（了解：configure → make → make install）

#### 1.7 systemd 服务管理
- `systemctl start/stop/restart/status/enable/disable`
- 写一个自己的 systemd unit 文件（`/etc/systemd/system/xxx.service`）

#### 1.8 磁盘与文件系统
- `lsblk`、`fdisk`、`parted`、`mkfs`
- 挂载 `/etc/fstab`、`mount`
- **LVM**：pv/vg/lv，扩容（企业常用，必学）
- 软硬链接 inode

#### 1.9 定时任务
- `crontab -e`，格式：分 时 日 月 周
- 案例：每天凌晨 2 点备份、清理日志

#### 1.10 日志
- `/var/log/messages`、`/var/log/secure`、`/var/log/nginx/*`
- `journalctl -xe`、`tail -f`

#### 阶段 1 项目验收
搭一台"标准化服务器"：创建普通用户+sudo、配置静态 IP、配好国内 yum 源、开防火墙只放行 22/80/443、装 nginx 能访问、写 crontab 每小时清一次日志。

---

### 阶段 2：Shell 脚本（第 7–9 周）

#### 三剑客（必须熟练）
- **grep**：`-v -i -n -r -E`，正则
- **sed**：`sed -n '2p'`、`sed 's/old/new/g'`、`sed -i`
- **awk**：`awk '{print $1}'`、`awk -F:`、`awk '$3>100 {print}'`

#### 脚本基础
- 变量、`$1` `$?` `$#` `$0`
- 判断：`if / elif / else`，`test`、`[ ]`、`[[ ]]`
- 循环：`for`、`while`、`until`
- 函数
- 读取输入 read，管道，重定向 `> >> < 2>&1`

#### 必写脚本案例（都要亲手写）
1. 批量创建用户
2. 每天备份 `/var/www` 到指定目录，保留 7 天
3. 监控 CPU/内存/磁盘超过阈值发告警（先打印，后接告警）
4. Nginx 日志按 IP 统计访问量 TOP10（结合 awk/sort/uniq）
5. 一键安装 LAMP 脚本

---

### 阶段 3：网络基础（第 10–12 周）

- OSI 七层 / TCP-IP 四层模型（理解即可，不用背死）
- IP 地址、子网掩码、网关、CIDR、子网划分（会算）
- 三次握手、四次挥手（面试必考）
- DNS 解析流程、`/etc/hosts`、`/etc/resolv.conf`、`dig`/`nslookup`
- HTTP/HTTPS、状态码（200/301/302/400/403/404/500/502/503/504）、GET/POST
- 端口：22(SSH)、80(HTTP)、443(HTTPS)、3306(MySQL)、6379(Redis)
- 抓包：`tcpdump`、Wireshark
- 排查命令：`ping`、`traceroute`、`mtr`、`ss`、`telnet`、`nc`、`curl -v`

#### 网络排查思路（面试/工作都考）
访问一个网站很慢/不通，怎么排查？
1. 本机网络通不通：`ping 网关`
2. DNS 正不正常：`dig 域名` 或 `nslookup`
3. 到目标主机通不通：`ping IP`、`telnet IP 80`
4. 服务起没起：`ss -tlnp | grep 80`
5. 防火墙/安全组：`firewall-cmd --list-all`
6. 看日志：应用日志、`/var/log/messages`

---

### 阶段 4：核心服务（第 13–18 周）⭐ 求职关键

#### 4.1 Nginx（重中之重）
- 安装、配置文件 `nginx.conf`、虚拟主机 server 块、location 匹配
- 反向代理 `proxy_pass`
- 负载均衡 upstream（轮询、权重、ip_hash）
- 动静分离
- HTTPS 配置、证书
- 日志格式、日志切割
- 常用参数：`worker_processes`、`keepalive_timeout`、`client_max_body_size`、`gzip`

#### 4.2 MySQL
- 安装（dnf 或源码/二进制）
- 基础 SQL：增删改查、创建用户授权 `GRANT`
- 配置文件 `/etc/my.cnf`，字符集 utf8mb4
- 备份恢复：`mysqldump`、`xtrabackup`（了解）
- 慢查询、索引（了解优化）
- **主从复制**：一主一从搭建（面试高频）
- 用户权限、忘记 root 密码重置

#### 4.3 Redis
- 安装、基本数据类型、`redis-cli`
- 持久化 RDB / AOF
- 缓存穿透/击穿/雪崩（面试概念题）
- 主从、哨兵、集群（了解）

#### 4.4 其他
- 防火墙：`firewalld`（firewall-cmd）或 `iptables`，会放行/拒绝端口
- SSH：密钥登录、改端口、禁用 root 密码登录（安全加固）
- DNS：`bind`（了解）、`/etc/hosts` 实战
- NFS 文件共享：搭一个共享目录
- Samba（Windows 共享，了解）
- 时间同步 chrony/NTP

#### 阶段 4 项目验收
**LNMP 实战**：Nginx + MySQL + PHP，部署一个真实网站（WordPress 或 Discuz），配好 HTTPS、Nginx 反代、MySQL 备份脚本、防火墙。这就是你简历上的第一个项目。

---

### 阶段 5：自动化运维 Ansible（第 19–22 周）

- 为什么需要自动化：机器多了手工管不过来
- 安装：`dnf install ansible`（控制机）
- **Inventory** 主机清单 `/etc/ansible/hosts`
- **模块**：`ping`、`shell`、`command`、`copy`、`file`、`yum`、`service`、`user`、`cron`
- **Playbook**：YAML 格式，`ansible-playbook xx.yml`
- 变量、条件 when、循环 loop、模板 template
- **Roles** 角色：目录结构、拆解复杂任务
- 免密 SSH 配置

#### 项目
写 Playbook：批量给 3 台机器装 Nginx、部署网站、改配置、开防火墙。做一个"一键初始化新服务器"的 role。

---

### 阶段 6：Docker（第 23–25 周）

- 为什么用容器：环境一致、轻量、快速交付
- 安装 Docker（`dnf install docker-ce` 或官方脚本）
- 核心概念：镜像 image、容器 container、仓库 registry
- 常用命令：`docker run`、`-d -p -v -e --name`、`docker ps`、`docker logs`、`docker exec -it`、`docker images`、`docker pull`、`docker rm`、`docker rmi`
- 镜像加速（国内配置阿里云镜像加速器）
- **Dockerfile**：FROM、RUN、COPY、WORKDIR、EXPOSE、CMD、ENTRYPOINT
- 数据卷 volume、网络（bridge/host）
- **docker-compose**：一键编排多容器
- 端口映射、环境变量

#### 项目
1. 用 Docker 跑 nginx/mysql/redis
2. 写 Dockerfile 打包一个自己的应用
3. 用 docker-compose 编排 LNMP 全套
4. 把自己写的镜像推到 Docker Hub 或阿里云容器镜像服务

---

### 阶段 7：Kubernetes（第 26–32 周）⭐ 加分项/高薪项

（入行不强制，但会了薪资高一大截，建议学）

- 概念：集群、节点、Pod、命名空间
- 核心资源：
  - Pod（最小单元，多容器）
  - Deployment（无状态应用、副本、滚动更新）
  - StatefulSet（有状态）
  - DaemonSet
  - Service（ClusterIP/NodePort/LoadBalancer）、Ingress
  - ConfigMap、Secret
  - Volume、PV、PVC
- 命令：`kubectl get/create/describe/logs/exec/apply/delete`
- 集群搭建：`kubeadm` 搭一个 1 主 2 从（虚拟机里练）
- YAML 清单写法
- **Helm** 包管理
- 网络插件 calico/flannel
- 国内镜像源配置

#### 项目
用 kubeadm 搭集群，把之前 Docker 里的 LNMP 应用迁移到 K8s，用 Deployment+Service+Ingress 暴露。

---

### 阶段 8：监控与日志（第 33–36 周）

#### 监控：Prometheus + Grafana（主流，必学）
- Prometheus 架构：拉取模型、exporter
- node_exporter 采集主机指标（CPU/内存/磁盘/网络）
- Grafana 可视化、导仪表盘（ID 1860 等）
- Alertmanager 告警（邮件/钉钉/企业微信）
- 监控 MySQL/Redis 的 exporter
- （Zabbix 老牌，国内仍用，可了解）

#### 日志：ELK / Loki
- Elasticsearch + Logstash + Kibana（ELK）或轻量的 Loki + Grafana
- 采集 Nginx/系统日志，做搜索和可视化

#### 项目
监控自己的几台虚拟机，配置磁盘>80% 发钉钉告警；把 Nginx 日志接入日志系统。

---

### 阶段 9：CI/CD（第 37–40 周）

- **Git**：`clone/add/commit/push/pull/branch/merge`，会解决冲突
- **GitHub（必会，完整教程见《GitHub使用完整教程.md》）**：注册账号、配 SSH 密钥、建仓库、写 README、日常 `add→commit→pull→push`、Fork+PR、GitHub Pages 搭博客、GitHub Actions（了解）
- 代码托管：GitHub / Gitee / 自建 GitLab
- **GitLab**：搭建、创建仓库、MR
- **Jenkins 或 GitLab CI**：流水线 pipeline
- 完整链路：开发 push 代码 → 自动构建 → 自动测试 → 打成镜像 → 推到仓库 → 部署到 K8s/服务器

#### 项目
搭一个完整 CI/CD：代码提交到 GitLab → Jenkins 拉代码 → 构建 Docker 镜像 → 推镜像 → 自动部署。这是简历上最亮的项目。

---

### 阶段 10：云平台（穿插学习）

企业大部分已经上云，必须会用至少一家：
- 阿里云（国内最大）或腾讯云
- 核心产品：ECS（云服务器）、VPC（专有网络）、安全组、SLB（负载均衡）、OSS（对象存储）、CDN、RDS（云数据库）、云监控、域名备案
- 在云上把之前学的 LNMP/K8s 项目重新部署一遍
- 考个证书：阿里云 ACP（含金量还行，入门可加分）

---

### 阶段 11：进阶方向（入行后）

- **Python**：运维脚本、自动化、API、写运维平台（必学一门编程语言）
- **Go**：写高性能工具、K8s 生态
- **IaC**：Terraform（基础设施即代码）
- **SRE**：SLO/SLI、混沌工程、容量规划
- 深入：性能调优、大规模集群、数据库 DBA、安全

---

## 第 3 章 学习时间表（全职 6–8 个月参考）

| 月份 | 内容 | 里程碑 |
|---|---|---|
| 第 1 月 | 阶段 0+1 Linux 基础 | 熟练用 Linux，搭好标准服务器 |
| 第 2 月 | 阶段 2+3 Shell+网络 | 能写脚本、会排查网络 |
| 第 3 月 | 阶段 4 核心服务 | LNMP 完整项目 |
| 第 4 月 | 阶段 5+6 Ansible+Docker | 自动化+容器化 |
| 第 5–6 月 | 阶段 7 K8s | 集群+应用迁移 |
| 第 7 月 | 阶段 8+9 监控+CI/CD | 完整 DevOps 链路 |
| 第 8 月 | 项目整理+简历+面试 | 拿 offer |

如果每天只能学 2–3 小时（在职），时间翻倍到 12–18 个月。

---

## 第 4 章 学习方法论（决定成败）

1. **动手第一**：命令必须敲，报错必须自己解决一遍。
2. **做笔记**：用 Typora/语雀/Notion，记录"报错+原因+解法"，这是你面试的弹药库。
3. **搭博客**：GitHub Pages / 个人博客，把每个项目写成文章（面试加分，证明你懂）。
4. **费曼学习法**：能讲清楚才算会。
5. **一查二试三问**：先 Google/官方文档，再自己试，最后才问人（提问要贴报错、贴日志、说明你试了什么）。
6. **GitHub 记录**：所有脚本、Playbook、Dockerfile、YAML 都传 GitHub，这就是你的作品集。

---

## 第 5 章 求职与简历

### 简历怎么写（核心：项目代替学历）
- 学历一般没关系，运维看项目和技术
- 写 2–3 个真实项目：
  1. LNMP 高可用网站部署（含 HTTPS、备份、监控）
  2. 基于 Docker/K8s 的容器化部署
  3. 完整 CI/CD 流水线
- 项目写法：背景 → 你做了什么 → 解决了什么问题 → 用了什么技术
- 别写"精通"，写"熟悉/掌握"，面试会被深挖，写什么就要经得起问

### 面试高频题（提前准备）
1. Linux 查 CPU/内存/磁盘占用、查端口占用
2. 权限 755/644 什么意思，chmod/chown
3. 三次握手、四次挥手
4. Nginx 反向代理和负载均衡怎么配，location 优先级
5. 网站访问慢/502 怎么排查
6. MySQL 主从原理、备份恢复
7. Docker 镜像和容器区别、Dockerfile 常用指令
8. K8s 的 Pod/Deployment/Service 是什么
9. 磁盘满了怎么排查（du/find 大文件）
10. 常见故障：CPU 打满、内存泄漏、磁盘 100%、网络不通

### 证书（可选）
- RHCE（红帽，运维经典，认可度高）
- CKA（K8s 管理员，进阶加分）
- 阿里云 ACP（国内云计算入门）

---

## 第 6 章 学习资源推荐

**视频（B站，搜关键词）**
- 黑马程序员 Linux / 尚硅谷 Linux
- 尚硅谷 Shell、Docker、K8s
- 老男孩、马哥 Linux 运维

**书籍**
- 《鸟哥的 Linux 私房菜》（基础经典，可当字典）
- 《Linux 命令行与 Shell 脚本编程大全》
- 《跟老男孩学 Linux 运维》

**文档（最权威）**
- 官方文档：Rocky/RedHat 文档、Nginx 中文、K8s 官方、Docker 官方、Prometheus 官方
- 菜鸟教程、Linux 命令大全

**练习平台**
- 虚拟机自己搭（最推荐）
- 阿里云/腾讯云新用户低价服务器
- 牛客网刷 Linux 面试题

---

## 第 7 章 新手最容易踩的坑

1. **贪多嚼不烂**：别同时学一堆，按顺序来。
2. **只看不练**：Linux 运维是"练"出来的，不是"看"出来的。
3. **死记命令**：理解原理，命令忘了会查 `man`、`--help`。
4. **报错就慌**：报错是朋友，是学习最快的方式。
5. **纠结发行版**：先学透一个，其他大同小异。
6. **在生产环境乱试**：rm -rf 前想三遍，养成先备份、先测试的习惯（`rm -rf /` 的教训）。
7. **忽略英语**：文档/报错大多是英文，至少要能看懂报错关键词，坚持读英文文档。

---

## 第 8 章 最后的话

从零到入行，没有捷径，但有清晰的路：**Linux 基础 → Shell → 网络 → 核心服务 → 自动化 → 容器 → K8s → 监控 → CI/CD → 云**。

每天保证 3–4 小时有效学习，认真完成每个阶段的项目，6–9 个月后你就能拿着一份有真实项目的简历去面试。运维入门门槛其实不高，真正拉开差距的是**动手量和坚持**。祝你早日成为大佬 🚀
