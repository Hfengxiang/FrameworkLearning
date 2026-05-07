# Android Framework 开发学习蓝图

> *"9年应用开发不是负担，是最大的优势——你知道'正常'应该是什么样的，这让你学底层时能不断发出'原来如此'的感叹。"*

---

## 1. 现状评估

| 维度 | 当前水平 | 目标水平 |
|------|---------|---------|
| Java/Kotlin | 9年应用开发，烂熟 | 保持，Framework 服务层主要用 Java |
| AOSP 源码 | 未系统读过，只在博客看过 | 能独立跟踪代码路径、定位问题 |
| Binder 机制 | 知道"通过内核做中转"，BP/BN 细节不了解 | 完全搞懂四大角色，能讲给外行听 |
| C/C++ | 曾经系统学过，长期不用基本遗忘 | 能看懂 Binder 驱动和 native 服务的关键代码 |
| Linux | 基本命令和提权操作 | 理解进程模型、内存管理、SELinux 基础 |
| 系统服务 (AMS/WMS等) | 只知道 API 怎么调用 | 能定制和维护 |

---

## 2. 第一性原理分析：Framework 到底在开发什么？

之前面对的世界：

```
App → Android SDK API → [黑盒] → 屏幕显示
```

钻进去后的世界：

```
┌─ App（Java/Kotlin）──────────────────┐  ← 已在这里
├─ Framework 服务层（AMS/WMS/PMS...）───┤  ← 要来这里
├─ Native 服务层（SurfaceFlinger等）────┤  ← 顺手了解
├─ HAL 层（硬件抽象）───────────────────┤  ← 知道存在即可
├─ Linux Kernel（Binder/驱动/内存）─────┘  ← 需要懂一些
```

**核心转变：不是调用系统，是系统调用你写的服务；不是处理点击事件，是处理跨进程的生死调度。思维模型要变。**

---

## 3. 学习计划蓝图

### 积木 0：环境与工具 —— 先搭工作台

在学习原理之前，先把"看源码"这件事变简单。

- [ ] 下载 AOSP 源码（至少 `frameworks/base/` 核心目录到本地）
- [ ] 学会全文搜索源码（grep / rg / Android Studio 导入源码目录）
- [ ] 搭建编译环境，学会 `mmm` 编译单个模块并推到设备验证
- [ ] **动手实验**：在 `ActivityManagerService` 里加一行 Log，编译，推到手机，看它生效

> **费曼说：别读了不动手。你看 100 篇博客不如自己改一行代码然后推到手机上看到效果。**

---

### 积木 1：Binder —— 必须先搞懂的"血液循环系统"

Binder 是 Android Framework 的血液循环系统。不理解它，AMS/WMS/PMS 永远是一堆看不懂的代码。

**学习路径：**

- [ ] **1.1** 画清楚"内核中转"的图景：两个进程无法直接通信 → Binder 驱动在内核开"公告板" → 数据从进程 A 的用户空间拷到内核空间，再拷到进程 B
- [ ] **1.2** 搞懂 Binder 四个角色：
  - Client（调用方，如 App）
  - Server（服务方，如 AMS）
  - ServiceManager（"电话簿"，Client 通过它找到 Server）
  - Binder Driver（内核"快递员"）
- [ ] **1.3** 手写 AIDL Binder 通信 demo，读自动生成的代码：
  - `Stub` = BnInterface（BnBinder 侧）
  - `Proxy` = BpInterface（BpBinder 侧）
  - `transact()` 和 `onTransact()` —— 数据的"发送"和"接收"
- [ ] **1.4** 读 Binder 驱动关键函数 `binder_transaction`：数据从发送进程用户空间 → 内核空间 → 接收进程用户空间。**只拷一次，这是 Binder 高效的核心。**

> **检验标准：能不能不用"IPC""跨进程通信"这些术语，用日常语言给外行人讲清楚 Binder 干了什么？**

---

### 积木 2：系统服务启动过程 —— 知道"舞台"怎么搭起来的

- [ ] 读 `SystemServer.java`：Android 框架层的"大脑"，所有 Java 层系统服务由它启动
- [ ] 画出系统服务启动顺序图（哪些先启动？哪些依赖哪些？）
- [ ] 理解 `addService` 和 `getService` 的完整调用链
  - 服务启动 → `ServiceManager.addService()` 注册到"电话簿"
  - App 调用 → `ServiceManager.getService()` 拿到 Binder 代理

---

### 积木 3：AMS 深度解析 —— 第一个"目标猎物"

**从你熟悉的地方出发：**

```
startActivity()
  → Instrumentation.execStartActivity()
    → ActivityTaskManager.getService().startActivity()  // ← 这里跨了 Binder！
      → ActivityTaskManagerService.startActivity()
```

- [ ] 跟踪 `startActivity()` 从 App 到 AMS 的完整调用链
- [ ] 理解 AMS 核心数据结构：
  - `ActivityRecord`：一个 Activity 在系统里的"档案"
  - `Task`：Activity 的任务栈
  - `ProcessRecord`：一个进程的"档案"
- [ ] **关键技巧**：不要一次读完 AMS 几万行代码。只追一个场景——"点桌面图标到 MainActivity 显示"——把这个场景完全搞懂，AMS 架构自然在你脑中

---

### 积木 4：WMS —— 窗口是怎么画出来的

AMS 管"哪个 Activity 在跑"，WMS 管"屏幕上哪个区域画什么"。

- [ ] 从 `addView()` / `WindowManager.addView()` 往下追调用链
- [ ] 理解核心数据结构：`WindowToken`、`WindowState`
- [ ] 理解 Surface 概念：每个窗口对应一个 Surface，SurfaceFlinger 负责合成

---

### 积木 5：补充基础 —— Linux 进程与内存模型

Framework 服务层主要是 Java，但底层机制的理解需要这些基础。**学到哪个模块需要了，回头补一点，不要先花三个月学 C++ 再回来。**

| 知识点 | 为什么需要 | 深度要求 |
|--------|-----------|---------|
| Linux 进程模型 | AMS 本质是管理进程 | 能讲清楚 fork 和 Zygote 的关系 |
| 内存管理基础 | 理解 LMK 回收策略 | 知道虚拟内存、mmap 干什么 |
| C++ 基础 | 读 Binder 驱动、native 服务 | 能看懂关键函数即可 |
| SEAndroid/SELinux | 系统服务有权限控制 | 知道 context 是什么，权限怎么配 |

---

### 积木 6：其他模块 —— "知其所以然"

PMS、InputManagerService、AlarmManagerService...它们的骨架都一样：

> **注册到 ServiceManager 的系统服务，通过 Binder 给 App 或其他服务提供接口。**

学会了 AMS 和 WMS，其他模块就是看具体的业务逻辑。

---

## 4. 学习节奏

假设每天 **2-3 小时**：

| 阶段 | 时间 | 内容 |
|------|------|------|
| 第 1-2 周 | 环境搭建 + Binder 彻底搞懂 + 手写 Demo | 积木 0、1 |
| 第 3-4 周 | SystemServer 启动流程 + AMS 核心场景跟踪 | 积木 2、3 |
| 第 5-6 周 | AMS 深入 + WMS 核心场景 | 积木 3、4 |
| 第 7-8 周 | 其他模块了解 + 尝试改一个小功能 | 积木 5、6 |

**两个月左右达到面试能聊的水平：**
- 讲清楚 AMS 怎么管理 Activity
- 讲清楚 Binder 通信的完整链路
- 讲清楚系统服务怎么启动的

---

## 5. 费曼式提醒

1. **不要相信博客能教会你。** 博客是二手信息。直接读 AOSP 源码。源码不会骗你。
2. **遇到不懂的词，追查到底。** 每次遇到新词，问自己：我能用日常语言解释这个东西是干什么的吗？
3. **动手比动眼快十倍。** 别光看代码，改它。哪怕只是加一行 Log。
4. **你不笨，你只是需要换一种学法。** 9 年应用开发给了你一个巨大优势——当看到 Framework 代码时，你会不断有"哦！原来我调的那个 API 背后是这样干的！"的体验。这会让你学得比零基础的人快得多。

---

> *"每条路都不一样。找到你的路，然后走到底。"* — 费曼
