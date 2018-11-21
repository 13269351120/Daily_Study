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
       



