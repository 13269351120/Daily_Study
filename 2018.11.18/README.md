## C++Primer 16章 模板与泛型编程  
`引言` : 面向对象编程OOP 和 泛型编程 GP 都能处理 编写程序时不知道类型的情况, 不同之处在于 OOP能处理类型在程序运行之前都未知的情况 , 而在泛型编程中 在编译的时候 就能获知类型了 , 这对于 运行时效率是比较友好的  

### 定义模板
#### 函数模板
template <模板参数列表> : 模板参数列表不能为空!   
通过编译器生成的具体的版本通常被称为模板的实例   

#### 模板类型参数    
一般而言,我们见到的大多都是模板类型参数.定义的是一个类型 T   

#### 非类型模板参数  
通过一个特定的类型名而非关键字class 或 typename 来指定非类型参数 这种情况之前出现的比较少 , 我在 `元编程` 里看到过 就特地重温了指出了这一点  
例如  
```cpp
template<unsigned N,unsigned M> 
int comp(const char (&p1)[N] , const char (&p2)[M] )  
{
	return strcmp(p1,p2);  
} 
```
***
### std::move是如何工作的
我们不能讲一个右值引用绑定到一个左值上 , 但可以使用 move 获得一个绑定到左值上的右值引用 , 这也是 move函数的作用, move 本质上可以接受任何类型的实参 , 因此它是一个函数模板 

#### std::move的定义:
标准库源码分析:
```cpp
template <typename T>
typename remove_reference<T>::type&& move(T &&t)
{
	return static_cast<typename remove_reference<T>::type&&>(t);
}
```  

要想看懂这段代码 需要了解 `remove_reference`
这个 涉及到 类型转换 , 模板定义在 头文件 `type_traits`中 , 我们可以使用 remove_reference来获得元素的类型 , remove_reference 是一个模板 , 它有一个 名为 `type`的公有成员 , 用来表示脱去引用的类型
举个例子:
```cpp
remove_reference<int&>::type = int 
```
更一般的, 给定一个 迭代器 beg 
```cpp
remove_reference<decltype(*beg)>::type 
```  
*beg 是一个左值 , decltype(左值) 会返回左值引用 , 就和上面 int& 是一个道理了 , 
但是需要注意的是 , 为了使用 模板参数的成员 必须使用 typename来告知编译器 type是一个类型
例如
```cpp
template <typename It>
auto fun(It beg, It end) -> typename remove_reference<delctype(*beg)>::type
{
	return *beg ;
}
```  

好的 , 现在 我们再回过头去看 move的源代码 , 这一次 函数的 第一行能够看懂了 ,再来看函数体内部 , 函数的参数一个 指向模板类型参数的右值引用`T&&`  , 此参数可以和 **任何类型进行实参匹配** , 为什么呢? 它能够接受右值 毫无争议 , 它为什么也能够接受左值呢? 我们可以将 T&& 看成是 
T& &t , 传入的是一个左值 引用,T就会被推导成 左值引用 , 然后 在函数体内部 可以通过 static_cast 将类型强转为 T&&的 , 这边也利用了 右值引用的一个特许规则 , 虽然不能隐式的将一个左值转换为右值引用 但我们可以用`static_cast` 显式的将一个左值转换为一个右值引用 , **那么我在思考 move 其实就是一个 给static_cast加了一层外衣 , 统一了 右值 和 左值 到 右值引用的转换 .**
将一个 右值引用绑定到左值的特性 允许他们`截断左值` , 截断左值的 大致意思是 : 这个左值 未来将**不再使用**了 , **通过一个 右值来截断** , 这个左值名 所对应的内存 不归它管理了 .  
***
## 元编程  
`引言:`  C++元编程是一种典型的函数式编程 , 实质是一种编译期计算 , 这种方式可以对程序在运行期的优化 , 如何去更好的理解这里的编译期计算呢? 举一个例子 , 我们使用C++ , 使用到的OOP面向对象的三大特性 : `封装` `多态` `继承` ,  有这样一种情况 : 函数的参数类型 是 一个 基类 或者是抽象基类 , 传入实参是其 派生出的子类对象 , 这个时候 , 我们可以利用 多态 来使用 基类指针 调用 子类的虚函数 , 这种特性是很好的 , 很多设计模式都基于这个多态的特性 例如 工厂模式 , Template Method 等 , 但是 我们也必须考虑到 多态是一种 `RTTI` , 它有一定的性能损耗 , 试想一种可能: 我们传入的一个参数 是 很特殊的 , 比如 基类是一个 矩阵类 , 传入的是一个 单位矩阵 或者 是 全0矩阵 , 这个时候 函数的加减 乘除 是可以有一定的优化的 , 那么如何通过 多态来 确定参数类型呢 ?  这里可以通过 `dynamic_cast` 来试图 转换类型 , 如果说可以转 , 那么就会返回 非空指针 :    
```cpp
Matrix Add(const AbstractMatrix * mat1 , const AbstractMatrix * mat2)
{
	if(auto ptr = dynamic_cast<const ZeroMatrix*>(mat1) )
	{...}
	else if()
	{...}
}
```  
看上去 我们已经解决了这样一个问题 , 但是会随着 特殊情况的增多 代码量和 判断量要上升 , 不是 长久之计 , 这个时候 我们就会 使用 函数的`重载` , 从本质上来说 函数的重载 将类型的判断 从 `运行期` 提前到 `编译期` , 那么可能会想到 既然 重载可以解决特殊问题 , 那我全都使用 重载的方法来适应不同的情况好了 , 考虑比如数的相加这样一个函数 , 数据类型 有 `int` `float` `double` 甚至会有 `complex` 等等 , 那么定义 重载函数 , 需要定义许多许多个 , 不切实际 , 这个时候 引入了 模板 ,模板是一种通用的形式 , **和生活哲理一样 , 通用的东西 往往不是最优的** ,  而在模板里 要处理 特殊的情况 , 就是会用到 模板的特化 ,这些东西 现在还没有深入,这边留一个坑给自己 ,未来 来填 .  

---
元编程中 使用的函数 更接近`数学意义上的函数` :是一种`无副作用`的映射或变换 ,即 在输入相同的前提下 , **多次调用同一个函数  得到的结果也是相同的** 
那么 为什么需要 无副作用呢 ? 我们使用 元编程 是想要在编译期进行计算 , 编译阶段 , 编译器只能构造常量作为其中间结果 无法构造并维护可以记录系统状态并随之改变的量
```cpp
constexpr int fun(int a) {return a + 1 ;}  
```
其中`constexpr`是C++11中的关键字 , 表明这个函数可以在编译期被调用 , 是一个 元函数 , 如果去掉了这个`constexpr`关键字 , fun函数虽然是一个 无副作用的函数 ,但是也只能在运行期使用   
说了 半天的无副作用 这里举一个有副作用的函数
```cpp
static int call_count = 0 ;  
constexpr int fun(int a){ return a+(call_count++) ;}
```
这个程序无法通过编译,因为这里的`call_count`会改变 每一次调用fun函数 输入相同的情况下 输出也会有所不同 , 所以 这是一个 有副作用的函数 .   

事实上 , 上述的值计算的元函数只是一小部分 , C++中用得更多的类型原函数 , 即 输入为类型 输出也是类型的元函数 .  






