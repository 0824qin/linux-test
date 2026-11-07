# 阶段 1：Linux 系统基础详细教程（零基础 · Rocky Linux 9 / AlmaLinux 9）

> 本教程面向完全没有 Linux 经验的读者，以 **Rocky Linux 9 / AlmaLinux 9**（二者均为 RHEL 9 的免费替代品，命令完全一致）为主。
> 文中每一处命令都会给出「可运行示例 + 参数/字段解释 + 作用原理」，照抄即可跑通。
> 约定：以 `#` 开头的命令需要用 `root` 身份执行；以 `$` 开头的命令普通用户即可执行；`# 注释` 表示后面的文字是解释，不用敲。

---

## 第 1 章 目录结构 FHS（Filesystem Hierarchy Standard）

### 1.1 是什么、为什么用

- **是什么**：FHS 是一套约定，规定 Linux 各目录应该放什么类型的文件，让任何发行版的系统管理员都能"猜到"文件在哪。
- **为什么用**：程序、用户、日志、临时文件如果乱放，系统会难以维护、难以排查、难以备份。统一目录让自动化脚本和运维规范有据可依。
- **怎么用**：拿到一台陌生机器，先按 FHS 去对应目录找东西即可。

### 1.2 核心目录逐一详解

先执行一条命令，看看根目录下有哪些东西：

```bash
$ ls -l /
total 28
lrwxrwxrwx   1 root root    7 Jul 20 10:00 bin -> usr/bin
dr-xr-xr-x.  5 root root 4096 Jul 20 10:00 boot
drwxr-xr-x  20 root root 4400 Jul 20 10:00 dev
drwxr-xr-x. 95 root root 8192 Jul 20 10:00 etc
drwxr-xr-x.  3 root root   22 Jul 20 10:00 home
lrwxrwxrwx   1 root root    7 Jul 20 10:00 lib -> usr/lib
drwxr-xr-x   2 root root    6 Jul 20 10:00 media
drwxr-xr-x   2 root root    6 Jul 20 10:00 mnt
drwxr-xr-x.  2 root root    6 Jul 20 10:00 opt
dr-xr-xr-x 200 root root    0 Jul 20 10:00 proc
drwx------   5 root root 4096 Jul 20 10:00 root
drwxr-xr-x  21 root root  620 Jul 20 10:00 run
lrwxrwxrwx   1 root root    8 Jul 20 10:00 sbin -> usr/sbin
drwxr-xr-x   2 root root    6 Jul 20 10:00 srv
dr-xr-xr-x  13 root root    0 Jul 20 10:00 sys
drwxrwxrwt   8 root root 4096 Jul 20 10:00 tmp
drwxr-xr-x. 13 root root  155 Jul 20 10:00 usr
drwxr-xr-x. 20 root root 4096 Jul 20 10:00 var
```

**说明**：第一列开头的 `l` 表示软链接（符号链接），`d` 表示目录，`-` 表示普通文件。现代 Linux 把 `bin`、`sbin`、`lib` 都合并进了 `usr`，所以看到 `bin -> usr/bin` 这种箭头链接是正常的。

下表是每个目录的"干什么 + 例子"：

| 目录 | 存放什么 | 作用/原理 | 示例文件 |
|------|----------|-----------|----------|
| `/etc` | 系统与服务的配置文件 | 几乎所有软件配置都在这，改配置先来这里 | `/etc/ssh/sshd_config`（SSH 配置）、`/etc/fstab`（开机挂载）、`/etc/passwd`（用户） |
| `/var` | 会持续变化的数据 | variable（可变）之意：日志、缓存、锁、邮件队列，会随运行不断增长 | `/var/log/messages`、`/var/log/nginx/` |
| `/usr` | 用户级程序与库（只读） | 相当于 Windows 的 Program Files，装完基本不再改 | `/usr/bin/ls`、`/usr/lib/`、`/usr/share/doc/` |
| `/home` | 普通用户的家目录 | 每个用户一个专属目录，存个人文件与个性化配置 | `/home/alice/`、`/home/alice/.bashrc` |
| `/opt` | 第三方可选软件 | 手动安装的独立软件包常放这，与系统自带程序隔离 | `/opt/google/chrome/`、`/opt/nginx/` |
| `/tmp` | 临时文件 | 任何人可写（粘滞位保护），重启通常清空 | `/tmp/xxx.tmp` |
| `/boot` | 启动所需文件 | 内核、initramfs、grub 引导配置，删错会导致无法开机 | `/boot/vmlinuz-5.14.0-...`、`/boot/grub2/` |
| `/proc` | 虚拟文件系统（内存态） | 内核与进程信息实时映射，不占磁盘，重启即消失 | `/proc/cpuinfo`、`/proc/meminfo`、`/proc/<PID>/` |
| `/sys` | 虚拟文件系统（设备内核态） | 内核导出设备/驱动/内核对象，用于查询和调整硬件 | `/sys/class/net/`、`/sys/block/sda/` |
| `/dev` | 设备文件 | 一切皆文件：硬盘、终端、键盘都以文件形式访问 | `/dev/sda`（磁盘）、`/dev/tty`、`/dev/null` |
| `/root` | root 用户的家目录 | 超级管理员专用，与普通用户的 `/home` 隔离 | `/root/.ssh/` |
| `/run` | 运行时数据（内存态） | 开机后到关机前进程产生的临时状态，如 PID 文件 | `/run/sshd.pid` |

### 1.3 用实例验证目录的"身份"

```bash
# 查看 CPU 型号（读 /proc 虚拟文件）
$ cat /proc/cpuinfo | grep "model name" | head -1
model name      : Intel(R) Xeon(R) CPU E5-2670 v2 @ 2.50GHz

# 查看内存总量（读 /proc/meminfo）
$ grep MemTotal /proc/meminfo
MemTotal:        3921000 kB

# 查看某进程的信息（/proc/<PID>）
$ ps -ef | grep sshd | grep -v grep
root        915       1  0 10:00 ?        00:00:00 sshd: /usr/sbin/sshd [listener]
$ ls /proc/915/            # 915 号进程的内核信息目录
```

**原理说明**：`/proc` 和 `/sys` 都是"伪文件系统"，内容是内核运行时实时生成的，你 `cat` 的时候内核才动态拼出内容。它们不占用磁盘空间，重启后目录内容重新生成。

### 1.4 动手练习

1. 用 `ls -l /` 观察根目录，找出所有是软链接（`l` 开头）的项。
2. 用 `cat /proc/meminfo` 查看本机内存总量，用 `grep -c processor /proc/cpuinfo` 数一下 CPU 逻辑核数。
3. 在 `/tmp` 下创建一个文件，说明为什么 `/tmp` 适合放临时文件。
4. 找出 `/etc` 下 3 个配置文件，说出它们大概是干什么的（例如 `hostname`、`fstab`、`passwd`）。

---

## 第 2 章 常用命令分类详解

> 每类命令都按「示例 → 解释 → 原理」展开。请务必在终端里逐条敲一遍。

### 2.1 文件与目录操作：`ls / cd / pwd / cp / mv / rm / mkdir / touch`

#### 2.1.1 `ls` 列出目录内容

```bash
$ ls                 # 列出当前目录（不显示隐藏文件）
$ ls -l              # 长格式：权限、链接数、属主、属组、大小、时间、名字
$ ls -a              # 显示全部，包括以 . 开头的隐藏文件
$ ls -lh             # -h 人类可读的大小（1.2K / 3.4M）
$ ls -lt             # 按修改时间排序，最新在前
$ ls -ld /etc        # -d 只显示目录本身，不展开内容
```

**参数解释**：`-l` long、`-a` all、`-h` human-readable、`-t` time 排序、`-d` directory。

**示例输出与字段解释**：

```bash
$ ls -lh /etc/passwd
-rw-r--r--. 1 root root 1.9K Jul 20 10:00 /etc/passwd
```

| 字段 | 含义 |
|------|------|
| `-rw-r--r--.` | 类型+权限（`-` 普通文件，随后 9 位权限），末尾 `.` 表示有 SELinux 上下文 |
| `1` | 硬链接数 |
| `root root` | 属主 / 属组 |
| `1.9K` | 大小 |
| `Jul 20 10:00` | 最后修改时间 |
| `/etc/passwd` | 文件名 |

#### 2.1.2 `cd / pwd` 切换与显示目录

```bash
$ pwd                # 打印当前目录，如 /home/alice
$ cd /etc            # 进入绝对路径 /etc
$ cd ssh             # 进入相对路径（相对当前目录）
$ cd ..              # 上一级
$ cd ~               # 回到自己的家目录（~ 等价于 /home/alice）
$ cd -               # 回到上一次所在目录
```

**原理**：绝对路径从 `/` 开始；相对路径从当前目录开始。`~` 是家目录缩写，`-` 记录上次目录。

#### 2.1.3 `cp` 复制

```bash
$ cp a.txt b.txt             # 复制文件并改名
$ cp a.txt /tmp/             # 复制到 /tmp，保持原名
$ cp -r dir1 dir2            # -r 递归复制整个目录
$ cp -p a.txt b.txt          # -p 保留权限、属主、时间戳
$ cp -i a.txt b.txt          # -i 覆盖前询问（避免误覆盖）
```

**原理**：`cp` 默认只复制文件，复制目录必须加 `-r`；不带 `-p` 时目标文件的属主/时间会变成当前用户和当前时间。

#### 2.1.4 `mv` 移动/重命名

```bash
$ mv old.txt new.txt         # 同一目录下 = 重命名
$ mv file.txt /opt/          # 移动到别的目录
$ mv dir1 dir2               # 移动目录（目录移动不需要 -r）
$ mv -i a.txt b.txt          # 覆盖前询问
```

**原理**：同一文件系统内 `mv` 只改目录项（很快）；跨文件系统则等价于"复制+删除"。所以 `mv` 也常被用来给文件改名。

#### 2.1.5 `rm` 删除

```bash
$ rm file.txt                # 删除文件（默认不询问，但别名常带 -i）
$ rm -f file.txt             # -f 强制，不提示
$ rm -r dir1                 # -r 递归删除目录及其内容
$ rm -rf /tmp/test           # -rf 组合：递归+强制（慎用！）
```

> **安全警告**：`rm -rf /` 或带空变量的 `rm -rf $DIR/` 可能删光整个系统。删除前务必 `pwd` 确认位置，绝对不要在生产机上随手敲 `rm -rf *`。

**原理**：`rm` 只把文件对应的 inode 标记为未使用，数据并未立即被物理擦除，所以误删后第一时间不要再写盘，恢复工具才有机会找回。

#### 2.1.6 `mkdir / touch` 建目录/建空文件

```bash
$ mkdir test                 # 建一个目录
$ mkdir -p a/b/c/d           # -p 递归创建多级目录，已存在也不报错
$ touch file.txt             # 文件存在则更新时间为现在，不存在则新建空文件
$ touch file{1,2,3}.txt      # 花括号展开，一次创建 file1/2/3.txt
```

**原理**：`mkdir -p` 是写脚本最常用的"确保目录存在"方式；`touch` 的展开写法 `file{1,2,3}.txt` 是 shell 的花括号展开特性。

### 2.2 文本查看：`cat / less / head / tail / tail -f`

#### 2.2.1 `cat` 输出整个文件

```bash
$ cat /etc/hostname        # 直接打印完整内容
$ cat -n /etc/passwd       # -n 显示行号
$ cat file1 file2          # 连接多个文件输出
```

**原理**：`cat` 把文件内容原样送到标准输出，适合看小文件；大文件会用它配合管道。

#### 2.2.2 `less` 分页查看（大文件首选）

```bash
$ less /var/log/messages
# 进入后按键：空格=下一页  b=上一页  /=向下搜索  n=下一个匹配  q=退出  G=跳到末尾  g=跳到开头
```

**原理**：`less` 按页加载，不把整个文件读进内存，看几个 GB 的日志也不卡；`more` 是早期版本只能向下翻，`less` 可上下自由翻页（"less is more"）。

#### 2.2.3 `head / tail` 看头/看尾

```bash
$ head /etc/passwd          # 默认看前 10 行
$ head -n 5 /etc/passwd     # 只看前 5 行
$ tail /etc/passwd          # 默认看最后 10 行
$ tail -n 5 /var/log/messages   # 看最后 5 行
$ tail -f /var/log/messages     # -f 持续跟踪，文件追加新内容时实时显示
$ tail -f /var/log/messages /var/log/secure   # 可同时跟踪多个文件
```

**原理**：`tail -f` 会一直"盯"着文件，新内容追加就立即打印，运维看实时日志必备；按 `Ctrl+C` 退出跟踪。

### 2.3 查找：`find / which / whereis`

#### 2.3.1 `find` 万能查找（按名字/类型/时间/大小）

```bash
# 按名字查找（当前目录及子目录，* 为通配符）
$ find /etc -name "*.conf"                # 在 /etc 下找所有 .conf 文件
$ find / -name "nginx.conf" 2>/dev/null   # 全盘找，错误信息丢弃

# 按类型：f 普通文件 d 目录 l 链接
$ find /home -type d -name "test"          # 找名为 test 的目录

# 按修改时间：-mtime n 表示 n 天前那天修改；+7 超过 7 天；-7 7 天内
$ find /var/log -type f -mtime +7         # 找 7 天前修改的日志文件

# 按大小：+100M 大于 100MB
$ find / -type f -size +100M 2>/dev/null  # 找大于 100MB 的文件

# 查找并执行操作：删除 7 天前的 .log
$ find /var/log -name "*.log" -mtime +7 -delete
# 或搭配 -exec：
$ find /tmp -type f -name "*.tmp" -exec rm -f {} \;   # {} 代表找到的每个文件，\; 表示命令结束
```

**原理**：`find` 逐级遍历目录树，对每个文件用表达式匹配；`-exec` 把匹配结果交给后续命令处理，是"批量处理文件"的利器。

#### 2.3.2 `which / whereis` 找命令

```bash
$ which ls           # 找到可执行文件 ls 的绝对路径
/usr/bin/ls          # 别名：alias ls='ls --color=auto'

$ which python3
/usr/bin/python3

$ whereis ls         # 更全面：二进制、手册、源码位置都列出
ls: /usr/bin/ls /usr/share/man/man1/ls.1.gz
```

**原理**：`which` 沿 `PATH` 环境变量逐目录找命令；`whereis` 额外搜索手册页和源码路径。若 `which` 找不到某命令，多半是没安装或不在 `PATH` 里。

### 2.4 统计与文本处理：`grep / wc / sort / uniq / cut / tr`

#### 2.4.1 `grep` 按关键字过滤行

```bash
$ grep "root" /etc/passwd            # 找包含 root 的行
$ grep -v "nologin" /etc/passwd      # -v 反向：排除包含 nologin 的行
$ grep -i "ERROR" app.log            # -i 忽略大小写
$ grep -n "root" /etc/passwd         # -n 显示行号
$ grep -c "root" /etc/passwd         # -c 统计匹配行数
$ grep -r "listen" /etc/nginx/       # -r 递归搜索目录
$ ps -ef | grep nginx                # 管道配合，过滤进程
```

**原理**：`grep` 逐行读入，输出符合正则表达式（默认是基本正则）的行；`-E` 或 `egrep` 支持扩展正则。

#### 2.4.2 `wc / sort / uniq`

```bash
$ wc -l /etc/passwd        # -l 行数；-w 单词数；-c 字节数
$ wc -l /etc/passwd        # 输出：45 /etc/passwd（45 行）

$ sort -n file.txt         # 按数字排序
$ sort -r file.txt         # 降序
$ sort -t: -k3 -n /etc/passwd   # -t: 以冒号分列，-k3 按第 3 列（UID）数字排序

$ uniq file.txt            # 去重（只对相邻重复行有效，通常先 sort）
$ sort file.txt | uniq -c  # -c 统计每个值出现次数
```

**原理**：`uniq` 只合并相邻相同行，所以经典组合是 `sort | uniq -c` 来做频次统计；`sort -t: -k3` 中的 `:` 是分隔符，`k3` 是"第 3 个字段"。

#### 2.4.3 `cut / tr` 切列与字符转换

```bash
# cut 按分隔符切出某一列
$ cut -d: -f1 /etc/passwd        # -d: 冒号分隔，-f1 第 1 列（用户名）
$ cut -d: -f1,3 /etc/passwd      # 第 1 和第 3 列（用户名+UID）
$ cut -c1-5 /etc/passwd          # 按字符位置取 1~5 个字符

# tr 字符转换/删除
$ echo "hello" | tr 'a-z' 'A-Z'  # 转大写，输出 HELLO
$ echo "a b c" | tr ' ' '\n'     # 空格换行
$ echo "abc123" | tr -d '0-9'    # -d 删除数字，输出 abc
```

**原理**：`cut` 适合处理"固定分隔"的文本（如 `/etc/passwd` 用冒号）；`tr` 做字符级映射或删除，配合管道常用于清洗输出。

### 2.5 打包与压缩：`tar / gzip / zip`

#### 2.5.1 `tar` 打包（归档）

```bash
# 打包+压缩一个目录（最常用组合）
$ tar -czvf backup.tar.gz /home/alice
# -c 创建 -z 用 gzip 压缩 -v 显示过程 -f 指定文件名

# 解压
$ tar -xzvf backup.tar.gz -C /opt/
# -x 解包 -z gzip 解压 -v 过程 -f 文件 -C 指定解压目录

# 只打包不压缩（.tar）
$ tar -cvf dir.tar dir/
# 用 bzip2（压缩率更高，更慢）
$ tar -cjvf dir.tar.bz2 dir/      # -j 表示 bzip2

# 不解压直接看包里有什么
$ tar -tzvf backup.tar.gz         # -t 列出内容
```

**参数助记**：`c`=create、`x`=extract、`z`=gzip、`j`=bzip2、`v`=verbose、`f`=file（必须紧跟文件名）、`t`=list、`C`=目录。

**原理**：`tar` 本身只"打包"（把多个文件合成一个归档），压缩交给 `gzip`/`bzip2` 完成，所以有 `.tar.gz`、`.tar.bz2` 之分。`.gz` 只能压单个文件，因此常见流程是"先 tar 再 gz"。

#### 2.5.2 `gzip` 单独压缩文件

```bash
$ gzip file.txt          # 生成 file.txt.gz，原文件被删除
$ gunzip file.txt.gz     # 解压，还原 file.txt
$ gzip -k file.txt       # -k 保留原文件
```

#### 2.5.3 `zip`（跨平台，与 Windows 互通）

```bash
$ zip -r backup.zip /home/alice    # -r 递归打包目录为 .zip
$ unzip backup.zip -d /opt/        # 解压到指定目录
$ zip -r -P 密码 backup.zip dir/   # -P 设置解压密码（不推荐，会暴露在进程列表）
```

**原理**：`zip` 同时完成打包与压缩，且 Windows/Mac 原生支持，适合跨平台传递；Linux 生态内部更常用 `tar.gz`。

### 2.6 进程管理：`ps / top / htop / kill`

#### 2.6.1 `ps` 快照查看进程

```bash
$ ps aux                 # 显示所有用户的全部进程（BSD 风格）
$ ps -ef                 # 显示所有进程（System V 风格，含 PPID 父进程）
$ ps -ef | grep nginx    # 配合 grep 查特定进程
$ ps -p 915 -o pid,ppid,user,cmd   # 只看 PID=915 的指定字段
```

**`ps aux` 输出字段解释**：

```bash
$ ps aux | head -3
USER  PID %CPU %MEM  VSZ    RSS TTY STAT START TIME COMMAND
root    1  0.0  0.1 168580 8912 ?   Ss   10:00 0:02 /usr/lib/systemd/systemd
```

| 字段 | 含义 |
|------|------|
| USER | 进程属主 |
| PID | 进程 ID |
| %CPU/%MEM | CPU / 内存占用百分比 |
| VSZ | 虚拟内存大小(KB) |
| RSS | 常驻内存(物理内存)大小(KB) |
| TTY | 关联终端，`?` 表示无终端（后台/服务） |
| STAT | 状态：S 睡眠、R 运行、D 不可中断、Z 僵尸、s 会话首进程 |
| START | 启动时间 |
| TIME | 累计占用 CPU 时间 |
| COMMAND | 命令 |

**原理**：`ps` 是"瞬时快照"，执行那一刻就退出，不会持续刷新；要持续监控用 `top`。

#### 2.6.2 `top` 实时监控（`htop` 更友好）

```bash
$ top
# 按键：q 退出、P 按 CPU 排序、M 按内存排序、k 输入 PID 杀进程、1 展开各 CPU 核
$ htop     # 彩色、可鼠标、更直观（若未安装：dnf install -y htop）
```

**原理**：`top` 默认每 3 秒刷新，首行 `load average`（1/5/15 分钟平均负载）是判断系统繁忙度的关键指标；`htop` 是 `top` 的增强版。

#### 2.6.3 `kill` 结束进程

```bash
$ kill 1234            # 给 PID=1234 发 TERM 信号（礼貌请求退出）
$ kill -9 1234         # 发 KILL 信号（强杀，进程无法忽略）
$ kill -15 1234        # 等价于 kill，TERM 信号
$ kill -l              # 列出所有信号
$ pkill -f nginx       # 按名字/命令行匹配杀进程（-f 匹配完整命令行）
$ killall nginx        # 按进程名杀（注意可能误伤同名）
```

**原理**：`kill` 本质是"发送信号"。`TERM(15)` 让进程有机会清理资源后退出；`KILL(9)` 直接由内核终止、进程无法捕获。日常优先用 `TERM`，无响应再 `KILL`。

### 2.7 磁盘与内存：`df / du / free / lsblk / mount`

#### 2.7.1 `df / du`

```bash
$ df -h        # -h 人类可读，查看文件系统磁盘占用
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda3        40G  8.1G   30G  22% /
devtmpfs        1.9G     0  1.9G   0% /dev
tmpfs           1.9G     0  1.9G   0% /dev/shm
tmpfs           1.9G  9.0M  1.9G   1% /run
/dev/sda1       1.0G  220M  785M  22% /boot

$ df -hT       # -T 额外显示文件系统类型（ext4/xfs）
$ df -i        # -i 查看 inode 使用率（小文件过多时 inode 会耗尽）

$ du -sh /var/log      # -s 汇总 -h 人类可读：整个目录占用多大
$ du -h --max-depth=1 /home   # 只显示 1 层子目录各自大小
$ du -sh * | sort -h          # 当前目录各文件/目录大小，按大小排序
```

**原理**：`df` 看的是"文件系统整体还剩多少"，`du` 看的是"某个目录实际用了多少"。两者数字可能对不上，因为已删除但被进程占用的文件，`du` 不统计而空间仍被占用（可用 `lsof +L1` 排查）。

#### 2.7.2 `free` 内存

```bash
$ free -h
               total        used        free      shared  buff/cache   available
Mem:           3.7Gi       1.1Gi       1.4Gi        18Mi       1.2Gi       2.3Gi
Swap:          2.0Gi          0B       2.0Gi
```

**字段解释**：`total` 总内存；`used` 已用；`free` 完全空闲；`buff/cache` 被内核用作磁盘缓存（可随时释放给程序）；`available` 真正可分配给新进程的内存（比 free 更准）。Linux 内存"看起来用满"通常是缓存占用，属正常。

#### 2.7.3 `lsblk / mount`

```bash
$ lsblk          # 以树形列出块设备
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda      8:0    0   40G  0 disk
├─sda1   8:1    0    1G  0 part /boot
├─sda2   8:2    0   20G  0 part /
└─sda3   8:3    0   19G  0 part [SWAP]

$ mount | grep " / "        # 查看根分区挂载信息
$ mount /dev/sdb1 /mnt      # 临时挂载一块盘到 /mnt
$ umount /mnt               # 卸载
```

**原理**：`lsblk` 显示磁盘→分区→挂载点的层次；`mount` 把设备文件（如 `/dev/sdb1`）挂到目录树上，挂载后访问目录即访问磁盘。

### 2.8 权限命令：`chmod / chown / chgrp / umask`（详见第 3 章）

### 2.9 用户命令：`useradd / usermod / passwd / groupadd`（详见第 4 章）

### 2.10 网络：`ip / ss / ping`

#### 2.10.1 `ip` 现代网络配置（替代 ifconfig）

```bash
$ ip addr             # 查看所有网卡 IP 地址
$ ip a                # 简写，同上
$ ip addr show ens160 # 只看 ens160 网卡
$ ip route            # 查看路由表（默认网关）
$ ip link set ens160 up/down   # 启停网卡
$ ip addr add 192.168.1.100/24 dev ens160   # 临时加一个 IP
```

**原理**：`ip` 属于 iproute2 工具集，功能覆盖并超越老旧的 `ifconfig`（net-tools 包）。`ip addr` 中 `state UP`、`inet 192.168.x.x/24` 是关键字段。

#### 2.10.2 `ss` 查看端口/连接（替代 netstat）

```bash
$ ss -tlnp          # -t tcp -l 监听 -n 数字(不解析域名) -p 显示进程
State  Recv-Q Send-Q Local Address:Port  Peer Address:Port Process
LISTEN 0      128          0.0.0.0:22          0.0.0.0:*     users:(("sshd",pid=915,fd=3))
LISTEN 0      128             [::]:22             [::]:*     users:(("sshd",pid=915,fd=4))

$ ss -tulnp        # 加 -u 看 UDP
$ ss -s            # 统计当前连接总数
$ ss -ant | grep ESTAB | wc -l   # 统计已建立连接数
```

**原理**：`ss` 从内核直接读取 socket 信息，比 `netstat` 更快。`0.0.0.0:22` 表示"所有网卡的 22 端口都在监听"。

#### 2.10.3 `ping` 连通性测试

```bash
$ ping -c 4 192.168.1.1     # -c 只发 4 个包就退出
$ ping -c 4 www.baidu.com
64 bytes from 192.168.1.1: icmp_seq=1 ttl=64 time=0.5 ms
# 出现 time= 说明网络通；"Destination Host Unreachable" 说明不通
```

**原理**：`ping` 用 ICMP 协议，能通只说明网络层可达，不代表对方服务端口开放；测端口开放用 `ss` 或 `telnet`/`nc`。

### 2.11 远程：`ssh / scp / rsync`

#### 2.11.1 `ssh` 登录远程主机

```bash
$ ssh alice@192.168.1.100          # 用 alice 账号登录
$ ssh -p 2222 alice@192.168.1.100  # -p 指定端口（默认 22）
$ ssh -i ~/.ssh/key alice@192.168.1.100   # -i 指定私钥（免密登录）
$ ssh 192.168.1.100 "hostname"     # 登录后立即执行一条命令并返回
```

**原理**：SSH 全程加密；首次连接会问"是否信任主机指纹"（`yes` 后写入 `~/.ssh/known_hosts`）。免密登录基于密钥对：本地私钥、服务器公钥（详见第 4 章）。

#### 2.11.2 `scp` 跨机复制文件

```bash
$ scp local.txt alice@192.168.1.100:/home/alice/     # 本地→远程
$ scp alice@192.168.1.100:/home/alice/remote.txt ./  # 远程→本地
$ scp -r dir/ alice@192.168.1.100:/home/alice/       # -r 递归传目录
$ scp -P 2222 f.txt alice@192.168.1.100:/tmp/        # 注意 scp 大写 -P
```

#### 2.11.3 `rsync` 高效同步（增量、可断点）

```bash
$ rsync -avz /home/alice/data/ alice@192.168.1.100:/backup/data/
# -a 归档模式(保留权限时间) -v 详细 -z 传输压缩

$ rsync -av --delete /src/ /dst/    # --delete 目标多余文件也删，保持完全一致
$ rsync -avP /bigfile alice@host:/tmp/   # -P 显示进度 + 支持断点续传
```

**原理**：`rsync` 只传输"差异部分"，比 `scp` 快得多，是备份/同步的首选；目录源结尾带 `/` 表示同步"目录里的内容"而非目录本身。

### 2.12 下载：`wget / curl`

```bash
# wget 下载文件
$ wget https://example.com/file.tar.gz
$ wget -O newname.tar.gz https://example.com/file.tar.gz   # -O 指定保存名
$ wget -c https://example.com/big.iso    # -c 断点续传

# curl 下载 + 更多功能
$ curl -o file.tar.gz https://example.com/file.tar.gz   # -o 指定输出文件
$ curl -O https://example.com/file.tar.gz               # -O 用远程文件名保存
$ curl -I https://example.com           # -I 只取响应头（看状态码、服务器）
$ curl -s https://example.com | head    # -s 静默模式
$ curl -X POST -d "a=1" https://api.example.com   # 发 POST 请求
```

**原理**：`wget` 专注下载、支持递归与断点续传；`curl` 更像通用网络工具，支持各种协议和方法，适合接口调试与脚本。安装：`dnf install -y wget curl`。

### 2.13 动手练习

1. 在 `/tmp` 建目录 `test/{a,b,c}`，在 `a` 下用 `touch` 建 3 个文件，再 `ls -R` 递归查看。
2. 用 `cp -r` 复制 `test` 为 `test2`，用 `rm -rf` 删掉 `test2`。
3. 用 `head -n 5` 和 `tail -n 5` 看 `/etc/passwd`，再用 `less` 翻页浏览。
4. 用 `grep -c /bin/bash /etc/passwd` 统计能用 bash 登录的用户数。
5. 用 `find /etc -name "*.conf" | wc -l` 数一下 `/etc` 下有多少 .conf 文件。
6. 用 `tar -czvf /tmp/mybackup.tar.gz /etc/hostname /etc/passwd` 打一个包，再用 `tar -tzvf` 看内容，最后解到 `/tmp`。

---

## 第 3 章 文件权限

### 3.1 是什么、为什么用

- **是什么**：Linux 对每个文件记录"谁（属主/属组/其他人）能对它做什么（读/写/执行）"。
- **为什么用**：多用户系统必须隔离权限，防止普通用户改系统文件、看他人隐私。
- **怎么用**：用 `ls -l` 看权限，用 `chmod/chown` 改权限，用 `umask` 控制新建文件默认权限。

### 3.2 rwx 含义

```bash
$ ls -l /etc/passwd
-rw-r--r--. 1 root root 1890 Jul 20 10:00 /etc/passwd
```

`-rw-r--r--` 分四段：第 1 位是类型，后面 9 位每 3 位一组：

| 位 | 含义（文件） | 含义（目录） |
|----|--------------|--------------|
| r (读) | 可查看文件内容 | 可列出目录内文件名 |
| w (写) | 可修改文件内容 | 可在目录内建/删/改名文件 |
| x (执行) | 可作为程序运行 | 可进入(cd)该目录，并访问其内文件 |

三段依次为：**属主(u)** / **属组(g)** / **其他人(o)**。

上例 `-rw-r--r--`：属主 root 可读写，属组 root 只读，其他人只读。

### 3.3 数字表示法：4/2/1

| 权限 | 数字 | 记忆 |
|------|------|------|
| r | 4 | 读 |
| w | 2 | 写 |
| x | 1 | 执行 |
| rwx | 7 | 4+2+1 |
| rw- | 6 | 4+2 |
| r-x | 5 | 4+1 |
| r-- | 4 | 4 |
| --- | 0 | 无 |

所以 `chmod 755 file` = `rwxr-xr-x`（属主 7、属组 5、其他 5）。

### 3.4 `chmod` 两种写法（符号 / 数字）

```bash
# ---- 符号法（在原有权限上增减）----
$ chmod u+x script.sh      # 给属主(u)加执行权限
$ chmod g-w file.txt       # 去掉属组(g)的写权限
$ chmod o=r file.txt       # 把其他(o)权限设为只读
$ chmod a+rx dir/          # 给所有人(a=all)加读和执行
$ chmod -R 755 /opt/web    # -R 递归应用到目录及其所有内容

# ---- 数字法（一次性设定完整权限）----
$ chmod 755 script.sh      # rwxr-xr-x
$ chmod 644 file.txt       # rw-r--r--
$ chmod 600 private.key    # rw-------（只有属主能读写，密钥常用）
$ chmod 750 /home/alice    # rwxr-x---（属主全权限，属组可读执行，其他无）
```

**原理**：符号法适合"微调"（加/减某一项），数字法适合"整体重设"。`chmod` 只改权限位，不改属主属组。

### 3.5 `chown` 改属主、`chgrp` 改属组

```bash
$ chown alice file.txt            # 把文件属主改为 alice
$ chown alice:staff file.txt      # 同时改属主和属组
$ chown -R alice:alice /home/alice   # -R 递归
$ chgrp staff file.txt            # 只改属组
$ chown :staff file.txt           # 只改属组（冒号前留空）
```

**原理**：只有 root 能随意把文件给别人；普通用户只能改自己文件的属组（且需是目标组成员）。`chown user:group` 中冒号前是属主、后是属组。

### 3.6 特殊权限：SUID / SGID / Sticky Bit

| 特殊权限 | 数字位 | 作用 | 常见场景 |
|----------|--------|------|----------|
| SUID | 4 (置于属主位) | 执行时以"文件属主"身份运行 | `/usr/bin/passwd`、`/usr/bin/su` |
| SGID | 2 (置于属组位) | 执行时以"文件属组"身份运行；目录上=新建文件继承目录属组 | 共享目录 `/shared` |
| Sticky Bit | 1 (置于其他位) | 目录里只有文件属主或 root 能删文件 | `/tmp`、`/var/tmp` |

```bash
# 查看：SUID 显示为属主位 s，SGID 为属组位 s，Sticky 为其他位 t
$ ls -l /usr/bin/passwd
-rwsr-xr-x. 1 root root 33K Jul 20 10:00 /usr/bin/passwd   # 属主位的 s 即 SUID
$ ls -ld /tmp
drwxrwxrwt. 8 root root 4096 Jul 20 10:00 /tmp              # 其他位 t 即 Sticky

# 设置
$ chmod u+s /usr/bin/mytool      # 加 SUID
$ chmod g+s /shared              # 目录加 SGID（新建文件继承目录属组）
$ chmod +t /shared               # 加 Sticky Bit
$ chmod 4755 file                # 数字法：4=SUID, 7=rwx, 5, 5
$ chmod 1777 /tmp2               # 数字法：1=Sticky, 777
```

**原理**：普通用户改自己密码时要写 `/etc/shadow`（仅 root 可写），靠的就是 `passwd` 命令带 SUID，执行瞬间获得 root 权限。Sticky Bit 保证 `/tmp` 里你的文件别人删不掉。

### 3.7 `umask` 默认权限掩码

```bash
$ umask          # 查看当前掩码，通常 0022
0022

# 新建文件/目录后观察默认权限
$ touch new.txt && ls -l new.txt
-rw-r--r--. 1 alice alice 0 Jul 20 11:00 new.txt    # 644
$ mkdir newdir && ls -ld newdir
drwxr-xr-x. 2 alice alice 6 Jul 20 11:00 newdir     # 755

# 临时修改掩码
$ umask 077      # 新建文件 600、目录 700（更私密）
```

**原理**：默认权限 = 基础权限 − umask。文件基础是 `666`，目录是 `777`。`umask 022` 时：文件 `666-022=644`，目录 `777-022=755`。注意 umask 是"减去的权限位"，数字越大越严格。永久生效可写进 `~/.bashrc` 或 `/etc/profile`。

### 3.8 动手练习

1. 建文件 `test.sh` 写一行 `echo hi`，观察默认权限，用 `chmod u+x` 后执行 `./test.sh`。
2. 分别用符号法和数字法把 `test.sh` 设为 `rwxr-xr-x`，比较结果。
3. 建目录 `/tmp/share`，`chmod 1777` 后用两个不同用户各建文件，验证互相能否删除。
4. 把 `umask` 改为 027，新建文件和目录，算出并验证默认权限。
5. 解释 `chmod 4755` 和 `chmod 2770` 每个数字的含义。

---

## 第 4 章 用户与组

### 4.1 三个关键文件逐列解释

#### 4.1.1 `/etc/passwd` 用户账号

```bash
$ cat /etc/passwd
root:x:0:0:root:/root:/bin/bash
alice:x:1000:1000:Alice:/home/alice:/bin/bash
```

每行 7 个字段，用 `:` 分隔：

| 序号 | 字段 | 示例 | 含义 |
|------|------|------|------|
| 1 | 用户名 | alice | 登录名 |
| 2 | 密码占位 | x | 真密码已移到 `/etc/shadow`，这里恒为 x |
| 3 | UID | 1000 | 用户 ID，0=root，1-999 系统保留，1000+ 普通用户 |
| 4 | GID | 1000 | 主组 ID |
| 5 | 注释/GECOS | Alice | 描述信息（全名等） |
| 6 | 家目录 | /home/alice | 登录后所在目录 |
| 7 | 登录 shell | /bin/bash | 登录后用的 shell；`/sbin/nologin` 表示禁止登录（服务账号） |

#### 4.1.2 `/etc/shadow` 密码影子文件（仅 root 可读）

```bash
$ cat /etc/shadow | head -1
root:$6$8YxZ...hash...:18000:0:99999:7:::
```

每行 9 个字段，用 `:` 分隔：

| 序号 | 字段 | 含义 |
|------|------|------|
| 1 | 用户名 | 对应 passwd |
| 2 | 加密密码 | `$6$` 开头表示 SHA-512；`!!` 或 `*` 表示未设密码/锁定 |
| 3 | 上次改密时间 | 从 1970-01-01 起的天数 |
| 4 | 最小改密间隔 | 天内不能再次改密 |
| 5 | 密码有效期 | 天后强制改密（99999=几乎不过期） |
| 6 | 到期前警告天数 | |
| 7 | 过期后宽限天数 | 仍可登录但须改密 |
| 8 | 账号失效日期 | 过期后完全锁定 |
| 9 | 保留 | 暂未使用 |

**原理**：把密码从 `/etc/passwd` 挪到仅 root 可读的 `/etc/shadow`，是为了防止普通用户读到加密密码后离线暴力破解。

#### 4.1.3 `/etc/group` 组

```bash
$ cat /etc/group
wheel:x:10:alice,bob
```

每行 4 字段：**组名:密码占位:GID:组内成员列表**（逗号分隔）。`wheel` 组是 RHEL 系的管理员组，成员可用 `sudo` 提权。

### 4.2 建用户、设密码、删改用户

```bash
# 创建用户
# useradd alice                      # 创建 alice，用系统默认（家目录 /home/alice、bash、UID 自增）
# useradd -m -s /bin/bash -G wheel bob   # -m 建家目录 -s 指定shell -G 追加附加组wheel
# useradd -u 1500 -g users -c "Web Admin" webuser   # -u 指定UID -g 主组 -c 注释

# 设置/修改密码（交互式，输入两次）
# passwd alice
# echo "MyPass123" | passwd --stdin alice   # 非交互（脚本用，注意密码会留命令历史）

# 修改用户
# usermod -aG wheel alice           # -aG 追加到附加组（不加 -a 会覆盖原附加组！）
# usermod -l newname oldname        # 改登录名
# usermod -s /sbin/nologin alice    # 禁止登录
# usermod -L alice                  # 锁定账号；-U 解锁

# 删除
# userdel alice                     # 只删账号，保留家目录
# userdel -r alice                  # -r 连家目录和邮件一起删

# 组
# groupadd dev                      # 建组
# groupmod -n developers dev        # 改组名
# groupdel dev                      # 删组

# 查看用户所属组
$ id alice
uid=1000(alice) gid=1000(alice) groups=1000(alice),10(wheel)
```

**原理**：`useradd` 的默认行为（是否建家目录、UID 起点）由 `/etc/default/useradd` 和 `/etc/login.defs` 控制。`-G` 是"附加组"（额外身份），`-g` 是"主组"（文件默认属组）。改附加组务必加 `-a`，否则会清掉原有附加组。

### 4.3 sudo 配置（visudo）

`sudo` 让普通用户临时以 root 身份执行命令，且有日志可查，比直接给 root 密码安全。

```bash
# 查看 sudo 是否安装，未装则安装
# dnf install -y sudo

# 方式一：把用户加入 wheel 组（wheel 组默认已在 sudoers 里启用）
# usermod -aG wheel alice

# 方式二：用 visudo 编辑 sudoers（必须用 visudo，会自动语法检查防锁死）
# visudo
```

在 `visudo` 打开的编辑器里，找到并取消注释这一行（RHEL 9 默认已启用）：

```
## Allows people in group wheel to run all commands
%wheel  ALL=(ALL)       ALL
```

再添加一条只给 alice 特定权限的规则：

```
alice   ALL=(ALL)       NOPASSWD: /usr/bin/systemctl restart nginx
```

**字段解释**：

| 段 | 示例 | 含义 |
|----|------|------|
| 用户/组 | `alice` / `%wheel` | 谁有权限（`%` 前缀表示组） |
| 主机 | `ALL` | 从哪些主机可用 |
| 运行身份 | `(ALL)` | 可以切换到哪些用户执行 |
| 命令 | `NOPASSWD: /usr/bin/systemctl restart nginx` | 可执行的命令；加 NOPASSWD 免密 |

验证与使用：

```bash
$ su - alice              # 切换成 alice
$ sudo -l                 # 列出当前用户可用的 sudo 权限
$ sudo systemctl status nginx    # 用 sudo 执行特权命令（首次要求输 alice 自己的密码）
$ sudo -i                 # 获得一个 root 的交互式 shell
```

**原理**：`sudo` 记日志到 `/var/log/secure`，谁在何时执行了什么命令都可追溯；`visudo` 会在保存前做语法检查，避免手改坏 `/etc/sudoers` 导致所有 sudo 失效。

### 4.4 动手练习

1. 用 `useradd -m -s /bin/bash -G wheel dev1` 建用户，`passwd` 设密码，用 `id dev1` 验证组。
2. `su - dev1` 切换到 dev1，`sudo whoami` 验证能提权到 root。
3. 逐列解释 `/etc/passwd`、`/etc/shadow`、`/etc/group` 中 dev1 相关行每个字段。
4. 用 `visudo` 给 dev1 加一条只能 `systemctl restart sshd` 的 NOPASSWD 权限并测试。
5. `useradd` 一个服务账号 `nagios`，shell 设为 `/sbin/nologin`，解释为什么它不能登录。

---

## 第 5 章 vim 文本编辑器

### 5.1 是什么、为什么用

- **是什么**：Linux 最经典的文本编辑器，几乎每台机器都预装，远程改配置的标配工具。
- **为什么用**：改 `/etc` 下配置文件、写脚本都要靠它；图形界面不可用的服务器上，vim 是唯一可靠的编辑手段。
- **怎么用**：记住"**三种模式**"和十几个高频按键就能熟练使用。

### 5.2 三种模式

| 模式 | 进入方式 | 用途 | 提示 |
|------|----------|------|------|
| 普通模式 (Normal) | 打开 vim 默认进入 | 移动光标、复制粘贴删除、执行命令 | 左下角无 `-- INSERT --` |
| 插入模式 (Insert) | 普通模式按 `i` / `a` / `o` | 输入文字 | 左下角显示 `-- INSERT --` |
| 命令模式 (Command-line) | 普通模式按 `:` | 保存、退出、查找替换、设置 | 左下角出现 `:` |

三者关系：**普通模式 ↔ 插入模式**（`i` 进入，`Esc` 退出）；**普通模式 → 命令模式**（`:` 进入，`Esc` 返回普通模式）。

### 5.3 高频按键速查

| 按键 | 作用 | 说明 |
|------|------|------|
| `i` | 光标前插入 | insert |
| `a` | 光标后插入 | append |
| `o` | 下一行插入 | 另起新行 |
| `Esc` | 回到普通模式 | 任何模式按它退回 |
| `dd` | 删除（剪切）整行 | 按 `5dd` 删 5 行 |
| `yy` | 复制整行 | `5yy` 复制 5 行 |
| `p` | 粘贴 | 光标后粘贴 |
| `u` | 撤销 | 可连续撤销 |
| `Ctrl+r` | 重做 | 撤销的撤销 |
| `gg` / `G` | 跳到文件头 / 文件尾 | |
| `0` / `$` | 行首 / 行尾 | |
| `/word` | 向下搜索 word | `n` 下一个，`N` 上一个 |
| `:%s/old/new/g` | 全文替换 | 把全部 old 换成 new |
| `:w` | 保存 | |
| `:wq` | 保存并退出 | 等价 `:x` 或 `ZZ` |
| `:q!` | 不保存强制退出 | 丢弃修改 |
| `:set nu` | 显示行号 | `:set nonu` 关闭 |

### 5.4 一个完整的编辑示例

```bash
$ vim /tmp/hello.txt
```

1. 按 `i` 进入插入模式，输入两行文字：
   ```
   hello linux
   hello vim
   ```
2. 按 `Esc` 回普通模式。
3. 按 `yy` 复制一行，按 `p` 粘贴，会多出一行。
4. 按 `dd` 删掉一行。
5. 按 `:` 输入 `%s/hello/hi/g` 回车，把两个 `hello` 全部替换成 `hi`。
6. 按 `:` 输入 `wq` 回车保存退出。
7. 用 `cat /tmp/hello.txt` 验证内容。

### 5.5 10 分钟练习（照着做）

```bash
# 1) 创建并打开文件
$ vim practice.txt

# 2) 按 i 进入插入模式，粘贴以下 5 行（练习复制/删除/替换）
apple
banana
apple pie
cherry
apple juice

# 3) 按 Esc，依次练习：
#    gg        -> 跳到第 1 行
#    yy 后 p   -> 复制第 1 行并粘贴，观察新增一行 apple
#    dd        -> 删除刚粘贴的那行
#    3dd       -> 删除下面 3 行
#    u         -> 撤销，恢复被删的 3 行
#    :set nu   -> 打开行号
#    :%s/apple/orange/g   -> 全文 apple 换成 orange
#    /cherry   -> 搜索 cherry，n 跳到下一个
#    G         -> 跳到文件末尾
#    o         -> 末尾新起一行，输入 "done"

# 4) 保存退出：Esc 后 :wq
# 5) 验证：cat practice.txt
```

---

## 第 6 章 软件包管理

### 6.1 是什么、为什么用

- **是什么**：Linux 用"包管理器"自动处理软件的下载、依赖解析、安装、升级、卸载，而不是像 Windows 那样一个个装 exe。
- **为什么用**：手动编译会陷入"缺依赖 → 编译失败 → 装依赖 → 又缺依赖"的死循环；包管理器自动解决依赖冲突。
- **怎么用**：RHEL 系（Rocky/Alma/CentOS）用 `dnf`；Debian 系（Ubuntu/Debian）用 `apt`。

### 6.2 `dnf` 常用子命令

```bash
# 搜索
# dnf search nginx                 # 按关键字搜包
# dnf info nginx                   # 查看包详情（版本、描述、依赖）
# dnf list installed | grep nginx  # 列出已装包
# dnf list available | grep nginx  # 列出可装包

# 安装/卸载
# dnf install -y nginx             # 安装（-y 自动确认）
# dnf remove -y nginx              # 卸载
# dnf reinstall nginx              # 重装（修复损坏）

# 更新/升级
# dnf update                       # 更新所有已装包
# dnf upgrade                      # 同 update
# dnf update -y nginx              # 只更新 nginx
# dnf check-update                 # 只检查有无更新，不安装

# 仓库与缓存
# dnf repolist                     # 列出已启用仓库
# dnf repolist all                 # 含禁用仓库
# dnf clean all                    # 清缓存
# dnf makecache                    # 重建缓存

# 查询某文件属于哪个包
# dnf provides /usr/bin/nginx      # 反查文件归属
# dnf provides */nginx.conf

# 历史与回滚
# dnf history                      # 查看安装/更新历史
# dnf history undo <ID>            # 回滚某次操作
```

**原理**：`dnf` 从配置的"软件仓库（repo，即一堆 rpm 包 + 元数据索引）"拉取包，读取每个 rpm 的依赖信息（如 nginx 依赖 openssl），自动把所有依赖一并装好，并做完整性校验。

### 6.3 配置国内 yum 源（阿里云 / 清华，完整步骤）

> 默认官方源在海外，速度慢。换成阿里云或清华镜像可大幅提速。

#### 步骤 1：备份原仓库

```bash
# mkdir /etc/yum.repos.d/backup
# mv /etc/yum.repos.d/*.repo /etc/yum.repos.d/backup/
```

#### 步骤 2：新建阿里云源（Rocky 9 为例）

```bash
# cat > /etc/yum.repos.d/rocky.repo <<'EOF'
[baseos]
name=Rocky Linux $releasever - BaseOS (Aliyun)
baseurl=https://mirrors.aliyun.com/rockylinux/$releasever/BaseOS/$basearch/os/
gpgcheck=1
enabled=1
gpgkey=https://mirrors.aliyun.com/rockylinux/RPM-GPG-KEY-Rocky-9

[appstream]
name=Rocky Linux $releasever - AppStream (Aliyun)
baseurl=https://mirrors.aliyun.com/rockylinux/$releasever/AppStream/$basearch/os/
gpgcheck=1
enabled=1
gpgkey=https://mirrors.aliyun.com/rockylinux/RPM-GPG-KEY-Rocky-9

[extras]
name=Rocky Linux $releasever - Extras (Aliyun)
baseurl=https://mirrors.aliyun.com/rockylinux/$releasever/extras/$basearch/os/
gpgcheck=1
enabled=1
gpgkey=https://mirrors.aliyun.com/rockylinux/RPM-GPG-KEY-Rocky-9
EOF
```

**字段解释**：

| 字段 | 含义 |
|------|------|
| `[baseos]` | 仓库 ID（唯一标识） |
| `name` | 人类可读名称 |
| `baseurl` | 包所在 URL；`$releasever` 系统版本、`$basearch` 架构自动展开 |
| `gpgcheck=1` | 开启 GPG 签名校验（防篡改） |
| `enabled=1` | 启用该仓库 |
| `gpgkey` | 校验用公钥地址 |

**清华源**只需把 `mirrors.aliyun.com` 换成 `mirrors.tuna.tsinghua.edu.cn`，其余不变。AlmaLinux 9 用户把路径中的 `rockylinux` 换成 `almalinux` 即可。

#### 步骤 3：清理并重建缓存，验证

```bash
# dnf clean all
# dnf makecache
# dnf repolist
repo id                         repo name
appstream                       Rocky Linux 9 - AppStream (Aliyun)
baseos                          Rocky Linux 9 - BaseOS (Aliyun)
extras                          Rocky Linux 9 - Extras (Aliyun)
```

看到三个阿里云仓库即成功。测试安装：

```bash
# dnf install -y wget curl vim bash-completion
```

### 6.4 `rpm` 底层包管理

`dnf` 底层就是用 `rpm` 装 `.rpm` 文件；`rpm` 适合单包操作。

```bash
# rpm -qa                      # 列出所有已装包
# rpm -qa | grep nginx         # 查某包是否安装
# rpm -qi nginx                # 查看已装包信息
# rpm -ql nginx                # 列出该包安装了哪些文件
# rpm -qc nginx                # 列出该包的配置文件
# rpm -qf /usr/bin/nginx       # 反查文件属于哪个包
# rpm -ivh 包.rpm              # 安装：-i 安装 -v 详细 -h 进度条
# rpm -Uvh 包.rpm              # 升级
# rpm -e nginx                 # 卸载（不处理依赖）
# rpm -V nginx                 # 校验包文件是否被篡改
```

**原理**：`rpm` 不自动装依赖（缺依赖会报错），`dnf` 才负责依赖解析，所以日常用 `dnf`，底层诊断用 `rpm`。

### 6.5 `apt` 对照表（Debian/Ubuntu 系）

| 功能 | RHEL 系 (dnf) | Debian 系 (apt) |
|------|---------------|-----------------|
| 更新索引 | `dnf makecache` | `apt update` |
| 安装 | `dnf install pkg` | `apt install pkg` |
| 卸载 | `dnf remove pkg` | `apt remove pkg` |
| 卸载+删配置 | `dnf remove pkg`（-y） | `apt purge pkg` |
| 升级全部 | `dnf update` | `apt upgrade` |
| 搜索 | `dnf search kw` | `apt search kw` |
| 仓库列表 | `/etc/yum.repos.d/*.repo` | `/etc/apt/sources.list` |
| 已装列表 | `rpm -qa` | `dpkg -l` |
| 文件归属 | `rpm -qf /path` | `dpkg -S /path` |

### 6.6 源码编译安装（configure / make / make install）

当仓库没提供、或需要自定义功能时，从源码编译。

```bash
# 以安装 nginx 源码为例（先装编译依赖）
# dnf install -y gcc make pcre-devel zlib-devel openssl-devel

$ cd /usr/local/src
$ curl -O https://nginx.org/download/nginx-1.26.0.tar.gz
$ tar -xzf nginx-1.26.0.tar.gz
$ cd nginx-1.26.0

# 1) configure：检测环境、生成 Makefile、指定安装路径和功能
$ ./configure --prefix=/usr/local/nginx --with-http_ssl_module
# 参数解释：--prefix 安装目录；--with-http_ssl_module 启用 HTTPS 模块
# 成功会输出 "Configuration summary"，失败会提示缺什么依赖

# 2) make：按 Makefile 编译源码为二进制
$ make -j$(nproc)     # -j 并行编译，nproc 为 CPU 核数，加快速度

# 3) make install：把编译产物复制到 --prefix 指定的目录
# make install

# 验证
# /usr/local/nginx/sbin/nginx -V      # 查看版本与编译参数
```

**三层说明**：`configure` 是"配置"（检测依赖、生成构建规则）、`make` 是"编译"（把 C 源码转成二进制）、`make install` 是"安装"（复制到目标目录）。卸载通常删除 `--prefix` 目录即可（无包管理器记录）。

### 6.7 动手练习

1. 用 `dnf search` 找一个软件，`dnf info` 看详情，`dnf install -y` 安装，`rpm -ql` 看它装了哪些文件。
2. 完整配置一次阿里云源，`dnf makecache` 后用 `dnf repolist` 确认。
3. 用 `dnf provides /usr/bin/ssh` 反查 ssh 命令属于哪个包。
4. 安装 `gcc make`，用源码方式编译安装一个最简单的软件（如 `wget` 或任意 tar 包），验证 `make` 和 `make install`。

---

## 第 7 章 systemd 系统与服务管理

### 7.1 是什么、为什么用

- **是什么**：systemd 是现代 Linux 的初始化系统（PID 1），负责开机启动所有服务、管理服务生命周期。
- **为什么用**：统一用 `systemctl` 管理所有服务（启动/停止/重启/开机自启），并提供日志、依赖、资源限制等能力。
- **怎么用**：核心命令是 `systemctl`，服务配置文件是 `.service`（unit 单元）。

### 7.2 `systemctl` 常用子命令

```bash
# 服务启停（以 nginx 为例）
# systemctl start nginx           # 启动
# systemctl stop nginx            # 停止
# systemctl restart nginx         # 重启
# systemctl reload nginx          # 重载配置（不中断服务，比 restart 优雅）
# systemctl status nginx          # 查看状态（是否运行、PID、最近日志）
# systemctl enable nginx          # 设置开机自启
# systemctl disable nginx         # 取消开机自启
# systemctl is-enabled nginx      # 是否已设自启
# systemctl is-active nginx       # 是否正在运行

# 整体状态
# systemctl list-units --type=service   # 列出所有已加载的服务单元
# systemctl list-unit-files --type=service   # 列出所有服务及其自启状态
# systemctl get-default                 # 查看默认启动目标（multi-user.target）
# systemctl set-default multi-user.target   # 设置默认启动级别

# 系统操作
# systemctl reboot                    # 重启系统
# systemctl poweroff                  # 关机
# systemctl daemon-reload             # 修改了 unit 文件后重载 systemd 配置

# 查看服务日志
# journalctl -u nginx                 # 只看 nginx 的日志
# journalctl -u nginx -f              # 实时跟踪 nginx 日志
```

**状态字段说明**：`status` 输出的 `Active: active (running)` 表示运行中；`enabled` 表示已设自启（注意 active 和 enabled 是两个维度：enabled 只管开机是否拉起，active 管当前是否运行）。

### 7.3 写一个自定义 `.service` 完整示例

需求：写一个简单脚本作为服务，开机自启、失败自动重启、可用 systemctl 管理。

#### 步骤 1：写脚本

```bash
# cat > /usr/local/bin/myapp.sh <<'EOF'
#!/bin/bash
while true; do
    echo "$(date '+%F %T') myapp is running" >> /var/log/myapp.log
    sleep 10
done
EOF
# chmod +x /usr/local/bin/myapp.sh
```

#### 步骤 2：写 unit 文件

```bash
# cat > /etc/systemd/system/myapp.service <<'EOF'
[Unit]
Description=My Demo App Service          # 服务描述
After=network.target                    # 在网络就绪后启动

[Service]
Type=simple                             # 主进程类型：simple 表示直接前台运行
ExecStart=/usr/local/bin/myapp.sh       # 要执行的启动命令
Restart=always                          # 退出后总是自动重启
RestartSec=3                            # 重启间隔 3 秒
User=root                               # 以哪个用户运行
StandardOutput=append:/var/log/myapp.out   # 标准输出追加到日志

[Install]
WantedBy=multi-user.target              # 多用户模式下启用（开机自启的关键）
EOF
```

**各段解释**：

| 段 | 关键字段 | 作用 |
|----|----------|------|
| `[Unit]` | `Description` / `After` | 描述 + 启动顺序依赖 |
| `[Service]` | `Type` / `ExecStart` / `Restart` / `User` | 运行方式、启动命令、自动重启、运行身份 |
| `[Install]` | `WantedBy` | 指定在哪个 target 下建立软链接实现自启 |

#### 步骤 3：加载并启用

```bash
# systemctl daemon-reload          # 重载配置，让 systemd 识别新 unit
# systemctl start myapp            # 启动
# systemctl status myapp           # 查看状态（应显示 active (running)）
# systemctl enable myapp           # 设置开机自启
Created symlink /etc/systemd/system/multi-user.target.wants/myapp.service ...
# systemctl is-enabled myapp       # 验证：enabled

# 验证日志与重启
# tail -f /var/log/myapp.log       # 观察每 10 秒新增一行
# systemctl stop myapp && systemctl start myapp   # 手动重启验证
```

**原理**：`systemctl enable` 会在 `multi-user.target.wants/` 目录下创建指向 unit 的软链接，开机时 systemd 按该 target 的依赖拉起服务；`daemon-reload` 让 systemd 重新读取改动过的 unit 文件。

### 7.4 动手练习

1. 用 `systemctl status sshd` 查看 SSH 服务状态，解释 `Active` 和 `Loaded` 的区别。
2. `systemctl enable --now sshd` 一步完成"启用+立即启动"，验证 `is-enabled` 和 `is-active`。
3. 照 7.3 完整创建 `myapp.service`，改 `ExecStart` 为一个会报错的命令，观察 `Restart=always` 的自动重启现象。
4. 用 `journalctl -u myapp -f` 实时查看服务的 systemd 日志。

---

## 第 8 章 磁盘管理

### 8.1 分区：`fdisk` / `parted`

#### 8.1.1 `fdisk`（MBR 及小盘常用）

```bash
# fdisk -l                    # 列出所有磁盘和分区
# fdisk /dev/sdb              # 进入 /dev/sdb 的交互式分区
```

进入 `fdisk` 后按键：

| 键 | 作用 |
|----|------|
| `n` | 新建分区 |
| `p` | 主分区（primary） |
| `d` | 删除分区 |
| `t` | 改分区类型 |
| `w` | 保存并退出 |
| `q` | 不保存退出 |
| `m` | 帮助 |

完整例子（新建一个 Linux 分区并保存）：

```bash
# fdisk /dev/sdb
Command (m for help): n     # 新建
Partition type: p           # 主分区
Partition number (1-4): 1   # 第 1 个分区
First sector: (直接回车)      # 默认起始
Last sector: +10G           # 分 10G
Command (m for help): w     # 保存
```

#### 8.1.2 `parted`（大容量 GPT 盘，>2TB 必须用）

```bash
# parted /dev/sdb mklabel gpt              # 建 GPT 分区表（>2TB 必须 GPT）
# parted /dev/sdb mkpart primary 1MiB 10GiB   # 建一个 1MiB~10GiB 的分区
# parted /dev/sdb print                    # 查看分区表
# parted /dev/sdb rm 1                     # 删除第 1 个分区
```

**原理**：MBR 分区表最多 4 个主分区、最大 2TB；GPT 无此限制且支持更多分区，新盘一律推荐 GPT。

### 8.2 格式化：`mkfs`

```bash
# mkfs.xfs /dev/sdb1         # 格式化为 XFS（RHEL 9 默认文件系统）
# mkfs.ext4 /dev/sdb1        # 格式化为 ext4（通用，更老兼容）
# mkfs.xfs -f /dev/sdb1      # -f 强制（已有文件系统时）
```

**原理**：格式化=在分区上建立文件系统结构（超级块、inode 表、数据块）。XFS 擅长大文件、支持在线扩容，是 RHEL 9 默认；ext4 更通用、适合小文件与老系统。

### 8.3 挂载与卸载：`mount` / `umount`

```bash
# mkdir /data                      # 建挂载点（空目录）
# mount /dev/sdb1 /data            # 挂载分区到 /data
# df -hT /data                     # 验证挂载成功
Filesystem  Type  Size  Used Avail Use% Mounted on
/dev/sdb1   xfs    10G   33M  10G   1% /data

# umount /data                     # 卸载（卸载前确保无进程占用该目录）
# umount -l /data                  # 懒卸载（有占用时强制脱离）
```

**原理**：挂载把磁盘分区"接入"目录树，之后读写 `/data` 就是读写该分区。挂载点是普通目录，卸载后原目录内容恢复可见。

### 8.4 开机自动挂载：`/etc/fstab`

```bash
# blkid /dev/sdb1                 # 先查分区的 UUID
/dev/sdb1: UUID="a1b2c3d4-..." TYPE="xfs"

# 编辑 /etc/fstab，追加一行
# echo 'UUID=a1b2c3d4-...  /data  xfs  defaults  0 0' >> /etc/fstab

# 立即测试挂载（不重启验证 fstab 是否正确）
# mount -a
# df -hT /data
```

**fstab 字段逐列解释**：

| 字段 | 示例 | 含义 |
|------|------|------|
| 1 设备 | `UUID=...` 或 `/dev/sdb1` | 用什么挂（推荐 UUID，设备名可能变化） |
| 2 挂载点 | `/data` | 挂到哪 |
| 3 文件系统 | `xfs` | 类型 |
| 4 选项 | `defaults` | 挂载选项（rw,suid,dev,exec,auto,nouser,async） |
| 5 dump | `0` | 是否备份（0 不备份） |
| 6 fsck | `0` | 开机检查顺序（0 不检查，1 根分区，2 其他） |

> **风险提示**：fstab 写错会导致开机失败。改完务必先 `mount -a` 测试，能挂上再重启。

### 8.5 LVM 扩容全流程

LVM（逻辑卷管理）把物理盘抽象成"可动态伸缩"的逻辑卷，是服务器标配。

```bash
# 概念：PV(物理卷) <- VG(卷组) <- LV(逻辑卷)，从下往上层层聚合

# 1) 建 PV：把磁盘/分区标记为物理卷
# pvcreate /dev/sdb1
# pvs                          # 查看 PV

# 2) 建 VG：把 PV 加入卷组
# vgcreate vg_data /dev/sdb1
# vgs                          # 查看 VG

# 3) 建 LV：从 VG 划出逻辑卷
# lvcreate -L 5G -n lv_data vg_data
# lvs                          # 查看 LV

# 4) 格式化并挂载
# mkfs.xfs /dev/vg_data/lv_data
# mkdir /data && mount /dev/vg_data/lv_data /data

# 5) 扩容：加一块盘，扩展 VG，再扩展 LV
# pvcreate /dev/sdc1
# vgextend vg_data /dev/sdc1
# lvextend -L +5G /dev/vg_data/lv_data     # 或 -l +100%FREE 用完所有剩余空间

# 6) 让文件系统感知新空间（XFS 用 xfs_growfs，ext4 用 resize2fs）
# xfs_growfs /data
# df -h /data                    # 验证容量已变大
```

**原理**：`lvextend` 只是把 LV 的块数变多，文件系统还不知道，所以必须再 `xfs_growfs`（XFS）或 `resize2fs`（ext4）让文件系统用上新空间。XFS 支持在线扩容（挂载状态下直接扩）。

### 8.6 软链接 / 硬链接 与 inode

```bash
# 软链接（符号链接）
$ ln -s /usr/local/nginx /nginx           # 建软链接，类似 Windows 快捷方式
$ ls -l /nginx
lrwxrwxrwx 1 root root 15 Jul 20 10:00 /nginx -> /usr/local/nginx

# 硬链接
$ ln /etc/passwd /tmp/passwd_hard         # 建硬链接（同一 inode 的别名）
$ ls -li /etc/passwd /tmp/passwd_hard     # -i 显示 inode 号
  123456 -rw-r--r-- 2 root root 1890 ... /etc/passwd
  123456 -rw-r--r-- 2 root root 1890 ... /tmp/passwd_hard
# 注意两个文件 inode 号相同(123456)，链接数从 1 变 2
```

**软硬链接对比**：

| 特性 | 软链接 (ln -s) | 硬链接 (ln) |
|------|----------------|-------------|
| 本质 | 指向目标路径的"指针"文件 | 同一 inode 的另一个名字 |
| 跨文件系统 | 可以 | 不可以 |
| 指向目录 | 可以 | 不可以 |
| 目标删除后 | 链接变"悬空"(红色) | 内容仍存在（inode 计数未归零） |
| inode 号 | 与目标不同 | 与目标相同 |

**inode 原理**：inode 是文件的"身份证"，记录权限、属主、大小、数据块位置（不存文件名）。文件名存在目录项里，指向 inode。硬链接就是多个文件名指向同一 inode；软链接本身是独立 inode，内容是目标路径字符串。

```bash
# 查看 inode 使用情况
$ df -i
Filesystem   Inodes IUsed IFree IUse% Mounted on
/dev/sda3    5242880 12345 5230535  1% /
```

### 8.7 动手练习

1. 用 `lsblk` 列出本机磁盘结构，`df -hT` 看各分区文件系统类型。
2. 用 `fdisk` 或 `parted` 在新盘分一个 5G 分区，`mkfs.xfs` 格式化，挂载到 `/mnt/data`，`df -h` 验证。
3. 把该分区写入 `/etc/fstab`（用 UUID），`mount -a` 测试无误。
4. 完整走一遍 LVM：`pvcreate` → `vgcreate` → `lvcreate` → 格式化挂载 → `lvextend` → `xfs_growfs`，验证容量扩大。
5. 建一个软链接和一个硬链接，删掉原文件，分别观察两种链接的差异，并用 `ls -li` 对比 inode 号。

---

## 第 9 章 定时任务 crontab

### 9.1 是什么、为什么用

- **是什么**：cron 是 Linux 的定时任务守护进程，`crontab` 是配置它的命令，可让系统在指定时间自动执行命令。
- **为什么用**：备份、清日志、健康检查等重复性工作交给系统定时做，避免人工遗漏。
- **怎么用**：`crontab -e` 编辑任务，按五段式时间格式书写。

### 9.2 格式五段详解

```bash
# crontab -e      # 编辑当前用户的任务
# crontab -l      # 列出任务
# crontab -r      # 删除全部任务（慎用）
```

一条任务格式：

```
分 时 日 月 周  要执行的命令
*  *  *  *  *   /path/command
```

| 段 | 范围 | 含义 |
|----|------|------|
| 分 | 0-59 | 分钟 |
| 时 | 0-23 | 小时 |
| 日 | 1-31 | 每月的哪一天 |
| 月 | 1-12 | 月份 |
| 周 | 0-7 | 星期几（0 和 7 都代表周日） |

**符号**：`*` 任意值；`*/5` 每 5 单位；`1,15` 第 1 和第 15；`1-5` 1 到 5。

| 表达式 | 含义 |
|--------|------|
| `0 3 * * *` | 每天凌晨 3:00 |
| `*/5 * * * *` | 每 5 分钟 |
| `0 */2 * * *` | 每 2 小时整点 |
| `30 8 1 * *` | 每月 1 日 8:30 |
| `0 22 * * 5` | 每周五 22:00 |
| `0 0 1 1 *` | 每年 1 月 1 日 0:00 |

### 9.3 案例一：每天凌晨备份

```bash
# 建备份脚本
# cat > /usr/local/bin/backup.sh <<'EOF'
#!/bin/bash
DATE=$(date +%F)                     # 取当天日期，如 2026-07-20
tar -czf /backup/etc-$DATE.tar.gz /etc
# 只保留最近 7 天备份，删除更早的
find /backup -name "etc-*.tar.gz" -mtime +7 -delete
EOF
# chmod +x /usr/local/bin/backup.sh
# mkdir -p /backup

# 配置 crontab：每天凌晨 2 点备份
# crontab -e
# 添加一行：
0 2 * * * /usr/local/bin/backup.sh >> /var/log/backup.log 2>&1
```

**解释**：`>> /var/log/backup.log` 把标准输出追加到日志，`2>&1` 把错误也重定向到同一文件；`+7` 让 `find` 只匹配 7 天前修改的文件并删除，实现"滚动保留"。

### 9.4 案例二：每小时清日志

```bash
# crontab -e
# 添加：
0 * * * * find /var/log/myapp -name "*.log" -mtime +3 -delete
```

**解释**：每天整点，删除 `/var/log/myapp` 下超过 3 天的 `.log` 文件，防止日志撑爆磁盘。

### 9.5 排查 cron 不执行

```bash
# systemctl status crond          # 确认 cron 服务在运行
# systemctl enable crond          # 确认开机自启
# grep CRON /var/log/cron         # 查看 cron 是否真的触发了任务
$ journalctl -u crond -f          # 实时看 cron 日志
```

**常见坑**：① 命令要用绝对路径（cron 的 PATH 很有限）；② 脚本要有执行权限；③ 环境变量（如 PATH、日期）要自己在脚本里定义；④ `%` 在 crontab 里是特殊字符需转义。

### 9.6 动手练习

1. 用 `crontab -e` 加一条 `*/2 * * * * echo test >> /tmp/cron.log`，等 2 分钟 `cat /tmp/cron.log` 验证。
2. 写一个每 5 分钟记录当前时间到文件的脚本，并配好 cron。
3. 配置每天 23:30 备份 `/home` 到 `/backup`，并只保留最近 5 份。
4. 用 `crontab -l` 确认，用 `journalctl -u crond` 观察触发日志。

---

## 第 10 章 日志管理

### 10.1 是什么、为什么用

- **是什么**：Linux 把系统和服务运行信息写入 `/var/log` 下的日志文件，systemd 还提供统一的 `journalctl` 查询入口。
- **为什么用**：出问题时日志是第一手证据；安全审计也靠日志。
- **怎么用**：先 `ls /var/log` 找对应日志，再用 `tail -f`、`grep`、`journalctl` 分析。

### 10.2 `/var/log` 下常用日志

| 日志文件 | 记录内容 |
|----------|----------|
| `/var/log/messages` | 系统全局日志（内核、服务启动等） |
| `/var/log/secure` | 认证/安全（登录、sudo、ssh） |
| `/var/log/cron` | cron 定时任务执行记录 |
| `/var/log/boot.log` | 开机启动过程日志 |
| `/var/log/dmesg` | 内核启动消息（硬件探测） |
| `/var/log/maillog` | 邮件服务日志 |
| `/var/log/nginx/` | nginx 的 access/error 日志 |
| `/var/log/audit/audit.log` | SELinux/审计日志 |

```bash
$ tail -n 20 /var/log/secure        # 看最近登录/认证记录
$ grep "Failed password" /var/log/secure | tail    # 查暴力破解尝试
$ tail -f /var/log/messages         # 实时跟踪系统日志
```

### 10.3 `journalctl` 查询 systemd 日志

```bash
$ journalctl -xe            # 查看最近的日志，-x 展开 -e 跳到末尾（排错首选）
$ journalctl -f             # 实时跟踪（类似 tail -f）
$ journalctl -u nginx       # 只看 nginx 服务
$ journalctl -u sshd -n 50  # sshd 最近 50 条
$ journalctl --since "1 hour ago"       # 最近 1 小时
$ journalctl --since "2026-07-20 08:00" --until "2026-07-20 09:00"
$ journalctl -p err         # 只看错误级别(emerg/alert/crit/err)
$ journalctl -b             # 只看本次开机以来的日志
$ journalctl _PID=915       # 按进程 ID 过滤
$ journalctl --disk-usage   # 查看日志占用磁盘
```

**原理**：`journalctl` 读的是 systemd-journald 维护的二进制日志（`/run/log/journal` 或 `/var/log/journal`），比文本日志更结构化，支持按时间/服务/级别/进程精确过滤。

**限制日志大小（防止占满磁盘）**：

```bash
# journalctl --vacuum-size=500M     # 日志总量限制在 500M
# journalctl --vacuum-time=7d       # 只保留 7 天
# 永久配置：编辑 /etc/systemd/journald.conf
# SystemMaxUse=500M
# systemctl restart systemd-journald
```

### 10.4 动手练习

1. 用 `tail` 看 `/var/log/secure`，找一条登录成功和一条失败记录。
2. 故意输错一次密码登录，再用 `grep` 在 `/var/log/secure` 里找到该失败记录。
3. 用 `journalctl -u sshd -n 20` 看 sshd 最近日志，`journalctl -xe` 看系统最近事件。
4. `journalctl --since "1 hour ago" -p err` 找出最近 1 小时内的错误日志。
5. 用 `journalctl --vacuum-size=200M` 限制日志占用。

---

## 本章项目验收：搭一台标准化服务器（从零到完成）

> 目标：把一台全新的 Rocky Linux 9 / AlmaLinux 9 配置成可上线的标准化服务器。
> 验收清单：普通用户 + sudo、静态 IP、国内源、防火墙只放 22/80/443、装 nginx 可访问、crontab 每小时清日志。
> 每一步都可照抄，做完逐项核对。

### 步骤 1：创建普通用户并配置 sudo

```bash
# useradd -m -s /bin/bash -G wheel ops
# passwd ops
# （输入两次密码）

# 验证 wheel 组 sudo 权限（visudo 里 %wheel ALL=(ALL) ALL 已启用）
$ su - ops
$ sudo whoami          # 应输出 root，首次会要求输入 ops 的密码
```

### 步骤 2：配置静态 IP

先看网卡名和当前网络：

```bash
$ ip addr              # 记下网卡名，如 ens160
$ ip route             # 记下网关，如 192.168.1.1
```

Rocky/Alma 9 使用 NetworkManager，用 `nmcli` 配静态 IP（假设 IP=192.168.1.100/24，网关 192.168.1.1，DNS 223.5.5.5）：

```bash
# 查看当前连接名
# nmcli connection show
NAME   UUID                                  TYPE      DEVICE
ens160 1234...                              ethernet  ens160

# 设静态 IP（连接名按实际填写）
# nmcli connection modify ens160 ipv4.addresses 192.168.1.100/24
# nmcli connection modify ens160 ipv4.gateway 192.168.1.1
# nmcli connection modify ens160 ipv4.dns 223.5.5.5
# nmcli connection modify ens160 ipv4.method manual
# nmcli connection up ens160        # 重新激活连接使配置生效

# 验证
$ ip addr show ens160      # 应看到 inet 192.168.1.100/24
$ ping -c 2 192.168.1.1    # 网关通
$ ping -c 2 223.5.5.5      # 外网通
```

> 传统方式也可直接编辑 `/etc/NetworkManager/system-connections/ens160.nmconnection` 或 `/etc/sysconfig/network-scripts/ifcfg-ens160`，但 nmcli 更标准。

### 步骤 3：配置国内 yum 源

按第 6.3 节完整步骤操作（阿里云或清华源）：

```bash
# mkdir /etc/yum.repos.d/backup
# mv /etc/yum.repos.d/*.repo /etc/yum.repos.d/backup/
# cat > /etc/yum.repos.d/rocky.repo <<'EOF'
[baseos]
name=Rocky Linux $releasever - BaseOS (Aliyun)
baseurl=https://mirrors.aliyun.com/rockylinux/$releasever/BaseOS/$basearch/os/
gpgcheck=1
enabled=1
gpgkey=https://mirrors.aliyun.com/rockylinux/RPM-GPG-KEY-Rocky-9

[appstream]
name=Rocky Linux $releasever - AppStream (Aliyun)
baseurl=https://mirrors.aliyun.com/rockylinux/$releasever/AppStream/$basearch/os/
gpgcheck=1
enabled=1
gpgkey=https://mirrors.aliyun.com/rockylinux/RPM-GPG-KEY-Rocky-9

[extras]
name=Rocky Linux $releasever - Extras (Aliyun)
baseurl=https://mirrors.aliyun.com/rockylinux/$releasever/extras/$basearch/os/
gpgcheck=1
enabled=1
gpgkey=https://mirrors.aliyun.com/rockylinux/RPM-GPG-KEY-Rocky-9
EOF

# dnf clean all
# dnf makecache
# dnf repolist          # 确认三个阿里云仓库已启用
```

### 步骤 4：配置防火墙只放 22/80/443

RHEL 9 用 firewalld：

```bash
# systemctl enable --now firewalld          # 启动并自启防火墙
# firewall-cmd --state                      # 确认 running
# firewall-cmd --get-default-zone           # 查看默认区域（public）

# 放行端口（--permanent 永久生效）
# firewall-cmd --permanent --add-service=ssh      # 放行 22
# firewall-cmd --permanent --add-service=http     # 放行 80
# firewall-cmd --permanent --add-service=https    # 放行 443
# firewall-cmd --reload                          # 重载使配置生效

# 查看已放行的端口（验证只有 22/80/443）
# firewall-cmd --list-all
public (active)
  target: default
  services: cockpit dhcpv6-client http https ssh
  ports:
  ...
```

> 说明：`cockpit`、`dhcpv6-client` 是默认服务，可保留；若想极简可用 `--remove-service` 去掉。核心是确认 `http https ssh` 在列表里，且没有多余端口。

### 步骤 5：安装并验证 nginx 可访问

```bash
# dnf install -y nginx
# systemctl enable --now nginx          # 启动并自启
# systemctl status nginx                # active (running)

# 本机验证（返回 200 和 HTML 即成功）
$ curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1
200

# 从浏览器访问 http://192.168.1.100   （应看到 nginx 默认欢迎页）

# 若访问不了，检查防火墙是否放行 80、SELinux 是否拦截：
# getenforce                            # 查看 SELinux 状态
# 若为 Enforcing 且被拦，放行 http：
# setsebool -P httpd_can_network_connect on   # 允许 nginx 联网（如需反向代理）
```

### 步骤 6：配置 crontab 每小时清日志

```bash
# 建日志清理脚本
# cat > /usr/local/bin/cleanlog.sh <<'EOF'
#!/bin/bash
find /var/log/nginx -name "*.log" -mtime +3 -delete
journalctl --vacuum-time=7d > /dev/null 2>&1
EOF
# chmod +x /usr/local/bin/cleanlog.sh

# 配置 root 的 crontab
# crontab -e
# 添加一行（每小时整点执行）：
0 * * * * /usr/local/bin/cleanlog.sh

# 验证
# crontab -l                          # 确认任务已写入
# journalctl -u crond | tail          # 确认 crond 在运行
```

### 最终验收清单（逐项打勾）

| # | 验收项 | 验证命令 | 期望结果 |
|---|--------|----------|----------|
| 1 | 普通用户+sudo | `su - ops` 后 `sudo whoami` | 输出 `root` |
| 2 | 静态 IP | `ip addr show ens160` | 看到设定的 `192.168.1.100/24` |
| 3 | 国内源 | `dnf repolist` | 显示阿里云/清华仓库 |
| 4 | 防火墙只放 22/80/443 | `firewall-cmd --list-all` | services 含 http https ssh |
| 5 | nginx 可访问 | 浏览器打开 `http://192.168.1.100` | 显示 nginx 欢迎页 |
| 6 | 每小时清日志 | `crontab -l` | 看到 cleanlog.sh 的 `0 * * * *` 任务 |

全部打勾，即完成一台标准化服务器的搭建。恭喜你完成阶段 1 全部内容！

---

## 附：快速命令速查表

| 类别 | 命令 |
|------|------|
| 文件 | `ls -lh` `cp -rp` `mv` `rm -rf` `mkdir -p` `touch` |
| 文本 | `cat` `less` `head -n` `tail -f` |
| 查找 | `find / -name "*.log" -mtime +7` `which` `whereis` |
| 统计 | `grep -rn` `wc -l` `sort -t: -k3 -n` `uniq -c` `cut -d: -f1` `tr` |
| 打包 | `tar -czvf` `tar -xzvf` `gzip` `zip -r` |
| 进程 | `ps aux` `top` `htop` `kill -9` |
| 磁盘 | `df -h` `du -sh` `free -h` `lsblk` `mount` |
| 权限 | `chmod 755` `chmod u+x` `chown -R user:grp` `umask` |
| 用户 | `useradd -m -G wheel` `usermod -aG` `passwd` `visudo` |
| 网络 | `ip addr` `ss -tlnp` `ping -c4` |
| 远程 | `ssh` `scp -r` `rsync -avz` |
| 下载 | `wget -O` `curl -o` |
| 包管理 | `dnf install` `rpm -qa` |
| systemd | `systemctl start/enable/status` `journalctl -u` |
| 定时 | `crontab -e` `crontab -l` |
| 日志 | `journalctl -xe` `tail -f /var/log/messages` |
