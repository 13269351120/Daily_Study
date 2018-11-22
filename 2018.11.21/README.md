##### 昨天有点事,耽误了一个白天,今天刚刚还在填前天留下的坑 -.-


#### 句柄
`what` :

`why` :

`how` :


#### 模板元编程   
##### 容器模板
`what` :   

元数据的"数组" 数组中的元素可以是 数值 类型 模板   

`how`:  
> * 可以用C++`模板元编程库MPL`来实现 编译期表示数组 集合 映射等复杂的数据结构   
> * 使用C++11新特性 `变长参数模板`(variadic template)     

```cpp
//int容器  
template <int... Vals> struct IntContainer;
//类型容器  
template <typename... Vals> struct TypeContainer ;  
//模板容器  且模板里有多种类型  
template<template <typename...>class... T> struct TemplateCont2 ;  
```     
   
**元编程的一个特点** : 上述的语句都是`声明`而非定义,事实上 ,我们不需要定义,声明中已经包含了编译器需要使用的全部信息  , 仅仅在有必要时才引入定义 , 其他的时候直接使用声明即可 .      
 
##### 顺序分支循环代码的编写 
`顺序` :   
```cpp
template <typename T>
struct RemoveReferenceConst_ 
{
private:
	using inter_type = typename std::remove_reference<T>::type ;
public:
	using type = typename std::remove_const<inter_type>::type ;
}

template <typename T>
using RemoveReferenceConst = typename RemoveReferenceConst_<T>::type ;

RemoveReferenceConst<const int&> h = 3 ;  
```  

**需要注意的点** :    
顺序代码的顺序使不能随意调换的 , 但也`并非一定不能调换`的 , 这个其实很容易理解 , 就比如 
```cpp
int a = 0 ; int b = 0 ; 
```
就是可以交换的 , 但是如果
```cpp
int a = 0 ; int b = a ; 
```
这样的 就不能交换了 , 在元编程中 , 也有类似的情况 , 但是会比 运行时的顺序代码更加复杂一些     
**考虑** :   
```cpp
struct RunTimeExample
{
	static void fun1() {fun2();} 
	static void fun2() {cerr << "hello" << endl ; } 
}
```  
可能一开始会觉得上述代码是错误的  , fun1()里面调用到了 fun2() , 而fun2() 还没有出现过 , 但是要注意的是 : 在编译期 , 编译器会扫描两遍结构体的代码 , 第一遍处理声明 , 第二遍才会深入到函数的定义之中 . (这是所有编译的情况 ? )  但是 开始的那个 顺序代码不能调换顺序 , 所以说 顺序代码的 顺序性还是值得注意的   

##### 分支执行的代码
编译期的分支逻辑 既可以表现为纯粹的 元函数 , 也可以与运行期的执行逻辑相结合  此前有一个代码比较复杂 其实就是使用了典型的分支结构 
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
上例中使用了 模板的特化 或者 部分特化(是不是可以称为 偏特化)  实现分支 , 这是一种非常常见的分支实现方式 ,未来会介绍讨论更多的分支实现方式以及其优缺点 . (今天就看到这儿 ...)   
       
### Linux高性能服务器编程
#### 高性能IO框架库Libevent
`前言` : Linux服务器程序必须处理三类事件 : I/O事件 信号 和 定时事件 在处理这三类事件时 我们通常需要考虑如下三个问题 :   
> * 统一事件源 : 统一处理这三类事件既能使代码简单易懂,又能避免一些潜在的逻辑错误 . 实现统一事件源的一般方法是 利用IO复用系统调用来管理所有事件 

> * 可移植性 : 不同的操作系统具有不同的IO复用方式  

> * 对并发编程的支持 : 在多进程 和 多线程环境下 , 需要考虑各执行实体如何协同处理客户连接 信号 和 定时器 , 以避免竞态条件  

有一些优秀的IO框架库帮助我们解决了上述问题 , 而且稳定性 性能等各方面都相当出色
这里讨论相对轻量级的`Libevent`框架库 . 

##### IO框架库概述  
`what` : 
IO框架库 以库函数的形式 ,`封装`了较为底层的系统调用 , 给应用程序提供了一组更便于使用的接口 .  
IO框架库的实现原理基本类似 , 以 `Reactor模式` 或者 `Proactor模式`实现 , 要么`同时使用`两种模式.

##### Reactor模式实现IO框架库 : 
包含如下几个组件 : `句柄(Handle)` `事件多路分发器(EventDemultiplexer)` `事件处理器(EventHandler)`  和 `具体的事件处理器(ConcreteEventHandler)`  和 `Reactor`  


<div align=center><img width="600" height="300" src="https://github.com/13269351120/Daily_Study/raw/master/2018.11.21/IOkuangjia.png"/></div>

上图是IO框架库组件图片
> * 1.句柄(Handler)
IO框架库要处理的对象 即IO事件 信号 和 定时事件 统一称为 事件源 , **一个事件源通常和一个句柄绑定在一起** , `句柄的作用` : 是 当内核检测到就绪事件时 , 它将通过句柄来`通知(notify)`应用程序这一事件 , 在Linux环境下 , IO事件对应的句柄是`文件描述符` , 信号事件对应的句柄就是`信号值`  

> * 2.事件多路分发器(EventDemultiplexer)
事件的到来是**随机的 异步的** , 所以程序需要循环地等待并处理事件 , 这就是事件循环 , 在事件循环中 等待事件一般使用`IO复用技术`来实现 , IO框架库将系统支持的各种IO复用系统调用封装成统一的接口 称为事件多路分发器 , 事件多路分发器的`demultiplex`方法是等待事件的核心函数 , 其内部调用的是 `select` `poll` `epoll_wait`等函数 , 除了等待事件到来 , 事件多路分发器还需要实现 `register_event` 和 `remove_event`  

> * 3.事件处理器 和 具体事件处理器  
事件处理器执行事件对应的业务逻辑 . 通常包含一个或多个`handle_event`回调函数 , 这些回调函数 在事件循环中 被执行 . IO框架库提供的事件处理器通常是一个接口  , 用户需要继承它来实现自己的事件处理器  (具体事件处理器) (是不是有点像 工厂方法 使用了多态机制 )  以支持用户的扩展     
除了 回调函数 , 事件处理器 还提供一个 `get_handle`方法 它返回与该事件处理器关联的句柄 , 当事件多路分发器检测到 有事件发生时 , 它是**通过句柄来通知应用程序**的 , 因此 必须将事件处理器将句柄绑定 (这也很容易理解 , 处理 什么东西(句柄) 以及怎么处理 (回调函数))

> * 4.Reactor  
Reactor是IO框架库的核心 , 它提供的几个主要方法是:  
handle_events . 该方法执行事件循环 , 它重复如下过程 : 等待事件 -> 依次处理所有就绪事件对应的事件处理器  
register_handler : 该方法`调用`事件多路分发器的 `register_event`方法来 往事件多路分发器中注册一个事件  
remove_handler : 该方法`调用`事件多路分发器的`remove_event`方法来删除事件多路分发器中的一个事件 
  
  



