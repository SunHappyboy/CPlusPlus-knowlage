# Qt 面试知识点整理

## 一、Qt 整体架构

### 1.1 Qt 框架分层

```
┌─────────────────────────────────────────────────────────┐
│                    Qt 应用层                              │
│            (用户代码 + UI界面)                            │
├─────────────────────────────────────────────────────────┤
│                  Qt 核心模块                              │
│  ┌───────────┐ ┌───────────┐ ┌───────────┐             │
│  │   QtGui   │ │  QtWidgets│ │ QtNetwork │             │
│  │ (图形/绘图) │ │ (UI控件)  │ │  (网络)   │             │
│  └───────────┘ └───────────┘ └───────────┘             │
│  ┌───────────┐ ┌───────────┐ ┌───────────┐             │
│  │ QtCore    │ │ QtCharts  │ │ QtSql     │             │
│  │ (核心基础) │ │  (图表)   │ │  (数据库) │             │
│  └───────────┘ └───────────┘ └───────────┘             │
├─────────────────────────────────────────────────────────┤
│                   Qt 平台抽象层                           │
│           (事件循环、窗口系统、文件系统)                   │
├─────────────────────────────────────────────────────────┤
│                  操作系统层                               │
│        Windows / Linux / macOS / Android / iOS          │
└─────────────────────────────────────────────────────────┘
```

### 1.2 Qt4 与 Qt5/Qt6 的主要区别

```cpp
// ========== Qt4 vs Qt5/Qt6 ==========

// 1. connect 写法变化
// Qt4 风格（需要运行时检查，容易出错）
connect(sender, SIGNAL(valueChanged(int)), receiver, SLOT(updateValue(int)));

// Qt5 风格（编译时检查，类型安全）
connect(sender, &Sender::valueChanged, receiver, &Receiver::updateValue);

// 2. 头文件变化
// Qt4
#include <QtGui>
#include <QtCore>

// Qt5/Qt6（模块化）
#include <QWidget>      // QtWidgets
#include <QPainter>     // QtGui
#include <QChart>       // QtCharts

// 3. QString 初始化
// Qt4
QString str = "Hello";  // 隐式转换

// Qt5/Qt6（推荐显式转换）
QString str = QStringLiteral("Hello");
QString str2 = QLatin1String("Hello");

// 4. 高分屏支持
// Qt5.6+ / Qt6
QApplication::setAttribute(Qt::AA_EnableHighDpiScaling);
QApplication::setAttribute(Qt::AA_UseHighDpiPixmaps);

// 5. foreach 保留关键字变为普通宏
// Qt4/Qt5
foreach(QString item, list) { ... }

// Qt6（推荐使用范围for）
for(const QString& item : list) { ... }
```

### 1.3 跨平台原理

```cpp
// Qt 跨平台通过 "Write Once, Deploy Everywhere" 实现

// 1. 条件编译
#ifdef Q_OS_WIN
    // Windows 特定代码
    #include <windows.h>
#elif defined(Q_OS_LINUX)
    // Linux 特定代码
    #include <unistd.h>
#elif defined(Q_OS_MACOS)
    // macOS 特定代码
#endif

// 2. 路径处理（自动适配平台）
QString path = QDir::homePath();  // 自动使用正确分隔符
QFileInfo fi(path);

// 3. 线程相关（跨平台API）
QThread::msleep(100);  // 跨平台睡眠

// 4. 编译配置
// .pro 文件
win32 {
    SOURCES += windows_specific.cpp
    LIBS += -luser32
}
unix:!macx {
    SOURCES += linux_specific.cpp
    LIBS += -lpthread
}
macx {
    SOURCES += mac_specific.cpp
}
```

---

## 二、Qt 核心机制 ⭐

### 2.1 信号与槽机制

**信号槽是 Qt 实现的观察者模式，用于对象间通信**

```cpp
// ========== 基本用法 ==========

// 1. 定义信号（signals 部分自动由 moc 实现）
class DataSource : public QObject {
    Q_OBJECT
public:
    void setData(int value) {
        m_data = value;
        emit dataChanged(value);  // 发射信号
    }

signals:
    void dataChanged(int newValue);      // 信号声明
    void progressUpdated(int value);     // 可以声明多个
    void errorOccurred(const QString& msg);

private:
    int m_data;
};

// 2. 定义槽函数（普通成员函数、静态函数、lambda都可以是槽）
class DataDisplay : public QWidget {
    Q_OBJECT
public:
    DataDisplay(QWidget* parent = nullptr) : QWidget(parent) {
        label = new QLabel(this);
    }

public slots:  // slots 关键字（Qt5 可省略）
    void onDataChanged(int value) {
        label->setText(QString("Value: %1").arg(value));
    }

    void onError(const QString& msg) {
        QMessageBox::warning(this, "Error", msg);
    }

private:
    QLabel* label;
};

// 3. 连接信号与槽
DataSource* source = new DataSource();
DataDisplay* display = new DataDisplay();

// Qt5 风格（推荐）- 编译时类型检查
connect(source, &DataSource::dataChanged,
        display, &DataDisplay::onDataChanged);

// 使用 Lambda 表达式
connect(source, &DataSource::progressUpdated, [](int value) {
    qDebug() << "Progress:" << value;
});

// 带参数的 Lambda
connect(source, &DataSource::dataChanged, display, [display](int value) {
    // display 捕获，避免悬垂引用
    display->setWindowTitle(QString("Value: %1").arg(value));
});

// 4. 断开连接
disconnect(source, &DataSource::dataChanged,
           display, &DataDisplay::onDataChanged);

// 断开所有连接
disconnect(source);
```

**Qt::ConnectionType 五种连接方式**

```cpp
// ========== 连接方式详解 ==========

class Sender : public QObject {
    Q_OBJECT
public:
    Sender() { threadId = std::this_thread::get_id(); }

    void sendSignal() {
        qDebug() << "Sender thread:" << threadId;
        emit testSignal();
    }

signals:
    void testSignal();

private:
    std::thread::id threadId;
};

class Receiver : public QObject {
    Q_OBJECT
public:
    Receiver() { threadId = std::this_thread::get_id(); }

public slots:
    void onReceive() {
        qDebug() << "Receiver thread:" << threadId;
    }

private:
    std::thread::id threadId;
};

// ========== 1. AutoConnection（默认）==========
// 同线程：DirectConnection
// 跨线程：QueuedConnection
connect(sender, &Sender::testSignal,
        receiver, &Receiver::onReceive,
        Qt::AutoConnection);

// ========== 2. DirectConnection（直接连接）==========
// 槽函数在信号发射线程立即执行（同步）
connect(sender, &Sender::testSignal,
        receiver, &Receiver::onReceive,
        Qt::DirectConnection);

// ========== 3. QueuedConnection（队列连接）==========
// 槽函数在接收者线程的事件循环中执行（异步）
connect(sender, &Sender::testSignal,
        receiver, &Receiver::onReceive,
        Qt::QueuedConnection);

// ========== 4. BlockingQueuedConnection（阻塞队列）==========
// 信号发射线程会阻塞等待槽函数执行完成
// 只能用于跨线程，同线程会死锁！
connect(sender, &Sender::testSignal,
        receiver, &Receiver::onReceive,
        Qt::BlockingQueuedConnection);

// ========== 5. UniqueConnection（唯一连接）==========
// 防止重复连接，可与其他标志位或运算
connect(sender, &Sender::testSignal,
        receiver, &Receiver::onReceive,
        Qt::UniqueConnection);
```

**信号槽的优势与不足**

| 优势 | 不足 |
|------|------|
| 松耦合，对象无需直接引用 | 运行时开销比直接函数调用大 |
| 类型安全（Qt5+） | 调试相对困难 |
| 支持跨线程通信 | 信号不能是模板函数 |
| 一对多连接 | 槽函数返回值难以获取 |

---

#### 信号槽底层实现原理

// 【知识铺垫】Qt 信号槽采用观察者模式，通过 MOC 预处理实现反射机制
// 【核心概念】emit 不是关键字，只是宏定义的 #define emit
// 【面试要点】信号槽是线程安全的吗？跨线程如何保证安全？

**完整调用流程**

```
【代码层】 emit signal(args)
    ↓
【MOC层】 void ClassName::signal(args)  // MOC 生成的函数
    ↓
【核心层】 QMetaObject::activate(sender, signal_index, args)
    ↓
【数据层】 获取 QObjectPrivate::ConnectionVector
    ↓
【遍历层】 遍历该信号的所有 Connection（链表结构）
    ↓
【分发层】 根据 Qt::ConnectionType 分发：
    ├─ DirectConnection:    直接调用槽函数（同栈，同步）
    ├─ QueuedConnection:    创建 QMetaCallEvent，postEvent（异步）
    ├─ BlockingQueued:      postEvent + QSemaphore::wait()（阻塞等待）
    └─ AutoConnection:      判断线程，自动选择 Direct 或 Queued
    ↓
【执行层】 槽函数执行 (通过 qt_metacall 或直接函数指针调用)
```

// 【要点总结】
// 1. emit 只是语法糖，实际调用 MOC 生成的函数
// 2. 连接存储在 QObjectPrivate 中，使用链表结构
// 3. 跨线程时会创建 QMetaCallEvent 投递到事件队列
// 4. ConnectionType 决定调用方式，影响线程安全性

**QObjectPrivate 连接存储结构**

```cpp
// qobject_p.h - Qt 内部连接存储机制
// 【设计模式】使用 Pimpl 模式（私有实现），隐藏实现细节

namespace QObjectPrivate {

// ========== 单个连接结构 ==========
// 【作用】存储一个信号-槽连接的所有信息
class Connection {
public:
    QObject *receiver;           // 接收者对象指针：槽函数所属对象
    QObject *sender;             // 发送者对象指针（Qt5.15+，用于调试）

    // 槽函数指针（多种形式，使用 union 共享内存）
    union {
        void (**slot)(void **);      // 函数指针数组：用于 lambda 或函数对象
        int method_offset;           // 方法在元对象表中的偏移：用于成员函数槽
        QStaticSlotResultFunc func;  // 静态函数指针
        // union: 多个成员共享同一块内存，节省空间
    };

    Connection *next;            // 链表下一个节点：实现多个连接
    Connection *prev;            // 链表前一个节点（Qt5 优化，便于删除）

    // 连接属性（位域技术，16个位被分割使用）
    ushort connectionType : 3;   // Qt::ConnectionType (0-4): Auto/Direct/Queued/Blocking/Unique
    ushort isSlotObject : 1;     // 是否是槽对象对象（而非普通成员函数）
    ushort ownArgumentTypes : 1;  // 是否拥有参数类型（是否需要负责释放）
    ushort argumentTypes : 12;   // 参数类型数量（最多12个参数）

    int *argumentTypes;          // 参数类型数组：用于运行时类型检查（Qt5 新语法）

    // 【位域说明】ushort 16位，分割成 3+1+1+12 = 17位？实际超过会编译失败
    // Qt 实际上可能使用多个位域或更大的整数
};

// ========== 连接链表（每个信号一个）==========
// 【作用】管理某个信号的所有连接（一个信号可连接多个槽）
class ConnectionList {
public:
    Connection *first;           // 链表头节点：第一个连接
    Connection *last;            // 链表尾节点：优化追加操作，不需要遍历

    // 并发控制（原子操作，线程安全）
    QAtomicInt inUse;           // 使用计数：防止重入问题时多次访问
    QAtomicInt orphaned;        // 孤儿连接计数：发送者被删除但连接未清理
};

// ========== 连接向量（所有信号的集合）==========
// 【作用】管理类的所有信号的连接列表
class ConnectionVector {
public:
    ConnectionList *vector;      // ConnectionList 数组：每个信号一个链表
    int count;                   // 信号数量：数组大小

    // 动态扩容：当新增信号时自动扩展
    void grow(int signalCount);
};

// ========== 数据结构图解 ==========
/*
 * ConnectionVector (类级别)
 *   ├─ ConnectionList[0] → Connection → ... → Connection (signal1 的所有连接)
 *   ├─ ConnectionList[1] → Connection → ... → Connection (signal2 的所有连接)
 *   └─ ConnectionList[n] → ... (signal n 的所有连接)
 *
 * 每个 Connection 包含：
 *   - 接收者对象
 *   - 槽函数指针（或方法偏移）
 *   - 连接类型
 *   - 下一个连接指针
 */

} // namespace QObjectPrivate

// ========== QObject::d_ptr 包含连接数据 ==========
// 【Pimpl 模式】QObject 使用 d_ptr 指向私有数据，保持二进制兼容性
class QObjectPrivate : public QObjectPrivateData {
public:
    ConnectionVector connectionLists;  // 所有信号-槽连接的存储
    // ... 其他私有数据（父对象、子对象列表、线程所属等）
};
```

// 【要点总结】
// 1. 使用链表存储连接：一个信号可连接多个槽（一对多）
// 2. 使用 union 节省内存：不同类型槽共享存储空间
// 3. 位域技术：节省连接属性的存储空间
// 4. 原子操作：inUse 确保并发访问安全

**MOC 生成的信号函数源码**

```cpp
// moc_myclass.cpp - MOC 自动生成的信号实现
// 【来源】由 myclass.h 经 MOC 处理后生成

// 1. 信号函数实现（emit signal() 展开后就是这个）
// 【命名规则】void ClassName::signalName(参数)
void MyClass::dataChanged(int _t1)
{
    // _t1 是参数值（MOC 自动添加下划线前缀避免命名冲突）
    // void** _a 是通用参数数组：Qt 用数组传递所有参数（包括返回值）

    void *_a[] = {
        nullptr,  // [0] 返回值位置（信号无返回值，设为 nullptr）
        const_cast<void*>(reinterpret_cast<const void*>(&_t1))  // [1] 参数1
        // [2] 参数2（如果有）...
    };

    // 调用 QMetaObject::activate 激活信号
    QMetaObject::activate(
        this,                           // sender
        &staticMetaObject,             // 元对象
        staticMetaObject.methodOffset(0),  // 信号索引
        _a                             // 参数数组
    );
}

// 2. 无参数信号
void MyClass::progressUpdated()
{
    void *_a[] = { nullptr };
    QMetaObject::activate(this, &staticMetaObject,
                          staticMetaObject.methodOffset(1), _a);
}
```

**QMetaObject::activate 核心实现（简化）**

```cpp
// qmetaobject.cpp - 信号激活核心函数
// 【作用】发射信号时调用，遍历所有连接并调用对应槽函数
// 【面试要点】这是信号槽机制的核心，了解它对理解跨线程调用很重要

void QMetaObject::activate(QObject *sender,
                           const QMetaObject *m,
                           int local_signal_index,  // 本类中的信号索引
                           void **argv)             // 参数数组
{
    // 计算全局信号索引 = 父类方法数 + 本类信号索引
    int signal_index = m->methodOffset() + local_signal_index;

    // ========== 1. 获取连接列表 ==========
    // QObjectPrivate::get() 获取对象的私有数据（d_ptr）
    QObjectPrivate *senderPriv = QObjectPrivate::get(sender);
    QObjectPrivate::ConnectionVector *connectionVector =
        &senderPriv->connectionLists;

    // 检查信号索引是否有效（防止越界访问）
    if (signal_index >= connectionVector->count)
        return;  // 没有连接，直接返回

    // 获取该信号对应的连接链表
    QObjectPrivate::ConnectionList *list =
        &connectionVector->vector[signal_index];

    // ========== 2. 遍历所有连接 ==========
    QObjectPrivate::Connection *c = list->first;

    // 标记使用中（防止重入时连接被删除）
    // 【重入问题】信号触发过程中可能再次触发同一信号
    while (c) {
        QObjectPrivate::Connection *next = c->next;  // 先保存下一个，防止 c 被删除
        QObject *receiver = c->receiver;

        if (!receiver) {
            c = next;
            continue;  // 接收者已被删除，跳过
        }

        // ========== 3. 根据连接类型分发 ==========
        switch (c->connectionType) {
            case Qt::AutoConnection:
                // 【自动连接】Qt 根据线程自动选择同步/异步
                // 同线程：DirectConnection（直接调用，性能好）
                // 跨线程：QueuedConnection（队列调用，安全）
                if (QThread::currentThread() == receiver->thread()) {
                    // 同线程：直接调用（同步）
                    QObjectPrivate::activate(receiver, c, argv);
                } else {
                    // 跨线程：投递事件到接收者线程的事件队列（异步）
                    // 【线程安全】通过事件队列实现跨线程通信
                    QCoreApplication::postEvent(
                        receiver,
                        new QMetaCallEvent(c, argv)
                    );
                }
                break;

            case Qt::DirectConnection:
                // 【直接连接】强制在发送者线程调用（不管接收者在哪个线程）
                // 【危险】跨线程时会导致槽函数在发送者线程执行，可能不安全
                QObjectPrivate::activate(receiver, c, argv);
                break;

            case Qt::QueuedConnection:
                // 【队列连接】强制异步调用（总是投递到事件队列）
                QCoreApplication::postEvent(
                    receiver,
                    new QMetaCallEvent(c, sender, signal_index, argv)
                );
                break;

            case Qt::BlockingQueuedConnection:
                // 【阻塞队列】跨线程时阻塞等待槽函数执行完成
                // 【注意】只能用于跨线程！同线程会死锁
                QSemaphore semaphore;
                QCoreApplication::postEvent(
                    receiver,
                    new QMetaCallEvent(c, sender, signal_index, argv, &semaphore)
                );
                semaphore.acquire();  // 等待槽函数执行完成
                break;

            case Qt::UniqueConnection:
                // 【唯一连接】不是真正的连接类型，用于 connect 时检查重复
                break;
        }

        c = next;  // 处理下一个连接
    }
}
```

// 【要点总结】
// 1. AutoConnection：同线程同步调用，跨线程异步调用（默认推荐）
// 2. DirectConnection：总是同步，跨线程不安全
// 3. QueuedConnection：总是异步，通过事件队列
// 4. BlockingQueued：跨线程阻塞等待，仅用于特定场景
// 5. 链表遍历：先保存 next 指针，防止连接被删除后访问无效内存

// ========== 槽函数调用（内部实现）==========
namespace QObjectPrivate {
    // 【作用】实际调用槽函数的内部函数
    void activate(QObject *receiver,
                  Connection *c,
                  void **argv)  // 参数数组
    {
        // 方式1：通过方法偏移调用（moc 生成的成员函数槽）
        if (c->method_offset) {
            QMetaObject::metacall(
                receiver,
                QMetaObject::InvokeMetaMethod,  // 调用类型：方法
                c->method_offset - 1,                // 方法索引（从0开始，-1转换）
                argv                                 // 参数
            );
        }
        // 方式2：直接函数指针调用（lambda 或函数对象）
        else if (c->slot) {
            c->slot(argv);  // 函数指针直接调用
        }
    }
}
```

**跨线程信号槽原理**

```cpp
// ========== QMetaCallEvent - 跨线程调用的事件 ==========

class QMetaCallEvent : public QEvent {
public:
    QMetaCallEvent(Connection *c, QObject *sender,
                   int signalId, void **argv,
                   QSemaphore *semaphore = nullptr)
        : QEvent(MetaCall), connection(c), sender(sender),
          signalId(signalId), semaphore(semaphore)
    {
        // 拷贝参数（因为跨线程，原参数可能失效）
        // ... 参数序列化
    }

    // 事件被接收者线程处理时调用
    void placeMetaCall(QObject *receiver) {
        // 调用槽函数
        QObjectPrivate::activate(receiver, connection, args);

        // 如果是 BlockingQueued，释放信号量
        if (semaphore)
            semaphore->release();
    }

private:
    Connection *connection;    // 连接信息：包含接收者、槽函数等
    QObject *sender;           // 发送者对象：用于识别信号来源
    int signalId;              // 信号ID：用于调试
    QSemaphore *semaphore;     // 信号量：用于 BlockingQueued 等待
    // ... 参数存储：参数被拷贝存储，因为跨线程原参数可能失效
};

// 【要点总结】
// 1. QMetaCallEvent 是跨线程信号槽的载体
// 2. 参数被序列化存储，避免跨线程访问失效
// 3. 事件类型为 QEvent::MetaCall，优先级高

// ========== 跨线程调用流程详解 ==========
/*
 * 【图解】跨线程信号槽完整流程：
 *
 * 线程 A (sender 线程)        线程 B (receiver 线程)
 * ┌─────────────────┐         ┌─────────────────┐
 * │  emit signal()  │         │                 │
 * └────────┬────────┘         └─────────────────┘
 *          │
 *          v
 * ┌─────────────────┐
 * │ MOC 信号函数     │
 * │ void signal()   │
 * └────────┬────────┘
 *          │
 *          v
 * ┌─────────────────┐
 * │ QMetaObject::    │
 * │ activate()       │  检测到接收者在其他线程
 * └────────┬────────┘
 *          │
 *          v
 * ┌─────────────────┐          ┌─────────────────┐
 * │ QCoreApplication │          │                 │
 * │ ::postEvent()    │─────────>│ 事件队列         │
 * │ 创建事件并投递   │          │ (线程间安全传输)  │
 * └─────────────────┘          └────────┬────────┘
 *                                        │
 *                                        v
 *                                 ┌─────────────────┐
 *                                 │ 线程 B 的          │
 *                                 │ QEventLoop::exec()│  处理事件
 *                                 └────────┬────────┘
 *                                        │
 *                                        v
 *                                 ┌─────────────────┐
 *                                 │ QApplication::  │  分发事件
 *                                 │ notify()         │
 *                                 └────────┬────────┘
 *                                        │
 *                                        v
 *                                 ┌─────────────────┐
 *                                 │ receiver->       │
 *                                 │ event(MetaCall)  │
 *                                 └────────┬────────┘
 *                                        │
 *                                        v
 *                                 ┌─────────────────┐
 *                                 │ placeMetaCall()  │  调用槽
 *                                 └────────┬────────┘
 *                                        │
 *                                        v
 *                                 ┌─────────────────┐
 *                                 │ qt_metacall()    │  分发到具体槽
 *                                 └────────┬────────┘
 *                                        │
 *                                        v
 *                                 ┌─────────────────┐
 *                                 │ 实际槽函数        │
 *                                 │ updateValue()   │  执行业务逻辑
 *                                 └─────────────────┘
 */
```

**connect 函数底层流程（Qt5 指针语法）**

```cpp
// qobject.cpp - Qt5 connect 实现简化
QMetaObject::Connection QObject::connect(
    const QObject *sender,
    const void **signal_ptr,      // &Sender::valueChanged 的地址
    const QObject *receiver,
    const void **method_ptr,      // &Receiver::updateValue 的地址
    Qt::ConnectionType type)
{
    // ========== 1. 从指针中提取信号索引 ==========
    // signal_ptr 指向成员函数指针，需要解析出信号在元对象表中的索引
    int signal_index = QMetaObject::connectImpl(
        sender, signal_ptr, receiver, method_ptr, type);

    if (signal_index < 0)
        return QMetaObject::Connection();

    // ========== 2. 创建 Connection 结构 ==========
    QObjectPrivate::Connection *c = new QObjectPrivate::Connection;
    c->sender = const_cast<QObject*>(sender);
    c->receiver = const_cast<QObject*>(receiver);
    c->connectionType = type;
    c->method_offset = 0;  // 对于函数指针形式
    c->slot = /* 解析 method_ptr 得到函数指针 */;

    // ========== 3. 添加到连接链表 ==========
    QObjectPrivate::get(sender)->addConnection(signal_index, c);

    return QMetaObject::Connection(c);  // 返回连接句柄
}

// ========== 信号索引查找（通过函数指针）==========
// Qt 使用函数指针地址的唯一性来定位信号
// 这是 Qt5 新语法能实现编译时检查的关键
```

### 2.2 元对象系统

```cpp
// ========== Q_OBJECT 宏的作用 ==========

class MyClass : public QObject {
    Q_OBJECT  // 启用元对象系统

    // Q_OBJECT 宏展开后包含：
    // 1. virtual const QMetaObject* metaObject() const;
    // 2. virtual void* qt_metacast(const char*);
    // 3. virtual int qt_metacall(QMetaObject::Call, int, void**);

    Q_PROPERTY(int value READ value WRITE setValue NOTIFY valueChanged)
    // Q_PROPERTY: 声明属性，可动态访问

    Q_ENUMS(MyEnum)
    // Q_ENUMS: 注册枚举到元对象系统

public:
    enum MyEnum {
        Value1,
        Value2,
        Value3
    };
    Q_ENUM(MyEnum)  // Qt5 风格

    MyClass(QObject* parent = nullptr) : QObject(parent), m_value(0) {}

    int value() const { return m_value; }
    void setValue(int val) {
        if (m_value != val) {
            m_value = val;
            emit valueChanged(val);
        }
    }

signals:
    void valueChanged(int newValue);

public slots:
    void reset() { setValue(0); }

private:
    int m_value;
};

// ========== 反射机制使用 ==========

MyClass* obj = new MyClass();

// 1. 获取元对象
const QMetaObject* metaObj = obj->metaObject();

// 2. 获取类名
qDebug() << "Class name:" << metaObj->className();

// 3. 获取信号/槽数量
qDebug() << "Method count:" << metaObj->methodCount();
qDebug() << "Signal count:" << metaObj->methodCount() - metaObj->methodOffset();

// 4. 遍历所有方法
for (int i = 0; i < metaObj->methodCount(); ++i) {
    QMetaMethod method = metaObj->method(i);
    qDebug() << method.methodType() << method.name() << method.methodSignature();
}

// 5. 动态调用方法（反射）
QByteArray methodName = "reset";
int methodIndex = metaObj->indexOfMethod(methodName);
if (methodIndex != -1) {
    QMetaMethod method = metaObj->method(methodIndex);
    method.invoke(obj, Qt::QueuedConnection);  // 异步调用
    // method.invoke(obj, Qt::DirectConnection);  // 同步调用
}

// 6. 读取/写入属性（动态访问）
int propIndex = metaObj->indexOfProperty("value");
if (propIndex != -1) {
    QMetaProperty prop = metaObj->property(propIndex);
    qDebug() << "Property value:" << prop.read(obj).toInt();

    prop.write(obj, 42);  // 设置属性值
    qDebug() << "New value:" << obj->value();
}

// 7. 枚举值转换
QString enumStr = QMetaEnum::fromType<MyClass::MyEnum>().valueToKey(MyClass::Value2);
qDebug() << "Enum string:" << enumStr;  // "Value2"

// ========== moc 生成文件示例 ==========
// moc 会生成 moc_myclass.cpp 文件，包含：
// - 元对象数据
// - 信号实现
// - qt_metacall 实现
```

---

#### 元对象系统底层实现

**QMetaObject 结构详解**

```cpp
// qmetaobject.h - 元对象核心结构
// 【知识铺垫】QMetaObject 是 Qt 元对象系统的核心，存储类的所有元数据
// 【设计目标】提供反射能力：运行时获取类信息、调用方法、访问属性
// 【面试要点】理解 QMetaObject 对理解信号槽机制至关重要

struct QMetaObject {
    // ========== 元数据表（字符串和索引）==========
    const char *stringdata;      // 字符串数据池：存储类名、方法名、参数名等所有字符串
    const uint *data;            // 索引数据：指向整型数组，每个位置含义不同

    // ========== 类继承链（支持多态）==========
    const QMetaObject *superdata; // 父类元对象：指向父类的 QMetaObject，形成链表
                              // 例如：QWidget::staticMetaObject.superdata → QObject::staticMetaObject

    // ========== 扩展数据（Qt5 新增）==========
    const QMetaObjectPrivate *d;  // 私有数据：存储更多信息，如信号数量

    // ========== 静态元对象（每个类只有一个，单例模式）==========
    static const QMetaObject staticMetaObject;  // MOC 生成的全局唯一实例

    // ========== 核心接口函数 ==========
    int methodOffset() const;     // 信号起始索引：类中第一个信号在 data 数组中的位置
                               // 例如：如果有 2 个属性，methodOffset = 2，信号从索引 2 开始
    int propertyOffset() const;   // 属性起始索引：第一个属性的索引
    int classInfoOffset() const;  // 类信息起始索引：类的额外信息（版本、作者等）

    int methodCount() const;      // 方法总数：信号 + 槽 + 其他方法的总数
    int propertyCount() const;    // 属性数量：使用 Q_PROPERTY 声明的属性个数
    int enumeratorCount() const;  // 枚举数量：使用 Q_ENUM 声明的枚举个数

    // ========== 查找接口（反射能力）==========
    int indexOfMethod(const char *method) const;    // 根据方法签名查找索引
    int indexOfProperty(const char *name) const;    // 查找属性索引
    int indexOfEnumerator(const char *name) const;  // 查找枚举索引

    // ========== 方法调用接口（动态调用）==========
    // 【作用】通过索引动态调用方法
    static bool invokeMethod(QObject *obj, const char *member,
                              Qt::ConnectionType type,
                              QGenericReturnArgument ret,
                              QGenericArgument val0 = QGenericArgument(),
                              QGenericArgument val1 = QGenericArgument(),
                              QGenericArgument val2 = QGenericArgument(),
                              QGenericArgument val3 = QGenericArgument(),
                              QGenericArgument val4 = QGenericArgument(),
                              QGenericArgument val5 = QGenericArgument(),
                              QGenericArgument val6 = QGenericArgument(),
                              QGenericArgument val7 = QGenericArgument(),
                              QGenericArgument val8 = QGenericArgument(),
                              QGenericArgument val9 = QGenericArgument());

    // 【常用接口】
    const char *className() const;           // 返回类名
    const char *superClassName() const;     // 返回父类名
    QMetaMethod method(int index) const;     // 获取指定索引的方法信息
    QMetaProperty property(int index) const; // 获取指定索引的属性信息
};

// ========== 元数据布局（data 数组详细解析）==========
/*
 * data 数组是 4 字节对齐的整数数组，每个位置存储不同的元数据索引
 * 索引指向 stringdata 中的字符串或存储整型常量
 *
 * ================= 头部信息（每个类都有）==================
 * 索引  |  含义              | 示例值      | 说明
 * -----------------------------------------------------------------------
 * [0]   | revision          | 14          | 元对象系统版本号
 * [1]   | className         | 0           | 类名在 stringdata 中的索引
 * [2]   | classInfoCount    | 0           | 类信息条目数量
 * [3]   | classInfoOffset   | 14          | 类信息在 data 数组中的起始索引
 * [4]   | methodCount       | 5           | 方法数量（信号+槽+其他）
 * [5]   | methodOffset      | 14          | 方法在 data 数组中的起始索引
 * [6]   | propertyCount     | 1           | 属性数量
 * [7]   | propertyOffset    | 19          | 属性在 data 数组中的起始索引
 * [8]   | enumeratorCount   | 0           | 枚举数量
 * [9]   | enumeratorOffset  | 20          | 枚举在 data 数组中的起始索引
 * [10]  | constructorCount   | 0           | 构造函数数量
 * [11]  | constructorOffset  | 20          | 构造函数在 data 数组中的起始索引
 * [12]  | flags             | 0x04        | 类标志位
 *                                        0x04 = RequiresConstructor (Qt5)
 *
 * ================= 方法数据（从 methodOffset 开始）==================
 * 每个方法占用 5 个整数：
 * 索引  |  含义              | 示例                      | 说明
 * ------------------------------------------------------------------------------------
 * [N+0] | name              | 4                         | 方法名在 stringdata 中的索引
 * [N+1] | signature         | 5                         | 方法签名在 stringdata 中的索引
 * [N+2] | type              | 6                         | 参数类型在 stringdata 中的索引（逗号分隔）
 * [N+3] | tag               | 7                         | 相关标签在 stringdata 中的索引
 * [N+4] | flags             | 0x08                      | 方法标志位
 *
 * flags 位域含义：
 *   - AccessPrivate (0x01)      - 私有方法
 *   - AccessProtected (0x02)   - 保护方法
 *   - AccessPublic (0x04)      - 公有方法
 *   - MethodMethod (0x08)      - 普通方法
 *   - MethodSignal (0x10)      - 信号
 *   - MethodSlot (0x20)        - 槽
 *   - MethodConstructor (0x40) - 构造函数
 *   - MethodType (0x80)        - 类型
 *
 * ================= 数据存储方式图解 ===================
 * stringdata (字符串池):                    data (索引数组):
 * ┌────────────────────────┐              ┌──────────────────────┐
 * │ 0: "MyClass"            │              │ [0]  14 (revision)   │
 * │ 1: "valueChanged"       │              │ [1]  0 (className)    │ ─┐
 * │ 2: "valueChanged(int)"  │              │ [2]  0 (classInfoCnt) │  │
 * │ 3: "int"                │              │ [3]  14 (classInfoOff) │  │
 * │ 4: "value"              │              │ [4]  5 (methodCount)  │  │
 * │ 5: "setValue"           │              │ [5]  14 (methodOffset) │ ─┼┐
 * │ 6: "setValue(int)"      │              │ [6]  1 (propCount)    │  │ │
 * │ 7: "int"                │              │ [7]  19 (propOffset)   │  │ │
 * │ ...                     │              │ ...                    │  │ │
 * └────────────────────────┘              └──────────────────────┘  │ │
 *                                                  索引指向字符串池 ───────┘ │
 *                                                           │
 *                           方法数据从索引 14 开始  ◄────────┘
 */
```
 *
 * // 方法数据区（从 methodOffset 开始）
 * [N+0] name                 // 方法名在 stringdata 中的索引
 * [N+1] signature            // 方法签名（含参数）在 stringdata 中索引
 * [N+2] type                 // 参数类型（在 stringdata 中索引），逗号分隔
 * [N+3] tag                  // 标签
 * [N+4] flags                // 标志位
 *
 * // flags 位域含义：
 * // 0x01: AccessPrivate     0x02: AccessProtected
 * // 0x04: AccessPublic      0x08: MethodMethod
 * // 0x10: MethodSignal      0x20: MethodSlot
 * // 0x40: MethodConstructor 0x80: MethodType
 */
```

**MOC 生成的完整源码（moc_myclass.cpp）**

```cpp
/****************************************************************************
** Meta object code from reading C++ file 'myclass.h'
**
** Created by: The Qt Meta Object Compiler version 5.15.x
**
** WARNING! All changes made in this file will be lost!
*****************************************************************************/

#include <memory>
#include "myclass.h"
#include <QtCore/qmetatype.h>
#include <QtCore/qstring.h>

// ========== 1. 字符串数据池 ==========
static const uint qt_meta_data_MyClass[] = {
    // content:
    14,       // revision
    0,        // classname
    0,    0,  // classinfo
    3,   14,  // methods (3 个方法)
    1,   29,  // properties (1 个属性)
    0,    0,  // enums/sets
    0,    0,  // constructors
    4,       // flags (RequiresConstructor)

    // 字符串索引
    0x0,     // methods: name (valueChanged)
    0x1,     // methods: signature
    0x13,    // methods: parameter types (int)
    0x17,    // methods: tag
    0x10,    // methods: flags (AccessPublic, MethodSignal)

    0x18,    // methods: name (onDataChanged)
    0x27,    // methods: signature
    0x3A,    // methods: parameter types
    0x17,    // methods: tag
    0x08,    // methods: flags (AccessPublic, MethodSlot)

    0x3C,    // methods: name (reset)
    0x42,    // methods: signature
    0x17,    // methods: tag
    0x28,    // methods: flags (AccessPublic, MethodSlot, MethodCompatibility)

    0x48,    // properties: name
    0x4E,    // properties: type
    0x00,    // properties: flags (Readable, Writable)

    0x00     // eod
};

// ========== 2. 字符串表 ==========
static const char qt_meta_stringdata_MyClass[] = {
    "MyClass\0valueChanged\0valueChanged(int)\0int\0"
    "\0onDataChanged\0onDataChanged(QString)\0QString\0"
    "\0reset\0reset()\0\0value\0int\0"
};

// ========== 3. QMetaObject 实例 ==========
const QMetaObject MyClass::staticMetaObject = {
    { &QObject::staticMetaObject,     // superdata (父类)
      qt_meta_stringdata_MyClass,     // stringdata
      qt_meta_data_MyClass,          // data
      qt_static_metacall,            // 静态元调用函数指针
      nullptr,
      nullptr
    }
};

// ========== 4. 获取元对象（虚函数实现）==========
const QMetaObject *MyClass::metaObject() const
{
    return &staticMetaObject;
}

// ========== 5. 元调用分发器（核心）==========
int MyClass::qt_metacall(QMetaObject::Call _c, int _id, void **_a)
{
    // ========== 先调用父类的元调用 ==========
    _id = QObject::qt_metacall(_c, _id, _a);
    if (_id < 0)
        return _id;  // 父类已处理，直接返回

    // ========== 处理本类的方法 ==========
    // 根据调用类型分发
    switch (_c) {
        case QMetaObject::InvokeMetaMethod:
            // 方法调用
            switch (_id) {
                case 0: // valueChanged(int)
                    valueChanged((*reinterpret_cast<int*>(_a[1])));
                    break;
                case 1: // onDataChanged(QString)
                    onDataChanged((*reinterpret_cast<QString*>(_a[1])));
                    break;
                case 2: // reset()
                    reset();
                    break;
                default: ;
            }
            break;

        case QMetaObject::ReadProperty:
        case QMetaObject::WriteProperty:
            // 属性读写
            switch (_id) {
                case 0: // value 属性
                    if (_c == QMetaObject::ReadProperty)
                        *_a[0] = QVariant::fromValue(this->value());
                    else
                        this->setValue((*reinterpret_cast<int*>(_a[1])));
                    break;
                default: ;
            }
            break;

        case QMetaObject::ResetProperty:
            // 属性重置
            if (_id == 0)
                this->setValue(0);  // 重置为默认值
            break;

        default: ;
    }

    // 返回未处理的偏移量
    _id -= 3;  // 本类有 3 个方法
    return _id;
}

// ========== 6. 类型转换（RTTI 替代）==========
void *MyClass::qt_metacast(const char *_clname)
{
    if (!_clname) return nullptr;
    if (!strcmp(_clname, qt_meta_stringdata_MyClass))
        return static_cast<void*>(this);
    return QObject::qt_metacast(_clname);  // 查父类
}

// ========== 7. 静态元调用（用于信号激活等）==========
void MyClass::qt_static_metacall(QObject *_o, QMetaObject::Call _c,
                                  int _id, void **_a)
{
    if (_c == QMetaObject::InvokeMetaMethod) {
        MyClass *_t = static_cast<MyClass *>(_o);
        switch (_id) {
            case 0: _t->valueChanged((*reinterpret_cast<int*>(_a[1]))); break;
            case 1: _t->onDataChanged((*reinterpret_cast<QString*>(_a[1]))); break;
            case 2: _t->reset(); break;
            default: ;
        }
    } else if (_c == QMetaObject::RegisterMethodArgumentMetaType) {
        // 注册参数类型
        switch (_id) {
            case 0:
                *reinterpret_cast<int*>(_a[0]) = qRegisterMetaType<int>();
                break;
            case 1:
                *reinterpret_cast<int*>(_a[0]) = qRegisterMetaType<QString>();
                break;
            default: ;
        }
    }
}

// ========== 8. 信号实现（由 MOC 生成）==========
// SIGNAL 0: valueChanged(int)
void MyClass::valueChanged(int _t1)
{
    void *_a[] = { nullptr, const_cast<void*>(reinterpret_cast<const void*>(&_t1)) };
    QMetaObject::activate(this, &staticMetaObject, 0, _a);
}

// ========== 元对象私有数据 ==========
const QMetaObjectPrivate MyClass::staticMetaObjectPrivate = {
    // ... 版本、标志等
};
```

**qt_metacall 调用流程详解**

```cpp
// ========== qt_metacall 是反射的核心 ==========

/*
 * 调用链路：
 * QMetaObject::invokeMethod()
 *     ↓
 * QMetaObject::metacall()  // 静态方法
 *     ↓
 * QObject::qt_metacall()  // 虚函数
 *     ↓
 * MyClass::qt_metacall()  // 子类重写
 *     ↓
 * switch (_id) -> 实际方法调用
 */

// ========== invokeMethod 使用示例 ==========
QMetaObject::invokeMethod(obj, "reset");
// 等价于 obj->reset();

// ========== 带参数的调用 ==========
QMetaObject::invokeMethod(obj, "setValue",
                         Q_ARG(int, 42));

// ========== 异步调用（跨线程）==========
QMetaObject::invokeMethod(obj, "reset",
                         Qt::QueuedConnection);

// ========== invokeMethod 内部实现（简化）==========
bool QMetaObject::invokeMethod(
    QObject *obj,
    const char *member,
    Qt::ConnectionType type,
    QGenericReturnArgument ret,
    QGenericArgument val0,
    QGenericArgument val1,
    QGenericArgument val2,
    QGenericArgument val3,
    QGenericArgument val4,
    QGenericArgument val5,
    QGenericArgument val6,
    QGenericArgument val7,
    QGenericArgument val8,
    QGenericArgument val9)
{
    // 1. 查找方法索引
    int idx = obj->metaObject()->indexOfMethod(member);
    if (idx < 0) return false;

    // 2. 准备参数数组
    void *_a[11];
    _a[0] = ret.data();  // 返回值位置
    _a[1] = val0.data();
    _a[2] = val1.data();
    // ... _a[10]

    // 3. 根据连接类型分发
    if (type == Qt::DirectConnection) {
        // 同步调用
        return QMetaObject::metacall(obj, InvokeMetaMethod, idx, _a) >= 0;
    } else {
        // 异步调用（创建 QMetaCallEvent）
        return QMetaObject::invokeMethod(obj, member, type, ret,
                                         val0, val1, val2, val3, val4,
                                         val5, val6, val7, val8, val9);
    }
}
```

**QMetaObjectPrivate 结构（内部元数据）**

```cpp
// qmetaobject_p.h
struct QMetaObjectPrivate {
    int revision;              // 版本号
    int className;             // 类名索引
    int classInfoCount;        // 类信息数量
    int classInfoOffset;       // 类信息偏移
    int methodCount;           // 方法数量
    int methodOffset;          // 方法偏移
    int propertyCount;         // 属性数量
    int propertyOffset;        // 属性偏移
    int enumeratorCount;       // 枚举数量
    int enumeratorOffset;      // 枚举偏移
    int constructorCount;      // 构造函数数量
    int constructorOffset;     // 构造函数偏移
    int flags;                 // 标志位

    // 计算属性
    int signalCount() const {
        // 从 methodOffset 开始，找到第一个非 signal 的方法
        int count = 0;
        for (int i = methodOffset; i < methodOffset + methodCount; ++i) {
            if (data[i * 5 + 4] & 0x10)  // MethodSignal 标志
                ++count;
            else
                break;
        }
        return count;
    }
};
```

### 2.3 事件循环与事件处理

```cpp
// ========== 事件循环原理 ==========

/*
 * 事件循环流程:
 * 1. 系统事件产生（鼠标、键盘、网络等）
 * 2. 事件进入队列
 * 3. QCoreApplication::exec() 启动事件循环
 * 4. 从队列取出事件
 * 5. 分发到对应对象 (sendEvent/postEvent)
 * 6. 对象处理事件
 * 7. 循环继续...
 */

// ========== 事件处理顺序 ==========

class MyWidget : public QWidget {
public:
    // 1. 最底层：event() - 所有事件的入口
    bool event(QEvent* e) override {
        switch (e->type()) {
            case QEvent::KeyPress:
                // 键盘事件
                return keyPressEvent(static_cast<QKeyEvent*>(e));
            case QEvent::MouseButtonPress:
                // 鼠标按下事件
                return mousePressEvent(static_cast<QMouseEvent*>(e));
            case QEvent::Timer:
                // 定时器事件
                return timerEvent(static_cast<QTimerEvent*>(e));
            default:
                return QWidget::event(e);
        }
    }

    // 2. 具体事件处理器
    void keyPressEvent(QKeyEvent* e) override {
        if (e->key() == Qt::Key_Escape) {
            close();  // ESC 关闭窗口
        } else if (e->modifiers() & Qt::ControlModifier && e->key() == Qt::Key_S) {
            saveFile();  // Ctrl+S 保存
        }
        QWidget::keyPressEvent(e);
    }

    void mousePressEvent(QMouseEvent* e) override {
        if (e->button() == Qt::LeftButton) {
            qDebug() << "Left click at:" << e->pos();
            // 记录拖拽起始点
            dragStartPos = e->pos();
        }
        QWidget::mousePressEvent(e);
    }

    void mouseMoveEvent(QMouseEvent* e) override {
        if (e->buttons() & Qt::LeftButton) {
            // 拖拽逻辑
            QPoint delta = e->pos() - dragStartPos;
            move(x() + delta.x(), y() + delta.y());
        }
        QWidget::mouseMoveEvent(e);
    }

private:
    QPoint dragStartPos;
};

// ========== 事件过滤器 ==========

// 事件过滤器可以在一个对象中监控另一个对象的事件

class FilterWidget : public QWidget {
    Q_OBJECT
public:
    FilterWidget(QWidget* parent = nullptr) : QWidget(parent) {
        lineEdit = new QLineEdit(this);

        // 安装事件过滤器
        lineEdit->installEventFilter(this);
    }

    // 重写 eventFilter
    bool eventFilter(QObject* watched, QEvent* event) override {
        if (watched == lineEdit) {
            if (event->type() == QEvent::KeyPress) {
                QKeyEvent* keyEvent = static_cast<QKeyEvent*>(event);

                // 拦截某些按键
                if (keyEvent->key() == Qt::Key_Tab) {
                    qDebug() << "Tab pressed in QLineEdit";
                    return true;  // 事件已处理，不再传递
                }

                // 记录所有按键
                qDebug() << "Key pressed:" << keyEvent->text();
            }
        }

        // 调用父类继续处理
        return QWidget::eventFilter(watched, event);
    }

private:
    QLineEdit* lineEdit;
};

// ========== 自定义事件 ==========

// 1. 定义事件类型
const QEvent::Type MyCustomEvent = static_cast<QEvent::Type>(QEvent::User + 100);

// 2. 自定义事件类
class CustomEvent : public QEvent {
public:
    CustomEvent(const QString& message)
        : QEvent(MyCustomEvent), m_message(message) {}

    QString message() const { return m_message; }

private:
    QString m_message;
};

// 3. 发送自定义事件
class EventSender : public QObject {
    Q_OBJECT
public:
    void sendCustomEvent(QObject* receiver) {
        // 方式1：postEvent - 异步（事件进入队列）
        QCoreApplication::postEvent(receiver, new CustomEvent("Hello"));

        // 方式2：sendEvent - 同步（立即处理）
        // 注意：sendEvent 会自动删除事件
        CustomEvent e("World");
        QCoreApplication::sendEvent(receiver, &e);
    }
};

// 4. 处理自定义事件
class EventReceiver : public QObject {
    Q_OBJECT
protected:
    bool event(QEvent* e) override {
        if (e->type() == MyCustomEvent) {
            CustomEvent* customEvent = static_cast<CustomEvent*>(e);
            qDebug() << "Received:" << customEvent->message();
            return true;  // 事件已处理
        }
        return QObject::event(e);
    }
};

// ========== 常见事件类型 ==========
/*
 * QEvent::MouseButtonPress      - 鼠标按下
 * QEvent::MouseButtonRelease    - 鼠标释放
 * QEvent::MouseMove             - 鼠标移动
 * QEvent::KeyPress              - 键盘按下
 * QEvent::KeyRelease            - 键盘释放
 * QEvent::FocusIn               - 获得焦点
 * QEvent::FocusOut              - 失去焦点
 * QEvent::Enter                 - 鼠标进入
 * QEvent::Leave                 - 鼠标离开
 * QEvent::Paint                 - 绘图事件
 * QEvent::Resize                - 大小改变
 * QEvent::Timer                 - 定时器事件
 * QEvent::Close                 - 关闭事件
 * QEvent::Show                  - 显示事件
 * QEvent::Hide                  - 隐藏事件
 */
```

---

#### 事件循环底层实现

**QEventLoop::exec 核心实现**

```cpp
// qeventloop.cpp - 事件循环核心实现
int QEventLoop::exec(ProcessEventsFlags flags)
{
    // ========== 1. 初始化 ==========
    Q_D(QEventLoop);
    d->threadData->eventLoop = this;  // 记录当前事件循环
    d->exit = false;                   // 退出标志
    d->returnCode = 0;

    // ========== 2. 进入循环 ==========
    while (!d->exit) {
        // ========== 3. 处理事件队列中的事件 ==========
        processEvents(flags |
                      WaitForMoreEvents |  // 没有事件时等待
                      EventLoopExec);      // 标记为 exec 模式

        // ========== 4. 处理投递的事件 ==========
        // sendEvent 发送的事件存储在 postedEvents 链表中
        QCoreApplication::sendPostedEvents();

        // ========== 5. 空闲处理 ==========
        // 如果队列中没有事件，执行空闲任务
        if (d->eventQueue.isEmpty()) {
            QCoreApplication::processEvents(QEventLoop::ProcessEventsFlag::DeferredDeletion);
        }
    }

    // ========== 6. 清理并退出 ==========
    d->threadData->eventLoop = nullptr;
    return d->returnCode;
}
```

**processEvents 实现（处理系统事件）**

```cpp
// qeventloop.cpp
bool QEventLoop::processEvents(ProcessEventsFlags flags)
{
    Q_D(QEventLoop);

    // ========== 1. 处理 Qt 事件队列 ==========
    while (!d->eventQueue.isEmpty()) {
        // 从队列取出事件
        QEvent* e = d->eventQueue.takeFirst();

        // 分发事件
        if (!QCoreApplication::sendEvent(d->eventReceiver, e)) {
            // 事件未被处理，删除事件
            delete e;
        }
    }

    // ========== 2. 处理系统事件（Windows、X11、macOS）==========
    if (flags & WaitForMoreEvents) {
        // 没有事件时，等待系统事件
        // Windows: GetMessage/PeekMessage
        // X11: XNextEvent
        // macOS: NSRunLoop
        return processSystemEvents(flags);
    }

    return true;
}

// ========== Windows 平台实现 ==========
#ifdef Q_OS_WIN
bool QEventLoop::processSystemEvents(ProcessEventsFlags flags)
{
    // MSG 结构体
    MSG msg;

    // PeekMessage 检查消息队列
    if (flags & WaitForMoreEvents) {
        // 等待模式：GetMessage（阻塞）
        if (GetMessage(&msg, NULL, 0, 0)) {
            // 转换虚拟键码
            TranslateMessage(&msg);

            // 分发到窗口过程
            DispatchMessage(&msg);
        }
    } else {
        // 非等待模式：PeekMessage（非阻塞）
        while (PeekMessage(&msg, NULL, 0, 0, PM_REMOVE)) {
            TranslateMessage(&msg);
            DispatchMessage(&msg);
        }
    }

    return true;
}
#endif
```

**QApplication::notify 事件分发链路**

```cpp
// qapplication.cpp - 事件分发入口
bool QApplication::notify(QObject *receiver, QEvent *event)
{
    // ========== 1. 应用级事件过滤 ==========
    if (QApplicationPrivate::eventFilter) {
        if (QApplicationPrivate::eventFilter->eventFilter(receiver, event))
            return true;  // 被过滤器拦截
    }

    // ========== 2. 事件压缩（用于连续事件）==========
    if (event->type() == QEvent::UpdateRequest ||
        event->type() == QEvent::LayoutRequest) {
        // 多个相同事件可以合并为一个
        if (compressEvent(event))
            return true;
    }

    // ========== 3. 特殊事件处理 ==========
    switch (event->type()) {
        case QEvent::ApplicationActivate:
            handleApplicationActivateEvent(event);
            return true;

        case QEvent::FileOpen:
            handleFileOpenEvent(static_cast<QFileOpenEvent*>(event));
            return true;
    }

    // ========== 4. 递归事件检测 ==========
    // 防止事件处理中再次处理相同事件导致无限递归
    if (receiver->d_func()->isDeletingChildren) {
        // 对象正在删除子对象，跳过事件
        return true;
    }

    // ========== 5. 调用对象的 event() 方法 ==========
    bool result = receiver->event(event);

    // ========== 6. 事件后处理 ==========
    if (!result && event->type() == QEvent::DeferredDelete) {
        // 延迟删除事件：未被处理时自动删除对象
        delete receiver;
        result = true;
    }

    return result;
}
```

**sendEvent vs postEvent 区别**

```cpp
// qcoreapplication.cpp

// ========== sendEvent：同步事件 ==========
bool QCoreApplication::sendEvent(QObject *receiver, QEvent *event)
{
    if (!receiver || !event)
        return false;

    // 直接调用 receiver->event()
    // 在同一线程，立即执行
    QEventLoop *eventLoop = QThread::currentEventLoop();

    // 处理递归检测
    if (eventLoop) {
        eventLoop->d_func()->execRecursion++;
    }

    // ========== 同步调用 ==========
    bool result = receiver->event(event);

    // 清理事件
    if (event->spontaneous()) {
        // 自发事件（系统产生）需要手动删除
        delete event;
    } else if (!result && event->type() == QEvent::DeferredDelete) {
        // DeferredDelete 未被处理时删除对象
        delete receiver;
    }

    return result;
}

// ========== postEvent：异步事件 ==========
void QCoreApplication::postEvent(QObject *receiver, QEvent *event, int priority)
{
    if (!receiver || !event)
        return;

    // ========== 添加到事件队列 ==========
    QThreadData *data = QThreadData::get(receiver->thread());

    // 清除旧事件（DeferredDelete 类型）
    if (event->type() == QEvent::DeferredDelete) {
        data->postEventList.removeAll(event);
    }

    // ========== 按优先级插入 ==========
    QPostEventList &list = data->postEventList;
    int i = 0;

    if (priority == Qt::HighEventPriority) {
        // 高优先级：插入到头部
        i = 0;
    } else if (priority == Qt::LowEventPriority) {
        // 低优先级：插入到尾部
        i = list.size();
    } else {
        // 正常优先级：在 DeferredDelete 之前
        for (; i < list.size(); ++i) {
            const QPostEvent &pe = list.at(i);
            if (pe.event->type() == QEvent::DeferredDelete &&
                !pe.event->spontaneous())
                break;
        }
    }

    // 插入事件
    list.insert(i, QPostEvent(receiver, event));

    // ========== 唤醒事件循环 ==========
    // 如果接收者在其他线程，需要唤醒该线程的事件循环
    if (receiver->thread() != QThread::currentThread()) {
        // 跨线程：唤醒接收者线程
        QAbstractEventDispatcher *dispatcher =
            data->eventDispatcher;
        if (dispatcher)
            dispatcher->wakeUp();
    }
}

// ========== sendPostedEvents：处理投递的事件 ==========
void QCoreApplication::sendPostedEvents()
{
    QThreadData *data = QThreadData::current();

    // ========== 处理事件队列 ==========
    QPostEventList &list = data->postEventList;

    while (!list.isEmpty()) {
        // 取出事件
        QPostEvent pe = list.takeFirst();

        // 检查接收者是否还存活
        if (!pe.receiver) {
            delete pe.event;
            continue;
        }

        // 调用 sendEvent 处理
        sendEvent(pe.receiver, pe.event);
    }
}
```

**事件队列数据结构**

```cpp
// qcoreapplication_p.h

// ========== 投递事件结构 ==========
struct QPostEvent {
    QObject *receiver;        // 接收者
    QEvent *event;            // 事件
    int priority;            // 优先级

    QPostEvent(QObject *r, QEvent *e, int p = NormalEventPriority)
        : receiver(r), event(e), priority(p) {}
};

// ========== 投递事件列表 ==========
class QPostEventList : public QList<QPostEvent> {
public:
    // 优化：移除所有指向已删除对象的事件
    void removeAll(QObject *obj) {
        for (int i = size() - 1; i >= 0; --i) {
            if (at(i).receiver == obj) {
                delete at(i).event;
                removeAt(i);
            }
        }
    }
};

// ========== 线程数据 ==========
class QThreadData {
public:
    QPostEventList postEventList;     // 投递事件队列
    QEventLoop *eventLoop;            // 当前事件循环
    QAbstractEventDispatcher *eventDispatcher;  // 事件分发器

    // ... 其他数据
};
```

**事件完整生命周期**

```
系统产生事件（鼠标点击）
    |
    v
Windows/X11/macOS 消息队列
    |
    v
QAbstractEventDispatcher::processEvents()
    |
    v
转换为 QEvent (QMouseEvent)
    |
    v
QCoreApplication::postEvent()  (异步)
    |
    v
QPostEventList (事件队列)
    |
    v
QEventLoop::exec() 检测到新事件
    |
    v
sendPostedEvents() 处理投递事件
    |
    v
QApplication::notify() 分发
    |
    v
事件过滤器链 eventFilter()
    |
    v
QObject::event() 入口
    |
    v
具体事件处理器 (mousePressEvent)
    |
    v
事件处理完成，delete event
```

---

#### Windows 平台事件处理详解

**Windows 消息循环 vs Qt 事件循环**

```cpp
// ========== Windows 原生消息循环 ==========

// 1. Windows 消息结构（MSG）
typedef struct tagMSG {
    HWND   hwnd;      // 消息目标窗口
    UINT   message;   // 消息类型（WM_*）
    WPARAM wParam;    // 附加参数1
    LPARAM lParam;    // 附加参数2
    DWORD  time;      // 消息时间
    POINT  pt;        // 鼠标位置
} MSG;

// 2. Windows 消息循环（WinMain 入口）
int WINAPI WinMain(HINSTANCE hInstance, HINSTANCE hPrevInstance,
                    LPSTR lpCmdLine, int nCmdShow)
{
    HWND hWnd = CreateWindow(...);
    ShowWindow(hWnd, nCmdShow);
    UpdateWindow(hWnd);

    // ========== 消息循环 ==========
    MSG msg;
    while (GetMessage(&msg, NULL, 0, 0)) {  // 从消息队列获取消息
        TranslateMessage(&msg);              // 翻译键盘消息
        DispatchMessage(&msg);               // 分发到窗口过程
    }

    return msg.wParam;
}

// 3. 窗口过程（WndProc）
LRESULT CALLBACK WndProc(HWND hwnd, UINT message, WPARAM wParam, LPARAM lParam)
{
    switch (message) {
        case WM_LBUTTONDOWN:         // 鼠标左键按下
            // 处理鼠标点击
            return 0;
        case WM_KEYDOWN:             // 键盘按下
            // 处理按键
            return 0;
        case WM_PAINT:               // 绘图
            // 处理绘图
            return 0;
        case WM_DESTROY:             // 窗口销毁
            PostQuitMessage(0);
            return 0;
    }
    return DefWindowProc(hwnd, message, wParam, lParam);
}

// ========== Qt 事件循环（对比）==========

int main(int argc, char *argv[])
{
    QApplication app(argc, argv);

    MyWidget window;
    window.show();

    // ========== Qt 事件循环 ==========
    return app.exec();  // 内部调用 QEventLoop::exec()
}

// Qt 内部实现（Windows 平台）
bool QEventDispatcherWindows::processEvents(QEventLoop::ProcessEventsFlags flags)
{
    // 处理 Qt 事件队列
    sendPostedEvents();

    // 处理 Windows 消息
    MSG msg;
    while (PeekMessage(&msg, NULL, 0, 0, PM_REMOVE)) {
        if (msg.message == WM_QUIT) {
            // 收到退出消息
            return false;
        }

        TranslateMessage(&msg);
        // Qt 拦截 DispatchMessage，使用自己的钩子
        DispatchMessage(&msg);
    }

    return true;
}
```

**Qt 在 Windows 平台的架构**

```
┌─────────────────────────────────────────────────────────────┐
│                    Qt 应用层                                │
│                  (QWidget, QObject 等)                       │
├─────────────────────────────────────────────────────────────┤
│                  Qt 事件抽象层                              │
│           (QEvent, QMouseEvent, QKeyEvent 等)                │
├─────────────────────────────────────────────────────────────┤
│              QEventDispatcherWindows                         │
│        (转换 Windows 消息为 Qt 事件)                         │
├─────────────────────────────────────────────────────────────┤
│                    Windows API                              │
│          (GetMessage, DispatchMessage, WndProc)             │
├─────────────────────────────────────────────────────────────┤
│                     操作系统                                  │
│                      (Windows)                               │
└─────────────────────────────────────────────────────────────┘
```

**Qt 如何处理 Windows 消息**

```cpp
// ========== Qt 窗口注册（Windows 平台）==========

// qwindowscontext.cpp
HWND QWindowsContext::createHWND(QWidget *widget, const QWindowsWindowData &data)
{
    // 创建 Windows 窗口
    HWND hwnd = CreateWindowEx(
        data.exStyle,           // 扩展样式
        data.windowClass,       // 窗口类
        data.title,             // 标题
        data.style,             // 窗口样式
        data.geometry.x,        // 位置
        data.geometry.y,
        data.geometry.width,
        data.geometry.height,
        data.parent,            // 父窗口
        data.menu,              // 菜单
        data.instance,          // 实例
        data.createParam        // 创建参数
    );

    return hwnd;
}

// ========== Qt 窗口过程（WndProc 钩子）==========

// qwindowswindow.cpp
LRESULT CALLBACK QWindowsWindowWndProc(HWND hwnd, UINT message,
                                       WPARAM wParam, LPARAM lParam)
{
    // Qt 的全局窗口过程
    QWindowsWindow *platformWindow = ...

    // 转换 Windows 消息为 Qt 事件
    switch (message) {
        case WM_LBUTTONDOWN:
            return platformWindow->handleMouseEvent(QEvent::MouseButtonPress, ...);

        case WM_KEYDOWN:
            return platformWindow->handleKeyEvent(QEvent::KeyPress, ...);

        case WM_PAINT:
            return platformWindow->handlePaintEvent(...);

        // 默认处理
        default:
            return DefWindowProc(hwnd, message, wParam, lParam);
    }
}

// ========== nativeEvent：处理 Windows 消息 ==========

class MyWidget : public QWidget {
protected:
#if QT_VERSION >= QT_VERSION_CHECK(6, 0, 0)
    bool nativeEvent(const QByteArray& eventType, void* message, qintptr* result) override#else
    bool nativeEvent(const QByteArray& eventType, void* message, long* result) override#endif
    {
        if (eventType == "windows_generic_MSG") {
            MSG* msg = static_cast<MSG*>(message);

            switch (msg->message) {
                case WM_POWERBROADCAST:
                    // 系统电源事件
                    if (msg->wParam == PBT_APMSUSPEND) {
                        qDebug() << "System suspending";
                        saveData();
                        *result = TRUE;
                        return true;
                    }
                    break;

                case WM_DEVICECHANGE:
                    // 设备变化事件
                    qDebug() << "Device changed";
                    break;

                case WM_HOTKEY:
                    // 全局热键
                    qDebug() << "Hotkey pressed:" << msg->wParam;
                    *result = 0;
                    return true;

                case WM_COPYDATA:
                    // 进程间通信（WM_COPYDATA）
                    COPYDATASTRUCT* cds = (COPYDATASTRUCT*)msg->lParam;
                    QString data = QString::fromUtf8((char*)cds->lpData);
                    qDebug() << "Received data:" << data;
                    *result = TRUE;
                    return true;
            }
        }

        return QWidget::nativeEvent(eventType, message, result);
    }
};

// ========== 注册全局热键（Windows API）==========

class HotKeyWidget : public QWidget {
public:
    HotKeyWidget(QWidget* parent = nullptr) : QWidget(parent) {
        // 注册 Ctrl+Shift+A 作为全局热键
        int id = 1;
        if (RegisterHotKey((HWND)winId(), id,
                           MOD_CONTROL | MOD_SHIFT, 0x41)) {  // 0x41 = 'A'
            qDebug() << "Hotkey registered";
        }
    }

    ~HotKeyWidget() {
        UnregisterHotKey((HWND)winId(), 1);
    }

protected:
    bool nativeEvent(const QByteArray& eventType, void* message, qintptr* result) override {
        if (eventType == "windows_generic_MSG") {
            MSG* msg = static_cast<MSG*>(message);
            if (msg->message == WM_HOTKEY) {
                qDebug() << "Global hotkey pressed!";
                showWindow();
                *result = 0;
                return true;
            }
        }
        return QWidget::nativeEvent(eventType, message, result);
    }

private:
    void showWindow() {
        showNormal();
        raise();
        activateWindow();
    }
};
```

**Qt 事件与 Windows 消息映射表**

```cpp
// ========== 鼠标事件映射 ==========

/*
 * Windows 消息              Qt 事件                    说明
 * ───────────────────────────────────────────────────────
 * WM_LBUTTONDOWN        → QEvent::MouseButtonPress    左键按下
 * WM_LBUTTONUP          → QEvent::MouseButtonRelease  左键释放
 * WM_LBUTTONDBLCLK      → QEvent::MouseButtonDblClick 左键双击
 * WM_RBUTTONDOWN        → QEvent::MouseButtonPress    右键按下
 * WM_RBUTTONUP          → QEvent::MouseButtonRelease  右键释放
 * WM_RBUTTONDBLCLK      → QEvent::MouseButtonDblClick 右键双击
 * WM_MBUTTONDOWN        → QEvent::MouseButtonPress    中键按下
 * WM_MBUTTONUP          → QEvent::MouseButtonRelease  中键释放
 * WM_MOUSEMOVE          → QEvent::MouseMove           鼠标移动
 * WM_MOUSEWHEEL         → QEvent::Wheel               鼠标滚轮
 * WM_MOUSEHOVER         → QEvent::HoverEnter          鼠标悬停进入
 * WM_MOUSELEAVE         → QEvent::HoverLeave          鼠标悬停离开
 */

// ========== 键盘事件映射 ==========

/*
 * Windows 消息              Qt 事件                    说明
 * ───────────────────────────────────────────────────────
 * WM_KEYDOWN            → QEvent::KeyPress           键盘按下
 * WM_KEYUP              → QEvent::KeyRelease         键盘释放
 * WM_SYSKEYDOWN         → QEvent::KeyPress           系统键按下
 * WM_SYSKEYUP           → QEvent::KeyRelease         系统键释放
 * WM_CHAR               → QEvent::InputMethodEvent   字符输入
 *
 * 虚拟键码映射（部分）：
 * VK_RETURN     → Qt::Key_Return
 * VK_ESCAPE     → Qt::Key_Escape
 * VK_SPACE      → Qt::Key_Space
 * VK_TAB        → Qt::Key_Tab
 * VK_BACK       → Qt::Key_Backspace
 * VK_DELETE     → Qt::Key_Delete
 * VK_HOME       → Qt::Key_Home
 * VK_END        → Qt::Key_End
 * VK_PRIOR      → Qt::Key_PageUp
 * VK_NEXT       → Qt::Key_PageDown
 * VK_LEFT       → Qt::Key_Left
 * VK_RIGHT      → Qt::Key_Right
 * VK_UP         → Qt::Key_Up
 * VK_DOWN       → Qt::Key_Down
 * VK_CONTROL    → Qt::Key_Control
 * VK_SHIFT      → Qt::Key_Shift
 * VK_MENU       → Qt::Key_Alt
 */

// ========== 窗口事件映射 ==========

/*
 * Windows 消息              Qt 事件                    说明
 * ───────────────────────────────────────────────────────
 * WM_CREATE             → QEvent::Create              窗口创建
 * WM_SHOWWINDOW         → QEvent::Show / Hide         显示/隐藏
 * WM_MOVE               → QEvent::Move                移动
 * WM_SIZE               → QEvent::Resize              大小改变
 * WM_PAINT              → QEvent::Paint               绘图
 * WM_ERASEBKGND         → (内部处理)                 擦除背景
 * WM_SETFOCUS           → QEvent::FocusIn             获得焦点
 * WM_KILLFOCUS          → QEvent::FocusOut            失去焦点
 * WM_ACTIVATE           → QEvent::ActivationChange    激活改变
 * WM_ENABLE             → QEvent::EnabledChange       使能状态改变
 * WM_CLOSE              → QEvent::Close               关闭
 * WM_DESTROY            → QEvent::Destroy             销毁
 * WM_QUERYENDSESSION    → QEvent::Close               系统关机
 */

// ========== 定时器事件映射 ==========

/*
 * Windows 消息              Qt 事件                    说明
 * ───────────────────────────────────────────────────────
 * WM_TIMER              → QEvent::Timer               定时器
 *                         (QTimerEvent)
 *
 * Qt 在 Windows 上使用 SetTimer/KillTimer 或
 * 时间队列池（Timer Queue）实现定时器
 */

// ========== 系统事件映射 ==========

/*
 * Windows 消息              Qt 事件                    说明
 * ───────────────────────────────────────────────────────
 * WM_COMMAND            → QAction 触发               菜单/工具栏
 * WM_NOTIFY             → (内部处理)                 控件通知
 * WM_HSCROLL            → QEvent::Wheel               水平滚动
 * WM_VSCROLL            → QEvent::Wheel               垂直滚动
 * WM_CTLCOLOR*          → (内部处理)                 控件颜色
 * WM_DRAWITEM           → (内部处理)                 自绘控件
 * WM_CONTEXTMENU        → QEvent::ContextMenu        右键菜单
 * WM_DRAG*              → QEvent::*                   拖放事件
 */

// ========== Qt 处理 Windows 消息示例 ==========

// qwindowsmousehandler.cpp
bool QWindowsMouseHandler::translateMouseEvent(QEvent::Type type, HWND hwnd,
                                                Qt::MouseButton button,
                                                const MSG &msg)
{
    // 提取坐标
    QPoint globalPos(GET_X_LPARAM(msg.lParam), GET_Y_LPARAM(msg.lParam));
    QPoint pos = globalPos - widget->mapToGlobal(QPoint(0, 0));

    // 提取按键状态
    Qt::MouseButtons buttons = 0;
    if (msg.wParam & MK_LBUTTON) buttons |= Qt::LeftButton;
    if (msg.wParam & MK_RBUTTON) buttons |= Qt::RightButton;
    if (msg.wParam & MK_CONTROL) modifiers |= Qt::ControlModifier;
    if (msg.wParam & MK_SHIFT)   modifiers |= Qt::ShiftModifier;

    // 创建 Qt 鼠标事件
    QMouseEvent *qe = new QMouseEvent(type, pos, globalPos,
                                       button, buttons, modifiers);

    // 发送到 Qt 事件队列
    QCoreApplication::postEvent(widget, qe);

    return true;
}
```

**Windows 特定事件处理示例**

```cpp
// ========== 监听系统关机事件 ==========

class ShutdownHandler : public QObject {
    Q_OBJECT
public:
    ShutdownHandler(QObject* parent = nullptr) : QObject(parent) {}

protected:
    bool eventFilter(QObject* obj, QEvent* event) override {
        // Qt 提供的跨平台方式
        if (event->type() == QEvent::Close) {
            QCloseEvent* closeEvent = static_cast<QCloseEvent*>(event);
            // 检查是否是系统关机
            if (isSystemShutdown()) {
                qDebug() << "System is shutting down";
                // 保存数据
                saveCriticalData();
            }
        }
        return QObject::eventFilter(obj, event);
    }

private:
    bool isSystemShutdown() {
        // 检查会话是否正在关闭
        return GetSessionMetrics(0) == 0;
    }

    void saveCriticalData() {
        // 保存关键数据
    }
};

// ========== Windows 平台特定实现 ==========

#ifdef Q_OS_WIN
class WindowsPlatformWidget : public QWidget {
public:
    // ========== 设置窗口透明度（Windows API）==========
    void setWindowOpacity(qreal opacity) {
        // Windows 2000+
        HWND hwnd = (HWND)winId();
        LONG_PTR style = GetWindowLongPtr(hwnd, GWL_EXSTYLE);
        SetWindowLongPtr(hwnd, GWL_EXSTYLE, style | WS_EX_LAYERED);
        SetLayeredWindowAttributes(hwnd, 0, int(opacity * 255), LWA_ALPHA);
    }

    // ========== 设置窗口层级（TopMost）==========
    void setAlwaysOnTop(bool on) {
        HWND hwnd = (HWND)winId();
        SetWindowPos(hwnd, on ? HWND_TOPMOST : HWND_NOTOPMOST,
                    0, 0, 0, 0, SWP_NOMOVE | SWP_NOSIZE);
    }

    // ========== 闪动窗口（任务栏）==========
    void flashWindow() {
        HWND hwnd = (HWND)winId();
        FLASHWINFO fi;
        fi.cbSize = sizeof(FLASHWINFO);
        fi.hwnd = hwnd;
        fi.dwFlags = FLASHW_ALL | FLASHW_TIMERNOFG;
        fi.uCount = 5;
        fi.dwTimeout = 0;
        FlashWindowEx(&fi);
    }

    // ========== 获取窗口矩形 ==========

    QRect getWindowRect() const {
        HWND hwnd = (HWND)winId();
        RECT rect;
        GetWindowRect(hwnd, &rect);
        return QRect(rect.left, rect.top,
                     rect.right - rect.left,
                     rect.bottom - rect.top);
    }
};
#endif
```

---

## 三、内存管理

### 3.1 对象树机制

```cpp
// ========== Qt 父子对象机制 ==========

/*
 * Qt 使用对象树进行内存管理：
 * 1. 每个 QObject 可以有父对象
 * 2. 父对象销毁时，自动销毁所有子对象
 * 3. 子对象在父对象的 children() 列表中
 */

// ========== 会被自动删除的情况 ==========
/*
 * 1. 通过构造函数指定父对象：new QObject(parent)
 * 2. 通过 setParent() 设置父对象：obj->setParent(parent)
 * 3. 添加到 QWidget 的布局中：layout->addWidget(widget)
 * 4. QObject 派生类对象在对象树中时，父对象析构会自动删除子对象
 */

void exampleAutoDelete() {
    QWidget* window = new QWidget();

    // 情况1：构造时指定父对象
    QPushButton* button1 = new QPushButton("Button 1", window);
    // button1 会被 window 自动删除

    // 情况2：后设置父对象
    QPushButton* button2 = new QPushButton();
    button2->setParent(window);
    // button2 会被 window 自动删除

    // 情况3：通过布局添加（widget 会被重父对象设置为 window）
    QVBoxLayout* layout = new QVBoxLayout(window);
    QLabel* label = new QLabel();
    layout->addWidget(label);  // label 的父对象自动设为 window
    // label 会被 window 自动删除

    delete window;  // button1, button2, label, layout 全部自动删除
}

// ========== 不会被自动删除的情况 ==========
/*
 * 1. 栈上创建的对象（非指针）
 * 2. 创建时未指定父对象，且后续未设置父对象
 * 3. 使用 deleteLater() 但事件循环未运行时对象未被实际删除
 * 4. 对象被从对象树中移除（setParent(nullptr) 或移除自身）
 * 5. 非 QObject 派生类的对象
 */

void exampleNoAutoDelete() {
    QWidget* window = new QWidget();

    // 情况1：栈对象，超出作用域自动销毁（与对象树无关）
    QPushButton stackButton;  // 栈上对象
    // stackButton 超出作用域时自动析构，与 window 无关

    // 情况2：未设置父对象，需要手动管理
    QPushButton* orphan = new QPushButton();  // 无父对象
    // orphan 不会被 window 删除，需要手动 delete orphan;

    // 情况3：设置父对象为 nullptr，从对象树中移除
    QPushButton* child = new QPushButton(window);
    child->setParent(nullptr);  // 从对象树移除
    // child 不再被 window 管理，需要手动 delete child;

    delete window;
    delete orphan;    // 必须手动删除
    delete child;     // 必须手动删除
}

// ========== 特殊情况详解 ==========

// 1. 重复删除问题（Double Delete）
class DoubleDeleteExample : public QWidget {
public:
    DoubleDeleteExample() {
        // 错误做法：手动删除有父对象的子对象
        QPushButton* btn = new QPushButton(this);
        delete btn;  // ❌ 危险！父对象析构时会再次尝试删除，导致崩溃
    }
};

// 2. 父子类型不匹配
void typeMismatchExample() {
    // 父对象必须是 QObject 派生类
    QObject* parent = new QObject();
    QWidget* child = new QWidget(parent);  // ✅ 可以，QWidget 继承自 QObject

    // 子对象析构顺序：先子后父
    // 析构时，会先调用所有子对象的析构函数，再析构父对象
}

// 3. 对象所有权转移
void ownershipTransfer() {
    QPushButton* button = new QPushButton();
    QWidget* window1 = new QWidget();
    QWidget* window2 = new QWidget();

    button->setParent(window1);  // window1 拥有 button
    button->setParent(window2);  // 所有权转移到 window2，window1 不再拥有

    delete window1;  // button 不会被删除
    delete window2;  // button 会被删除
}

// 4. deleteLater() 的特殊行为
void deleteLaterExample() {
    QPushButton* button = new QPushButton();
    button->deleteLater();  // 不会立即删除，而是发送 DeferredDelete 事件
    // 在下一个事件循环中真正删除

    // 在事件循环运行前
    qDebug() << button->isNull();  // false，对象仍然存在

    // 事件循环运行后，对象才会真正被删除
    // 这使得在事件处理中安全地删除发送者
    // 例如：on_buttonClicked() { sender()->deleteLater(); }
}

// ========== 对象树遍历 ==========
void traverseObjectTree() {
    QWidget* window = new QWidget();
    new QPushButton("Button1", window);
    new QPushButton("Button2", window);
    new QLabel("Label", window);

    // 获取直接子对象数量
    qDebug() << "Children count:" << window->children().count();

    // 遍历所有子对象
    foreach (QObject* child, window->children()) {
        qDebug() << "Child:" << child->metaObject()->className();
    }

    // 查找特定类型的子对象
    QPushButton* btn = window->findChild<QPushButton*>();
    QList<QPushButton*> buttons = window->findChildren<QPushButton*>();

    // 递归查找（包括子对象的子对象）
    QList<QWidget*> allWidgets = window->findChildren<QWidget*>(Qt::FindDirectChildrenOnly);
    QList<QWidget*> recursiveWidgets = window->findChildren<QWidget*>();
}

// ========== 内存管理最佳实践 ==========

class BestPractices : public QWidget {
    Q_OBJECT
public:
    BestPractices(QWidget* parent = nullptr) : QWidget(parent) {
        // ✅ 方式1：构造时指定父对象（推荐用于UI组件）
        m_button = new QPushButton("Click", this);

        // ✅ 方式2：使用成员变量初始化
        m_label = new QLabel("Label", this);

        // ✅ 方式3：临时对象使用 QScopedPointer（无需父对象）
        QScopedPointer<QTimer> timer(new QTimer());
        timer->start(1000);
        // timer 超出作用域自动删除

        // ✅ 方式4：跨作用域使用 QPointer（守护指针）
        m_weakButton = QPointer<QPushButton>(new QPushButton());
    }

    ~BestPractices() {
        // 不需要手动删除 m_button 和 m_label
        // 父对象 this 析构时会自动删除它们
    }

    // ✅ 对于需要在父对象析构前移除的情况
    void removeChildManually() {
        if (m_label) {
            m_label->setParent(nullptr);  // 从对象树移除
            delete m_label;               // 手动删除
            m_label = nullptr;
        }
    }

private:
    QPushButton* m_button;      // 由父对象管理
    QLabel* m_label;            // 由父对象管理
    QPointer<QPushButton> m_weakButton;  // 守护指针
};

// ========== 禁止复制QObject ==========
/*
 * QObject 及其派生类禁止复制（拷贝构造和赋值运算符被删除）
 * 这是为了避免对象树混乱和重复删除
 */
void noCopyExample() {
    QWidget widget1;
    QWidget widget2;

    // 以下操作编译错误：
    // QWidget widget3 = widget1;  // ❌ 拷贝构造被删除
    // widget2 = widget1;           // ❌ 赋值运算符被删除

    // 正确做法：使用指针
    QWidget* widget3 = &widget1;  // ✅ 指针复制没问题
    widget3 = &widget2;           // ✅ 指针重指向没问题
}
```

**对象树删除规则总结表：**

| 情况 | 是否自动删除 | 说明 |
|------|-------------|------|
| `new QObject(parent)` | ✅ 是 | 构造时指定父对象 |
| `obj->setParent(parent)` | ✅ 是 | 设置父对象后 |
| `layout->addWidget(widget)` | ✅ 是 | widget 自动重父对象为 layout 的父窗口 |
| 栈对象 `QWidget w;` | ❌ 否 | 栈对象，作用域结束时自动析构 |
| `new QObject()` 无父 | ❌ 否 | 孤儿对象，需手动 delete |
| `obj->setParent(nullptr)` | ❌ 否 | 从对象树移除，需手动管理 |
| `deleteLater()` | ⏳ 延迟 | 事件循环中延迟删除 |
| 非 QObject 派生类 | ❌ 否 | 不在对象树管理范围内 |

### 3.2 智能指针

```cpp
// ========== Qt 智能指针 ==========

#include <QPointer>
#include <QSharedPointer>
#include <QWeakPointer>
#include <QScopedPointer>

// 1. QPointer - 监控 QObject 指针，对象被删除时自动置空
void exampleQPointer() {
    QPointer<QPushButton> ptr = new QPushButton();
    ptr->setText("Hello");

    if (ptr) {  // 检查对象是否仍然存在
        ptr->show();
    }

    delete ptr;  // 或对象被其他地方删除
    // ptr 现在自动为 nullptr

    if (ptr) {
        qDebug() << "Still alive";  // 不会执行
    }
}

// 2. QSharedPointer - 引用计数智能指针
void exampleQSharedPointer() {
    QSharedPointer<QString> ptr1 = QSharedPointer<QString>::create("Hello");
    qDebug() << "Ref count:" << ptr1.use_count();  // 1

    {
        QSharedPointer<QString> ptr2 = ptr1;
        qDebug() << "Ref count:" << ptr1.use_count();  // 2
    }
    // ptr2 离开作用域
    qDebug() << "Ref count:" << ptr1.use_count();  // 1
}

// 3. QWeakPointer - 弱引用，不影响引用计数
void exampleQWeakPointer() {
    QSharedPointer<QString> strong = QSharedPointer<QString>::create("Data");
    QWeakPointer<QString> weak = strong;

    if (!weak.isNull()) {
        QSharedPointer<QString> locked = weak.toStrongRef();
        if (locked) {
            qDebug() << *locked;  // 安全访问
        }
    }

    strong.clear();  // 释放对象
    // weak 现在为空
}

// 4. QScopedPointer - 作用域指针（类似 std::unique_ptr）
void exampleQScopedPointer() {
    QScopedPointer<QPushButton> ptr(new QPushButton());
    ptr->setText("Scoped Button");

    // 作用域结束时自动删除
    // 不能复制，只能移动
}

// ========== 标准库智能指针在 Qt 中使用 ==========
#include <memory>

void exampleStdPointers() {
    // std::unique_ptr - 独占所有权
    std::unique_ptr<QWidget> widget = std::make_unique<QWidget>();
    // std::unique_ptr<QWidget> widget2 = widget;  // 错误：不能复制
    std::unique_ptr<QWidget> widget2 = std::move(widget);  // OK：移动

    // std::shared_ptr - 共享所有权
    std::shared_ptr<QPushButton> btn = std::make_shared<QPushButton>();
    std::shared_ptr<QPushButton> btn2 = btn;  // OK：共享

    // std::weak_ptr - 弱引用
    std::weak_ptr<QPushButton> weak = btn;

    // 注意：不要混用 new 和智能指针
    // 错误示例：
    // std::shared_ptr<QWidget> w(new QWidget(), [](QWidget* p){ p->deleteLater(); });
    // Qt 对象如果有父对象，不要用智能指针管理
}
```

### 3.3 内存泄漏检测

```cpp
// ========== 内存泄漏检测技巧 ==========

// 1. 使用 Qt 的调试函数
void debugObjectTree() {
    QWidget* widget = new QWidget();

    // 输出对象树信息
    widget->dumpObjectInfo();
    widget->dumpObjectTree();

    // 检查对象是否仍存活（仅调试模式）
    #ifdef QT_DEBUG
        if (!widget->parent()) {
            qDebug() << "Warning: widget without parent!";
        }
    #endif
}

// 2. 重写 deletableLater 检测
class TrackedObject : public QObject {
    Q_OBJECT
public:
    TrackedObject(QObject* parent = nullptr) : QObject(parent) {
        qDebug() << "TrackedObject created";
    }

    ~TrackedObject() override {
        qDebug() << "TrackedObject destroyed";
    }
};

// 3. 使用 Qt 的对象追踪
class MemoryLeakDetector {
public:
    static void track(QObject* obj) {
        connect(obj, &QObject::destroyed, [](QObject*) {
            qDebug() << "Object destroyed safely";
        });
        m_objects.append(obj);
    }

    static void report() {
        qDebug() << "Tracked objects:" << m_objects.size();
        for (QObject* obj : m_objects) {
            if (obj) {
                qDebug() << " - Still alive:" << obj->metaObject()->className();
            }
        }
    }

private:
    static QList<QPointer<QObject>> m_objects;
};
```

---

## 四、常用控件详解

### 4.1 QListWidget / QListView（列表控件）

// 【使用场景】
// - QListWidget: 少量数据（< 1000 项），快速开发，项可以包含图标和复选框
// - QListView + Model: 大量数据（> 1000 项），高性能，需要自定义显示
// 【面试要点】何时用 Widget vs Model/View？

```cpp
// ========== QListWidget（便捷方式，小数据量）==========
// 【特点】所有功能封装好的类，使用简单
// 【优点】快速开发，内置编辑、拖拽、选择功能
// 【缺点】数据量大时性能差，不够灵活

QListWidget* listWidget = new QListWidget(this);

// 添加项目
listWidget->addItem("Item 1");  // 添加纯文本项
listWidget->addItem("Item 2");
listWidget->addItem("Item 3");

// 使用自定义项目
QListWidgetItem* item = new QListWidgetItem();  // 创建列表项
item->setText("Custom Item");                        // 设置文本
item->setIcon(QIcon(":/icons/item.png"));           // 设置图标
item->setToolTip("This is a tooltip");           // 设置提示
item->setFlags(item->flags() | Qt::ItemIsEditable);  // 设置可编辑
listWidget->addItem(item);

// ========== 常用设置 ==========
// 设置选择模式
listWidget->setSelectionMode(QAbstractItemView::SingleSelection);     // 单选
// listWidget->setSelectionMode(QAbstractItemView::MultiSelection);        // 多选
// listWidget->setSelectionMode(QAbstractItemView::ExtendedSelection);  // 扩展选择（Shift+点击）

// 设置选择行为
listWidget->setSelectionBehavior(QAbstractItemView::SelectItems);   // 选中项
listWidget->setSelectionBehavior(QAbstractItemView::SelectRows);    // 选中行

// 设置编辑触发器
listWidget->setEditTriggers(QAbstractItemView::DoubleClicked |     // 双击编辑
                              QAbstractItemView::EditKeyPressed);  // 按F2编辑

// 拖拽支持
listWidget->setDragEnabled(true);             // 允许拖动项
listWidget->setAcceptDrops(true);            // 允许放置项
listWidget->setDropIndicatorShown(true);     // 显示拖动指示器

// ========== 信号与槽 ==========
connect(listWidget, &QListWidget::itemClicked, [](QListWidgetItem* item) {
    qDebug() << "Clicked:" << item->text();
});

connect(listWidget, &QListWidget::itemDoubleClicked, [](QListWidgetItem* item) {
    qDebug() << "Double clicked:" << item->text();
    item->setFlags(item->flags() | Qt::ItemIsEditable);  // 双击后可编辑
});

connect(listWidget, &QListWidget::currentRowChanged, [](int currentRow) {
    qDebug() << "Current row:" << currentRow;
});

connect(listWidget, &QListWidget::itemChanged, [](QListWidgetItem* item, int column) {
    qDebug() << "Changed:" << item->text(column);
});

// ========== 数据操作 ==========
// 遍历所有项目
for (int i = 0; i < listWidget->count(); ++i) {
    QListWidgetItem* item = listWidget->item(i);
    qDebug() << item->text();
}

// 获取选中项目
QList<QListWidgetItem*> selected = listWidget->selectedItems();
for (QListWidgetItem* item : selected) {
    qDebug() << "Selected:" << item->text();
}

// 删除选中项目
qDeleteAll(listWidget->selectedItems());

// 清空列表
listWidget->clear();

// 当前项操作
QListWidgetItem* current = listWidget->currentItem();
if (current) {
    qDebug() << "Current:" << current->text();
    // 设置文本、图标等
    current->setText("Modified");
    current->setIcon(QIcon(":/icons/modified.png"));
}

// 在单元格中插入控件
QComboBox* combo = new QComboBox();
combo->addItem("Option 1");
combo->addItem("Option 2");
listWidget->setItemWidget(row, 1, combo);  // 在(row, column)位置插入
```

```cpp
// ========== QListView + Model（MVC模式，大数据量）==========
// 【特点】数据与界面分离，适合大量数据和自定义显示
// 【优点】性能好，灵活，多个视图共享同一数据
// 【缺点】代码量多，需要理解 Model/View 架构

QListView* listView = new QListView(this);
QStringListModel* model = new QStringListModel(this);

QStringList data;
data << "Apple" << "Banana" << "Cherry";
model->setStringList(data);

listView->setModel(model);

// 动态添加数据
int row = model->rowCount();
model->insertRows(row, 1);
QModelIndex index = model->index(row);
model->setData(index, "Elderberry");

// 删除数据
model->removeRows(0, 1);  // 删除第一行


// 【要点总结】
// 1. QListWidget 适合简单场景，快速开发
// 2. QListView + Model 适合复杂数据和大数据量
// 3. 使用 setItemWidget 可以在列表项中嵌入控件
// 4. currentRowChanged 信号比 itemClicked 更适合处理当前行变化
listWidget->addItem("Item 2");
listWidget->addItem("Item 3");

// 使用自定义项目
QListWidgetItem* item = new QListWidgetItem();
item->setText("Custom Item");
item->setIcon(QIcon(":/icons/item.png"));
item->setToolTip("This is a tooltip");
listWidget->addItem(item);

// 设置选择模式
listWidget->setSelectionMode(QAbstractItemView::SingleSelection);      // 单选
// listWidget->setSelectionMode(QAbstractItemView::MultiSelection);     // 多选
// listWidget->setSelectionMode(QAbstractItemView::ExtendedSelection);  // 扩展选择
// listWidget->setSelectionMode(QAbstractItemView::ContiguousSelection);// 连续选择

listWidget->setSelectionBehavior(QAbstractItemView::SelectRows);

// 设置编辑触发器
listWidget->setEditTriggers(QAbstractItemView::DoubleClicked | QAbstractItemView::EditKeyPressed);

// 拖拽支持
listWidget->setDragEnabled(true);
listWidget->setAcceptDrops(true);
listWidget->setDropIndicatorShown(true);

// 信号与槽
connect(listWidget, &QListWidget::itemClicked, [](QListWidgetItem* item) {
    qDebug() << "Clicked:" << item->text();
});

connect(listWidget, &QListWidget::itemDoubleClicked, [](QListWidgetItem* item) {
    qDebug() << "Double clicked:" << item->text();
    item->setFlags(item->flags() | Qt::ItemIsEditable);  // 使可编辑
});

connect(listWidget, &QListWidget::currentRowChanged, [](int currentRow) {
    qDebug() << "Current row:" << currentRow;
});

connect(listWidget, &QListWidget::itemChanged, [](QListWidgetItem* item) {
    qDebug() << "Item changed:" << item->text();
});

// 遍历所有项目
for (int i = 0; i < listWidget->count(); ++i) {
    QListWidgetItem* item = listWidget->item(i);
    qDebug() << item->text();
}

// 获取选中项目
QList<QListWidgetItem*> selected = listWidget->selectedItems();
for (QListWidgetItem* item : selected) {
    qDebug() << "Selected:" << item->text();
}

// 删除选中项目
qDeleteAll(listWidget->selectedItems());

// 清空列表
listWidget->clear();

// ========== QListView + Model（MVC模式，大数据量）==========

// 1. 使用 QStringListModel
QListView* listView = new QListView(this);
QStringListModel* model = new QStringListModel(this);

QStringList data;
data << "Apple" << "Banana" << "Cherry" << "Durian";
model->setStringList(data);

listView->setModel(model);

// 动态添加数据
int row = model->rowCount();
model->insertRows(row, 1);
QModelIndex index = model->index(row);
model->setData(index, "Elderberry");

// 删除数据
model->removeRows(0, 1);  // 删除第一行

// 2. 使用 QStandardItemModel
QListView* listView2 = new QListView(this);
QStandardItemModel* standardModel = new QStandardItemModel(this);

for (int i = 0; i < 10; ++i) {
    QStandardItem* item = new QStandardItem(QString("Item %1").arg(i));
    item->setIcon(QIcon(":/icons/item.png"));
    item->setEditable(true);
    standardModel->appendRow(item);
}

listView2->setModel(standardModel);

// 3. 自定义 Model（推荐用于复杂数据）
// 【为什么要自定义 Model】
// - QListWidget 的数据存储和显示耦合，不便于管理复杂数据
// - 自定义 Model 实现 Model/View 分离，同一数据可用于多个视图
// - 更好的性能，大数据量时只加载可见项
// - 支持自定义排序、过滤、修改操作

class CustomListModel : public QAbstractListModel {
    Q_OBJECT
public:
    // 【数据结构】定义 Model 管理的数据类型
    // 可以根据实际需求设计，这里以联系人信息为例
    struct Contact {
        QString name;    // 姓名
        QString phone;   // 电话
        QString email;   // 邮箱
    };

    // 【构造函数】
    // parent: 父对象，用于内存管理（对象树）
    explicit CustomListModel(QObject* parent = nullptr)
        : QAbstractListModel(parent) {
        // 初始化数据（实际应用中可能从数据库或文件加载）
        m_contacts.append({"张三", "13800138000", "zhangsan@qq.com"});
        m_contacts.append({"李四", "13900139000", "lisi@qq.com"});
        m_contacts.append({"王五", "13700137000", "wangwu@qq.com"});
    }

    // 【自定义角色】enum Roles
    // Qt::UserRole 是 Qt 预定义的用户角色起始值
    // 自定义角色应该从 Qt::UserRole + 1 开始，避免冲突
    enum Roles {
        NameRole = Qt::UserRole + 1,  // 姓名角色
        PhoneRole,                    // 电话角色（自动递增为 Qt::UserRole + 2）
        EmailRole                     // 邮箱角色（自动递增为 Qt::UserRole + 3）
    };

    // 【rowCount()】必须重写，返回行数
    // parent: 父索引（List Model 通常忽略）
    // 返回值：列表中的数据项数量
    int rowCount(const QModelIndex& parent = QModelIndex()) const override {
        Q_UNUSED(parent)  // 宏：声明参数未使用，避免编译警告
        return m_contacts.size();
    }

    // 【data()】必须重写，返回指定索引和角色的数据
    // index: 数据索引（包含行、列信息）
    // role: 数据角色（DisplayRole, EditRole, 或自定义角色）
    // 返回值：对应的数据，无效则返回 QVariant()
    QVariant data(const QModelIndex& index, int role = Qt::DisplayRole) const override {
        // 【有效性检查】确保索引有效且在范围内
        if (!index.isValid() || index.row() >= m_contacts.size())
            return QVariant();

        // 获取对应行的数据
        const Contact& contact = m_contacts.at(index.row());

        // 【角色分发】根据角色返回不同的数据
        switch (role) {
            case Qt::DisplayRole:      // 默认显示角色（用于普通显示）
                return contact.name;
            case NameRole:             // 自定义角色：姓名
                return contact.name;
            case PhoneRole:            // 自定义角色：电话
                return contact.phone;
            case EmailRole:            // 自定义角色：邮箱
                return contact.email;
            default:
                return QVariant();     // 未知角色返回空值
        }
    }

    // 【添加数据】addContact()
    // 注意：修改 Model 数据时必须通知视图更新
    void addContact(const Contact& contact) {
        // beginInsertRows(): 通知视图将要插入行
        // 参数：父索引，起始行，结束行
        beginInsertRows(QModelIndex(), m_contacts.size(), m_contacts.size());

        // 执行插入操作
        m_contacts.append(contact);

        // endInsertRows(): 通知视图插入完成，触发视图刷新
        endInsertRows();
    }

    // 【删除数据】removeContact()
    void removeContact(int row) {
        // 边界检查
        if (row < 0 || row >= m_contacts.size())
            return;

        // beginRemoveRows(): 通知视图将要删除行
        beginRemoveRows(QModelIndex(), row, row);

        // 执行删除操作
        m_contacts.removeAt(row);

        // endRemoveRows(): 通知视图删除完成
        endRemoveRows();
    }

private:
    // 【数据存储】Model 内部数据容器
    QList<Contact> m_contacts;  // 使用 QList 存储联系人列表
};

// 使用自定义 Model
CustomListModel* customModel = new CustomListModel(this);
QListView* listView3 = new QListView(this);
listView3->setModel(customModel);

// 自定义委托
// 【什么是委托（Delegate）】
// - 委托负责为 View 绘制和编辑数据项
// - QStyledItemDelegate 是 Qt 提供的样式感知委托基类
// - 自定义委托可以实现复杂的显示效果（如多行文本、图标+文字等）
class ContactDelegate : public QStyledItemDelegate {
    Q_OBJECT
public:
    // 【paint()】绘制函数
    // 负责在视图中绘制数据项的外观
    void paint(QPainter* painter, const QStyleOptionViewItem& option,
               const QModelIndex& index) const override {
        // 【绘制步骤】

        // 1. 获取绘制区域
        QRect rect = option.rect;  // 当前项的矩形区域

        // 2. 绘制背景（选中状态）
        // option.state: 项的状态标志（选中、悬停、焦点等）
        if (option.state & QStyle::State_Selected) {
            painter->fillRect(rect, option.palette.highlight());  // 高亮色填充
        }

        // 3. 获取数据
        // index.data(role): 根据 DataRole 获取数据
        QString name = index.data(CustomListModel::NameRole).toString();
        QString phone = index.data(CustomListModel::PhoneRole).toString();

        // 4. 绘制文字
        painter->setPen(option.palette.text().color());
        painter->drawText(rect.adjusted(5, 5, -5, -5), Qt::AlignLeft | Qt::AlignTop, name);
        painter->drawText(rect.adjusted(5, 25, -5, -5), Qt::AlignLeft | Qt::AlignTop, phone);
    }

    QSize sizeHint(const QStyleOptionViewItem& option,
                   const QModelIndex& index) const override {
        Q_UNUSED(option)
        Q_UNUSED(index)
        return QSize(200, 50);  // 每项高度 50
    }
};

// 【使用自定义委托】
// setItemDelegate(): 为视图设置委托
listView3->setItemDelegate(new ContactDelegate(this));

// 【要点总结】
// 1. 自定义 Model 需要继承 QAbstractListModel/QAbstractTableModel
// 2. 必须重写 rowCount() 和 data() 函数
// 3. 修改数据时必须调用 begin*Rows()/end*Rows() 通知视图
// 4. 自定义角色从 Qt::UserRole + 1 开始定义
// 5. 委托负责绘制和编辑，可以实现复杂的显示效果
```

### 4.2 QTreeWidget / QTreeView（树形控件）

// 【使用场景】
// - QTreeWidget: 中小数据量（< 5000 节点），快速开发，文件树、组织结构图
// - QTreeView + Model: 大数据量（> 5000 节点），文件系统浏览、复杂层级数据
// 【面试要点】何时用 Widget vs Model/View？如何实现三态复选框？

```cpp
// ========== QTreeWidget（便捷方式）==========
// 【特点】封装好的树形控件，操作简单
// 【优点】快速开发，内置编辑、拖拽、展开/折叠功能
// 【缺点】数据量大时性能差，内存占用高

// 创建树控件并设置表头
QTreeWidget* treeWidget = new QTreeWidget(this);
// setHeaderLabels: 设置每一列的标题，参数是字符串列表
treeWidget->setHeaderLabels(QStringList() << "名称" << "类型" << "大小");
//                     列0      列1        列2

// 隐藏表头（可选）
// treeWidget->setHeaderHidden(true);

// ========== 添加节点 ==========
// 【父子关系】创建节点时指定父项，自动建立层级关系
// 【内存管理】父项删除时，所有子项自动删除

// 添加顶层节点（根节点）
// 参数 treeWidget 表示这是一个顶层节点
QTreeWidgetItem* rootItem = new QTreeWidgetItem(treeWidget);
rootItem->setText(0, "根目录");
rootItem->setIcon(0, QIcon(":/icons/folder.png"));
rootItem->setToolTip(0, "这是根目录");

// 设置第一列宽度
treeWidget->setColumnWidth(0, 200);

// 添加子节点
QTreeWidgetItem* childItem1 = new QTreeWidgetItem(rootItem);
childItem1->setText(0, "子项1");
childItem1->setText(1, "文件");
childItem1->setText(2, "1.2 MB");
childItem1->setIcon(0, QIcon(":/icons/file.png"));

QTreeWidgetItem* childItem2 = new QTreeWidgetItem(rootItem);
childItem2->setText(0, "子项2");
childItem2->setText(1, "文件夹");
childItem2->setText(2, "-");
childItem2->setIcon(0, QIcon(":/icons/folder.png"));

// 批量添加子节点
for (int i = 0; i < 5; ++i) {
    QTreeWidgetItem* item = new QTreeWidgetItem(childItem2);
    item->setText(0, QString("项目 %1").arg(i + 1));
    item->setText(1, "文件");
    item->setText(2, QString("%1 KB").arg((i + 1) * 100));

    // 设置复选框
    item->setFlags(item->flags() | Qt::ItemIsUserCheckable);
    item->setCheckState(0, Qt::Unchecked);
}

// 展开节点
rootItem->setExpanded(true);
childItem2->setExpanded(true);

// 设置编辑属性
treeWidget->setEditTriggers(QAbstractItemView::DoubleClicked | QAbstractItemView::EditKeyPressed);

// 设置选择模式
treeWidget->setSelectionBehavior(QAbstractItemView::SelectRows);
treeWidget->setSelectionMode(QAbstractItemView::SingleSelection);

// 信号与槽
connect(treeWidget, &QTreeWidget::itemClicked, [](QTreeWidgetItem* item, int column) {
    qDebug() << "Clicked:" << item->text(column) << "Column:" << column;
});

connect(treeWidget, &QTreeWidget::itemDoubleClicked, [](QTreeWidgetItem* item, int column) {
    qDebug() << "Double clicked:" << item->text(column);
});

connect(treeWidget, &QTreeWidget::itemChanged, [](QTreeWidgetItem* item, int column) {
    qDebug() << "Changed:" << item->text(column);
});

connect(treeWidget, &QTreeWidget::itemExpanded, [](QTreeWidgetItem* item) {
    qDebug() << "Expanded:" << item->text(0);
});

connect(treeWidget, &QTreeWidget::itemCollapsed, [](QTreeWidgetItem* item) {
    qDebug() << "Collapsed:" << item->text(0);
});

// 连接复选框状态变化
connect(treeWidget, &QTreeWidget::itemChanged, [](QTreeWidgetItem* item, int column) {
    if (column == 0) {
        qDebug() << "Check state:" << item->checkState(0);
    }
});

// 遍历所有节点（递归）
void traverseTree(QTreeWidgetItem* item, int level = 0) {
    QString indent(level * 4, ' ');
    qDebug() << indent << item->text(0);

    for (int i = 0; i < item->childCount(); ++i) {
        traverseTree(item->child(i), level + 1);
    }
}

// 遍历顶层节点
for (int i = 0; i < treeWidget->topLevelItemCount(); ++i) {
    traverseTree(treeWidget->topLevelItem(i));
}

// 获取选中项
QTreeWidgetItem* current = treeWidget->currentItem();
if (current) {
    qDebug() << "Current item:" << current->text(0);
    qDebug() << "Parent:" << (current->parent() ? current->parent()->text(0) : "None");
}

// 查找项目
QList<QTreeWidgetItem*> items = treeWidget->findItems("子项1", Qt::MatchExactly | Qt::MatchRecursive);

// ========== QTreeView + Model（标准MVC模式）==========

// 1. 使用 QStandardItemModel
QTreeView* treeView = new QTreeView(this);
QStandardItemModel* standardModel = new QStandardItemModel(this);
standardModel->setHorizontalHeaderLabels(QStringList() << "名称" << "类型");

// 创建根节点
QStandardItem* rootItem = standardModel->invisibleRootItem();

// 添加一级节点
QStandardItem* item1 = new QStandardItem("部门1");
item1->setIcon(QIcon(":/icons/folder.png"));
rootItem->appendRow(item1);

// 添加二级节点
QStandardItem* subItem1 = new QStandardItem("员工1");
QStandardItem* subItem2 = new QStandardItem("员工2");
item1->appendRow(subItem1);
item1->appendRow(subItem2);

treeView->setModel(standardModel);
treeView->expandAll();

// 2. 使用 QFileSystemModel（文件系统模型）
QTreeView* fileTreeView = new QTreeView(this);
QFileSystemModel* fileModel = new QFileSystemModel(this);
fileModel->setRootPath(QDir::rootPath());

fileTreeView->setModel(fileModel);
fileTreeView->setRootIndex(fileModel->index(QDir::homePath()));
fileTreeView->setColumnWidth(0, 250);

// 隐藏某些列
fileTreeView->setColumnHidden(1, true);  // 隐藏大小列
fileTreeView->setColumnHidden(2, true);  // 隐藏类型列
fileTreeView->setColumnHidden(3, true);  // 隐藏修改时间列

// 3. 自定义树形 Model
class FileSystemModel : public QAbstractItemModel {
    Q_OBJECT
public:
    explicit FileSystemModel(QObject* parent = nullptr) : QAbstractItemModel(parent) {
        m_rootItem = new TreeItem({ "名称", "大小", "日期" });
        setupModelData(m_rootItem);
    }

    ~FileSystemModel() override {
        delete m_rootItem;
    }

    QModelIndex index(int row, int column,
                      const QModelIndex& parent = QModelIndex()) const override {
        if (!hasIndex(row, column, parent))
            return QModelIndex();

        TreeItem* parentItem;

        if (!parent.isValid())
            parentItem = m_rootItem;
        else
            parentItem = static_cast<TreeItem*>(parent.internalPointer());

        TreeItem* childItem = parentItem->child(row);
        if (childItem)
            return createIndex(row, column, childItem);
        return QModelIndex();
    }

    QModelIndex parent(const QModelIndex& index) const override {
        if (!index.isValid())
            return QModelIndex();

        TreeItem* childItem = static_cast<TreeItem*>(index.internalPointer());
        TreeItem* parentItem = childItem->parentItem();

        if (parentItem == m_rootItem)
            return QModelIndex();

        return createIndex(parentItem->row(), 0, parentItem);
    }

    int rowCount(const QModelIndex& parent = QModelIndex()) const override {
        TreeItem* parentItem;
        if (parent.column() > 0)
            return 0;

        if (!parent.isValid())
            parentItem = m_rootItem;
        else
            parentItem = static_cast<TreeItem*>(parent.internalPointer());

        return parentItem->childCount();
    }

    int columnCount(const QModelIndex& parent = QModelIndex()) const override {
        if (parent.isValid())
            return static_cast<TreeItem*>(parent.internalPointer())->columnCount();
        return m_rootItem->columnCount();
    }

    QVariant data(const QModelIndex& index, int role = Qt::DisplayRole) const override {
        if (!index.isValid())
            return QVariant();

        if (role != Qt::DisplayRole)
            return QVariant();

        TreeItem* item = static_cast<TreeItem*>(index.internalPointer());

        return item->data(index.column());
    }

    QVariant headerData(int section, Qt::Orientation orientation,
                       int role = Qt::DisplayRole) const override {
        if (orientation == Qt::Horizontal && role == Qt::DisplayRole)
            return m_rootItem->data(section);

        return QVariant();
    }

private:
    class TreeItem {
    public:
        explicit TreeItem(const QVector<QVariant>& data, TreeItem* parent = nullptr)
            : m_itemData(data), m_parentItem(parent) {}

        ~TreeItem() {
            qDeleteAll(m_childItems);
        }

        void appendChild(TreeItem* item) {
            m_childItems.append(item);
        }

        TreeItem* child(int row) const {
            return m_childItems.value(row);
        }

        int childCount() const {
            return m_childItems.count();
        }

        int columnCount() const {
            return m_itemData.count();
        }

        QVariant data(int column) const {
            return m_itemData.value(column);
        }

        int row() const {
            if (m_parentItem)
                return m_parentItem->m_childItems.indexOf(const_cast<TreeItem*>(this));
            return 0;
        }

        TreeItem* parentItem() const {
            return m_parentItem;
        }

    private:
        QVector<TreeItem*> m_childItems;
        QVector<QVariant> m_itemData;
        TreeItem* m_parentItem;
    };

    TreeItem* m_rootItem;

    void setupModelData(TreeItem* parent) {
        // 添加示例数据
        QVector<QVariant> data1;
        data1 << "文件夹1" << "2 KB" << "2024-01-01";
        TreeItem* folder1 = new TreeItem(data1, parent);
        parent->appendChild(folder1);

        QVector<QVariant> data2;
        data2 << "文件1.txt" << "1 KB" << "2024-01-02";
        folder1->appendChild(new TreeItem(data2, folder1));
    }
};
```

### 4.3 QTableWidget / QTableView（表格控件）

// 【使用场景】
// - QTableWidget: 中小数据量（< 10000 行），快速开发，数据表格、设置界面
// - QTableView + Model: 大数据量（> 10000 行），数据库显示、报表系统
// 【面试要点】如何实现单元格合并？如何排序？如何自定义绘制？

```cpp
// ========== QTableWidget（便捷方式）==========
// 【特点】封装好的表格控件，类似 Excel
// 【优点】使用简单，内置编辑、选择、排序功能
// 【缺点】大数据量性能差，内存占用高

// 创建表格控件
QTableWidget* tableWidget = new QTableWidget(this);

// ========== 设置表格尺寸 ==========
// setRowCount: 设置行数，可动态增减
tableWidget->setRowCount(10);     // 初始10行
// setColumnCount: 设置列数，通常固定
tableWidget->setColumnCount(5);   // 5列

// ========== 设置表头 ==========
// setHorizontalHeaderLabels: 设置水平表头（列标题）
// 参数：字符串列表，每个元素对应一列
tableWidget->setHorizontalHeaderLabels(
    QStringList() << "ID" << "姓名" << "年龄" << "部门" << "薪资"
//               列0    列1      列2      列3      列4
);

// ========== 表头样式设置 ==========
// horizontalHeader(): 获取水平表头对象
// QFont: 字体设置类
QFont headerFont = tableWidget->horizontalHeader()->font();
headerFont.setBold(true);         // 设置粗体
tableWidget->horizontalHeader()->setFont(headerFont);

// 设置单元格内容
tableWidget->setItem(0, 0, new QTableWidgetItem("001"));
tableWidget->setItem(0, 1, new QTableWidgetItem("张三"));
tableWidget->setItem(0, 2, new QTableWidgetItem("28"));

// 设置单元格对齐
QTableWidgetItem* salaryItem = new QTableWidgetItem("15000");
salaryItem->setTextAlignment(Qt::AlignRight | Qt::AlignVCenter);
tableWidget->setItem(0, 4, salaryItem);

// 使用组合框作为单元格
QComboBox* combo = new QComboBox();
combo->addItem("技术部");
combo->addItem("市场部");
combo->addItem("人事部");
tableWidget->setCellWidget(0, 3, combo);

// 使用复选框
QTableWidgetItem* checkItem = new QTableWidgetItem();
checkItem->setFlags(Qt::ItemIsUserCheckable | Qt::ItemIsEnabled);
checkItem->setCheckState(Qt::Unchecked);
tableWidget->setItem(1, 0, checkItem);

// 设置表格属性
tableWidget->setSelectionBehavior(QAbstractItemView::SelectRows);   // 整行选择
tableWidget->setSelectionMode(QAbstractItemView::SingleSelection);  // 单选
tableWidget->setEditTriggers(QAbstractItemView::DoubleClicked);     // 双击编辑
tableWidget->setAlternatingRowColors(true);                         // 交替行颜色
tableWidget->setSortingEnabled(true);                               // 允许排序

// 表头设置
tableWidget->horizontalHeader()->setStretchLastSection(true);       // 最后一列拉伸
tableWidget->horizontalHeader()->setSectionResizeMode(0, QHeaderView::Interactive);
tableWidget->horizontalHeader()->setSectionResizeMode(1, QHeaderView::Stretch);
tableWidget->verticalHeader()->setVisible(false);                    // 隐藏行号

// 合并单元格
tableWidget->setSpan(0, 0, 2, 1); // 从(0,0)开始，合并2行1列

// 冻结行/列（需要使用 QTableView + SelectionModel）
tableWidget->setRowHeight(0, 40);  // 设置行高
tableWidget->setColumnWidth(0, 80);  // 设置列宽

// 信号与槽
connect(tableWidget, &QTableWidget::cellClicked, [](int row, int column) {
    qDebug() << "Clicked cell:" << row << column;
});

connect(tableWidget, &QTableWidget::cellDoubleClicked, [](int row, int column) {
    qDebug() << "Double clicked:" << row << column;
});

connect(tableWidget, &QTableWidget::cellChanged, [](int row, int column) {
    qDebug() << "Cell changed:" << row << column;
});

connect(tableWidget, &QTableWidget::currentCellChanged, [](int row, int col, int prevRow, int prevCol) {
    qDebug() << "Current changed:" << row << col << "from:" << prevRow << prevCol;
});

connect(tableWidget, &QTableWidget::itemSelectionChanged, []() {
    qDebug() << "Selection changed";
});

// 获取选中行
QList<QTableWidgetItem*> selectedItems = tableWidget->selectedItems();
QSet<int> rows;
for (QTableWidgetItem* item : selectedItems) {
    rows.insert(item->row());
}
qDebug() << "Selected rows:" << rows.toList();

// 获取指定单元格
QTableWidgetItem* item = tableWidget->item(0, 1);
if (item) {
    qDebug() << "Cell value:" << item->text();
    qDebug() << "Cell flags:" << item->flags();
}

// 设置行/列颜色
for (int row = 0; row < tableWidget->rowCount(); ++row) {
    for (int col = 0; col < tableWidget->columnCount(); ++col) {
        QTableWidgetItem* cell = tableWidget->item(row, col);
        if (!cell) {
            cell = new QTableWidgetItem();
            tableWidget->setItem(row, col, cell);
        }

        // 偶数行设置背景色
        if (row % 2 == 0) {
            cell->setBackground(QColor(240, 240, 240));
        }
    }
}

// 右键菜单
tableWidget->setContextMenuPolicy(Qt::CustomContextMenu);
connect(tableWidget, &QTableWidget::customContextMenuRequested, [=](const QPoint& pos) {
    QMenu menu;
    menu.addAction("添加行");
    menu.addAction("删除行");
    menu.addAction("清空");
    menu.exec(tableWidget->mapToGlobal(pos));
});

// ========== QTableView + Model（推荐用于大数据量）==========

// 自定义 Table Model
class TableModel : public QAbstractTableModel {
    Q_OBJECT
public:
    struct RowData {
        QString id;
        QString name;
        QString age;
        QString department;
        QString salary;
    };

    explicit TableModel(QObject* parent = nullptr) : QAbstractTableModel(parent) {
        // 初始化数据
        for (int i = 0; i < 100; ++i) {
            m_data.append({
                QString::number(i + 1, 'd', 3).rightJustified(3, '0'),
                QString("用户%1").arg(i + 1),
                QString::number(20 + (i % 40)),
                (i % 3 == 0) ? "技术部" : (i % 3 == 1) ? "市场部" : "人事部",
                QString::number(5000 + (i * 1000))
            });
        }
    }

    int rowCount(const QModelIndex& parent = QModelIndex()) const override {
        Q_UNUSED(parent)
        return m_data.size();
    }

    int columnCount(const QModelIndex& parent = QModelIndex()) const override {
        Q_UNUSED(parent)
        return 5;
    }

    QVariant data(const QModelIndex& index, int role = Qt::DisplayRole) const override {
        if (!index.isValid() || index.row() >= m_data.size())
            return QVariant();

        const RowData& row = m_data[index.row()];

        if (role == Qt::DisplayRole || role == Qt::EditRole) {
            switch (index.column()) {
                case 0: return row.id;
                case 1: return row.name;
                case 2: return row.age;
                case 3: return row.department;
                case 4: return row.salary;
            }
        }

        if (role == Qt::TextAlignmentRole) {
            if (index.column() == 4)  // 薪资列右对齐
                return Qt::AlignRight | Qt::AlignVCenter;
        }

        if (role == Qt::BackgroundRole) {
            if (index.row() % 2 == 0)
                return QColor(245, 245, 245);
        }

        return QVariant();
    }

    QVariant headerData(int section, Qt::Orientation orientation,
                       int role = Qt::DisplayRole) const override {
        if (role == Qt::DisplayRole && orientation == Qt::Horizontal) {
            switch (section) {
                case 0: return "ID";
                case 1: return "姓名";
                case 2: return "年龄";
                case 3: return "部门";
                case 4: return "薪资";
            }
        }

        if (role == Qt::FontRole && orientation == Qt::Horizontal) {
            QFont font;
            font.setBold(true);
            return font;
        }

        return QVariant();
    }

    Qt::ItemFlags flags(const QModelIndex& index) const override {
        if (!index.isValid())
            return Qt::NoItemFlags;

        return QAbstractTableModel::flags(index) | Qt::ItemIsEditable;
    }

    bool setData(const QModelIndex& index, const QVariant& value, int role = Qt::EditRole) override {
        if (!index.isValid() || role != Qt::EditRole || index.row() >= m_data.size())
            return false;

        RowData& row = m_data[index.row()];

        switch (index.column()) {
            case 0: row.id = value.toString(); break;
            case 1: row.name = value.toString(); break;
            case 2: row.age = value.toString(); break;
            case 3: row.department = value.toString(); break;
            case 4: row.salary = value.toString(); break;
            default: return false;
        }

        emit dataChanged(index, index);
        return true;
    }

    // 添加行
    void appendRow(const RowData& data) {
        beginInsertRows(QModelIndex(), m_data.size(), m_data.size());
        m_data.append(data);
        endInsertRows();
    }

    // 删除行
    void removeRow(int row) {
        if (row < 0 || row >= m_data.size())
            return;

        beginRemoveRows(QModelIndex(), row, row);
        m_data.removeAt(row);
        endRemoveRows();
    }

private:
    QList<RowData> m_data;
};

// 使用 TableModel
QTableView* tableView = new QTableView(this);
TableModel* model = new TableModel(this);

tableView->setModel(model);

// 视图设置
tableView->setSelectionBehavior(QAbstractItemView::SelectRows);
tableView->setSelectionMode(QAbstractItemView::SingleSelection);
tableView->setAlternatingRowColors(true);
tableView->setSortingEnabled(true);

// 列宽设置
tableView->horizontalHeader()->setStretchLastSection(true);
tableView->horizontalHeader()->setSectionResizeMode(0, QHeaderView::Interactive);
tableView->horizontalHeader()->setSectionResizeMode(1, QHeaderView::Stretch);
tableView->setColumnWidth(0, 80);

// 自定义委托
class SpinBoxDelegate : public QStyledItemDelegate {
    Q_OBJECT
public:
    QWidget* createEditor(QWidget* parent, const QStyleOptionViewItem& option,
                         const QModelIndex& index) const override {
        Q_UNUSED(option)
        Q_UNUSED(index)

        QSpinBox* editor = new QSpinBox(parent);
        editor->setFrame(false);
        editor->setMinimum(0);
        editor->setMaximum(100);
        return editor;
    }

    void setEditorData(QWidget* editor, const QModelIndex& index) const override {
        int value = index.model()->data(index, Qt::EditRole).toInt();
        QSpinBox* spinBox = static_cast<QSpinBox*>(editor);
        spinBox->setValue(value);
    }

    void setModelData(QWidget* editor, QAbstractItemModel* model,
                     const QModelIndex& index) const override {
        QSpinBox* spinBox = static_cast<QSpinBox*>(editor);
        spinBox->interpretText();
        int value = spinBox->value();
        model->setData(index, value, Qt::EditRole);
    }

    void updateEditorGeometry(QWidget* editor, const QStyleOptionViewItem& option,
                             const QModelIndex& index) const override {
        Q_UNUSED(index)
        editor->setGeometry(option.rect);
    }
};

// 为特定列设置委托
tableView->setItemDelegateForColumn(2, new SpinBoxDelegate(this));  // 年龄列
```

### 4.4 QChart（图表控件）

// 【使用场景】
// - 折线图: 显示趋势变化（股票走势、温度变化、销售趋势）
// - 柱状图: 比较数据大小（销售额对比、成绩统计、资源使用）
// - 饼图: 显示占比关系（市场份额、磁盘使用、人口分布）
// - 散点图: 显示数据分布（相关性分析、抽样数据）
// - 面积图: 显示累积趋势（流量统计、降雨量）

**需要添加模块: `QT += charts`**

```cpp
// ========== 基础设置 ==========
#include <QtCharts>

// QT_CHARTS_USE_NAMESPACE: 使用 QtCharts 命名空间，避免前缀 QtCharts::
QT_CHARTS_USE_NAMESPACE

// ========== 创建图表视图 ==========
// QChartView: 显示图表的控件（继承自 QGraphicsView）
// QChart: 图表数据容器
QChartView* chartView = new QChartView(this);
QChart* chart = new QChart();
chart->setTitle("数据统计分析");                    // 设置图表标题
// setAnimationOptions: 设置动画效果
//   - NoAnimation: 无动画
//   - GridAxisAnimations: 坐标轴动画
//   - SeriesAnimations: 系列动画（推荐）
//   - AllAnimations: 全部动画
chart->setAnimationOptions(QChart::SeriesAnimations);
chartView->setChart(chart);

// ========== 1. 折线图（QLineSeries）==========
// 【应用场景】趋势分析、时间序列数据
// 【特点】显示数据随时间/类别变化的趋势

// QLineSeries: 折线系列，继承自 QXYSeries
QLineSeries* lineSeries = new QLineSeries();
lineSeries->setName("2024年销售额");  // 图例显示的名称

// append(x, y): 添加数据点
// x: X轴值（通常是类别或时间）
// y: Y轴值（数值）
lineSeries->append(0, 15);   // 点1: (0, 15)
lineSeries->append(1, 28);   // 点2: (1, 28)
lineSeries->append(2, 35);   // 点3: (2, 35)
lineSeries->append(3, 42);   // 点4: (3, 42)
lineSeries->append(4, 58);   // 点5: (4, 58)
lineSeries->append(5, 65);   // 点6: (5, 65)
lineSeries->append(6, 72);   // 点7: (6, 72)
lineSeries->append(7, 88);   // 点8: (7, 88)
lineSeries->append(8, 95);
lineSeries->append(9, 105);
lineSeries->append(10, 115);
lineSeries->append(11, 128);

// 设置线条样式
lineSeries->setColor(QColor("#3498db"));
lineSeries->setPointsVisible(true);  // 显示数据点
lineSeries->setPointLabelsVisible(true);  // 显示点标签
lineSeries->setPointLabelsFormat("(@xPoint, @yPoint)");
lineSeries->setPointLabelsClipping(false);  // 标签不裁剪

// 设置线条宽度
QPen pen(QColor("#3498db"));
pen.setWidth(3);
lineSeries->setPen(pen);

chart->addSeries(lineSeries);

// ========== 2. 柱状图 ==========

QBarSeries* barSeries = new QBarSeries();
barSeries->setName("年度对比");

QBarSet* set2023 = new QBarSet("2023年");
QBarSet* set2024 = new QBarSet("2024年");

*set2023 << 120 << 150 << 180 << 210 << 250 << 280
         << 300 << 320 << 350 << 380 << 400 << 420;
*set2024 << 150 << 180 << 220 << 280 << 320 << 360
         << 400 << 450 << 500 << 550 << 600 << 650;

barSeries->append(set2023);
barSeries->append(set2024);

chart->addSeries(barSeries);

// 设置柱状图颜色
set2023->setColor(QColor("#95a5a6"));
set2024->setColor(QColor("#3498db"));

// ========== 3. 饼图 ==========

QPieSeries* pieSeries = new QPieSeries();
pieSeries->setHoleSize(0.3);  // 设置为环形图

QPieSlice* slice1 = pieSeries->append("技术部", 35);
QPieSlice* slice2 = pieSeries->append("市场部", 28);
QPieSlice* slice3 = pieSeries->append("人事部", 15);
QPieSlice* slice4 = pieSeries->append("财务部", 12);
QPieSlice* slice5 = pieSeries->append("其他", 10);

// 设置切片样式
slice1->setColor(QColor("#3498db"));
slice1->setLabelVisible(true);
slice1->setLabelColor(Qt::white);
slice1->setLabel("技术部\n35%");

slice2->setColor(QColor("#2ecc71"));
slice2->setLabelVisible(true);
slice2->setLabelColor(Qt::white);

// 突出显示某个切片
slice1->setExploded(true);
slice1->setExplodeDistanceFactor(0.1);

chart->addSeries(pieSeries);

// ========== 4. 面积图 ==========
QAreaSeries* areaSeries = new QAreaSeries();
QLineSeries* upperSeries = new QLineSeries();
upperSeries->append(0, 50);
upperSeries->append(1, 60);
upperSeries->append(2, 70);
upperSeries->append(3, 80);
upperSeries->append(4, 90);

QLineSeries* lowerSeries = new QLineSeries();
lowerSeries->append(0, 30);
lowerSeries->append(1, 40);
lowerSeries->append(2, 45);
lowerSeries->append(3, 50);
lowerSeries->append(4, 55);

areaSeries->setUpperSeries(upperSeries);
areaSeries->setLowerSeries(lowerSeries);
areaSeries->setColor(QColor("#9b59b6"));
areaSeries->setBorderColor(QColor("#8e44ad"));

chart->addSeries(areaSeries);

// ========== 5. 散点图 ==========
QScatterSeries* scatterSeries = new QScatterSeries();
scatterSeries->setName("数据分布");
scatterSeries->setMarkerSize(10);
scatterSeries->setColor(QColor("#e74c3c"));

// 添加随机数据
for (int i = 0; i < 50; ++i) {
    scatterSeries->append(i, qrand() % 100);
}

chart->addSeries(scatterSeries);

// ========== 坐标轴设置 ==========

// 创建数值轴
QValueAxis* axisX = new QValueAxis();
axisX->setTitleText("月份");
axisX->setRange(0, 12);
axisX->setTickCount(13);
axisX->setLabelFormat("%d");

QValueAxis* axisY = new QValueAxis();
axisY->setTitleText("销售额(万元)");
axisY->setRange(0, 150);
axisY->setTickCount(10);
axisY->setLabelFormat("%d");

// 添加到图表
chart->addAxis(axisX, Qt::AlignBottom);
chart->addAxis(axisY, Qt::AlignLeft);

// 关联系列与坐标轴
lineSeries->attachAxis(axisX);
lineSeries->attachAxis(axisY);

// ========== 类别轴（用于柱状图）==========
QStringList categories;
categories << "1月" << "2月" << "3月" << "4月" << "5月" << "6月"
          << "7月" << "8月" << "9月" << "10月" << "11月" << "12月";

QBarCategoryAxis* axisXCat = new QBarCategoryAxis();
axisXCat->append(categories);
chart->addAxis(axisXCat, Qt::AlignBottom);
barSeries->attachAxis(axisXCat);

// ========== 日期时间轴 ==========
QDateTimeAxis* axisDateTime = new QDateTimeAxis();
axisDateTime->setTitleText("时间");
axisDateTime->setFormat("yyyy-MM-dd");
axisDateTime->setCount(10);

QDateTime start = QDateTime::currentDateTime();
QDateTime end = start.addDays(30);
axisDateTime->setRange(start, end);

chart->addAxis(axisDateTime, Qt::AlignBottom);

// ========== 图例设置 ==========
chart->legend()->setAlignment(Qt::AlignBottom);
chart->legend()->setBackgroundVisible(true);
chart->legend()->setBorderColor(QColor(200, 200, 200));
chart->legend()->setFont(QFont("Arial", 10));

// 反向图例
chart->legend()->setReverse(true);

// 隐藏图例
// chart->legend()->hide();

// ========== 交互功能 ==========

// 设置视图可拖拽
chartView->setRubberBand(QChartView::RectangleRubberBand);

// 设置缩放
chartView->setRenderHint(QPainter::Antialiasing);

// 响应点击
connect(scatterSeries, &QScatterSeries::clicked, [=](const QPointF& point) {
    qDebug() << "Clicked point:" << point.x() << point.y();
});

// 响应悬停
connect(lineSeries, &QLineSeries::hovered, [=](const QPointF& point, bool state) {
    if (state) {
        qDebug() << "Hovered:" << point.x() << point.y();
        chartView->setToolTip(QString("Value: %1").arg(point.y()));
    }
});
```

### 4.5 QGraphicsScene / QGraphicsView（图形视图框架）

// 【使用场景】
// - 绘图应用：画板、流程图设计器、CAD 软件
// - 游戏：2D 游戏场景、地图编辑器
// - 数据可视化：网络拓扑图、组织结构图
// - 图像编辑：图片标注、图层管理

```cpp
// ========== 图形视图框架概述 ==========
/*
 * 【三核心类】
 * QGraphicsScene  - 场景，管理所有图形项（数据层）
 * QGraphicsView  - 视图，显示场景内容（显示层）
 * QGraphicsItem  - 图形项，场景中的元素（元素层）
 *
 * 【坐标系统】
 *
 *  ┌───────────────────────────────────────────────────────────┐
 *  │                    QGraphicsView (视图坐标)                │
 *  │                    屏幕像素坐标系统                          │
 *  │  (0,0) 在左上角，X向右增长，Y向下增长                        │
 *  │                                                             │
 *  │   ┌────────────────────────────────────────────┐           │
 *   │   │         QGraphicsScene (场景坐标)          │           │
 *   │   │         逻辑坐标系统，可任意设置             │           │
 *   │   │   (0,0) 可在中心，默认在左上角               │           │
 *   │   │                                            │           │
 *   │   │   ┌──────────────────┐                    │           │
 *   │   │   │ QGraphicsItem   │                    │           │
 *   │   │   │ (图形项坐标)      │                    │           │
 *   │   │   │ 相对坐标系统       │                    │           │
 *   │   │   │ (0,0) 在项中心    │                    │           │
 *   │   │   └──────────────────┘                    │           │
 *   │   └────────────────────────────────────────────┘           │
 *   └───────────────────────────────────────────────────────────┘
 *
 * 【坐标转换函数】
 * - view->mapToScene(): 视图坐标 → 场景坐标
 * - view->mapFromScene(): 场景坐标 → 视图坐标
 * - item->mapToScene(): 图形项坐标 → 场景坐标
 * - item->mapFromScene(): 场景坐标 → 图形项坐标
 * - item->mapToParent(): 图形项坐标 → 父项坐标
 * - item->mapFromParent(): 父项坐标 → 图形项坐标
 */

// ========== 创建场景和视图 ==========
// QGraphicsScene: 场景，管理所有图形项
QGraphicsScene* scene = new QGraphicsScene(this);
// setSceneRect(x, y, w, h): 设置场景矩形范围
// 参数：x=左上角X, y=左上角Y, w=宽度, h=高度
// 设置为 (-200, -150, 400, 300) 表示 (0,0) 在场景中心
scene->setSceneRect(-200, -150, 400, 300);

// QGraphicsView: 视图，显示场景的窗口
QGraphicsView* view = new QGraphicsView(scene, this);
// setRenderHint: 设置渲染提示
//   - Antialiasing: 抗锯齿（线条更平滑）
//   - TextAntialiasing: 文字抗锯齿
//   - SmoothPixmapTransform: 图片平滑缩放
view->setRenderHint(QPainter::Antialiasing);  // 抗锯齿

// setDragMode: 设置拖拽模式
//   - NoDrag: 禁止拖拽
//   - ScrollHandDrag: 鼠标拖动平移视图
//   - RubberBandDrag: 橡皮筋选择框（推荐）
view->setDragMode(QGraphicsView::RubberBandDrag);

// setViewportUpdateMode: 视图更新模式
//   - FullViewportUpdate: 完全重绘（最慢但最准确）
//   - MinimalViewportUpdate: 最小更新区域
//   - SmartViewportUpdate: 智能更新（推荐）
view->setViewportUpdateMode(QGraphicsView::FullViewportUpdate);

// setTransformationAnchor: 变换锚点（缩放/旋转时的中心点）
//   - AnchorUnderMouse: 鼠标位置（推荐）
//   - AnchorViewCenter: 视图中心
view->setTransformationAnchor(QGraphicsView::AnchorUnderMouse);
// setResizeAnchor: 调整大小锚点
view->setResizeAnchor(QGraphicsView::AnchorCenter);

// ========== 1. 基本图形项 ==========

// 【矩形】QGraphicsRectItem(x, y, w, h)
// 参数：x=左上角X（相对坐标）, y=左上角Y, w=宽度, h=高度
QGraphicsRectItem* rectItem = new QGraphicsRectItem(-50, -50, 100, 100);
// setPos(x, y): 设置图形项在场景中的位置
rectItem->setPos(0, 0);  // 将矩形中心放在场景 (0,0) 处

// setBrush(): 设置填充画刷
// QBrush: 画刷类，可设置颜色、渐变、图片
rectItem->setBrush(QBrush(QColor("#3498db")));  // 蓝色填充

// setPen(): 设置边框画笔
// QPen: 画笔类，可设置颜色、宽度、样式
// QPen(color, width): 颜色, 像素宽度
rectItem->setPen(QPen(Qt::black, 2));  // 黑色边框，2像素宽

// setFlag(): 设置图形项标志
// ItemIsMovable: 可移动
// ItemIsSelectable: 可选择
// ItemIsFocusable: 可获取键盘焦点
// ItemIsHoverable: 可响应鼠标悬停
// ItemSendsGeometryChanges: 发送位置变化信号
rectItem->setFlag(QGraphicsItem::ItemIsMovable);       // 可移动
rectItem->setFlag(QGraphicsItem::ItemIsSelectable);    // 可选择
rectItem->setFlag(QGraphicsItem::ItemIsFocusable);     // 可聚焦
rectItem->setFlag(QGraphicsItem::ItemIsHoverable);     // 可悬停

// setZValue(): 设置 Z 值（层级）
// 数值越大，显示越靠前（在上面）
rectItem->setZValue(1);  // 层级1

scene->addItem(rectItem);  // 将图形项添加到场景

// 圆角矩形
QGraphicsRectItem* roundRect = new QGraphicsRectItem(-30, -30, 60, 60);
roundRect->setPos(-100, -50);
roundRect->setBrush(QBrush(QColor("#e74c3c")));

// 设置圆角（需要通过 painter 绘制）
class RoundedRectItem : public QGraphicsItem {
public:
    RoundedRectItem(QGraphicsItem* parent = nullptr) : QGraphicsItem(parent) {
        setFlag(ItemIsMovable);
    }

    // boundingRect(): 必须重写，返回图形项的边界矩形
    QRectF boundingRect() const override {
        return QRectF(-40, -40, 80, 80);  // 中心在 (0,0)，宽80高80
    }

    // paint(): 必须重写，绘制图形项
    void paint(QPainter* painter, const QStyleOptionGraphicsItem*, QWidget*) override {
        painter->setBrush(QBrush(QColor("#e74c3c")));
        painter->setPen(QPen(Qt::darkRed, 2));
        // drawRoundedRect(): 绘制圆角矩形
        // 参数：矩形, x半径, y半径
        painter->drawRoundedRect(boundingRect(), 10, 10);
    }
};
scene->addItem(new RoundedRectItem());

// 椭圆
QGraphicsEllipseItem* ellipseItem = new QGraphicsEllipseItem(-40, -40, 80, 80);
ellipseItem->setPos(100, 0);
ellipseItem->setBrush(QBrush(QColor("#2ecc71")));
ellipseItem->setPen(QPen(Qt::darkGreen, 2));
ellipseItem->setFlag(QGraphicsItem::ItemIsMovable);
scene->addItem(ellipseItem);

// 文本
QGraphicsTextItem* textItem = new QGraphicsTextItem("Hello Qt Graphics");
textItem->setPos(-80, -100);
textItem->setDefaultTextColor(QColor("#2c3e50"));
QFont font("Arial", 16, QFont::Bold);
textItem->setFont(font);
scene->addItem(textItem);

// 线条
QGraphicsLineItem* lineItem = new QGraphicsLineItem(-150, 80, 150, 80);
lineItem->setPen(QPen(Qt::darkBlue, 3, Qt::DashLine));
scene->addItem(lineItem);

// 多边形
QGraphicsPolygonItem* polygonItem = new QGraphicsPolygonItem();
QPolygonF polygon;
polygon << QPointF(0, -50) << QPointF(50, 50) << QPointF(-50, 50);
polygonItem->setPolygon(polygon);
polygonItem->setPos(0, 100);
polygonItem->setBrush(QBrush(QColor("#f39c12")));
polygonItem->setPen(QPen(Qt::darkYellow, 2));
polygonItem->setFlag(QGraphicsItem::ItemIsMovable);
scene->addItem(polygonItem);

// 路径
QGraphicsPathItem* pathItem = new QGraphicsPathItem();
QPainterPath path;
path.moveTo(0, 0);
path.cubicTo(50, -50, 100, 50, 150, 0);
pathItem->setPath(path);
pathItem->setPos(-150, 0);
pathItem->setPen(QPen(QColor("#9b59b6"), 3));
scene->addItem(pathItem);

// 图片
QGraphicsPixmapItem* pixmapItem = new QGraphicsPixmapItem(QPixmap(":/images/logo.png"));
pixmapItem->setPos(100, 100);
pixmapItem->setFlag(QGraphicsItem::ItemIsMovable);
pixmapItem->setScale(0.5);  // 缩放
scene->addItem(pixmapItem);

// ========== 2. 自定义图形项 ==========

class CustomItem : public QGraphicsItem {
public:
    CustomItem(QGraphicsItem* parent = nullptr) : QGraphicsItem(parent), m_angle(0) {
        setFlag(QGraphicsItem::ItemIsMovable);
        setFlag(QGraphicsItem::ItemIsSelectable);
        setFlag(QGraphicsItem::ItemIsFocusable);
        setAcceptHoverEvents(true);  // 接受悬停事件

        // 启动动画定时器
        connect(&m_timer, &QTimer::timeout, this, [this]() {
            m_angle = (m_angle + 5) % 360;
            update();
        });
        m_timer.start(50);
    }

    QRectF boundingRect() const override {
        return QRectF(-50, -50, 100, 100);
    }

    void paint(QPainter* painter, const QStyleOptionGraphicsItem*, QWidget*) override {
        // 绘制渐变背景
        QRadialGradient gradient(0, 0, 50);
        gradient.setColorAt(0, QColor(60 + m_angle / 10, 140, 200));
        gradient.setColorAt(1, QColor(40, 100, 160));

        painter->setBrush(QBrush(gradient));
        painter->setPen(QPen(Qt::darkGray, 2));

        // 绘制旋转的图形
        painter->save();
        painter->rotate(m_angle);
        painter->drawEllipse(-30, -30, 60, 60);
        painter->drawLine(-30, 0, 30, 0);
        painter->drawLine(0, -30, 0, 30);
        painter->restore();

        // 绘制文字
        painter->setPen(Qt::white);
        QFont font("Arial", 12, QFont::Bold);
        painter->setFont(font);
        painter->drawText(boundingRect(), Qt::AlignCenter, "Custom");
    }

protected:
    void hoverEnterEvent(QGraphicsSceneHoverEvent*) override {
        setScale(1.2);
        setCursor(Qt::PointingHandCursor);
        update();
    }

    void hoverLeaveEvent(QGraphicsSceneHoverEvent*) override {
        setScale(1.0);
        setCursor(Qt::ArrowCursor);
        update();
    }

    void mousePressEvent(QGraphicsSceneMouseEvent* event) override {
        if (event->button() == Qt::LeftButton) {
            m_timer.stop();  // 点击停止动画
        }
        QGraphicsItem::mousePressEvent(event);
    }

private:
    int m_angle;
    QTimer m_timer;
};

scene->addItem(new CustomItem());

// ========== 3. 图形组 ==========

QGraphicsItemGroup* group = scene->createItemGroup(QList<QGraphicsItem*>());

// 向组中添加项目
QGraphicsRectItem* rect1 = new QGraphicsRectItem(-20, -20, 40, 40);
rect1->setBrush(Qt::red);
scene->addItem(rect1);

QGraphicsRectItem* rect2 = new QGraphicsRectItem(20, -20, 40, 40);
rect2->setBrush(Qt::blue);
scene->addItem(rect2);

// 添加到组
group->addToGroup(rect1);
group->addToGroup(rect2);

// 组整体移动
group->setPos(-50, -100);
group->setFlag(QGraphicsItem::ItemIsMovable);

// ========== 4. 坐标变换 ==========

// 旋转
rectItem->setRotation(45);

// 缩放
ellipseItem->setScale(1.5);

// 平移（通过 pos）
textItem->setPos(50, 50);

// 组合变换
QTransform transform;
transform.rotate(30);
transform.scale(1.5, 1.5);
rectItem->setTransform(transform);

// ========== 5. 碰撞检测 ==========

// 检测与指定项的碰撞
QList<QGraphicsItem*> collidingItems = rectItem->collidingItems();
for (QGraphicsItem* item : collidingItems) {
    qDebug() << "Colliding with:" << item->type();
}

// 按形状检测碰撞
collidingItems = rectItem->collidingItems(Qt::IntersectsItemShape);

// 检测两个项是否碰撞
if (rectItem->collidesWithItem(ellipseItem)) {
    qDebug() << "Rectangle collides with ellipse";
}

// 检测点是否在项内
if (rectItem->contains(QPointF(10, 10))) {
    qDebug() << "Point (10, 10) is inside rectangle";
}

// ========== 6. 选择与焦点 ==========

// 设置选择模式
scene->setSelectionArea(QPainterPath());

// 获取选中项
QList<QGraphicsItem*> selected = scene->selectedItems();
for (QGraphicsItem* item : selected) {
    qDebug() << "Selected item type:" << item->type();
}

// 设置焦点
rectItem->setFocus();
if (rectItem->hasFocus()) {
    qDebug() << "Rectangle has focus";
}

// ========== 7. 场景操作 ==========

// 居中显示
view->centerOn(rectItem);

// 适应视图
view->fitInView(scene->sceneRect(), Qt::KeepAspectRatio);

// 缩放
view->scale(1.2, 1.2);  // 放大 20%
view->rotate(15);       // 旋转视图 15 度

// 设置拖拽模式
view->setDragMode(QGraphicsView::ScrollHandDrag);     // 手形拖拽
view->setDragMode(QGraphicsView::RubberBandDrag);     // 橡皮筋选择
view->setDragMode(QGraphicsView::NoDrag);             // 无拖拽

// ========== 8. 图层与 Z 值 ==========
/*
 * Z 值决定绘制顺序：
 * - Z 值大的项在上层
 * - Z 值相同时，后添加的项在上层
 */
rectItem->setZValue(10);
ellipseItem->setZValue(5);

// ========== 9. 事件处理 ==========
connect(scene, &QGraphicsScene::selectionChanged, []() {
    qDebug() << "Selection changed";
});

connect(scene, &QGraphicsScene::focusItemChanged, [](QGraphicsItem* newFocus, QGraphicsItem* oldFocus, Qt::FocusReason reason) {
    qDebug() << "Focus changed from" << oldFocus << "to" << newFocus;
});

// ========== 10. 场景索引（优化性能）==========
// 对于大量图形项，使用场景索引提高碰撞检测性能
scene->setItemIndexMethod(QGraphicsScene::BspTreeIndex);  // BSP树索引
// scene->setItemIndexMethod(QGraphicsScene::NoIndex);    // 无索引
```

---

## 五、QSS 样式表

### 5.1 基本语法与加载方式

// 【知识铺垫】QSS 是 Qt 的样式表语言，类似 CSS，用于定制界面外观
// 【核心概念】样式优先级：具体控件 > 类选择器 > 全局样式
// 【面试要点】QSS 选择器优先级、伪状态使用是常见考点

```cpp
// ========== 方式1：代码中直接设置（适合动态样式）==========
widget->setStyleSheet("background-color: #f0f0f0; color: #333;");
// 【注意】频繁调用会影响性能，建议在初始化时设置

// ========== 方式2：从文件加载（推荐，便于维护）==========
QFile file(":/styles/style.qss");  // 资源文件中定义
if (file.open(QFile::ReadOnly)) {
    QString styleSheet = QLatin1String(file.readAll());
    qApp->setStyleSheet(styleSheet);  // 全局应用，影响所有控件
    file.close();
}

// ========== 方式3：资源文件 ==========
// 在 .qrc 文件中定义：
// <file>styles/style.qss</file>
// 编译时会将 QSS 文件嵌入可执行文件
```

```cpp
// ========== 选择器类型 ==========
/*
 * 选择器类型      | 语法示例              | 说明
 * ─────────────────────────────────────────────────────────────
 * 类型选择器      | QPushButton           | 匹配所有 QPushButton
 * ID 选择器        | QPushButton#okBtn    | 匹配 objectName() 为 "okBtn" 的按钮
 * 类选择器        | .QDialog              | 匹配 QDialog 及其子类（注意前面有 .）
 * 属性选择器      | QPushButton[flat="true"] | 匹配 flat 属性为 true 的按钮
 * 后代选择器      | QDialog QPushButton   | 匹配 QDialog 下的 QPushButton
 * 伪状态          | QPushButton:hover      | 鼠标悬停时的样式
 * 伪元素          | QComboBox::drop-down  | 控件的子元素（下拉按钮等）
 *
 * 【优先级】ID选择器 > 属性选择器 > 类选择器 > 后代选择器 > 类型选择器
 */
```

### 5.2 选择器类型

```css
/* ========== 1. 类型选择器 ==========
 * 【作用】匹配所有指定类型的控件
 * 【语法】类名 { 样式规则 }
 * 【优先级】最低，会被 ID 选择器覆盖
 */
QPushButton {
    background-color: #3498db;      /* 蓝色背景 */
    color: white;                 /* 白色文字 */
    border: none;                 /* 无边框 */
    padding: 8px 16px;             /* 内边距：上下8px，左右16px */
    border-radius: 4px;           /* 圆角半径4像素 */
    font-size: 14px;              /* 字号14像素 */
}

/* ========== 2. ID 选择器 ==========
 * 【作用】匹配指定 objectName() 的控件
 * 【语法】类名#objectName { 样式 }
 * 【优先级】最高
 * 【注意】需要在代码中设置 setObjectName("okButton")
 */
QPushButton#okButton {
    background-color: #27ae60;    /* 绿色 */
}

QPushButton#cancelButton {
    background-color: #e74c3c;    /* 红色 */
}

/* ========== 3. 属性选择器 ==========
 * 【作用】匹配指定属性值的控件
 * 【语法】类名[属性="值"] { 样式 }
 * 【优先级】中等
 * 【用法】setStyleSheet("class=\"primary\"");
 */
QPushButton[class="primary"] {
    background-color: #2980b9;    /* 深蓝色 */
}

QPushButton[class="secondary"] {
    background-color: #95a5a6;    /* 灰色 */
}

/* ========== 4. 后代选择器 ==========
 * 【作用】匹配指定容器下的控件
 * 【语法】父控件 子控件 { 样式 }
 * 【优先级】低
 */
QDialog QPushButton {
    min-width: 80px;              /* 最小宽度80像素 */
    min-height: 25px;             /* 最小高度25像素 */
}

QGroupBox QPushButton {
    background-color: #ecf0f1;    /* 浅灰背景 */
    color: #2c3e50;               /* 深灰文字 */
}

/* ========== 5. 子控件选择器 ==========
 * 【作用】匹配控件的子元素（下拉按钮、滚动条等）
 * 【语法】控件::子控件 { 样式 }
 * 【常见子控件】drop-down, down-arrow, up-arrow, add-line, sub-line, handle, groove
 */
QComboBox::drop-down {
    border: none;                 /* 移除边框 */
    width: 20px;                  /* 下拉按钮宽度 */
}

QComboBox::down-arrow {
    image: url(:/icons/arrow-down.png);
    width: 12px;
    height: 12px;
}

QComboBox::down-arrow:hover {
    image: url(:/icons/arrow-down-hover.png);
}

/* ========== 6. 伪状态 ========== */
QPushButton:hover {
    background-color: #2980b9;
}

QPushButton:pressed {
    background-color: #1a5276;
    padding: 9px 15px 7px 17px;  /* 按下效果 */
}

QPushButton:disabled {
    background-color: #bdc3c7;
    color: #7f8c8d;
}

QLineEdit:focus {
    border: 2px solid #3498db;
}

QCheckBox:checked {
    color: #27ae60;
}

QCheckBox:unchecked {
    color: #7f8c8d;
}

QRadioButton:checked {
    color: #3498db;
}

/* 组合伪状态 */
QPushButton:hover:!pressed {
    background-color: #5dade2;
}

QCheckBox:hover:checked {
    color: #1e8449;
}

/* ========== 7. 伪元素 ========== */
QScrollBar:vertical {
    background: #f0f0f0;
    width: 12px;
    border-radius: 6px;
}

QScrollBar::handle:vertical {
    background: #bdc3c7;
    min-height: 30px;
    border-radius: 6px;
}

QScrollBar::handle:vertical:hover {
    background: #95a5a6;
}

QScrollBar::add-line:vertical,
QScrollBar::sub-line:vertical {
    height: 0px;  /* 隐藏上下箭头 */
}
```

### 5.3 常用控件样式示例

```css
/* ========== Button 样式 ========== */
QPushButton {
    background-color: #3498db;
    color: #ffffff;
    border: none;
    border-radius: 6px;
    padding: 10px 20px;
    font-size: 14px;
    font-weight: bold;
    min-width: 80px;
}

QPushButton:hover {
    background-color: #2980b9;
}

QPushButton:pressed {
    background-color: #21618c;
    padding: 11px 19px 9px 21px;
}

QPushButton:disabled {
    background-color: #bdc3c7;
    color: #7f8c8d;
}

/* 扁平按钮 */
QPushButton[flat="true"] {
    background-color: transparent;
    border: 1px solid #3498db;
    color: #3498db;
}

QPushButton[flat="true"]:hover {
    background-color: #3498db;
    color: white;
}

/* ========== LineEdit 样式 ========== */
QLineEdit {
    background-color: #ffffff;
    border: 2px solid #bdc3c7;
    border-radius: 4px;
    padding: 6px 10px;
    font-size: 14px;
    selection-background-color: #3498db;
}

QLineEdit:focus {
    border-color: #3498db;
}

QLineEdit:disabled {
    background-color: #ecf0f1;
    color: #95a5a6;
}

QLineEdit[readOnly="true"] {
    background-color: #f8f9fa;
    border-color: #dee2e6;
}

/* ========== ComboBox 样式 ========== */
QComboBox {
    background-color: #ffffff;
    border: 2px solid #bdc3c7;
    border-radius: 4px;
    padding: 6px 10px;
    min-width: 100px;
}

QComboBox:hover {
    border-color: #95a5a6;
}

QComboBox::drop-down {
    border: none;
    width: 24px;
}

QComboBox::down-arrow {
    image: url(:/icons/arrow-down.png);
}

QComboBox QAbstractItemView {
    background-color: #ffffff;
    border: 1px solid #bdc3c7;
    selection-background-color: #3498db;
    selection-color: #ffffff;
}

/* ========== Table 样式 ========== */
QTableView {
    background-color: #ffffff;
    border: 1px solid #bdc3c7;
    gridline-color: #ecf0f1;
    selection-background-color: #3498db;
    selection-color: #ffffff;
    alternate-background-color: #f8f9fa;
}

QTableView::item {
    padding: 6px;
    border: none;
}

QTableView::item:selected {
    background-color: #3498db;
    color: #ffffff;
}

QTableView::item:hover {
    background-color: #ebf5fb;
    color: #2c3e50;
}

QHeaderView::section {
    background-color: #ecf0f1;
    color: #2c3e50;
    padding: 8px;
    border: none;
    border-right: 1px solid #bdc3c7;
    border-bottom: 1px solid #bdc3c7;
    font-weight: bold;
}

QHeaderView::section:hover {
    background-color: #d5dbdb;
}

/* ========== Tab 样式 ========== */
QTabWidget::pane {
    border: 1px solid #bdc3c7;
    background: #ffffff;
    border-radius: 4px;
    top: -1px;
}

QTabBar::tab {
    background: #ecf0f1;
    color: #2c3e50;
    padding: 10px 20px;
    margin-right: 2px;
    border-top-left-radius: 4px;
    border-top-right-radius: 4px;
}

QTabBar::tab:selected {
    background: #ffffff;
    color: #3498db;
    border-bottom: 2px solid #3498db;
}

QTabBar::tab:hover:!selected {
    background: #d5dbdb;
}

QTabBar::tab:disabled {
    color: #bdc3c7;
}

/* ========== ScrollBar 样式 ========== */
QScrollBar:vertical {
    background: #f0f0f0;
    width: 14px;
    border-radius: 7px;
    margin: 0;
}

QScrollBar::handle:vertical {
    background: #bdc3c7;
    min-height: 30px;
    border-radius: 7px;
    margin: 2px;
}

QScrollBar::handle:vertical:hover {
    background: #95a5a6;
}

QScrollBar::handle:vertical:pressed {
    background: #7f8c8d;
}

QScrollBar::add-line:vertical,
QScrollBar::sub-line:vertical {
    height: 0;
    background: none;
}

QScrollBar:horizontal {
    background: #f0f0f0;
    height: 14px;
    border-radius: 7px;
}

QScrollBar::handle:horizontal {
    background: #bdc3c7;
    min-width: 30px;
    border-radius: 7px;
    margin: 2px;
}

/* ========== GroupBox 样式 ========== */
QGroupBox {
    background-color: #ffffff;
    border: 2px solid #3498db;
    border-radius: 8px;
    margin-top: 12px;
    padding: 12px;
    font-size: 14px;
    font-weight: bold;
}

QGroupBox::title {
    subcontrol-origin: margin;
    subcontrol-position: top left;
    padding: 4px 12px;
    background: #ffffff;
    color: #3498db;
}

/* ========== ProgressBar 样式 ========== */
QProgressBar {
    border: 2px solid #bdc3c7;
    border-radius: 8px;
    text-align: center;
    height: 25px;
    background: #ffffff;
    color: #2c3e50;
    font-weight: bold;
}

QProgressBar::chunk {
    background-color: #3498db;
    border-radius: 6px;
}

QProgressBar::chunk:disabled {
    background-color: #bdc3c7;
}

/* ========== Slider 样式 ========== */
QSlider::groove:horizontal {
    height: 8px;
    background: #ecf0f1;
    border-radius: 4px;
}

QSlider::handle:horizontal {
    background: #3498db;
    width: 20px;
    height: 20px;
    margin: -6px 0;
    border-radius: 10px;
}

QSlider::handle:horizontal:hover {
    background: #2980b9;
    width: 24px;
    height: 24px;
    margin: -8px 0;
}

QSlider::sub-page:horizontal {
    background: #3498db;
    border-radius: 4px;
}

/* ========== SpinBox 样式 ========== */
QSpinBox, QDoubleSpinBox {
    background-color: #ffffff;
    border: 2px solid #bdc3c7;
    border-radius: 4px;
    padding: 6px;
    selection-background-color: #3498db;
}

QSpinBox:focus, QDoubleSpinBox:focus {
    border-color: #3498db;
}

QSpinBox::up-button, QDoubleSpinBox::up-button,
QSpinBox::down-button, QDoubleSpinBox::down-button {
    background-color: #ecf0f1;
    border: none;
    width: 20px;
}

QSpinBox::up-button:hover, QDoubleSpinBox::up-button:hover,
QSpinBox::down-button:hover, QDoubleSpinBox::down-button:hover {
    background-color: #d5dbdb;
}

QSpinBox::up-arrow, QDoubleSpinBox::up-arrow {
    image: url(:/icons/arrow-up-small.png);
}

QSpinBox::down-arrow, QDoubleSpinBox::down-arrow {
    image: url(:/icons/arrow-down-small.png);
}
```

### 5.4 完整主题示例

```css
/* ========== 深色主题 ========== */
* {
    color: #ecf0f1;
}

QWidget {
    background-color: #2c3e50;
}

/* ---------- 窗口和对话框 ---------- */
QMainWindow, QDialog {
    background-color: #2c3e50;
}

/* ---------- 按钮 ---------- */
QPushButton {
    background-color: #34495e;
    border: 1px solid #4a5f7a;
    border-radius: 6px;
    padding: 8px 16px;
    min-width: 80px;
    font-weight: bold;
}

QPushButton:hover {
    background-color: #3d566e;
    border: 1px solid #5a7495;
}

QPushButton:pressed {
    background-color: #2c3e50;
}

QPushButton:checked {
    background-color: #3498db;
    border: 1px solid #5dade2;
}

QPushButton:disabled {
    background-color: #2c3e50;
    color: #7f8c8d;
}

/* ---------- 输入框 ---------- */
QLineEdit, QTextEdit, QPlainTextEdit {
    background-color: #34495e;
    border: 1px solid #4a5f7a;
    border-radius: 4px;
    padding: 6px 10px;
    selection-background-color: #3498db;
}

QLineEdit:focus, QTextEdit:focus, QPlainTextEdit:focus {
    border: 1px solid #3498db;
}

QLineEdit:disabled, QTextEdit:disabled, QPlainTextEdit:disabled {
    background-color: #2c3e50;
    color: #7f8c8d;
}

/* ---------- 列表/树/表格 ---------- */
QListView, QTreeView, QTableView {
    background-color: #34495e;
    border: 1px solid #4a5f7a;
    selection-background-color: #3498db;
    alternate-background-color: #3d566e;
}

QListView::item:selected, QTreeView::item:selected, QTableView::item:selected {
    background-color: #3498db;
    color: #ffffff;
}

QListView::item:hover, QTreeView::item:hover, QTableView::item:hover {
    background-color: #3d566e;
}

QHeaderView::section {
    background-color: #2c3e50;
    color: #ecf0f1;
    border: none;
    border-bottom: 1px solid #4a5f7a;
    border-right: 1px solid #4a5f7a;
    padding: 8px;
    font-weight: bold;
}

QHeaderView::section:hover {
    background-color: #34495e;
}

/* ---------- 滚动条 ---------- */
QScrollBar:vertical, QScrollBar:horizontal {
    background: #2c3e50;
    border: none;
}

QScrollBar::handle:vertical, QScrollBar::handle:horizontal {
    background: #5a6e7f;
    border-radius: 4px;
}

QScrollBar::handle:vertical:hover, QScrollBar::handle:horizontal:hover {
    background: #6a7e8f;
}

QScrollBar::add-line:vertical, QScrollBar::sub-line:vertical,
QScrollBar::add-line:horizontal, QScrollBar::sub-line:horizontal {
    border: none;
    background: none;
}

/* ---------- 菜单 ---------- */
QMenuBar {
    background-color: #2c3e50;
    border-bottom: 1px solid #4a5f7a;
}

QMenuBar::item {
    background-color: transparent;
    padding: 8px 12px;
}

QMenuBar::item:selected {
    background-color: #34495e;
}

QMenu {
    background-color: #34495e;
    border: 1px solid #4a5f7a;
}

QMenu::item {
    padding: 8px 24px;
}

QMenu::item:selected {
    background-color: #3498db;
}

QMenu::separator {
    height: 1px;
    background: #4a5f7a;
    margin: 4px 8px;
}

/* ---------- 标签页 ---------- */
QTabWidget::pane {
    border: 1px solid #4a5f7a;
    background: #34495e;
}

QTabBar::tab {
    background: #2c3e50;
    color: #bdc3c7;
    padding: 10px 20px;
    margin-right: 2px;
    border-top-left-radius: 4px;
    border-top-right-radius: 4px;
}

QTabBar::tab:selected {
    background: #3498db;
    color: #ffffff;
}

QTabBar::tab:hover:!selected {
    background: #34495e;
    color: #ecf0f1;
}

/* ---------- 组合框 ---------- */
QGroupBox {
    border: 1px solid #4a5f7a;
    border-radius: 6px;
    margin-top: 12px;
    padding: 12px;
    font-weight: bold;
}

QGroupBox::title {
    subcontrol-origin: margin;
    subcontrol-position: top left;
    padding: 4px 12px;
    background: #2c3e50;
}

/* ---------- 进度条 ---------- */
QProgressBar {
    border: 1px solid #4a5f7a;
    border-radius: 6px;
    text-align: center;
    height: 25px;
    background: #34495e;
}

QProgressBar::chunk {
    background-color: #3498db;
    border-radius: 4px;
}

/* ---------- 滑块 ---------- */
QSlider::groove:horizontal {
    height: 6px;
    background: #4a5f7a;
    border-radius: 3px;
}

QSlider::handle:horizontal {
    background: #3498db;
    width: 18px;
    height: 18px;
    margin: -6px 0;
    border-radius: 9px;
}

QSlider::handle:horizontal:hover {
    background: #5dade2;
    width: 22px;
    height: 22px;
    margin: -8px 0;
}

QSlider::sub-page:horizontal {
    background: #3498db;
    border-radius: 3px;
}

/* ---------- 下拉框 ---------- */
QComboBox {
    background-color: #34495e;
    border: 1px solid #4a5f7a;
    border-radius: 4px;
    padding: 6px 10px;
}

QComboBox:hover {
    border: 1px solid #5a7495;
}

QComboBox::drop-down {
    border: none;
}

QComboBox QAbstractItemView {
    background-color: #34495e;
    border: 1px solid #4a5f7a;
    selection-background-color: #3498db;
}

/* ---------- 复选框/单选框 ---------- */
QCheckBox, QRadioButton {
    spacing: 8px;
}

QCheckBox::indicator, QRadioButton::indicator {
    width: 18px;
    height: 18px;
    border: 2px solid #4a5f7a;
    border-radius: 3px;
    background: #34495e;
}

QCheckBox::indicator:checked, QRadioButton::indicator:checked {
    background: #3498db;
    border-color: #3498db;
}

QCheckBox::indicator:checked {
    image: url(:/icons/check.png);
}

QRadioButton::indicator {
    border-radius: 9px;
}

QRadioButton::indicator:checked {
    image: url(:/icons/radio-check.png);
}

/* ---------- 工具提示 ---------- */
QToolTip {
    background-color: #34495e;
    color: #ecf0f1;
    border: 1px solid #4a5f7a;
    padding: 6px;
    border-radius: 4px;
}

/* ---------- 状态栏 ---------- */
QStatusBar {
    background-color: #2c3e50;
    border-top: 1px solid #4a5f7a;
}

QStatusBar::item {
    border: none;
}

QStatusBar QLabel {
    color: #bdc3c7;
}
```

---

## 六、多线程

// 【线程安全注意事项】
// 1. UI 操作只能在主线程（GUI 线程）进行
// 2. 跨线程访问共享数据必须加锁（QMutex、QReadWriteLock）
// 3. 跨线程信号槽默认使用 QueuedConnection（异步）
// 4. 不要在子线程中创建 QWidget 及其子类
// 5. 线程结束后务必调用 quit() 和 wait()

### 6.1 QThread 基本用法

```cpp
// ========== 方式1：继承 QThread（不推荐）==========
// 【缺点】
// - run() 执行完后线程对象才真正工作
// - 灵活性差，不适合复杂场景
// 【适用场景】简单的独立任务

class WorkerThread : public QThread {
    Q_OBJECT
public:
    explicit WorkerThread(QObject* parent = nullptr) : QThread(parent) {}

    // 【线程停止方法】
    void stop() {
        // requestInterruption(): 设置中断标志位
        // 不会强制终止线程，而是让 isInterruptionRequested() 返回 true
        requestInterruption();
        // wait(): 阻塞等待线程结束
        // 参数：超时时间（毫秒），默认 ULONG_MAX（无限等待）
        wait();  // 等待线程结束
    }

protected:
    // 【run() 函数】线程入口函数
    // 在新线程中执行，返回后线程结束
    void run() override {
        // 这里是子线程的代码
        // isInterruptionRequested(): 检查是否请求中断
        // 【最佳实践】在循环中定期检查此标志
        for (int i = 0; i < 100 && !isInterruptionRequested(); ++i) {
            // emit: 发送信号
            // 跨线程发送信号时，会自动使用 QueuedConnection
            emit progressChanged(i);

            // msleep(): 毫秒级睡眠（静态函数）
            // 模拟耗时操作
            msleep(100);
        }
        emit workFinished();
    }

signals:
    void progressChanged(int value);
    void workFinished();
};

// 【使用示例】
WorkerThread* thread = new WorkerThread();
// 连接信号槽
// 默认使用 AutoConnection，跨线程时变为 QueuedConnection
// progressChanged 在子线程发射，在主线程处理
connect(thread, &WorkerThread::progressChanged, [](int value) {
    qDebug() << "Progress:" << value;  // 在主线程执行
});
connect(thread, &WorkerThread::workFinished, []() {
    qDebug() << "Work finished!";
});
connect(thread, &WorkerThread::finished, thread, &WorkerThread::deleteLater);

thread->start();

// 停止线程
// thread->stop();
```

### 6.2 moveToThread 模式（推荐）

```cpp
// ========== 方式2：moveToThread（推荐）==========

class Worker : public QObject {
    Q_OBJECT
public:
    explicit Worker(QObject* parent = nullptr) : QObject(parent) {}

public slots:
    void doWork() {
        qDebug() << "Worker thread:" << QThread::currentThread();

        // 执行耗时操作
        QString result;
        for (int i = 0; i < 100; ++i) {
            if (QThread::currentThread()->isInterruptionRequested()) {
                emit workCancelled();
                return;
            }

            result.append(QString::number(i) + " ");
            emit progress(i);

            QThread::msleep(50);  // 模拟耗时
        }

        emit workFinished(result);
    }

    void stopWork() {
        qDebug() << "Stop requested";
        QThread::currentThread()->requestInterruption();
    }

signals:
    void progress(int value);
    void workFinished(const QString& result);
    void workCancelled();
};

// ========== 使用 moveToThread ==========

void exampleMoveToThread() {
    // 创建线程
    QThread* thread = new QThread();
    thread->setObjectName("WorkerThread");

    // 创建 Worker 对象
    Worker* worker = new Worker();

    // 将 Worker 移动到新线程
    worker->moveToThread(thread);

    // 连接信号槽
    // 1. 线程启动时执行工作
    connect(thread, &QThread::started, worker, &Worker::doWork);

    // 2. 工作完成后停止线程
    connect(worker, &Worker::workFinished, [thread](const QString& result) {
        qDebug() << "Result:" << result;
        thread->quit();  // 退出事件循环
    });

    connect(worker, &Worker::workCancelled, thread, &QThread::quit);

    // 3. 线程结束后清理
    connect(thread, &QThread::finished, worker, &Worker::deleteLater);
    connect(thread, &QThread::finished, thread, &QThread::deleteLater);

    // 4. 进度更新（跨线程自动使用队列连接）
    connect(worker, &Worker::progress, [](int value) {
        qDebug() << "Progress:" << value;
    });

    // 启动线程
    thread->start();
    qDebug() << "Main thread:" << QThread::currentThread();
}

// ========== 注意事项 ==========
/*
 * 1. Worker 对象在创建时不能有父对象
 * 2. Worker 中的槽函数都在子线程中执行
 * 3. 跨线程信号槽默认使用 QueuedConnection
 * 4. 不要在子线程中操作 UI（UI 只能在主线程）
 * 5. 线程结束后必须调用 quit() 退出事件循环
 */
```

### 6.3 线程同步机制

```cpp
// ========== QMutex 互斥锁 ==========

class SharedData {
public:
    void addData(const QString& data) {
        QMutexLocker locker(&m_mutex);  // 自动加锁/解锁
        m_data.append(data);
    }

    QStringList getData() {
        QMutexLocker locker(&m_mutex);
        return m_data;
    }

private:
    QMutex m_mutex;
    QStringList m_data;
};

// 手动加锁方式
void manualLock() {
    QMutex mutex;

    mutex.lock();
    // 临界区代码
    mutex.unlock();

    // 或使用 tryLock
    if (mutex.tryLock(1000)) {  // 尝试锁定1秒
        // 成功获取锁
        mutex.unlock();
    }
}

// ========== QReadWriteLock 读写锁 ==========

class DataStore {
public:
    void write(const QString& data) {
        QWriteLocker locker(&m_lock);  // 写锁
        m_data = data;
    }

    QString read() const {
        QReadLocker locker(&m_lock);  // 读锁
        return m_data;
    }

private:
    mutable QReadWriteLock m_lock;
    QString m_data;
};

// ========== QSemaphore 信号量 ==========

class ProducerConsumer {
public:
    void producer() {
        forever {
            // 生产数据
            m_mutex.lock();
            m_data.append("item");
            qDebug() << "Produced, count:" << m_data.size();
            m_mutex.unlock();

            m_available.release();  // 增加可用资源
            QThread::msleep(100);
        }
    }

    void consumer() {
        forever {
            m_available.acquire();  // 等待可用资源

            m_mutex.lock();
            if (!m_data.isEmpty()) {
                QString item = m_data.takeFirst();
                qDebug() << "Consumed:" << item << ", count:" << m_data.size();
            }
            m_mutex.unlock();
        }
    }

private:
    QSemaphore m_available = 1;  // 初始有1个资源
    QMutex m_mutex;
    QStringList m_data;
};

// ========== QWaitCondition 条件变量 ==========

class ThreadPool {
public:
    void addTask(const QString& task) {
        QMutexLocker locker(&m_mutex);
        m_tasks.append(task);
        m_condition.wakeOne();  // 唤醒一个等待的线程
    }

    QString waitForTask() {
        QMutexLocker locker(&m_mutex);
        while (m_tasks.isEmpty()) {
            m_condition.wait(&m_mutex);  // 等待条件
        }
        return m_tasks.takeFirst();
    }

private:
    QMutex m_mutex;
    QWaitCondition m_condition;
    QStringList m_tasks;
};

// ========== QAtomicInt 原子操作 ==========

QAtomicInt counter = 0;

void atomicIncrement() {
    // 原子递增，无需加锁
    counter.fetchAndAddRelaxed(1);

    // 或使用
    int value = counter.load();
    counter.store(value + 1);
}

// ========== Qt Concurrent（高级接口）==========

#include <QtConcurrent>

void exampleQtConcurrent() {
    // 1. 在线程中运行函数
    QFuture<int> future = QtConcurrent::run([]() {
        int sum = 0;
        for (int i = 0; i < 1000; ++i) {
            sum += i;
        }
        return sum;
    });

    // 等待结果
    future.waitForFinished();
    qDebug() << "Sum:" << future.result();

    // 2. 并行处理容器
    QList<int> numbers;
    for (int i = 0; i < 100; ++i) {
        numbers.append(i);
    }

    QFuture<void> future2 = QtConcurrent::map(numbers, [](int& value) {
        value = value * value;  // 每个元素平方
    });
    future2.waitForFinished();

    // 3. 并行过滤
    QFuture<int> future3 = QtConcurrent::filtered(numbers, [](int value) {
        return value > 50;  // 保留大于50的
    });

    // 4. 使用 QFutureWatcher 监控
    QFutureWatcher<int>* watcher = new QFutureWatcher<int>();
    connect(watcher, &QFutureWatcher<int>::finished, [watcher]() {
        qDebug() << "Done, result:" << watcher->result();
        watcher->deleteLater();
    });

    watcher->setFuture(future);
}
```

---

## 七、文件与配置

// 【路径处理注意事项】
// 1. 使用 QDir/QFileInfo 处理路径，自动适配平台
// 2. 路径分隔符：Windows 用 "\"，Unix 用 "/"
//    - Qt::DirSeparator 或 "/" 可自动适配
// 3. 相对路径 vs 绝对路径：
//    - 相对路径：相对于程序工作目录
//    - 绝对路径：完整路径，更可靠
// 4. 路径拼接使用 "/" 或 QDir::separator()
// 5. 编码问题：Windows 文件名可能包含非 ASCII 字符

### 7.1 QFile 文件读写

```cpp
// ========== 路径处理（跨平台）==========

// 【获取标准路径】
void getStandardPaths() {
    // 应用程序目录
    QString appPath = QCoreApplication::applicationDirPath();
    // Windows: "C:/MyApp"
    // Linux: "/usr/bin"

    // 当前工作目录
    QString currentPath = QDir::currentPath();

    // 用户主目录
    QString homePath = QDir::homePath();
    // Windows: "C:/Users/Username"
    // Linux: "/home/username"

    // 临时目录
    QString tempPath = QDir::tempPath();

    // 【应用程序数据目录】
    // QStandardPaths::AppDataLocation: 跨平台应用数据目录
    QString dataPath = QStandardPaths::writableLocation(QStandardPaths::AppDataLocation);
    // Windows: "C:/Users/Username/AppData/Roaming/MyApp"
    // macOS: "~/Library/Application Support/MyApp"
    // Linux: "~/.config/MyApp"
}

// 【路径拼接】
void pathJoining() {
    // 方式1：使用 "/" 自动适配
    QString fullPath1 = QDir::homePath() + "/Documents/file.txt";

    // 方式2：使用 QDir::filePath()
    QDir dir(QDir::homePath());
    QString fullPath2 = dir.filePath("Documents/file.txt");

    // 方式3：使用 QFileInfo（推荐）
    QFileInfo fileInfo(QDir::homePath(), "Documents/file.txt");
    QString fullPath3 = fileInfo.absoluteFilePath();
}

// ========== 文本文件读写 ==========

// 写入文本文件
void writeTextFile() {
    // 【QFile 构造】可以传入绝对路径或相对路径
    // 相对路径相对于程序工作目录
    QFile file("output.txt");
    // 或者使用绝对路径
    // QFile file(QDir::homePath() + "/output.txt");

    // 【open() 打开文件】
    // QIODevice::OpenModeFlag 枚举：
    //   - ReadOnly: 只读
    //   - WriteOnly: 只写
    //   - ReadWrite: 读写
    //   - Append: 追加
    //   - Truncate: 清空文件
    //   - Text: 文本模式（转换换行符）
    if (file.open(QIODevice::WriteOnly | QIODevice::Text)) {
        // QTextStream: 文本流，支持格式化输出
        QTextStream out(&file);

        // 【编码设置】
        // Qt6: setEncoding(QStringConverter::Utf8)
        // Qt5: setCodec("UTF-8")
        // 常用编码：UTF-8, GBK, System
        out.setEncoding(QStringConverter::Utf8);

        // 【写入数据】
        out << "Hello, Qt!" << Qt::endl;  // Qt::endl: 跨平台换行
        out << "中文内容" << Qt::endl;
        out << 12345 << Qt::endl;

        file.close();  // 【关闭文件】析构时自动关闭，显式关闭更安全
    } else {
        qDebug() << "Failed to open file:" << file.errorString();
    }
}

// 读取文本文件
void readTextFile() {
    QFile file("input.txt");
    if (file.open(QIODevice::ReadOnly | QIODevice::Text)) {
        QTextStream in(&file);
        in.setEncoding(QStringConverter::Utf8);

        // 【按行读取】
        // atEnd(): 是否到达文件末尾
        while (!in.atEnd()) {
            QString line = in.readLine();  // readLine(): 读取一行
            qDebug() << line;
        }

        // 【一次性读取全部】
        // file.readAll(): 返回 QByteArray
        // QString::fromUtf8(): 转换为 QString
        // QString allContent = QString::fromUtf8(file.readAll());

        file.close();
    }
}

// ========== 二进制文件读写 ==========

void writeBinaryFile() {
    QFile file("data.bin");
    // 二进制文件不需要 Text 标志
    if (file.open(QIODevice::WriteOnly)) {
        // QDataStream: 二进制数据流
        // 自动处理字节序（大端/小端）
        QDataStream out(&file);

        // 【写入数据】
        // QDataStream 支持大多数 Qt 类型
        out << QString("Hello");           // 字符串
        out << (qint32)42;                 // 整数（指定大小）
        out << 3.14;                       // 浮点数
        out << QByteArray("Binary data");  // 字节数组

        file.close();
    }
}

void readBinaryFile() {
    QFile file("data.bin");
    if (file.open(QIODevice::ReadOnly)) {
        QDataStream in(&file);

        QString str;
        int number;
        double pi;
        QByteArray data;

        in >> str >> number >> pi >> data;

        qDebug() << str << number << pi << data;

        file.close();
    }
}

// ========== QFileInfo 文件信息 ==========

void getFileInfo() {
    QFileInfo fileInfo("example.txt");

    qDebug() << "Exists:" << fileInfo.exists();
    qDebug() << "Size:" << fileInfo.size() << "bytes";
    qDebug() << "Created:" << fileInfo.birthTime();
    qDebug() << "Modified:" << fileInfo.lastModified();
    qDebug() << "Readable:" << fileInfo.isReadable();
    qDebug() << "Writable:" << fileInfo.isWritable();
    qDebug() << "Absolute path:" << fileInfo.absoluteFilePath();
    qDebug() << "Suffix:" << fileInfo.suffix();
    qDebug() << "Base name:" << fileInfo.baseName();
}

// ========== QDir 目录操作 ==========

void directoryOperations() {
    // 创建目录
    QDir dir;
    if (!dir.exists("myfolder")) {
        dir.mkpath("myfolder/subfolder");  // 创建多级目录
    }

    // 遍历目录
    QDir folder("myfolder");
    QStringList filters;
    filters << "*.txt" << "*.md";
    folder.setNameFilters(filters);

    QFileInfoList fileList = folder.entryInfoList(QDir::Files | QDir::NoDotAndDotDot);
    for (const QFileInfo& fileInfo : fileList) {
        qDebug() << fileInfo.fileName() << fileInfo.size();
    }

    // 删除文件
    folder.remove("oldfile.txt");

    // 重命名
    folder.rename("old.txt", "new.txt");
}
```

### 7.2 QSettings 配置文件

```cpp
// ========== QSettings 基本用法 ==========

void settingsExample() {
    // 创建 Settings 对象（自动选择平台）
    // Windows: 注册表
    // macOS: plist 文件
    // Linux: conf 文件
    QSettings settings("MyCompany", "MyApp");

    // ========== 写入设置 ==========
    settings.setValue("window/width", 800);
    settings.setValue("window/height", 600);
    settings.setValue("window/x", 100);
    settings.setValue("window/y", 100);
    settings.setValue("theme", "dark");
    settings.setValue("language", "zh_CN");

    // 写入数组
    QStringList recentFiles;
    recentFiles << "file1.txt" << "file2.txt" << "file3.txt";
    settings.setValue("recentFiles", recentFiles);

    // 写入自定义类型
    QVariantMap customData;
    customData["key1"] = "value1";
    customData["key2"] = 42;
    settings.setValue("custom", customData);

    // ========== 读取设置 ==========
    int width = settings.value("window/width", 1024).toInt();  // 默认值1024
    int height = settings.value("window/height", 768).toInt();
    QString theme = settings.value("theme", "light").toString();
    QString language = settings.value("language", "en_US").toString();

    qDebug() << "Window size:" << width << "x" << height;
    qDebug() << "Theme:" << theme;
    qDebug() << "Language:" << language;

    // 读取数组
    QStringList files = settings.value("recentFiles").toStringList();
    qDebug() << "Recent files:" << files;

    // ========== 组设置 ==========
    settings.beginGroup("network");
    settings.setValue("host", "192.168.1.1");
    settings.setValue("port", 8080);
    settings.setValue("timeout", 30000);
    settings.endGroup();

    // 读取组设置
    settings.beginGroup("network");
    QString host = settings.value("host", "localhost").toString();
    int port = settings.value("port", 80).toInt();
    settings.endGroup();

    // ========== 删除设置 ==========
    settings.remove("theme");  // 删除单个值
    settings.beginGroup("window");
    settings.remove("");  // 删除整个组
    settings.endGroup();

    // ========== 清除所有设置 ==========
    settings.clear();

    // ========== 使用 INI 文件格式 ==========

    QSettings iniSettings("config.ini", QSettings::IniFormat);
    iniSettings.setValue("section1/key1", "value1");
    iniSettings.setValue("section1/key2", "value2");
    iniSettings.setValue("section2/key1", "value3");

    // INI 文件内容示例：
    // [section1]
    // key1=value1
    // key2=value2
    //
    // [section2]
    // key1=value3
}

// ========== 窗口状态保存/恢复 ==========

class MainWindow : public QMainWindow {
    Q_OBJECT
public:
    MainWindow(QWidget* parent = nullptr) : QMainWindow(parent) {
        loadSettings();
    }

    ~MainWindow() {
        saveSettings();
    }

private:
    void saveSettings() {
        QSettings settings("MyCompany", "MyApp");

        settings.setValue("geometry", saveGeometry());
        settings.setValue("windowState", saveState());
        settings.setValue("splitter", m_splitter->saveState());
    }

    void loadSettings() {
        QSettings settings("MyCompany", "MyApp");

        restoreGeometry(settings.value("geometry").toByteArray());
        restoreState(settings.value("windowState").toByteArray());
        m_splitter->restoreState(settings.value("splitter").toByteArray());
    }

    QSplitter* m_splitter;
};
```

### 7.3 JSON/XML 解析

```cpp
// ========== JSON 解析（QJsonDocument）==========

#include <QJsonDocument>
#include <QJsonObject>
#include <QJsonArray>

void jsonExample() {
    // ========== 1. 解析 JSON ==========
    QString jsonString = R"(
        {
            "name": "Qt Application",
            "version": "1.0.0",
            "settings": {
                "theme": "dark",
                "language": "zh_CN"
            },
            "users": [
                {"id": 1, "name": "Alice"},
                {"id": 2, "name": "Bob"},
                {"id": 3, "name": "Charlie"}
            ]
        }
    )";

    QJsonParseError error;
    QJsonDocument doc = QJsonDocument::fromJson(jsonString.toUtf8(), &error);

    if (error.error != QJsonParseError::NoError) {
        qDebug() << "JSON parse error:" << error.errorString();
        return;
    }

    QJsonObject root = doc.object();

    // 读取基本值
    QString name = root["name"].toString();
    QString version = root["version"].toString();
    qDebug() << "Name:" << name << "Version:" << version;

    // 读取嵌套对象
    QJsonObject settings = root["settings"].toObject();
    QString theme = settings["theme"].toString();
    QString language = settings["language"].toString();

    // 读取数组
    QJsonArray users = root["users"].toArray();
    for (const QJsonValue& value : users) {
        QJsonObject user = value.toObject();
        int id = user["id"].toInt();
        QString userName = user["name"].toString();
        qDebug() << "User:" << id << userName;
    }

    // ========== 2. 生成 JSON ==========
    QJsonObject newRoot;
    newRoot["name"] = "MyApp";
    newRoot["version"] = "2.0.0";

    QJsonObject newSettings;
    newSettings["theme"] = "light";
    newSettings["language"] = "en_US";
    newRoot["settings"] = newSettings;

    QJsonArray newUsers;
    for (int i = 1; i <= 3; ++i) {
        QJsonObject user;
        user["id"] = i;
        user["name"] = QString("User%1").arg(i);
        newUsers.append(user);
    }
    newRoot["users"] = newUsers;

    QJsonDocument newDoc(newRoot);
    QString jsonOutput = newDoc.toJson(QJsonDocument::Indented);
    qDebug() << "Generated JSON:\n" << jsonOutput;
}

// ========== XML 解析（QDomDocument）==========

#include <QDomDocument>

void xmlExample() {
    // ========== 解析 XML ==========
    QString xmlString = R"(
        <config>
            <application>
                <name>MyApp</name>
                <version>1.0</version>
            </application>
            <settings>
                <setting key="theme" value="dark"/>
                <setting key="language" value="zh_CN"/>
            </settings>
            <users>
                <user id="1" name="Alice"/>
                <user id="2" name="Bob"/>
            </users>
        </config>
    )";

    QDomDocument doc;
    QString errorMsg;
    int errorLine, errorColumn;

    if (!doc.setContent(xmlString, &errorMsg, &errorLine, &errorColumn)) {
        qDebug() << "XML parse error:" << errorMsg << "Line:" << errorLine;
        return;
    }

    QDomElement root = doc.documentElement();

    // 读取元素
    QDomNodeList appNodes = root.elementsByTagName("application");
    if (!appNodes.isEmpty()) {
        QDomElement app = appNodes.at(0).toElement();
        QString name = app.firstChildElement("name").text();
        QString version = app.firstChildElement("version").text();
        qDebug() << "App:" << name << version;
    }

    // 读取属性
    QDomNodeList settingNodes = root.elementsByTagName("setting");
    for (int i = 0; i < settingNodes.size(); ++i) {
        QDomElement setting = settingNodes.at(i).toElement();
        QString key = setting.attribute("key");
        QString value = setting.attribute("value");
        qDebug() << "Setting:" << key << "=" << value;
    }

    // ========== 生成 XML ==========
    QDomDocument newDoc;
    QDomElement newRoot = newDoc.createElement("config");

    QDomElement app = newDoc.createElement("application");
    app.appendChild(newDoc.createElement("name")).appendChild(newDoc.createTextNode("NewApp"));
    app.appendChild(newDoc.createElement("version")).appendChild(newDoc.createTextNode("2.0"));
    newRoot.appendChild(app);

    newDoc.appendChild(newRoot);

    QString xmlOutput = newDoc.toString(4);  // 缩进4空格
    qDebug() << "Generated XML:\n" << xmlOutput;
}
```

---

## 八、布局管理

### 8.1 基本布局

```cpp
// ========== QVBoxLayout 垂直布局 ==========

QWidget* widget = new QWidget();
QVBoxLayout* vLayout = new QVBoxLayout(widget);

// 添加控件
vLayout->addWidget(new QPushButton("Button 1"));
vLayout->addWidget(new QPushButton("Button 2"));
vLayout->addWidget(new QPushButton("Button 3"));

// 设置间距
vLayout->setSpacing(10);  // 控件间距
vLayout->setContentsMargins(20, 20, 20, 20);  // 边距

// 添加弹性空间（会自动扩展）
vLayout->addStretch();  // 在末尾添加弹簧

// ========== QHBoxLayout 水平布局 ==========

QHBoxLayout* hLayout = new QHBoxLayout();
hLayout->addWidget(new QLineEdit());
hLayout->addWidget(new QPushButton("Search"));

// 添加固定间距
hLayout->addSpacing(20);

hLayout->addWidget(new QPushButton("Cancel"));

// ========== QGridLayout 网格布局 ==========

QGridLayout* gridLayout = new QGridLayout();

// 添加控件到网格（行, 列, 行跨度, 列跨度）
gridLayout->addWidget(new QLabel("Name:"), 0, 0);
gridLayout->addWidget(new QLineEdit(), 0, 1);

gridLayout->addWidget(new QLabel("Age:"), 1, 0);
gridLayout->addWidget(new QSpinBox(), 1, 1);

gridLayout->addWidget(new QLabel("Address:"), 2, 0);
gridLayout->addWidget(new QTextEdit(), 2, 1, 3, 1);  // 跨3行1列

gridLayout->addWidget(new QLabel("City:"), 2, 2);
gridLayout->addWidget(new QLineEdit(), 2, 3);

gridLayout->addWidget(new QLabel("Country:"), 3, 2);
gridLayout->addWidget(new QLineEdit(), 3, 3);

// 设置列宽比例
gridLayout->setColumnStretch(1, 2);  // 第1列权重为2
gridLayout->setColumnStretch(3, 1);  // 第3列权重为1

// 设置行高比例
gridLayout->setRowStretch(4, 1);  // 最后一行可扩展

// ========== QFormLayout 表单布局 ==========

QFormLayout* formLayout = new QFormLayout();

formLayout->addRow("Name:", new QLineEdit());
formLayout->addRow("Email:", new QLineEdit());
formLayout->addRow("Password:", new QLineEdit());
formLayout->addRow("Confirm:", new QLineEdit());

// 添加自定义行
formLayout->addRow("Agree:", new QCheckBox("I accept the terms"));

// 设置标签对齐方式
formLayout->setLabelAlignment(Qt::AlignRight);

// 设置表单换行策略
formLayout->setFormAlignment(Qt::AlignLeft | Qt::AlignTop);
```

### 8.2 嵌套布局与 Spacer

```cpp
// ========== 嵌套布局示例 ==========

QWidget* window = new QWidget();
QVBoxLayout* mainLayout = new QVBoxLayout(window);

// 顶部：水平布局
QHBoxLayout* topLayout = new QHBoxLayout();
topLayout->addWidget(new QLabel("Search:"));
topLayout->addWidget(new QLineEdit());
topLayout->addWidget(new QPushButton("Go"));
mainLayout->addLayout(topLayout);

// 中间：网格布局
QGridLayout* gridLayout = new QGridLayout();
for (int row = 0; row < 3; ++row) {
    for (int col = 0; col < 3; ++col) {
        gridLayout->addWidget(new QPushButton(QString("Button %1").arg(row * 3 + col)), row, col);
    }
}
mainLayout->addLayout(gridLayout);

// 中间添加弹性空间
mainLayout->addStretch(1);

// 底部：水平布局
QHBoxLayout* bottomLayout = new QHBoxLayout();
bottomLayout->addStretch();  // 左边弹簧
bottomLayout->addWidget(new QPushButton("OK"));
bottomLayout->addWidget(new QPushButton("Cancel"));
mainLayout->addLayout(bottomLayout);

// ========== Spacer（弹簧）==========

QVBoxLayout* layout = new QVBoxLayout();

layout->addWidget(new QLabel("Top"));

// 添加固定高度的弹簧
layout->addSpacing(50);

layout->addWidget(new QLabel("Middle"));

// 添加可扩展的弹簧（stretch = 1）
layout->addStretch(1);

layout->addWidget(new QLabel("Bottom"));

// ========== QSplitter（可调整大小的分割器）==========

QSplitter* splitter = new QSplitter(Qt::Horizontal);

// 添加控件
QTextEdit* leftEdit = new QTextEdit();
leftEdit->setPlainText("Left Panel");
splitter->addWidget(leftEdit);

QTextEdit* rightEdit = new QTextEdit();
rightEdit->setPlainText("Right Panel");
splitter->addWidget(rightEdit);

// 设置初始比例
splitter->setStretchFactor(0, 1);  // 左侧权重1
splitter->setStretchFactor(1, 2);  // 右侧权重2

// 设置最小尺寸
leftEdit->setMinimumWidth(100);
rightEdit->setMinimumWidth(100);

// 可折叠
splitter->setChildrenCollapsible(true);

// 垂直分割器
QSplitter* vSplitter = new QSplitter(Qt::Vertical);
vSplitter->addWidget(new QTextEdit());
vSplitter->addWidget(new QTextEdit());
```

### 8.3 响应式布局技巧

```cpp
// ========== 响应式布局 ==========

class ResponsiveWidget : public QWidget {
public:
    ResponsiveWidget(QWidget* parent = nullptr) : QWidget(parent) {
        setupUI();
    }

private:
    void setupUI() {
        QVBoxLayout* mainLayout = new QVBoxLayout(this);

        // 顶部工具栏
        QHBoxLayout* toolbarLayout = new QHBoxLayout();
        toolbarLayout->addWidget(new QPushButton("New"));
        toolbarLayout->addWidget(new QPushButton("Open"));
        toolbarLayout->addWidget(new QPushButton("Save"));
        toolbarLayout->addStretch();
        toolbarLayout->addWidget(new QLabel("Status: Ready"));
        mainLayout->addLayout(toolbarLayout);

        // 中间内容区
        QSplitter* splitter = new QSplitter(Qt::Horizontal);

        // 左侧面板
        QWidget* leftPanel = createLeftPanel();
        splitter->addWidget(leftPanel);

        // 右侧面板
        QWidget* rightPanel = createRightPanel();
        splitter->addWidget(rightPanel);

        mainLayout->addWidget(splitter);

        // 底部状态栏
        QHBoxLayout* statusLayout = new QHBoxLayout();
        statusLayout->addWidget(new QLabel("Line: 1, Col: 1"));
        statusLayout->addStretch();
        statusLayout->addWidget(new QLabel("UTF-8"));
        mainLayout->addLayout(statusLayout);
    }

    QWidget* createLeftPanel() {
        QWidget* panel = new QWidget();
        QVBoxLayout* layout = new QVBoxLayout(panel);

        QListWidget* listWidget = new QListWidget();
        listWidget->addItem("Item 1");
        listWidget->addItem("Item 2");
        listWidget->addItem("Item 3");
        layout->addWidget(listWidget);

        return panel;
    }

    QWidget* createRightPanel() {
        QWidget* panel = new QWidget();
        QVBoxLayout* layout = new QVBoxLayout(panel);

        QTextEdit* textEdit = new QTextEdit();
        textEdit->setPlaceholderText("Enter text here...");
        layout->addWidget(textEdit);

        return panel;
    }
};

// ========== 动态调整布局 ==========

class DynamicLayoutWidget : public QWidget {
    Q_OBJECT
public:
    DynamicLayoutWidget(QWidget* parent = nullptr) : QWidget(parent) {
        QVBoxLayout* layout = new QVBoxLayout(this);

        m_stackedWidget = new QStackedWidget();
        layout->addWidget(m_stackedWidget);

        // 添加不同视图
        m_stackedWidget->addWidget(createSmallView());
        m_stackedWidget->addWidget(createLargeView());

        // 默认显示小视图
        m_stackedWidget->setCurrentIndex(0);
    }

protected:
    void resizeEvent(QResizeEvent* event) override {
        // 根据窗口大小切换视图
        if (width() < 600) {
            m_stackedWidget->setCurrentIndex(0);  // 小视图
        } else {
            m_stackedWidget->setCurrentIndex(1);  // 大视图
        }
        QWidget::resizeEvent(event);
    }

private:
    QWidget* createSmallView() {
        QWidget* view = new QWidget();
        QVBoxLayout* layout = new QVBoxLayout(view);
        layout->addWidget(new QPushButton("Vertical Layout 1"));
        layout->addWidget(new QPushButton("Vertical Layout 2"));
        layout->addWidget(new QPushButton("Vertical Layout 3"));
        return view;
    }

    QWidget* createLargeView() {
        QWidget* view = new QWidget();
        QHBoxLayout* layout = new QHBoxLayout(view);
        layout->addWidget(new QPushButton("Horizontal Layout 1"));
        layout->addWidget(new QPushButton("Horizontal Layout 2"));
        layout->addWidget(new QPushButton("Horizontal Layout 3"));
        return view;
    }

    QStackedWidget* m_stackedWidget;
};
```

---

## 九、对话框与窗口

### 9.1 模态/非模态对话框

```cpp
// ========== 模态对话框（阻塞）==========

void showModalDialog() {
    QDialog dialog;
    dialog.setWindowTitle("Modal Dialog");
    dialog.resize(300, 200);

    QVBoxLayout* layout = new QVBoxLayout(&dialog);
    layout->addWidget(new QLabel("This is a modal dialog"));

    QPushButton* okButton = new QPushButton("OK");
    connect(okButton, &QPushButton::clicked, &dialog, &QDialog::accept);
    layout->addWidget(okButton);

    // 阻塞直到对话框关闭
    if (dialog.exec() == QDialog::Accepted) {
        qDebug() << "User clicked OK";
    } else {
        qDebug() << "Dialog was rejected";
    }
}

// ========== 非模态对话框（非阻塞）==========

void showModelessDialog() {
    QDialog* dialog = new QDialog();
    dialog->setAttribute(Qt::WA_DeleteOnClose);  // 关闭时自动删除
    dialog->setWindowTitle("Modeless Dialog");
    dialog->resize(300, 200);

    QVBoxLayout* layout = new QVBoxLayout(dialog);
    layout->addWidget(new QLabel("This is a modeless dialog"));

    QPushButton* closeButton = new QPushButton("Close");
    connect(closeButton, &QPushButton::clicked, dialog, &QDialog::accept);
    connect(dialog, &QDialog::finished, dialog, &QDialog::deleteLater);
    layout->addWidget(closeButton);

    dialog->show();  // 非阻塞显示
    qDebug() << "Dialog shown, code continues...";
}

// ========== 自定义对话框 ==========

class LoginDialog : public QDialog {
    Q_OBJECT
public:
    LoginDialog(QWidget* parent = nullptr) : QDialog(parent) {
        setWindowTitle("Login");
        setModal(true);

        QVBoxLayout* layout = new QVBoxLayout(this);

        // 用户名
        layout->addWidget(new QLabel("Username:"));
        m_usernameEdit = new QLineEdit();
        layout->addWidget(m_usernameEdit);

        // 密码
        layout->addWidget(new QLabel("Password:"));
        m_passwordEdit = new QLineEdit();
        m_passwordEdit->setEchoMode(QLineEdit::Password);
        layout->addWidget(m_passwordEdit);

        // 按钮
        QHBoxLayout* buttonLayout = new QHBoxLayout();
        QPushButton* okButton = new QPushButton("Login");
        QPushButton* cancelButton = new QPushButton("Cancel");

        connect(okButton, &QPushButton::clicked, this, &LoginDialog::tryLogin);
        connect(cancelButton, &QPushButton::clicked, this, &QDialog::reject);

        buttonLayout->addStretch();
        buttonLayout->addWidget(okButton);
        buttonLayout->addWidget(cancelButton);
        layout->addLayout(buttonLayout);
    }

    QString username() const { return m_usernameEdit->text(); }
    QString password() const { return m_passwordEdit->text(); }

private slots:
    void tryLogin() {
        if (username().isEmpty() || password().isEmpty()) {
            QMessageBox::warning(this, "Error", "Please enter username and password");
            return;
        }
        accept();  // 关闭对话框，返回 Accepted
    }

private:
    QLineEdit* m_usernameEdit;
    QLineEdit* m_passwordEdit;
};

// 使用自定义对话框
void exampleCustomDialog() {
    LoginDialog dialog;
    if (dialog.exec() == QDialog::Accepted) {
        qDebug() << "Login successful:" << dialog.username();
    }
}
```

### 9.2 QMessageBox 消息框

```cpp
// ========== 各种消息框 ==========

void messageBoxExamples() {
    // 1. 信息框
    QMessageBox::information(nullptr, "Information",
        "This is an information message.",
        QMessageBox::Ok);

    // 2. 警告框
    QMessageBox::warning(nullptr, "Warning",
        "This is a warning message!",
        QMessageBox::Ok | QMessageBox::Cancel);

    // 3. 错误框
    QMessageBox::critical(nullptr, "Error",
        "An error occurred!",
        QMessageBox::Abort | QMessageBox::Retry | QMessageBox::Ignore);

    // 4. 询问框
    QMessageBox::StandardButton reply = QMessageBox::question(nullptr,
        "Question",
        "Do you want to continue?",
        QMessageBox::Yes | QMessageBox::No | QMessageBox::Cancel);

    if (reply == QMessageBox::Yes) {
        qDebug() << "User clicked Yes";
    } else if (reply == QMessageBox::No) {
        qDebug() << "User clicked No";
    } else {
        qDebug() << "User clicked Cancel";
    }

    // 5. 关于框
    QMessageBox::about(nullptr, "About",
        "<h2>My Application</h2>"
        "<p>Version 1.0.0</p>"
        "<p>Copyright © 2024</p>");

    // ========== 详细配置 ==========

    QMessageBox msgBox;
    msgBox.setWindowTitle("Save Changes");
    msgBox.setText("The document has been modified.");
    msgBox.setInformativeText("Do you want to save your changes?");
    msgBox.setStandardButtons(QMessageBox::Save | QMessageBox::Discard | QMessageBox::Cancel);
    msgBox.setDefaultButton(QMessageBox::Save);

    // 添加自定义按钮
    QPushButton* saveAllButton = msgBox.addButton("Save All", QMessageBox::ActionRole);

    int ret = msgBox.exec();

    if (msgBox.clickedButton() == saveAllButton) {
        qDebug() << "Save All clicked";
    } else if (ret == QMessageBox::Save) {
        qDebug() << "Save clicked";
    } else if (ret == QMessageBox::Discard) {
        qDebug() << "Discard clicked";
    } else {
        qDebug() << "Cancel clicked";
    }
}

// ========== 自定义消息框 ==========

void customMessageBox() {
    QMessageBox msgBox;
    msgBox.setIcon(QMessageBox::Question);
    msgBox.setWindowTitle("Custom Dialog");
    msgBox.setText("<h3>Custom Message</h3>");

    // 添加复选框
    QCheckBox* checkBox = new QCheckBox("Don't show this again");
    msgBox.setCheckBox(checkBox);

    // 添加详细文本
    msgBox.setDetailedText("Additional information:\nLine 1\nLine 2\nLine 3");

    msgBox.exec();

    if (checkBox->isChecked()) {
        qDebug() << "User checked 'Don't show again'";
    }
}
```

### 9.3 QMainWindow、QWidget、QDialog 区别

```cpp
/*
 * ========== 三种窗口类型区别 ==========
 *
 * QWidget:
 *   - 最基础的窗口类
 *   - 可以作为顶层窗口或子窗口
 *   - 没有预定义的布局结构
 *   - 适合自定义窗口
 *
 * QMainWindow:
 *   - 专为主窗口设计
 *   - 包含菜单栏、工具栏、状态栏、停靠窗口
 *   - 有中心部件区域
 *   - 适合应用程序主窗口
 *
 * QDialog:
 *   - 专为对话框设计
 *   - 有模态/非模态模式
 *   - 返回值（Accepted/Rejected）
 *   - 适合对话框和设置窗口
 */

// ========== QMainWindow 示例 ==========

class MainWindow : public QMainWindow {
    Q_OBJECT
public:
    MainWindow(QWidget* parent = nullptr) : QMainWindow(parent) {
        setWindowTitle("Main Window");
        resize(800, 600);

        // ========== 菜单栏 ==========
        QMenuBar* menuBar = this->menuBar();

        QMenu* fileMenu = menuBar->addMenu("File");
        fileMenu->addAction("New", this, &MainWindow::newFile, QKeySequence::New);
        fileMenu->addAction("Open", this, &MainWindow::openFile, QKeySequence::Open);
        fileMenu->addSeparator();
        fileMenu->addAction("Exit", this, &MainWindow::close, QKeySequence::Quit);

        QMenu* editMenu = menuBar->addMenu("Edit");
        editMenu->addAction("Undo", this, &MainWindow::undo, QKeySequence::Undo);
        editMenu->addAction("Redo", this, &MainWindow::redo, QKeySequence::Redo);

        QMenu* helpMenu = menuBar->addMenu("Help");
        helpMenu->addAction("About", this, &MainWindow::about);

        // ========== 工具栏 ==========
        QToolBar* toolBar = addToolBar("Main Toolbar");
        toolBar->addAction("New", this, &MainWindow::newFile);
        toolBar->addAction("Open", this, &MainWindow::openFile);
        toolBar->addSeparator();
        toolBar->addAction("Save", this, &MainWindow::saveFile);

        // 添加工具栏按钮
        QPushButton* toolButton = new QPushButton("Tool");
        toolBar->addWidget(toolButton);

        // ========== 状态栏 ==========
        QStatusBar* statusBar = this->statusBar();
        statusBar->showMessage("Ready", 3000);

        // 添加永久部件
        statusBar->addPermanentWidget(new QLabel("Line: 1"));
        statusBar->addPermanentWidget(new QLabel("Col: 1"));

        // ========== 中心部件 ==========
        QTextEdit* textEdit = new QTextEdit();
        setCentralWidget(textEdit);

        // ========== 停靠窗口 ==========
        QDockWidget* dock = new QDockWidget("Project", this);
        dock->setAllowedAreas(Qt::LeftDockWidgetArea | Qt::RightDockWidgetArea);

        QListWidget* projectList = new QListWidget();
        projectList->addItem("File 1");
        projectList->addItem("File 2");
        dock->setWidget(projectList);

        addDockWidget(Qt::LeftDockWidgetArea, dock);

        // 允许停靠窗口关闭
        dock->setFeatures(QDockWidget::DockWidgetMovable | QDockWidget::DockWidgetFloatable);
    }

private slots:
    void newFile() { qDebug() << "New file"; }
    void openFile() { qDebug() << "Open file"; }
    void saveFile() { qDebug() << "Save file"; }
    void undo() { qDebug() << "Undo"; }
    void redo() { qDebug() << "Redo"; }
    void about() {
        QMessageBox::about(this, "About", "My Application v1.0");
    }

private:
};

// ========== QWidget 示例 ==========

class CustomWidget : public QWidget {
public:
    CustomWidget(QWidget* parent = nullptr) : QWidget(parent) {
        setWindowTitle("Custom Widget");
        resize(400, 300);

        // 窗口标志
        setWindowFlags(Qt::Window | Qt::WindowMinimizeButtonHint | Qt::WindowMaximizeButtonHint);

        QVBoxLayout* layout = new QVBoxLayout(this);
        layout->addWidget(new QLabel("This is a custom QWidget"));

        // 可以作为顶层窗口
        // w.show();

        // 也可以嵌入其他窗口
        // layout->addWidget(w);
    }
};

// ========== QDialog 示例 ==========

class SettingsDialog : public QDialog {
public:
    SettingsDialog(QWidget* parent = nullptr) : QDialog(parent) {
        setWindowTitle("Settings");
        setModal(true);  // 设置为模态

        QVBoxLayout* layout = new QVBoxLayout(this);

        // 设置选项
        QCheckBox* darkMode = new QCheckBox("Dark Mode");
        layout->addWidget(darkMode);

        QComboBox* language = new QComboBox();
        language->addItems({"English", "中文", "日本語"});
        layout->addWidget(language);

        // 按钮
        QDialogButtonBox* buttons = new QDialogButtonBox(
            QDialogButtonBox::Ok | QDialogButtonBox::Cancel | QDialogButtonBox::Apply);

        connect(buttons, &QDialogButtonBox::accepted, this, &QDialog::accept);
        connect(buttons, &QDialogButtonBox::rejected, this, &QDialog::reject);
        connect(buttons->button(QDialogButtonBox::Apply), &QPushButton::clicked, [=]() {
            qDebug() << "Apply settings";
        });

        layout->addWidget(buttons);
    }
};
```

---

## 十、Qt 编译系统 ⭐

### 10.1 moc（元对象编译器）

```cpp
/*
 * ========== moc 工作原理 ==========
 *
 * 1. moc 预处理含有 Q_OBJECT 宏的头文件
 * 2. 生成 moc_*.cpp 文件，包含：
 *    - 元对象数据（QMetaObject）
 *    - 信号的实现
 *    - qt_metacall、qt_metacast 等函数
 * 3. moc_*.cpp 与其他源文件一起编译链接
 */

// ========== 需要 moc 处理的情况 ==========

// 1. 使用 Q_OBJECT 宏
class MyObject : public QObject {
    Q_OBJECT  // 必须由 moc 处理
public:
    MyObject(QObject* parent = nullptr) : QObject(parent) {}

signals:
    void mySignal();  // moc 会生成实现

public slots:
    void mySlot() {}  // 标记为槽
};

// 2. 使用 Q_PROPERTY
class Widget : public QWidget {
    Q_OBJECT
    Q_PROPERTY(QString title READ title WRITE setTitle NOTIFY titleChanged)

public:
    QString title() const { return m_title; }
    void setTitle(const QString& title) {
        if (m_title != title) {
            m_title = title;
            emit titleChanged(title);
        }
    }

signals:
    void titleChanged(const QString& title);

private:
    QString m_title;
};

// 3. 使用 Q_ENUM / Q_FLAG
class MyClass : public QObject {
    Q_OBJECT
public:
    enum Priority {
        Low,
        Normal,
        High,
        Urgent
    };
    Q_ENUM(Priority)  // 注册到元对象系统
};

// 4. 声明信号
class Emitter : public QObject {
    Q_OBJECT
public:
    Emitter(QObject* parent = nullptr) : QObject(parent) {}

    void emitSignal() {
        emit dataReady(42);  // 信号由 moc 实现
    }

signals:
    void dataReady(int value);
};

/*
 * ========== moc 命令行使用 ==========
 *
 * moc header.h -o moc_header.cpp
 *
 * Qt 的构建系统（qmake/CMake）会自动调用 moc
 */
```

### 10.2 qmake

// 【.pro 文件变量速查表】
/*
 * ┌─────────────────┬────────────────────────────────────────┐
 * │ 变量名          │ 用途说明                               │
 * ├─────────────────┼────────────────────────────────────────┤
 * │ TEMPLATE        │ 项目模板 (app/lib/subdirs/aux)         │
 * │ TARGET          │ 目标文件名（不含扩展名）               │
 * │ QT              │ Qt 模块 (core/gui/widgets/...)        │
 * │ SOURCES         │ 源文件列表 (*.cpp)                     │
 * │ HEADERS         │ 头文件列表 (*.h)                       │
 * │ FORMS           │ UI 文件列表 (*.ui)                     │
 * │ RESOURCES       │ 资源文件列表 (*.qrc)                   │
 * │ CONFIG          │ 编译配置选项                           │
 * │ DEFINES         │ 预处理器宏定义                         │
 * │ INCLUDEPATH     │ 头文件搜索路径                         │
 * │ LIBS            │ 链接库 (-lxxx 或 /path/to/lib)         │
 * │ LFLAGS          │ 链接器标志                             │
 * │ CXXFLAGS        │ C++ 编译器标志                         │
 * │ MOC_DIR         │ MOC 生成文件目录                       │
 * │ OBJECTS_DIR     │ 目标文件目录 (.obj)                    │
 * │ UI_DIR          │ UI 生成文件目录                        │
 * │ RCC_DIR         │ RCC 生成文件目录                       │
 * └─────────────────┴────────────────────────────────────────┘
 *
 * 【CONFIG 常用选项】
 * ┌─────────────────┬────────────────────────────────────────┐
 * │ c++11/14/17/20  │ C++ 语言标准                           │
 * │ debug/release   │ 构建类型                               │
 * │ debug_and_release │ 同时构建两种版本                    │
 * │ console         │ 控制台程序（Windows 显示控制台）       │
 * │ warn_on/off     │ 编译警告开关                           │
 * │ exceptions      │ 启用异常（默认）                       │
 * │ rtti            │ 启用 RTTI（默认）                      │
 * │ stl             │ 启用 STL（默认）                       │
 * │ static/shared   │ 静态库/动态库（lib 模板）              │
 * │ plugin          │ 生成插件（lib 模板）                   │
 * └─────────────────┴────────────────────────────────────────┘
 */

```qmake
# ========== .pro 文件基本结构 ==========

# 【项目模板】TEMPLATE
# 指定项目类型，决定 qmake 生成的 Makefile 类型
TEMPLATE = app
# 可选值:
#   app        - 应用程序（默认）生成可执行文件
#   lib        - 库项目，生成 .dll/.so/.dylib
#   subdirs    - 子目录项目，用于构建多目录工程
#   aux        - 辅助项目，不编译，仅用于安装等操作

# 【目标名称】TARGET
# 指定生成的目标文件名（不含扩展名）
TARGET = MyApplication
# Windows:   MyApplication.exe
# Linux:     MyApplication
# macOS:     MyApplication.app

# 【Qt 模块】QT +=/ -=
# 添加需要的 Qt 模块，自动处理依赖关系
QT += core gui widgets
# 常用模块:
#   core          - 核心功能（默认包含，无需显式添加）
#   gui           - GUI 功能（依赖 core）：QPainter, QImage, QColor
#   widgets       - 控件（依赖 gui）：QWidget, QPushButton, QLabel
#   network       - 网络功能：QTcpSocket, QNetworkAccessManager
#   sql           - 数据库：QSqlDatabase, QSqlQuery
#   xml           - XML 处理：QDomDocument, QXmlStreamReader
#   charts        - 图表（需要单独安装）：QChart, QLineSeries
#   printsupport  - 打印支持：QPrinter, QPrintDialog
#   testlib       - 测试框架：QTest
#   concurrent    - 并发：QtConcurrent, QFuture
#   multimedia    - 多媒体：QMediaPlayer, QCamera

# 排除模块（慎用）
QT -= gui  # 移除 gui 模块，用于纯控制台程序

# ========== 源文件 ==========
# 【SOURCES】指定 C++ 源文件
SOURCES += main.cpp \
           widget.cpp \
           model.cpp

# ========== 头文件 ==========
# 【HEADERS】指定 C++ 头文件
HEADERS += widget.h \
           model.h

# ========== UI 文件 ==========
# 【FORMS】指定 Qt Designer UI 文件
FORMS += widget.ui \
         dialog.ui

# ========== 资源文件 ==========
# 【RESOURCES】指定 Qt 资源文件
RESOURCES += resources.qrc

# ========== 编译配置 ==========
# 【CONFIG】编译选项配置
CONFIG += c++17              # C++ 标准（c++11/c++14/c++17/c++20）
CONFIG += debug_and_release  # 同时生成 debug 和 release
CONFIG += console            # 控制台程序（Windows 显示控制台窗口）
CONFIG += warn_on            # 显示编译警告（默认）
CONFIG += exceptions         # 启用 C++ 异常（默认）
CONFIG += rtti               # 启用运行时类型信息（默认）
CONFIG += stl                # 启用 STL（默认）

# 【DEFINES】预处理器宏定义
DEFINES += QT_DEPRECATED_WARNINGS
DEFINES += MY_APP_VERSION=\\\"1.0.0\\\"  # 字符串宏需转义引号

# 【INCLUDEPATH】头文件搜索路径（-I 选项）
INCLUDEPATH += /usr/local/include \
               ./include

# 【LIBS】链接库（-L -l 选项或直接指定路径）
LIBS += -L/usr/local/lib -lmylib  # -L:库路径 -l:库名
LIBS += -lxml2 -lz
# Windows 直接指定 .lib 文件
LIBS += C:/mylib/mylib.lib

# 【条件编译】平台判断
win32 {                # Windows 平台
    SOURCES += windows_specific.cpp
    LIBS += -luser32 -lgdi32

# 目标名称
TARGET = MyApplication
# Windows: MyApplication.exe
# Linux/macOS: MyApplication

# Qt 模块
QT += core gui widgets
# 可用模块:
#   core     - 核心功能（默认包含）
#   gui      - GUI 功能（依赖 core）
#   widgets  - 控件（依赖 gui）
#   network  - 网络功能
#   sql      - 数据库
#   xml      - XML 处理
#   charts   - 图表
#   printsupport - 打印支持
#   testlib  - 测试框架

# 排除模块
QT -= gui

# ========== 源文件 ==========
SOURCES += main.cpp \
           widget.cpp \
           model.cpp

# 或使用通配符
SOURCES += *.cpp

# ========== 头文件 ==========
HEADERS += widget.h \
           model.h

# ========== UI 文件 ==========
FORMS += widget.ui \
         dialog.ui

# ========== 资源文件 ==========
RESOURCES += resources.qrc

# ========== 编译配置 ==========
CONFIG += c++17  # C++ 标准
CONFIG += debug_and_release  # 同时生成 debug 和 release
CONFIG += console  # 控制台程序（Windows）
CONFIG += warn_on  # 显示警告
CONFIG += exceptions  # 启用异常
CONFIG += rtti  # 启用 RTTI
CONFIG += stl  # 启用 STL

# 定义宏
DEFINES += QT_DEPRECATED_WARNINGS
DEFINES += MY_APP_VERSION=\\\"1.0.0\\\"

# ========== 包含路径 ==========
INCLUDEPATH += /usr/local/include \
               ./include

# ========== 库路径和链接 ==========
LIBS += -L/usr/local/lib -lmylib
LIBS += -lxml2 -lz
# Windows:
LIBS += C:/mylib/mylib.lib

# ========== 条件编译 ==========
# 平台判断
win32 {
    SOURCES += windows_specific.cpp
    LIBS += -luser32 -lgdi32
    DEFINES += WINDOWS_OS
}

unix:!macx {
    SOURCES += linux_specific.cpp
    LIBS += -lpthread
    TARGET = myapp-linux
}

macx {
    SOURCES += mac_specific.cpp
    LIBS += -framework Cocoa
    TARGET = myapp-mac
}

# Debug/Release 区分
CONFIG(debug, debug|release) {
    SOURCES += debug_tools.cpp
    DEFINES += DEBUG_MODE
    TARGET = myapp_d
} else {
    DEFINES += NDEBUG
    TARGET = myapp
}

# ========== 自定义编译规则 ==========
# 指定 moc 生成的文件位置
MOC_DIR = build/moc
OBJECTS_DIR = build/obj
RCC_DIR = build/rcc
UI_DIR = build/ui

# ========== 子目录项目 ==========
TEMPLATE = subdirs
SUBDIRS = app \
          lib1 \
          lib2

app.depends = lib1 lib2  # app 依赖 lib1 和 lib2
```

### 10.3 CMake

```cmake
# ========== CMakeLists.txt 基本结构 ==========

cmake_minimum_required(VERSION 3.16)
project(MyApp VERSION 1.0.0 LANGUAGES CXX)

# 设置 C++ 标准
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# 自动处理 Qt 的 moc、rcc、uic
set(CMAKE_AUTOMOC ON)
set(CMAKE_AUTORCC ON)
set(CMAKE_AUTOUIC ON)

# ========== 查找 Qt 包 ==========
find_package(Qt6 REQUIRED COMPONENTS Core Widgets)
# 或 Qt5:
# find_package(Qt5 REQUIRED COMPONENTS Core Widgets)

# ========== 创建可执行文件 ==========
qt_add_executable(MyApp
    main.cpp
    widget.cpp
    widget.h
    widget.ui
    resources.qrc
)

# ========== 链接 Qt 库 ==========
target_link_libraries(MyApp PRIVATE
    Qt6::Core
    Qt6::Widgets
)

# ========== 包含目录 ==========
target_include_directories(MyApp PRIVATE
    ${CMAKE_CURRENT_SOURCE_DIR}/include
)

# ========== 编译定义 ==========
target_compile_definitions(MyApp PRIVATE
    QT_DEPRECATED_WARNINGS
    MY_APP_VERSION="${PROJECT_VERSION}"
)

# ========== 创建库 ==========
qt_add_library(MyLib
    lib.cpp
    lib.h
)

target_link_libraries(MyLib PUBLIC
    Qt6::Core
)

# ========== 安装规则 ==========
install(TARGETS MyApp
    BUNDLE DESTINATION .
    RUNTIME DESTINATION bin
)

# ========== 条件编译 ==========
if(WIN32)
    target_sources(MyApp PRIVATE windows_specific.cpp)
    target_link_libraries(MyApp PRIVATE user32)
elseif(UNIX AND NOT APPLE)
    target_sources(MyApp PRIVATE linux_specific.cpp)
    target_link_libraries(MyApp PRIVATE pthread)
elseif(APPLE)
    target_sources(MyApp PRIVATE mac_specific.cpp)
endif()

# ========== Debug/Release 配置 ==========
set(CMAKE_CONFIGURATION_TYPES "Debug;Release")
set(CMAKE_DEBUG_POSTFIX "_d")

# ========== 子目录项目 ==========
add_subdirectory(lib)
add_subdirectory(app)

# ========== 自定义命令 ==========
# 生成版本信息
execute_process(
    COMMAND git describe --tags --always
    OUTPUT_VARIABLE GIT_VERSION
    OUTPUT_STRIP_TRAILING_WHITESPACE
)
target_compile_definitions(MyApp PRIVATE GIT_VERSION="${GIT_VERSION}")
```

### 10.4 编译流程

```bash
# ========== Qt 编译完整流程 ==========

# 1. 预处理
#    - 处理 #include、#define 等预处理指令
#    - moc 处理 Q_OBJECT 宏，生成 moc_*.cpp

# 2. 编译
#    - 将 .cpp 和 moc_*.cpp 编译成 .o 文件
#    - rcc 处理 .qrc 资源文件，生成 qrc_*.cpp
#    - uic 处理 .ui 文件，生成 ui_*.h

# 3. 链接
#    - 将所有 .o 文件链接成最终可执行文件
#    - 链接 Qt 库

# ========== qmake 构建流程 ==========
qmake project.pro    # 生成 Makefile
make                  # 编译
# Windows:
qmake project.pro
nmake
# 或使用 mingw32-make

# ========== CMake 构建流程 ==========
mkdir build
cd build
cmake ..             # 生成 Makefile/VS 项目
cmake --build .      # 编译
# 或
make

# ========== 跨平台编译示例 ==========
# 为 Windows 编译（从 Linux）
mingw32-cmake .. -DCMAKE_TOOLCHAIN_FILE=mingw.cmake
cmake --build .

# 为 macOS 编译（从 Linux）
osxcross-cmake .. -DCMAKE_TOOLCHAIN_FILE=osx.cmake
cmake --build .
```

---

## 十一、Qt 与 Windows API 互操作 ⭐

### 11.1 获取窗口句柄

```cpp
#include <windows.h>

// ========== 获取 HWND ==========
void exampleGetHwnd() {
    QWidget* widget = new QWidget();
    widget->show();

    // Qt5/Qt6 方式
    HWND hwnd = reinterpret_cast<HWND>(widget->winId());

    qDebug() << "HWND:" << hwnd;

    // 验证句柄有效性
    if (IsWindow(hwnd)) {
        qDebug() << "Valid window handle";
    }
}

// ========== WId 类型定义 ==========
/*
 * WId 是 Qt 的窗口标识符类型
 * Windows: HWND
 * X11: Window
 * macOS: NSView*
 * iOS/macOS: UIView*
 */

// ========== 获取子窗口句柄 ==========
void exampleChildHwnd() {
    QMainWindow* mainWindow = new QMainWindow();
    QTextEdit* textEdit = new QTextEdit(mainWindow);
    mainWindow->setCentralWidget(textEdit);
    mainWindow->show();

    // 获取主窗口句柄
    HWND mainHwnd = reinterpret_cast<HWND>(mainWindow->winId());

    // 获取子控件句柄
    HWND editHwnd = reinterpret_cast<HWND>(textEdit->winId());

    qDebug() << "Main HWND:" << mainHwnd;
    qDebug() << "Edit HWND:" << editHwnd;
}

// ========== 有效平台判断 ==========
void crossPlatformExample() {
    QWidget* widget = new QWidget();

    #ifdef Q_OS_WIN
        HWND hwnd = reinterpret_cast<HWND>(widget->winId());
        // Windows 特定操作
    #elif defined(Q_OS_LINUX)
        // X11 特定操作
    #elif defined(Q_OS_MACOS)
        // macOS 特定操作
    #endif
}
```

### 11.2 Qt 调用 Windows API

```cpp
// ========== 窗口操作 ==========

void windowOperations() {
    QWidget* widget = new QWidget();
    widget->setWindowTitle("Windows API Demo");
    widget->resize(400, 300);
    widget->show();

    HWND hwnd = reinterpret_cast<HWND>(widget->winId());

    // ========== 设置窗口位置 ==========
    SetWindowPos(hwnd, HWND_TOP, 100, 100, 400, 300, SWP_SHOWWINDOW);

    // ========== 获取窗口矩形 ==========
    RECT rect;
    GetWindowRect(hwnd, &rect);
    qDebug() << "Window rect:" << rect.left << rect.top << rect.right << rect.bottom;

    // ========== 获取客户区矩形 ==========
    RECT clientRect;
    GetClientRect(hwnd, &clientRect);
    qDebug() << "Client rect:" << clientRect.right << clientRect.bottom;

    // ========== 设置窗口标题 ==========
    SetWindowText(hwnd, L"New Title from Windows API");

    // ========== 显示/隐藏窗口 ==========
    ShowWindow(hwnd, SW_MINIMIZE);  // 最小化
    Sleep(1000);
    ShowWindow(hwnd, SW_RESTORE);   // 恢复

    // ========== 窗口动画 ==========
    // 淡入效果
    AnimateWindow(hwnd, 500, AW_BLEND | AW_ACTIVATE);
}

// ========== 系统托盘 ==========

class SystemTrayIcon : public QObject {
    Q_OBJECT
public:
    SystemTrayIcon(QObject* parent = nullptr) : QObject(parent) {
        // 使用 Qt 的托盘功能
        m_trayIcon = new QSystemTrayIcon(this);
        m_trayIcon->setIcon(QIcon(":/icons/app.png"));
        m_trayIcon->setToolTip("My Application");

        // 创建菜单
        QMenu* menu = new QMenu();
        menu->addAction("Show", this, &SystemTrayIcon::showWindow);
        menu->addAction("Quit", this, &SystemTrayIcon::quit);
        m_trayIcon->setContextMenu(menu);

        m_trayIcon->show();
    }

    void setWindow(QWidget* window) {
        m_window = window;
        connect(m_trayIcon, &QSystemTrayIcon::activated, [=](QSystemTrayIcon::ActivationReason reason) {
            if (reason == QSystemTrayIcon::DoubleClick) {
                showWindow();
            }
        });
    }

private slots:
    void showWindow() {
        if (m_window) {
            m_window->showNormal();
            m_window->raise();
            m_window->activateWindow();
        }
    }

    void quit() {
        qApp->quit();
    }

private:
    QSystemTrayIcon* m_trayIcon;
    QWidget* m_window = nullptr;
};

// ========== 注册表操作 ==========

#include <QSettings>

void registryOperations() {
    // Qt 方式（跨平台）
    QSettings settings("HKEY_CURRENT_USER\\Software\\MyCompany", QSettings::NativeFormat);

    // 写入
    settings.setValue("MyApp/Version", "1.0.0");
    settings.setValue("MyApp/Path", "C:\\Program Files\\MyApp");

    // 读取
    QString version = settings.value("MyApp/Version").toString();

    // 删除
    settings.remove("MyApp");

    // ========== Windows API 方式 ==========
    HKEY hKey;
    LPCWSTR subKey = L"Software\\MyCompany\\MyApp";

    // 打开注册表键
    if (RegOpenKeyEx(HKEY_CURRENT_USER, subKey, 0, KEY_WRITE, &hKey) == ERROR_SUCCESS) {
        // 写入值
        const wchar_t* value = L"1.0.0";
        RegSetValueEx(hKey, L"Version", 0, REG_SZ, (const BYTE*)value, (wcslen(value) + 1) * sizeof(wchar_t));

        RegCloseKey(hKey);
    }

    // 读取值
    DWORD type, size;
    if (RegOpenKeyEx(HKEY_CURRENT_USER, subKey, 0, KEY_READ, &hKey) == ERROR_SUCCESS) {
        // 获取大小
        RegQueryValueEx(hKey, L"Version", NULL, &type, NULL, &size);

        // 读取数据
        wchar_t* buffer = new wchar_t[size / sizeof(wchar_t)];
        RegQueryValueEx(hKey, L"Version", NULL, &type, (BYTE*)buffer, &size);
        qDebug() << "Version:" << QString::fromWCharArray(buffer);
        delete[] buffer;

        RegCloseKey(hKey);
    }
}

// ========== DLL 加载 ==========

void dllOperations() {
    // Qt 方式
    QLibrary library("mylib.dll");
    if (library.load()) {
        // 解析函数
        typedef void (*MyFunction)();
        MyFunction myFunc = (MyFunction)library.resolve("myFunction");

        if (myFunc) {
            myFunc();
        }

        library.unload();
    }

    // ========== Windows API 方式 ==========
    HINSTANCE hDll = LoadLibrary(L"mylib.dll");
    if (hDll) {
        // 获取函数地址
        typedef void (*MyFunction)();
        MyFunction myFunc = (MyFunction)GetProcAddress(hDll, "myFunction");

        if (myFunc) {
            myFunc();
        }

        FreeLibrary(hDll);
    }
}

// ========== Windows 消息处理 ==========

class NativeEventWidget : public QWidget {
protected:
    // 处理 Windows 消息
#if QT_VERSION >= QT_VERSION_CHECK(6, 0, 0)
    bool nativeEvent(const QByteArray& eventType, void* message, qintptr* result) override#else
    bool nativeEvent(const QByteArray& eventType, void* message, long* result) override#endif
    {
        if (eventType == "windows_generic_MSG") {
            MSG* msg = static_cast<MSG*>(message);

            if (msg->message == WM_KEYDOWN) {
                // 处理键盘按下
                WPARAM key = msg->wParam;
                qDebug() << "Key pressed:" << key;

                // 可以返回 true 阻止消息继续传递
                // *result = 0;
                // return true;
            }

            if (msg->message == WM_SIZE) {
                // 窗口大小改变
                int width = LOWORD(msg->lParam);
                int height = HIWORD(msg->lParam);
                qDebug() << "Size:" << width << height;
            }

            if (msg->message == WM_POWERBROADCAST) {
                // 系统电源事件
                if (msg->wParam == PBT_APMSUSPEND) {
                    qDebug() << "System suspending";
                    saveData();
                }
            }
        }

        return QWidget::nativeEvent(eventType, message, result);
    }

private:
    void saveData() {
        // 保存数据
    }
};

// ========== 设备通知 ==========

class DeviceNotificationWidget : public QWidget {
    Q_OBJECT
public:
    DeviceNotificationWidget(QWidget* parent = nullptr) : QWidget(parent) {
        registerDeviceNotification();
    }

    ~DeviceNotificationWidget() {
        unregisterDeviceNotification();
    }

protected:
#if QT_VERSION >= QT_VERSION_CHECK(6, 0, 0)
    bool nativeEvent(const QByteArray& eventType, void* message, qintptr* result) override#else
    bool nativeEvent(const QByteArray& eventType, void* message, long* result) override#endif
    {
        if (eventType == "windows_generic_MSG") {
            MSG* msg = static_cast<MSG*>(message);

            if (msg->message == WM_DEVICECHANGE) {
                onDeviceChange(msg->wParam, msg->lParam);
            }
        }

        return QWidget::nativeEvent(eventType, message, result);
    }

private:
    HDEVNOTIFY m_hDevNotify = nullptr;

    void registerDeviceNotification() {
        HWND hwnd = reinterpret_cast<HWND>(winId());

        // 注册 USB 设备通知
        DEV_BROADCAST_DEVICEINTERFACE filter = {};
        filter.dbcc_size = sizeof(filter);
        filter.dbcc_devicetype = DBT_DEVTYP_DEVICEINTERFACE;
        // filter.dbcc_classguid = ...;  // 特定设备类型

        m_hDevNotify = RegisterDeviceNotification(
            hwnd,
            &filter,
            DEVICE_NOTIFY_WINDOW_HANDLE
        );
    }

    void unregisterDeviceNotification() {
        if (m_hDevNotify) {
            UnregisterDeviceNotification(m_hDevNotify);
            m_hDevNotify = nullptr;
        }
    }

    void onDeviceChange(WPARAM wParam, LPARAM lParam) {
        switch (wParam) {
            case DBT_DEVICEARRIVAL:
                qDebug() << "Device connected";
                break;
            case DBT_DEVICEREMOVECOMPLETE:
                qDebug() << "Device removed";
                break;
        }
    }
};
```

### 11.3 常用 Windows API 集成

```cpp
// ========== 文件关联 ==========

void registerFileAssociation() {
    QString appPath = QCoreApplication::applicationFilePath();
    QString fileType = "MyApp.File";
    QString ext = ".myfile";

    QSettings settings("HKEY_CURRENT_USER\\Software\\Classes", QSettings::NativeFormat);

    // 设置文件类型
    settings.setValue(fileType + "/DefaultIcon/.", appPath + ",0");
    settings.setValue(fileType + "/shell/open/command/.", "\"" + appPath + "\" \"%1\"");

    // 关联扩展名
    settings.setValue("." + ext + "/Default", fileType);
    settings.setValue("." + ext + "/PerceivedType", "text");

    // 通知系统刷新
    SHChangeNotify(SHCNE_ASSOCCHANGED, SHCNF_IDLIST, NULL, NULL);
}

// ========== 启动外部程序 ==========

void startExternalProcess() {
    // Qt 方式（跨平台）
    QProcess::startDetached("notepad.exe", {"C:\\test.txt"});

    // Windows API 方式
    SHELLEXECUTEINFO info = {};
    info.cbSize = sizeof(info);
    info.fMask = SEE_MASK_NOCLOSEPROCESS;
    info.lpVerb = L"open";
    info.lpFile = L"notepad.exe";
    info.lpParameters = L"C:\\test.txt";
    info.nShow = SW_SHOW;

    ShellExecuteEx(&info);

    // 使用 CreateProcess
    STARTUPINFO si = {};
    PROCESS_INFORMATION pi = {};
    si.cb = sizeof(si);

    wchar_t cmd[] = L"notepad.exe C:\\test.txt";
    if (CreateProcess(NULL, cmd, NULL, NULL, FALSE, 0, NULL, NULL, &si, &pi)) {
        // 等待进程结束
        WaitForSingleObject(pi.hProcess, INFINITE);

        CloseHandle(pi.hProcess);
        CloseHandle(pi.hThread);
    }
}

// ========== 获取系统信息 ==========

void getSystemInfo() {
    // Qt 方式
    qDebug() << "Computer name:" << QSysInfo::machineHostName();
    qDebug() << "OS:" << QSysInfo::prettyProductName();
    qDebug() << "Kernel:" << QSysInfo::kernelType();
    qDebug() << "Architecture:" << QSysInfo::currentCpuArchitecture();
    qDebug() << "Word size:" << QSysInfo::WordSize;

    // Windows API 方式
    // 获取内存信息
    MEMORYSTATUSEX memStatus = {};
    memStatus.dwLength = sizeof(memStatus);
    GlobalMemoryStatusEx(&memStatus);

    qDebug() << "Total memory:" << memStatus.ullTotalPhys / (1024 * 1024) << "MB";
    qDebug() << "Available memory:" << memStatus.ullAvailPhys / (1024 * 1024) << "MB";
    qDebug() << "Memory usage:" << memStatus.dwMemoryLoad << "%";

    // 获取磁盘信息
    ULARGE_INTEGER freeBytes, totalBytes;
    if (GetDiskFreeSpaceEx(L"C:", &freeBytes, &totalBytes, NULL)) {
        qDebug() << "Total disk:" << totalBytes.QuadPart / (1024 * 1024 * 1024) << "GB";
        qDebug() << "Free disk:" << freeBytes.QuadPart / (1024 * 1024 * 1024) << "GB";
    }

    // 获取系统版本
    OSVERSIONINFOEX osvi = {};
    osvi.dwOSVersionInfoSize = sizeof(osvi);
    // GetVersionEx 已弃用，使用其他方式
}
```

---

## 十二、C++ 标准库在 Qt 中的应用 ⭐

### 12.1 容器类对比

```cpp
// ========== STL 容器 vs Qt 容器 ==========

void containerComparison() {
    // ========== std::vector vs QVector ==========
    std::vector<int> stlVec = {1, 2, 3, 4, 5};
    QVector<int> qtVec = {1, 2, 3, 4, 5};

    // 访问元素
    int stlValue = stlVec[0];
    int qtValue = qtVec[0];

    // 添加元素
    stlVec.push_back(6);
    qtVec.append(6);

    // 迭代器
    for (auto it = stlVec.begin(); it != stlVec.end(); ++it) {
        qDebug() << *it;
    }

    for (QVector<int>::iterator it = qtVec.begin(); it != qtVec.end(); ++it) {
        qDebug() << *it;
    }

    // 范围 for（C++11）
    for (const auto& val : stlVec) {
        qDebug() << val;
    }

    for (const int& val : qtVec) {
        qDebug() << val;
    }

    // ========== std::map vs QMap ==========
    std::map<QString, int> stlMap;
    QMap<QString, int> qtMap;

    // 插入
    stlMap["key1"] = 1;
    stlMap.insert({"key2", 2});

    qtMap["key1"] = 1;
    qtMap.insert("key2", 2);

    // 查找
    auto stlIt = stlMap.find("key1");
    if (stlIt != stlMap.end()) {
        qDebug() << "Found:" << stlIt->second;
    }

    auto qtIt = qtMap.find("key1");
    if (qtIt != qtMap.end()) {
        qDebug() << "Found:" << qtIt.value();
    }

    // Qt 的 QMap 提供 [] 操作符，找不到会自动创建
    int value = qtMap["nonexistent"];  // 返回 0（默认值）

    // ========== std::unordered_map vs QHash ==========
    std::unordered_map<QString, int> stlHash;
    QHash<QString, int> qtHash;

    // QHash 查找更快，但不排序
    // QMap 排序，查找 O(log n)
    // QHash 不排序，查找 O(1) 平均
}

// ========== 隐式共享（写时复制）==========

void implicitSharing() {
    /*
     * Qt 容器使用隐式共享（COW - Copy On Write）
     * - 复制容器时不复制数据，只增加引用计数
     * - 写入时才真正复制数据
     * - 提高效率，减少内存使用
     */

    QVector<int> vec1 = {1, 2, 3, 4, 5};
    QVector<int> vec2 = vec1;  // 共享数据，不复制

    qDebug() << "vec1 data:" << vec1.data();
    qDebug() << "vec2 data:" << vec2.data();  // 相同地址

    vec2[0] = 99;  // 写入时才复制数据
    vec2.detach();  // 强制分离

    // detach() 确保拥有独立副本
    // constData() 返回常量指针，不会触发分离
}

// ========== 使用 STL 算法与 Qt 容器 ==========

void stlAlgorithmsWithQt() {
    QVector<int> numbers = {5, 2, 8, 1, 9, 3};

    // 使用 STL 算法
    std::sort(numbers.begin(), numbers.end());
    // numbers: {1, 2, 3, 5, 8, 9}

    // 查找
    auto it = std::find(numbers.begin(), numbers.end(), 5);
    if (it != numbers.end()) {
        qDebug() << "Found 5 at index:" << std::distance(numbers.begin(), it);
    }

    // 计数
    int count = std::count(numbers.begin(), numbers.end(), 5);

    // Lambda + STL 算法
    int sum = std::accumulate(numbers.begin(), numbers.end(), 0);

    // 条件查找
    auto greaterThan5 = std::find_if(numbers.begin(), numbers.end(), [](int val) {
        return val > 5;
    });

    // 复制
    QVector<int> dest;
    std::copy_if(numbers.begin(), numbers.end(), std::back_inserter(dest), [](int val) {
        return val > 3;
    });
}
```

### 12.2 字符串处理

```cpp
// ========== QString vs std::string ==========

void stringComparison() {
    // ========== 创建字符串 ==========
    std::string stlStr = "Hello";
    QString qtStr = "Hello";
    QString qtStr2 = QString("Hello");
    QString qtStr3 = QStringLiteral("Hello");  // 编译期优化（推荐）

    // ========== 相互转换 ==========
    // QString -> std::string
    std::string str1 = qtStr.toStdString();
    std::string str2 = qtStr.toLocal8Bit().toStdString();
    std::string str3 = qtStr.toUtf8().toStdString();

    // std::string -> QString
    QString qtStr4 = QString::fromStdString(stlStr);
    QString qtStr5 = QString::fromLocal8Bit(stlStr.c_str());
    QString qtStr6 = QString::fromUtf8(stlStr.c_str());

    // ========== 字符串操作 ==========
    QString text = "Hello World";

    // 大小写
    QString upper = text.toUpper();      // "HELLO WORLD"
    QString lower = text.toLower();      // "hello world"

    // 查找
    int index = text.indexOf("World");   // 6
    bool contains = text.contains("World");  // true
    bool starts = text.startsWith("Hello");  // true
    bool ends = text.endsWith("World");      // true

    // 替换
    QString replaced = text;
    replaced.replace("World", "Qt");  // "Hello Qt"

    // 分割
    QStringList parts = text.split(" ");
    // parts: ["Hello", "World"]

    // 连接
    QString joined = parts.join(", ");  // "Hello, World"

    // 格式化
    int age = 25;
    QString formatted = QString("Age: %1").arg(age);
    QString formatted2 = QString("Name: %1, Age: %2").arg("John").arg(age);

    // 数字转换
    QString numStr = QString::number(3.14159, 'f', 2);  // "3.14"
    double num = numStr.toDouble();  // 3.14

    // ========== 编码转换 ==========
    QString unicode = QString::fromUtf8("中文测试");
    QByteArray utf8 = unicode.toUtf8();
    QByteArray gbk = unicode.toLocal8Bit();  // 在中文 Windows 上是 GBK

    // UTF-16
    QString::const_iterator it;
    for (it = unicode.constBegin(); it != unicode.constEnd(); ++it) {
        QChar ch = *it;
        ushort code = ch.unicode();
        qDebug() << "Code:" << code;
    }
}

// ========== std::string_view 与 QStringView ==========

void stringViewExample() {
    // C++17 string_view 避免复制
    std::string_view sv = "Hello World";
    qDebug() << sv.length();

    // Qt5.10+ QStringView
    QStringView qv = u"Hello World";
    qDebug() << qv.length();

    // QString 接受 QStringView 参数
    QString str = "Hello";
    QStringView view(str);
    qDebug() << view;  // 不复制数据
}
```

### 12.3 智能指针

```cpp
// ========== std::unique_ptr 与 Qt ==========
void uniquePtrExample() {
    // 独占所有权
    std::unique_ptr<QWidget> widget = std::make_unique<QWidget>();
    widget->setWindowTitle("Unique Ptr Demo");
    widget->show();

    // 移动语义
    std::unique_ptr<QWidget> widget2 = std::move(widget);
    // widget 现在为空

    // 自定义删除器
    auto deleter = [](QWidget* w) {
        qDebug() << "Custom deleter";
        w->deleteLater();  // 使用 Qt 的删除方式
    };
    std::unique_ptr<QWidget, decltype(deleter)> widget3(new QWidget, deleter);
}

// ========== std::shared_ptr 与 Qt ==========
void sharedPtrExample() {
    // 共享所有权
    std::shared_ptr<QWidget> widget = std::make_shared<QWidget>();
    std::shared_ptr<QWidget> widget2 = widget;  // 引用计数 +1

    qDebug() << "Use count:" << widget.use_count();  // 2

    // 使用 weak_ptr 避免循环引用
    std::weak_ptr<QWidget> weakWidget = widget;

    // 检查对象是否存活
    if (auto locked = weakWidget.lock()) {
        locked->setWindowTitle("Still alive");
    }

    widget.reset();
    widget2.reset();
    // weakWidget 现在过期
    if (weakWidget.expired()) {
        qDebug() << "Widget destroyed";
    }
}

// ========== Qt 智能指针 ==========
void qtSmartPointerExample() {
    // ========== QPointer ==========
    // 监控 QObject 指针，对象被删除时自动置空
    QPointer<QPushButton> ptr = new QPushButton();
    ptr->setText("QPointer Demo");

    if (ptr) {  // 检查对象是否仍然存在
        ptr->show();
    }

    delete ptr;  // 或对象被其他地方删除
    if (!ptr) {
        qDebug() << "QPointer is now null";
    }

    // ========== QSharedPointer ==========
    // 引用计数智能指针（类似 std::shared_ptr）
    QSharedPointer<QString> str = QSharedPointer<QString>::create("Hello");
    QSharedPointer<QString> str2 = str;  // 引用计数 +1

    qDebug() << "Ref count:" << str.use_count();  // 2

    // ========== QWeakPointer ==========
    // 弱引用，不影响引用计数
    QWeakPointer<QString> weak = str;
    if (!weak.isNull()) {
        QSharedPointer<QString> locked = weak.toStrongRef();
        if (locked) {
            qDebug() << *locked;  // 安全访问
        }
    }

    // ========== QScopedPointer ==========
    // 作用域指针（类似 std::unique_ptr）
    QScopedPointer<QTimer> timer(new QTimer());
    timer->start(1000);
    // 作用域结束时自动删除
}

// ========== 混合使用注意事项 ==========
void mixedPointers() {
    /*
     * 不要混用 new/delete 和智能指针
     * Qt 的父子对象管理不要与智能指针混用
     */

    // 错误：有父对象的指针不要用智能指针管理
    // QWidget* parent = new QWidget();
    // std::unique_ptr<QWidget> child(new QPushButton(parent));  // 会导致双重删除

    // 正确：只使用一种管理方式
    // 方式1：Qt 父子管理
    QWidget* parent = new QWidget();
    QPushButton* child = new QPushButton(parent);

    // 方式2：智能指针管理
    auto parent2 = std::make_unique<QWidget>();
    auto child2 = std::make_unique<QPushButton>(parent2.get());
}
```

### 12.4 Lambda 表达式与 STL 算法

```cpp
// ========== Lambda 表达式在 Qt 中的应用 ==========

void lambdaInQt() {
    // ========== 连接信号槽 ==========
    QPushButton* button = new QPushButton("Click me");

    // Lambda 作为槽（Qt5 风格）
    connect(button, &QPushButton::clicked, [=]() {
        qDebug() << "Button clicked";
    });

    // 捕获上下文
    QString message = "Hello";
    connect(button, &QPushButton::clicked, [message]() {
        qDebug() << message;  // 按值捕获
    });

    // 捕获 this
    class MyClass : public QObject {
    public:
        MyClass() {
            QPushButton* btn = new QPushButton();
            connect(btn, &QPushButton::clicked, this, [this]() {
                this->handleClick();
            });
        }

        void handleClick() {
            qDebug() << "Clicked";
        }
    };

    // ========== std::function 与 std::bind ==========

    // std::function 包装可调用对象
    std::function<QString(int)> formatter = [](int value) {
        return QString("Value: %1").arg(value);
    };
    qDebug() << formatter(42);

    // std::bind 绑定参数
    class Calculator {
    public:
        int add(int a, int b) { return a + b; }
    };

    Calculator calc;
    // 绑定第一个参数为 10
    auto addTen = std::bind(&Calculator::add, &calc, 10, std::placeholders::_1);
    qDebug() << "10 + 5 =" << addTen(5);

    // ========== Qt 并发 + Lambda ==========
    QVector<int> numbers = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};

    // 使用 QtConcurrent::map
    QtConcurrent::map(numbers, [](int& value) {
        value = value * value;
    }).waitForFinished();

    // 过滤
    auto future = QtConcurrent::filtered(numbers, [](int value) {
        return value > 25;
    });

    QList<int> result = future.results();
}
```

### 12.5 完整示例：Qt + STL

```cpp
// ========== 使用 STL 容器存储 Qt 对象 ==========

class ContactManager {
public:
    struct Contact {
        QString name;
        QString phone;
        QString email;
    };

    void addContact(const Contact& contact) {
        m_contacts.push_back(contact);
    }

    // 使用 STL 算法
    QVector<Contact> findByName(const QString& name) const {
        QVector<Contact> result;
        std::copy_if(m_contacts.begin(), m_contacts.end(),
                     std::back_inserter(result),
                     [&name](const Contact& c) {
            return c.name.contains(name, Qt::CaseInsensitive);
        });
        return result;
    }

    void sortByName() {
        std::sort(m_contacts.begin(), m_contacts.end(),
                  [](const Contact& a, const Contact& b) {
            return a.name < b.name;
        });
    }

    void removeDuplicates() {
        std::sort(m_contacts.begin(), m_contacts.end(),
                  [](const Contact& a, const Contact& b) {
            return a.phone < b.phone;
        });

        auto last = std::unique(m_contacts.begin(), m_contacts.end(),
                                [](const Contact& a, const Contact& b) {
            return a.phone == b.phone;
        });

        m_contacts.erase(last, m_contacts.end());
    }

private:
    QVector<Contact> m_contacts;
};

// ========== 使用 std::optional (C++17) ==========
#include <optional>

std::optional<int> parseNumber(const QString& str) {
    bool ok;
    int value = str.toInt(&ok);
    if (ok) {
        return value;
    }
    return std::nullopt;
}

void optionalExample() {
    auto result = parseNumber("42");
    if (result) {
        qDebug() << "Value:" << *result;
    } else {
        qDebug() << "Invalid number";
    }

    // 提供默认值
    int value = parseNumber("invalid").value_or(0);
}

// ========== 使用 std::variant (C++17) ==========
#include <variant>

using ConfigValue = std::variant<int, double, QString, bool>;

void printConfig(const ConfigValue& value) {
    std::visit([](auto&& arg) {
        using T = std::decay_t<decltype(arg)>;
        if constexpr (std::is_same_v<T, QString>) {
            qDebug() << "String:" << arg;
        } else if constexpr (std::is_same_v<T, int>) {
            qDebug() << "Int:" << arg;
        } else if constexpr (std::is_same_v<T, double>) {
            qDebug() << "Double:" << arg;
        } else if constexpr (std::is_same_v<T, bool>) {
            qDebug() << "Bool:" << arg;
        }
    }, value);
}

// ========== 使用 std::chrono 与 QTimer ==========
#include <chrono>

void chronoExample() {
    using namespace std::chrono;

    // 获取当前时间
    auto now = system_clock::now();
    auto time_t_now = system_clock::to_time_t(now);
    qDebug() << "Current time:" << ctime(&time_t_now);

    // 计算时间差
    auto start = high_resolution_clock::now();

    // 执行操作
    QThread::msleep(100);

    auto end = high_resolution_clock::now();
    auto duration = duration_cast<milliseconds>(end - start);
    qDebug() << "Duration:" << duration.count() << "ms";
}
```

---

## 十三、常见面试问题汇总 ⭐

### 13.1 信号槽相关问题

**Q1: 信号槽的连接方式有哪些？区别是什么？**

```
Qt::AutoConnection（默认）：
  - 同线程：DirectConnection（直接调用）
  - 跨线程：QueuedConnection（队列调用）

Qt::DirectConnection：
  - 槽函数在信号发射线程立即执行
  - 同步调用，信号发射后槽函数立即执行完

Qt::QueuedConnection：
  - 槽函数进入接收者线程的事件队列
  - 异步调用，信号发射后立即返回

Qt::BlockingQueuedConnection：
  - 信号发射线程会阻塞等待槽函数执行完成
  - 只能跨线程使用，同线程会死锁

Qt::UniqueConnection：
  - 防止重复连接
  - 可与其他标志位或运算
```

**Q2: 跨线程信号槽在哪個线程执行？**

```cpp
// 槽函数在接收者所在线程执行
// 信号放入接收者线程的事件队列

// 发送线程（主线程）
QThread* thread = new QThread;
Worker* worker = new Worker;
worker->moveToThread(thread);

// connect 默认使用 QueuedConnection
connect(sender, &Sender::dataChanged, worker, &Worker::processData);
// dataChanged 在 worker 线程执行
```

**Q3: 信号可以连接信号吗？**

```cpp
// 可以，信号可以连接到信号
connect(button, &QPushButton::clicked, this, &MyClass::buttonClicked);
connect(this, &MyClass::buttonClicked, this, &MyClass::handleAction);

// 信号转发
connect(sender, &Sender::valueChanged, this, &MyClass::relaySignal);
```

**Q4: 如何断开信号槽连接？**

```cpp
// 断开特定连接
disconnect(sender, &Sender::valueChanged, receiver, &Receiver::updateValue);

// 断开对象的所有连接
disconnect(sender);

// 使用连接对象
QMetaObject::Connection conn = connect(sender, &Sender::valueChanged, ...);
disconnect(conn);
```

**Q5: Lambda 表达式作为槽有什么注意事项？**

```cpp
// 捕获 this（推荐）
connect(button, &QPushButton::clicked, this, [this]() {
    this->handleClick();
});

// 捕获上下文变量
QString msg = "Hello";
connect(button, &QPushButton::clicked, [msg]() {
    qDebug() << msg;  // 按值捕获，安全
});

// 注意：不要捕获临时对象的指针
// 错误示例：
// QString* p = &QString("Hello");  // 临时对象
// connect(button, &QPushButton::clicked, [p]() { ... });  // 危险
```

### 13.2 事件处理相关问题

**Q1: Qt 事件处理顺序是什么？**

```
1. 事件被发送到 QObject::event()
2. event() 根据事件类型分发到具体事件处理器
3. 具体处理器：mousePressEvent、keyPressEvent 等
4. 如果事件未处理，传递给基类
5. 事件过滤器在事件到达对象之前拦截
```

**Q2: 事件过滤器和重写事件函数有什么区别？**

```cpp
// 重写事件函数：只能处理自己的事件
void MyWidget::mousePressEvent(QMouseEvent* event) {
    // 处理自己的鼠标事件
}

// 事件过滤器：可以监控其他对象的事件
bool MyWidget::eventFilter(QObject* watched, QEvent* event) {
    if (watched == lineEdit && event->type() == QEvent::KeyPress) {
        // 拦截 lineEdit 的键盘事件
        return true;
    }
    return QWidget::eventFilter(watched, event);
}

// 安装过滤器
lineEdit->installEventFilter(this);
```

**Q3: 如何发送自定义事件？**

```cpp
// 定义事件类型
const QEvent::Type MyEventType = static_cast<QEvent::Type>(QEvent::User + 100);

// 自定义事件类
class MyEvent : public QEvent {
public:
    MyEvent(const QString& data) : QEvent(MyEventType), m_data(data) {}
    QString data() const { return m_data; }
private:
    QString m_data;
};

// 发送事件
QApplication::postEvent(receiver, new MyEvent("Hello"));

// 处理事件
bool Receiver::event(QEvent* e) {
    if (e->type() == MyEventType) {
        MyEvent* myEvent = static_cast<MyEvent*>(e);
        qDebug() << myEvent->data();
        return true;
    }
    return QObject::event(e);
}
```

**Q4: accept() 和 ignore() 的区别？**

```cpp
void MyWidget::closeEvent(QCloseEvent* event) {
    if (hasUnsavedChanges()) {
        event->ignore();  // 忽略事件，窗口不关闭
        askSave();
    } else {
        event->accept();  // 接受事件，窗口关闭
    }
}

// ignore() 后事件继续传递给父对象
// accept() 后事件停止传递
```

### 13.3 内存管理相关问题

**Q1: Qt 对象树机制如何工作？**

```cpp
// 每个 QObject 可以有父对象
// 父对象销毁时，自动销毁所有子对象

QWidget* parent = new QWidget();
QPushButton* button = new QPushButton(parent);
QLabel* label = new QLabel(parent);

// 只需要删除 parent
delete parent;  // button 和 label 自动删除

// 注意：不要手动 delete 有父对象的子对象
```

**Q2: deleteLater() 和 delete 有什么区别？**

```cpp
// delete：立即删除
delete object;

// deleteLater：延迟删除
// 对象在当前事件循环结束后删除
// 安全：可以在槽函数中调用
// 槽函数执行完后才真正删除
object->deleteLater();

// 适用于：
// 1. 在对象自己的槽函数中删除自己
// 2. 不确定对象是否还在使用
connect(socket, &QTcpSocket::disconnected, socket, &QTcpSocket::deleteLater);
```

**Q3: 如何避免内存泄漏？**

```cpp
// 1. 使用对象树
QWidget* parent = new QWidget();
new QPushButton(parent);  // 自动管理

// 2. 使用智能指针
QPointer<QPushButton> ptr = new QPushButton();
// 对象被删除时 ptr 自动置空

// 3. 连接 finished 信号
QThread* thread = new QThread;
connect(thread, &QThread::finished, thread, &QThread::deleteLater);

// 4. 检查对象树
widget->dumpObjectInfo();
widget->dumpObjectTree();
```

### 13.4 多线程相关问题

**Q1: QThread::run() 和 moveToThread() 有什么区别？**

```cpp
// 方式1：继承 QThread，重写 run()
class MyThread : public QThread {
    void run() override {
        // 代码在新线程执行
        // this->thread() 在新线程
    }
};

// 方式2：moveToThread
class Worker : public QObject {
    Q_OBJECT
public slots:
    void doWork() {
        // 槽函数在 worker 所在线程执行
    }
};

QThread* thread = new QThread;
Worker* worker = new Worker;
worker->moveToThread(thread);

// 推荐 moveToThread：
// - 更灵活
// - Worker 可以有多个槽函数
// - 可以控制工作线程
```

**Q2: 为什么不能在子线程中操作 UI？**

```
原因：
1. UI 操作不是线程安全的
2. UI 事件循环在主线程
3. GDK/X11/Windows API 都是单线程的

解决方案：
1. 通过信号槽跨线程通信
2. 使用 QMetaObject::invokeMethod
3. 使用 QMetaObject::invokeMethod(..., Qt::QueuedConnection)
```

```cpp
// 子线程更新 UI 的正确方式
class Worker : public QObject {
    Q_OBJECT
public slots:
    void doWork() {
        // 执行耗时操作
        QString result = heavyWork();

        // 通过信号更新 UI
        emit workFinished(result);
    }

signals:
    void workFinished(const QString& result);
};

// 主线程连接信号
connect(worker, &Worker::workFinished, this, [this](const QString& result) {
    label->setText(result);  // 在主线程执行
});
```

**Q3: 如何正确停止线程？**

```cpp
class StoppableThread : public QThread {
public:
    void stop() {
        requestInterruption();   // 请求中断
        quit();                  // 退出事件循环
        wait();                  // 等待线程结束
    }

protected:
    void run() override {
        while (!isInterruptionRequested()) {
            // 定期检查中断
            doWork();
        }
    }
};

// 使用 moveToThread 方式
class Worker : public QObject {
    Q_OBJECT
public:
    void stop() {
        QThread::currentThread()->requestInterruption();
    }
};

// 停止时
worker->stop();
thread->quit();
thread->wait();
```

### 13.5 Model/View 架构相关问题

**Q1: 什么时候用 Model/View，什么时候用 Widget 类？**

```
Widget 类（QListWidget、QTreeWidget、QTableWidget）：
  - 数据量小（< 1000 项）
  - 简单的列表/树/表格显示
  - 快速开发

Model/View（QListView、QTreeView、QTableView + Model）：
  - 数据量大（> 1000 项）
  - 复杂的数据结构
  - 需要自定义显示/编辑
  - 多个视图共享同一数据
  - 更好的性能
```

**Q2: 如何实现自定义 Model？**

```cpp
// 最少需要重写 5 个函数
class CustomModel : public QAbstractListModel {
    Q_OBJECT
public:
    // 1. 返回行数
    int rowCount(const QModelIndex& parent = QModelIndex()) const override {
        return m_data.size();
    }

    // 2. 返回数据
    QVariant data(const QModelIndex& index, int role = Qt::DisplayRole) const override {
        if (!index.isValid())
            return QVariant();
        return m_data.at(index.row());
    }

    // 3. 返回表头
    QVariant headerData(int section, Qt::Orientation orientation, int role) const override {
        if (role == Qt::DisplayRole)
            return QString("Column %1").arg(section);
        return QVariant();
    }

    // 4. 返回标志
    Qt::ItemFlags flags(const QModelIndex& index) const override {
        return QAbstractListModel::flags(index) | Qt::ItemIsEditable;
    }

    // 5. 设置数据
    bool setData(const QModelIndex& index, const QVariant& value, int role) override {
        if (role == Qt::EditRole && index.isValid()) {
            m_data[index.row()] = value.toString();
            emit dataChanged(index, index);
            return true;
        }
        return false;
    }

private:
    QStringList m_data;
};
```

**Q3: Model 的 beginInsertRows 和 endInsertRows 作用？**

```cpp
// 通知视图数据将要变化/已经变化
// 必须成对调用

void Model::addItem(const QString& item) {
    beginInsertRows(QModelIndex(), m_data.size(), m_data.size());
    m_data.append(item);
    endInsertRows();  // 视图会自动更新显示新行
}

void Model::removeItem(int row) {
    beginRemoveRows(QModelIndex(), row, row);
    m_data.removeAt(row);
    endRemoveRows();  // 视图会自动更新
}

// 批量修改
void Model::updateAll() {
    beginResetModel();
    m_data.clear();
    // 重新填充数据
    endResetModel();
}
```

### 13.6 编译与构建相关问题

**Q1: moc 是什么？为什么需要它？**

```
moc（Meta-Object Compiler）是 Qt 的元对象编译器

作用：
1. 处理 Q_OBJECT 宏
2. 为信号生成实现代码
3. 生成 QMetaObject 元对象数据
4. 处理 Q_PROPERTY、Q_ENUM 等宏

为什么需要：
- C++ 没有反射机制
- Qt 需要在运行时获取类信息
- 信号槽机制需要元对象系统

生成的文件：
- moc_*.cpp 文件
- 自动加入编译系统
```

**Q2: qmake 和 CMake 有什么区别？**

| 特性 | qmake | CMake |
|------|-------|-------|
| 语法 | 简单 | 复杂 |
| Qt 专用 | 是 | 否 |
| 跨平台 | 是 | 是 |
| 现代化 | 较老 | 较新 |
| IDE 支持 | Qt Creator | 广泛 |
| 社区 | Qt 为主 | 通用 |

**Q3: .pro 文件常用变量？**

```qmake
TEMPLATE     # 模板类型（app/lib/subdirs）
TARGET       # 目标文件名
QT           # Qt 模块（core gui widgets）
SOURCES      # 源文件
HEADERS      # 头文件
FORMS        # UI 文件
RESOURCES    # 资源文件
LIBS         # 链接库
INCLUDEPATH  # 包含路径
DEFINES      # 预定义宏
CONFIG       # 编译配置
```

---

## 十四、调试技巧

### 14.1 qDebug 日志输出

```cpp
// ========== 基本日志输出 ==========

// qDebug - 调试信息
qDebug() << "Debug message";
qDebug() << "Value:" << 42;
qDebug() << "String:" << QString("Hello");

// qWarning - 警告信息
qWarning() << "Warning message";

// qCritical - 严重错误
qCritical() << "Critical error";

// qFatal - 致命错误（会终止程序）
// qFatal("Fatal error: %s", "something");

// ========== 格式化输出 ==========

int value = 42;
QString name = "Qt";

// Qt5 风格
qDebug() << "Value:" << value << "Name:" << name;

// printf 风格（不推荐，类型不安全）
qDebug("Value: %d, Name: %s", value, name.toLocal8Bit().data());

// ========== 条件输出 ==========
if (value > 100) {
    qDebug() << "Value is large:" << value;
}

// Qt 提供的条件宏
qDebug() << "This is always shown";

// ========== 日志级别控制 ==========

// 自定义日志函数
void myLog(QtMsgType type, const QMessageLogContext& context, const QString& msg) {
    QByteArray localMsg = msg.toLocal8Bit();
    const char* file = context.file ? context.file : "";
    const char* function = context.function ? context.function : "";

    switch (type) {
        case QtDebugMsg:
            fprintf(stderr, "[DEBUG] %s (%s:%u, %s)\n",
                    localMsg.constData(), file, context.line, function);
            break;
        case QtWarningMsg:
            fprintf(stderr, "[WARN] %s (%s:%u, %s)\n",
                    localMsg.constData(), file, context.line, function);
            break;
        case QtCriticalMsg:
            fprintf(stderr, "[ERROR] %s (%s:%u, %s)\n",
                    localMsg.constData(), file, context.line, function);
            break;
        case QtFatalMsg:
            fprintf(stderr, "[FATAL] %s (%s:%u, %s)\n",
                    localMsg.constData(), file, context.line, function);
            abort();
    }
}

// 安装日志处理器
qInstallMessageHandler(myLog);

// ========== 输出到文件 ==========

void logToFile() {
    QFile file("app.log");
    if (file.open(QIODevice::Append | QIODevice::Text)) {
        QTextStream out(&file);
        out << QDateTime::currentDateTime().toString("yyyy-MM-dd hh:mm:ss")
            << " - " << "Log message" << "\n";
    }
}

// ========== 分类日志 ==========
// 使用 qCategorizedDebug（Qt5.2+）
Q_LOGGING_CATEGORY(networkLog, "network")
Q_LOGGING_CATEGORY(uiLog, "ui")

void myFunction() {
    qCDebug(networkLog) << "Network operation";
    qCWarning(uiLog) << "UI warning";
}

// 运行时设置日志过滤
// QT_LOGGING_RULES="network.debug=false;ui.debug=true"
```

### 14.2 dumpObjectInfo / dumpObjectTree

```cpp
// ========== 对象信息调试 ==========

void debugObjectTree() {
    QWidget* widget = new QWidget();
    widget->setObjectName("mainWidget");

    QPushButton* button = new QPushButton(widget);
    button->setObjectName("okButton");

    QLabel* label = new QLabel(widget);
    label->setObjectName("statusLabel");

    // ========== dumpObjectInfo ==========
    // 输出对象信号连接信息
    widget->dumpObjectInfo();

    /* 输出示例：
    OBJECT QWidget::mainWidget
      SIGNALS OUT
        signal: destroyed(QObject*)
      SIGNALS IN
        (none)
    */

    // ========== dumpObjectTree ==========
    // 输出对象树结构
    widget->dumpObjectTree();

    /* 输出示例：
    QWidget::mainWidget
        QPushButton::okButton
        QLabel::statusLabel
    */

    // ========== 手动遍历对象树 ==========
    void printObjectTree(QObject* obj, int depth = 0) {
        qDebug() << QString(depth * 4, ' ')
                 << obj->metaObject()->className()
                 << "::" << obj->objectName();

        foreach (QObject* child, obj->children()) {
            printObjectTree(child, depth + 1);
        }
    }

    printObjectTree(widget);
}

// ========== 查找子对象 ==========

void findChildrenExample() {
    QWidget* parent = new QWidget();

    QPushButton* button1 = new QPushButton(parent);
    button1->setObjectName("button1");

    QPushButton* button2 = new QPushButton(parent);
    button2->setObjectName("button2");

    // 查找指定名称的子对象
    QPushButton* button = parent->findChild<QPushButton*>("button1");
    if (button) {
        qDebug() << "Found button1";
    }

    // 查找所有 QPushButton
    QList<QPushButton*> buttons = parent->findChildren<QPushButton*>();
    qDebug() << "Found" << buttons.size() << "buttons";
}
```

### 14.3 Qt Creator 调试器使用

```cpp
// ========== 断点调试 ==========

void debugExample() {
    int value = 0;

    // 在这里设置断点：按 F9 或点击行号
    value = 42;  // <- 断点

    // 条件断点
    for (int i = 0; i < 100; ++i) {
        value += i;
        // 右键断点 -> 设置条件：i == 50
    }

    // ========== 查看变量 ==========
    QString text = "Hello";
    QList<int> numbers = {1, 2, 3};

    // 在调试器中查看：
    // - 局部变量窗口
    // - 监视窗口（添加表达式）
    // - 鼠标悬停变量
}

// ========== 表达式求值 ==========

void expressionEvaluation() {
    QString str = "Hello World";

    // 调试器中可以求值：
    // str.length() -> 11
    // str.toUpper() -> "HELLO WORLD"
    // str.split(" ") -> QStringList
}

// ========== 内存查看 ==========

struct MyStruct {
    int a;
    double b;
    QString c;
};

void memoryDebug() {
    MyStruct s = {42, 3.14, "Hello"};

    // 在内存窗口查看：
    // &s -> 查看内存地址
    // sizeof(MyStruct) -> 查看大小
}

// ========== 调试宏 ==========

// 自定义调试宏
#ifdef QT_DEBUG
    #define DBG(x) qDebug() << #x << "=" << (x)
    #define DBG_EXPR(x) qDebug() << #x
#else
    #define DBG(x)
    #define DBG_EXPR(x)
#endif

void useDebugMacro() {
    int value = 42;
    DBG(value);        // 输出: value = 42
    DBG_EXPR(value);   // 输出: value
}

// ========== 断言 ==========

void assertionExample() {
    // Q_ASSERT（仅在 Debug 模式有效）
    Q_ASSERT(1 + 1 == 2);

    // Q_ASSERT_X（带消息）
    Q_ASSERT_X(1 + 1 == 2, "math", "one plus one should equal two");

    // Q_VERIFY（总是执行，用于有副作用的检查）
    bool ok = false;
    Q_VERIFY(doSomething(&ok));  // doSomething 总是执行

    // 静态断言（编译时检查）
    static_assert(sizeof(int) == 4, "int must be 4 bytes");
}

bool doSomething(bool* ok) {
    *ok = true;
    return true;
}
```

---

## 十五、Qt 特有数据类型与工具类

### 15.1 基本数据类型

```cpp
// ========== Qt 整数类型（跨平台固定大小）==========

void integerTypes() {
    // 有符号整数
    qint8 i8 = -128;             // 8位，-128 ~ 127
    qint16 i16 = -32768;         // 16位，-32768 ~ 32767
    qint32 i32 = -2147483648;    // 32位
    qint64 i64 = -9223372036854775808LL;  // 64位

    // 无符号整数
    quint8 ui8 = 255;            // 8位，0 ~ 255
    quint16 ui16 = 65535;        // 16位，0 ~ 65535
    quint32 ui32 = 4294967295;   // 32位
    quint64 ui64 = 18446744073709551615ULL;  // 64位

    // 其他类型
    qlonglong ll = 123456789012LL;
    qulonglong ull = 123456789012ULL;
    qreal r = 3.14;              // double（默认）
    qreal r2 = 3.14f;            // float（某些平台）

    // 指针整数类型（足够存放指针）
    quintptr ptr = 0;
    qintptr iptr = 0;
}

// ========== 类型检查 ==========

void typeChecking() {
    qDebug() << "sizeof(qint8):" << sizeof(qint8);    // 1
    qDebug() << "sizeof(qint16):" << sizeof(qint16);  // 2
    qDebug() << "sizeof(qint32):" << sizeof(qint32);  // 4
    qDebug() << "sizeof(qint64):" << sizeof(qint64);  // 8
    qDebug() << "sizeof(quintptr):" << sizeof(quintptr);  // 4或8
}
```

### 15.2 字符串处理

```cpp
// ========== QString ==========

void stringFunctions() {
    QString str = "Hello World";

    // ========== 大小写转换 ==========
    QString upper = str.toUpper();      // "HELLO WORLD"
    QString lower = str.toLower();      // "hello world"

    // ========== 查找 ==========
    int index = str.indexOf("World");        // 6
    int lastIndexOf = str.lastIndexOf("l");  // 9
    bool contains = str.contains("World");   // true
    bool starts = str.startsWith("Hello");   // true
    bool ends = str.endsWith("World");       // true

    // ========== 替换 ==========
    QString replaced = str;
    replaced.replace("World", "Qt");  // "Hello Qt"

    // 分割
    QStringList parts = str.split(" ");
    // ["Hello", "World"]

    // 连接
    QString joined = parts.join(", ");  // "Hello, World"

    // ========== 截取 ==========
    QString left = str.left(5);        // "Hello"
    QString right = str.right(5);      // "World"
    QString mid = str.mid(6, 5);       // "World"

    // ========== 去空格 ==========
    QString withSpaces = "  Hello  ";
    QString trimmed = withSpaces.trimmed();      // "Hello"
    QString simplified = withSpaces.simplified(); // "Hello"

    // ========== 填充 ==========
    QString filled = str.leftJustified(20, '-');  // "Hello World---------"
    QString filled2 = str.rightJustified(20, '-'); // "---------Hello World"

    // ========== 重复 ==========
    QString repeated = QString("Hi ").repeated(3);  // "Hi Hi Hi "
}

// ========== QByteArray ==========

void byteFunctions() {
    // ========== 字节数组 ==========
    QByteArray bytes = "Hello";

    // 添加数据
    bytes.append(" World");

    // 大小
    int size = bytes.size();  // 11

    // 转换为十六进制
    QByteArray hex = bytes.toHex();  // "48656c6c6f"

    // 从十六进制解析
    QByteArray fromHex = QByteArray::fromHex("48656c6c6f");

    // Base64 编码/解码
    QByteArray base64 = bytes.toBase64();
    QByteArray decoded = QByteArray::fromBase64(base64);

    // ========== 压缩 ==========
    QByteArray data = "lots of data...";
    QByteArray compressed = qCompress(data);
    QByteArray uncompressed = qUncompress(compressed);

    // ========== 数字转换 ==========
    QByteArray numStr = QByteArray::number(42);       // "42"
    QByteArray numStr2 = QByteArray::number(3.14);    // "3.14"

    int num = numStr.toInt();           // 42
    double num2 = numStr2.toDouble();   // 3.14
}

// ========== QLatin1String ==========

void latinString() {
    // 避免临时 QString 创建
    QString str = "Hello";

    // 使用 QLatin1String 比较更高效
    if (str == QLatin1String("Hello")) {
        // 不会创建临时 QString
    }

    // 自定义字面量
    using namespace QtLiterals;
    QString str2 = "Hello"_L1;
}
```

### 15.3 容器类

```cpp
// ========== QList ==========

void listFunctions() {
    QList<int> list;

    // 添加
    list.append(1);
    list.prepend(0);
    list << 2 << 3;  // 链式添加

    // 插入
    list.insert(2, 99);

    // 访问
    int first = list.first();
    int last = list.last();
    int value = list.at(2);

    // 删除
    list.removeAt(2);
    list.removeOne(3);
    list.removeAll(0);

    // 查找
    bool contains = list.contains(1);
    int index = list.indexOf(2);

    // 遍历
    for (int i = 0; i < list.size(); ++i) {
        qDebug() << list[i];
    }

    QListIterator<int> it(list);
    while (it.hasNext()) {
        qDebug() << it.next();
    }
}

// ========== QMap ==========

void mapFunctions() {
    QMap<QString, int> map;

    // 插入
    map["key1"] = 1;
    map.insert("key2", 2);
    map.insertMulti("key1", 3);  // 允许重复键

    // 访问
    int value = map.value("key1", -1);  // 默认值 -1

    // 查找
    bool contains = map.contains("key1");
    QList<QString> keys = map.keys();
    QList<int> values = map.values();

    // 删除
    map.remove("key1");

    // QMultiMap（允许重复键）
    QMultiMap<QString, int> multiMap;
    multiMap.insert("key", 1);
    multiMap.insert("key", 2);
    QList<int> valuesForKey = multiMap.values("key");  // [1, 2]
}

// ========== QHash ==========

void hashFunctions() {
    QHash<QString, int> hash;

    // 使用方式与 QMap 相同
    hash["key1"] = 1;
    hash["key2"] = 2;

    // QHash 特点：
    // - 查找更快（平均 O(1)）
    // - 不排序
    // - 键必须提供 operator==()

    // QMap 特点：
    // - 查找 O(log n)
    // - 键排序
}

// ========== QSet ==========

void setFunctions() {
    QSet<int> set;

    // 插入
    set.insert(1);
    set << 2 << 3 << 1;  // 1 只会存储一次

    // 查找
    bool contains = set.contains(2);

    // 删除
    set.remove(1);

    // 集合操作
    QSet<int> set1 = {1, 2, 3};
    QSet<int> set2 = {2, 3, 4};

    QSet<int> united = set1.unite(set2);        // {1, 2, 3, 4}
    QSet<int> intersected = set1.intersect(set2);  // {2, 3}
}
```

### 15.4 日期时间

```cpp
#include <QDateTime>
#include <QDate>
#include <QTime>

void dateTimeFunctions() {
    // ========== 当前时间 ==========
    QDateTime now = QDateTime::currentDateTime();
    QDate currentDate = QDate::currentDate();
    QTime currentTime = QTime::currentTime();

    qDebug() << "Now:" << now;
    qDebug() << "Date:" << currentDate;
    qDebug() << "Time:" << currentTime;

    // ========== 格式化输出 ==========
    QString formatted = now.toString("yyyy-MM-dd hh:mm:ss");
    // "2024-01-15 14:30:25"

    QString isoFormat = now.toString(Qt::ISODate);
    // "2024-01-15T14:30:25"

    // ========== 解析字符串 ==========
    QDateTime parsed = QDateTime::fromString("2024-01-15 14:30:25", "yyyy-MM-dd hh:mm:ss");

    // ========== 日期计算 ==========
    QDate date = QDate(2024, 1, 15);
    QDate tomorrow = date.addDays(1);
    QDate nextMonth = date.addMonths(1);
    QDate nextYear = date.addYears(1);

    // ========== 时间差 ==========
    QDateTime dt1 = QDateTime::currentDateTime();
    QThread::sleep(1);
    QDateTime dt2 = QDateTime::currentDateTime();

    qint64 msecs = dt1.msecsTo(dt2);  // 毫秒差
    qint64 secs = dt1.secsTo(dt2);    // 秒差
    qint64 days = dt1.daysTo(dt2);    // 天差

    // ========== 时间戳 ==========
    qint64 timestamp = now.toSecsSinceEpoch();  // Unix 时间戳（秒）

    // 从时间戳创建
    QDateTime fromTimestamp = QDateTime::fromSecsSinceEpoch(timestamp);

    // ========== UTC 时间 ==========
    QDateTime utc = QDateTime::currentDateTimeUtc();
    QDateTime local = utc.toLocalTime();

    // ========== 时区 ==========
    QDateTime timeZone = QDateTime::currentDateTime();
    timeZone.setTimeZone(QTimeZone("Asia/Shanghai"));
}
```

### 15.5 文件与路径

```cpp
#include <QDir>
#include <QFileInfo>

void pathFunctions() {
    // ========== 路径拼接 ==========
    QString path = QDir::homePath();  // 用户主目录
    QString filePath = path + "/Documents/file.txt";
    QString filePath2 = QDir(path).filePath("Documents/file.txt");

    // ========== 目录操作 ==========
    QDir dir;

    // 创建目录
    dir.mkpath("parent/child");  // 创建多级目录

    // 判断目录是否存在
    if (dir.exists("myfolder")) {
        qDebug() << "Folder exists";
    }

    // 遍历目录
    QDir folder("/path/to/folder");
    QStringList filters;
    filters << "*.txt" << "*.md";
    folder.setNameFilters(filters);

    QFileInfoList fileList = folder.entryInfoList(QDir::Files);
    for (const QFileInfo& fileInfo : fileList) {
        qDebug() << fileInfo.fileName() << fileInfo.size();
    }

    // ========== QFileInfo ==========
    QFileInfo info("/path/to/file.txt");

    qDebug() << "Exists:" << info.exists();
    qDebug() << "Size:" << info.size();
    qDebug() << "Created:" << info.birthTime();
    qDebug() << "Modified:" << info.lastModified();
    qDebug() << "Readable:" << info.isReadable();
    qDebug() << "Writable:" << info.isWritable();
    qDebug() << "Absolute path:" << info.absoluteFilePath();
    qDebug() << "Base name:" << info.baseName();
    qDebug() << "Suffix:" << info.suffix();
    qDebug() << "Complete suffix:" << info.completeSuffix();

    // ========== 路径工具 ==========
    QString cleanPath = QDir::cleanPath("/path/to/../file.txt");  // "/path/file.txt"

    // ========== 临时文件 ==========
    QTemporaryFile tempFile;
    if (tempFile.open()) {
        qDebug() << "Temp file:" << tempFile.fileName();
        // 使用后自动删除
    }

    // 临时目录
    QTemporaryDir tempDir;
    if (tempDir.isValid()) {
        QString tempPath = tempDir.path();
        // 使用后自动删除
    }

    // ========== 标准路径 ==========
    QString home = QDir::homePath();
    QString root = QDir::rootPath();
    QString temp = QDir::tempPath();
    QString current = QDir::currentPath();

    // Qt5.4+ 标准位置
    QString desktop = QStandardPaths::writableLocation(QStandardPaths::DesktopLocation);
    QString documents = QStandardPaths::writableLocation(QStandardPaths::DocumentsLocation);
    QString appData = QStandardPaths::writableLocation(QStandardPaths::AppDataLocation);
}
```

---

## 附录：快速参考

### A.1 常用头文件列表

```cpp
// ========== 核心 ==========
#include <QObject>          // 对象基类
#include <QCoreApplication> // 应用程序类
#include <QDebug>           // 调试输出
#include <QTimer>           // 定时器
#include <QThread>          // 线程
#include <QMutex>           // 互斥锁
#include <QFile>            // 文件操作

// ========== GUI ==========
#include <QWidget>          // 窗口基类
#include <QPushButton>      // 按钮
#include <QLabel>           // 标签
#include <QLineEdit>        // 单行输入
#include <QTextEdit>        // 多行输入
#include <QCheckBox>        // 复选框
#include <QRadioButton>     // 单选框
#include <QComboBox>        // 下拉框
#include <QSlider>          // 滑块
#include <QProgressBar>     // 进度条

// ========== 布局 ==========
#include <QVBoxLayout>      // 垂直布局
#include <QHBoxLayout>      // 水平布局
#include <QGridLayout>      // 网格布局
#include <QFormLayout>      // 表单布局
#include <QSplitter>        // 分割器

// ========== 容器控件 ==========
#include <QListWidget>      // 列表
#include <QTreeWidget>      // 树形
#include <QTableWidget>     // 表格

// ========== 高级控件 ==========
#include <QTabWidget>       // 标签页
#include <QStackedWidget>   // 堆栈窗口
#include <QGroupBox>        // 组框
#include <QScrollArea>      // 滚动区域
#include <QMainWindow>      // 主窗口
#include <QDialog>          // 对话框
#include <QMessageBox>      // 消息框
#include <QFileDialog>      // 文件对话框
#include <QInputDialog>     // 输入对话框

// ========== Model/View ==========
#include <QListView>        // 列表视图
#include <QTreeView>        // 树形视图
#include <QTableView>       // 表格视图
#include <QAbstractListModel>
#include <QAbstractTableModel>
#include <QStringListModel>

// ========== 图形 ==========
#include <QPainter>         // 绘图
#include <QPainterPath>     // 绘图路径
#include <QImage>           // 图片
#include <QPixmap>          // 像素图
#include <QPicture>         // 图片
#include <QIcon>            // 图标
#include <QBrush>           // 画刷
#include <QPen>             // 画笔
#include <QColor>           // 颜色
#include <QFont>            // 字体
#include <QRegion>          // 区域

// ========== 图形视图框架 ==========
#include <QGraphicsScene>   // 场景
#include <QGraphicsView>    // 视图
#include <QGraphicsItem>    // 图形项
#include <QGraphicsRectItem>
#include <QGraphicsEllipseItem>
#include <QGraphicsPixmapItem>

// ========== 图表 ==========
#include <QtCharts>         // 图表模块
#include <QChart>
#include <QLineSeries>
#include <QBarSeries>
#include <QPieSeries>

// ========== 网络 ==========
#include <QTcpSocket>       // TCP 套接字
#include <QTcpServer>       // TCP 服务器
#include <QUdpSocket>       // UDP 套接字
#include <QNetworkAccessManager>
#include <QNetworkRequest>
#include <QNetworkReply>

// ========== 数据 ==========
#include <QString>          // 字符串
#include <QStringList>      // 字符串列表
#include <QByteArray>       // 字节数组
#include <QVariant>         // 变体
#include <QVector>          // 向量
#include <QList>            // 列表
#include <QMap>             // 映射
#include <QHash>            // 哈希
#include <QSet>             // 集合
#include <QPair>            // 对

// ========== 日期时间 ==========
#include <QDateTime>        // 日期时间
#include <QDate>            // 日期
#include <QTime>            // 时间

// ========== 文件 ==========
#include <QDir>             // 目录
#include <QFileInfo>        // 文件信息
#include <QFile>            // 文件
#include <QTemporaryFile>   // 临时文件
#include <QStandardPaths>   // 标准路径

// ========== 设置 ==========
#include <QSettings>        // 配置文件

// ========== JSON/XML ==========
#include <QJsonDocument>    // JSON 文档
#include <QJsonObject>      // JSON 对象
#include <QJsonArray>       // JSON 数组
#include <QDomDocument>     // XML 文档

// ========== 资源 ==========
#include <QResource>        // 资源
```

### A.2 .pro 文件模板

```qmake
# ========== 应用程序模板 ==========
TEMPLATE = app
TARGET = MyApp
QT += core gui widgets
CONFIG += c++17

SOURCES += \
    main.cpp \
    mainwindow.cpp

HEADERS += \
    mainwindow.h

FORMS += \
    mainwindow.ui

RESOURCES += \
    resources.qrc

# ========== 库模板 ==========
TEMPLATE = lib
TARGET = MyLib
QT += core gui
CONFIG += staticlib  # 或 dll

SOURCES += \
    mylib.cpp

HEADERS += \
    mylib.h

# ========== 子目录模板 ==========
TEMPLATE = subdirs
SUBDIRS = app lib1 lib2

app.depends = lib1 lib2
```

### A.3 CMakeLists.txt 模板

```cmake
cmake_minimum_required(VERSION 3.16)
project(MyApp VERSION 1.0.0 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_AUTOMOC ON)
set(CMAKE_AUTORCC ON)
set(CMAKE_AUTOUIC ON)

find_package(Qt6 REQUIRED COMPONENTS Core Widgets)

qt_add_executable(MyApp
    main.cpp
    mainwindow.cpp
    mainwindow.h
    mainwindow.ui
    resources.qrc
)

target_link_libraries(MyApp PRIVATE
    Qt6::Core
    Qt6::Widgets
)
```

### A.4 QSS 常用属性速查

```css
/* ========== 通用属性 ========== */
color                    /* 文字颜色 */
background-color        /* 背景颜色 */
border                  /* 边框 */
border-radius          /* 圆角 */
padding                /* 内边距 */
margin                 /* 外边距 */
font                   /* 字体 */
font-size              /* 字号 */
font-weight            /* 字重 */

/* ========== 伪状态 ========== */
:hover                 /* 鼠标悬停 */
:pressed              /* 按下状态 */
:focus                /* 焦点状态 */
:checked              /* 选中状态 */
:selected             /* 选择状态 */
:disabled             /* 禁用状态 */
:enabled              /* 启用状态 */
:!hover               /* 非悬停 */

/* ========== 子控件 ========== */
::down-arrow           /* 下拉箭头 */
::drop-down           /* 下拉区域 */
::item                /* 列表项 */
::branch              /* 树形分支 */
::handle              /* 滑块句柄 */
::groove              /* 滑块槽 */
::chunk               /* 进度块 */
::title               /* 标题 */
::indicator           /* 指示器 */
::scroller            /* 滚动条 */
::section             /* 头部节 */
```

---

**完**

*本文档涵盖了 Qt 面试的核心知识点，建议结合实际项目经验深入理解。*

