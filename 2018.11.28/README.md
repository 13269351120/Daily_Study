##### 继续学习~~~~ 加油

#### 模板元编程
接上次的 分支执行代码   
```cpp
namespace std
{
	template <bool B , typename T = void>
	struct enable_if {} ;
	
	template <class T>
	struct enable_if<true,T> { using type = T ;} ;
	
	template <bool B , class T = void > 
	using enable_if_t = typename enable_if<B,T>::type ;
}
```  
对于分支而言 , T并不特别重要 重要的是 当`B = true` 的时候 , `enable_if` 元函数 可以返回结果 `type` , 可以基于这个构造实现分支  
```cpp
template <bool IsFeedbackOut , typename T , std::enable_if_t<IsFeedbackOut>* = nullptr>
auto FeedbackOut_(T &&) { .... } 

template <bool IsFeedbackOut ,typename T , std::enable_if_t<!IsFeedbackOut>* = nullptr> 
auto FeedbackOut_(T&&) { .... }
```
当`IsFeedbackOut= ture`时 ,  `std::enable_if_t<IsFeedbackOut>*` 是有意义的 , 这就使得 第一个函数`匹配成功` , 与之对应的 , 第二个函数`匹配是失败`的 , 反之 `IsFeedbackOut = false` 时 . `std::enable_if_t<!IsFeedbackOut>*` 是有意义的 , 所以 第二个函数匹配成功  
C++中有一个**特性**是 `SFINAE(Substitution Failure is not an error)` , **匹配失败并非是错误** , 编译器会选择 匹配成功的函数 而不会报告 错误 . 
##### 这里自我总结一下 : 
`enable_if` 是通过 第一个参数 `true or false` 来控制 模板的内置` type` 是否有意义 , 再根据这个`type` 进行 分支的选择 , 编译器自动选择 匹配成功的 函数 .  
个人理解而言 , 目前还很浅陋 , 基于 enable_if 的例子就是一个典型的 编译期 和 运行期 结合的使用方式 , FeedBackOut_ 包含了 运行期的逻辑 , 而选择哪个 FeedbackOut_则是通过编译期的分支来实现的 , 到目前为止 还没发现 这种做法 相较于 特化实现分支的好处???   

> * 编译期分支实现方式 	 优点	     缺点   
> * 模板的特化或者偏特化	书写自然,容易理解	代码比较长    
> * std::enable_if	    很灵活	    不直观   

#### 编译期分支 与 多种返回类型  

考虑  
```cpp
auto wrap1(bool check)
{
	if(check) return (int) 0 ;
	else return (double) 0 ; 
}
```
这里要注意 在C++14中 函数声明中可以不用显式指明其返回类型 , 编译器可以根据函数体中的return语句来自动推导其返回类型 , 但**要求函数体重的所有 return 语句所返回的类型均相同 !!!**.  
对于以上代码 , `返回的类型会不同` 所以 编译会出错 , 事实上, 在运行期的函数 , 其返回类型在编译期就已经确定了 , 无论采用何种写法, 都无法改变   
`想一想` : 多态的机制 , 典型的 多态new 而产生的工厂模式中 , 工厂会 有一个纯虚函数 返回 一个 抽象基类, , 而 通过继承 这个工厂类 派生出的 具体工厂 会 实现这个 纯虚函数 返回 一个 具体类 , 这有一个好处 用抽象工厂基类的虚方法 利用多态的机制 能够 返回具体类 并用 基类指针指代 即
```cpp
base* p = concreteFactory->create() ;  
```  
这减低了程序的耦合性 , 让程序 可以在编译时 关系分离 , 而在运行时 ??? 运用多态 RTTI  ?? **这一段自己感觉写的不好**     
这是一种 利用 多态 实现 运行时 返回多种类型的方法 , 但这种方法 返回的多种类型 是需要有一个基类的 , 感觉就是 返回的是 `'一家子'`而不像上述 通过编译期处理 就可以 使用同一个 接口来返回 多种类型 

##### 思考   
用一个 `bool`值 来控制 `enable_if` , 从而使用`SFINAE` 原理 , 是不是 只能产生两个分支呢?   

回到我们的编译期优化 , 
使用`enable_if`在编译期 , 就可以打破这样的限制 ,   
```cpp
template <bool check , std::enable_if_t<check>* =nullptr>
auto fun() {return (int)0 ; }  

template <bool check , std::enable_if_t<check> * = nullptr >
auto fun() { return (double) 0 ; }   
```
