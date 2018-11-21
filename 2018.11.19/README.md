#### 学学学!
***
### __Linux高性能服务器开发__
#### 高性能服务器程序框架  
`引言:` 按照服务器程序的一般原理,将服务器解构为 如下三个主要模块:  
 
*  I/O处理单元: 介绍i/o处理单元的四种i/o模型和两种高效事件处理模式  
*  逻辑单元: 可以理解为是子进程 或者 子线程 等处理逻辑任务的  介绍逻辑单元的两种高效并发模式,以及高效的逻辑处理方式 有限状态机  
*  存储单元: 与网络编程本身无关   

  
#### 服务器模型  
##### C/S模型
TCP/IP协议在设计和实现上并没有客户端和服务器的概念,在通信过程中所有的机器都是`对等的`,但是由于 资源被数据提供者所垄断,大多数网络应用程序都很自然的采用了 C/S模型
`C/S模型` : 所有客户端都通过访问服务器来获取所需要的资源


<div align=center><img width="300" height="300" src="https://github.com/13269351120/Daily_Study/raw/master/2018.11.19/CS%E6%A8%A1%E5%BC%8F.png"/></div>


`服务器端逻辑`: 服务器通过`socket()` 创建一个或者多个socket ,并调用`bind()`将其绑定到服务器感兴趣的端口上 , 然后调用`listen()`函数等待客户连接 `connect()` , 服务器需要使用某种I/O模型来监听这一事件 , I/O复用的技术有`select` `poll` `epoll` 系统调用 , 当监听到连接请求后 ,经过三次握手,连接套接字进入 已握完手的队列中 ,  服务器调用`accept()`从握完手的队列中 取出套接字 , 并且分配一个逻辑单元给新的连接, 逻辑单元读取客户请求 处理该请求, 然后将处理结果返回给客户端 , 当服务器接收到 客户端关闭连接请求后,经过四次挥手 , 双方的通信结束 . 在主线程中 , 服务器 只负责监听 其他客户请求 , 不做逻辑处理 
客户端逻辑: 创建一个socket 连接 指定ip地址 和 端口号 , `connect()` 尝试连接 如果连接成功 就往服务器 发送请求 , 然后等待接收服务器的应答 , 最后关闭客户端 .


<div align=center><img width="300" height="300" src="https://github.com/13269351120/Daily_Study/raw/master/2018.11.19/liuchengtu.png"/></div>


C/S模型非常适合资源相对集中的场合 , 服务器是通信的中心 , 当访问量过大时 , 客户都将得到很慢的响应 .

#### P2P模型
`P2P`(peer to peer , 点对点)模型 比 C/S模型更符合网络通信的实际情况 , 它使得 网络上所有主机重新回归对等的地位 ,

<div align=center><img width="300" height="300" src="https://github.com/13269351120/Daily_Study/raw/master/2018.11.19/p2p.png"/></div>


P2P模型使得每台机器在消耗服务的同时 也给别人提供服务 , 这样资源能够充分自由的共享 ,云计算机群可以看做P2P模型的一个典范，但P2P模型的缺点也很明显 , 当用户之间传输的请求过多时,网络的负载将加重 , 而且 P2P模型存在一个显著的问题 , 即主机之间很难互相发现 , 所以通常会带有一个专门的 发现服务器 ,这个发现服务器通常提供 查找服务 使得客户都能尽快地找到自己需要的资源


<div align=center><img width="300" height="300" src="https://github.com/13269351120/Daily_Study/raw/master/2018.11.19/faxianfuwuqi.png"/></div>


---    
 
#### I/O模型
主要讲 `阻塞` 和 `非阻塞`   
socket在创建的时候默认是阻塞的 , 我们可以给socket系统调用的第2个参数传递 `SOCK_NONBLOCK`标志 或者通过`fcntl`系统调用的`F_SETFL`命令 将其设置为非阻塞的 , 阻塞和非阻塞的概念能应用于所有的文件描述符 .

`概念`:  
`阻塞I/O`: 系统调用因为无法立即完成而被操作系统挂起 ,不再拥有CPU的权利,直到等待的时间发生为止 ,可能被阻塞的系统调用包括 `accept` `send` `recv` `connect` 


`非阻塞I/O`: 系统调用总是立即返回 , 而不管时间是否已经发生 如果时间没有立即 发生 这些系统调用就返回 -1 , 和出错的情况一样 这个时候我们必须根据 errno来区分这两种情况 . 
事件未发生 `errno` : `EAGAIN`  `EWOULDBLOCK` `EINPROGRESS`

阻塞I/O模式 很安全 , 一步一步来 , 在非阻塞I/O模式中 , 只有在事件 已经发生的情况下再去操作比较合适 , 这个时候就需要用到 **通知机制** , `I/O复用` 和 `SIGIO信号` 

  
* I/O复用:  
应用程序通过I/O复用函数向内核注册一组事件 , 内核通过I/O复用函数把其中就绪的时间通知给应用程序 , linux下 用select poll 和 epoll_wait 要注意的是 , I/O复用函数本身是阻塞的 , 它们能提高程序效率地原因在于它们具有同时监听多个I/O事件的能力 

* SIGNO信号:  
为一个目标文件描述符指定宿主进程 , 那么背指定的宿主进程将捕获到 SIGIO信号,这样当目标文件描述符上有事件发生时,SIGIO信号的信号处理函数将被触发 , 我们可以在信号处理函数中对目标文件描述符执行非阻塞I/O的操作了 

`同步I/O和异步I/O的比较`:  
`同步I/O` :从理论上说 阻塞I/O和I/O复用 信号驱动I/O都是同步I/O模型 , 因为在这三种I/O模型中 I/O的读写操作 都是在**I/O事件发生之后** , 同步I/O要求用户**自行执行**I/O操作 , 将数据从内核缓冲区读入用户缓冲区 或将数据从用户缓冲区 写入内核缓冲区   
`异步I/O` : 用户可以直接对I/O执行读写操作 , 这些操作告诉内核用户读写缓冲区的位置,以及I/O操作完成之后内核通知应用程序的方式 , 异步I/O的读写操作总是立即返回 , 异步I/O由内核来执行, 数据从内核缓冲区 和用户缓冲区之间的移动是由 内核在后台完成的 

__总之__   
* 同步I/O向应用层序通知的是I/O就绪事件 , 就绪了 , 用户就可以对I/O进行主动的操作了 .  
* 异步I/O向应用程序通知的I/O完成事件.  

### 两种高效的事件处理模式
服务器程序通常需要处理三类事件 , `I/O事件` `信号` 以及 `定时事件`   
两种高效的事件处理模式 : `Reactor` 和 `Proactor` 

#### Reactor模式  
同步I/O模型通常用于实现`Reactor模式` . Reactor模式要求 `主线程` 只 负责监听文件描述符上 是否有事件发生 , 有的话就立即通知`工作线程` , 除此之外 主线程**不做任何其他实质性**的 工作 .  

<div align=center><img width="600" height="300" src="https://github.com/13269351120/Daily_Study/raw/master/2018.11.19/reactor.png"/></div>   

`实现的Reactor模式的工作流程是`:
> * 主线程往epoll内核事件表中注册socket上的 读就绪 事件 
> * 主线程调用epoll_wait 等待socket 上有数据可读 
> * 当socket上有数据可读时 , epoll_wait 通知主线程 , 主线程则将 socket 可读事件放入请求队列
> * 睡眠在请求队列上的某个工作线程被唤醒 , 它从socket读取数据 , 并处理客户请求 ,然后往epoll内核事件表中注册socket 上的 写就绪时间 
> * 主线程调用 epoll_wait 等待socket 可写 
> * 当socket可写时 , epoll_wait 通知主线程 , 主线程将socket可写时间放入请求队列 
> * 睡眠在请求队列上的某个工作线程被唤醒 , 它往socket上写入服务器处理客户请求结果


#### Proactor(不太理解)  
与`Reactor模式`不同 `Proactor模式` 将所有的I/O操作都交给`主线程` 和 `内核`来处理 , 工作线程仅仅负责业务逻辑 使用`异步I/O模型`   
<div align=center><img width="600" height="300" src="https://github.com/13269351120/Daily_Study/raw/master/2018.11.19/proactor.png"/></div>    

`实现的Proactor的工作流程`:      

> * 主线程调用aio_read函数向内核注册socket上的 读完成事件 , 并告诉内核用户 读缓冲区的位置 ? 很抽象 没有例子 , 以及读操作完成时 如何通知应用程序 这里以 信号为例 
> * 主线程继续处理其他逻辑
> * 当socket上的数据被读入用户缓冲区后 , 内核将向应用程序发送一个信号 , 以通知应用程序数据已经可用 
> * 应用程序预先定义好的信号处理函数选择一个工作线程来处理客户请求 . 工作线程处理完 客户请求之后 ,调用 aio_write函数向内核注册socket上的写完成时间 , 并告诉内核用户写缓冲区的位置 , 以及写操作完成时 如何通知应用程序

> * 主线程继续处理其他逻辑 
> * 当用户缓冲区的数据被写入socket之后 , 内核将向应用程序发送一个信号 , 以通知应用程序数据已经发送完毕 
> * 应用程序预先定义好的信号处理函数选择一个工作线程来做 善后处理 , 比如决定是否关闭 socket  

连接socket上的读写事件是通过 `aio_read` 和 `aio_write` 向 内核 注册的 , 因此内核将通过信号来向应用程序报告连接socket上的读写事件 , 所以 主线程中的 `epoll_wait`调用仅能用来检测监听`socket`上的连接请求事件 , 而不能用来检测连接`socket`上的读写事件   

#### 两种高效的并发模式 :  
并发编程的目的是让 程序 "同时"执行多个任务,从而提高效率 , 但是我理解这个并发编程 是这样子的:
如果说 所有的任务都不需要 通过I/O 只是计算的 , 那么一个接着一个任务的计算有什么不好呢 , 为什么还要 搞出 并发 这种假装 同时的效果呢?毕竟 切换线程 也是有时间代价的 , 更不用说是进程之间的切换了 , 那么为什么并发编程 这么流行呢 ? 是因为 有任务 是 I/O 密集的 , I/O的速度 很慢 , 这个时候 程序占着CPU也没有任何的作用 , 这也被称为 阻塞 , 如果程序由多个执行线程 , 那么就可以在 某个线程 阻塞的时候 , 切换到下一个线程 , 这个 和 我们 在烧水时 不会傻傻等在火炉前是一个道理的 . 
从实现上来说 并发编程主要有 多进程 和 多线程 两种方式 , 这里要讨论服务器并发模式 , 服务器 经常出现 I/O操作 当然也有 逻辑运算 , 并发模式 就是指 I/O处理单元 和 多个逻辑单元之间协调完成任务的方法 .
服务器主要有两种并发编程模式 : `半同步/半异步` 模式 和 `领导者 / 追随者` 模式 


#### 半同步/半异步模式 :   
这里再重新理解一下 同步和异步的概念 顺便回忆一下 阻塞 和 非阻塞的概念 : 这是一个太重要 且 不好理解 理解了 还容易忘的 概念..................
以下的文字 大部分参考于一篇CSDN上的博客, 然后使用自己的语言重新组织 和 理解了一下 .  
https://blog.csdn.net/historyasamirror/article/details/5778378    
感谢博主分享 ~ 谢谢~     

I/O发生时涉及到的对象和步骤 
> * 1.对于内核而言 : 等待数据准备
> * 2.对于用户而言: 将数据从内核拷贝到进程(线程)中 

`阻塞I/O`    
在linux,默认情况下所有的socket都是阻塞的 , 一个典型的 读操作的流程  
<div align=center><img width="500" height="300" src="https://github.com/13269351120/Daily_Study/raw/master/2018.11.19/block.gif"/></div>  

可以看到 用户调用 recvfrom这个系统调用, 内核开始了I/O的第一个阶段 : 准备数据   
对于网络IO来说 , 很多数据一开始 还未到达,这个时候 内核 需要等待数据到来 , 而在用户进程这边 , 整个进程会被阻塞 , 当 内核等到 数据全部准备好了 , 再将数据从 内核 拷贝到 用户内存中 ,然后 内核返回结果 , 用户进程 才解除 阻塞的状态 .重新运行起来 .  
`总结` : 阻塞I/O在 I/O的两个阶段都发生了阻塞   
`感性的去理解`: 阻塞I/O好比是一个 很懒的人 , 他(用户)一旦有什么事情 拜托给了别人(内核) , 在别人也没做完之前(内核返回结果) , 自己不做任何事 呼呼大睡 天天划水 (阻塞).  

`非阻塞I/O`    
linux下可以通过socket设置将其变成非阻塞的,下图    
<div align=center><img width="500" height="300" src="https://github.com/13269351120/Daily_Study/raw/master/2018.11.19/nonblock.gif"/></div>  

图中可以看出 用户进程发出 recvfrom系统调用后 内核的数据还没有准备好 , 它并不会阻塞用户的进程 , 而是立即返回一个 errono 比如EWOULDBLOCK , 根据这立即返回的 错误信息 它就知道 数据还没有准备好 , 然后隔一会又去问 内核 , 如果准备好了 ,内核将数据拷贝到用户内存中 , 并返回一个 ok .   
`总结` : 非阻塞I/O在内核里 准备数据的阶段是阻塞的 , 但是 用户会一直去主动询问内核 数据准备好了没有  
`感性的去理解` : 非阻塞I/O是一个急性子 , 他把他处理不了的事情 拜托给了别人 , 就会不断去问那个人 做好了没有 , 自己其实很累啊  ......  这个比喻是不是已经黑了 非阻塞I/O一把 感觉没什么用啊 , 该返回的时候还是会返回的 , 那 要 非阻塞I/O干啥呢 ?   

`I/O复用`    
I/O复用是通过 select poll 或者 epoll_wait 系统调用实现的 , 好处在于 单个进程能够处理多个 网络连接的I/O,它的基本原理是select epoll会不断地轮询 所负责的所有socket , 当某个socket有数据到达了,就会通知用户进程 进行处理 .
<div align=center><img width="500" height="300" src="https://github.com/13269351120/Daily_Study/raw/master/2018.11.19/IOmulti.gif"/></div>     
当用户进程调用select 那么整个进程会被阻塞 , 而同时, 内核会 监视所有select负责的socket, 当负责的任何一个socket中的数据准备好了 , select 就会返回, 这个时候用户取消阻塞 , 进行 读处理 , 需要注意的是 : 在I/O复用的时候 , 实际上 对于每一个 socket我们都设置成 non-blocking的 , 但是 如上图所示 和 我们的分析 , 用户进程是一直 阻塞的 , 这里要注意 , 用户进程 是被 select 阻塞的 , 而不是被socket阻塞的 , 为了彻底理解这个 我又查了 知乎 https://www.zhihu.com/question/33072351 
在这里 提出了 使用 I/O复用的时候 大多数情况是要把 socket设置成 non-blocking的 , 理由是出自 最高赞者: `晨随` , 用户通过select 让内核帮忙 监听了 若干个 `socket` , 当内核检测到 某个`socket`可读的时候 就会返回 通知 用户进程进行 对该socket的一系列操作 , 但是这里要注意 (需要注意的东西也太多了吧 >.<), 返回的`socket` 的`读操作` 不一定是 顺畅的 , 比如 协议栈检查到这个`分节检验和错误` , 然后`丢弃`了这个分节 , 这个时候直接调用 `read` 这个socket 则无 数据可读 , 如果`socket` 设置成 阻塞形式 , 则会一直阻塞在这里 , 所以 为了 安全起见 , 应该把socket 设置成 **non-blocking** 的 , 那么又在想 , **为什么还会有少数情况 可以不设置 成 non-blocking的呢 ? 挖个坑 未来填吧!!!**      
`总结` : I/O复用和 阻塞I/O是差不多的 , 在 用户进程 和 内核中都会阻塞 , 只不过 I/O复用在内核中 会 轮询多个 socket , 阻塞I/O只有 等一个 , 那这个的效率地提升 可想而知 , 所以 select 或者 epoll 能够 使服务器的连接数上升 也 很容易理解了 ,     
`感性的去理解 `: I/O复用 是一个 比 阻塞I/O更加懒的一个人 , 他把一大坨事情 都交给了别人去干 , 只要那个人做完了其中一件事 , 就通知他 , 啧啧啧  , 压榨啊 .  

`异步I/O`   
linux下异步I/O用的比较少 看图   
<div align=center><img width="500" height="300" src="https://github.com/13269351120/Daily_Study/raw/master/2018.11.19/asyn.gif"/></div>  

`感性的理解` :用户进程发起`read` 操作之后 , 立刻就可以去干其他的事情了 , 它不像是 `阻塞I/O` 很懒 等着 内核 通知 , 也不像 `非阻塞I/O` 那么 急躁 , 一遍又一遍的去催内核 , 所以 异步IO可以看成是一个 正常人...  拜托内核的事 就等着内核处理呗 , 自己干接下来的事 ... 

#### 整理和总结  
`blocking 和 non-blocking的区别`:   
> * 阻塞的程度不同 : 通过把IO的步骤分成两个后 , 我们可以清楚的看到 , blocking IO 在 内核 和 用户线程都阻塞了 , 而 non-blocking IO 在用户线程中 非阻塞  
> * 调用函数返回的时间不同:  blocking IO 在调用函数后 , 不是立即返回的 , 等到内核收到数据,并把数据拷贝到用户线程中后返回 , 而non-blocking IO调用函数后是立即返回的 通过其错误值的类型去配短状态 .

`synchronous 和 asynchronous的区别` :   
引出两者的定义:   
> * synchronous IO : A synchronous IO operation causes the requesting process to be blocked until that IO operation completes   
> * asynchronous IO : An asynchronous IO operation does not cause the requesting process to be blocked  
一言以蔽之 : 同步IO 就会在 `IO请求`的时候 导致**发出请求的进程(线程)进行阻塞** , 而异步则不会 , 同步和异步 表示的是 发出IO请求的进程的情况.    

`哪些属于同步IO呢` :  
那么其实 阻塞IO IO复用 非阻塞IO 都属于同步IO , 阻塞IO 和 IO复用在发出请求后 , 都阻塞了用户线程  这个没有问题 , `特别注意` 非阻塞IO 刚刚说了 , 非阻塞IO 是在发出请求后 , 自己还可以不停去询问 去检查立即返回的errono来判断状态 , 那怎么是同步IO呢? 根据博客的分析 , 定义中的IO操作 , 是指真实的IO操作 , 就是 recvfrom 这个系统调用 , 在内核准备好数据之前 , 非阻塞IO确实没有阻塞用户进程 , 但是当内核准备好数据了 , 将数据从内核拷贝到用户内存的时候 , 这个时候 用户进程 是被阻塞的 , 可以去看 非阻塞IO的那个图 .   

`哪些属于异步IO呢` :    
异步IO操作之后  就直接返回再也不理睬了 , 直到内核发送一个信号 , 告诉进程说 IO 完成了 , 在整个过程中 , 进程是不会因为IO而阻塞的 .   
  
`综上所述`:     
我们对 各种IO形式有了新的理解, 在这里做一个形象的总结 以便记忆  :      
##### 同步IO系列 :    
> * 阻塞IO : `苦苦傻等僧` 他是一个懒人, 把事情交给别人(内核)后 , 自己就不再干别的什么事情了 , 也从侧面体现了 交给别人的这件事 他仿佛很希望得到结果 就一直苦等  
> * 非阻塞IO : `急躁傻催僧` 他是一个急性子 , 把事情交给别人(内核)后 , 自己就经常去催 , 也从侧面体现了 交给别人的事 他也很上心 , 等到别人说 事情快做完了(内核中数据接收完整后),他就迫不及待去拿(从内核中copy,这个时候用户进程是阻塞的)  
> * IO复用 : `更懒傻等僧` 在懒人的基础上更进一步 , 将很多件事 都交给别人做 , 当然自己对这些事 也还算是上心     

##### 异步IO系列 : 
> * 异步IO :  `无心正常人`  他是一个正常人的思维 ,交给别人的事情 就抱着 `信人不疑` 的态度 , 所以对交给别人的事 可以说是 不上心了 放下了包袱 , 继续自己的任务和旅程   

##### 至于什么时候使用具体哪个IO 以及为什么 在这里先留一个坑 , 以后经验丰富了再来记录感想 ....

--- 

### 元编程
#### 类型元函数
```cpp
template <typename T>
struct Fun_ {using type = T ;};

template <>
struct Fun_<int> {using type = unsigned int ;};

template <>
struct Fun_<long> {using type = unsigned long ;};

Fun_<int> :: type h = 3 ;
```  
`解释`:
后两个`template<>` 是 第一个 `template<typename T>`的**特化版本** ,  看到 空的<>很不熟悉 但是下面的代码应该是看到过的~ 是一种`偏特化`  
```cpp
template<class T1, class T2> struct foo
{
  void doStuff() { std::cout << "generic foo "; }
};
template<class T1>
struct foo<T1, int>
{
 void doStuff() { std::cout << "specific foo with T2=int"; }
};
```  

Fun_ 与C++一般意义上的函数看起来完全不同, 但是Fun_具备了一个元函数所需要的全部性质 

> * 输入为某个类型信息T , 以模板参数的形式传递到Fun_模板中
> * 输出为Fun_模板的内部类型 type, 即 Fun_<T>::type 
> * 映射体现为模板通过特化实现的转换逻辑 

#### 各式各样的元函数
一个模板就是一个元函数:
```cpp
template <typename T>
struct Fun {} ;
```  
#### 无参元函数
```cpp
struct Fun {using type = int ;} ;
constexpr int fun() {return 10 ;}
```  
前者返回一个 类型 int 后者返回数值 10 这样看元函数还是蛮简单的 

#### 基于C++14中对constexpr 的扩展 
我们可以简化
```cpp
constexpr int fun(int a) { return a+1 ;}
```
`调用方式` : **fun(3)**

```cpp
template <int a>
constexpr int fun = a + 1 ;
```
`调用方式` : **fun<3>**

#### 多返回值的元函数:
```cpp
template<>
struct Fun_<int>
{
	using reference_type = int& ;
	using const_reference_type = const int & ;
	using value_type = int ;
};
```  
#### type_traits
这是一个元函数库 , 由`boost`引入的 `C++11`将其纳入其中,这个库实现了 `类型变换`  `类型比较` 与 `判断` 等功能  

#### 元函数的命名方式(C++模板元编程实战中的命名方式 可以借鉴)  
根据原函数的`返回值形式`的不同 元函数的命名方式可以区分开   
如果函数的返回值 要用某种依赖型的名称表示 那么函数将被命名为` xxx_ ` 形式   
否则就可以直接用 `xxx` 表示 

`典型例子` :
* 依赖型的:
```cpp
template <int a , int b>
struct Add_
{
	constexpr static int value = a + b ; 
};
```
`调用方式` : **constexpr int x1 = Add_<2,3>::value**;

* 非依赖型的:
```cpp
template <int a , int b >
constexpr int Add = a + b ;
```   
`调用方式` : **constexpr int x2 = Add<2,3>** ;

#### 模板作为元函数的输入  
C++元函数可以操作的数据包含3类: `数值` `类型` 和 `模板` , 前面讨论了 `数值` 和 `类型` 作为输入 元数据 ,现在讨论 `模板` 作为 `元数据`    
```cpp
template <temolate <typename> class T1 , typename T2>
struct Fun_{
	using type = typename T1<T2>::type;
}; 

template <template<typename>class T1 , typename T2>
using Fun = typename Fun_<T1,T2>::type ;

Fun<std::remove_reference , int& > h = 3 ;
```  
这个元函数 接收两个输入参数 , 第一个参数是 模板 第二个参数是 类型 , 将类型应用于模板之上 , 获取到的 结果类型作为返回值 , 传入 `std::remove_reference` , 这是一个模板 , 和 `int &` 这是一个类型 , 根据调用规则 , 这个函数将返回 int 

从函数式程序设计的角度来说 , 上述代码所定义的Fun 是一个典型的高阶函数 , 即 以另一个函数为输入参数的函数 , 可以总结为 如下的数学表达式 
                                                   `Fun(T,t) = T(t)` 

#### 模板作为元函数的输出   
比较复杂 需要时间去沉淀理解...  
```cpp
template <bool AddOrRemoveRef> struct Fun_ ;

template <>
struct Fun_<true>
{
	template <typename T>
	using type = std::add_lvalue_reference<T> ;
};

template <>
struct Fun_<false>
{
	template <typename T>
	using type = std::remove_reference<T> ;
} ;

template <typename T>
template <bool AddorRemove>
using Fun = typename Fun_<AddOrRemove> ::template type<T>;

template<typename T>
using Res_ = Fun<false> ;

Res_<int&>::type h = 3 ; 
```  
这个比较复杂 现在暂时 先记着 未来填坑   
这个整个的处理过程表示为数学过程:
                                              `Fun(addOrRemove) = T`  
其中,T是一个元函数















