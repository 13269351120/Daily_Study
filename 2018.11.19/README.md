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






`服务器端逻辑`: 服务器通过`socket()` 创建一个或者多个socket ,并调用`bind()`将其绑定到服务器感兴趣的端口上 , 然后调用`listen()`函数等待客户连接 `connect()` , 服务器需要使用某种I/O模型来监听这一事件 , I/O复用的技术有`select` `poll` `epoll` 系统调用 , 当监听到连接请求后 ,经过三次握手,连接套接字进入 已握完手的队列中 ,  服务器调用`accept()`从握完手的队列中 取出套接字 , 并且分配一个逻辑单元给新的连接, 逻辑单元读取客户请求 处理该请求, 然后将处理结果返回给客户端 , 当服务器接收到 客户端关闭连接请求后,经过四次挥手 , 双方的通信结束 . 在主线程中 , 服务器 只负责监听 其他客户请求 , 不做逻辑处理 
客户端逻辑: 创建一个socket 连接 指定ip地址 和 端口号 , `connect()` 尝试连接 如果连接成功 就往服务器 发送请求 , 然后等待接收服务器的应答 , 最后关闭客户端 .

C/S模型非常适合资源相对集中的场合 , 服务器是通信的中心 , 当访问量过大时 , 客户都将得到很慢的响应 .

#### P2P模型
`P2P`(peer to peer , 点对点)模型 比 C/S模型更符合网络通信的实际情况 , 它使得 网络上所有主机重新回归对等的地位 ,




P2P模型使得每台机器在消耗服务的同时 也给别人提供服务 , 这样资源能够充分自由的共享 ,云计算机群可以看做P2P模型的一个典范，但P2P模型的缺点也很明显 , 当用户之间传输的请求过多时,网络的负载将加重 , 而且 P2P模型存在一个显著的问题 , 即主机之间很难互相发现 , 所以通常会带有一个专门的 发现服务器 ,这个发现服务器通常提供 查找服务 使得客户都能尽快地找到自己需要的资源

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















