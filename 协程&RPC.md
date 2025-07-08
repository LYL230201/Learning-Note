# 架构

一个主线程为**主reactor**，只负责接收新连接，并将创建好的来凝结分发给其他线程去处理

每个IO线程为一个**从reactor**，收到连接后，把连接放在自己的epoll上

一个客户端连接对应一个**协程**，实现yeild和resume



# RPC

## 协议设计

### 存在的问题

- 长度冗余，了解一下TLV(Type-Length-Value)格式优化
- 校验和不如CRC32，且没有加密机制
- 缺乏版本号，不利于协议升级和向后兼容
- 优化字段对齐，减少内存占用

## 序列化与反序列化的实现

### 序列化的必要性

一台服务器调用另一台服务器的服务时，需要传入一个**对象参数**，对方返回的也是一个对象参数（比如一个**struct结构体的引用**，里面有价格，销量等信息）。

**对象无法像字符串和数字那样直接传输，需要转化为字节流**。这就是序列化，反过程为反序列化。

protobuf提供了一种序列化反序列化方法。把刚才远程调用用到的两个对象写成message格式：

```C++
message QueryReq {
  string date;
};

message QueryRes {
  int32 list_id;
  int32 price;
  string time;
}
```

然后使用 **protoc命令**，他会自动生成类 **QueryReq** 以及类 **QueryRes**。这两个类是继承 **google::protobuf::Message**的，因此可以直接调用其成员函数进行序列化和反序列化。

注意，**由于UserServer 和OrderServer都需要这两个类，因此双方都需要维持同一个 proto文件。**

Protobuf只是序列化的方案之一，当然也有其他的方案，如使用**json**等。不过 protobuf 具有**轻量化、快速**等优点，通常是第一选择。

### 具体序列化过程



## 编码与解码

### 需要编解码的原因

光把参数对象传输过去是不够的，首先是要**保证对端接收后顺利读取数据**并进行处理，另外还要把**方法名**发过去对端才知道怎么处理。

因此，需要**自定义协议**，进行**数据包的制造和拆分**。这个过程称为编码和解码。

### 编解码的具体实现



## RPC调用的完整流程

### 客户端：

1.创建服务器的地址

2.创建RPC通道（channel）

> [!NOTE]
>
> ###### 通道的作用？
>
> 通道是**客户端和服务器之间通信的抽象**，主要负责：1）消息的序列化，2）编码消息，3）发送并接收响应，4）反序列化响应
>
> 另外还有网络连接的管理和错误处理

3.创建服务存根

> [!NOTE]
>
> ###### 服务存根（stub）的作用？
>
> 服务存根是由protobuf编译器自动生成的代码。stub里面有channel对象，客户端**通过stub调用相应的服务**
>
> **服务存根封装了RPC调用的细节**

### 服务端：

启动阶段：

1.整体**初始化阶段**（test_tinypb_server.cc）

```C++
int main(int argc, char* argv[]) {
  // 1. 配置文件检查
  if (argc != 2) {
    printf("Start TinyRPC server error, input argc is not 2!");
    return 0;
  }

  // 2. 初始化配置
  tinyrpc::InitConfig(argv[1]);

  // 3. 注册服务
  REGISTER_SERVICE(QueryServiceImpl);

  // 4. 启动RPC服务器
  tinyrpc::StartRpcServer();
  return 0;
}
```

2.**配置**初始化 (tinyrpc/comm/start.cc 和 tinyrpc/comm/config.cc)

- 首先**设置协程钩子**tinyrpc::**SetHook**(false);

- 然后**读取配置文件**gRpcConfig->**readConf**();

  - 执行**readLogConfig**读取日志配置，并创建了日志实例，执行init(初始化双缓冲区并注册各种信号处理函数)
  - 读取时间轮，协程，服务器配置（包括IP、端口、协议）
  - 初始化全局服务器实例gRpcServer = **std**::**make_shared**<**TcpServer**>(addr, protocol);
  
  > [!NOTE]
  >
  > ###### readConf函数的作用？
  >
  > 不仅读取了各种配置数据，还给start.cc文件创建的全局变量**gRpcLogger**和**gRpcServer**进行了**初始化**（通过make_shared），在此**TcpServer**的**构造函数被调用**，而在其构造函数中，进行了
  >
  > 1）**IO线程池**的初始化，
  >
  > 2）协议相关组件的初始化（即**m_dispatcher**和**m_codec**），
  >
  > 3）**主reactor**初始化，
  >
  > 4）**时间轮**初始化，
  >
  > 5）**清理定时器**初始化（时间轮负责检测，有超时连接就调用shutdownConnection()并把将连接状态设置为Closed，清理定时器负责定时检查连接状态，释放已关闭连接的资源，从m_clients移除无效连接）
  

3.**服务注册** (tinyrpc/net/tinypb/tinypb_rpc_dispatcher.cc)

- **REGISTER_SERVICE**(**QueryServiceImpl**);

  > [!NOTE]
  >
  > ###### 这个宏做了什么？
  >
  > 将QueryServiceImpl传入并进行注册：**tinyrpc**::**GetServer**()->registerService(std**::**make_shared**<**service>())
  >
  > 并在宏内部做了错误处理：输出日志并退出

  获取**服务器实例**后，调用**registerService**方法，获取m_dispatcher实例，将其调用其**registerService**方法，将服务注册到map中。其中键为service的全名，值为service的指针。

- QueryServiceImpl类继承了由protoc命令生成的pb文件中的**QueryService**类。重写了已有的两个服务函数的实现。

> [!NOTE]
>
> ###### **query_name**函数接收四个参数：
>
> - **控制器**作为RPC调用的控制中心，由**客户端发起调用时自动创建并传入**，服务端无需手动实例化。服务端可以利用controller来检查调用是否失败或标记错误
> - **传入参数和传出参数**均为客户端提供，其中传入参数服务端不能修改（const）
> - 第四个参数**Closure* done**，用于实现回调。再方法末尾调用done->Run()通知调用完成。客户端可以传**nullptr**作为done参数实现**同步调用**，也可以传入**回调函数**来处理RPC完成后的逻辑，**异步调用**）

4.**服务器启动**阶段 (tinyrpc/comm/start.cc 和 tinyrpc/net/tcp/tcp_server.cc)

main函数调用StartRpcServer函数：

- 调用**gRpcLogger->start**()来启动日志。具体而言就是**启动定时器**，定时器定期触发**loopFunc**，将缓冲区的日志交给**异步日志器**，异步日志器在**独立线程中写入文件**。
- 调用**gRpcServer->start**()启动服务器。服务器**start函数**内进行了各种初始化，包括
  - 创建和初始化Acceptor
  - 创建接受连接的协程（此处执行了**GetCoroutinePool**()进行了**协程池的初始化**）
  - 启动IO线程池——**m_io_pool->start()**
  - 启动主reactor（开启loop事件循环）

### 服务器启动阶段的初始化

#### **TcpServer**的**构造函数被调用**后：

##### 1.IO线程池的初始化

在readConf的末尾执行了TcpServer的构造函数，在其构造函数中首先创建了线程池

```C++
m_io_pool = std::make_shared<IOThreadPool>(gRpcConfig->m_iothread_num);
```

IO线程池的构造函数接收一个**size**(即线程池中的线程数)，首先执行**resize**将保存线程指针的数组**m_io_threads**大小进行调整。然后for循环创建线程，并调用**setThreadIndex**(i)设置线程id（m_index = index;此index即为传入的i）

> [!NOTE]
>
> ###### 线程池有哪些属性？
>
> - 一个保存线程指针的数组m_io_threads；
> - 线程池的大小size;
> - 一个原子计数器（atomic），记录线程id。用于需要选择一个IO线程来处理新连接时，通过自增该值并取模来选择，即**轮询**分配策略

###### 1.1线程

创建线程，即调用线程构造函数：

- 首先初始化m_init_semaphore信号量和m_start_semaphore信号量
- 调用**pthread_create**创建线程，线程函数是main函数
- 等待m_init_semaphore信号（main函数进行到某处会执行post）
- 等到信号后销毁m_init_semaphore信号量

> [!NOTE]
>
> ###### 使用m_init_semaphore信号的原因？
>
> 首先，main函数中有线程初始化相关的逻辑，IOTHread构造函数必须等待main函数中相关的逻辑执行完才能返回
>
> 其次，这部分逻辑不能放在IOThread构造函数中，因为这部分逻辑创建了**线程局部变量**，如果在构造函数中执行则是设置了主线程的变量，事与愿违。另外，**gettid**函数以及**协程的初始化**都需要在子线程中完成，不能放在构造函数中

###### 1.2线程函数main

传入参数为**该线程所属的IOThread对象指针**，只有IO线程有这个对象，主线程没有

主要步骤:

1）**创建Reactor**

```C++
t_reactor_ptr = new Reactor();
```

Reactor的构造函数逻辑：

- 首先判断是否已经创建过，如果创建过直接退出（一个线程一个Reactor）

- 记录线程id——系统id，用于输出日志

- t_reactor_ptr = this;这里的t_reactor_ptr是线程局部变量

- 创建监听树（epoll_create）

- 创建m_wake_fd，并将其添加到监听树上——**addWakeupFd**();

  > **addWakeupFd**()除了把m_wake_fd添加到监听树上，还会把m_wake_fd记录到m_fds中——一个记录已经被注册到树上的fd的vector，属于reactor对象

2）初始化**IOThread**

一些赋值操作：

- 把传入的IOThread指针转化为**IOThread***（因为传入的是void*）赋值给t_cur_io_thread
- 把刚创建的Reactor赋值给m_reactor，并设置其类型为SubReactor——**setReactorType**(SubReactor)
- 给m_tid赋值——**gettid**()

3）创建**主协程**

调用**GetCurrentCoroutine**函数，如果t_cur_coroutine == nullptr，就new一个**Coroutine**对象。最后返回协程指针

协程构造函数逻辑（**Coroutine**构造函数有三种实现，此处调用无参版本）：

- 设置主协程id——m_cor_id = 0

- 协程计数加一——t_coroutine_count**++**

  > t_coroutine用于统计所有线程中协程的总数，是全局原子变量

- 初始化当前协程的上下文，即把将协程上下文结构体的所有字节置为 0——**memset**(&m_coctx, 0, sizeof(m_coctx));

4）通知初始化完成——**sem_post**(&thread->m_init_semaphore);

5）等待启动信号m_start_semaphore（会由**IOThreadPool**::**start**（）函数发送启动信号），接到信号量后就销毁信号量

6）进入主循环loop（**程序运行阶段的核心**）

> [!NOTE]
>
> ###### 一个线程有哪些属性？
>
> - 私有Reactor指针（m_reactor），线程局部变量Reactor指针（t_reactor_ptr）
>
> 为什么要有两个Reactor指针？
>
> **线程局部变量用于快速拿到当前线程的Reactor对象，私有Reactor指针用于生命周期管理**
>
> - 线程ID (m_thread)，类型取决于具体实现，仅在进程内部有效，同一个进程中的不同线程可以通过它相互识别。用于**pthread库的API调用**（如pthread_create(), pthread_join()）
> - 系统线程ID (m_tid)，由操作系统内核分配和管理，是一个整数类型，在整个系统范围内唯一，通过syscall获得。可用于**系统工具追踪**（如top、ps、strace等）
> - 定时器事件 (m_timer_event)
> - 线程索引 (m_index)，标识线程在线程池中的位置
> - 信号量。m_init_semaphore：确保线程初始化完成；m_start_semaphore：控制线程的启动时机

> [!NOTE]
>
> ###### Reactor有哪些属性？
>
> | **文件描述符相关** | **m_epfd，m_wake_fd，m_timer_fd**                            |
> | :----------------- | :----------------------------------------------------------- |
> | 状态标志           | m_stop_flag，m_is_looping，m_is_init_timer（定时器是否初始化），m_tid |
> | *同步相关*         | Mutex m_mutex（只有这一把锁）                                |
> | *文件描述符管理*   | **std**::vector<int> m_fds，**std**::atomic<int> m_fd_size（文件描述符数量） |
> | *待处理事件*       | **std**::map<int, epoll_event> m_pending_add_fds; （待添加的fd和事件）**std**::vector<int> m_pending_del_fds;（待删除的fd），**std**::vector<std::function<void()>> m_pending_tasks; （ 待处理的任务队列） |

##### 2.协议相关组件的初始化

###### 2.1分发器的初始化——m_dispatcher **=** **std**::**make_shared**<**HttpDispacther**>()

TinyPbRpcDispacther是一个RPC请求分发器，用于将接收到的 TinyPb 协议请求分发到对应的服务实现类处理。内部维护了一个保存已注册服务的map，有一个请求分发函数**dispatch**，注册服务函数**registerService**，以及一个解析服务全名的函数**parseServiceFullName**

继承自AbstractDispatcher 抽象类，使用**默认构造**

###### 2.2协议解析器的初始化——m_codec **=** **std**::**make_shared**<**TinyPbCodeC**>()

负责序列化和反序列化，有四个函数，其中**encode**调用**encodePbData**

###### 2.3确定协议类型

m_protocal_type = TinyPb_Protocal;

##### 3.初始化主Reactor

m_main_reactor = **tinyrpc**::**Reactor**::**GetReactor**();

函数逻辑：检查**t_reactor_ptr（线程局部变量）**是否为空，不为空就创建Reactor实例。

创建过程和从Reactor一样。（**主从Reactor都有epoll监听树和m_wake_fd**）

设置反应堆类型——m_main_reactor->**setReactorType**(MainReactor);

##### 4.初始化时间轮

时间轮的构造函数：

```C++
TcpTimeWheel::TcpTimeWheel(Reactor* reactor, int bucket_count, int inteval /*= 10*/) 
  : m_reactor(reactor)
  , m_bucket_count(bucket_count)
  , m_inteval(inteval)
```

要创建bucket_count个桶，每个桶是一个vector，放在queue里（m_wheel）

每个vector里放的是抽象槽，每一个槽有一个连接对象的弱引用

创建定时器事件，每隔interval秒（配置为10秒）执行一次loopFunc，将定时器事件添加到**主reactor**上

**定时器相关之后再看**……

##### 5.创建一个定时事件

每隔10秒执行一次ClearClientTimerFunc，

- 时间轮负责检测连接是否超时,将超时的连接关闭(设置为 Closed 状态)

- 这个定时器负责清理已经关闭的连接对象,释放内存

- 两者配合完成连接的生命周期管理

为什么需要

- 连接可能因为多种原因关闭(超时、对端关闭、错误等)

- 关闭后的连接对象不能立即删除,因为可能还有其他地方在使用

- 通过定时清理,可以确保连接完全不再使用后再释放内存

- 避免内存泄漏

### 线程主循环的逻辑——**Reactor**::**loop**






### dispatcher

#### 作用：

实现了基本的**服务注册和分发**功能。

在一次RPC调用中，当服务端收到客户端请求时,dispatcher负责:

- 解析请求中的服务名和方法名
- 从注册的服务**map**中找到对应服务实例
- 反序列化请求数据
- 调用实际的服务方法
- 序列化响应数据返回给客户端

#### 不足：

- 缺乏服务发现机制
- 错误处理相对简单,没有重试机制
- 没有实现服务端限流和熔断
- 缺少详细的监控指标
- 安全性考虑不足(如认证授权)

#### 工业级RPC框架的分发器应当具备：

不仅要完成基本的服务调用分发，还需要提供完善的服务治理能力，包括：服务注册发现，智能负载均衡，流量控制，服务降级，监控统计，插件扩展，可观测性

#### 当前框架服务注册与发现和工业级RPC的对比：

1.注册时机：

- 当前框架：服务**启动时一次性注册**

- 工业级：支持**运行时动态注册/注销**

2.注册方式：

- 当前框架：**本地注册**到dispatcher

- 工业级：注册到**分布式注册中心**

3.服务发现：

- 当前框架：客户端**直连**服务端

- 工业级：通过**注册中心**实现**服务发现**

4.服务治理：

- 当前框架：基础的服务调用

- 工业级：完整的服务生命周期管理

5.可用性：

- 当前框架：**单点故障风险**

- 工业级：支持高可用和容错

## protobuf相关

1.编写proto文件后，需在文件中设置**option cc_generic_services = true**;以生成服务代码。

2.然后使用**protoc命令**生成对应的.pb.h和.pb.cc文件。

- xxx.pb.h: 包含消息类型定义和服务接口定义

- xxx.pb.cc: 包含消息类型和服务接口的实现

3.服务端与客户端如何使用：

- 服务端需要**继承生成的服务类并实现接口方法**

- 客户端**使用生成的Stub类来调用远程服务**

4.**消息格式**由用户自定义，**服务名**也是自定义的

##### 消息格式（.proto文件）

- 每个字段都有一个类型，可以是**int32，string或枚举**等复合类型
- 一条消息中的每个字段都要指定一个介于`1` 和 `536,870,911` 之间的编号，**必须唯一**，且字段编号 `19,000` 到 `19,999` 保留用于 Protocol Buffers 实现。如果您在消息中使用这些**保留字段**编号之一，协议缓冲区编译器将发出警告
- 字段编号 1-15 比更高的编号编码少一个字节，优先使用
- `optional` 表示字段是**可选的**，即该字段在消息中可以存在，也可以不存在，若未显式赋值，解析时会自动赋予类型默认值
- `repeated` 表示字段是**可重复的**，即该字段可包含多个值（类似数组或列表）

##### protoc生成的文件

每个消息都有自己的类，如果有嵌套关系则还有嵌套类

1.一般API

- 每个字段有一些操作函数，包括getter,setter方法，clear方法等

- 每个消息类还包含许多其他方法，允许检查或操作整个消息，包括：

  - `bool IsInitialized() const;`：检查所有必需字段是否已设置。

  - `string DebugString() const;`：返回消息的人类可读表示形式，特别适用于调试。

    > [!NOTE]
    >
    > ###### 项目中使用ShortDebugString()的原因？
    >
    > 在 Protocol Buffers（Protobuf）的 C++ 实现中，`ShortDebugString()` 是一个用于生成**紧凑、无换行符的调试字符串**的工具函数，与 `DebugString()` 功能类似但格式更简洁。**适合日志记录**。

  - `void CopyFrom(const Person& from);`：使用给定消息的值覆盖当前消息。

  - `void Clear();`：将所有元素清除回空状态。

2.**解析和序列化的API**

每个 protocol buffer 类都有使用 protocol buffer二进制格式读写自己设计的消息的方法。这些方法包括：

- `bool SerializeToString(string* output) const;`：序列化消息并将字节存储在给定的字符串中。请注意，字节是二进制的，而不是文本；我们仅使用 `string` 类作为方便的容器。
- `bool ParseFromString(const string& data);`：从给定的字符串解析消息。
- `bool SerializeToOstream(ostream* output) const;`：将消息写入给定的 C++ `ostream`。
- `bool ParseFromIstream(istream* input);`：从给定的 C++ `istream` 解析消息。

这些只是提供的用于解析和序列化的几个选项。

> [!NOTE]
>
> ###### 为什么序列化后的数据可以是string和ostream？
>
> Protobuf 序列化后的结果本质上是**二进制字节流**，而 `string` 和 `ostream` 只是存储或传输该二进制数据的**容器或通道**。



# 协程

## 协程是如何实现的



## 为什么要使用协程



## 协程切换的逻辑



## hook机制的实现

在服务器**初始化阶段调**用**SetHook**函数，将accept,read,write,connect,sleep进行hook。

##### hook的常见使用场景

- **系统监控与调试**：通过Hook系统API（如文件操作、网络调用），记录调用参数、频率、耗时等数据。
- **性能优化**：Hook资源分配函数（如内存分配、线程调度），优化资源使用策略。
- **安全防护**：Hook敏感函数（如密码校验、加密接口），插入安全检测逻辑。
- **功能扩展与插件化**：通过Hook暴露扩展点，允许第三方代码注入逻辑（如IDE插件、输入法）

> [!NOTE]
>
> ###### 协程为何依赖非阻塞IO？



## 协程调度与管理



# 时间轮实现



# 异步日志

日志宏的执行流程：先判断日志实例是否存在，然后检查日志级别，再创建临时日志对象并获取输出流，最后写入日志内容

# 读取配置模块

配置对象不是单例实现，在程序启动时通过InitConfig创建InitConfig创建，并用智能指针管理，在整个程序运行期间一直存在，被多模块共享使用

**构造函数**：接收**配置文件的路径**，函数内创建一个**TiXmlDocument**对象，并打开文件（xml自带的LoadFile函数）

**析构函数**：delete掉一开始创建的**TiXmlDocument**对象

# 其他问题

## 1.什么时候需要用到线程局部变量？

