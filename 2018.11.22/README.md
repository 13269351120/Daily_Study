##### day day up !!! 


### 模板元编程   
昨天看到 分支执行的代码 , 使用的是 模板的特化 或 部分特化 来实现分支 , 那个代码我还没有完全的吃透 , 也不知道为什么 内外两层template的顺序不能搞错      
```cpp  
template <typename T>
template <bool AddOrRemove>
using Fun = typename Fun_<AddOrRemove>::template type<T> ;
```  

#### 使用std::conditional 和 std::conditional_t 实现分支 
其源代码:  
```cpp
namespace std
{
template <bool B , typename T , typename F>
	struct conditional {using type = T ; } ;
 
//模板偏特化  
template <typename T,typename F> 
	struct conditional<false,T,F> { using type = F ;} ;
	
template <bool B , typename T, typename F>
using conditional_t = typename conditional<B,T,F>::type ;
	
}
```  
**其典型的使用方式** : 
```cpp
std::conditional<true,int,float>::type x = 3 ;  
std::conditional_t<false,int,float>::type y = 1.0f ;  
```

`conditional` 和 `conditional_t` 的优势在于使用比较`简单` , 但缺点是 `表达能力不强` : 它只能实现`二元分支`(真假分支 ) , 其行为更像运行期 的问号表达式  x = B ? T : F ;  
对于`多元分支` 类似于 switch 的功能 支持起来就比较困难了 , 所以 `conditional` 和 `conditional_t`的使用场景是相对比较少的, 除非是特别简单的分支情况 , 否则并不建议使用这两个元函数    

#### 使用(部分)特化实现分支   
在前文中 已经有使用特化 来实现分支的情况 , (部分)特化 本来就是设置用来引入差异的  因此使用它来实现分支也是十分自然的  
考虑 :  
```cpp
struct A ; struct B ; 

template <typename T>  
struct Fun_{ constexpr static size_t value = 0 ; } ;

template <> 
struct Fun<A>{ constexpr static size_t value = 1 ; } ;

template <> 
struct Fun_<B> { constexpr static size_t value = 2 ; } ;

constexpr size_t h = Fun_<B>::value ;   
```  
好久没有看到`constexpr` , 之前不需要用到 `constexpr` 的原因是 模板的内部类型`type`  , 而这里定义了 一个变量 所以要 用 `constexpr static`  

在`C++14`中 ,  除了可以使用上述方法进行特化 , 还可以有其他的特化方式 , 考虑 :   
```cpp
struct A ; struct B ;

template <typename T>  
constexpr size_t Fun = 0  ;  

template <>  
constexpr size_t Fun<A>  = 1 ;  

template <>  
constexpr size_t Fun<B>  =  2;  

constexpr size_t h = Fun<B>  ;    
```  
与上一段代码进行比较 :  
后者实现简单 , 如果希望分支返回的结果是单一的数值 , 则可以考虑这种方式 , 前者给出了 依赖型名称 :: value  
  
#### 使用std::enable_if 与 std::enable_if_t 实现分支  
看了 , 明天再总结吧~  



















