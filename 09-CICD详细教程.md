# 阶段 9 CI/CD 持续集成与持续交付 — 完整实操教程

> 本教程面向**零基础小白**，从"什么是 CI/CD"讲到"一条流水线全自动上线"。
> 每个知识点都包含【命令示例 + 参数/字段解释 + 作用或原理说明】，照着敲就能跑通。
> 文中所有命令默认在 **Rocky Linux 9 / CentOS Stream 9（dnf 包管理器）** 上执行，使用普通用户操作，需要 root 权限的命令会以 `sudo` 开头。

---

## 目录

- 第 1 章：什么是 CI/CD、为什么需要、经典流水线流程
- 第 2 章：Git 详解（安装、配置、全部常用命令、冲突解决、.gitignore）
- 第 3 章：GitHub 关键操作（简述，指向已有教程）
- 第 4 章：GitLab（dnf 安装、初始化、改密码、仓库/组/用户、MR 流程）
- 第 5 章：Jenkins（安装、初始化、插件、凭据、Job、声明式 Pipeline）
- 第 6 章：完整 CI/CD 项目实战（GitLab → Jenkins → Docker → 镜像仓库 → 自动部署）
- 第 7 章：本章项目验收（改一次代码 push，全自动上线）

---

# 第 1 章 什么是 CI/CD、为什么需要、经典流水线流程

## 1.1 传统软件交付方式的痛点（是什么 / 为什么）

在讲 CI/CD 之前，先看**没有 CI/CD 时**，一个程序员是怎么把代码上线的：

1. 本地写完代码，打包（或者直接 `scp` 传文件）。
2. 手动上传到服务器，手动重启进程。
3. 如果人多，大家各改各的，合并代码时各种冲突、各种"这个版本是我测试过的吗？"。
4. 上线靠"人肉记忆"，出问题回滚困难。

**后果：** 上线慢、容易出错、版本混乱、无法追溯"这次上线到底改了啥"。

**什么是 CI/CD？**

- **CI（Continuous Integration，持续集成）：** 开发人员**频繁地把代码合并到主干**，每次合并都**自动**触发"编译 + 测试"，尽早发现错误。
- **CD（Continuous Delivery / Deployment，持续交付/持续部署）：** 在 CI 通过后，**自动**把软件打包、构建镜像、部署到目标环境。

一句话总结：**CI/CD = 让"代码提交"到"上线"整个过程自动化，人工只做"写代码 + 审核"。**

### 传统方式 vs CI/CD 对比表

| 对比项 | 传统手工交付 | CI/CD 自动化交付 |
| --- | --- | --- |
| 编译 | 手动执行命令 | 每次提交自动编译 |
| 测试 | 手动跑，常常跳过 | 自动跑，失败即阻断 |
| 打包 | 手动打 jar / 手动 docker build | 流水线自动构建镜像 |
| 部署 | 手动 scp + 重启 | SSH / K8s 自动部署 |
| 回滚 | 靠运气和记忆 | 一键回到上一个镜像版本 |
| 可追溯 | 靠嘴说 | 每次构建有编号、有日志、有提交记录 |
| 出错的时机 | 上线后才被发现 | 提交后几分钟内被发现 |

## 1.2 经典流水线流程（怎么做）

一条经典的 CI/CD 流水线（Pipeline）包含以下阶段，每个阶段**只有上一个阶段成功才会继续**：

```
代码提交(Push) → 触发构建(Trigger) → 拉取代码(Checkout) → 编译构建(Build)
     → 自动测试(Test) → 构建 Docker 镜像(Docker Build) → 推送镜像仓库(Push Image)
     → 部署(Deploy: SSH 或 K8s) → 验证(Verify) → 完成
```

各阶段职责表：

| 阶段 | 英文 | 做什么 | 失败会怎样 |
| --- | --- | --- | --- |
| 提交 | Push | 开发者把代码推到 Git 仓库 | — |
| 触发 | Trigger | Webhook 通知 Jenkins 有新提交 | 流水线不启动 |
| 拉取 | Checkout | 把仓库代码克隆到构建机 | 无法构建 |
| 编译 | Build | 编译/打包（如 mvn package、npm run build） | 代码编译不过 |
| 测试 | Test | 单元测试、静态检查 | 有 bug，阻断上线 |
| 镜像 | Docker Build | 用 Dockerfile 打包镜像 | 无镜像可部署 |
| 推送 | Push Image | 把镜像推到 Harbor/私服 | 部署节点拿不到镜像 |
| 部署 | Deploy | SSH 登录目标机拉新镜像重启，或 kubectl 更新 | 上线失败，需告警 |

**原理说明：** 流水线把"人肉操作"变成了"代码"，这个流水线代码就是后面要学的 **Jenkinsfile**。它和业务代码一样存在 Git 里、一样被版本管理，所以"流水线本身"也是可追溯、可回滚的。

## 1.3 动手练习

1. 用自己的话写一段 50 字以内的"CI 是什么、CD 是什么"。
2. 默写流水线 8 个阶段的顺序。
3. 想一想：如果没有"自动测试"这个阶段，一个改了公共方法的程序员会带来什么风险？

---

# 第 2 章 Git 详解

## 2.1 Git 是什么、为什么用（三层）

- **是什么：** Git 是一个**分布式版本控制系统**，记录文件的每一次修改历史。
- **为什么用：** 多人协作、代码回滚、分支开发、追责追溯都靠它。没有 Git，团队写代码就是灾难。
- **怎么用：** 下面从安装开始一步步来。

### 2.1.1 安装 Git

```bash
# Rocky / CentOS Stream（dnf）
sudo dnf install -y git

# 验证安装
git --version
# 预期输出类似：git version 2.43.5
```

- 参数解释：`-y` 表示所有确认都回答 yes，不交互。
- 作用：安装 Git 命令行工具。

## 2.2 全局配置（身份信息）

```bash
# 设置用户名和邮箱（提交历史里会显示，必须设置）
git config --global user.name  "张三"
git config --global user.email "zhangsan@example.com"

# 查看配置
git config --global --list
```

- 参数解释：
  - `--global`：全局生效，写到 `~/.gitconfig`，对这台机器的所有仓库生效。
  - `user.name` / `user.email`：提交者的名字和邮箱。
- 原理：每次 commit 时 Git 会把这些信息写进提交记录，方便追溯"谁改的"。

## 2.3 Git 的"三个区"和"三个状态"（核心原理）

这是新手最容易懵的地方，务必理解：

| 区 | 说明 | 文件状态 |
| --- | --- | --- |
| 工作区（Working Directory） | 你电脑上看得见的文件 | 已修改/未跟踪 |
| 暂存区（Staging Area / Index） | `git add` 之后的"待提交清单" | 已暂存 |
| 版本库（Repository） | `git commit` 之后的正式历史 | 已提交 |

**命令与区的对应关系：**

```
工作区 --git add--> 暂存区 --git commit--> 版本库
版本库 --git checkout/restore--> 工作区
```

原理说明：Git 不是"保存整个文件快照"就完事，而是用**指针 + 差异**来记录，所以可以随时回到任意历史版本。

## 2.4 常用命令逐条详解

### 2.4.1 clone（克隆远程仓库）

```bash
# 把远程仓库完整复制到本地
git clone https://github.com/user/demo.git

# 克隆并指定本地目录名
git clone https://github.com/user/demo.git my-demo
```

- 参数解释：`my-demo` 是本地文件夹名，不写就用仓库名 `demo`。
- 作用：克隆完成后，本地会有一个 `.git` 隐藏目录，里面存了全部历史，且远程地址已自动配置好。

### 2.4.2 add（添加到暂存区）

```bash
git add README.md       # 只添加一个文件
git add .               # 添加当前目录所有改动（最常用）
git add src/            # 添加某个目录
git add -A              # 添加所有（含删除）
```

- 参数解释：`.` 表示当前目录；`-A` 等于 `--all`，连"删除文件"这个动作也记录。
- 作用：把工作区的改动登记到暂存区，此时还没进历史。

### 2.4.3 commit（提交到版本库）

```bash
git commit -m "feat: 新增用户登录接口"
```

- 参数解释：`-m` 后面跟提交说明（message）。规范写法见下表。
- 作用：把暂存区内容正式写入本地版本库，生成一个提交（带唯一 SHA 哈希）。

**提交信息规范（约定式提交）：**

| 前缀 | 含义 | 示例 |
| --- | --- | --- |
| feat | 新功能 | feat: 新增登录接口 |
| fix | 修 bug | fix: 修复密码为空崩溃 |
| docs | 文档 | docs: 更新 README |
| refactor | 重构（不改变功能） | refactor: 抽取公共方法 |
| chore | 杂项 | chore: 升级依赖 |

### 2.4.4 push（推送到远程）

```bash
git push                      # 推送到默认远程和当前分支
git push origin main          # 推送到 origin 的 main 分支
git push -u origin main       # 首次推送并建立"跟踪关系"
```

- 参数解释：
  - `origin`：远程仓库的默认别名（clone 时自动生成）。
  - `main`：要推送到的远程分支名。
  - `-u`（`--set-upstream`）：把本地 `main` 和远程 `origin/main` 绑定，之后直接 `git push` 即可。
- 作用：把本地提交同步到远程服务器（GitHub/GitLab）。

### 2.4.5 pull（从远程拉取并合并）

```bash
git pull                      # 拉取当前分支最新代码并合并
git pull origin main          # 指定远程和分支
```

- 作用：`git pull` = `git fetch`（下载）+ `git merge`（合并）两步的合体。
- 原理：先下载远程新提交，再合并到本地当前分支；如果有冲突，会进入冲突状态（见 2.5）。

### 2.4.6 branch（查看/创建/删除分支）

```bash
git branch              # 列出所有本地分支，前面带 * 的是当前分支
git branch feature-login   # 创建新分支 feature-login
git branch -d feature-login # 删除已合并的分支
git branch -D feature-login # 强制删除未合并的分支
```

- 参数解释：`-d` 安全删除（只有已合并才删），`-D` 强制删除。
- 作用：分支让你在"独立副本"上开发新功能，不影响主干。

### 2.4.7 checkout 与 switch（切换分支）

```bash
# 方式一：checkout（老命令，功能多但易混淆）
git checkout feature-login          # 切换到已存在的分支
git checkout -b feature-login       # 创建并切换到新分支

# 方式二：switch（Git 2.23+ 新命令，职责更清晰，推荐）
git switch feature-login            # 切换到已有分支
git switch -c feature-login         # 创建并切换到新分支
```

- 参数解释：
  - `checkout -b` / `switch -c`：`c` 是 create，创建新分支并切换过去。
- 推荐：**切换分支用 switch，恢复文件用 checkout/restore**，不要混用。

### 2.4.8 merge（合并分支）

```bash
# 先切回要"合并进去"的目标分支，例如 main
git switch main
# 再把 feature 分支合并进来
git merge feature-login
```

- 作用：把 `feature-login` 的改动合并到当前所在分支 `main`。
- 合并结果有两种：
  - **快进合并（Fast-forward）**：main 没有新提交，直接指针前移，无冲突。
  - **三方合并（Three-way）**：两边都有新提交，Git 自动合并；改到同一行才会冲突。

### 2.4.9 log（查看历史）

```bash
git log                        # 完整历史
git log --oneline              # 一行一条，最常用
git log --oneline --graph --all   # 画分支图
git log -3                     # 只看最近 3 条
```

- 参数解释：
  - `--oneline`：每条提交压缩成一行（短 SHA + 说明）。
  - `--graph`：用 ASCII 画分支线。
  - `--all`：显示所有分支历史。
- 作用：查看"谁、什么时候、改了啥"。

### 2.4.10 diff（查看差异）

```bash
git diff                        # 工作区 vs 暂存区（未 add 的改动）
git diff --staged               # 暂存区 vs 版本库（已 add 未 commit 的改动）
git diff HEAD                   # 工作区 vs 最新提交
git diff main..feature-login    # 两个分支之间的差异
```

- 作用：逐行查看改了什么。`+` 开头是新增，`-` 开头是删除。
- 原理：Git 逐行比对文件，输出差异块（hunk）。

## 2.5 解决冲突完整演示（必练）

**场景：** 两个人（模拟用两个目录）改了同一个文件的同一行。

### 第 1 步：准备一个仓库

```bash
mkdir conflict-demo && cd conflict-demo
git init
echo "版本号: v1" > app.txt
git add app.txt
git commit -m "init: v1"
```

- 解释：`git init` 初始化本地仓库，`echo "..." > app.txt` 把内容写入文件。

### 第 2 步：模拟"开发者 A"在 main 上改同一行

```bash
echo "版本号: v2-A" > app.txt
git add app.txt
git commit -m "A: 改成 v2-A"
```

### 第 3 步：模拟"开发者 B"基于旧的 v1 改同一行

```bash
# 回到 v1 的提交（HEAD~1 表示上一次提交）
git checkout HEAD~1 -- app.txt   # 把 app.txt 恢复成 v1 内容
echo "版本号: v2-B" > app.txt
git add app.txt
git commit -m "B: 改成 v2-B"
```

现在历史变成两条岔路，两个提交都改了 `app.txt` 第一行。

### 第 4 步：制造冲突

```bash
# 把当前分支（B 的提交）合并到 main……先假设当前就在 main 的 B 提交上
# 我们用更直观的方式：创建两个分支
```

更清晰的完整演示（推荐照抄）：

```bash
# 重新开一个干净的演示
cd .. && rm -rf conflict-demo && mkdir conflict-demo && cd conflict-demo
git init
echo "第一行内容" > file.txt
git add file.txt && git commit -m "初始版本"

# 创建 feature 分支，改第一行
git switch -c feature
echo "feature 分支的修改" > file.txt
git add file.txt && git commit -m "feature 改了第一行"

# 切回 main，改同一行
git switch main
echo "main 分支的修改" > file.txt
git add file.txt && git commit -m "main 也改了第一行"

# 合并 feature，触发冲突
git merge feature
# 预期输出：CONFLICT (content): Merge conflict in file.txt
#            Automatic merge failed; fix conflicts and then commit the result.
```

### 第 5 步：查看冲突文件

```bash
cat file.txt
# 预期内容如下：
# <<<<<<< HEAD
# main 分支的修改
# =======
# feature 分支的修改
# >>>>>>> feature
```

- 字段解释：
  - `<<<<<<< HEAD` 到 `=======` 之间：**当前分支（main）**的内容。
  - `=======` 到 `>>>>>>> feature` 之间：**被合并分支（feature）**的内容。

### 第 6 步：手动解决冲突

把 `file.txt` 改成你想要保留的最终内容，例如：

```bash
echo "最终保留：两边合并后的正确内容" > file.txt
git add file.txt
git commit -m "fix: 解决 file.txt 冲突"
```

- 作用：`git add` 告诉 Git"冲突已解决"，之后 commit 完成合并。
- 原理：冲突没有自动答案，必须人判断；解决后合并提交会把两条历史"缝"在一起。

### 第 7 步：验证

```bash
git log --oneline --graph --all
# 预期：能看到两条岔路和一个合并提交
```

### 中止合并（如果解决不了想反悔）

```bash
git merge --abort     # 放弃本次合并，回到合并前状态
```

## 2.6 .gitignore（忽略不需要版本管理的文件）

**是什么：** 一个叫 `.gitignore` 的文本文件，告诉 Git 哪些文件/目录**不要**跟踪。

**为什么用：** 编译产物、依赖目录、密钥、日志、IDE 配置不该进仓库（又大又泄密）。

**怎么用：**

```bash
# 在仓库根目录创建 .gitignore
cat > .gitignore <<'EOF'
# 编译产物
target/
dist/
build/
*.class
*.jar

# 依赖目录
node_modules/
__pycache__/

# 密钥与本地配置（千万别提交！）
*.key
.env
application-local.yml

# 日志
*.log

# IDE
.idea/
.vscode/
EOF
```

- 规则解释：
  - `target/`：忽略整个目录（含子文件）。
  - `*.class`：`*` 通配符，忽略所有 `.class` 文件。
  - `!` 开头表示"反选"（不忽略），本示例未用。
  - 以 `#` 开头是注释。
- 作用：`git add .` 时会自动跳过这些文件。
- 注意：**已被跟踪的文件**即使加进 `.gitignore` 也不会被忽略，需先 `git rm --cached <文件>` 解除跟踪。

## 2.7 动手练习

1. 初始化一个仓库，完成 add → commit → 建分支 → 改代码 → 合并的完整流程。
2. 故意在 `file.txt` 制造一次冲突，手动解决并提交，观察 `git log --graph` 的分支图。
3. 写一个包含 `node_modules/`、`*.log`、`.env` 的 `.gitignore`，验证 `git add .` 后它们没被跟踪。

---

# 第 3 章 GitHub 关键操作（简述）

GitHub 是**托管 Git 仓库的云端平台**，核心操作（注册、SSH Key 配置、创建仓库、push/pull、Pull Request、Fork、Actions）已在配套文档 **《GitHub使用完整教程.md》** 中详细展开，本阶段不重复。

这里只补充一条与 CI/CD 强相关的要点：

- **Webhook（网络钩子）**：在 GitHub 仓库的 `Settings → Webhooks` 里配置一个 URL（通常是 Jenkins 的触发地址）。每次 push，GitHub 会向这个 URL 发送一个 POST 请求，Jenkins 收到后自动启动流水线。这是 CI/CD 的"触发"环节。

```text
你在本地 git push → GitHub 收到新提交 → GitHub 调用 Webhook → Jenkins 被唤醒 → 开始构建
```

> 下一章 GitLab 的 MR（Merge Request）流程与 GitHub 的 PR 逻辑完全一致，学会了 GitLab，GitHub 也就通了。

---

# 第 4 章 GitLab（自建代码托管平台）

## 4.1 GitLab 是什么、为什么用（三层）

- **是什么：** GitLab 是**可自建（私有部署）**的 Git 仓库管理平台，功能对标 GitHub，但代码数据完全在自己服务器上。
- **为什么用：** 企业内部代码不能放公网 GitHub，需要私有化部署、权限管控、内置 CI/CD。
- **怎么用：** 下面用 `dnf` 在 Rocky/CentOS 上安装。

### 4.1.1 安装依赖与添加仓库

```bash
# 1. 安装必要依赖
sudo dnf install -y curl policycoreutils openssh-server perl

# 2. 启动并开机自启 SSH（GitLab 走 SSH 协议需要）
sudo systemctl enable --now sshd

# 3. 关闭防火墙或放行端口（演示环境可直接关闭）
sudo systemctl stop firewalld && sudo systemctl disable firewalld

# 4. 添加 GitLab 官方仓库并安装（RPM 脚本方式，最省事）
curl -sS https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.rpm.sh | sudo bash
sudo dnf install -y gitlab-ce
```

- 参数解释：
  - `--now`：立即启动并同时设置开机自启。
  - `curl ... | sudo bash`：官方脚本会检测系统版本并写入 `/etc/yum.repos.d/gitlab_gitlab-ce.repo`。
- 作用：装好 GitLab Community Edition（社区版）。

### 4.1.2 配置访问地址（重要）

```bash
# 编辑主配置，把 external_url 改成你自己的域名或 IP
sudo vi /etc/gitlab/gitlab.rb

# 找到并改成（例如你的服务器 IP 是 192.168.1.100）：
# external_url 'http://192.168.1.100'
```

- 原理：GitLab 默认绑 `gitlab.example.com`，不改成真实地址会无法访问。

### 4.1.3 初始化（首次启动，耗时数分钟）

```bash
# 应用配置并启动所有组件
sudo gitlab-ctl reconfigure

# 查看服务状态
sudo gitlab-ctl status
# 预期输出：run: gitaly, postgresql, puma, sidekiq 等都显示 run
```

- 命令解释：
  - `gitlab-ctl reconfigure`：根据 `gitlab.rb` 重新生成并应用所有配置，首次会初始化数据库。
  - `gitlab-ctl status`：查看各组件是否在运行。
- 作用：首次执行时间较长（5~15 分钟），耐心等待，别中途 Ctrl+C。

## 4.2 登录并修改初始密码

1. 浏览器访问 `http://你的IP`，首次会要求设置 **root 用户密码**。
2. 设置一个强密码后，用 `root` 账号登录。

命令行查看/改密码（可选，备用）：

```bash
# 进入 GitLab 控制台
sudo gitlab-rails console -e production

# 在控制台内执行（输入后回车）：
user = User.find_by(username: 'root')
user.password = '你的新密码'
user.password_confirmation = '你的新密码'
user.save!

# 退出控制台
quit
```

- 字段解释：`username: 'root'` 是内置管理员；`password` 与 `password_confirmation` 必须一致。
- 作用：忘记密码时命令行重置。

## 4.3 创建组（Group）与用户（User）

### 4.3.1 创建组

**界面操作：** 左侧菜单 `Groups → New group → Create group`，填：

| 字段 | 示例值 | 说明 |
| --- | --- | --- |
| Group name | devops-team | 组名，URL 会用 |
| Visibility level | Private | Private=仅组成员可见；Internal=登录可见；Public=所有人可见 |

**作用：** 组用来管理一批项目和一组成员，权限统一控制。

### 4.3.2 创建用户

**界面操作：** 左侧菜单 `Admin Area → Users → New user`（需要 root 权限），填用户名、邮箱，勾选 "Can create group" 等权限后保存。

**作用：** 为团队每个成员开账号，加入组即可共享项目权限。

### 4.3.3 把用户加入组并授权

**界面操作：** 进入组 → `Members → Invite members`，选择用户，角色选 `Developer`。

**角色权限表：**

| 角色 | 权限要点 |
| --- | --- |
| Guest | 只能看 Issue |
| Reporter | 只读仓库 |
| Developer | 可 push、可创建分支、可提 MR |
| Maintainer | 可合并 MR、管理仓库设置 |
| Owner | 组拥有者，最高权限 |

- 作用：`Developer` 是日常开发角色，能提交代码但需 Maintainer 审核合并。

## 4.4 创建项目仓库

**界面操作：** `New project → Create blank project`，填：

| 字段 | 示例值 | 说明 |
| --- | --- | --- |
| Project name | hello-cicd | 项目名 |
| Project URL | devops-team/hello-cicd | 归属哪个组 |
| Initialize with a README | 勾选 | 初始化一个 README，便于立刻 clone |

**把代码推上去：**

```bash
# 克隆空仓库到本地
git clone http://192.168.1.100/devops-team/hello-cicd.git
cd hello-cicd

# 写点代码
echo "hello cicd" > README.md
git add README.md
git commit -m "init: 第一个提交"
git push -u origin main
```

- 作用：现在你的代码已经在自己的 GitLab 上了。

## 4.5 Merge Request（MR）流程

**MR 是什么：** 合并请求，等于"我改好了，请审核并合并到主干"。这是团队协作的核心流程。

### 完整 MR 流程演示

```bash
# 1. 从 main 切出功能分支
git switch main && git pull
git switch -c feature-login

# 2. 改代码并提交
echo "登录功能" > login.py
git add login.py
git commit -m "feat: 登录功能"
git push -u origin feature-login
```

**界面操作（关键步骤）：**

1. 在 GitLab 页面会看到提示 "Create merge request"，点它。
2. 填写：
   - **Title**：`feat: 登录功能`（说明这个 MR 做什么）。
   - **Description**：写清楚改动内容、测试方法。
   - **Source branch**：`feature-login`；**Target branch**：`main`。
   - **Assignee / Reviewer**：指定审核人。
3. 点 `Create merge request`。
4. 审核人进入 MR，点 `Changes` 看代码 diff，评论或点 `Approve`。
5. 通过后，维护者点 `Merge`，选择 `Merge commit`（保留完整历史）或 `Squash`（压缩成一条）。
6. 合并后删除源分支：勾选 `Delete source branch`。

**MR 与直接 push 的区别：**

| 方式 | 谁能改 main | 是否有审核 | 风险 |
| --- | --- | --- | --- |
| 直接 push main | 有权限的人 | 无 | 高，容易带坏主干 |
| MR 流程 | 只有审核通过才合并 | 有 | 低，每次改动可评审 |

- 原理：MR 强制"先审后合"，是团队质量的最后一道闸门。

## 4.6 动手练习

1. 在 GitLab 创建组 `devops-team` 和用户 `tom`，把 tom 加为 `Developer`。
2. 创建项目 `hello-cicd`，clone 到本地，push 一个 `main` 提交。
3. 切 `feature-login` 分支改代码 push，走一遍完整 MR 并合并。

---

# 第 5 章 Jenkins（自动化构建引擎）

## 5.1 Jenkins 是什么、为什么用（三层）

- **是什么：** Jenkins 是开源的**持续集成/交付工具**，负责按流水线自动执行"拉代码、构建、测试、部署"。
- **为什么用：** 它是 CI/CD 的"发动机"，GitLab/GitHub 负责存代码，Jenkins 负责自动干活。
- **怎么用：** 装好 Jenkins，写流水线（Job / Pipeline），配 Webhook 触发。

### 5.1.1 安装 JDK（Jenkins 需要 Java）

```bash
# Rocky/CentOS 安装 Java 17（Jenkins 2.361+ 要求 Java 11 或 17）
sudo dnf install -y java-17-openjdk java-17-openjdk-devel

# 验证
java -version
# 预期输出类似：openjdk version "17.0.x"
```

### 5.1.2 方式一：yum 安装 Jenkins（推荐）

```bash
# 1. 添加 Jenkins 稳定版仓库
sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key

# 2. 安装
sudo dnf install -y jenkins

# 3. 启动并设置开机自启
sudo systemctl enable --now jenkins

# 4. 查看状态
sudo systemctl status jenkins
# 预期：active (running)
```

- 参数解释：
  - `-O`：wget 指定输出文件名。
  - `rpm --import`：导入 GPG 密钥，验证包来源可信。
- 作用：以系统服务方式运行，默认监听端口 **8080**。

### 5.1.3 方式二：war 包运行（无需 root、灵活）

```bash
# 下载 war 包
wget https://get.jenkins.io/war-stable/latest/jenkins.war

# 直接运行（前台，Ctrl+C 停止）
java -jar jenkins.war --httpPort=8080

# 或后台运行
nohup java -jar jenkins.war --httpPort=8080 > jenkins.log 2>&1 &
```

- 参数解释：
  - `--httpPort=8080`：指定监听端口。
  - `nohup ... &`：后台运行，日志写到 `jenkins.log`。
- 作用：war 方式适合临时体验或没有 root 权限的场景。

## 5.2 初始化（首次进入）

1. 浏览器访问 `http://你的IP:8080`。
2. 页面提示输入初始密码，命令行获取：

```bash
# yum 安装的路径
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
# 复制输出的那串密码，粘贴到页面
```

3. 选择 `Install suggested plugins`（安装推荐插件，覆盖大部分场景）。
4. 创建第一个管理员账号（用户名/密码/邮箱）。
5. 设置 Jenkins URL（默认 `http://IP:8080`），完成。

- 原理：初始密码是临时密钥，防止别人在你未配置前抢占 Jenkins。

## 5.3 插件管理

**插件是什么：** Jenkins 本身是骨架，功能靠插件扩展（Git 集成、Docker、Pipeline、SSH 全靠插件）。

**安装插件（界面操作）：** `Manage Jenkins → Plugins → Available plugins`，搜索并勾选安装。本教程需要以下插件：

| 插件名 | 作用 |
| --- | --- |
| Git | 拉取 Git 仓库代码 |
| Pipeline | 声明式流水线核心 |
| Docker Pipeline | 流水线里构建/推送 Docker 镜像 |
| Publish Over SSH | SSH 远程部署 |
| Credentials Binding | 把凭据安全注入流水线 |
| Blue Ocean（可选） | 更美观的流水线可视化 |

**离线/命令行安装（备选）：**

```bash
# 通过 Jenkins CLI 安装（需要先装 cli jar）
java -jar jenkins-cli.jar -s http://localhost:8080 install-plugin git pipeline docker-workflow
```

## 5.4 凭据（Credentials）配置

**为什么需要凭据：** 流水线要访问 GitLab（拉代码）、Docker Hub（推镜像）、服务器（SSH 部署），这些密码不能写死在脚本里，要存到 Jenkins 凭据库。

**添加凭据（界面操作）：** `Manage Jenkins → Credentials → System → Global credentials → Add Credentials`。

本教程需要三类凭据，配置如下表：

| Kind | 字段 | 值示例 | 用途 |
| --- | --- | --- | --- |
| Username with password | Username / Password | GitLab 账号 / 密码或 token | 拉 GitLab 代码 |
| Username with password | Username / Password | Docker Hub 账号 / 密码 | 推镜像 |
| SSH Username with private key | Username / Private Key | root / 私钥内容 | SSH 部署到服务器 |

- **ID 字段很重要**：添加时填一个 ID（如 `gitlab-cred`、`dockerhub-cred`、`server-ssh`），流水线里靠这个 ID 引用。
- 原理：凭据被 Jenkins 加密存储，流水线通过 ID 调用，脚本里看不到明文。

## 5.5 创建第一个 Job（自由风格，先跑通）

1. `New Item → 输入名字 hello-world → 选 Freestyle project → OK`。
2. 在 `Build Steps` 里选 `Execute shell`，输入：

```bash
echo "Jenkins 第一次构建成功！"
date
echo "当前工作目录: $(pwd)"
```

3. 保存后点 `Build Now`，构建完成后点构建编号 → `Console Output` 查看输出。

- 字段解释：`Freestyle project` 是"图形化配置"的 Job，适合入门；后面会用更强大的 Pipeline。
- 作用：验证 Jenkins 能正常执行构建，这是你第一条"流水线"（虽然只有一步）。

## 5.6 Pipeline 声明式语法（重点）

**Pipeline 是什么：** 把流水线写成代码（Jenkinsfile），存进仓库，和业务代码一起版本管理。这是现代 Jenkins 的主流方式。

**声明式 vs 脚本式对比：**

| 对比项 | 声明式（Declarative） | 脚本式（Scripted） |
| --- | --- | --- |
| 结构 | 固定骨架（pipeline/stages/steps） | 自由 Groovy 代码 |
| 上手难度 | 简单、可读性强 | 灵活但复杂 |
| 推荐 | ✅ 生产推荐 | 高级场景用 |

**最小可运行的声明式 Jenkinsfile：**

```groovy
pipeline {
    agent any                    // 在任意可用节点上运行

    stages {                     // 阶段集合
        stage('拉代码') {         // 一个阶段
            steps {
                echo '模拟 git clone'
            }
        }
        stage('构建') {
            steps {
                sh 'echo "正在构建..."'
            }
        }
        stage('测试') {
            steps {
                sh 'echo "正在测试..."'
            }
        }
    }

    post {                        // 收尾：无论成败都执行
        always {
            echo '本次流水线结束'
        }
    }
}
```

**语法字段逐项解释：**

| 关键字 | 含义 |
| --- | --- |
| `pipeline` | 根节点，表示这是一条流水线 |
| `agent any` | 在哪台机器上跑（any=任意节点，可写 `agent { label 'docker' }`） |
| `stages` | 包含多个 `stage` 的容器 |
| `stage('名字')` | 一个阶段，界面上显示为一个方块 |
| `steps` | 阶段里具体执行的命令 |
| `sh '...'` | 执行 shell 命令 |
| `echo '...'` | 输出日志 |
| `post` | 收尾钩子，`always`=总是执行，`success`=成功时，`failure`=失败时 |
| `environment` | 定义环境变量（后面会用到） |
| `credentials()` | 从凭据库取凭据注入 |

## 5.7 与 Git 集成 + 自动触发

### 5.7.1 创建 Pipeline Job 关联 Git 仓库

1. `New Item → 名字 cicd-demo → 选 Pipeline → OK`。
2. 在 `Pipeline` 配置区：
   - `Definition` 选 `Pipeline script from SCM`。
   - `SCM` 选 `Git`。
   - `Repository URL` 填 `http://192.168.1.100/devops-team/hello-cicd.git`。
   - `Credentials` 选之前建的 `gitlab-cred`。
   - `Branches to build` 填 `*/main`。
   - `Script Path` 填 `Jenkinsfile`（仓库根目录的流水线文件）。
3. 保存，点 `Build Now` 测试能否拉取。

- 字段解释：
  - `Pipeline script from SCM`：流水线代码来自仓库（推荐），而不是在 Jenkins 里手写。
  - `Script Path`：告诉 Jenkins 去仓库里找哪个文件当流水线。
- 作用：以后改流水线只需改仓库里的 Jenkinsfile，push 即生效。

### 5.7.2 配置 GitLab Webhook 自动触发

1. 在 Jenkins 安装插件 `GitLab`（或 `Generic Webhook Trigger`）。
2. Pipeline Job 勾选 `Build when a change is pushed to GitLab`，记下给出的 **URL 和 Secret Token**。
3. 去 GitLab：`项目 → Settings → Webhooks`：
   - URL 填 Jenkins 给的触发地址。
   - Secret Token 填一致。
   - 勾选 `Push events`。
   - 点 `Add webhook`，再点 `Test → Push events` 测试。

- 原理：GitLab 收到 push → POST 到 Jenkins → Jenkins 校验 token → 启动构建，实现"代码一推，自动构建"。

## 5.8 动手练习

1. 安装 JDK + Jenkins（yum 或 war），完成初始化并创建管理员账号。
2. 安装 Git / Pipeline / Docker Pipeline / Publish Over SSH 插件。
3. 创建一个自由风格 Job，执行 `date && echo hello`，查看 Console Output。
4. 创建一个 Pipeline Job，从 GitLab 拉取包含上面"最小 Jenkinsfile"的仓库并构建成功。

---

# 第 6 章 完整 CI/CD 项目实战

> 目标流水线：**代码 push 到 GitLab → Jenkins 自动拉取 → 构建 Docker 镜像 → push 到镜像仓库 → 自动部署到服务器(SSH)（或 K8s）。**

## 6.1 整体架构图

```text
开发机 ──git push──▶ GitLab ──Webhook──▶ Jenkins ──▶ 拉代码/构建/测试
                                               │
                                               ├─▶ docker build ──▶ docker push ──▶ 镜像仓库(Harbor/DockerHub)
                                               │
                                               └─▶ SSH 登录部署机 ──▶ docker pull + run 新镜像（或 kubectl 更新 K8s）
```

**需要 3~4 台机器（演示可用一台机器装多个角色）：**

| 角色 | 软件 | 说明 |
| --- | --- | --- |
| GitLab 服务器 | GitLab | 存代码、发 Webhook |
| Jenkins 服务器 | Jenkins + Docker | 执行流水线、构建镜像 |
| 部署服务器 | Docker（+SSH） | 跑最终应用 |
| 镜像仓库 | Harbor / Docker Hub | 中转镜像（可选，生产必备） |

## 6.2 准备一个简单 Web 应用

在 `hello-cicd` 仓库里创建如下文件结构：

```text
hello-cicd/
├── app.py          # Python Web 应用
├── Dockerfile      # 构建镜像用
├── Jenkinsfile     # 流水线定义
└── requirements.txt
```

**app.py（一个极简 Flask 应用）：**

```python
from flask import Flask
app = Flask(__name__)

@app.route('/')
def index():
    return "<h1>Hello CI/CD! 这是第 1 版</h1>"

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

**requirements.txt：**

```text
flask
```

**Dockerfile：**

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY app.py .
EXPOSE 5000
CMD ["python", "app.py"]
```

- 字段解释：
  - `FROM`：基础镜像。
  - `WORKDIR`：容器内工作目录。
  - `COPY`：把宿主机文件拷进镜像。
  - `RUN`：构建镜像时执行。
  - `EXPOSE`：声明容器监听端口。
  - `CMD`：容器启动时执行的命令。

## 6.3 部署机与 Jenkins 机安装 Docker

```bash
# 两台机器都执行
sudo dnf install -y dnf-utils
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo dnf install -y docker-ce docker-ce-cli containerd.io
sudo systemctl enable --now docker

# 验证
docker version
```

- 作用：Jenkins 机需要 Docker 来 `build/push` 镜像；部署机需要 Docker 来 `pull/run` 镜像。

## 6.4 配置 Jenkins 机免密 SSH 到部署机（关键）

Jenkins 要自动登录部署机，必须配置**免密 SSH**。

```bash
# 在 Jenkins 机器上，切换到 jenkins 用户（Jenkins 用这个身份执行流水线）
sudo su - jenkins -s /bin/bash

# 生成密钥对（一路回车，不设密码）
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa -N ""

# 查看公钥
cat ~/.ssh/id_rsa.pub
```

- 参数解释：
  - `-t rsa`：密钥类型 RSA。
  - `-b 4096`：密钥长度 4096 位。
  - `-f ~/.ssh/id_rsa`：输出文件路径。
  - `-N ""`：不设私钥口令（自动化必须无口令）。

**在部署机上，把公钥加入授权列表：**

```bash
# 在部署机执行，把上一步的公钥内容追加进去
mkdir -p ~/.ssh && chmod 700 ~/.ssh
echo "粘贴 jenkins 机器的 id_rsa.pub 内容" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

**验证免密登录：**

```bash
# 回到 Jenkins 机器，用 jenkins 用户测试
ssh root@192.168.1.200 "echo 免密成功"
# 预期输出：免密成功（不再要求输入密码）
```

- 原理：SSH 用公钥认证替代密码认证，Jenkins 才能在无人工干预下登录部署机。

## 6.5 在 Jenkins 配置三类凭据（复用 5.4）

1. **gitlab-cred**（Username with password）：拉代码。
2. **dockerhub-cred**（Username with password）：推镜像。
3. **server-ssh**（SSH Username with private key）：把上一步 `~/.ssh/id_rsa` 的**私钥内容**粘贴进 Private Key 框，ID 填 `server-ssh`，Username 填 `root`。

## 6.6 编写完整 Jenkinsfile（核心）

在仓库根目录创建 `Jenkinsfile`：

```groovy
pipeline {
    agent any

    // 环境变量：镜像仓库、部署机地址，凭据 ID
    environment {
        DOCKER_REGISTRY = 'docker.io'                 // 镜像仓库（生产可换成 Harbor 地址）
        IMAGE_NAME     = 'zhangsan/hello-cicd'        // 镜像名（用户名/仓库名）
        IMAGE_TAG      = "v${BUILD_NUMBER}"           // 用构建号当版本，如 v1 v2 v3
        DEPLOY_HOST    = '192.168.1.200'              // 部署机 IP
        DEPLOY_USER    = 'root'                       // 部署机用户名
    }

    stages {

        stage('1. 拉取代码') {
            steps {
                // checkout 由 Job 里的 SCM 配置负责，这里只是确认
                echo "当前构建号: ${BUILD_NUMBER}, 分支: ${env.GIT_BRANCH}"
            }
        }

        stage('2. 构建 & 测试') {
            steps {
                sh 'python3 -m pip install -r requirements.txt'
                sh 'python3 -m compileall app.py'     // 语法检查，模拟"测试"
                echo '构建与测试通过'
            }
        }

        stage('3. 构建 Docker 镜像') {
            steps {
                sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."
                sh "docker tag  ${IMAGE_NAME}:${IMAGE_TAG} ${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}"
            }
        }

        stage('4. 推送镜像到仓库') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-cred',
                    usernameVariable: 'DH_USER',
                    passwordVariable: 'DH_PASS'
                )]) {
                    sh "echo $DH_PASS | docker login -u $DH_USER --password-stdin ${DOCKER_REGISTRY}"
                }
                sh "docker push ${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}"
            }
        }

        stage('5. SSH 部署到服务器') {
            steps {
                withCredentials([sshUserPrivateKey(
                    credentialsId: 'server-ssh',
                    usernameVariable: 'SSH_USER',
                    keyFileVariable: 'SSH_KEY'
                )]) {
                    // 用私钥登录部署机：拉新镜像、停旧容器、起新容器
                    sh """
                        ssh -i $SSH_KEY -o StrictHostKeyChecking=no $SSH_USER@$DEPLOY_HOST '''
                            docker login -u 用户名 -p 密码 ${DOCKER_REGISTRY} || true
                            docker pull ${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}
                            docker stop hello-cicd || true
                            docker rm hello-cicd || true
                            docker run -d --name hello-cicd -p 5000:5000 ${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}
                        '''
                    """
                }
            }
        }
    }

    post {
        success { echo '✅ 部署成功，可访问 http://部署机:5000' }
        failure { echo '❌ 流水线失败，请查看日志' }
        always  { sh 'docker logout ${DOCKER_REGISTRY} || true' }
    }
}
```

**逐段解释：**

| 代码 | 作用 / 原理 |
| --- | --- |
| `environment { }` | 定义流水线级变量，后面用 `${变量}` 引用 |
| `IMAGE_TAG = "v${BUILD_NUMBER}"` | `BUILD_NUMBER` 是 Jenkins 内置变量，每次构建 +1，保证镜像版本唯一、可回滚 |
| `withCredentials([usernamePassword(...)])` | 从凭据库取账号密码注入成临时变量 `DH_USER`/`DH_PASS`，不落明文 |
| `--password-stdin` | 密码从标准输入读，避免 `docker login -p 密码` 明文进历史记录 |
| `withCredentials([sshUserPrivateKey(...)])` | 取私钥注入，`$SSH_KEY` 是临时私钥文件路径 |
| `-o StrictHostKeyChecking=no` | 首次连接不提示"是否信任主机"，自动化必须加 |
| `docker stop/rm \|\| true` | 旧容器不存在时报错也不中断（`\|\| true` 兜底） |

## 6.7 配置 Webhook 完成自动触发（复用 5.7.2）

在 GitLab 项目 Webhook 填 Jenkins 触发地址 + token，勾 `Push events`。

**端到端验证：**

```bash
# 本地改一行代码，触发整条流水线
sed -i 's/第 1 版/第 2 版/' app.py
git add app.py
git commit -m "feat: 升级到第 2 版"
git push origin main
```

**预期结果（无需任何人工操作）：**

1. GitLab 收到 push → 通知 Jenkins。
2. Jenkins 自动拉代码 → 构建 → 打镜像 `vXX` → 推 Docker Hub → SSH 部署。
3. 浏览器访问 `http://部署机:5000` 显示 "Hello CI/CD! 这是第 2 版"。

## 6.8 部署到 Kubernetes（K8s，进阶替换方案）

如果目标环境是 K8s，只需把第 5 阶段的 SSH 换成 `kubectl`：

```groovy
stage('5. 部署到 K8s') {
    steps {
        // 用 sed 替换 YAML 里的镜像版本，再 apply
        sh """
            sed -i "s|image:.*|image: ${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}|" deploy.yaml
            kubectl apply -f deploy.yaml
            kubectl rollout status deployment/hello-cicd
        """
    }
}
```

- 字段解释：
  - `sed -i "s|image:.*|..."`：原地替换 `deploy.yaml` 里的 image 行为新版本。
  - `kubectl apply -f`：声明式更新部署。
  - `kubectl rollout status`：等待滚动更新完成，失败会非零退出使流水线标红。
- 原理：K8s 用"声明式"更新，镜像 tag 变了就滚动升级，天然支持回滚（`kubectl rollout undo`）。

## 6.9 动手练习

1. 按 6.2 建好 4 个文件，推到 GitLab。
2. 完成 Docker 安装、免密 SSH、三类凭据配置。
3. 把 Jenkinsfile 提交到仓库，Pipeline Job 指向它，手动 `Build Now` 跑通。
4. 配好 Webhook，改代码 push，观察是否**全自动**完成"构建 + 推镜像 + 部署"。

---

# 第 7 章 本章项目验收（从零到全自动上线）

> 按下面步骤**从零**操作一遍，能全自动跑通即算通过。每步都标注了"验证点"。

## 7.1 验收步骤清单

### 步骤 1：搭 GitLab 并建仓库

```bash
# GitLab 服务器
sudo dnf install -y curl policycoreutils openssh-server perl
sudo systemctl enable --now sshd
sudo systemctl stop firewalld; sudo systemctl disable firewalld
curl -sS https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.rpm.sh | sudo bash
sudo dnf install -y gitlab-ce
sudo sed -i "s|external_url '.*'|external_url 'http://192.168.1.100'|" /etc/gitlab/gitlab.rb
sudo gitlab-ctl reconfigure
```

- **验证点：** 浏览器打开 `http://192.168.1.100` 能设置 root 密码并登录。
- 建项目 `hello-cicd`，得到 clone 地址 `http://192.168.1.100/root/hello-cicd.git`。

### 步骤 2：提交应用代码到 GitLab

```bash
mkdir hello-cicd && cd hello-cicd
# 创建 app.py / Dockerfile / requirements.txt / Jenkinsfile（内容见第 6 章）
git init
git remote add origin http://192.168.1.100/root/hello-cicd.git
git add .
git commit -m "init: 完整 CI/CD 项目"
git push -u origin main
```

- **验证点：** GitLab 页面上能看到 4 个文件。

### 步骤 3：搭 Jenkins 并配置

```bash
# Jenkins 服务器
sudo dnf install -y java-17-openjdk java-17-openjdk-devel
sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
sudo dnf install -y jenkins
sudo systemctl enable --now jenkins
sudo cat /var/lib/jenkins/secrets/initialAdminPassword   # 记下密码
```

- **验证点：** 打开 `http://192.168.1.101:8080`，完成初始化，安装 Git/Pipeline/Docker Pipeline/SSH 相关插件。

### 步骤 4：配置凭据与免密 SSH

- 添加 `gitlab-cred`、`dockerhub-cred`、`server-ssh` 三类凭据（见 5.4）。
- 生成 jenkins 用户密钥，公钥拷到部署机 `authorized_keys`（见 6.4）。
- **验证点：** `sudo su - jenkins -s /bin/bash -c "ssh root@部署机 'echo ok'"` 输出 `ok`。

### 步骤 5：创建 Pipeline Job 指向 Jenkinsfile

- `New Item → cicd-demo → Pipeline`。
- SCM 选 Git，URL 填 GitLab 地址，凭据选 `gitlab-cred`，`Script Path = Jenkinsfile`。
- **验证点：** 手动 `Build Now`，蓝色（成功）通过全部 5 个阶段。

### 步骤 6：配置 Webhook 自动触发

- GitLab 项目 `Settings → Webhooks → 填 Jenkins 触发地址 + token → 勾 Push events`。
- **验证点：** 点 `Test → Push events` 返回 HTTP 200。

### 步骤 7：终极验收——改一次代码，全自动上线

```bash
# 本地改代码
sed -i 's/第 1 版/验收通过版/' app.py
git add app.py
git commit -m "feat: 验收测试"
git push origin main
```

**验收成功标准（全部满足才算通过）：**

1. ✅ 无需手动点 Jenkins，push 后**自动**触发构建。
2. ✅ Jenkins 控制台能看到 `拉取 → 构建测试 → 打镜像 → 推镜像 → 部署` 5 个阶段全绿。
3. ✅ Docker Hub 上出现新 tag 的镜像（如 `v7`）。
4. ✅ 浏览器访问 `http://部署机:5000` 显示 "Hello CI/CD! 这是 验收通过版"。
5. ✅ 再执行 `git revert` 回滚一次，流水线应能重新部署回旧版本（验证可回滚）。

## 7.2 常见问题排查表

| 症状 | 可能原因 | 解决 |
| --- | --- | --- |
| Webhook 不触发 | token 不一致 / URL 写错 / 网络不通 | 核对 token，`curl` 测试 Jenkins 地址 |
| 拉代码失败 | 凭据错误 / 仓库地址不对 | 用 `git clone` 手动验证账号密码 |
| docker login 失败 | 账号密码错误 / 网络 | 手动 `docker login` 验证 |
| SSH 部署失败 | 免密未配好 / 私钥填错 | 用 jenkins 用户手动 `ssh root@IP` 验证 |
| 构建蓝色但容器没起 | 端口被占 / 镜像拉取慢 | 部署机 `docker logs hello-cicd` 看日志 |
| Pipeline 报 `Permission denied` | jenkins 用户不在 docker 组 | `sudo usermod -aG docker jenkins && sudo systemctl restart jenkins` |

## 7.3 验收总结

完成以上 7 步，你就拥有了一条**生产级最小闭环 CI/CD 流水线**：

```text
写代码 → git push → GitLab 自动通知 → Jenkins 拉取/构建/测试 → Docker 打镜像
       → 推镜像仓库 → SSH 自动部署 → 浏览器看到新版本
```

这条流水线把"人肉上线"变成了"代码上线"，是进入 DevOps、云原生、K8s 世界的必经之路。
