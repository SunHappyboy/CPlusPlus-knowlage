# Windows 消息与 Qt 事件循环完整机制详解

> 本文档详细说明 Qt 在 Windows 平台上如何将 Windows 消息 (MSG) 转换为 Qt 事件 (QEvent) 的完整流程

---

## 目录

1. [核心概念概述](#1-核心概念概述)
2. [架构组件](#2-架构组件)
3. [窗口类注册详解](#3-窗口类注册详解)
4. [processEvents 完整解析](#4-processevents-完整解析)
5. [nativeEvent 过滤器机制](#5-nativeevent-过滤器机制)
6. [事件分发详解 - handleMouseEvent & sendEvent](#6-事件分发详解---handlemouseevent--sendevent)
7. [窗口过程与消息转换](#7-窗口过程与消息转换)
8. [控件与 HWND 的关系](#8-控件与-hwnd-的关系)
9. [事件循环启动流程](#9-事件循环启动流程)
10. [完整流程图](#10-完整流程图)
11. [关键代码位置索引](#11-关键代码位置索引)

---

## 1. 核心概念概述

### 1.1 Windows 消息系统

Windows 使用**消息驱动**模型：

```
用户操作/系统事件 → Windows 消息队列 → 应用程序消息循环 → WndProc → 窗口行为
```

### 1.2 Qt 事件系统

Qt 使用**事件驱动**模型：

```
QEvent → QEventDispatcher → QEventLoop → QObject::event() → 具体处理
```

### 1.3 桥接机制

Qt 在 Windows 上通过以下方式桥接两个系统：

| Windows 组件 | Qt 对应组件 | 桥接方式 |
|-------------|------------|---------|
| MSG 结构体 | QEvent | 在 WndProc 中转换 |
| WndProc | QEventDispatcher | 拦截并转换消息 |
| HWND | QWidget* | 通过平台插件映射 |
| 消息队列 | 事件队列 | 同步处理 |

---

## 2. 架构组件

### 2.1 核心类结构

```
QAbstractEventDispatcher
    │
    └── QEventDispatcherWin32          [QtCore]
            │
            ├── QWindowsMessageWindowClassContext  (窗口类注册)
            ├── qt_internal_proc                  (内部窗口过程)
            └── internalHwnd                      (隐藏消息窗口)

QWindowsContext                        [QtPlatformSupport]
    │
    ├── windowsProc()                   (主窗口过程)
    ├── QWindowsKeyMapper               (键盘消息转换)
    └── QWindowsPointerHandler          (鼠标/触摸消息转换)
```

### 2.2 隐藏消息窗口

每个使用 Qt 的线程都有一个**隐藏的消息窗口**：

```cpp
// qeventdispatcher_win.cpp:273
Q_GLOBAL_STATIC(QWindowsMessageWindowClassContext, qWindowsMessageWindowClassContext)
```

**特点：**
- 类型：`HWND_MESSAGE`（消息窗口，不可见）
- 用途：接收 socket 通知、定时器事件、唤醒信号
- 不参与 UI 显示

---

## 3. 窗口类注册详解

### 3.1 注册时机 - Q_GLOBAL_STATIC 机制

窗口类注册发生在**首次访问** `Q_GLOBAL_STATIC` 变量时：

```cpp
// qeventdispatcher_win.cpp:273
Q_GLOBAL_STATIC(QWindowsMessageWindowClassContext, qWindowsMessageWindowClassContext)
```

**Q_GLOBAL_STATIC 工作原理：**

1. **延迟初始化**：首次访问时才创建对象
2. **线程安全**：使用原子操作保证线程安全
3. **自动清理**：程序退出时自动析构

### 3.2 触发时机序列

```
┌─────────────────────────────────────────────────────────────────┐
│  程序启动                                                        │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  QApplication a(argc, argv)                                      │
│  └── QCoreApplication::QCoreApplication()                       │
│      └── QGuiApplicationPrivate::platform_integration_init()    │
│          └── 创建 QWindowsIntegration                           │
│              └── QWindowsContext::QWindowsContext()             │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  首次需要事件调度器时                                             │
│  └── QEventDispatcherWin32 构造                                  │
│      └── qt_create_internal_window(this)                         │
│          └── qWindowsMessageWindowClassContext()  ← 首次访问！  │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  QWindowsMessageWindowClassContext 构造函数                       │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  1. 生成唯一类名：                                          │  │
│  │     "QEventDispatcherWin32_Internal_Widget" + 地址          │  │
│  │                                                            │  │
│  │  2. 填充 WNDCLASS：                                         │  │
│  │     wc.lpfnWndProc = qt_internal_proc                      │  │
│  │     wc.lpszClassName = className                           │  │
│  │                                                            │  │
│  │  3. RegisterClass(&wc)  ← 向 Windows 注册窗口类！          │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  创建隐藏消息窗口                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  CreateWindow(                                             │  │
│  │      ctx->className,           // 使用注册的类名            │  │
│  │      ...,                                                     │  │
│  │      HWND_MESSAGE,              // 消息窗口类型             │  │
│  │      ...                                                      │  │
│  │  )                                                          │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### 3.3 窗口类注册完整代码

```cpp
// qeventdispatcher_win.cpp:236-263

QWindowsMessageWindowClassContext::QWindowsMessageWindowClassContext()
    : atom(0), className(0)
{
    // === 步骤1: 生成唯一类名 ===
    // 包含函数地址，确保多个 Qt DLL 可以共存于同一进程
    const QString qClassName = QStringLiteral("QEventDispatcherWin32_Internal_Widget")
        + QString::number(quintptr(qt_internal_proc));

    // === 步骤2: 转换为宽字符字符串 ===
    className = new wchar_t[qClassName.size() + 1];
    qClassName.toWCharArray(className);
    className[qClassName.size()] = 0;  // null 终止符

    // === 步骤3: 填充 WNDCLASS 结构 ===
    WNDCLASS wc;
    wc.style         = 0;                  // 不需要特殊样式
    wc.lpfnWndProc   = qt_internal_proc;   // 窗口过程函数（关键！）
    wc.cbClsExtra    = 0;
    wc.cbWndExtra    = 0;
    wc.hInstance     = GetModuleHandle(0); // 当前模块句柄
    wc.hIcon         = 0;                  // 不需要图标
    wc.hCursor       = 0;                  // 不需要光标
    wc.hbrBackground = 0;                  // 不需要背景
    wc.lpszMenuName  = NULL;
    wc.lpszClassName = className;          // 类名

    // === 步骤4: 向 Windows 注册窗口类 ===
    atom = RegisterClass(&wc);
    if (!atom) {
        qErrnoWarning("%ls RegisterClass() failed", qUtf16Printable(qClassName));
        delete [] className;
        className = 0;
    }
}
```

### 3.4 隐藏窗口创建

```cpp
// qeventdispatcher_win.cpp:275-297

static HWND qt_create_internal_window(const QEventDispatcherWin32 *eventDispatcher)
{
    // 获取窗口类上下文（触发首次访问时构造）
    QWindowsMessageWindowClassContext *ctx = qWindowsMessageWindowClassContext();
    if (!ctx->atom)
        return 0;

    // 创建消息窗口（HWND_MESSAGE 类型，不可见）
    HWND wnd = CreateWindow(
        ctx->className,           // 使用注册的类名
        ctx->className,           // 窗口名
        0,                        // 样式：0 表示不可见
        0, 0, 0, 0,               // 位置和大小：都为 0
        HWND_MESSAGE,             // 父窗口：HWND_MESSAGE 表示消息窗口
        0,                        // 菜单：无
        GetModuleHandle(0),       // 实例句柄
        0                         // 创建数据
    );

    if (!wnd) {
        qErrnoWarning("CreateWindow() for QEventDispatcherWin32 internal window failed");
        return 0;
    }

    // 将事件调度器指针存储在窗口的用户数据中
    // 这样在 WndProc 中可以获取到调度器实例
    SetWindowLongPtr(wnd, GWLP_USERDATA, reinterpret_cast<LONG_PTR>(eventDispatcher));

    return wnd;
}
```

---

## 4. processEvents 完整解析

### 4.1 processEvents 是什么？

`processEvents()` 是**事件循环的核心引擎**，负责：
1. 从 Windows 消息队列获取消息
2. 过滤和处理特殊消息
3. 将消息分发到窗口过程
4. 在没有消息时等待

### 4.2 调用链

```
QApplication::exec()
    ↓
QEventLoop::exec()
    ↓
while (!exit)
    ↓
QEventDispatcherWin32::processEvents()
```

### 4.3 processEvents 完整代码逐行解析

```cpp
// qeventdispatcher_win.cpp:476-555

bool QEventDispatcherWin32::processEvents(QEventLoop::ProcessEventsFlags flags)
{
    // === 准备工作 ===
    auto threadData = d->threadData.loadRelaxed();
    bool canWait;
    bool retVal = false;

    // ========== 外层循环：处理消息 + 等待 ==========
    do {
        QVarLengthArray<MSG> processedTimers;  // 记录已处理的定时器，防止活锁

        // ----------- 内层循环：获取并处理消息 -----------
        while (!d->interrupt.loadRelaxed()) {
            MSG msg;

            // 【步骤1】获取消息 - 优先级顺序

            // 1a. 优先处理排队的用户输入事件
            if (!(flags & QEventLoop::ExcludeUserInputEvents)
                && !d->queuedUserInputEvents.isEmpty()) {
                msg = d->queuedUserInputEvents.takeFirst();
            }
            // 1b. 优先处理排队的 socket 事件
            else if (!(flags & QEventLoop::ExcludeSocketNotifiers)
                     && !d->queuedSocketEvents.isEmpty()) {
                msg = d->queuedSocketEvents.takeFirst();
            }
            // 1c. 从系统消息队列获取消息（非阻塞）
            else if (PeekMessage(&msg, 0, 0, 0, PM_REMOVE)) {

                // 检查是否需要排除用户输入事件
                if (flags.testFlag(QEventLoop::ExcludeUserInputEvents)
                    && isUserInputMessage(msg.message)) {
                    // 排队延后处理
                    d->queuedUserInputEvents.append(msg);
                    continue;
                }

                // 检查是否需要排除 socket 事件
                if ((flags & QEventLoop::ExcludeSocketNotifiers)
                    && (msg.message == WM_QT_SOCKETNOTIFIER
                        && msg.hwnd == d->internalHwnd)) {
                    d->queuedSocketEvents.append(msg);
                    continue;
                }
            }
            // 1d. 检查是否有新消息到达（非阻塞）
            else if (MsgWaitForMultipleObjectsEx(0, NULL, 0, QS_ALLINPUT, MWMO_ALERTABLE)
                     == WAIT_OBJECT_0) {
                // 有新消息，继续循环获取
                continue;
            }
            // 1e. 没有消息了，退出内层循环
            else {
                break;
            }

            // 【步骤2】处理特殊 Qt 消息

            // 2a. Qt 唤醒消息
            if (d->internalHwnd == msg.hwnd
                && msg.message == WM_QT_SENDPOSTEDEVENTS) {
                d->startPostedEventsTimer();
                retVal = true;
                continue;  // 不继续分发此消息
            }

            // 2b. 定时器消息 - 排除 postEvent 用的定时器
            if (msg.message == WM_TIMER) {
                if (d->internalHwnd == msg.hwnd
                    && msg.wParam == d->sendPostedEventsTimerId) {
                    continue;  // 跳过内部定时器
                }

                // 防止活锁：检查是否已处理过
                bool found = false;
                for (int i = 0; !found && i < processedTimers.count(); ++i) {
                    const MSG processed = processedTimers.constData()[i];
                    found = (processed.wParam == msg.wParam
                            && processed.hwnd == msg.hwnd
                            && processed.lParam == msg.lParam);
                }
                if (found)
                    continue;
                processedTimers.append(msg);
            }

            // 2c. WM_QUIT - 退出消息
            else if (msg.message == WM_QUIT) {
                if (QCoreApplication::instance())
                    QCoreApplication::instance()->quit();
                return false;
            }

            // 【步骤3】nativeEvent 过滤器
            if (!filterNativeEvent(
                    QByteArrayLiteral("windows_generic_MSG"), &msg, 0)) {

                // 【步骤4】分发消息到窗口过程
                TranslateMessage(&msg);    // 虚拟键码 → 字符消息
                DispatchMessage(&msg);     // 调用目标窗口的 WndProc
            }

            retVal = true;
        }
        // ----------- 内层循环结束 -----------

        // 【步骤5】等待新消息（如果没有处理任何消息）
        canWait = (!retVal
                   && !d->interrupt.loadRelaxed()
                   && flags.testFlag(QEventLoop::WaitForMoreEvents)
                   && threadData->canWaitLocked());

        if (canWait) {
            emit aboutToBlock();  // 即将阻塞信号
            // 阻塞等待，直到有消息或事件到达
            MsgWaitForMultipleObjectsEx(0, NULL, INFINITE, QS_ALLINPUT,
                                        MWMO_ALERTABLE | MWMO_INPUTAVAILABLE);
            emit awake();  // 唤醒信号
        }

    } while (canWait);  // 如果等待了，再循环一次处理新消息

    return retVal;
}
```

### 4.4 processEvents 状态机

```
                      ┌─────────────────┐
                      │   processEvents │
                      │   开始执行      │
                      └────────┬────────┘
                               │
                               ▼
                      ┌─────────────────┐
                      │  内层 while 循环 │
                      │  (!interrupt)   │
                      └────────┬────────┘
                               │
                    ┌──────────▼──────────┐
                    │  获取 MSG            │
                    │  - queuedUserInput?  │
                    │  - queuedSocket?     │
                    │  - PeekMessage()     │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │  检查 flags          │
                    │  - ExcludeUserInput?│
                    │  - ExcludeSocket?   │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │  处理特殊消息        │
                    │  - WM_QT_*?         │
                    │  - WM_TIMER?        │
                    │  - WM_QUIT?         │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │  filterNativeEvent  │ ← 应用自定义过滤器
                    │  过滤器链           │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │  TranslateMessage   │
                    │  DispatchMessage    │ → WndProc
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │  有更多消息？        │
                    └──────┬────────┬─────┘
                           │ YES  │ NO
                           │      │
              ┌────────────┘      └─────────────┐
              │                                │
              ▼                                ▼
      继续内层循环                    ┌───────────────┐
                                      │  内层循环结束  │
                                      │  检查 canWait │
                                      └───────┬───────┘
                                              │
                                  ┌───────────▼───────────┐
                                  │  canWait == true?     │
                                  │  - emit aboutToBlock  │
                                  │  - MsgWaitFor...()    │
                                  │  - emit awake         │
                                  └───────────┬───────────┘
                                              │ YES
                                              ▼
                                      ┌───────────────┐
                                      │  继续 do 循环  │
                                      └───────────────┘
```

---

## 5. nativeEvent 过滤器机制

### 5.1 什么是 nativeEvent 过滤器？

nativeEvent 过滤器允许应用程序在**消息转换为 QEvent 之前**拦截原生系统消息。

### 5.2 过滤器链结构

```
QAbstractEventDispatcher
    │
    └── eventFilters: QList<QAbstractNativeEventFilter*>
            │
            ├── filter1
            ├── filter2
            └── filter3
```

### 5.3 filterNativeEvent 实现

```cpp
// qabstracteventdispatcher.cpp:554-570

bool QAbstractEventDispatcher::filterNativeEvent(
    const QByteArray &eventType,   // 事件类型标识
    void *message,                 // 原生消息（Windows 上是 MSG*）
    qintptr *result)               // [输出] 过滤器返回的结果
{
    Q_D(QAbstractEventDispatcher);

    // 检查是否有注册的过滤器
    if (!d->eventFilters.isEmpty()) {

        // 提高循环级别，使 deleteLater() 在主事件循环中处理
        QScopedScopeLevelCounter scopeLevelCounter(d->threadData.loadAcquire());

        // 遍历所有过滤器
        for (int i = 0; i < d->eventFilters.size(); ++i) {
            QAbstractNativeEventFilter *filter = d->eventFilters.at(i);
            if (!filter)
                continue;

            // 调用过滤器
            // 如果返回 true，表示消息被消费，停止后续处理
            if (filter->nativeEventFilter(eventType, message, result))
                return true;
        }
    }

    return false;  // 没有过滤器处理此消息
}
```

### 5.4 在 processEvents 中的调用

```cpp
// 在 processEvents 中，DispatchMessage 之前
if (!filterNativeEvent(
        QByteArrayLiteral("windows_generic_MSG"), &msg, 0)) {

    // 只有当所有过滤器都返回 false 时，才继续分发消息
    TranslateMessage(&msg);
    DispatchMessage(&msg);
}
```

### 5.5 在 qt_internal_proc 中的调用

```cpp
// qt_internal_proc:88-106
LRESULT QT_WIN_CALLBACK qt_internal_proc(HWND hwnd, UINT message, WPARAM wp, LPARAM lp)
{
    MSG msg;
    msg.hwnd = hwnd;
    msg.message = message;
    msg.wParam = wp;
    msg.lParam = lp;

    QAbstractEventDispatcher* dispatcher = QAbstractEventDispatcher::instance();
    qintptr result;

    // 首先通过 nativeEvent 过滤器
    if (dispatcher->filterNativeEvent(
            QByteArrayLiteral("windows_dispatcher_MSG"), &msg, &result)) {
        return result;  // 过滤器处理了消息，直接返回
    }

    // ... 继续处理消息
}
```

### 5.6 过滤器的两种调用点

| 调用位置 | eventType 参数 | 时机 |
|---------|---------------|------|
| `processEvents` | `windows_generic_MSG` | DispatchMessage 之前，**所有消息** |
| `qt_internal_proc` | `windows_dispatcher_MSG` | WndProc 内部，**internalHwnd 的消息** |

### 5.7 自定义过滤器示例

```cpp
// 定义自定义过滤器
class MyNativeEventFilter : public QAbstractNativeEventFilter
{
public:
    bool nativeEventFilter(const QByteArray &eventType, void *message, qintptr *result) override
    {
        if (eventType == "windows_generic_MSG") {
            MSG *msg = static_cast<MSG*>(message);

            // 拦截所有 WM_KEYDOWN 消息
            if (msg->message == WM_KEYDOWN) {
                *result = 0;
                return true;  // 消费消息，阻止默认处理
            }
        }
        return false;  // 不处理，继续传递
    }
};

// 安装过滤器
int main(int argc, char *argv[])
{
    QApplication app(argc, argv);

    MyNativeEventFilter filter;
    app.installNativeEventFilter(&filter);

    return app.exec();
}
```

### 5.3 QWindowSystemInterface::handleMouseEvent() 详解

#### 5.3.1 是什么？

`QWindowSystemInterface::handleMouseEvent()` 是**平台插件层到 Qt 事件系统的桥梁函数**。

```cpp
// qwindowsysteminterface.h:43-64
template<typename Delivery = QWindowSystemInterface::DefaultDelivery>
static bool handleMouseEvent(QWindow *window, ulong timestamp, const QPointF &local,
                             const QPointF &global, Qt::MouseButtons state,
                             Qt::MouseButton button, QEvent::Type type,
                             Qt::KeyboardModifiers mods = Qt::NoModifier,
                             Qt::MouseEventSource source = Qt::MouseEventNotSynthesized);
```

#### 5.3.2 工作流程

```
┌─────────────────────────────────────────────────────────────┐
│  QWindowsPointerHandler::translateMouseEvent()              │
│  // 平台插件层：Windows MSG → Qt 参数                        │
│  • 提取 MSG 信息：localPos, globalPos, button, etc.        │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│  QWindowSystemInterface::handleMouseEvent<Delivery>()      │
│  // 模板函数，根据 Delivery 类型决定同步/异步分发           │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  if (Delivery == SynchronousDelivery) {                │  │
│  │      // 同步：立即处理事件                               │  │
│  │  } else {                                             │  │
│  │      // 异步：放入事件队列                               │  │
│  │  }                                                    │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│  handleWindowSystemEvent<MouseEvent, Delivery>()          │
│  // 调用 QWindowSystemHelper::handleEvent                  │
└─────────────────────────────────────────────────────────────┘
                              ↓
         ┌──────────────────────┴──────────────────────┐
         │                                       │
         ▼                                       ▼
┌───────────────────────┐             ┌───────────────────────────┐
│  SynchronousDelivery  │             │  AsynchronousDelivery     │
│  (立即处理)             │             │  (放入队列)                │
└───────────────────────┘             └───────────────────────────┘
         │                                       │
         ▼                                       ▼
┌───────────────────────┐             ┌───────────────────────────┐
│  QGuiApplicationPrivate::          │  windowSystemEventQueue   │
│  processMouseEvent()  │             │  .append(new MouseEvent)  │
│      ↓                 │             │      ↓                    │
│  立即处理事件           │             │  dispatcher->wakeUp()    │
└───────────────────────┘             │      ↓                    │
                                       │  事件循环稍后处理          │
                                       └───────────────────────────┘
```

#### 5.3.3 两种分发模式

| 模式 | Delivery 参数 | 行为 | 使用场景 |
|------|---------------|------|----------|
| **同步分发** | `SynchronousDelivery` | 立即处理事件，不放入队列 | 测试框架、需要立即响应的事件 |
| **异步分发** | `DefaultDelivery` | 放入 `windowSystemEventQueue` | 正常的用户输入事件 |

#### 5.3.4 异步分发代码

```cpp
// qwindowsysteminterface.cpp:125-133
template<>
template<typename EventType, typename ...Args>
bool QWindowSystemHelper<QWindowSystemInterface::AsynchronousDelivery>::handleEvent(Args ...args)
{
    // 1. 创建事件对象并放入队列
    QWindowSystemInterfacePrivate::windowSystemEventQueue.append(new EventType(args...));

    // 2. 唤醒事件调度器，让它知道有新事件
    if (QAbstractEventDispatcher *dispatcher = QGuiApplicationPrivate::qt_qpa_core_dispatcher())
        dispatcher->wakeUp();  // ← 发送 WM_QT_SENDPOSTEDEVENTS 消息

    return true;
}
```

#### 5.3.5 同步分发代码

```cpp
// qwindowsysteminterface.cpp:100-108
template<typename EventType, typename ...Args>
bool QWindowSystemHelper<QWindowSystemInterface::SynchronousDelivery>::handleEvent(Args ...args)
{
    EventType event(args...);

    // 直接处理，不经过队列
    if (QWindowSystemInterfacePrivate::eventHandler) {
        return QWindowSystemInterfacePrivate::eventHandler->sendEvent(&event);
    } else {
        return QGuiApplicationPrivate::processWindowSystemEvent(&event);
    }
}
```

---

### 5.4 QCoreApplication::sendEvent() 详解

#### 5.4.1 是什么？

`QCoreApplication::sendEvent()` 是**Qt 事件分发的核心函数**，直接发送事件到接收者。

```cpp
// qcoreapplication.cpp:1538-1547
bool QCoreApplication::sendEvent(QObject *receiver, QEvent *event)
{
    Q_ASSERT_X(receiver, "QCoreApplication::sendEvent", "Unexpected null receiver");
    Q_ASSERT_X(event, "QCoreApplication::sendEvent", "Unexpected null event");

    // 设置 spontaneous 标志为 false（表示程序主动发送，非系统自发）
    event->m_spont = false;
    return notifyInternal2(receiver, event);
}
```

**对比：** `QGuiApplication::sendSpontaneousEvent()` 设置 `m_spont = true`（表示系统自发事件）。

#### 5.4.2 事件分发链

```
┌─────────────────────────────────────────────────────────────┐
│  QGuiApplicationPrivate::processMouseEvent()                │
│  // 创建 QMouseEvent 对象                                   │
│  QMouseEvent ev(type, localPoint, localPoint, globalPoint, │
│                 button, e->buttons, e->modifiers, ...);    │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│  QGuiApplication::sendSpontaneousEvent(window, &ev)        │
│  // event->m_spont = true                                  │
│  // return notifyInternal2(receiver, event);              │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│  QCoreApplication::notifyInternal2(receiver, event)        │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  1. 检查线程一致性                                      │  │
│  │  2. 调用内部回调 (QInternal::activateCallbacks)      │  │
│  │  3. 调用 qApp->notify(receiver, event)                │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│  QCoreApplication::notify(receiver, event)                │
│  // 检查应用是否正在关闭                                   │
│  // return doNotify(receiver, event);                     │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│  QCoreApplicationPrivate::notify_helper(receiver, event)   │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  1. sendThroughApplicationEventFilters()              │  │
│  │     → 遍历所有应用程序级事件过滤器                      │  │
│  │     → obj->eventFilter(receiver, event)               │  │
│  │                                                        │  │
│  │  2. sendThroughObjectEventFilters()                   │  │
│  │     → 遍历接收者的事件过滤器                            │  │
│  │     → receiver->eventFilter(receiver, event)          │  │
│  │                                                        │  │
│  │  3. receiver->event(event)  ← 核心！                  │  │
│  │     → 调用接收者的 event() 方法                        │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│  QObject::event(QEvent *event)                             │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  switch (event->type()) {                              │  │
│  │      case QEvent::MouseButtonPress:                    │  │
│  │          mousePressEvent(static_cast<QMouseEvent*>)   │  │
│  │          break;                                       │  │
│  │      case QEvent::MouseMove:                          │  │
│  │          mouseMoveEvent(...)                          │  │
│  │          break;                                       │  │
│  │      ...                                               │  │
│  │  }                                                    │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

#### 5.4.3 notify_helper 完整代码

```cpp
// qcoreapplication.cpp:1255-1280
bool QCoreApplicationPrivate::notify_helper(QObject *receiver, QEvent * event)
{
    // === 步骤1: 应用程序级事件过滤器 ===
    if (QThread::isMainThread()
            && QCoreApplication::self
            && QCoreApplication::self->d_func()->sendThroughApplicationEventFilters(receiver, event)) {
        return true;  // 过滤器处理了事件，停止传递
    }

    // === 步骤2: 对象级事件过滤器 ===
    if (sendThroughObjectEventFilters(receiver, event)) {
        return true;  // 过滤器处理了事件，停止传递
    }

    // === 步骤3: 传递到接收者 ===
    return receiver->event(event);  // ← 核心！调用对象的 event() 方法
}
```

#### 5.4.4 事件过滤器的优先级

```
┌─────────────────────────────────────────────────────────────┐
│  1. 应用程序级事件过滤器                                     │
│     QCoreApplication::instance()->installEventFilter()     │
│     └── 可以拦截任何对象的任何事件                           │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│  2. 对象级事件过滤器                                         │
│     receiver->installEventFilter()                          │
│     └── 只能拦截该对象的事件                                 │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│  3. 接收者的 event() 方法                                    │
│     receiver->event(event)                                  │
│     └── 最终处理                                            │
└─────────────────────────────────────────────────────────────┘
```

#### 5.4.5 spontaneous 标志的意义

| 标志 | 值 | 含义 |
|------|-----|------|
| `m_spont = false` | `sendEvent()` | 程序主动发送的事件 |
| `m_spont = true` | `sendSpontaneousEvent()` | 系统自发事件（用户输入） |

这个标志影响一些特殊行为，例如：
- `QDialog` 默认只对 spontaneous 鼠标事件关闭
- 某些事件过滤器根据此标志决定是否处理

---

### 5.5 完整的 MSG → QEvent 转换链总结

```
Windows MSG (WM_LBUTTONDOWN)
       ↓
QWindowsContext::windowsProc(hwnd, message, wParam, lParam)
       ↓
QWindowsPointerHandler::translateMouseEvent(window, hwnd, et, msg, result)
       ↓
QWindowSystemInterface::handleMouseEvent<DefaultDelivery>(window, ...)
       ↓
┌─────────────────────────────────────────────────────────────┐
│  QWindowSystemHelper<AsynchronousDelivery>::handleEvent()    │
│  • windowSystemEventQueue.append(new MouseEvent(...))      │
│  • dispatcher->wakeUp()                                     │
└─────────────────────────────────────────────────────────────┘
       ↓
┌─────────────────────────────────────────────────────────────┐
│  事件循环稍后处理：                                          │
│  sendWindowSystemEvents()                                   │
│  • getWindowSystemEvent()                                   │
│  • QGuiApplicationPrivate::processWindowSystemEvent(e)      │
│  • processMouseEvent(static_cast<MouseEvent*>(e))          │
└─────────────────────────────────────────────────────────────┘
       ↓
QGuiApplication::sendSpontaneousEvent(window, &ev)
       ↓
QCoreApplication::notifyInternal2(receiver, event)
       ↓
QCoreApplication::notify(receiver, event)
       ↓
QCoreApplicationPrivate::notify_helper(receiver, event)
       ↓
receiver->event(event)  → QWidget::event(QEvent*)
       ↓
QMouseEvent → mousePressEvent(QMouseEvent*) → QPushButton::mousePressEvent
       ↓
emit clicked() → 槽函数执行
```

---

## 7. 窗口过程与消息转换

### 7.1 窗口过程函数

Qt 注册了两个主要的窗口过程：

#### 6.1.1 隐藏窗口的窗口过程

```cpp
// qeventdispatcher_win.cpp:88-212
LRESULT QT_WIN_CALLBACK qt_internal_proc(HWND hwnd, UINT message, WPARAM wp, LPARAM lp)
{
    // === 步骤1: 构建 MSG 结构 ===
    MSG msg;
    msg.hwnd = hwnd;
    msg.message = message;
    msg.wParam = wp;
    msg.lParam = lp;

    // === 步骤2: 获取事件调度器 ===
    QAbstractEventDispatcher* dispatcher = QAbstractEventDispatcher::instance();
    qintptr result;
    if (!dispatcher) {
        if (message == WM_TIMER)
            KillTimer(hwnd, wp);
        return 0;
    }

    // === 步骤3: nativeEvent 过滤 ===
    if (dispatcher->filterNativeEvent(
            QByteArrayLiteral("windows_dispatcher_MSG"), &msg, &result))
        return result;

    // === 步骤4: 获取 QEventDispatcherWin32 实例 ===
    // 从窗口的用户数据中获取（在 CreateWindow 时设置）
    auto q = reinterpret_cast<QEventDispatcherWin32 *>(
        GetWindowLongPtr(hwnd, GWLP_USERDATA));
    QEventDispatcherWin32Private *d = nullptr;
    if (q != nullptr)
        d = q->d_func();

    // === 步骤5: 根据消息类型处理 ===
    switch (message) {

    // --- Socket 通知 ---
    case WM_QT_SOCKETNOTIFIER: {
        int type = -1;
        switch (WSAGETSELECTEVENT(lp)) {
        case FD_READ:
        case FD_ACCEPT:
            type = 0; break;
        case FD_WRITE:
        case FD_CONNECT:
            type = 1; break;
        case FD_OOB:
            type = 2; break;
        case FD_CLOSE:
            type = 3; break;
        }
        if (type >= 0) {
            QSNDict *sn_vec[4] = { &d->sn_read, &d->sn_write, &d->sn_except, &d->sn_read };
            QSNDict *dict = sn_vec[type];
            QSockNot *sn = dict ? dict->value(qintptr(wp)) : 0;

            if (sn != nullptr) {
                QSockFd &sd = d->active_fd[sn->fd];
                if (sd.selected) {
                    d->doWsaAsyncSelect(sn->fd, 0);
                    sd.selected = false;
                }
                d->postActivateSocketNotifiers();

                const long eventCode = WSAGETSELECTEVENT(lp);
                if ((sd.mask & eventCode) != eventCode) {
                    sd.mask |= eventCode;
                    // === 转换：Socket 消息 → QEvent ===
                    QEvent event(type < 3 ? QEvent::SockAct : QEvent::SockClose);
                    QCoreApplication::sendEvent(sn->obj, &event);
                }
            }
        }
        return 0;
    }

    // --- 激活 Socket 通知器 ---
    case WM_QT_ACTIVATENOTIFIERS: {
        MSG msg;
        if (!PeekMessage(&msg, d->internalHwnd,
                         WM_QT_SOCKETNOTIFIER, WM_QT_SOCKETNOTIFIER, PM_NOREMOVE)
            && d->queuedSocketEvents.isEmpty()) {
            // 重新注册所有 socket 通知器
            for (QSFDict::iterator it = d->active_fd.begin(), end = d->active_fd.end();
                 it != end; ++it) {
                QSockFd &sd = it.value();
                if (!sd.selected) {
                    d->doWsaAsyncSelect(it.key(), sd.event);
                    sd.mask = 0;
                    sd.selected = true;
                }
            }
        }
        d->activateNotifiersPosted = false;
        return 0;
    }

    // --- 定时器 ---
    case WM_TIMER: {
        if (wp == d->sendPostedEventsTimerId)
            q->sendPostedEvents();
        else
            d->sendTimerEvent(wp);  // === 发送 QTimerEvent ===
        return 0;
    }

    // --- 发送已投递事件 ---
    case WM_QT_SENDPOSTEDEVENTS: {
        static const UINT mask = QS_ALLEVENTS;
        if (HIWORD(GetQueueStatus(mask)) == 0)
            q->sendPostedEvents();
        else
            d->startPostedEventsTimer();
        return 0;
    }
    }

    // === 默认处理 ===
    return DefWindowProc(hwnd, message, wp, lp);
}
```

#### 6.1.2 普通窗口的窗口过程

```cpp
// qwindowscontext.cpp:976-1250
bool QWindowsContext::windowsProc(HWND hwnd, UINT message,
                                  QtWindows::WindowsEventType et,
                                  WPARAM wParam, LPARAM lParam,
                                  LRESULT *result,
                                  QWindowsWindow **platformWindowPtr)
{
    // === 步骤1: 重建 MSG 结构 ===
    MSG msg;
    msg.hwnd = hwnd;
    msg.message = message;
    msg.wParam = wParam;
    msg.lParam = lParam;

    // === 步骤2: 查找对应的 QWindow ===
    QWindowsWindow *platformWindow = findPlatformWindow(hwnd);

    // === 步骤3: 原生事件过滤 ===
    if (!isInputMessage(msg.message) && filterNativeEvent(&msg, result))
        return true;
    if (platformWindow && filterNativeEvent(platformWindow->window(), &msg, result))
        return true;

    // === 步骤4: 根据事件类型分发 ===
    switch (et) {

    // --- 键盘事件 ---
    case QtWindows::KeyDownEvent:
    case QtWindows::KeyEvent:
        // WM_KEYDOWN, WM_KEYUP, WM_CHAR → QKeyEvent
        return d->m_keyMapper.translateKeyEvent(platformWindow->window(),
                                                 hwnd, msg, result);

    // --- 鼠标事件 ---
    case QtWindows::MouseEvent:
    case QtWindows::MouseWheelEvent:
    case QtWindows::LeaveEvent:
        // WM_*MOUSE* → QMouseEvent / QWheelEvent
        return d->m_pointerHandler.translateMouseEvent(platformWindow->window(),
                                                        hwnd, et, msg, result);

    // --- 窗口大小改变 ---
    case QtWindows::ResizeEvent:
        // WM_SIZE → QResizeEvent
        platformWindow->handleResized(static_cast<int>(wParam), lParam);
        return true;

    // --- 焦点事件 ---
    case QtWindows::FocusInEvent:
    case QtWindows::FocusOutEvent:
        // WM_SETFOCUS / WM_KILLFOCUS → QFocusEvent
        handleFocusEvent(et, platformWindow);
        return true;

    // --- 绘制事件 ---
    case QtWindows::ExposeEvent:
        // WM_PAINT → QPaintEvent
        return platformWindow->handleWmPaint(hwnd, message, wParam, lParam, result);

    // --- 关闭事件 ---
    case QtWindows::CloseEvent:
        // WM_CLOSE → QCloseEvent
        QWindowSystemInterface::handleCloseEvent(platformWindow->window());
        return true;
    }

    return false;
}
```

### 7.2 键盘事件转换

```cpp
// qwindowskeymapper.cpp:821-900
bool QWindowsKeyMapper::translateKeyEvent(QWindow *widget, HWND hwnd,
                                          const MSG &msg, LRESULT *result)
{
    *result = 0;

    // 1. 处理键盘布局改变
    if (msg.message == WM_INPUTLANGCHANGE) {
        changeKeyboard();
        return true;
    }

    // 2. 更新键盘状态映射
    if (msg.message != WM_CHAR && msg.message != WM_IME_CHAR)
        updateKeyMap(msg);

    // 3. 处理死键
    MSG peekedMsg;
    if (PeekMessage(&peekedMsg, hwnd, 0, 0, PM_NOREMOVE)
        && peekedMsg.message == WM_DEADCHAR) {
        return true;
    }

    // 4. 根据消息类型创建 Qt 事件
    QKeyEvent::Type keyType;
    Qt::Key qtKey;
    QString text;

    if (msg.message == WM_KEYDOWN || msg.message == WM_SYSKEYDOWN) {
        keyType = QEvent::KeyPress;
        qtKey = Qt::Key(/* 虚拟键码转换 */);
        text = /* 转换字符 */;
    } else {
        keyType = QEvent::KeyRelease;
    }

    // 5. 发送事件到 Qt
    QKeyEvent event(keyType, qtKey, modifiers, text, autoRepeat, count);
    QWindowSystemInterface::handleKeyEvent(widget, event);

    return true;
}
```

### 7.3 鼠标事件转换

```cpp
// qwindowspointerhandler.cpp:350-740
bool QWindowsPointerHandler::translateMouseEvent(QWindow *window,
                                                 HWND hwnd,
                                                 QtWindows::WindowsEventType et,
                                                 const MSG &msg,
                                                 LRESULT *result)
{
    // 1. 获取鼠标位置
    const QPoint globalPos = QPoint(msg.pt.x, msg.pt);
    QPoint localPos = QWindowsGeometryHint::mapFromGlobal(hwnd, globalPos);

    // 2. 确定事件类型和按钮
    QEvent::Type type;
    Qt::MouseButton button;
    Qt::MouseButtons buttons;

    switch (et) {
    case QtWindows::MousePressedEvent:
        type = QEvent::MouseButtonPress;
        if (msg.message == WM_LBUTTONDOWN) button = Qt::LeftButton;
        else if (msg.message == WM_RBUTTONDOWN) button = Qt::RightButton;
        else if (msg.message == WM_MBUTTONDOWN) button = Qt::MiddleButton;
        break;

    case QtWindows::MouseReleasedEvent:
        type = QEvent::MouseButtonRelease;
        // ...
        break;

    case QtWindows::MouseMoveEvent:
        type = QEvent::MouseMove;
        break;

    case QtWindows::MouseWheelEvent:
        type = QEvent::Wheel;
        break;
    }

    // 3. 发送事件到 Qt
    QMouseEvent event(type, localPos, globalPos, button, buttons, modifiers);
    QWindowSystemInterface::handleMouseEvent(window, event);

    return true;
}
```

---

## 8. 控件与 HWND 的关系

### 8.1 不是每个控件都有 HWND

**重要概念：** Qt 默认只为顶级窗口创建 HWND，子控件共享父窗口。

### 8.2 创建 HWND 的条件

| 条件 | 说明 |
|------|------|
| `widget->isWindow()` | 顶级窗口（QMainWindow, QDialog） |
| `Qt::WA_NativeWindow` | 强制原生窗口属性 |
| `Qt::WA_PaintOnScreen` | 需要直接绘制 |
| `Qt::AA_NativeWindows` | 全局原生窗口模式 |

### 8.3 典型窗口结构

```
┌─────────────────────────────────────────────────────────────┐
│  QMainWindow (HWND: 0x1234)                                 │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  WndProc 接收所有消息                                 │   │
│  │                                                     │   │
│  │  WM_LBUTTONDOWN (在 x=100, y=50)                     │   │
│  │       ↓                                              │   │
│  │  Qt 判断坐标 → 属于 QPushButton                       │   │
│  │       ↓                                              │   │
│  │  发送 QMouseEvent 到 QPushButton                      │   │
│  └─────────────────────────────────────────────────────┘   │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                │
│  │  Button  │  │  Label   │  │  Edit    │  (无 HWND)     │
│  └──────────┘  └──────────┘  └──────────┘                │
└─────────────────────────────────────────────────────────────┘
```

### 8.4 消息在子控件间的分发

```cpp
// 1. Windows 消息到达父窗口 WndProc
WM_LBUTTONDOWN → QMainWindow.WndProc

// 2. Qt 转换为 QMouseEvent
QMouseEvent(QEvent::MouseButtonPress, localPos, ...)

// 3. Qt 根据坐标查找目标控件
QWidget *child = parent->childAt(pos);

// 4. 发送事件到子控件
QCoreApplication::sendEvent(child, &event);

// 5. 子控件处理
QPushButton::mousePressEvent(event)
```

---

## 9. 事件循环启动流程

### 9.1 典型的 Qt 程序

```cpp
int main(int argc, char *argv[])
{
    QApplication a(argc, argv);  // 1. 创建 QApplication
    MainWindow w;                // 2. 创建主窗口
    w.show();                    // 3. 显示窗口
    return a.exec();             // 4. 进入事件循环
}
```

### 9.2 QEventLoop::exec() 实现

```cpp
// qeventloop.cpp:130-190

int QEventLoop::exec(ProcessEventsFlags flags)
{
    Q_D(QEventLoop);
    auto threadData = d->threadData.loadRelaxed();

    // === 检查 ===
    if (threadData->quitNow)
        return -1;
    if (d->inExec) {
        qWarning("QEventLoop::exec: instance %p has already called exec()", this);
        return -1;
    }

    // === 设置循环状态 ===
    struct LoopReference {
        QEventLoopPrivate *d;
        QMutexLocker<QMutex> &locker;
        bool exceptionCaught;

        LoopReference(QEventLoopPrivate *d, QMutexLocker<QMutex> &locker)
            : d(d), locker(locker), exceptionCaught(true)
        {
            d->inExec = true;
            d->exit.storeRelease(false);

            auto threadData = d->threadData.loadRelaxed();
            ++threadData->loopLevel;
            threadData->eventLoops.push(d->q_func());

            locker.unlock();
        }

        ~LoopReference()
        {
            // 清理工作
            locker.relock();
            auto threadData = d->threadData.loadRelaxed();
            QEventLoop *eventLoop = threadData->eventLoops.pop();
            d->inExec = false;
            --threadData->loopLevel;
        }
    };

    LoopReference ref(d, locker);

    // 移除已投递的 Quit 事件
    QCoreApplication *app = QCoreApplication::instance();
    if (app && app->thread() == thread())
        QCoreApplication::removePostedEvents(app, QEvent::Quit);

    // ========== 事件循环核心 ==========
    while (!d->exit.loadAcquire())          // 直到 exit() 被调用
        processEvents(flags | WaitForMoreEvents | EventLoopExec);
    // ==================================

    ref.exceptionCaught = false;
    return d->returnCode.loadRelaxed();
}
```

---

## 10. 完整流程图

### 10.1 从用户操作到 Qt 事件处理

```
用户点击鼠标
       ↓
┌─────────────────────────────────────────────────────────────┐
│  Windows 系统层                                              │
│  • 驱动程序检测到硬件中断                                    │
│  • Windows 将输入转换为消息                                  │
│  • MSG { hwnd=0x1234, message=WM_LBUTTONDOWN, ... }        │
│  • 放入线程消息队列                                          │
└─────────────────────────────────────────────────────────────┘
       ↓
┌─────────────────────────────────────────────────────────────┐
│  QEventLoop::exec()                                          │
│  while (!exit) {                                             │
│      processEvents(WaitForMoreEvents);                       │
│  }                                                           │
└─────────────────────────────────────────────────────────────┘
       ↓
┌─────────────────────────────────────────────────────────────┐
│  QEventDispatcherWin32::processEvents()                      │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  while (!interrupt) {                                 │  │
│  │      MSG msg;                                          │  │
│  │      PeekMessage(&msg, 0, 0, 0, PM_REMOVE);           │  │
│  │      if (PeekMessage 成功) {                           │  │
│  │          // 获取到 WM_LBUTTONDOWN                      │  │
│  │      }                                                 │  │
│  │  }                                                     │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
       ↓
┌─────────────────────────────────────────────────────────────┐
│  filterNativeEvent("windows_generic_MSG", &msg, 0)          │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  遍历所有已安装的 QAbstractNativeEventFilter           │  │
│  │  如果有过滤器返回 true → 停止处理                      │  │
│  │  否则继续                                              │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
       ↓
┌─────────────────────────────────────────────────────────────┐
│  TranslateMessage(&msg)                                      │
│  • 转换字符消息 (WM_KEYDOWN → WM_CHAR)                       │
└─────────────────────────────────────────────────────────────┘
       ↓
┌─────────────────────────────────────────────────────────────┐
│  DispatchMessage(&msg)                                       │
│  • 调用目标窗口的 WndProc                                    │
│  → QWindowsContext::windowsProc()                            │
└─────────────────────────────────────────────────────────────┘
       ↓
┌─────────────────────────────────────────────────────────────┐
│  windowsProc() - 消息分类                                    │
│  • 查找 Qt 窗口: findPlatformWindow(hwnd)                    │
│  • 确定事件类型: QtWindows::MouseEvent                       │
└─────────────────────────────────────────────────────────────┘
       ↓
┌─────────────────────────────────────────────────────────────┐
│  QWindowsPointerHandler::translateMouseEvent()               │
│  • 提取信息:                                                 │
│    - globalPos = QPoint(msg.pt.x, msg.pt.y)                 │
│    - localPos = globalPos - windowGeometry.topLeft()        │
│    - button = Qt::LeftButton                                 │
│  • 创建 QMouseEvent                                          │
└─────────────────────────────────────────────────────────────┘
       ↓
┌─────────────────────────────────────────────────────────────┐
│  QWindowSystemInterface::handleMouseEvent()                  │
│  • 将事件放入 Qt 事件队列或立即分发                           │
└─────────────────────────────────────────────────────────────┘
       ↓
┌─────────────────────────────────────────────────────────────┐
│  QCoreApplication::sendEvent()                               │
│  • 查找接收者: window()                                       │
│  • 调用 notify()                                             │
└─────────────────────────────────────────────────────────────┘
       ↓
┌─────────────────────────────────────────────────────────────┐
│  QWindow::event()                                            │
│  • switch (event->type())                                    │
│  • case QEvent::MouseButtonPress:                            │
│      • mousePressEvent(static_cast<QMouseEvent*>(event))    │
└─────────────────────────────────────────────────────────────┘
       ↓
┌─────────────────────────────────────────────────────────────┐
│  具体控件处理                                                 │
│  • QPushButton::mousePressEvent()                            │
│  • 发射 clicked() 信号                                        │
│  • 连接的槽函数执行                                           │
└─────────────────────────────────────────────────────────────┘
```

### 10.2 完整初始化时序图

```
┌─────────────────────────────────────────────────────────────────┐
│  main()                                                         │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  QApplication a(argc, argv);                              │  │
│  │      ↓                                                    │  │
│  │  QCoreApplication 构造                                    │  │
│  │      ↓                                                    │  │
│  │  QCoreApplicationPrivate::init()                          │  │
│  │      - 设置全局实例                                       │  │
│  │      - 初始化 locale                                      │  │
│  │      - 准备事件调度器 (延迟创建)                           │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  MainWindow w;                                                 │
│  w.show();                                                      │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  首次访问 eventDispatcher()                               │  │
│  │      ↓                                                    │  │
│  │  创建 QEventDispatcherWin32                               │  │
│  │      ↓                                                    │  │
│  │  QEventDispatcherWin32 构造                               │  │
│  │      - qt_create_internal_window(this)                    │  │
│  │      - qWindowsMessageWindowClassContext()  ← 首次访问    │  │
│  │          ↓                                                │  │
│  │      - QWindowsMessageWindowClassContext 构造             │  │
│  │          - 生成类名                                       │  │
│  │          - 填充 WNDCLASS                                  │  │
│  │          - RegisterClass(&wc)  ← 注册窗口类！            │  │
│  │      - CreateWindow(..., HWND_MESSAGE, ...) ← 创建窗口   │  │
│  │      - SetWindowLongPtr(..., GWLP_USERDATA, this)         │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  return a.exec();                                               │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  QCoreApplication::exec()                                 │  │
│  │      ↓                                                    │  │
│  │  QEventLoop eventLoop;                                    │  │
│  │      ↓                                                    │  │
│  │  QEventLoop::exec()                                       │  │
│  │      - 设置 in_exec = true                                │  │
│  │      - 移除 QEvent::Quit                                  │  │
│  │      ↓                                                    │  │
│  │  while (!exit) {                                          │  │
│  │      processEvents(WaitForMoreEvents);                    │  │
│  │  }                                                        │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  QEventDispatcherWin32::processEvents()                         │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  do {                                                     │  │
│  │      while (有消息) {                                     │  │
│  │          MSG msg;                                         │  │
│  │          PeekMessage(&msg, ...);                          │  │
│  │          filterNativeEvent(...);                          │  │
│  │          TranslateMessage(&msg);                          │  │
│  │          DispatchMessage(&msg);                           │  │
│  │      }                                                    │  │
│  │      if (canWait) {                                       │  │
│  │          MsgWaitForMultipleObjectsEx(...);                │  │
│  │      }                                                    │  │
│  │  } while (canWait);                                       │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 11. 关键代码位置索引

### 11.1 事件调度器核心

| 文件 | 路径 | 说明 |
|------|------|------|
| qeventdispatcher_win.cpp | `qtbase/src/corelib/kernel/` | Windows 事件调度器实现 |
| qeventdispatcher_win_p.h | `qtbase/src/corelib/kernel/` | 事件调度器私有头文件 |

### 11.2 关键函数位置

| 函数 | 文件 | 行号 | 功能 |
|------|------|------|------|
| `qt_internal_proc` | qeventdispatcher_win.cpp | 88 | 隐藏窗口的窗口过程 |
| `QWindowsMessageWindowClassContext` 构造 | qeventdispatcher_win.cpp | 236 | 窗口类注册 |
| `qt_create_internal_window` | qeventdispatcher_win.cpp | 275 | 创建隐藏消息窗口 |
| `processEvents` | qeventdispatcher_win.cpp | 476 | 事件循环核心 |
| `filterNativeEvent` | qabstracteventdispatcher.cpp | 554 | nativeEvent 过滤器 |
| `QEventLoop::exec` | qeventloop.cpp | 130 | 事件循环启动 |

### 11.3 自定义消息

```cpp
#define WM_QT_SOCKETNOTIFIER     WM_USER + 1  // Socket 通知
#define WM_QT_ACTIVATENOTIFIERS  WM_USER + 2  // 激活通知器
#define WM_QT_SENDPOSTEDEVENTS   WM_USER + 3  // 发送已投递事件
```

---

## 总结

Qt 在 Windows 上的事件机制是一个精巧设计的桥接系统：

1. **窗口类注册时机**：使用 `Q_GLOBAL_STATIC` 延迟初始化，在首次访问时触发
2. **processEvents 是核心**：负责获取消息、过滤、分发、等待的完整循环
3. **nativeEvent 过滤器**：在消息转换前拦截，允许自定义处理
4. **两个窗口过程**：`qt_internal_proc`（隐藏窗口）和 `windowsProc`（普通窗口）
5. **不是钩子**：通过标准窗口过程函数拦截消息
6. **默认只为顶级窗口创建 HWND**：子控件共享父窗口

---

## 5. 消息转换详解

### 5.1 窗口过程函数

Qt 注册了两个主要的窗口过程：

#### 5.1.1 隐藏窗口的窗口过程

```cpp
// qeventdispatcher_win.cpp:88-230
LRESULT QT_WIN_CALLBACK qt_internal_proc(HWND hwnd, UINT message, WPARAM wp, LPARAM lp)
{
    MSG msg;
    msg.hwnd = hwnd;
    msg.message = message;
    msg.wParam = wp;
    msg.lParam = lp;

    // 1. 通过原生事件过滤器
    QAbstractEventDispatcher* dispatcher = QAbstractEventDispatcher::instance();
    if (dispatcher && dispatcher->filterNativeEvent(
        QByteArrayLiteral("windows_dispatcher_MSG"), &msg, &result)) {
        return result;
    }

    // 2. 获取事件调度器
    auto q = reinterpret_cast<QEventDispatcherWin32 *>(
        GetWindowLongPtr(hwnd, GWLP_USERDATA));

    // 3. 处理特定消息
    switch (message) {
    case WM_QT_SOCKETNOTIFIER: {
        // Socket 通知 → QEvent::SockAct / QEvent::SockClose
        int type = /* 根据 WSAGETSELECTEVENT(lp) 判断 */;
        QEvent event(type < 3 ? QEvent::SockAct : QEvent::SockClose);
        QCoreApplication::sendEvent(sn->obj, &event);
        return 0;
    }
    case WM_QT_ACTIVATENOTIFIERS: {
        // 激活 socket 通知器
        d->activateSocketNotifiers();
        return 0;
    }
    case WM_TIMER: {
        // 定时器事件 → QTimerEvent
        QEventDispatcherWin32Private *d = q->d_func();
        if (d && !d->sendPostedEventsTimerId) {
            d->startPostedEventsTimer();
        }
        return 0;
    }
    }

    return DefWindowProc(hwnd, message, wp, lp);
}
```

#### 5.1.2 普通窗口的窗口过程

```cpp
// qwindowscontext.cpp:976-1250
bool QWindowsContext::windowsProc(HWND hwnd, UINT message,
                                  QtWindows::WindowsEventType et,
                                  WPARAM wParam, LPARAM lParam,
                                  LRESULT *result,
                                  QWindowsWindow **platformWindowPtr)
{
    // 1. 重建 MSG 结构
    MSG msg;
    msg.hwnd = hwnd;
    msg.message = message;
    msg.wParam = wParam;
    msg.lParam = lParam;

    // 2. 查找对应的 QWindow
    QWindowsWindow *platformWindow = findPlatformWindow(hwnd);

    // 3. 原生事件过滤
    if (!isInputMessage(msg.message) && filterNativeEvent(&msg, result))
        return true;
    if (platformWindow && filterNativeEvent(platformWindow->window(), &msg, result))
        return true;

    // 4. 根据事件类型分发
    switch (et) {
    case QtWindows::KeyDownEvent:
    case QtWindows::KeyEvent:
        // WM_KEYDOWN, WM_KEYUP, WM_CHAR → QKeyEvent
        return d->m_keyMapper.translateKeyEvent(platformWindow->window(),
                                                 hwnd, msg, result);

    case QtWindows::MouseEvent:
    case QtWindows::MouseWheelEvent:
    case QtWindows::LeaveEvent:
        // WM_*MOUSE* → QMouseEvent / QWheelEvent
        return d->m_pointerHandler.translateMouseEvent(platformWindow->window(),
                                                        hwnd, et, msg, result);

    case QtWindows::ResizeEvent:
        // WM_SIZE → QResizeEvent
        platformWindow->handleResized(static_cast<int>(wParam), lParam);
        return true;

    case QtWindows::FocusInEvent:
    case QtWindows::FocusOutEvent:
        // WM_SETFOCUS / WM_KILLFOCUS → QFocusEvent
        handleFocusEvent(et, platformWindow);
        return true;

    case QtWindows::ExposeEvent:
        // WM_PAINT → QPaintEvent
        return platformWindow->handleWmPaint(hwnd, message, wParam, lParam, result);

    case QtWindows::CloseEvent:
        // WM_CLOSE → QCloseEvent
        QWindowSystemInterface::handleCloseEvent(platformWindow->window());
        return true;

    // ... 更多事件类型
    }

    return false;
}
```

### 5.2 键盘事件转换

```cpp
// qwindowskeymapper.cpp:821-900
bool QWindowsKeyMapper::translateKeyEvent(QWindow *widget, HWND hwnd,
                                          const MSG &msg, LRESULT *result)
{
    *result = 0;

    // 1. 处理键盘布局改变
    if (msg.message == WM_INPUTLANGCHANGE) {
        changeKeyboard();
        return true;
    }

    // 2. 更新键盘状态映射
    if (msg.message != WM_CHAR && msg.message != WM_IME_CHAR)
        updateKeyMap(msg);

    // 3. 处理死键（如 `~` + `a` = `ã`）
    MSG peekedMsg;
    if (PeekMessage(&peekedMsg, hwnd, 0, 0, PM_NOREMOVE)
        && peekedMsg.message == WM_DEADCHAR) {
        return true;
    }

    // 4. 根据消息类型创建 Qt 事件
    QKeyEvent::Type keyType;
    int code;
    Qt::Key qtKey;

    if (msg.message == WM_KEYDOWN || msg.message == WM_SYSKEYDOWN) {
        keyType = QEvent::KeyPress;
        qtKey = Qt::Key(/* 虚拟键码转换 */);
    } else {
        keyType = QEvent::KeyRelease;
    }

    // 5. 发送事件到 Qt
    QKeyEvent event(keyType, qtKey, modifiers, text, autoRepeat, count);
    QWindowSystemInterface::handleKeyEvent(widget, event);

    return true;
}
```

### 5.3 鼠标事件转换

```cpp
// qwindowspointerhandler.cpp:350-740
bool QWindowsPointerHandler::translateMouseEvent(QWindow *window,
                                                 HWND hwnd,
                                                 QtWindows::WindowsEventType et,
                                                 const MSG &msg,
                                                 LRESULT *result)
{
    // 1. 获取鼠标位置
    const QPoint globalPos = QPoint(msg.pt.x, msg.pt);
    QPoint localPos = QWindowsGeometryHint::mapFromGlobal(hwnd, globalPos);

    // 2. 确定事件类型
    QEvent::Type type;
    Qt::MouseButton button;
    Qt::MouseButtons buttons;

    switch (et) {
    case QtWindows::MousePressedEvent:
        type = QEvent::MouseButtonPress;
        if (msg.message == WM_LBUTTONDOWN)
            button = Qt::LeftButton;
        else if (msg.message == WM_RBUTTONDOWN)
            button = Qt::RightButton;
        else if (msg.message == WM_MBUTTONDOWN)
            button = Qt::MiddleButton;
        break;

    case QtWindows::MouseReleasedEvent:
        type = QEvent::MouseButtonRelease;
        // ... 类似逻辑
        break;

    case QtWindows::MouseMoveEvent:
        type = QEvent::MouseMove;
        break;

    case QtWindows::MouseWheelEvent:
        type = QEvent::Wheel;
        // 滚轮事件特殊处理
        break;
    }

    // 3. 发送事件到 Qt
    QMouseEvent event(type, localPos, globalPos, button, buttons, modifiers);
    QWindowSystemInterface::handleMouseEvent(window, event);

    return true;
}
```

---

## 6. 控件与 HWND 的关系

### 6.1 不是每个控件都有 HWND

**重要概念：** Qt 默认只为顶级窗口创建 HWND，子控件共享父窗口。

### 6.2 创建 HWND 的条件

```cpp
// qwidget.cpp:2397-2417
void QWidgetPrivate::createWinId()
{
    const bool forceNativeWindow = q->testAttribute(Qt::WA_NativeWindow);
    if (!q->testAttribute(Qt::WA_WState_Created)
        || (forceNativeWindow && !q->internalWinId())) {

        // 创建原生窗口
    }
}
```

| 条件 | 说明 |
|------|------|
| `widget->isWindow()` | 顶级窗口（QMainWindow, QDialog） |
| `Qt::WA_NativeWindow` | 强制原生窗口属性 |
| `Qt::WA_PaintOnScreen` | 需要直接绘制 |
| `Qt::AA_NativeWindows` | 全局原生窗口模式 |

### 6.3 典型窗口结构

```
┌─────────────────────────────────────────────────────────────┐
│  QMainWindow (HWND: 0x1234)                                 │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  WndProc 接收所有消息                                 │   │
│  │                                                     │   │
│  │  WM_LBUTTONDOWN (在 x=100, y=50)                     │   │
│  │       ↓                                              │   │
│  │  Qt 判断坐标 → 属于 QPushButton                       │   │
│  │       ↓                                              │   │
│  │  发送 QMouseEvent 到 QPushButton                      │   │
│  └─────────────────────────────────────────────────────┘   │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                │
│  │  Button  │  │  Label   │  │  Edit    │  (无 HWND)     │
│  └──────────┘  └──────────┘  └──────────┘                │
└─────────────────────────────────────────────────────────────┘
```

### 6.4 消息在子控件间的分发

```cpp
// 1. Windows 消息到达父窗口 WndProc
WM_LBUTTONDOWN → QMainWindow.WndProc

// 2. Qt 转换为 QMouseEvent
QMouseEvent(QEvent::MouseButtonPress, localPos, ...)

// 3. Qt 根据坐标查找目标控件
QWidget *child = parent->childAt(pos);

// 4. 发送事件到子控件
QCoreApplication::sendEvent(child, &event);

// 5. 子控件处理
QPushButton::mousePressEvent(event)
```

---

## 7. 完整流程图

### 7.1 从用户操作到 Qt 事件处理

```
用户点击鼠标
       ↓
┌─────────────────────────────────────────────────────────────┐
│  Windows 系统层                                              │
│  • 驱动程序检测到硬件中断                                    │
│  • Windows 将输入转换为消息                                  │
│  • MSG { hwnd=0x1234, message=WM_LBUTTONDOWN, ... }        │
│  • 放入线程消息队列                                          │
└─────────────────────────────────────────────────────────────┘
       ↓
┌─────────────────────────────────────────────────────────────┐
│  Qt 事件循环 - processEvents()                               │
│  • PeekMessage(&msg, 0, 0, 0, PM_REMOVE)                    │
│  • 获取消息: WM_LBUTTONDOWN                                  │
└─────────────────────────────────────────────────────────────┘
       ↓
┌─────────────────────────────────────────────────────────────┐
│  TranslateMessage(&msg)                                      │
│  • 转换字符消息 (WM_KEYDOWN → WM_CHAR)                       │
└─────────────────────────────────────────────────────────────┘
       ↓
┌─────────────────────────────────────────────────────────────┐
│  DispatchMessage(&msg)                                       │
│  • 调用窗口过程函数                                           │
│  → QWindowsContext::windowsProc()                            │
└─────────────────────────────────────────────────────────────┘
       ↓
┌─────────────────────────────────────────────────────────────┐
│  windowsProc() - 消息分类                                    │
│  • 查找 Qt 窗口: findPlatformWindow(hwnd)                    │
│  • 确定事件类型: QtWindows::MouseEvent                       │
│  • 调用转换器                                                 │
└─────────────────────────────────────────────────────────────┘
       ↓
┌─────────────────────────────────────────────────────────────┐
│  QWindowsPointerHandler::translateMouseEvent()               │
│  • 提取信息:                                                 │
│    - globalPos = QPoint(msg.pt.x, msg.pt.y)                 │
│    - localPos = globalPos - windowGeometry.topLeft()        │
│    - button = Qt::LeftButton                                 │
│  • 创建 QMouseEvent                                          │
└─────────────────────────────────────────────────────────────┘
       ↓
┌─────────────────────────────────────────────────────────────┐
│  QWindowSystemInterface::handleMouseEvent()                  │
│  • 将事件放入 Qt 事件队列或立即分发                           │
└─────────────────────────────────────────────────────────────┘
       ↓
┌─────────────────────────────────────────────────────────────┐
│  QCoreApplication::sendEvent()                               │
│  • 查找接收者: window()                                       │
│  • 调用 notify()                                             │
└─────────────────────────────────────────────────────────────┘
       ↓
┌─────────────────────────────────────────────────────────────┐
│  QWindow::event()                                            │
│  • switch (event->type())                                    │
│  • case QEvent::MouseButtonPress:                            │
│      • mousePressEvent(static_cast<QMouseEvent*>(event))    │
└─────────────────────────────────────────────────────────────┘
       ↓
┌─────────────────────────────────────────────────────────────┐
│  具体控件处理                                                 │
│  • QPushButton::mousePressEvent()                            │
│  • 发射 clicked() 信号                                        │
│  • 连接的槽函数执行                                           │
└─────────────────────────────────────────────────────────────┘
```

### 7.2 Socket 通知流程

```
应用程序调用 QSocketNotifier
       ↓
┌─────────────────────────────────────────────────────────────┐
│  QEventDispatcherWin32::registerSocketNotifier()             │
│  • WSAAsyncSelect(socket, internalHwnd,                      │
│                  WM_QT_SOCKETNOTIFIER, FD_READ | ...)       │
└─────────────────────────────────────────────────────────────┘
       ↓
┌─────────────────────────────────────────────────────────────┐
│  Windows Socket 层                                           │
│  • socket 有数据可读时                                        │
│  • Windows 发送 WM_QT_SOCKETNOTIFIER 到 internalHwnd         │
└─────────────────────────────────────────────────────────────┘
       ↓
┌─────────────────────────────────────────────────────────────┐
│  qt_internal_proc() - 隐藏窗口过程                           │
│  • case WM_QT_SOCKETNOTIFIER:                                │
│  • 确定 socket 和事件类型                                     │
└─────────────────────────────────────────────────────────────┘
       ↓
┌─────────────────────────────────────────────────────────────┐
│  创建 QEvent::SockAct                                        │
│  • QCoreApplication::sendEvent(socketNotifier, &event)      │
└─────────────────────────────────────────────────────────────┘
       ↓
┌─────────────────────────────────────────────────────────────┐
│  QSocketNotifier::event()                                    │
│  • 发射 activated() 信号                                     │
└─────────────────────────────────────────────────────────────┘
```

### 7.3 定时器流程

```
应用程序启动 QTimer
       ↓
┌─────────────────────────────────────────────────────────────┐
│  QEventDispatcherWin32::registerTimer()                      │
│  • SetTimer(internalHwnd, timerId, interval, NULL)          │
└─────────────────────────────────────────────────────────────┘
       ↓
┌─────────────────────────────────────────────────────────────┐
│  Windows 定时器服务                                           │
│  • 间隔到达                                                   │
│  • Windows 发送 WM_TIMER 到 internalHwnd                     │
└─────────────────────────────────────────────────────────────┘
       ↓
┌─────────────────────────────────────────────────────────────┐
│  processEvents() 收到 WM_TIMER                               │
│  • 检查是否为 sendPostedEventsTimerId (跳过)                 │
│  • 避免重复处理                                               │
└─────────────────────────────────────────────────────────────┘
       ↓
┌─────────────────────────────────────────────────────────────┐
│  DispatchMessage()                                           │
│  → qt_internal_proc(hwnd, WM_TIMER, wParam, lParam)         │
└─────────────────────────────────────────────────────────────┘
       ↓
┌─────────────────────────────────────────────────────────────┐
│  qt_internal_proc() - WM_TIMER 处理                          │
│  • 查找 WinTimerInfo                                          │
│  • 创建 QTimerEvent                                          │
│  • QCoreApplication::postEvent(object, new QTimerEvent(id)) │
└─────────────────────────────────────────────────────────────┘
       ↓
┌─────────────────────────────────────────────────────────────┐
│  QObject::event()                                            │
│  • case QEvent::Timer:                                       │
│  • timerEvent(static_cast<QTimerEvent*>(event))             │
└─────────────────────────────────────────────────────────────┘
```

---

## 8. 事件循环启动流程 - 从 QApplication 开始

### 8.1 程序入口与 QApplication 初始化

一个典型的 Qt 程序：

```cpp
int main(int argc, char *argv[])
{
    QApplication a(argc, argv);  // 1. 创建 QApplication
    MainWindow w;                // 2. 创建主窗口
    w.show();                    // 3. 显示窗口
    return a.exec();             // 4. 进入事件循环
}
```

### 8.2 QApplication 构造流程

```
┌─────────────────────────────────────────────────────────────────┐
│  1. QApplication 构造                                            │
│     └── QCoreApplication(argc, argv)                            │
│         └── QCoreApplicationPrivate::init()                     │
│             ├── 初始化 locale                                   │
│             ├── 设置 application 名称                           │
│             ├── 初始化事件调度器 (!)                            │
│             └── 初始化平台集成                                  │
└─────────────────────────────────────────────────────────────────┘
```

**关键代码 - qcoreapplication.cpp:779**

```cpp
void QCoreApplicationPrivate::init()
{
    // 1. 设置全局实例
    QCoreApplication::self = q;

    // 2. 初始化 locale
    initLocale();

    // 3. 存储应用名称/版本
    coreappdata()->application = appName();

    // 4. 其他初始化...
}
```

### 8.3 事件调度器创建时机

事件调度器在**首次访问时延迟创建**：

```cpp
// 首次调用 QThreadData::eventDispatcher 时
QAbstractEventDispatcher *QAbstractEventDispatcher::instance(QThread *thread)
{
    QThreadData *data = thread ? QThreadData::get2(thread) : QThreadData::current();
    // 如果还没有创建，会调用 createEventDispatcher()
    return data->eventDispatcher.loadRelaxed();
}
```

在 Windows 上，会创建 `QEventDispatcherWin32`：

**关键代码 - qeventdispatcher_win.cpp:430-436**

```cpp
QEventDispatcherWin32::QEventDispatcherWin32(QEventDispatcherWin32Private &dd, QObject *parent)
    : QAbstractEventDispatcher(dd, parent)
{
    Q_D(QEventDispatcherWin32);

    // 创建隐藏的消息窗口
    d->internalHwnd = qt_create_internal_window(this);
}
```

### 8.4 app.exec() - 启动事件循环

**关键代码 - qcoreapplication.cpp:1430-1456**

```cpp
int QCoreApplication::exec()
{
    // 1. 检查实例
    if (!QCoreApplicationPrivate::checkInstance("exec"))
        return -1;

    // 2. 必须在主线程调用
    QThreadData *threadData = self->d_func()->threadData.loadAcquire();
    if (threadData != QThreadData::current()) {
        qWarning("QCoreApplication::exec: Must be called from the main thread");
        return -1;
    }

    // 3. 检查是否已有事件循环
    if (!threadData->eventLoops.isEmpty()) {
        qWarning("QCoreApplication::exec: The event loop is already running");
        return -1;
    }

    // 4. 创建并启动事件循环
    threadData->quitNow = false;
    QEventLoop eventLoop;           // ← 创建事件循环对象
    self->d_func()->in_exec = true;
    int returnCode = eventLoop.exec(QEventLoop::ApplicationExec); // ← 启动!
    threadData->quitNow = false;

    return returnCode;
}
```

### 8.5 QEventLoop::exec() - 事件循环核心

**关键代码 - qeventloop.cpp:130-190**

```cpp
int QEventLoop::exec(ProcessEventsFlags flags)
{
    Q_D(QEventLoop);

    // 1. 检查是否已退出
    if (threadData->quitNow)
        return -1;

    // 2. 检查是否已在执行
    if (d->inExec) {
        qWarning("QEventLoop::exec: instance %p has already called exec()", this);
        return -1;
    }

    // 3. 设置循环状态
    d->inExec = true;
    d->exit.storeRelease(false);
    ++threadData->loopLevel;
    threadData->eventLoops.push(this);

    // 4. 移除已投递的 Quit 事件
    QCoreApplication::removePostedEvents(app, QEvent::Quit);

    // ========== 事件循环核心 ==========
    while (!d->exit.loadAcquire())          // 直到 exit() 被调用
        processEvents(flags | WaitForMoreEvents | EventLoopExec);
    // ==================================

    // 5. 清理
    d->inExec = false;
    --threadData->loopLevel;

    return d->returnCode.loadRelaxed();
}
```

### 8.6 完整启动流程图

```
┌─────────────────────────────────────────────────────────────────┐
│  main() 函数                                                     │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  QApplication a(argc, argv);                              │  │
│  │       ↓                                                   │  │
│  │  QCoreApplication::QCoreApplication()                    │  │
│  │       ↓                                                   │  │
│  │  QCoreApplicationPrivate::init()                         │  │
│  │       ↓                                                   │  │
│  │  • 设置全局实例                                            │  │
│  │  • 初始化 locale                                           │  │
│  │  • 准备事件调度器 (延迟创建)                                │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  w.show() / 创建窗口                                            │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  • 首次访问 eventDispatcher 时创建 QEventDispatcherWin32  │  │
│  │  • QEventDispatcherWin32 构造:                             │  │
│  │      ↓                                                    │  │
│  │  • qt_create_internal_window()                            │  │
│  │      ↓                                                    │  │
│  │  • CreateWindow(..., HWND_MESSAGE, ...)                   │  │
│  │      ↓                                                    │  │
│  │  • 注册窗口类 (Q_GLOBAL_STATIC 首次触发)                   │  │
│  │      ↓                                                    │  │
│  │  • RegisterClass(&wc)                                     │  │
│  │      wc.lpfnWndProc = qt_internal_proc                    │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  return a.exec();                                               │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  QCoreApplication::exec()                                 │  │
│  │      ↓                                                    │  │
│  │  QEventLoop eventLoop;                                    │  │
│  │      ↓                                                    │  │
│  │  eventLoop.exec(ApplicationExec)                          │  │
│  │      ↓                                                    │  │
│  │  while (!exit) {                                          │  │
│  │      processEvents(WaitForMoreEvents);                    │  │
│  │  }                                                        │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  processEvents() - 消息循环                                       │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  while (true) {                                           │  │
│  │      MSG msg;                                              │  │
│  │                                                            │  │
│  │      // 从 Windows 消息队列获取消息                          │  │
│  │      if (PeekMessage(&msg, 0, 0, 0, PM_REMOVE)) {         │  │
│  │          TranslateMessage(&msg);  // 转换字符消息           │  │
│  │          DispatchMessage(&msg);   // 分发到 WndProc        │  │
│  │      } else {                                              │  │
│  │          // 没有消息，等待新消息                             │  │
│  │          MsgWaitForMultipleObjectsEx(..., INFINITE, ...);  │  │
│  │          break;  // 退出内层循环，检查 exit 标志            │  │
│  │      }                                                      │  │
│  │  }                                                          │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  DispatchMessage(&msg)                                           │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  → qt_internal_proc(hwnd, message, wParam, lParam)        │  │
│  │      (隐藏窗口: internalHwnd)                              │  │
│  │  → QWindowsContext::windowsProc(...)                      │  │
│  │      (普通窗口: QMainWindow 等)                            │  │
│  │      ↓                                                    │  │
│  │  消息转换: WM_* → QEvent                                   │  │
│  │      ↓                                                    │  │
│  │  QCoreApplication::sendEvent(receiver, event)             │  │
│  │      ↓                                                    │  │
│  │  QObject::event() → 具体控件处理                           │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### 8.7 事件循环退出

```cpp
// 用户调用 quit() 或 close() 主窗口时
QCoreApplication::quit()
    ↓
QCoreApplication::exit(0)
    ↓
QEventLoop::exit()
    ↓
d->exit.storeRelease(true)      // 设置退出标志
    ↓
eventDispatcher->wakeUp()       // 唤醒事件循环
    ↓
PostMessage(internalHwnd, WM_QT_SENDPOSTEDEVENTS, ...)
    ↓
processEvents() 返回
    ↓
QEventLoop::exec() 的 while (!exit) 循环结束
    ↓
返回到 main()
```

### 8.8 关键时序总结

| 时刻 | 操作 | 说明 |
|------|------|------|
| **1** | `QApplication a(argc, argv)` | 创建应用对象，初始化基础结构 |
| **2** | 首次访问 `eventDispatcher()` | 创建 `QEventDispatcherWin32` |
| **2.1** | `QEventDispatcherWin32` 构造 | 创建隐藏消息窗口 |
| **2.2** | `RegisterClass()` | 注册窗口类（首次） |
| **2.3** | `CreateWindow(HWND_MESSAGE)` | 创建消息窗口 |
| **3** | `w.show()` | 创建并显示窗口（如有 HWND） |
| **4** | `a.exec()` | 启动事件循环 |
| **4.1** | `QEventLoop::exec()` | 进入 while 循环 |
| **4.2** | `processEvents()` | 反复处理消息 |
| **5** | `a.quit()` | 设置退出标志 |
| **6** | 返回 `main()` | 程序结束 |

---

## 9. 关键代码位置索引

### 8.1 事件调度器核心

| 文件 | 路径 | 说明 |
|------|------|------|
| qeventdispatcher_win.cpp | `qtbase/src/corelib/kernel/` | Windows 事件调度器实现 |
| qeventdispatcher_win_p.h | `qtbase/src/corelib/kernel/` | 事件调度器私有头文件 |

### 8.2 平台插件

| 文件 | 路径 | 说明 |
|------|------|------|
| qwindowscontext.cpp | `qtbase/src/plugins/platforms/windows/` | Windows 上下文和主窗口过程 |
| qwindowscontext.h | `qtbase/src/plugins/platforms/windows/` | Windows 上下文头文件 |
| qwindowskeymapper.cpp | `qtbase/src/plugins/platforms/windows/` | 键盘消息转换 |
| qwindowspointerhandler.cpp | `qtbase/src/plugins/platforms/windows/` | 鼠标/触摸消息转换 |
| qwindowswindow.cpp | `qtbase/src/plugins/platforms/windows/` | 窗口实现 |

### 8.3 关键函数位置

| 函数 | 文件 | 行号 |
|------|------|------|
| `qt_internal_proc` | qeventdispatcher_win.cpp | 88 |
| `QWindowsMessageWindowClassContext` 构造 | qeventdispatcher_win.cpp | 236 |
| `qt_create_internal_window` | qeventdispatcher_win.cpp | 275 |
| `processEvents` | qeventdispatcher_win.cpp | 476 |
| `QWindowsContext::windowsProc` | qwindowscontext.cpp | 976 |
| `QWindowsKeyMapper::translateKeyEvent` | qwindowskeymapper.cpp | 821 |
| `QWindowsPointerHandler::translateMouseEvent` | qwindowspointerhandler.cpp | ~350 |
| `QWidgetPrivate::createWinId` | qwidget.cpp | 2397 |

### 8.4 自定义消息

```cpp
// Qt 自定义的 Windows 消息
#define WM_QT_SOCKETNOTIFIER     WM_USER + 1  // Socket 通知
#define WM_QT_ACTIVATENOTIFIERS  WM_USER + 2  // 激活通知器
#define WM_QT_SENDPOSTEDEVENTS   WM_USER + 3  // 发送已投递事件
```

---

## 总结

Qt 在 Windows 上的事件机制是一个精巧设计的桥接系统：

1. **不是使用钩子**，而是通过标准窗口过程函数拦截消息
2. **每个线程一个隐藏消息窗口**，用于接收非 UI 相关的系统消息
3. **默认只为顶级窗口创建 HWND**，子控件共享父窗口，由 Qt 内部分发
4. **消息在 WndProc 中转换为 QEvent**，然后通过 Qt 事件系统分发
5. **支持原生事件过滤器**，允许应用程序在转换前/后拦截消息

这种设计既保持了 Qt 的跨平台抽象，又能够高效地利用 Windows 原生消息系统。
