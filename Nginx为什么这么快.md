# Nginx为什么这么快

Nginx 是一个免费的，开源的，高性能的 HTTP 服务器和反向代理，以及 IMAP / POP3 代理服务器。Nginx 以其高性能，稳定性，丰富的功能，简单的配置和低资源消耗而闻名。本文从底层原理分析 Nginx 为什么这么快！

## 基本原理

### Nginx的进程模型

![](.\imgs-nginx\nginx-master-workers.png)

Nginx 服务器，正常运行过程中：

* 多进程：一个 Master 进程、多个 Worker 进程
* Master 进程：管理 Worker 进程
  1. 实际处理：网络请求，由 Worker 进程处理
  2. Worker 进程数量：在 nginx.conf 中配置，一般设置为`核心数`，充分利用 CPU 资源，同时，避免进程数量过多，避免进程竞争 CPU 资源，增加上下文切换的损耗。

思考：

1. 请求是连接到 Nginx，Master 进程负责处理和转发？

2. 如何选定哪个 Worker 进程处理请求？请求的处理结果，是否还要经过 Master 进程？

   ![](.\imgs-nginx\master-procedure.png)

### <font color=#3da742>HTTP 连接建立和请求处理过程</font>

* Nginx 启动时，Master 进程，加载配置文件
* Master 进程，初始化监听的 socket
* Master 进程，fork 出多个 Worker 进程
* Worker 进程，竞争新的连接，获胜方通过三次握手，建立 Socket 连接，并处理请求

### <font color=#3da742>Nginx 高性能、高并发</font>

* Nginx 采用：`多进程` + `异步非阻塞`方式（`IO 多路复用` epoll）
* 请求的完整过程：
  1. 建立连接
  2. 读取请求：解析请求
  3. 处理请求
  4. 响应请求
* 请求的完整过程，对应到底层，就是：读写 socket 事件

## Nginx 的事件处理模型

request：Nginx 中 http 请求。

基本的 HTTP Web Server 工作模式：

1. 接收请求：逐行读取`请求行`和`请求头`，判断段有请求体后，读取`请求体`
2. 处理请求
3. 返回响应：根据处理结果，生成相应的 HTTP 请求（`响应行`、`响应头`、`响应体`）

Nginx 也是这个套路，整体流程一致。

![](.\imgs-nginx\nginx-work-mode.png)

### <font color=#3da742>模块化体系结构</font>

![](.\imgs-nginx\architecture.png)

nginx的模块根据其功能基本上可以分为以下几种类型：

* **event module**: 搭建了独立于操作系统的事件处理机制的框架，及提供了各具体事件的处理。包括ngx_events_module， ngx_event_core_module和ngx_epoll_module等。nginx具体使用何种事件处理模块，这依赖于具体的操作系统和编译选项。
* **phase handler**: 此类型的模块也被直接称为handler模块。主要负责处理客户端请求并产生待响应内容，比如ngx_http_static_module模块，负责客户端的静态页面请求处理并将对应的磁盘文件准备为响应内容输出。
* **output filter**: 也称为filter模块，主要是负责对输出的内容进行处理，可以对输出进行修改。例如，可以实现对输出的所有html页面增加预定义的footbar一类的工作，或者对输出的图片的URL进行替换之类的工作。
* **upstream**: upstream模块实现反向代理的功能，将真正的请求转发到后端服务器上，并从后端服务器上读取响应，发回客户端。upstream模块是一种特殊的handler，只不过响应内容不是真正由自己产生的，而是从后端服务器上读取的。
* **load-balancer**: 负载均衡模块，实现特定的算法，在众多的后端服务器中，选择一个服务器出来作为某个请求的转发服务器。

## 常见问题剖析

<font color=3da742 size=5>Nginx vs. Apache</font>

<font color=#3da742 size=4>Nginx:</font>

* IO 多路复用，epoll(freebsd 上是 kqueue )
* 高性能
* 高并发
* 占用系统资源少

<font color=#3da742 size=4>Apache:</font>

* 阻塞 + 多进程/多线程
* 更稳定，bug 少
* 模块更丰富

<font color=3da742 size=5>场景</font>

处理多个请求时，可以采用：`IO 多路复用` 或者 `阻塞 IO` +`多线程`

* **IO 多路服用**：一个线程，跟踪多个 socket 状态，哪个就绪，就读写哪个；
* **阻塞 IO** + **多线程**：每一个请求，新建一个服务线程

<font color=#3da742 size=5>思考</font>

<font color=#3da742>IO多路复用</font>和<font color=#3da742>多线程</font>的适用场景？

<font color=#3da742>IO多路复用</font>：单个连接的请求处理速度没有优势，适合 **IO 密集型** 场景，事件驱动

* **大并发量**：只使用一个线程，处理大量的并发请求，降低**上下文环境**切换损耗，也不需要考虑并发问题，相对可以处理更多的请求；
* 消耗更少的系统资源（不需要`线程调度开销`）
* 适用于`长连接`的情况（多线程模式`长连接`容易造成`线程过多`，造成`频繁调度`）

<font color=#3da742>阻塞IO+多线程</font>: 实现简单，可以不依赖系统调用，适合 **CPU 密集型** 场景

* 每个线程，都需要时间和空间；
* 线程数量增长时，线程调度开销指数增长

### <font color=#3da742>Nginx 最大连接数</font>

<font color=3da742 size=5>基础背景</font>

* Nginx 是多进程模型，Worker 进程用于处理请求；
* 单个进程的连接数（文件描述符 fd），有上限（`nofile`）：`ulimit -n`
* Nginx 上配置单个 worker 进程的最大连接数：`worker_connections` 上限为 `nofile`
* Nginx 上配置 worker 进程的数量：`worker_processes`

<font color=3da742 size=5>因此，Nginx的最大连接数：</font>

* Nginx 的最大连接数：`Worker 进程数量` x `单个 Worker 进程的最大连接数`
* 上面是 Nginx 作为通用服务器时，最大的连接数
* Nginx 作为`反向代理`服务器时，能够服务的最大连接数：（`Worker 进程数量` x `单个 Worker 进程的最大连接数`）/ 2。
* Nginx 反向代理时，会建立 `Client 的连接`和`后端 Web Server 的连接`，占用 2 个连接

<font color=#3da742 size=5>思考</font>

1. 1.每打开一个 socket 占用一个 fd
2. 为什么，`一个进程`能够打开的 fd 数量有限制？

## 附录

### <font color=#3da742>HTTP 请求和响应</font>

#### HTTP请求：

* 请求行：method、uri、http version
* 请求头
* 请求体

#### HTTP 响应：

* 响应行：http version、status code
* 响应头
* 响应体

### <font color=#3da742>IO模型</font>

#### 场景：

处理多个请求时，可以采用<font color=#3da742>IO多路复用</font>和<font color=#3da742>阻塞IO+多线程</font>

<font color=#3da742>IO多路复用</font>：一个线程，跟踪多个socket状态，哪个就绪，就读写哪个

<font color=#3da742>阻塞IO+多线程</font>：每一个请求，新建一个服务线程

#### 思考：

<font color=#3da742>IO多路复用</font>和<font color=#3da742>阻塞IO+多线程</font>的适用场景

<font color=#3da742>IO多路复用</font>：单个连接的请求处理速度没有优势

* 大并发量：只适用一个线程，处理大量并发请求，降低上下文环境切换损耗，也不需要考虑并发问题，相对可以处理更多的请求
* 消耗更少的系统资源，不需要线程调度开销
* 适用于长连接的情况(多线程长连接容易造成线程过多，造成频繁调度)

<font color=#3da742>阻塞IO+多线程</font>：实现简单，可以不依赖系统调用

* 每个线程，都需要时间和空间
* 线程数量增长时，线程调度开销指数增长

### <font color=#3da742>select/poll 和 epoll 比较</font>

```c++
// select 系统调用
intselect(int maxfdp,fd_set *readfds,fd_set *writefds,fd_set *errorfds,struct timeval *timeout);

// poll 系统调用
int poll(struct pollfd fds[], nfds_t nfds, int timeout)；
```

* <font color=#3da742> select</font>
  * 查询 fd_set 中，是否有就绪的 fd，可以设定一个超时时间，当有 fd（File descripter）就绪或超时返回。
  * fd_set 是一个位集合，大小是在编译内核时的常量，默认大小为 1024。
  * 特点：连接数限制，fd_set 可表示的 fd 数量太小了；线性扫描：判断 fd 是否就绪，需要遍历一边 fd_set；数据复制：用户空间和内核空间，复制连接就绪状态信息。
* <font color=#3da742>poll</font>
  * 解决了连接数限制：poll 中将 select 中的 fd_set 替换成了一个 pollfd 数组，解决 fd 数量过小的问题。
  * 数据复制：用户空间和内核空间，复制连接就绪状态信息。
* <font color=#3da742> epoll,event事件驱动</font>
  * :bulb:事件机制：避免线性扫描，为每个 fd，注册一个监听事件，fd 变更为就绪时，将 fd 添加到就绪链表。
  * fd 数量：无限制（OS 级别的限制，单个进程能打开多少个 fd）。
* <font color=#3da742> select、poll、epoll</font>
  * I/O 多路复用的机制。
  * I/O 多路复用就通过一种机制，可以监视多个描述符，一旦某个描述符就绪（一般是读就绪或者写就绪），能够通知程序进行相应的读写操作；监视多个文件描述符。
  * 但 select，poll，epoll 本质上都是同步 I/O：用户进程负责读写（从内核空间拷贝到用户空间），读写过程中，用户进程是阻塞的；异步 IO，无需用户进程负责读写，异步 IO，会负责从内核空间拷贝到用户空间。

## Nginx的并发处理能力

关于 Nginx 的并发处理能力：并发连接数，一般优化后，峰值能保持在 1~3w 左右。（内存和 CPU 核心数不同，会有进一步优化空间）