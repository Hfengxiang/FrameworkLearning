# 第一阶段：环境搭建 & Binder 彻底搞懂

> **目标**：用两周时间，从"Binder 大概是个通过内核传信的东西"到"能自己写 Binder 通信、能看明白 Binder 驱动核心代码、能给别人讲清楚 Binder 的本质"。

---

## 1. 阶段目标清单

完成本阶段后，你应当能够：

- [ ] 在本机搜索和阅读 AOSP `frameworks/base/` 源码
- [ ] 编译单个 Framework 模块并推到设备/模拟器上验证
- [ ] 手写 AIDL Binder 通信 demo，并讲清楚 `Stub`/`Proxy` 是什么
- [ ] 画出 Binder 通信的完整数据流图（从 App 调用到 AMS 收到）
- [ ] 用日常语言（不用"IPC"、"跨进程通信"等术语）给外行人讲清楚 Binder 干了什么
- [ ] 跟踪一个真实的 Framework Binder 调用链（如 `startActivity()` 中的 `getService()`）

---

## 2. 前置准备

### 2.1 硬件/软件要求

| 项目 | 要求 |
|------|------|
| 磁盘空间 | 至少 100GB 空闲（完整 AOSP 源码约 200GB+，只拉核心模块约 20GB） |
| 内存 | 建议 16GB+（编译时需要） |
| 操作系统 | Linux（推荐 Ubuntu 20.04+）或 macOS |
| Android 设备 | 一台可刷机的 Pixel 设备或模拟器 |

> 如果没有 Linux 环境，优先用 WSL2（Windows Subsystem for Linux）搭建。

### 2.2 参考资料清单

| 资料 | 用途 | 链接 |
|------|------|------|
| AOSP 源码 | 核心学习材料 | https://cs.android.com/ 或本地源码 |
| Binder 驱动源码 | 读 `binder_transaction` | `kernel/common/drivers/android/binder.c` |
| 《Android Binder 分析与实战》| Binder 体系化讲解 | 参考书籍，选读 |
| AIDL 官方文档 | 写 Demo 参考 | https://developer.android.com/develop/background-work/services/aidl |

---

## 3. 第一周：环境搭建 + Binder 骨架

### Day 1：获取源码 & 搭建搜索环境

**目标**：在本机能搜到、读到 Framework 源码。

**任务：**

```
# 方案 A：只下载核心目录（推荐，快速）
mkdir -p ~/aosp && cd ~/aosp
repo init -u https://android.googlesource.com/platform/manifest -b android-14.0.0_r1 --depth=1
# 只同步 frameworks/base/ 和 system/core/ 等核心目录
repo sync -c frameworks/base system/core libcore art

# 方案 B：在线看（如果不能下载）
# 直接用 https://cs.android.com/ 搜索，关键字后面加 f:base
```

**验证标准**：能在本地（或 cs.android.com）搜到 `ActivityManagerService` 的源码位置。

**具体行动：**

- [ ] 安装 repo 工具
- [ ] 同步 `frameworks/base/` 到本地（或确认 cs.android.com 可访问）
- [ ] 在源码中找到以下文件并打开看一遍目录结构：
  - `frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java`
  - `frameworks/base/core/java/android/app/ActivityManager.java`
  - `frameworks/base/core/java/android/os/Binder.java`

---

### Day 2：搭建编译 & 调试环境

**目标**：能改一行 Framework 代码、编译、推到设备/模拟器上看到效果。

**具体行动：**

- [ ] 安装 JDK 17 和必要的编译工具
- [ ] 配置 Android 模拟器（或准备一台可 adb 连接的设备）
- [ ] **核心实验**：在 `ActivityManagerService.java` 的 `startActivity()` 方法中加入一行 `android.util.Log.e("FENGXIANG", "startActivity called")`
- [ ] 运行 `mmm frameworks/base/services` 编译 services.jar
- [ ] `adb root && adb remount && adb push services.jar /system/framework/ && adb reboot`
- [ ] 重启后在 logcat 中搜索 `FENGXIANG`，确认日志出现

> **这个实验是你和 Framework 的第一次互动。编译→推送→看到效果，这个闭环打通了，后面学什么都快十倍。**

**验证标准**：自己加的 Log 在 logcat 中能搜到。

---

### Day 3：Binder 概览 —— 建立整体心智模型

**目标**：搞清楚 Binder 的整体架构——不是细节，是"谁和谁、怎么连"。

**你不应该做的事情**：一头扎进 Binder 驱动的 C 代码。你今天的目标是画图，不是读代码。

**具体行动：**

- [ ] 用纸笔画一张 Binder 通信的示意图，标注四个角色：
  ```
  App(Client) ──→ ServiceManager ──→ AMS(Server)
       │                                    │
       └──── Binder Driver (Kernel) ────────┘
  ```
- [ ] 用你自己的话回答以下三个问题（写下来，不要抄任何博客）：
  1. ServiceManager 是干什么的？为什么需要它？
  2. App 调用 `getService("activity")` 后，拿到的到底是什么东西？
  3. 数据从 App 进程传到 AMS 进程，中间它到底经过了哪里？

**验证标准**：能用笔在纸上画出完整的调用链，并口头解释每一步。

---

### Day 4：手写 AIDL Demo（一）

**目标**：从零写一个跨进程通信的 demo，理解 AIDL 做了什么。

**具体行动：**

- [ ] 创建两个 App：`BinderClient` 和 `BinderServer`
  - Server 提供一个 `IHelloService`，方法 `String sayHello(String name)`
  - Client 调用 Server 的 `sayHello()`
- [ ] 定义 AIDL 文件：`IHelloService.aidl`
- [ ] 编译项目，查看 `build/generated/` 下 AIDL 自动生成的 Java 文件
- [ ] 找到生成的 `Stub` 类和 `Proxy` 类
- [ ] 在 Server 的 Service 中实现 `onBind()`，返回 `Stub` 实例

**关键观察**：AIDL 帮我们生成了什么？
- `IHelloService.Stub` — 这个是 Bn 侧（Binder Native），Server 用它
- `IHelloService.Stub.Proxy` — 这个是 Bp 侧（Binder Proxy），Client 用它

---

### Day 5：手写 AIDL Demo（二）—— 拆开黑盒

**目标**：手工拆解 AIDL 生成的代码，理解每一行在做什么。

**具体行动：**

- [ ] 逐行阅读生成的 Java 代码，重点关注：
  - `Stub.onTransact(int code, Parcel data, Parcel reply, int flags)` — **Server 侧收到数据**
  - `Proxy.sayHello(String name)` — **Client 侧发送数据**
  - 看清楚 `Parcel.writeString()` → `mRemote.transact()` → `onTransact()` → `Parcel.readString()` 这条链
- [ ] 回答以下问题：
  - `transact()` 的第一个参数 `code` 是干什么的？
  - `Parcel` 是什么？它为什么叫"包裹"？
  - 如果 Server 端在处理请求时崩溃了，Client 端会发生什么？

**验证标准**：能脱离 AIDL，手写一个简单的 Java Binder 通信（不依赖 .aidl 文件）。

---

### Day 6-7：周末巩固 & 回顾

- [ ] 重新画出 Binder 通信的完整数据流图，这次加上 Parcel 的处理细节
- [ ] 回答"能不能用日常语言给非技术人员解释 Binder？"——录一段 3 分钟语音，自己听一遍
- [ ] 整理第一周的笔记，标注没搞懂的地方，带着问题进入第二周

---

## 4. 第二周：Binder 深入 & 链接 Framework

### Day 8：Binder 驱动核心 —— `binder_transaction`

**目标**：读懂 Binder 驱动最关键的一个函数，理解"只拷一次"是什么意思。

**文件**：`kernel/common/drivers/android/binder.c` → 函数 `binder_transaction`

**具体行动：**

- [ ] 先不用逐行读，先搞清楚函数签名和主要的步骤（读关键注释）
- [ ] 画出 `binder_transaction` 的数据流：
  ```
  发送进程用户空间 buffer
    → copy_from_user() → 内核空间 buffer
      → binder_transaction() 内核处理
        → copy_to_user() → 接收进程用户空间 buffer
  ```
- [ ] 回答：为什么说 Binder **只拷贝一次**？和传统的 IPC（如管道、socket）相比少拷贝了几次？
- [ ] 了解 `binder_node`、`binder_ref`、`binder_proc` 这三个核心数据结构分别是干什么的

> **不要试图读懂驱动里每一行代码。** 你的目标是搞清楚"数据怎么过去的"，不是"内核内存怎么管理的"。

---

### Day 9：跟踪真实调用链 —— ServiceManager.getService

**目标**：从 Java API 一路追到底层，看到一个真实 Framework Binder 调用的完整路径。

**调用链跟踪：**

```
App 侧：
  ActivityManager.getService()
    → IActivityManager.Stub.asInterface(binder)
      → ServiceManager.getService("activity")
        → BinderProxy.transact()          ← Java 层

Native 侧：
  BpBinder::transact()                    ← C++ Native 层
    → IPCThreadState::transact()
      → ioctl(BINDER_WRITE_READ)          ← 进入驱动！
        → binder_ioctl()
          → binder_thread_write()
          → binder_transaction()          ← 就是 Day 8 读的那个函数
```

**具体行动：**

- [ ] 在源码中找到并阅读以下文件的关键段落：
  - `frameworks/base/core/java/android/os/ServiceManager.java` （重点关注 `getService` 方法）
  - `frameworks/base/core/jni/android_os_Binder.cpp` （Java → Native 的 JNI 桥）
  - `frameworks/native/libs/binder/IServiceManager.cpp` （Native 层的 SM 交互）
- [ ] 画出 Java → Native → Kernel 三层的调用栈图

---

### Day 10：BBinder vs BPBinder —— 拆开 Native 层的"双向"

**目标**：理解 BnBinder 和 BpBinder 在 C++ 层面是怎么对应的。

**文件**：`frameworks/native/libs/binder/Binder.cpp`、`BpBinder.cpp`

**具体行动：**

- [ ] 读懂 `BBinder::onTransact()` 和 `BpBinder::transact()` 的关系
- [ ] 理解 `IPCThreadState` 的角色——它是怎么管理 Binder 线程的？
- [ ] 回答：当 App 调用 `startActivity()` 时，经过了多少次 `ioctl` 系统调用？

---

### Day 11：Parcel 深入 —— 数据的"打包"和"拆包"

**目标**：理解 Binder 的数据序列化是怎么做的。

**文件**：`frameworks/native/libs/binder/Parcel.cpp`（只读关键方法）、`frameworks/base/core/java/android/os/Parcel.java`

**具体行动：**

- [ ] 追踪 `Parcel.writeString()` → `Parcel.readString()` 的完整路径
- [ ] 理解 `flatten_binder()` / `unflatten_binder()` —— Binder 引用是怎么通过 Parcel 传的
- [ ] 回答：为什么 Binder 对象能直接通过 Parcel 传递（不是序列化成 byte[]），而普通对象不行？

---

### Day 12：Binder 线程池 —— 系统服务怎么"等人来调"

**目标**：理解 Binder 线程模型。

**具体行动：**

- [ ] 阅读 `IPCThreadState::joinThreadPool()` — 系统服务进程的主循环
- [ ] 理解：一个系统服务（如 AMS）启动后，它在干什么？它为什么能"等"到 App 的调用？
- [ ] 回答：如果有 10 个 App 同时调 AMS，会创建几条线程处理？分别是谁发的？

---

### Day 13：综合实战 —— 用 Binder 通信改造第一个 Demo

**目标**：把第一周写的 AIDL demo 升级，不使用 AIDL 文件，纯手写 Java Binder 类。

**具体行动：**

- [ ] 手写 `IHelloService.java`（不依赖 .aidl 生成）
  - 必须包含 `Stub`（extends Binder implements IHelloService）
  - 必须包含 `Proxy`（implements IHelloService）
  - 必须手动实现 `onTransact()` 的逻辑
- [ ] 在 Server 和 Client 项目中分别引入手写的接口文件
- [ ] 验证跨进程通信正常工作

---

### Day 14：阶段总结 & 自检

- [ ] 重画 Binder 完整数据流图（Java → Native → Kernel → Native → Java），不看任何资料
- [ ] 用 3 分钟口头解释 Binder 是什么——录音，回听，看能不能让外行听懂
- [ ] 列出还不理解的点，作为下一阶段的"待追查清单"

---

## 5. 阶段自检题

完成本阶段后，你应该能自信回答以下问题：

| # | 问题 | 如果答不上来... |
|---|------|----------------|
| 1 | ServiceManager 是什么？它是怎么被找到的（它的 handle 固定是多少）？ | 回看 Day 3、9 |
| 2 | `Stub` 和 `Proxy` 分别用在什么时候？为什么需要这两个类？ | 回看 Day 4、5 |
| 3 | `transact()` 和 `onTransact()` 的关系是什么？谁调谁？ | 回看 Day 5、10 |
| 4 | 数据从 App 到 AMS 经过了哪几层？哪一次是"只拷一次"的那个关键步骤？ | 回看 Day 8、9 |
| 5 | Parcel 里的 `writeBinder()` 是干什么的？为什么 Binder 对象可以直接跨进程传？ | 回看 Day 11 |
| 6 | 系统服务进程启动后为什么不退出——它在等什么？ | 回看 Day 12 |

---

## 6. 每日学习节奏建议

每天 2-3 小时：

| 时间段 | 内容 | 时长 |
|--------|------|------|
| 前 30 分钟 | 复习前一天内容，画出昨天的核心图 | 30 min |
| 中间 60-90 分钟 | 当日核心任务（读代码/写 Demo） | 60-90 min |
| 最后 30 分钟 | 整理笔记，写下来"今天我搞懂了什么"和"今天哪里还迷糊" | 30 min |

> **关键纪律**：每天写下"今天我搞懂了什么"和"今天哪里还迷糊"。前者建立信心，后者给明天锚定方向。

---

> *"The first principle is that you must not fool yourself — and you are the easiest person to fool."*
> — Richard Feynman, 1974 Caltech Commencement
