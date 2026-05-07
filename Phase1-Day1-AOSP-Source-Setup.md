# Day 1：获取 AOSP 源码 & 搭建搜索环境

> **今天的核心目标**：在本机能搜到、读到 Framework 源码。就这一件事。做到了，你今天就算成功。

---

## 0. 为什么先搞这个？

你可能会想："我还没学 Binder 呢，先把源码下下来有什么用？光看也看不懂。"

**不对。** 这就跟你说"先学会开车再看仪表盘"一样——你得先把仪表盘装好，车开起来了随时能看。

整个学习过程中，你会反复的动作就是：**搜索 → 打开文件 → 阅读 → 跟踪调用链**。如果每次都要等网页加载、每次都要猜文件在哪，你的学习效率会低十倍。今天花半天把这件事搞定，后面每一天都受益。

---

## 1. 环境选择

Android Framework 开发的主力环境是 Linux。你在 Windows 上，有两条路：

| 方案 | 优点 | 缺点 | 推荐度 |
|------|------|------|--------|
| **WSL2 + 国内镜像下载源码** | 本地搜索快、能编译、完全离线可用 | 需要装 WSL2，下载 5-15GB | 强烈推荐 |
| **纯在线浏览（国内镜像站）** | 零安装，打开浏览器就能看 | 不能搜索，翻页慢，不能编译 | 临时过渡 |

> **重要前提**：本指南默认你**无法访问 Google 服务器**。所有 Google 源地址（android.googlesource.com、storage.googleapis.com、cs.android.com）均已替换为国内可访问的镜像站。

> 推荐走 **WSL2 + 国内镜像下载** 路线。纯在线方案（第 7 节）只作为临时过渡，不适合长期学习。

---

## 2. 安装 WSL2（如果还没有）

### Step 1：用一条命令安装

打开 **PowerShell（管理员模式）**，执行：

```powershell
wsl --install -d Ubuntu-24.04
```

这个命令会：
- 启用 WSL2 虚拟化功能
- 下载并安装 Ubuntu 24.04
- 让你设置一个 Linux 用户名和密码

安装完成后重启电脑。

### Step 2：验证安装

打开 PowerShell，执行：

```powershell
wsl -l -v
```

应该看到：

```
NAME            STATE           VERSION
Ubuntu-24.04    Running         2
```

如果 VERSION 显示 1，执行：

```powershell
wsl --set-version Ubuntu-24.04 2
```

### Step 3：进入 WSL

```powershell
wsl
```

你就进入了一个 Ubuntu 24.04 的终端。后续所有命令都在这里面执行。

### Step 4：基础配置（5 分钟）

```bash
# 更新包管理器
sudo apt update && sudo apt upgrade -y

# 安装基础工具（后面都会用到）
sudo apt install -y git curl wget unzip vim tree

# 确认一切正常
git --version   # 应输出 git version 2.x.x
python3 --version  # 应有 Python 3
```

> **为什么装 python3？** AOSP 的 repo 工具是 Python 脚本，依赖 Python 3。

---

## 3. 安装 repo 工具

`repo` 是 Google 开发的批量 Git 仓库管理工具。AOSP 由几百个 Git 仓库组成，你不可能手动一个个克隆，必须用 repo。

### Step 1：创建 bin 目录并下载 repo

> Google 官方的 `storage.googleapis.com` 在国内无法访问，使用清华镜像下载 repo 工具。

```bash
mkdir -p ~/bin
curl https://mirrors.tuna.tsinghua.edu.cn/git/git-repo > ~/bin/repo
chmod a+x ~/bin/repo

# 设置 repo 源为清华镜像（后续 repo init/sync 都会走这个镜像）
export REPO_URL='https://mirrors.tuna.tsinghua.edu.cn/git/git-repo'
echo 'export REPO_URL="https://mirrors.tuna.tsinghua.edu.cn/git/git-repo"' >> ~/.bashrc
```

### Step 2：把 ~/bin 加到 PATH

```bash
echo 'export PATH="$HOME/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

### Step 3：验证

```bash
repo --version
```

应该输出版本号。如果没有，检查两步：
1. `ls -la ~/bin/repo` 看文件是否存在
2. `echo $PATH | grep bin` 看 `~/bin` 是否在 PATH 中

---

## 4. 下载 AOSP 源码（精简版）

**完整 AOSP 源码约 200GB+。我们不需要全部。** 第一阶段只需要framework 核心模块，约 15-20GB。

### Step 1：创建工作目录

```bash
mkdir -p ~/aosp
cd ~/aosp
```

### Step 2：初始化 repo（清华 AOSP 镜像）

```bash
repo init -u https://mirrors.tuna.tsinghua.edu.cn/git/AOSP/platform/manifest \
  -b android-14.0.0_r1 \
  --depth=1
```

参数解释：

| 参数 | 含义 |
|------|------|
| `-u https://mirrors.tuna.tsinghua.edu.cn/...` | 使用**清华 AOSP 镜像**，国内可直接访问 |
| `-b android-14.0.0_r1` | Android 14 的第一个正式发布版本，稳定且文档多 |
| `--depth=1` | **浅克隆**：只下载最新一次 commit 记录，大幅减少下载量和时间 |

> **备选镜像**：如果清华镜像慢或不稳定，可以换中科大镜像：
> ```bash
> repo init -u https://mirrors.ustc.edu.cn/aosp/platform/manifest -b android-14.0.0_r1 --depth=1
> ```

### Step 3：只同步核心模块

```bash
repo sync -c -j8 \
  frameworks/base \
  frameworks/native \
  system/core \
  libcore \
  art \
  bionic \
  build \
  external/protobuf
```

模块说明：

| 模块 | 内容 | 为什么需要 |
|------|------|-----------|
| `frameworks/base` | **核心**：AMS、WMS、PMS 等所有 Java 系统服务都在这里 | 每天都要看 |
| `frameworks/native` | **核心**：Binder C++ 实现（BnBinder/BpBinder、Parcel、IPCThreadState） | Binder 深入必备 |
| `system/core` | init 进程、adb、logcat、servicemanager | 理解系统启动和服务注册 |
| `libcore` | Java 核心库（和 Framework 层交互频繁） | 理解 Java 层的边界 |
| `bionic` | Android 的 C 标准库 | 理解 C 层的系统调用 |
| `build` | 编译系统（包含 mmm 等编译命令） | 后面编译用 |

**预计下载量**：约 8-15GB，取决于网络速度，可能需要 30 分钟到 2 小时。

> 下载时可以继续做第 5 节和第 6 节的准备工作。

---

## 5. 安装搜索工具

源码下载完后，你需要快速找到你要的东西。不要用 Windows 资源管理器搜索——那太慢了。

### 先确保 apt 走国内镜像（快 10 倍）

```bash
sudo sed -i 's/archive.ubuntu.com/mirrors.tuna.tsinghua.edu.cn/g' /etc/apt/sources.list.d/ubuntu.sources 2>/dev/null
sudo sed -i 's/security.ubuntu.com/mirrors.tuna.tsinghua.edu.cn/g' /etc/apt/sources.list.d/ubuntu.sources 2>/dev/null
sudo apt update
```

> 如果是旧版 Ubuntu（sources.list 格式），改为：
> ```bash
> sudo sed -i 's/archive.ubuntu.com/mirrors.tuna.tsinghua.edu.cn/g' /etc/apt/sources.list
> sudo apt update
> ```

### 方案 A：ripgrep（rg）—— 推荐

ripgrep 是目前最快的文本搜索工具，秒搜几十 GB 的代码。

```bash
sudo apt install -y ripgrep
```

安装后验证：

```bash
cd ~/aosp/frameworks/base
rg "class ActivityManagerService" --type java
```

应该能搜到 AMS 的类定义。

### 方案 B：grep 备选

```bash
# grep 也够用，就是慢一点
grep -r "class ActivityManagerService" ~/aosp/frameworks/base --include="*.java"
```

### 方案 C：Android Studio 导入源码目录（可选）

如果你更习惯 IDE 看代码：

1. 打开 Android Studio
2. File → New → Import Project
3. 选择 `\\wsl$\Ubuntu-24.04\home\<你的用户名>\aosp\frameworks\base`
4. 只作为 Gradle 项目导入（不编译）
5. 导入完成后，双击 Shift 搜索任意类名

> Android Studio 导入 AOSP 源码目录会有些卡，但搜索和跳转比命令行方便。建议 ripgrep + Android Studio 配合使用。

---

## 6. 验证环境 —— 找到 5 个标志性文件

到这一步，你的机器上应该有了一份可搜索的 AOSP 源码。现在利用它，**亲手找到以下 5 个文件**，每个看一眼，心里有个印象：

### 文件 1：AMS 的"家"

```bash
# 搜索
rg "public class ActivityManagerService" ~/aosp/frameworks/base --type java -l
```

预期结果：`services/core/java/com/android/server/am/ActivityManagerService.java`

打开它，拉到最上面，看一眼 import 和 class 声明。不用看懂，就是混个脸熟。

### 文件 2：Binder 的 Java 层基类

```bash
rg "public class Binder" ~/aosp/frameworks/base --type java -l | grep "android/os/Binder.java"
```

打开 `frameworks/base/core/java/android/os/Binder.java`，找到 `transact()` 方法和 `onTransact()` 方法——这是 Java 层 Binder 通信的两个核心入口。

### 文件 3：Binder 的 C++ 层

```bash
find ~/aosp/frameworks/native -name "Binder.cpp"
```

打开 `frameworks/native/libs/binder/Binder.cpp`，找到 `BBinder::onTransact()`。

### 文件 4：ServiceManager——电话簿

```bash
rg "class ServiceManager" ~/aosp/frameworks/base --type java -l
```

打开 `frameworks/base/core/java/android/os/ServiceManager.java`，找到 `getService()` 和 `addService()` 方法。

### 文件 5：Binder 驱动（在线查看）

```bash
# 国内镜像站可在线看内核源码，保存到书签
echo "https://android-opengrok.bangnimang.net/source/s?path=binder.c&project=common-android14-6.1"
echo "备选：https://aosp.opersys.com/xref/common-android14-6.1/drivers/android/binder.c"
```

收藏其中一个链接。这就是 Binder 驱动的核心文件。Day 8 会用到。

---

## 7. 纯在线方案（临时过渡）

**先说结论：在没有 VPN 的情况下，纯在线浏览 AOSP 源码比较折腾。** Google 官方的 `cs.android.com` 无法访问，国内也没有完美的镜像浏览站。

如果你暂时不想装 WSL2，有以下过渡方案：

### 方案 A：Android SDK 自带的 sources（你已经有了）

你做了 9 年 Android 开发，电脑上一定有 Android SDK。SDK 里就带着部分 Framework 源码：

```
<Android SDK 路径>/sources/android-<API 版本>/
```

例如：
- `android-34/` → Android 14 的部分 Framework 源码

**操作**：在 Android Studio 里按住 Ctrl 点击任何 SDK 类名（比如 `Activity`、`ServiceManager`），就能看到源码。

**局限**：
- 只有公开 SDK 类的源码，**没有服务端实现**（比如你看不到 `ActivityManagerService.java`，因为 App 不直接引用它）
- 没有 native 层代码
- 没有 Binder 驱动代码

> 这个方案只能作为 Day 1 的临时浏览，后续学习**必须有本地完整源码**。

### 方案 B：尝试国内 AOSP 代码浏览镜像

以下镜像站可能可以从国内直接访问（不保证都可用）：

| 镜像站 | URL | 说明 |
|--------|-----|------|
| Android 开源项目镜像 | `https://mirrors.tuna.tsinghua.edu.cn/help/AOSP/` | 清华 AOSP 帮助页，指引如何下载 |
| Opersys AOSP XRef | `http://aosp.opersys.com/` | 社区维护的 AOSP 代码交叉引用 |

> 这些镜像站可能不稳定，且 URL 可能变化。**强烈建议走下面的"最小化本地下载"方案。**

### 方案 C：最小化本地下载（无需 20GB，只需 2 步）

如果你怕占磁盘空间，可以只在 WSL2 中下载**两个最核心模块**，总共约 5-8GB：

```bash
# 步骤 1：初始化（2分钟）
export REPO_URL='https://mirrors.tuna.tsinghua.edu.cn/git/git-repo'
mkdir -p ~/aosp && cd ~/aosp
repo init -u https://mirrors.tuna.tsinghua.edu.cn/git/AOSP/platform/manifest \
  -b android-14.0.0_r1 --depth=1

# 步骤 2：只下载 frameworks/base 和 frameworks/native（5-8GB，30分钟-1小时）
repo sync -c -j8 frameworks/base frameworks/native build
```

这两个模块包含：
- `frameworks/base`：所有 Java 系统服务（AMS、WMS、PMS...）
- `frameworks/native`：Binder C++ 实现

> **Binder 驱动代码**（`binder.c`）这两个模块里没有，但你可以直接在 GitHub 或 Gitee 上搜索 `binder.c` 或 `binder_transaction` 找到镜像。Day 8 会用到时再处理。

---

## 8. 常见问题排查

### Q1：repo init 失败，报 network error

```bash
# 确认使用的是清华镜像而非 Google 源（Google 源在国内无法访问）
repo init -u https://mirrors.tuna.tsinghua.edu.cn/git/AOSP/platform/manifest -b android-14.0.0_r1 --depth=1

# 如果清华镜像也失败，换中科大镜像
repo init -u https://mirrors.ustc.edu.cn/aosp/platform/manifest -b android-14.0.0_r1 --depth=1
```

### Q2：repo sync 下载很慢

```bash
# 确认 REPO_URL 环境变量已设为清华镜像
export REPO_URL='https://mirrors.tuna.tsinghua.edu.cn/git/git-repo'

# 重新 init 并 sync
repo init -u https://mirrors.tuna.tsinghua.edu.cn/git/AOSP/platform/manifest -b android-14.0.0_r1 --depth=1
repo sync -c -j8 frameworks/base frameworks/native system/core libcore art bionic build
```

### Q3：磁盘空间不足

如果磁盘不够 20GB，只下 `frameworks/base` 和 `frameworks/native`：

```bash
repo sync -c -j8 frameworks/base frameworks/native
```

这两个模块加起来约 5-8GB，足够第一、二阶段学习。

### Q4：WSL2 找不到 Windows 的磁盘

WSL2 中，Windows 的 C 盘在 `/mnt/c/`，D 盘在 `/mnt/d/`。如果你想把 AOSP 源码放 D 盘：

```bash
mkdir -p /mnt/d/aosp
ln -s /mnt/d/aosp ~/aosp   # 软链接到 home 目录
```

### Q5：ripgrep 安装失败

```bash
# 方式 A：用 apt
sudo apt install -y ripgrep

# 方式 B：用 cargo（Rust 的包管理器）
sudo apt install -y cargo
cargo install ripgrep

# 方式 C：直接用 grep（慢一点但也能用）
alias rg="grep -rn"
```

---

## 9. Day 1 完成标准

在睡觉之前，检查以下三项：

- [ ] **能搜到**：在源码目录中执行 `rg "ActivityManagerService" --type java` 能搜出结果
- [ ] **能找到**：你能在 10 秒内找到下列文件中的至少 3 个：
  - `ActivityManagerService.java`
  - `Binder.java`
  - `ServiceManager.java`
  - `Binder.cpp`（native 层）
  - `binder.c`（驱动层，在线看也行）
- [ ] **能打开**：随便打开一个文件（比如 `ServiceManager.java`），找到 `getService()` 方法，看一眼代码

> **今天就这些。不要贪多。** 明天 Day 2 你要开始编译环境搭建，如果你今天已经把源码和搜索搞定，明天会非常顺畅。

---

## 10. 延伸：今晚睡前可以想的一个问题

打开 `frameworks/base/core/java/android/os/ServiceManager.java`，找到 `getService(String name)` 方法。

方法体大概长这样：

```java
public static IBinder getService(String name) {
    // ... 
    IBinder service = sCache.get(name);
    if (service != null) {
        return service;
    }
    // ...
    return getIServiceManager().getService(name);
}
```

**睡前思考**：这个方法返回的 `IBinder` 对象到底是什么？它是一个真正的 AMS 对象在你的进程里吗？如果不是，它是什么？

> 不用回答，想就行了。Day 4 你做 AIDL demo 的时候，答案会自己浮出来。带着这个问题入睡，你的大脑会在后台帮你整理信息。

---

> *"知道一个东西的名字不等于知道它是什么。但如果你连它在哪都不知道，你连问问题都无从问起。"*
> — 费曼老爹的鸟的教导，包装为开发者版本
