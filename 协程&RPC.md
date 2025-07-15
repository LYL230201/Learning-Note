# 一、架构

一个主线程为**主reactor**，只负责接收新连接，并将创建好的连接分发给其他线程去处理

每个IO线程为一个**从reactor**，收到连接后，把连接放在自己的epoll上

一个客户端连接对应一个**协程**，实现yeild和resume

## 1.服务器的初始化与启动

### 1.1启动过程概览

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

### 1.2服务器初始化阶段

**TcpServer**的**构造函数被调用**后：

#### 1.IO线程池的初始化

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

##### 1.1线程

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

##### 1.2线程函数main

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

- 初始化当前协程的上下文，即将协程上下文结构体的所有字节置为 0——**memset**(&m_coctx, 0, sizeof(m_coctx));

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
> | **状态标志**       | m_stop_flag，m_is_looping，m_is_init_timer（定时器是否初始化），m_tid |
> | **同步相关**       | Mutex m_mutex（只有这一把锁）                                |
> | **文件描述符管理** | **std**::vector<int> m_fds，**std**::atomic<int> m_fd_size（文件描述符数量）<br />**作用？**m_fds用于判断一个 fd 是否已经注册过,以决定是 ADD 还是 MOD 操作 |
> | **待处理事件**     | **std**::map<int, epoll_event> m_pending_add_fds; （待添加的fd和事件）**std**::vector<int> m_pending_del_fds;（待删除的fd），**std**::vector<std::function<void()>> m_pending_tasks; （ 待处理的任务队列） |

#### 2.协议相关组件的初始化

##### 2.1分发器的初始化

```C++
m_dispatcher = std::make_shared<HttpDispacther>();
```

TinyPbRpcDispacther是一个RPC请求分发器，用于将接收到的 TinyPb 协议请求分发到对应的服务实现类处理。内部维护了一个保存已注册服务的map，有一个请求分发函数**dispatch**，注册服务函数**registerService**，以及一个解析服务全名的函数**parseServiceFullName**

继承自AbstractDispatcher 抽象类，使用**默认构造**

##### 2.2协议解析器的初始化

```
m_codec = std::make_shared<TinyPbCodeC>();
```

负责序列化和反序列化，有四个函数，其中**encode**调用**encodePbData**

##### 2.3确定协议类型

m_protocal_type = TinyPb_Protocal;

#### 3.初始化主Reactor

m_main_reactor = **tinyrpc**::**Reactor**::**GetReactor**();

函数逻辑：检查**t_reactor_ptr（线程局部变量）**是否为空，不为空就创建Reactor实例。

创建过程和从Reactor一样。（**主从Reactor都有epoll监听树和m_wake_fd**）

设置反应堆类型——m_main_reactor->**setReactorType**(MainReactor);

#### 4.初始化时间轮

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

#### 5.创建一个定时事件

每隔10秒执行一次ClearClientTimerFunc，

- 时间轮负责检测连接是否超时,将超时的连接关闭(设置为 Closed 状态)

- 这个定时器负责清理已经关闭的连接对象,释放内存

- 两者配合完成连接的生命周期管理

为什么需要

- 连接可能因为多种原因关闭(超时、对端关闭、错误等)

- 关闭后的连接对象不能立即删除,因为可能还有其他地方在使用

- 通过定时清理,可以确保连接完全不再使用后再释放内存

- 避免内存泄漏

### 1.3服务器启动阶段

main函数执行**tinyrpc**::**StartRpcServer**();之后

#### 1.启动日志

创建定时器事件，注册到主Reactor。

回调函数为loopFunc，采用双缓冲区记录应用日志和rpc日志，将日志推送到异步日志器。由异步日志器负责刷新日志到磁盘。

#### 2.启动服务器

##### 2.1Accepter的构造与初始化

1）构造函数

传入**NetAddress**::**ptr**类型地址，内部仅执行m_family = m_local_addr**->**getFamily()。

构造函数只负责构造对象，真正的初始化由init函数进行

2）init初始化

创建socket（socket），设置地址复用（setsockopt），绑定地址（bind），开始监听（listen）。

##### 2.2分配Accepter的协程

**主线程的主协程在此构造**

1）获取协程实例

首先调用**GetCoroutinePool**()函数，如果协程池未创建，则调用协程池的构造函数

协程池构造函数逻辑：

- 调用**GetCurrentCoroutine**函数，构造**主线程的主协程**
- 创建m_memory_pool(Memory构造函数被调用)
- 将分配的内存分为一个一个小块，每块有自己的索引和是否被使用标记

从协程池中获得一个协程实例（**getCoroutineInstanse**）。默认从协程池里获取，如果协程池空了就新创建一个协程。

2）设置Accepter的协程回调函数

调用**setCallBack**，传入MainAcceptCorFunc函数

回调函数**MainAcceptCorFunc**逻辑：

- 调用toAccept函数，内部调用accept_hook接收新连接。在accept_hook内部，**如果系统版本的accept失败，才把监听fd注册到监听树上**
- 获取新连接后，在线程池中拿到一个IOThread，**创建一个新的客户端连接**（该连接和一个子线程以及fd绑定。在**TcpConnection**的构造函数中，**该连接被分配了一个协程**，该协程是从协程池中取出的）
- 然后初始化这个连接——conn->**initServer**()。先将其注册到时间轮上，然后设置协程回调函数**MainServerLoopCorFunc**
- 最后将该连接对应的协程添加到IOThread中，等待被执行
- 增加当前活跃连接数——m_tcp_counts++;

普通工作协程回调函数**MainServerLoopCorFunc**逻辑：

- input()
- execute()
- output()

3）将Accepter对应的协程Resume

##### 2.3启动线程池

m_io_pool**->**start();唤醒每一个阻塞等待m_start_semaphore信号量的子线程，子线程进入loop循环

##### 2.4主Reactor循环

m_main_reactor->**loop**();主线程进入loop循环

## 2.服务器运行逻辑

主从Reactor都运行同一个**Reactor**::**loop**，Reactor::loop() 方法的执行，实际上是运行在 t_main_coroutine 的栈空间和上下文中的

**Reactor**::**loop**的逻辑：

1）如果有待处理的第一个协程，优先处理它

2）处理全局协程任务队列（仅子协程），不断地从队列中拿任务来处理（拿任务时加锁）

> [!NOTE]
>
> ###### 全局协程任务队列
>
> 全局协程任务队列对象t_couroutine_task_queue，内部有一个**std**::**queue**<**FdEvent***> m_task;用来保存全局任务
>
> 这个任务队列里放的是**FdEvent***，每一个FdEvent都绑定了一个协程

3）处理待处理任务，执行自己的线程任务队列里的所有任务

> [!NOTE]
>
> ###### 每个线程自己的任务队列
>
> 每一个Reactor都有的**m_pending_tasks**——一个vector，里面存着function对象

4）等待和处理IO事件（epoll_wait及后续循环处理）

epoll_wait对不同事件的处理：

- 如果文件描述符是**m_wake_fd**并且事件为可读事件，就把m_wake_fd中的数据循环读取完，不影响下一次使用
- 如果不满足以上条件则进入这个分支（此分支中的fd的epoll_events中的**data字段**保存的是**ptr指针**，指向用户自定义的**FdEvent**），首先通过ptr获得文件描述符fd
  - 如果事件类型既不是**EPOLLIN**也不是**EPOLLOUT**，就将其从当前线程的事件监听树上删除——**delEventInLoopThread**(fd)
  - 然后对事件的处理分为两个分支
    - 如果当前事件已经绑定了协程，则按照协程的处理逻辑：1）第一个进行特殊处理，不放入全局协程任务队列；2）在子Reactor中，将事件从Reactor中删除，并将任务放入全局协程任务队列；3）在主Reactor中，则直接恢复协程的执行，并清除first_coroutine标记
    - 如果当前事件未绑定协程，则按照回调的方式处理：1）定时器事件直接执行回调；2）普通读写事件放入m_pending_tasks

5）最后更新监听fd：1）把m_pending_add_fds中的fd加到监听树上；2）把m_pending_del_fds中的fd从监听树上删除

> [!NOTE]
>
> ###### m_wake_fd的作用？
>
> eventfd是Linux提供的一种线程间通信机制，只有一种类型，但可以有多种模式：
>
> | **模式**                       | **触发条件**              | **Read 操作行为**                                 | **适用场景**                     |
> | ------------------------------ | ------------------------- | ------------------------------------------------- | -------------------------------- |
> | **标准模式** `flags=0`         | 计数器 > 0 时触发可读事件 | 读取当前计数值，**计数器清零**                    | 通知事件累计次数（如任务完成数） |
> | **信号量模式** `EFD_SEMAPHORE` | 计数器 > 0 时触发可读事件 | 读取值为 `1`，**计数器减 1**（类似信号量 P 操作） | 资源计数（如连接数限制）         |
> | **非阻塞模式** `EFD_NONBLOCK`  | 计数器 = 0 时不阻塞       | 立即返回 `EAGAIN` 错误                            | 避免线程阻塞                     |
> | **执行关闭模式** `EFD_CLOEXEC` | -                         | -                                                 | 防止 `fork` 后子进程继承 fd      |
>
> 项目中创建：m_wake_fd = **eventfd**(0, EFD_NONBLOCK)
>
> 优势：相比管道，节省了一个文件描述符。管道需管理动态缓冲区，`eventfd` 固定 **8 字节计数器**，无内存管理开销。可与多路复用机制无缝集成
>
> 劣势：无法携带复杂数据
>
> **在当前项目中，m_wake_fd主要就是为m_pending_tasks服务的**，因为所有需要跨线程的操作都被封装成了task，通过addTask函数添加到m_pending_tasks。这些操作有：
>
> - 添加/删除fd的操作
>
> - 添加协程的操作
>
> - 定时器相关的操作

> [!NOTE]
>
> ###### FdEvent结构体的设计？
>
> 属性：
>
> | int m_fd {-1};                *// 文件描述符*                |
> | :----------------------------------------------------------- |
> | **std**::function<void()> m_read_callback;    *// 读事件回调* |
> | **std**::function<void()> m_write_callback;   *// 写事件回调* |
> | int m_listen_events {0};           *// 监听的事件类型*       |
> | Reactor* m_reactor {nullptr};        *// 所属的Reactor*      |
> | Coroutine* m_coroutine {nullptr};      *// 关联的协程*       |
> | Mutex m_mutex;                *// 互斥锁*                    |
>
> 设计思考：
>
> 为什么需要Reactor指针：
>
> - 每个FdEvent需要知道它属于哪个事件循环
>
> - 方便事件的注册和注销
>
> 为什么需要协程指针：
>
> - 支持协程的异步IO操作
>
> - 实现IO事件和协程的绑定
>
> 为什么需要回调函数：
>
> - 支持传统的回调方式
>
> - 提供更灵活的事件处理机制
>
> 为什么需要事件掩码：
>
> - 支持多种事件类型的组合
>
> - 方便事件的添加和删除

### 2.1主Reactor

主线程只处理accept和定时器任务

### 2.2从Reactor

线程主循环的逻辑——**Reactor**::**loop**

# 二、RPC

## 1.协议设计

### 存在的问题

- 长度冗余，了解一下TLV(Type-Length-Value)格式优化
- 校验和不如CRC32，且没有加密机制
- 缺乏版本号，不利于协议升级和向后兼容
- 优化字段对齐，减少内存占用

## 2.序列化与反序列化的实现

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



## 3.编码与解码

### 需要编解码的原因

光把参数对象传输过去是不够的，首先是要**保证对端接收后顺利读取数据**并进行处理，另外还要把**方法名**发过去对端才知道怎么处理。

因此，需要**自定义协议**，进行**数据包的制造和拆分**。这个过程称为编码和解码。

### 编解码的具体实现



## 4.分发器

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

## 5.protobuf相关

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

## 6.RPC调用的完整流程

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



# 三、协程

## 1.协程是如何实现的

### 1.1协程内存分配

#### 1.协程栈的分配

当前设计采用的是独立栈。

##### 1.1独立栈

每个协程都有自己独立、预分配的内存区域作为栈。

优点:

- 实现简单直观，上下文切换时只需要切换栈顶指针（RSP）和少数几个寄存器。

- **性能高**，因为栈空间是预分配的，**切换时没有内存拷贝或动态分配的开销**。

- 可以轻松地在协程中调用任何普通的 C/C++ 函数，因为它们就像在正常的线程栈上一样工作。

缺点:

- **内存消耗较大**。即使一个协程只用了几 KB 的栈，也必须为它预留完整的栈空间（比如 128KB）。当协程数量巨大时，总内存占用会很高。

- 可能会有栈溢出的风险，如果预分配的栈空间不够大。

##### 1.2共享栈

所有协程共享一个或少数几个大的栈空间。当一个协程 Yield 时，它已经使用的那部分栈内容会被拷贝到一个独立的堆内存中保存起来。当它 Resume 时，再把保存的内容拷贝回共享栈中。

优点:

- 极大地**节省内存**。每个协程只需要为它 *实际使用* 的栈大小分配备份空间，而不是预留一个固定的最大值。

缺点:

- 实现复杂得多。

- **上下文切换的开销更大**，因为涉及到了内存拷贝（memcpy）。

- 可能存在一些兼容性问题，比如协程栈上的变量地址在每次 Resume 后都可能变化。

### 1.2协程池

### 1.3协程

## 2.协程切换的逻辑

协程切换的过程其实就是主协程和子协程之间执行流切换的过程。主协程的栈就是线程栈，可以说主协程就代表线程，而子协程的栈则是我在堆区分配的。

**执行流如何切换？**首先是栈空间要切换，所以要切换RSP和RBP。执行的代码也需要切换，所以要切换RIP。还有其他的一些必要的寄存器，主要是callee-saved寄存器需要保存和恢复。

### 2.1协程上下文的初始化

**一个协程的生命周期：**

1）**获取实例**：当需要执行一个新任务时，代码会调用**GetCoroutinePool()->getCoroutineInstanse()**，从池里拿出来一个空闲的**协程对象**。这个对象此时仅仅是一个持有独立栈的“容器”。

2）**初始化上下文**：我们需要调用cor->setCallBack(你的业务函数)，这个函数做了两件事：

- 将业务逻辑（**std::function**）与这个协程绑定。

- **对 m_coctx 进行一次性的初始化设置**，伪造好第一次 **Resume** 时所需的 **RSP** 和 **RIP**。

3）**调度**：将这个**准备就绪**的协程交给 Reactor。Reactor 把它包装成一个普通的 **task**，放进任务队列。

4）**执行**：在 Reactor::loop 的某个时刻，这个 task被取出并执行，最终调用到 Coroutine::Resume(cor.get())。

因此有一个协程上下文初始化的过程……

### 2.2寄存器切换

#### 1.要保存和恢复的寄存器

根据ABI原则，我们只需要保存和恢复“被调用者”保存的寄存器即可。但实际实现中不仅保存了这些**Callee-Saved**寄存器，也保存了一部分**Caller-Saved**寄存器（这是一种更保守、更安全但理论上性能稍低的做法，因为它确保了协程恢复时，上下文与暂停时完全一致，**无需依赖编译器的行为**）。

**Caller-Saved**寄存器由**编译器自动管理**，在调用协程切换函数前压入当前栈，函数返回后自动恢复

| \| RAX \| Accumulator Register (通常存放函数返回值) \| |
| ------------------------------------------------------ |
| \| RDI \| Destination Index (存放第1个函数参数) \|     |
| \| RSI \| Source Index (存放第2个函数参数) \|          |
| \| RDX \| Data Register (存放第3个函数参数) \|         |
| \| RCX \| Counter Register (存放第4个函数参数) \|      |
| \| R8 \| General Purpose Register 8 (存放第5个参数) \| |
| \| R9 \| General Purpose Register 9 (存放第6个参数) \| |
| \| R10 \| General Purpose Register 10 \|               |
| \| R11 \| General Purpose Register 11 \|               |

**Callee-Saved**寄存器**必须由协程切换函数手动保存**到协程私有栈，并在切换目标协程时恢复

| \| RBX \| Base Register \|                |
| ----------------------------------------- |
| \| RBP \| Base Pointer / Frame Pointer \| |
| \| RSP \| Stack Pointer \|                |
| \| R12 \| General Purpose Register 12 \|  |
| \| R13 \| General Purpose Register 13 \|  |
| \| R14 \| General Purpose Register 14 \|  |
| \| R15 \| General Purpose Register 15 \|  |

#### 2.RAX的作用

RAX 在 coctx_swap 内部被用作临时存储，它的原始值在切换后会丢失（但这符合 ABI，因为调用者本就不应假设 RAX 的值在函数调用后保持不变）。

**为什么需要一个用作临时存储的寄存器？**

- 根本原因在于 CPU 指令的限制。大部分汇编指令，特别是数据移动指令 mov，**不能同时操作两个内存地址**。

- 也就是说，你无法写出一条像 mov [memory_A], [memory_B] 这样的指令，直接把内存 A 的内容拷贝到内存 B。

- 标准的数据传递范式必须通过寄存器来中转，遵循 **内存 -> 寄存器 -> 内存** 的模式。

让我们看一下 coctx_swap.S 中最需要中转的两处操作：

1.**保存 RIP**

- 目标：把栈顶的值（返回地址）拷贝到m_coctx 结构体中。

- movq 0(%rsp), 72(%rdi) 是非法的，因为 0(%rsp) 和 72(%rdi) 都是内存地址。

- 所以必须：

  1.movq 0(%rsp), %rax (内存 -> 寄存器)

  2.movq %rax, 72(%rdi) (寄存器 -> 内存)

- 这里就需要一个寄存器（RAX）来做这个“二传手”。

2.**保存 RSP**

- 目标：把 RSP 寄存器的值拷贝到m_coctx 结构体中。

- 你同样不能直接把一个专用寄存器 (RSP) 的值直接存到一个复杂的内存地址 (72(%rdi))。

- 所以需要：

  1.leaq (%rsp), %rax (将 RSP 的值加载到通用寄存器 RAX)

  2.movq %rax, 104(%rdi) (通用寄存器 -> 内存)

因此，一个用于临时存储的“中转”寄存器是必不可少的。

**这个寄存器必须是 RAX 吗？**

从技术上讲，不“必须”是 RAX，但 RAX 是最合理、最标准、最高效的选择。

选择 RAX 的原因，又回到了我们之前讨论的 ABI 调用约定。

- RAX 是最“易失”的寄存器：根据 ABI，RAX 是一个“调用者保存”的寄存器，并且主要用于存放函数返回值。任何函数都可以随时改写 RAX 的值而无需向上层负责。它就是 CPU 的“公共草稿纸”。正因为如此，我们在 coctx_swap 中可以肆意使用它，而完全不用担心破坏任何有用的数据，因为调用 coctx_swap 的代码本来就不指望 RAX 的值在调用后会保持不变。

- 使用其他寄存器会更“昂贵”：假设我们不使用 RAX，而是异想天开地选择一个“被调用者保存”的寄存器，比如 RBX，来做临时存储。那么我们的代码会变成这样：

```assembly
    # 错误且低效的示范
    pushq %rbx              # 1. 因为 RBX 是被调用者保存的，所以必须先把它原来的值压栈保护起来！
    leaq (%rsp), %rbx       # 2. 用 rbx 做中转
    movq %rbx, 104(%rdi)    # 3. 存入 m_coctx
    popq %rbx               # 4. 恢复 rbx 原来的值！
```

为了用一下 RBX，我们平白无故地增加了两次内存操作（push 和 pop），这会让这个对性能极其敏感的函数变慢。

#### 3.RSP的切换

当一个协程被创建但还没有运行过时，我们需要给它一个**初始的上下文**，这个工作在**Coroutine::setCallBack** 中完成。

在第一次 Resume 时，coctx_swap 要做的事就是把这个我们手动设置好的 m_coctx.regs[kRSP] 的值加载到 CPU 的 RSP 寄存器里。

**3.1.跳转**

```assembly
movq 104(%rsi), %rsp
```

这句汇编代码就是实现“跳转”的核心！让我们来解读它：

- %rsi：存放着要恢复的那个协程（比如 cor_B）的 m_coctx 的地址。

- 104(%rsi)：在 X86-64 架构中，指针是 8 字节。kRSP 的索引是 13，所以它在 regs 数组中的偏移量是 13 * 8 = 104 字节。这个表达式的意思就是：获取 cor_B->m_coctx.regs[13] 的值。

- movq ..., %rsp：把这个值，移动（加载）到 CPU 的 RSP 寄存器中。

这一行代码执行后，CPU 的栈顶指针 **RSP 就瞬间从线程栈地址，变成了协程独立栈的地址**。

**3.2存档**

```assembly
    leaq (%rsp),%rax
    movq %rax, 104(%rdi)
```

这段代码在换出协程时执行。让我们来解读它：

- %rdi：存放着当前正在运行、即将被暂停的协程（比如 cor_A）的 m_coctx 的地址。

- leaq (%rsp), %rax：leaq 是“加载有效地址”指令。这里的作用是把 CPU 当前 RSP 寄存器的值（也就是 cor_A 在独立栈上运行到的确切位置）加载到 rax 寄存器。

- movq %rax, 104(%rdi)：把 rax 寄存器的值（即当前栈顶位置），保存到 cor_A->m_coctx.regs[13] 中。

这就是“存档”的过程！

#### 4.RBP的切换

- **作用**：它指向当前正在执行的函数栈帧的底部。在一个函数中，RSP（栈顶）会随着局部变量的创建而移动，但 RBP（栈底）通常是固定的。通过 RBP 加上一个固定的偏移量，可以稳定地访问到函数的参数和局部变量，这对于调试和某些栈操作非常重要。

- **切换**：coctx_swap.S 中有 movq %rbp, 48(%rdi) (保存) 和 movq 48(%rsi), %rbp (恢复) 两句，专门负责切换 RBP。

#### 5.RIP的切换

RIP 寄存器有一个特殊的设计：程序员不能用 mov 这样的指令直接修改它。例如，movq some_address, %rip 是非法的。

因此，**coctx_swap 必须用一种间接的方式来切换 RIP**，这个方式就是巧妙地利用函数调用和返回的机制，特别是 **call** 和 **ret** 这两条汇编指令。

**5.1保存RIP**

当 C++ 代码调用 coctx_swap(...) 时，CPU 在底层执行的是 call coctx_swap 指令。call 指令会做两件事：

1. 跳转到 coctx_swap 函数的起始地址（修改 RIP）。

1. 在跳转前，自动地把下一条指令的地址（也就是 call 执行完后的返回地址）压入当前栈的顶部。

coctx_swap 的任务就是把这个刚被压入栈的“返回地址”给取出来，存到 m_coctx 里。

```assembly
    leaq (%rsp),%rax     # rax = 当前栈顶指针 RSP 的值

    # ... 其他寄存器保存 ...

    movq 0(%rax), %rax   # [关键] 读取 rax 指向的地址里的内容 (即刚才 call 指令压入的返回地址)
    movq %rax, 72(%rdi)  # 保存到 from_coctx->regs[kRETAddr] (kRETAddr 索引是 9, 偏移量 72)
```

解读：

- 当一个协程（比如主协程）即将被换出时，它调用了 coctx_swap。

- CPU 自动把 coctx_swap 调用结束后的返回地址压栈。

- coctx_swap 启动后，立刻从栈顶把这个返回地址读出来，并保存到主协程的 m_coctx.regs[kRETAddr] 中。这个地址，就是主协程下一次恢复时应该继续执行的地方。

**5.2恢复RIP**

ret 指令是 call 的逆操作，它只做一件事：从当前栈顶弹出一个地址，然后无条件跳转到这个地址（即把这个地址加载到 RIP）。

coctx_swap 在最后利用 ret 来完成 RIP 的切换。

```assembly
    # ... 其他寄存器恢复 ...
    leaq 8(%rsp), %rsp      # 清理栈，让 RSP 指向我们即将要 push 的位置
    pushq 72(%rsi)         # [关键] 把 to_coctx->regs[kRETAddr] 的值压入当前栈顶
    ret                      # [魔法发生!]
```

解读：

- %rsi 寄存器里是要换入的协程（比如业务协程 cor_B）的 m_coctx 地址。

- pushq 72(%rsi) 这行代码，把 cor_B 的 m_coctx 中保存的返回地址，强行地压入了当前栈的顶部。

- 紧接着 ret 指令执行。它从栈顶弹出我们刚刚放上去的那个地址，然后把它设置给 RIP 寄存器。

- CPU 的执行流瞬间就跳转到了 cor_B 上次被暂停时应该返回的地方。

**5.3返回地址**

在函数调用过程中，CPU会自动**把RIP中的返回地址push到栈顶**，被调函数结束时会**执行ret**，将返回地址写入RIP，程序流恢复。

协程切换时，CPU自动push返回地址到栈上，当coctx_swap是主协程调用的，这个栈就是主协程的栈，当coctx_swap是子协程调用的，这个栈就是子协程的栈。

在coctx_swap的最后我们手动进行了**ret**，一般的函数结束时就会自动执行**ret**，**两者不会重复**。因为高级语言中的“函数结束”，在汇编层面，其最终的实现就是一条 ret 指令。编译器会负责把代码翻译成汇编代码，在函数的末尾，编译器会自动为生成一些“收尾代码”（Epilogue），而这些收尾代码的最后一步，就是插入一条 ret 指令，用来将控制权交还给调用者。

**普通函数**：

- 当 CPU 执行一条 call some_function 指令时，它会在真正跳转之前，自动地、隐式地做一件事：把**返回地址**压入 (push) 到当前栈的顶部。所以，“栈”是返回地址的临时存放地。
- 函数执行到最后，调用 ret 指令。ret 会自动从栈顶找到当初 call 指令存放的那个返回地址，弹出来加载到 RIP，函数就成功返回了。

**协程切换**：

- 代码中调用 call coctx_swap，CPU 依然会自动把**返回地址**压入当前栈的顶部。
  coctx_swap 的汇编代码执行后，会立刻从当前栈顶把这个返回地址读出来。
  然后，它把这个地址值保存到代表当前协程的那个m_coctx 结构体的 **regs[kRETAddr]** 位置。这相当于为这个“书签”做了一个更长期的备份，因为协程可能会被挂起很久。
  所以，在协程切换的场景下，返回地址的存放路径是：RIP -> CPU自动压入栈 -> coctx_swap手动移入m_coctx结构体。
- 在切换到目标协程之前，它会从目标协程的 m_coctx 结构体中，取出之前为它保存好的那个“**返回地址**”。
  然后，它**强行把这个地址压入 (push) 到当前栈的顶部**。
  最后，**执行 ret 指令**。
  ret 指令并不知道这是我们“伪造”的，它只认栈顶。它看到栈顶有我们刚放上去的地址，就把它弹出并加载到 RIP。
  于是，CPU 的执行流就“返回”到了目标协程上次被中断的地方，切换完成。

> [!NOTE]
>
> ###### 线程切换概况
>
> 1. **完全由内核接管**
>
>    - 线程切换（如 Linux 内核线程）必须通过
>
>      系统调用或中断陷入内核态，由内核调度器统一处理上下文保存与恢复
>
>    - 内核会保存所有关键寄存器到**线程控制块**（TCB），包括：
>
>      - 程序计数器（PC）、栈指针（SP）
>      - 通用寄存器（如 x86 的 RAX-R15，ARM 的 R0-R12）
>      - 浮点/向量寄存器（若使用）
>      - 程序状态字（如 x86 的 RFLAGS，ARM 的 CPSR）
>
>    - 恢复时直接从 TCB 加载新线程的寄存器状态，无需应用层干预。
>
> 2. **切换触发与开销**
>
>    - 触发条件：时间片耗尽、I/O 阻塞、主动 yield 等
>    - 开销来源：用户态↔内核态切换（CPU 模式转换）、TLB 刷新、缓存失效等，耗时约 1000ns+

## 3.hook机制的实现

在服务器**初始化阶段调**用**SetHook**函数，将accept,read,write,connect,sleep进行hook。



## 4.协程调度与管理

采用主协程作为调度器，调度器本身也需要一个执行上下文（栈、寄存器等），将它实现为一个**特殊的协程**，可以使上下文切换的逻辑高度统一。

### 4.1“主协程调度器”模式的优劣势

#### 1.优势

1）实现优雅简洁 (Elegance & Simplicity)

这是最大的优势。**上下文切换的逻辑被统一为一种操作**：coctx_swap。无论是“调度器 -> 业务协程”还是“业务协程 -> 调度器”，调用的都是同一个底层函数，无需为调度器编写任何特殊的上下文管理代码。

2）清晰的控制流 (Clear Control Flow)

在单个线程内，控制权始终在主协程和业务协程之间线性转移，绝不会出现业务协程 A 直接切换到业务协程 B 的情况。这使得代码的**执行路径非常清晰，易于理解和调试**。

3）无缝集成事件循环 (Perfect for Event Loops)

这种模式是为 epoll 这类事件驱动模型量身定做的。**主协程的 loop 天然地包裹了 epoll_wait**，形成了一个高效的“等待-唤醒-执行”闭环。

4）资源开销小 (Lightweight)

**不需要为调度器单独创建线程或复杂的管理结构**。它的状态就是主协程栈上的局部变量，非常轻量。

#### 2.劣势

1）1:N 模型的固有局限性 (Limitation of 1:N Model)

这种模式本身只是一个 1:N 模型（1 个 OS 线程调度 N 个协程）。它无法利用多核 CPU 来并行执行协程。如果某个业务协程执行了密集的 CPU 计算（而不是 I/O 等待），它会完全阻塞该线程上的所有其他协程和调度器本身。

2）扩展性依赖于上层框架 (Scalability Depends on Framework)

要克服 1:N 模型的瓶颈，就必须在上层构建一个 M:N 的框架。TinyRPC 正是这么做的：它创建了一个 IOThreadPool，里面有多个 IOThread，每个 IOThread 内部都是一个“主协程调度器”。协程任务通过 CoroutineTaskQueue 在不同线程间分发。

劣势在于：**这个 M:N 的实现相对初级**。例如，它**没有实现工作窃取**。如果任务分配不均，某些线程可能很忙而另一些线程空闲。Go 和 Tokio 等成熟框架的调度器在这方面要智能得多。

3）调度器本身是协作式的 (Cooperative Scheduler)

调度完全依赖于业务协程的主动 Yield。如果一个协程写了一个死循环或者长时间的 CPU 计算而没有主动让出，整个线程都会被“饿死”。更高级的抢占式调度器（如 Go 的调度器）能检测到这种情况并强制切换，但这种简单模型做不到。

## 5.其他关于协程的信息

从函数执行的角度理解：协程机制使得函数变成了可以**多次中断和重入**的结构。

协程本质上**无法利用多核资源**，因为单线程下的多协程本质上还是串行执行的，所以协程需要和多进程或多线程一起使用。

# 四、其他模块

## 1.时间轮



## 2.定时器

## 3.异步日志

日志宏的执行流程：先判断日志实例是否存在，然后检查日志级别，再创建临时日志对象并获取输出流，最后写入日志内容

## 4.配置模块

配置对象不是单例实现，在程序启动时通过InitConfig创建InitConfig创建，并用智能指针管理，在整个程序运行期间一直存在，被多模块共享使用

**构造函数**：接收**配置文件的路径**，函数内创建一个**TiXmlDocument**对象，并打开文件（xml自带的LoadFile函数）

**析构函数**：delete掉一开始创建的**TiXmlDocument**对象

# 五、问题

## 1.什么时候需要用到线程局部变量？

## 2.为什么要使用协程？

## 3.协程为什么依赖非阻塞IO？

## 4.hook的作用是什么，没有hook行不行？

## 5.hook的常见使用场景？

- **系统监控与调试**：通过Hook系统API（如文件操作、网络调用），记录调用参数、频率、耗时等数据。
- **性能优化**：Hook资源分配函数（如内存分配、线程调度），优化资源使用策略。
- **安全防护**：Hook敏感函数（如密码校验、加密接口），插入安全检测逻辑。
- **功能扩展与插件化**：通过Hook暴露扩展点，允许第三方代码注入逻辑（如IDE插件、输入法）

## 6.为什么大量使用智能指针？

## 7.为什么使用protobuf，不用http或别的什么协议？

