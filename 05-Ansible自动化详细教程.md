# 阶段5：Ansible 自动化运维（零基础完整实操教程）

> 适用对象：从没接触过自动化运维的零基础小白。
> 学习目标：照着本教程敲一遍，能够独立完成"批量管理多台服务器"的完整流程，包括安装、免密登录、编写 Playbook、使用 Roles，最终交付一个能一键初始化新服务器并批量部署网站的项目。

---

## 目录

1. [为什么需要自动化 + Ansible 是什么 + 架构](#1-为什么需要自动化--ansible-是什么--架构)
2. [安装与验证（dnf / pip）](#2-安装与验证dnf--pip)
3. [配置免密 SSH（ssh-copy-id 完整步骤）](#3-配置免密-sshssh-copy-id-完整步骤)
4. [Inventory 主机清单](#4-inventory-主机清单)
5. [常用模块详解（14 个）](#5-常用模块详解14-个)
6. [临时命令 ansible 与 ad-hoc](#6-临时命令-ansible-与-ad-hoc)
7. [Playbook 基础（YAML + hosts/tasks + --check）](#7-playbook-基础yaml--hoststasks---check)
8. [变量 / 条件 when / 循环 loop / 模板 jinja2 / handlers / tags](#8-变量--条件-when--循环-loop--模板-jinja2--handlers--tags)
9. [Roles 角色](#9-roles-角色)
10. [完整项目：一键初始化新服务器 + 批量部署网站](#10-完整项目一键初始化新服务器--批量部署网站)
11. [本章项目验收（3 台机器装 Nginx + 部署网站）](#11-本章项目验收3-台机器装-nginx--部署网站)

---

## 1. 为什么需要自动化 + Ansible 是什么 + 架构

### 1.1 是什么：从"手工运维"到"自动化运维"

传统运维里，管理员要登录每一台服务器，一条条敲命令：

```bash
# 传统手工做法：要登录 10 台服务器，每台重复做同样的事
ssh root@192.168.1.10
dnf install -y nginx
systemctl enable --now nginx
systemctl start firewalld
useradd deploy
exit

ssh root@192.168.1.11   # 又重复一遍
dnf install -y nginx
...
```

当服务器只有 1 台时，手工没问题；当有 3 台、10 台、100 台时，重复劳动、漏配、配置不一致就成了灾难。

**Ansible 是什么**：Ansible 是一个"自动化运维/配置管理工具"，它让你在**一台控制机**上，用一段"剧本（Playbook）"或一条命令，同时对**成百上千台服务器**执行相同的操作。

- **是什么**：一个批量执行任务、批量下发配置的开源工具（由 Red Hat 主导，社区活跃）。
- **为什么用**：把重复、易错的手工操作变成可重复、可审计、可回放的代码；一次编写，处处执行；保证所有机器配置一致。
- **怎么用**：写 Inventory（主机清单）决定"管哪些机器"，写 Playbook/命令决定"做什么事"，然后一条命令推下去。

### 1.2 架构：控制端、被管端、无 Agent、SSH

Ansible 的架构非常简单，**没有专门的 Agent 常驻程序**，这是它与 Puppet/Chef 最大的区别。

| 组成部分 | 说明 | 类比 |
| --- | --- | --- |
| 控制端（Control Node） | 你安装 Ansible 的那台机器，负责发指令 | 指挥官 |
| 被管端（Managed Node） | 被你管理的目标服务器，**不需要安装 Ansible** | 士兵 |
| Inventory | 主机清单文件，记录被管端地址和分组 | 花名册 |
| Module | 模块，Ansible 的最小执行单元（如 ping、copy） | 工具箱里的工具 |
| Playbook | 剧本，用 YAML 写的一组任务 | 作战计划 |

```text
        ┌─────────────────────────┐
        │   控制端（Control Node） │  安装 Ansible
        │   192.168.1.100          │
        └───────────┬─────────────┘
                    │  SSH（22端口，免密或密码）
        ┌───────────┼───────────┬─────────────┐
        ▼           ▼           ▼             ▼
   ┌────────┐ ┌────────┐ ┌────────┐    ┌────────┐
   │ web1   │ │ web2   │ │ web3   │    │ db1    │
   │ 被管端 │ │ 被管端 │ │ 被管端 │    │ 被管端 │
   └────────┘ └────────┘ └────────┘    └────────┘
   （不需要装 Ansible，只要有 Python 和 SSH 即可）
```

**关键原理说明**：

1. **无 Agent**：被管端不需要安装任何 Ansible 客户端。Ansible 靠 SSH 连接过去，临时把一段 Python 代码推上去执行，执行完就清理掉。这大大降低了维护成本，也避免了"Agent 版本不一致"的坑。
2. **依赖 SSH**：Ansible 默认通过 SSH 协议连接被管端。所以被管端必须开启 SSH 服务（sshd），且控制端能通过 SSH 登录。
3. **依赖 Python**：被管端需要安装 Python（RHEL/CentOS 7 默认有 python2，RHEL 8/9 默认有 python3）。Ansible 推上去的模块本质是 Python 脚本。
4. **幂等性（Idempotent）**：这是 Ansible 最重要的特性之一。同一个任务重复执行多次，结果保持不变。例如"确保用户 deploy 存在"，如果用户已经存在，Ansible 不会报错也不会重复创建，而是直接跳过（返回 `ok`）。这保证了配置的安全性和可重复性。

**推送式 vs 拉取式**：Ansible 是"推送式"——由控制端主动把指令"推"给被管端，而不是被管端定时来"拉"。这让你随时可以立即执行，不需要等被管端轮询。

### 1.3 动手练习

1. 在纸上画出 1 台控制端 + 3 台被管端的拓扑图，并标注 SSH 端口。
2. 说出 Ansible 与 Puppet/Chef 相比最大的一个区别（提示：Agent）。
3. 解释"幂等性"用大白话怎么说，并举一个例子。

---

## 2. 安装与验证（dnf / pip）

> 本教程以 **RHEL 8 / 9 / Rocky Linux / AlmaLinux / CentOS Stream** 为主（用 `dnf`）。CentOS 7 把 `dnf` 换成 `yum` 即可，命令几乎一样。

### 2.1 是什么：安装方式

Ansible 有两种主流安装方式：

| 方式 | 命令 | 适用场景 |
| --- | --- | --- |
| 系统包管理器 `dnf` | `dnf install -y ansible-core` 或 `ansible` | 生产环境，稳定、易升级 |
| Python 包管理器 `pip` | `pip install ansible` | 需要特定版本、无 root 权限时 |

> 说明：RHEL 8+ 仓库里包名叫 `ansible-core`（精简核心）或 `ansible`（含更多集合）。CentOS 7 的 EPEL 源里叫 `ansible`。

### 2.2 为什么用：选择哪种

- **生产环境推荐 dnf/yum**：由系统统一管理依赖，升级安全，符合企业规范。
- **需要最新版或特定版本推荐 pip**：版本更新快，可用 `pip install ansible==版本号` 锁定。

### 2.3 怎么用：完整安装步骤

**方法一：dnf 安装（推荐）**

```bash
# 1. 安装 EPEL 仓库（RHEL/CentOS 需要，Rocky/Alma 通常已内置或同样需要）
dnf install -y epel-release

# 2. 安装 ansible-core（或 ansible）
dnf install -y ansible-core

# 3. 验证安装成功：查看版本
ansible --version
```

**预期结果（类似）：**

```text
ansible [core 2.16.x]
  config file = /etc/ansible/ansible.cfg
  configured module search path = ['/root/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  python version = 3.9.x (main, ...)
  jinja version = 3.1.x
  libyaml = True
```

**方法二：pip 安装**

```bash
# 1. 确保已装 python3 和 pip
dnf install -y python3 python3-pip

# 2. 用 pip 安装（建议升级 pip 先）
python3 -m pip install --upgrade pip
python3 -m pip install ansible

# 3. 验证
ansible --version
```

### 2.4 安装后的目录与文件

安装完成后，会有这些重要路径：

| 路径 | 作用 |
| --- | --- |
| `/etc/ansible/ansible.cfg` | 主配置文件（全局配置） |
| `/etc/ansible/hosts` | 默认主机清单文件 |
| `/etc/ansible/roles/` | 默认 Roles 目录 |

**为什么需要配置文件**：`ansible.cfg` 控制 Ansible 的默认行为，比如默认使用哪个 Inventory、是否开启 SSH 免密、重试次数等。新手可以先不修改，全部用默认值即可跑通。

### 2.5 动手练习

1. 用 `dnf` 安装 `ansible-core`，并执行 `ansible --version` 记录版本号。
2. 尝试用 `pip` 卸载再重装（可选，理解两种方式差异）。
3. 查看 `/etc/ansible/` 目录下有哪些文件，说出 `ansible.cfg` 和 `hosts` 各自的用途。

---

## 3. 配置免密 SSH（ssh-copy-id 完整步骤）

### 3.1 是什么：SSH 密钥认证

Ansible 通过 SSH 连接被管端。如果每台机器都要输入密码，批量执行就没法自动化了。免密 SSH 就是让控制端用"密钥对"登录被管端，不用输密码。

**原理（公钥/私钥）**：

- 控制端生成一对密钥：**私钥**（自己留好，绝不外传）+ **公钥**（可以发给任何人）。
- 把**公钥**放到被管端的 `~/.ssh/authorized_keys` 文件里。
- 之后控制端用私钥"加密"一段信息，被管端用公钥"验证"，验证通过即免密登录。

### 3.2 为什么用

- 自动化必须免密（或使用 vault 管理的密码），否则 `ansible` 批量执行会被密码交互卡住。
- 安全：密钥比密码更难暴力破解，且可随时吊销。

### 3.3 怎么用：完整步骤（照敲即可）

假设环境：

| 角色 | IP | 用户名 |
| --- | --- | --- |
| 控制端 | 192.168.1.100 | root |
| 被管端 web1 | 192.168.1.101 | root |
| 被管端 web2 | 192.168.1.102 | root |

**第一步：在控制端生成密钥对**

```bash
# 一路回车即可（默认生成到 ~/.ssh/id_rsa 和 id_rsa.pub）
ssh-keygen -t rsa -b 4096 -C "ansible-control"
```

参数解释：

| 参数 | 说明 |
| --- | --- |
| `-t rsa` | 指定密钥类型为 RSA |
| `-b 4096` | 密钥长度 4096 位（更安全） |
| `-C "注释"` | 加一段注释，方便识别这把钥匙是谁的 |

**预期结果：**

```text
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa):   # 直接回车
Enter passphrase (empty for no passphrase):                # 直接回车（不设密码）
Enter same passphrase again:                               # 直接回车
Your identification has been saved in /root/.ssh/id_rsa
Your public key has been saved in /root/.ssh/id_rsa.pub
```

> 注意：生产环境建议给私钥设 passphrase，并用 `ssh-agent` 管理；本教程为了零基础跑通，先不设密码。

**第二步：用 ssh-copy-id 把公钥拷贝到被管端**

```bash
# 拷贝到 web1，会提示输入一次被管端 root 密码
ssh-copy-id root@192.168.1.101

# 拷贝到 web2
ssh-copy-id root@192.168.1.102
```

`ssh-copy-id` 做的事（原理）：它读取控制端的 `~/.ssh/id_rsa.pub`，通过 SSH 登录到被管端，把公钥追加写入被管端的 `~/.ssh/authorized_keys`，并设置正确权限。等价于手工执行：

```bash
cat ~/.ssh/id_rsa.pub | ssh root@192.168.1.101 "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

**预期结果：**

```text
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/root/.ssh/id_rsa.pub"
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s)...
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed...
Number of key(s) added: 1

Now try logging into the machine, with:   "ssh 'root@192.168.1.101'"
and check to make sure that only the key(s) you wanted were added.
```

**第三步：验证免密登录成功**

```bash
# 不再提示输入密码，直接登录并执行 hostname
ssh root@192.168.1.101 "hostname"
```

**预期结果：** 直接返回被管端主机名，不提示密码，说明免密成功。

```text
web1
```

**常见报错排查：**

| 现象 | 原因 | 解决 |
| --- | --- | --- |
| 仍提示输入密码 | 公钥没写进去 / 权限错误 | 重新 `ssh-copy-id`；检查被管端 `.ssh` 权限为 700、`authorized_keys` 为 600 |
| `Permission denied (publickey)` | 被管端未开公钥认证 | 编辑被管端 `/etc/ssh/sshd_config`，确保 `PubkeyAuthentication yes`，然后 `systemctl restart sshd` |
| 提示 `Host key verification failed` | 首次连接未记录指纹 | 先手动 `ssh root@IP` 一次，回答 `yes`，或在 ansible 配置里关闭 `host_key_checking` |

### 3.4 动手练习

1. 生成一对密钥，查看 `~/.ssh/` 下生成了哪些文件，用 `ls -l` 观察权限。
2. 用 `ssh-copy-id` 把公钥拷贝到至少 1 台被管端。
3. 用 `ssh root@IP "uptime"` 验证免密，并确认无需密码。
4. 故意把 `authorized_keys` 权限改成 777，再次 ssh 观察是否失败（理解权限的重要性），再改回 600。

---

## 4. Inventory 主机清单

### 4.1 是什么：主机清单

Inventory（清单）告诉 Ansible"我要管理哪些机器"。默认文件是 `/etc/ansible/hosts`，也可以自定义文件后用 `-i` 指定。

### 4.2 为什么用

- 把机器分组，方便对不同组执行不同任务（web 组、db 组、all 组）。
- 支持给机器定义变量（端口、用户名等），实现"一套 Playbook 适配不同机器"。

### 4.3 怎么用：各种写法

**（1）最简单的写法：一行一个 IP**

```ini
# /etc/ansible/hosts
192.168.1.101
192.168.1.102
```

解释：这两个 IP 都在默认组 `ungrouped`（未分组）里，也属于内置的 `all` 组。

**（2）分组写法**

```ini
[web]
192.168.1.101
192.168.1.102
192.168.1.103

[db]
192.168.1.110
192.168.1.111
```

解释：`[web]` 和 `[db]` 是组名，组名下的行就是组成员。组名可以随便起，但建议语义化。

**（3）别名 + 变量写法**

```ini
[web]
web1 ansible_host=192.168.1.101 ansible_ssh_port=22 ansible_user=root
web2 ansible_host=192.168.1.102 ansible_ssh_port=2222 ansible_user=deploy
```

| 变量 | 说明 |
| --- | --- |
| `ansible_host` | 实际连接的 IP 或域名（别名 `web1` 只用于展示，真正连的是这个） |
| `ansible_ssh_port` | SSH 端口，默认 22，改端口时必填 |
| `ansible_user` | 以哪个用户登录，默认当前用户 |
| `ansible_password` | SSH 密码（不推荐明文写，用 vault） |
| `ansible_ssh_private_key_file` | 指定私钥文件路径 |
| `ansible_become` | 是否提权（sudo） |
| `ansible_become_user` | 提权到哪个用户，默认 root |
| `ansible_python_interpreter` | 指定被管端 python 路径 |

**（4）组变量写法**

```ini
[web]
web1 ansible_host=192.168.1.101
web2 ansible_host=192.168.1.102

[web:vars]
ansible_user=root
ansible_ssh_port=22

[db]
db1 ansible_host=192.168.1.110

[db:vars]
ansible_user=root
ansible_ssh_port=2222
```

解释：`[web:vars]` 是对 `web` 组整体生效的变量，组内所有主机继承，不用每台重复写。

**（5）范围写法（简写连续 IP）**

```ini
[web]
192.168.1.[101:110]      # 等价于 192.168.1.101 到 192.168.1.110 共 10 台
```

**（6）子组嵌套（组中组）**

```ini
[web]
web1
web2

[db]
db1

[prod:children]
web
db
```

解释：`[prod:children]` 表示 `prod` 组由 `web` 和 `db` 两个子组组成，对 `prod` 操作等于对 `web+db` 操作。

### 4.4 查看清单

```bash
# 列出所有主机
ansible all -i /etc/ansible/hosts --list-hosts

# 列出 web 组主机
ansible web --list-hosts
```

**预期结果：**

```text
  hosts (2):
    web1
    web2
```

### 4.5 动手练习

1. 编辑 `/etc/ansible/hosts`，添加至少 2 台真实机器，分到 `web` 组。
2. 给其中一台设置别名和 `ansible_ssh_port`、`ansible_user` 变量。
3. 用 `ansible all --list-hosts` 和 `ansible web --list-hosts` 验证。
4. 练习子组嵌套：定义 `prod:children` 包含 `web` 和 `db`。

---

## 5. 常用模块详解（14 个）

> 模块是 Ansible 的核心执行单元。每个模块的通用用法：
> `ansible <主机或组> -m <模块名> -a "<参数>"`
> 下面每个模块都给出"是什么 / 为什么 / 怎么用 / 参数解释 / 预期结果"。

### 5.0 先看执行结果的颜色（重要）

Ansible 执行结果有颜色，代表不同含义：

| 颜色 | 含义 |
| --- | --- |
| 绿色 `ok` | 目标状态已满足，无需改变（幂等） |
| 黄色 `changed` | 执行了变更，目标状态被修改 |
| 红色 `failed` | 执行失败 |

理解这三色，是读懂 Ansible 输出的关键。

---

### 5.1 ping 模块（连通性测试）

**是什么**：测试控制端与被管端能否正常通信（不是系统 `ping` 命令，而是 Ansible 层面的"握手"）。

**为什么用**：做任何批量操作前，先用 `ping` 确认所有主机可达，避免部分机器连不上导致误判。

**怎么用：**

```bash
ansible all -m ping
```

**预期结果：**

```text
web1 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
web2 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

解释：`"ping": "pong"` 表示连通正常；`"changed": false` 表示没有做任何修改（ping 只读不写）。

---

### 5.2 command 模块（执行命令，不支持管道/重定向）

**是什么**：在被管端执行一条简单命令，是默认模块（`-m command` 可省略）。

**为什么用**：执行简单的、不需要 shell 特性的命令。

**怎么用：**

```bash
# 以下三条等价（缺省 -m 默认就是 command）
ansible all -m command -a "hostname"
ansible all -m command -a "uptime"
ansible all -a "date"
```

**预期结果：**

```text
web1 | CHANGED | rc=0 >>
web1

web2 | CHANGED | rc=0 >>
web2
```

参数/字段解释：

| 字段 | 说明 |
| --- | --- |
| `rc=0` | 命令退出码（return code），0 表示成功 |
| `CHANGED` | command 每次都会执行，所以标 CHANGED（它不做幂等判断） |

**重要坑**：`command` 模块**不支持** `|`、`>`、`<`、`&&`、`;`、`$变量` 等 shell 特性，因为它不经过 shell 解析。

```bash
# 错误示例：这条会失败
ansible all -m command -a "echo hello | wc -c"
```

报错提示：`FAILED! ... 无法解析管道符`。要用管道必须改用 `shell` 模块。

---

### 5.3 shell 模块（执行命令，支持管道/重定向）

**是什么**：通过被管端的 shell（默认 `/bin/sh`）执行命令，支持管道、重定向、变量等。

**为什么用**：命令里包含 `|`、`>`、`&&`、通配符 `*` 等 shell 特性时必须用它。

**怎么用：**

```bash
# 支持管道
ansible all -m shell -a "echo hello | wc -c"

# 支持重定向
ansible all -m shell -a "date >> /tmp/time.log"

# 支持变量和命令替换
ansible all -m shell -a "echo 今天是 $(date +%F)"
```

**预期结果：**

```text
web1 | CHANGED | rc=0 >>
6
```

**安全提醒**：`shell` 模块如果拼接了外部变量，存在命令注入风险。能用 `command` 就别用 `shell`；能用专用模块（如 `copy`、`file`）就别用 `shell` 去拼。

---

### 5.4 copy 模块（拷贝文件到被管端）

**是什么**：把控制端本地的文件拷贝到被管端指定位置。

**为什么用**：批量分发配置文件、脚本、网站源码等，比 `scp` 更可控且幂等。

**怎么用：**

```bash
# 先在控制端准备一个文件
echo "hello ansible" > /tmp/hello.txt

# 拷贝到被管端 /etc/hello.txt，指定属主属组和权限
ansible all -m copy -a "src=/tmp/hello.txt dest=/etc/hello.txt owner=root group=root mode=0644"
```

参数解释：

| 参数 | 说明 |
| --- | --- |
| `src` | 控制端源文件路径 |
| `dest` | 被管端目标路径（文件或目录） |
| `owner` / `group` | 目标文件属主/属组 |
| `mode` | 权限（0644 等） |
| `backup=yes` | 覆盖前备份原文件 |

**预期结果：**

```text
web1 | CHANGED => {
    "changed": true,
    "checksum": "a0b65939670bc2c010f4d5d6a0b3e4e4590fb92b",
    "dest": "/etc/hello.txt",
    "size": 14
}
```

**幂等验证**：再执行一次同样的命令，会返回 `"changed": false`（绿色 ok），因为文件内容没变，Ansible 用 checksum 判断无需重复拷贝。

**content 用法（不用 src，直接写内容）：**

```bash
ansible all -m copy -a "content='这是内容\n第二行' dest=/tmp/c.txt"
```

---

### 5.5 file 模块（管理文件/目录/软链接属性）

**是什么**：创建/删除文件或目录、设置权限、创建软链接，但不负责传输内容（内容用 copy）。

**为什么用**：批量创建目录、授权、建软链，是文件系统管理的基础模块。

**怎么用：**

```bash
# 1. 创建目录
ansible all -m file -a "path=/data/www state=directory owner=root group=root mode=0755"

# 2. 创建空文件
ansible all -m file -a "path=/tmp/empty.txt state=touch mode=0644"

# 3. 创建软链接（nginx 配置常用）
ansible all -m file -a "src=/etc/nginx/sites-available/app dest=/etc/nginx/sites-enabled/app state=link"

# 4. 删除文件/目录
ansible all -m file -a "path=/tmp/empty.txt state=absent"
```

参数解释：

| 参数 | 说明 |
| --- | --- |
| `path` | 目标路径 |
| `state` | `directory`(目录) / `touch`(空文件) / `link`(软链) / `absent`(删除) / `file`(普通文件) |
| `src` | 当 `state=link` 时，软链接指向的源 |
| `owner` / `group` / `mode` | 属主/属组/权限 |

**预期结果（创建目录）：**

```text
web1 | CHANGED => {
    "changed": true,
    "mode": "0755",
    "path": "/data/www",
    "state": "directory"
}
```

---

### 5.6 yum / dnf 模块（软件包管理）

**是什么**：在被管端安装/卸载/更新软件包。

- RHEL 8/9、Rocky、Alma、CentOS Stream 用 `dnf`
- CentOS 7 用 `yum`
- 两者用法几乎一致，`dnf` 是 `yum` 的升级替代。

**为什么用**：批量装包，且幂等（已安装则跳过），比手工 `ssh 上去 yum install` 强太多。

**怎么用：**

```bash
# 安装（present = 确保存在）
ansible all -m dnf -a "name=nginx state=present"

# 一次装多个包
ansible all -m dnf -a "name=nginx,vim,git,wget state=present"

# 卸载（absent = 确保不存在）
ansible all -m dnf -a "name=nginx state=absent"

# 更新到最新（latest）
ansible all -m dnf -a "name=nginx state=latest"
```

参数解释：

| 参数 | 说明 |
| --- | --- |
| `name` | 包名，多个用逗号分隔 |
| `state` | `present`(安装) / `absent`(卸载) / `latest`(最新版) |
| `update_cache=yes` | 安装前先 `dnf makecache` 刷新元数据（首次装建议加） |

**预期结果（安装 nginx）：**

```text
web1 | CHANGED => {
    "changed": true,
    "results": ["已安装 nginx-1.20.x ..."]
}
```

**幂等验证**：再次执行，返回绿色 `ok`（已安装，无需变更）。

---

### 5.7 service / systemd 模块（服务管理）

**是什么**：启动/停止/重启服务，并设置开机自启。

- `service` 模块：兼容老系统，底层仍会调用 systemd。
- `systemd` 模块：直接管理 systemd 服务，RHEL 7+ 推荐用 `systemd`。

**为什么用**：装完包后要启动服务并设开机自启，这是运维高频操作。

**怎么用：**

```bash
# 启动 nginx 并设置开机自启（enabled=true 表示自启）
ansible all -m systemd -a "name=nginx state=started enabled=true"

# 重启服务
ansible all -m systemd -a "name=nginx state=restarted"

# 重新加载配置（不改进程，只重读配置，nginx -s reload 场景）
ansible all -m systemd -a "name=nginx state=reloaded"

# 停止并关闭自启
ansible all -m systemd -a "name=nginx state=stopped enabled=false"
```

参数解释：

| 参数 | 说明 |
| --- | --- |
| `name` | 服务名（不带 .service 后缀） |
| `state` | `started`/`stopped`/`restarted`/`reloaded` |
| `enabled` | `true`=开机自启，`false`=取消自启 |

**预期结果：**

```text
web1 | CHANGED => {
    "changed": true,
    "enabled": true,
    "name": "nginx",
    "state": "started"
}
```

**验证服务状态（用 command 执行 systemctl）：**

```bash
ansible all -m shell -a "systemctl status nginx --no-pager | head -5"
```

---

### 5.8 user 模块（用户管理）

**是什么**：创建/删除用户、设置属组、家目录、shell 等。

**为什么用**：批量创建统一账号（如 deploy、www），避免每台机器手工人肉建。

**怎么用：**

```bash
# 创建用户 deploy，主组为 deploy，附加组 wheel，家目录 /home/deploy
ansible all -m user -a "name=deploy state=present shell=/bin/bash groups=wheel append=yes create_home=yes"

# 删除用户（并删除家目录）
ansible all -m user -a "name=deploy state=absent remove=yes"
```

参数解释：

| 参数 | 说明 |
| --- | --- |
| `name` | 用户名 |
| `state` | `present`(创建) / `absent`(删除) |
| `shell` | 登录 shell |
| `groups` | 附加组（多个用逗号） |
| `append=yes` | 追加到附加组（不加会覆盖原附加组） |
| `create_home=yes` | 创建家目录 |
| `remove=yes` | 删除用户时连带删家目录 |

**预期结果：**

```text
web1 | CHANGED => {
    "changed": true,
    "name": "deploy",
    "state": "present",
    "uid": 1001
}
```

---

### 5.9 group 模块（组管理）

**是什么**：创建/删除用户组。

**为什么用**：配合 user 模块统一管理组，保持权限体系一致。

**怎么用：**

```bash
# 创建组 devops
ansible all -m group -a "name=devops state=present gid=2000"

# 删除组
ansible all -m group -a "name=devops state=absent"
```

参数解释：

| 参数 | 说明 |
| --- | --- |
| `name` | 组名 |
| `state` | `present` / `absent` |
| `gid` | 指定组 ID（可选） |

---

### 5.10 cron 模块（定时任务）

**是什么**：管理被管端的 crontab 定时任务。

**为什么用**：批量下发定时备份、日志清理、监控脚本等，无需逐台 `crontab -e`。

**怎么用：**

```bash
# 每天凌晨 3 点执行日志清理脚本
ansible all -m cron -a "name='clean log' minute=0 hour=3 job='/usr/local/bin/clean_log.sh' user=root"

# 删除这条定时任务（按 name 匹配）
ansible all -m cron -a "name='clean log' state=absent"
```

参数解释：

| 参数 | 说明 |
| --- | --- |
| `name` | 任务标识（用于幂等和删除，必填） |
| `minute` / `hour` / `day` / `month` / `weekday` | 对应 cron 时间字段，默认 `*` |
| `job` | 要执行的命令 |
| `user` | 哪个用户的 crontab |
| `state` | `present` / `absent` |

**验证（查看被管端 crontab）：**

```bash
ansible all -m shell -a "crontab -l"
```

---

### 5.11 get_url 模块（从 URL 下载文件）

**是什么**：让被管端从指定 URL 下载文件到本地。

**为什么用**：批量从内网/外网下载安装包、脚本，省去手动 scp。

**怎么用：**

```bash
# 下载文件到 /tmp/
ansible all -m get_url -a "url=https://example.com/file.tar.gz dest=/tmp/file.tar.gz mode=0644"

# 下载并校验 sha256
ansible all -m get_url -a "url=https://example.com/file.tar.gz dest=/tmp/file.tar.gz checksum=sha256:xxxxx"
```

参数解释：

| 参数 | 说明 |
| --- | --- |
| `url` | 下载地址 |
| `dest` | 保存路径 |
| `checksum` | 校验和，格式 `算法:值` |
| `mode` / `owner` / `group` | 权限属性 |

---

### 5.12 template 模块（渲染模板）

**是什么**：把控制端的"模板文件"用 Jinja2 引擎渲染（填充变量）后，生成最终配置，下发到被管端。

**为什么用**：不同机器需要不同配置值（如不同 IP、不同端口）时，用同一个模板 + 不同变量，生成各自的配置。这是 Ansible 最强大的能力之一。

**怎么用：**

先在控制端创建模板文件（后缀 `.j2`），内容里用 `{{ 变量 }}` 占位：

```bash
# 控制端创建模板
cat > /tmp/nginx.conf.j2 <<'EOF'
server {
    listen {{ http_port }};
    server_name {{ server_name }};
    root /usr/share/nginx/html;
}
EOF
```

然后在 Inventory 或 Playbook 里定义变量 `http_port`、`server_name`，再渲染：

```bash
# 传变量方式渲染（-e 定义变量）
ansible all -m template -a "src=/tmp/nginx.conf.j2 dest=/etc/nginx/conf.d/app.conf" \
  -e "http_port=8080 server_name=web1.example.com"
```

参数解释：

| 参数 | 说明 |
| --- | --- |
| `src` | 控制端模板文件（.j2） |
| `dest` | 渲染后的目标文件 |
| `owner` / `group` / `mode` | 目标文件权限 |

**原理说明**：`{{ http_port }}` 会被变量值 `8080` 替换，生成 `listen 8080;`。给不同机器传不同变量，就能一份模板生成不同配置。详细 Jinja2 语法见第 8 章。

---

### 5.13 lineinfile 模块（改文件中的某一行）

**是什么**：在被管端文件中，按正则匹配"找一行"，如果找到就替换，找不到就在末尾追加。

**为什么用**：批量修改配置文件里的某一行（如改监听端口、改内核参数），无需整文件覆盖，避免误改其他行。

**怎么用：**

```bash
# 确保 /etc/ssh/sshd_config 中 Port 这一行是 2222（找到则替换，找不到则追加）
ansible all -m lineinfile -a "path=/etc/ssh/sshd_config regexp='^Port ' line='Port 2222'"

# 确保存在某行（不覆盖已有内容，仅当不存在时追加）
ansible all -m lineinfile -a "path=/etc/hosts line='192.168.1.100 control-node'"
```

参数解释：

| 参数 | 说明 |
| --- | --- |
| `path` | 目标文件 |
| `regexp` | 正则，匹配要替换的那一行 |
| `line` | 要写入的内容 |
| `state` | `present`(默认，确保存在) / `absent`(删除匹配行) |
| `backrefs` | 配合正则反向引用 |

**对比 copy 模块**：

| | copy | lineinfile |
| --- | --- | --- |
| 粒度 | 整文件覆盖 | 只改一行 |
| 适用 | 配置文件整体替换 | 微调某一行 |
| 风险 | 会覆盖其他手改内容 | 精准，不动其他行 |

---

### 5.14 动手练习（模块综合）

1. 用 `ping` 验证所有主机连通。
2. 用 `dnf` 模块给所有主机安装 `nginx`、`vim`、`wget`。
3. 用 `user` + `group` 创建 `deploy` 用户和 `devops` 组。
4. 用 `file` 创建 `/data/www` 目录，权限 0755。
5. 用 `copy` 把控制端一个文件分发到所有主机 `/etc/`。
6. 用 `lineinfile` 在 `/etc/hosts` 里追加一行控制端记录。
7. 用 `cron` 添加一个每天 02:00 的任务。
8. 分别用 `command` 和 `shell` 执行 `echo hello | wc -c`，观察 command 为什么失败。

---

## 6. 临时命令 ansible 与 ad-hoc

### 6.1 是什么：ad-hoc 命令

"ad-hoc"（临时命令）就是**不写 Playbook 文件**，直接在命令行用 `ansible` 执行一条任务。适合快速查询、快速执行一次性的操作。

### 6.2 为什么用

- 快：一条命令立刻执行，不用先写文件。
- 适合巡检、临时操作、调试连通性。

### 6.3 怎么用：通用语法

```bash
ansible <目标> -m <模块> -a "<参数>" [-i <清单文件>] [-u <用户>] [-b] [-e "<变量>"] [其他选项]
```

| 选项 | 说明 |
| --- | --- |
| `<目标>` | 主机、组名、`all`，或 IP |
| `-m` | 模块名（默认 command） |
| `-a` | 模块参数 |
| `-i` | 指定 Inventory 文件（默认 `/etc/ansible/hosts`） |
| `-u` | 以哪个用户连接 |
| `-b` | become，提权（sudo 到 root） |
| `-e` | 传额外变量（key=value） |
| `-f` | 并发数（同时操作多少台，默认 5） |
| `--list-hosts` | 只列出匹配的主机，不执行 |
| `--limit` | 只对匹配的某台/某组执行 |

### 6.4 实战示例

```bash
# 1. 对 all 执行 ping
ansible all -m ping

# 2. 对 web 组执行 uptime
ansible web -a "uptime"

# 3. 指定 Inventory 文件
ansible web -i ./my_hosts -m ping

# 4. 用普通用户连接并提权（sudo）
ansible web -u deploy -b -a "dnf install -y htop"

# 5. 传变量
ansible web -m shell -a "echo {{ greeting }}" -e "greeting=hello"

# 6. 只看某台机器
ansible all -m ping --limit web1

# 7. 列出清单里所有主机
ansible all --list-hosts
```

### 6.5 动手练习

1. 用 ad-hoc 查所有主机的内存：`ansible all -a "free -h"`。
2. 用 ad-hoc 查所有主机磁盘：`ansible all -m shell -a "df -h /"`。
3. 用 `--limit` 只对其中一台执行，体会限制范围。
4. 用 `-e` 传变量配合 `shell` 输出一段自定义文本。

---

## 7. Playbook 基础（YAML + hosts/tasks + --check）

### 7.1 是什么：Playbook

Playbook（剧本）是用 **YAML** 写的自动化任务清单。它是 Ansible 的"代码"，把多个任务按顺序组织起来，可重复执行、可版本管理。

- ad-hoc 是"单条命令"；Playbook 是"一批命令的编排"。
- Playbook 能处理条件、循环、变量、模板、通知等高级逻辑，ad-hoc 做不到。

### 7.2 YAML 语法（必须掌握）

YAML 是"Yet Another Markup Language"（另一种标记语言），特点是用**缩进**表示层级，**冒号+空格**表示键值。

**核心规则：**

| 规则 | 说明 | 错误示例 | 正确示例 |
| --- | --- | --- | --- |
| 缩进用空格（禁止 Tab） | 层级靠空格，一般 2 个空格 | `\t tasks:` | `  tasks:` |
| 键值用 `key: value` | 冒号后必须有空格 | `name:install` | `name: install` |
| 列表用 `- ` | 短横线+空格表示列表项 | `-copy:` | `- copy:` |
| 字符串可加可不加引号 | 含特殊字符要加引号 | `msg: hello: world` | `msg: "hello: world"` |
| 注释用 `#` | 从 `#` 到行尾是注释 | — | `# 这是注释` |

**字典（键值）示例：**

```yaml
name: install nginx
state: present
```

**列表（数组）示例：**

```yaml
tasks:
  - name: 第一个任务
    dnf:
      name: nginx
  - name: 第二个任务
    service:
      name: nginx
      state: started
```

### 7.3 Playbook 结构

一个 Playbook 由若干 **Play** 组成，每个 Play 里定义 `hosts`（对谁执行）和 `tasks`（做什么）。

```yaml
---
- name: 安装并启动 nginx          # Play 的描述
  hosts: web                      # 对 web 组执行
  become: yes                     # 提权为 root
  tasks:                          # 任务列表
    - name: 安装 nginx            # 每个任务的描述
      dnf:                        # 模块名
        name: nginx               # 模块参数
        state: present
    - name: 启动 nginx 并自启
      systemd:
        name: nginx
        state: started
        enabled: true
```

**关键字段解释：**

| 字段 | 说明 |
| --- | --- |
| `---` | YAML 文档起始标记（可选但推荐） |
| `name` | Play 或 Task 的描述，执行时会打印出来，方便看进度 |
| `hosts` | 目标主机或组 |
| `become: yes` | 是否提权（等价 `sudo`），管理 root 权限操作时必加 |
| `tasks` | 任务列表，按顺序执行 |
| `dnf:` / `systemd:` | 模块名，参数写在下面缩进 |

### 7.4 state 参数（幂等核心）

`state` 是 Ansible 最常见的参数，表达"目标状态"而非"动作"：

| 模块 | state 取值 | 含义 |
| --- | --- | --- |
| dnf/yum | present / absent / latest | 已安装 / 已卸载 / 最新版 |
| file | directory / file / link / absent / touch | 目录 / 文件 / 软链 / 不存在 / 空文件 |
| service/systemd | started / stopped / restarted / reloaded | 启动 / 停止 / 重启 / 重载 |
| user/group | present / absent | 存在 / 不存在 |

**为什么强调 state**：Ansible 是"声明式"的——你告诉它"最终要什么状态"，它自己判断要不要动手。这保证了幂等性。

### 7.5 运行 Playbook：ansible-playbook

**语法：**

```bash
ansible-playbook <playbook.yml> [选项]
```

**常用选项：**

| 选项 | 说明 |
| --- | --- |
| `-i` | 指定 Inventory |
| `--check` | 检查模式（试运行，不真正改系统） |
| `--diff` | 显示将要做的差异（配合 --check） |
| `-e "k=v"` | 传额外变量 |
| `--limit` | 只对部分主机执行 |
| `-C` | `--check` 的简写 |
| `--syntax-check` | 只检查 YAML 语法，不执行 |

**完整示例：**

```bash
# 1. 先做语法检查
ansible-playbook install_nginx.yml --syntax-check

# 2. 试运行（只检查，不改系统）
ansible-playbook install_nginx.yml --check

# 3. 正式执行
ansible-playbook install_nginx.yml
```

**预期结果（正式执行）：**

```text
PLAY [安装并启动 nginx] *****************************************************

TASK [Gathering Facts] *****************************************************
ok: [web1]
ok: [web2]

TASK [安装 nginx] ***********************************************************
changed: [web1]
changed: [web2]

TASK [启动 nginx 并自启] *****************************************************
changed: [web1]
changed: [web2]

PLAY RECAP *********************************************************************
web1 : ok=3  changed=2  unreachable=0  failed=0  skipped=0  rescued=0  ignored=0
web2 : ok=3  changed=2  unreachable=0  failed=0  skipped=0  rescued=0  ignored=0
```

**输出字段解释：**

| 字段 | 说明 |
| --- | --- |
| `PLAY` | 当前 Play 标题 |
| `TASK` | 当前任务标题 |
| `Gathering Facts` | 自动收集被管端信息（IP、系统版本等），可用 `gather_facts: no` 关闭 |
| `ok` | 成功且未变更 |
| `changed` | 成功且发生变更 |
| `PLAY RECAP` | 汇总：每台机器 ok/changed/failed 统计 |
| `failed` | 失败数，非 0 说明有问题 |

### 7.6 --check 检查模式详解

`--check` 是"模拟运行"：Ansible 会预测哪些任务会变更，但**不真正执行**（少数模块如 command/shell 在 check 模式下会跳过）。

```bash
# 试运行 + 显示差异
ansible-playbook install_nginx.yml --check --diff
```

**预期结果：**

```text
TASK [安装 nginx] ***********************************************************
ok: [web1]              # 已安装，无需变更
changed: [web2]         # 预测 web2 会变更（未安装）

TASK [启动 nginx 并自启] *****************************************************
ok: [web1]
changed: [web2]
```

**为什么用 --check**：正式执行前先"彩排"，发现潜在错误（如路径写错、变量未定义），避免误操作生产环境。

### 7.7 动手练习

1. 新建 `install_nginx.yml`，内容为：安装 nginx、启动并自启、创建 `/data/www` 目录。
2. 用 `--syntax-check` 检查语法。
3. 用 `--check` 试运行，观察哪些任务预测为 changed。
4. 正式执行，观察 PLAY RECAP 汇总。
5. 故意在 YAML 里少打一个空格，运行观察报错信息，学会排查 YAML 错误。

---

## 8. 变量 / 条件 when / 循环 loop / 模板 jinja2 / handlers / tags

### 8.1 变量（Variables）

**是什么**：用一个名字代替会变化的值，让 Playbook 更灵活、可复用。

**为什么用**：同一套 Playbook，通过不同变量值适配不同主机（不同 IP、端口、用户名），避免写死。

**怎么用**：变量可以在多处定义，优先级从低到高：

1. Inventory 里（`[组:vars]` 或主机变量）
2. Playbook 里 `vars:` 块
3. 命令行 `-e` 传入（优先级最高）

**示例（Playbook 里定义变量）：**

```yaml
---
- name: 使用变量示例
  hosts: web
  vars:
    pkg_name: nginx
    service_name: nginx
    http_port: 8080
  tasks:
    - name: 安装 {{ pkg_name }}
      dnf:
        name: "{{ pkg_name }}"
        state: present
    - name: 输出变量
      debug:
        msg: "将监听端口 {{ http_port }}"
```

**引用变量**：用双花括号 `{{ 变量名 }}`。注意：当变量作为模块参数**第一个词**时，整个值要加引号，如 `name: "{{ pkg_name }}"`。

**命令行传变量（覆盖）：**

```bash
ansible-playbook vars.yml -e "http_port=9090"
```

**debug 模块**：打印变量/信息，调试必备。

```yaml
- debug:
    msg: "当前包名是 {{ pkg_name }}"
```

**预期结果：**

```text
TASK [输出变量] ***************************
ok: [web1] => {
    "msg": "将监听端口 8080"
}
```

### 8.2 条件 when

**是什么**：满足某个条件时才执行该任务。

**为什么用**：不同系统、不同主机执行不同逻辑（如 CentOS 7 用 yum，CentOS 8+ 用 dnf）。

**怎么用：**

```yaml
---
- name: 条件示例
  hosts: all
  tasks:
    - name: CentOS 8+ 用 dnf 安装
      dnf:
        name: nginx
        state: present
      when: ansible_facts['os_family'] == "RedHat" and ansible_facts['distribution_major_version'] >= "8"

    - name: CentOS 7 用 yum 安装
      yum:
        name: nginx
        state: present
      when: ansible_facts['os_family'] == "RedHat" and ansible_facts['distribution_major_version'] == "7"
```

**常用条件判断符：**

| 符号 | 含义 | 示例 |
| --- | --- | --- |
| `==` | 等于 | `when: ansible_facts['os_family'] == "RedHat"` |
| `!=` | 不等于 | `when: result != 0` |
| `and` / `or` | 与 / 或 | `when: a == 1 and b == 2` |
| `is defined` | 变量已定义 | `when: my_var is defined` |
| `not` | 取反 | `when: not ansible_check_mode` |

**条件里还能用 register 的结果**（register 见下）：

```yaml
- name: 检查 nginx 是否在运行
  shell: "systemctl is-active nginx"
  register: nginx_status
  ignore_errors: true

- name: 如果没运行就启动
  systemd:
    name: nginx
    state: started
  when: nginx_status.stdout != "active"
```

**register 是什么**：把任务执行结果保存到变量，供后续任务判断或复用。

### 8.3 循环 loop / with_items

**是什么**：让一个任务对一组数据重复执行。

**为什么用**：批量装多个包、创建多个用户、复制多个文件，不用写 N 个任务。

**怎么用：**

**（1）loop 写法（推荐，新版）：**

```yaml
---
- name: 循环示例
  hosts: all
  tasks:
    - name: 批量安装多个包
      dnf:
        name: "{{ item }}"
        state: present
      loop:
        - nginx
        - vim
        - git
        - wget
```

**（2）with_items 写法（老版，兼容）：**

```yaml
    - name: 批量创建多个用户
      user:
        name: "{{ item }}"
        state: present
      with_items:
        - alice
        - bob
        - carol
```

**对比 loop 和 with_items**：

| | loop | with_items |
| --- | --- | --- |
| 出现时间 | Ansible 2.5+ | 早期版本 |
| 推荐度 | 推荐 | 兼容保留 |
| 写法 | `loop:` + 列表 | `with_items:` + 列表 |

**循环 + 字典（更复杂场景）：**

```yaml
    - name: 批量创建用户和对应组
      user:
        name: "{{ item.name }}"
        group: "{{ item.group }}"
        state: present
      loop:
        - { name: 'alice', group: 'dev' }
        - { name: 'bob', group: 'ops' }
```

**预期结果：** 任务会对 `item` 依次取值，为每个用户执行一次 user 模块。

### 8.4 模板 template（Jinja2）

**是什么**：Jinja2 是 Python 的模板引擎，Ansible 用它把变量、条件、循环"填进"模板文件，生成最终配置。

**为什么用**：不同主机要不同配置（IP、端口、内存大小），用一份模板 + 变量批量生成，避免为每台机器手写配置。

**怎么用**：模板文件后缀 `.j2`，变量用 `{{ }}`，逻辑用 `{% %}`。

**Jinja2 常用语法：**

| 语法 | 作用 | 示例 |
| --- | --- | --- |
| `{{ 变量 }}` | 输出变量 | `listen {{ port }};` |
| `{% if %}...{% endif %}` | 条件 | `{% if env == "prod" %}...{% endif %}` |
| `{% for %}...{% endfor %}` | 循环 | `{% for ip in ips %}...{% endfor %}` |
| `{{ 变量 | default("值") }}` | 默认值 | `{{ port | default("80") }}` |
| `{# 注释 #}` | 模板注释 | `{# 这是注释 #}` |

**完整示例：**

模板文件 `nginx.conf.j2`：

```nginx
# 模板生成，主机名：{{ ansible_hostname }}
server {
    listen {{ http_port | default('80') }};
    server_name {{ server_name }};

    {% if enable_ssl %}
    listen 443 ssl;
    ssl_certificate {{ ssl_cert }};
    {% endif %}

    location / {
        root {{ web_root | default('/usr/share/nginx/html') }};
        index index.html;
    }
}
```

Playbook 使用模板：

```yaml
---
- name: 模板示例
  hosts: web
  vars:
    http_port: 8080
    server_name: web1.example.com
    enable_ssl: false
    web_root: /data/www
  tasks:
    - name: 渲染 nginx 配置
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/conf.d/app.conf
        owner: root
        group: root
        mode: "0644"
      notify: restart nginx

  handlers:
    - name: restart nginx
      systemd:
        name: nginx
        state: restarted
```

**渲染后的结果（在 web1 上的 app.conf）：**

```nginx
# 模板生成，主机名：web1
server {
    listen 8080;
    server_name web1.example.com;

    location / {
        root /data/www;
        index index.html;
    }
}
```

**原理说明**：`{{ http_port }}` 被替换成 `8080`；`{% if enable_ssl %}` 因为 `enable_ssl` 为 `false`，整段 SSL 配置被移除。这就是"一份模板，N 种配置"。

### 8.5 handlers（触发器）

**是什么**：handlers 是一类特殊任务，只有被 `notify` 通知时才执行，且**每个 handler 在一个 Play 里最多执行一次**（无论被通知多少次）。

**为什么用**：典型场景是"配置文件变了才重启服务"。如果配置没变（幂等、未 changed），就不重启，避免无意义的重启。

**怎么用：**

```yaml
---
- name: handlers 示例
  hosts: web
  tasks:
    - name: 修改 nginx 配置
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify: restart nginx    # 触发 handler

    - name: 修改第二个配置
      copy:
        src: app.conf
        dest: /etc/nginx/conf.d/app.conf
      notify: restart nginx    # 也触发同一个 handler

  handlers:
    - name: restart nginx       # handler 名字要和 notify 一致
      systemd:
        name: nginx
        state: restarted
```

**执行逻辑：**

- 如果两个任务都 `changed`，`notify` 会通知 handler 两次，但 handler 只执行一次（在 tasks 全部结束后执行）。
- 如果两个任务都 `ok`（未变更），handler 不会执行。

**预期结果：**

```text
TASK [修改 nginx 配置] **********
changed: [web1]

RUNNING HANDLER [restart nginx] **********
changed: [web1]
```

### 8.6 tags（标签）

**是什么**：给任务打标签，可以选择性执行某类任务。

**为什么用**：大型 Playbook 只想跑其中一部分（如只跑"装包"、只跑"配防火墙"），用 tags 精确控制。

**怎么用：**

```yaml
---
- name: tags 示例
  hosts: web
  tasks:
    - name: 安装 nginx
      dnf:
        name: nginx
        state: present
      tags:
        - install
        - pkg

    - name: 配置防火墙
      firewalld:
        service: http
        permanent: true
        state: enabled
      tags:
        - firewall

    - name: 启动服务
      systemd:
        name: nginx
        state: started
      tags:
        - service
```

**运行指定标签：**

```bash
# 只执行打了 install 标签的任务
ansible-playbook site.yml --tags install

# 只执行 firewall 和 service 标签的任务
ansible-playbook site.yml --tags "firewall,service"

# 跳过某个标签（执行除了 install 之外的所有任务）
ansible-playbook site.yml --skip-tags install

# 列出所有标签
ansible-playbook site.yml --list-tags
```

**预期结果（--list-tags）：**

```text
playbook: site.yml
  play #1 (web): tags: []
      TASK TAGS: [install, pkg, firewall, service]
```

### 8.7 动手练习

1. 写一个 Playbook，用 `vars` 定义 `pkg_list`，用 `loop` 批量安装 4 个软件包。
2. 用 `when` 判断系统版本，CentOS 8+ 用 dnf、CentOS 7 用 yum（结合 `ansible_facts`）。
3. 写一个 nginx 模板，用变量控制端口和是否启用 SSL，用 `template` 渲染。
4. 给"改配置"任务加 `notify`，配一个"重启服务"的 handler，验证配置未变时 handler 不执行。
5. 给任务加 tags，用 `--tags` 只跑其中一部分。

---

## 9. Roles 角色

### 9.1 是什么：Roles 目录结构

Role（角色）是把 Playbook 按"功能"拆分成标准目录结构的组织方式。一个 Role 就是"一组相关任务的封装"，例如 `nginx` role 负责装 nginx、配 nginx、启 nginx。

**为什么用**：Playbook 全写在一个文件里，任务多了会又长又乱、难复用。Roles 让代码模块化，像"积木"一样可以复用、可以分享（ansible-galaxy 下载别人的角色）。

### 9.2 Role 标准目录结构

```text
roles/
└── nginx/                    # 角色名
    ├── tasks/                # 任务：角色要执行的操作
    │   └── main.yml          # 入口任务文件（必填）
    ├── handlers/             # 触发器：restart/reload 服务等
    │   └── main.yml
    ├── templates/            # 模板：*.j2 文件
    │   └── nginx.conf.j2
    ├── files/                # 静态文件：copy 模块的源文件
    │   └── index.html
    ├── vars/                 # 变量：角色内部变量（优先级高）
    │   └── main.yml
    ├── defaults/             # 默认变量：可被外部覆盖（优先级低）
    │   └── main.yml
    ├── meta/                 # 角色元数据：依赖、作者信息
    │   └── main.yml
    └── README.md             # 说明文档
```

**各目录作用详解：**

| 目录 | 作用 | 何时自动加载 |
| --- | --- | --- |
| `tasks/` | 主任务列表 | 角色被引用时自动执行 `main.yml` |
| `handlers/` | 触发器 | 被 notify 触发时自动加载 |
| `templates/` | Jinja2 模板文件，用 `template` 模块引用时**不用写全路径** | 自动查找 |
| `files/` | 静态文件，用 `copy` 模块引用时不用写全路径 | 自动查找 |
| `vars/` | 角色私有变量（优先级高，不易被覆盖） | 自动加载 |
| `defaults/` | 默认变量（优先级最低，最易被覆盖） | 自动加载 |
| `meta/` | 依赖关系、作者、平台说明 | 加载角色时读取 |

**关键点**：`templates/` 和 `files/` 里的文件，在 `template`/`copy` 模块里直接写文件名即可（Ansible 会自动到角色目录下找），**不需要写绝对路径**。

### 9.3 手动创建一个 nginx Role

**第一步：创建目录结构**

```bash
mkdir -p roles/nginx/{tasks,handlers,templates,files,vars,defaults,meta}
```

**第二步：写 tasks/main.yml**

```yaml
---
# roles/nginx/tasks/main.yml
- name: 安装 nginx
  dnf:
    name: nginx
    state: present

- name: 创建网站根目录
  file:
    path: "{{ web_root }}"
    state: directory
    owner: root
    group: root
    mode: "0755"

- name: 渲染 nginx 配置
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/conf.d/app.conf
    owner: root
    group: root
    mode: "0644"
  notify: reload nginx

- name: 上传首页
  copy:
    src: index.html
    dest: "{{ web_root }}/index.html"
    owner: root
    group: root
    mode: "0644"

- name: 启动 nginx 并自启
  systemd:
    name: nginx
    state: started
    enabled: true
```

**第三步：写 handlers/main.yml**

```yaml
---
# roles/nginx/handlers/main.yml
- name: reload nginx
  systemd:
    name: nginx
    state: reloaded
```

**第四步：写 templates/nginx.conf.j2**

```nginx
server {
    listen {{ http_port | default('80') }};
    server_name {{ server_name | default('localhost') }};

    location / {
        root {{ web_root | default('/usr/share/nginx/html') }};
        index index.html;
    }
}
```

**第五步：写 defaults/main.yml（默认变量，可被覆盖）**

```yaml
---
# roles/nginx/defaults/main.yml
http_port: 80
server_name: localhost
web_root: /usr/share/nginx/html
```

**第六步：写 files/index.html**

```html
<!DOCTYPE html>
<html>
<head><meta charset="utf-8"><title>Ansible 部署</title></head>
<body>
<h1>Hello, Ansible!</h1>
<p>本页面由 Ansible 自动部署</p>
</body>
</html>
```

**第七步：写调用 Role 的 Playbook**

```yaml
---
# deploy-nginx.yml
- name: 用 Role 部署 nginx
  hosts: web
  become: yes
  roles:
    - nginx
```

**运行：**

```bash
ansible-playbook deploy-nginx.yml
```

**预期结果：** Role 会自动按 tasks/main.yml 顺序执行：装包 → 建目录 → 渲染配置 → 传首页 → 启服务。PLAY RECAP 显示 changed 数量。

**覆盖默认变量：**

```yaml
- name: 用 Role 部署 nginx（自定义端口）
  hosts: web
  become: yes
  roles:
    - role: nginx
      vars:
        http_port: 8080
        server_name: myapp.example.com
        web_root: /data/www
```

### 9.4 ansible-galaxy

**是什么**：Ansible 自带的"角色/集合"管理工具，可以搜索、下载、创建角色。

**为什么用**：不用重复造轮子，社区有海量现成 Role（如 geerlingguy.nginx、geerlingguy.mysql），下载即用；也能快速生成标准骨架。

**怎么用：**

```bash
# 1. 从 Ansible Galaxy 安装一个角色到默认 roles/ 目录
ansible-galaxy install geerlingguy.nginx

# 2. 指定安装路径
ansible-galaxy install geerlingguy.nginx -p ./roles

# 3. 搜索角色
ansible-galaxy search nginx

# 4. 快速生成一个角色骨架
ansible-galaxy init myrole

# 5. 查看已安装的角色
ansible-galaxy list
```

**预期结果（init 生成骨架）：**

```text
- Role myrole was created successfully
```

生成的结构就是标准 Role 目录（tasks/handlers/defaults/vars/meta/README 等）。

### 9.5 动手练习

1. 用 `ansible-galaxy init webserver` 生成角色骨架，观察目录结构。
2. 手写一个 `nginx` 角色，包含装包、渲染模板、启动服务、handler。
3. 写一个调用该 Role 的 Playbook 并运行。
4. 用 `ansible-galaxy install geerlingguy.nginx` 下载一个社区角色，查看它的目录结构。

---

## 10. 完整项目：一键初始化新服务器 + 批量部署网站

> 现在把前面所有知识串起来，做一个真实项目。项目分两个 Role 和一个主 Playbook。

### 10.1 项目目标

- **Role 1：`common`** —— 一键初始化新服务器：装基础包、配置软件源、创建用户、开防火墙。
- **Role 2：`nginx_site`** —— 批量部署网站：装 nginx、渲染配置、上传站点文件。
- **主 Playbook：`site.yml`** —— 统一调用两个 Role。

### 10.2 创建项目结构

```bash
mkdir -p ansible-project/roles/common/{tasks,handlers,templates,files,defaults}
mkdir -p ansible-project/roles/nginx_site/{tasks,handlers,templates,files,defaults}
cd ansible-project
```

### 10.3 Role 1：common（初始化新服务器）

**roles/common/tasks/main.yml：**

```yaml
---
# 初始化新服务器：基础包、软件源、用户、防火墙
- name: 安装基础软件包
  dnf:
    name: "{{ item }}"
    state: present
    update_cache: yes
  loop:
    - vim
    - wget
    - curl
    - git
    - net-tools
    - htop
    - chrony

- name: 配置 chrony 时间同步并启动
  systemd:
    name: chronyd
    state: started
    enabled: true

- name: 创建部署用户
  user:
    name: "{{ deploy_user }}"
    state: present
    shell: /bin/bash
    create_home: yes

- name: 创建部署组
  group:
    name: "{{ deploy_group }}"
    state: present

- name: 把部署用户加入组
  user:
    name: "{{ deploy_user }}"
    groups: "{{ deploy_group }}"
    append: yes

- name: 开启防火墙并自启
  systemd:
    name: firewalld
    state: started
    enabled: true

- name: 放行 http 服务
  firewalld:
    service: http
    permanent: true
    state: enabled
    immediate: true

- name: 放行 https 服务
  firewalld:
    service: https
    permanent: true
    state: enabled
    immediate: true
```

**roles/common/defaults/main.yml：**

```yaml
---
deploy_user: deploy
deploy_group: devops
```

> 说明：`firewalld` 模块的 `permanent: true` 表示永久生效（写入配置），`immediate: true` 表示立即生效（不用重启防火墙）。

### 10.4 Role 2：nginx_site（批量部署网站）

**roles/nginx_site/tasks/main.yml：**

```yaml
---
- name: 安装 nginx
  dnf:
    name: nginx
    state: present

- name: 创建网站根目录
  file:
    path: "{{ web_root }}"
    state: directory
    owner: nginx
    group: nginx
    mode: "0755"

- name: 渲染 nginx 站点配置
  template:
    src: site.conf.j2
    dest: /etc/nginx/conf.d/{{ site_name }}.conf
    owner: root
    group: root
    mode: "0644"
  notify: reload nginx

- name: 上传网站首页
  template:
    src: index.html.j2
    dest: "{{ web_root }}/index.html"
    owner: nginx
    group: nginx
    mode: "0644"

- name: 启动 nginx 并自启
  systemd:
    name: nginx
    state: started
    enabled: true
```

**roles/nginx_site/handlers/main.yml：**

```yaml
---
- name: reload nginx
  systemd:
    name: nginx
    state: reloaded
```

**roles/nginx_site/templates/site.conf.j2：**

```nginx
server {
    listen {{ http_port }};
    server_name {{ server_name }};

    location / {
        root {{ web_root }};
        index index.html;
    }

    access_log /var/log/nginx/{{ site_name }}.access.log;
    error_log  /var/log/nginx/{{ site_name }}.error.log;
}
```

**roles/nginx_site/templates/index.html.j2：**

```html
<!DOCTYPE html>
<html>
<head><meta charset="utf-8"><title>{{ site_name }}</title></head>
<body>
<h1>欢迎访问 {{ site_name }}</h1>
<p>主机名：{{ ansible_hostname }}</p>
<p>部署时间：{{ ansible_date_time.iso8601 }}</p>
</body>
</html>
```

**roles/nginx_site/defaults/main.yml：**

```yaml
---
http_port: 80
server_name: localhost
site_name: myapp
web_root: /usr/share/nginx/html
```

### 10.5 主 Playbook：site.yml

```yaml
---
# site.yml：一键初始化 + 部署网站
- name: 初始化所有服务器
  hosts: all
  become: yes
  roles:
    - common

- name: 部署网站到 web 组
  hosts: web
  become: yes
  roles:
    - role: nginx_site
      vars:
        http_port: 80
        server_name: "{{ inventory_hostname }}"
        site_name: myapp
        web_root: /data/www/myapp
```

> 说明：`{{ inventory_hostname }}` 是 Ansible 内置变量，表示当前主机的清单名（如 web1、web2），这样每台机器的 `server_name` 自动取自己的名字。

### 10.6 运行整个项目

```bash
# 语法检查
ansible-playbook site.yml --syntax-check

# 试运行
ansible-playbook site.yml --check

# 正式执行
ansible-playbook site.yml
```

### 10.7 验证

```bash
# 1. 验证服务运行
ansible web -m shell -a "systemctl is-active nginx firewalld chronyd"

# 2. 验证防火墙放行
ansible web -m shell -a "firewall-cmd --list-services"

# 3. 验证用户创建
ansible web -m shell -a "id deploy"

# 4. 本地访问网站（在控制端执行）
curl http://192.168.1.101/
```

**预期 curl 结果：**

```html
<!DOCTYPE html>
<html>
<head><meta charset="utf-8"><title>myapp</title></head>
<body>
<h1>欢迎访问 myapp</h1>
<p>主机名：web1</p>
<p>部署时间：2026-11-05T14:30:00+00:00</p>
</body>
</html>
```

### 10.8 动手练习

1. 按上述结构手动创建 `common` 和 `nginx_site` 两个 Role。
2. 编写 `site.yml` 调用两个 Role。
3. 用 `--check` 试运行，用正式运行部署。
4. 修改 `nginx_site/defaults/main.yml` 里的 `web_root`，重新运行，观察配置更新和 handler 重启。

---

## 11. 本章项目验收（3 台机器装 Nginx + 部署网站）

> 这是最终验收：从零到完成，把 3 台机器一次跑通。请严格按步骤操作，每步都能看到预期结果。

### 11.1 验收环境

| 角色 | IP | 说明 |
| --- | --- | --- |
| 控制端 | 192.168.1.100 | 已装 Ansible |
| web1 | 192.168.1.101 | 被管端 |
| web2 | 192.168.1.102 | 被管端 |
| web3 | 192.168.1.103 | 被管端 |

### 11.2 验收步骤（照着敲）

**步骤 1：准备控制端，确认 Ansible 版本**

```bash
ansible --version
```

预期：显示 `ansible [core 2.x]` 及 python 版本。

**步骤 2：配置免密 SSH（3 台）**

```bash
ssh-keygen -t rsa -b 4096 -C "ansible-control"   # 一路回车
ssh-copy-id root@192.168.1.101
ssh-copy-id root@192.168.1.102
ssh-copy-id root@192.168.1.103

# 验证免密
ssh root@192.168.1.101 "hostname"
ssh root@192.168.1.102 "hostname"
ssh root@192.168.1.103 "hostname"
```

预期：三台都直接返回主机名，无需密码。

**步骤 3：编写 Inventory**

```bash
cat > /etc/ansible/hosts <<'EOF'
[web]
web1 ansible_host=192.168.1.101
web2 ansible_host=192.168.1.102
web3 ansible_host=192.168.1.103

[web:vars]
ansible_user=root
ansible_ssh_port=22
EOF
```

**验证清单：**

```bash
ansible all --list-hosts
```

预期：列出 3 台主机。

**步骤 4：连通性测试**

```bash
ansible all -m ping
```

预期：3 台都返回 `"ping": "pong"`。

**步骤 5：创建项目目录和 Roles**

```bash
mkdir -p /root/ansible-project/roles/{common,nginx_site}/{tasks,handlers,templates,files,defaults}
cd /root/ansible-project
```

**步骤 6：编写 common Role（初始化）**

写入 `roles/common/tasks/main.yml`：

```yaml
---
- name: 安装基础软件包
  dnf:
    name: "{{ item }}"
    state: present
  loop: [vim, wget, curl, git, net-tools, htop, chrony]

- name: 创建部署用户
  user:
    name: deploy
    state: present
    shell: /bin/bash

- name: 启动并开启防火墙
  systemd:
    name: firewalld
    state: started
    enabled: true

- name: 放行 http 和 https
  firewalld:
    service: "{{ item }}"
    permanent: true
    state: enabled
    immediate: true
  loop: [http, https]
```

**步骤 7：编写 nginx_site Role（部署网站）**

写入 `roles/nginx_site/tasks/main.yml`：

```yaml
---
- name: 安装 nginx
  dnf:
    name: nginx
    state: present

- name: 创建站点目录
  file:
    path: /data/www/myapp
    state: directory
    owner: nginx
    group: nginx
    mode: "0755"

- name: 渲染站点配置
  template:
    src: site.conf.j2
    dest: /etc/nginx/conf.d/myapp.conf
    owner: root
    group: root
    mode: "0644"
  notify: reload nginx

- name: 上传首页
  template:
    src: index.html.j2
    dest: /data/www/myapp/index.html
    owner: nginx
    group: nginx
    mode: "0644"

- name: 启动 nginx 并自启
  systemd:
    name: nginx
    state: started
    enabled: true
```

写入 `roles/nginx_site/handlers/main.yml`：

```yaml
---
- name: reload nginx
  systemd:
    name: nginx
    state: reloaded
```

写入 `roles/nginx_site/templates/site.conf.j2`：

```nginx
server {
    listen 80;
    server_name {{ inventory_hostname }};

    location / {
        root /data/www/myapp;
        index index.html;
    }
}
```

写入 `roles/nginx_site/templates/index.html.j2`：

```html
<!DOCTYPE html>
<html>
<head><meta charset="utf-8"><title>{{ inventory_hostname }}</title></head>
<body>
<h1>Hello from {{ inventory_hostname }}</h1>
<p>Deployed by Ansible at {{ ansible_date_time.iso8601 }}</p>
</body>
</html>
```

**步骤 8：编写主 Playbook**

写入 `site.yml`：

```yaml
---
- name: 初始化服务器
  hosts: all
  become: yes
  roles:
    - common

- name: 部署网站
  hosts: web
  become: yes
  roles:
    - nginx_site
```

**步骤 9：语法检查 + 试运行**

```bash
ansible-playbook site.yml --syntax-check
ansible-playbook site.yml --check
```

预期：语法通过；`--check` 显示预测 changed 的任务（首次运行大多数任务都 changed）。

**步骤 10：正式执行（一次全部生效）**

```bash
ansible-playbook site.yml
```

预期 PLAY RECAP 三台机器 `failed=0`，且 `changed>0`。

**步骤 11：验证结果**

```bash
# 验证 nginx 运行
ansible web -m shell -a "systemctl is-active nginx"

# 验证防火墙放行 http
ansible web -m shell -a "firewall-cmd --list-services"

# 验证网站可访问（在控制端）
curl http://192.168.1.101/
curl http://192.168.1.102/
curl http://192.168.1.103/
```

**步骤 12：改配置，验证一键更新**

修改 `roles/nginx_site/templates/site.conf.j2`，把 `listen 80` 改成 `listen 8080`，然后重新运行：

```bash
ansible-playbook site.yml
```

预期：`template` 任务 changed，触发 handler reload，nginx 监听端口变为 8080。用 `curl http://192.168.1.101:8080/` 验证。

> 这一步验证了"改配置 → 一次运行全部生效"的核心能力：3 台机器无需逐台登录，一条命令全部更新。

### 11.3 验收检查清单

| 检查项 | 通过标准 |
| --- | --- |
| 免密 SSH | 3 台均可免密登录 |
| Inventory | `ansible all --list-hosts` 列出 3 台 |
| 连通性 | `ping` 全部返回 pong |
| 基础包 | 3 台都装上了 vim/git 等 |
| 用户 | 3 台都有 `deploy` 用户 |
| 防火墙 | 3 台 firewalld 运行且放行 http/https |
| nginx | 3 台 nginx 运行且开机自启 |
| 网站 | 3 台 curl 都能返回页面 |
| 改配置生效 | 改端口后一次运行 3 台全部更新 |

### 11.4 常见报错排查

| 报错 | 原因 | 解决 |
| --- | --- | --- |
| `UNREACHABLE!` | 网络不通/端口错/免密失败 | 检查 IP、`ansible_ssh_port`、免密配置 |
| `Failed to connect to the host via ssh` | 被管端 sshd 未启动 | 被管端 `systemctl start sshd` |
| `module firewalld not found` | 被管端未装 firewalld | `dnf install -y firewalld` |
| `Permission denied` | 用户权限不足 | 检查 `become: yes`，或用户 sudo 权限 |
| YAML 语法报错 | 缩进用了 Tab / 冒号后没空格 | 用空格缩进，`--syntax-check` 排查 |
| `changed` 但服务没起来 | 配置有误导致服务启动失败 | 登录被管端 `nginx -t` 检查配置 |

---

## 附：本阶段核心命令速查表

| 场景 | 命令 |
| --- | --- |
| 查看版本 | `ansible --version` |
| 连通性测试 | `ansible all -m ping` |
| 查看清单主机 | `ansible all --list-hosts` |
| ad-hoc 执行命令 | `ansible web -a "uptime"` |
| ad-hoc 提权 | `ansible web -b -a "dnf install -y htop"` |
| Playbook 语法检查 | `ansible-playbook site.yml --syntax-check` |
| Playbook 试运行 | `ansible-playbook site.yml --check` |
| Playbook 执行 | `ansible-playbook site.yml` |
| 指定标签执行 | `ansible-playbook site.yml --tags install` |
| 传变量 | `ansible-playbook site.yml -e "port=8080"` |
| 生成角色骨架 | `ansible-galaxy init myrole` |
| 下载社区角色 | `ansible-galaxy install geerlingguy.nginx` |

---

## 总结

本教程从"为什么要自动化"讲起，覆盖了 Ansible 的安装、免密 SSH、Inventory、14 个常用模块、ad-hoc、Playbook、变量/条件/循环/模板/handlers/tags、Roles，最后通过一个"一键初始化新服务器 + 批量部署网站"的项目完成验收。核心心法只有一句：

> **Ansible = Inventory（管谁）+ Playbook/Role（做什么）+ 幂等（放心重复执行）。**

掌握这三件事，再复杂的批量运维任务都能拆解、编写、执行。
