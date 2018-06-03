---
title: std::forward详解
date: 2018-06-03 22:28:37
categories: c++
tags: c++11
---
`std::forward`比`std::move`逻辑略复杂，`std::move`是无条件把参数转换为右值，  
而std::forward在特定情况下才会这样做：仅当参数是用右值初始化时，才会把它转换为右值。使用`std::forward`来转发参数一般被称为完美转发(也叫精确传递)。  
### 完美转发
参数的属性通常包括：左值／右值和 const/non-const。 完美转发就是在参数传递过程中，所有这些属性和参数值都不能改变。在泛型函数中，这样的需求非常普遍。  

- std::forward的大致实现如下：
```cpp
template<typename T>          // 在命名空间std中
T&& forward(typename remove_reference<T>::type& param)
{
    return static_cast<T&&>(param);
}
```
一个模板函数接受全局引用，然后用std::forward把参数传递给另一个函数。

- 如果传递给函数f的是个左值的Widget，T会被推断为Widget&：
```cpp
Widget& && forward(typename remove_reference<Widget&>::type& param)
{
    return static_cast<Widget& &&>(param);
}
// remove_reference<Widget&>::type产生的是Widget
Widget& && forward(Widget& param)
{    
	return static_cast<Widget& &&>(param);
}
// 发生引用折叠，最终版本的std::forward：
Widget& forward(Widget& param)
{    
	return static_cast<Widget&>(param);    
}
// 这次转换没造成任何影响
```
- 假设传递给函数f的是个右值的Widget,函数f的类型参数T会被推断为Widget  
```cpp
Widget&& forward(typename remove_reference<Widget>::type& param)
{
    return static_cast<Widget&&>(param);
}
// std::remove_reference会产生原来的类型（Widget）
Widget&& forward(Widget& param)
{    
	return static_cast<Widget&&>(param);
}
// 不存在引用折叠，这就是最终版本
```
- 由函数返回的右值引用被定义为右值，在这种情况下，std::forward会把f的参数fParam（一个左值）转换成一个右值

#### 是否可以摒弃std::move，只是用std::forward？
从纯粹的技术角度看，答案是可以的：std::forward可以应付所有场景，std::move不是必须的。

#### 例子
```cpp
Widget widgetFactory();      // 返回右值的函数
Widget w;       // 一个变量，左值
func(w);      // 用左值调用函数，T被推断为Widget&
func(widgetFactory());     // 用右值调用函数，T被推断为Widget
```
下面又一些调用:
```cpp
auto&& w1 = w; // w是个左值，auto被推导成了 Widget&，折叠成 Widget& w
auto&& w2 = widgetFactory(); // 函数返回时右值，右值初始化右值，auto推导为：Widget。
```
- typedef得到的，可能并不是你想要的类型,例如：
```cpp
template<typename T>
class Widget {
public:
    typedef T&& RvalueRefToT;
    ...
};
Widget<int&> w;
// typedef int& && RvalueRefToT; --引用折叠-->typedef int& RvalueRefToT;
```

### 避免对通用引用重载
我认为这里应当成为特化：
当一个模板实例化函数和一个非模板函数（即，一个普通函数）匹配度一样时，优先使用普通函数。在相同的签名下，拷贝构造（普通函数）胜过实例化模板函数。
- 一个例子：
```cpp
template<typename T>
void add(T&& rhs) {
    // ... do something
    std::string(rhs);
}

void add(int i) {
    // ..
}

// 以下：
add<std::string>("hello world");  // ok ！
int a = 1;
add<std::string>(a); // ok
short b = 2;
add<std::string>(b); // error,试图构造std::string
```

接受通用引用作为参数的函数是C++最贪婪的函数，它们可以为几乎所有类型的参数实例化，从而创建的精确匹配。  
这就是为什么结合重载和通用引用几乎总是个糟糕的想法：通用引用重载吸收的参数类型远多于开发者的期望。

#### 对右值引用使用std::move，对通用引用使用std::forward
右值引用：`Weight&&` ，类型确定，
通用引用：`typename &&`，可能会被推导成左值
- 如果确定是个右值引用，那么使用std::move处理它，`name(std::move(rhs.name)`;
- 如果它有可能是个右值引用，那么可以使用`std::forward`转发它，`name = std::forward<T>(newName);`
- 对右值引用使用std::forward会表现出正确的行为，但是源代码会是冗长的、易错的、不符合语言习惯的，因此你应该避免对右值引用使用std::forward。
- 不要对`T &&`使用`std::move`，因为很可能导致传进去的左值丢失。

#### 不要用左值和右值重载：只有通用引用才是正确的方式
```cpp
class Widget {
public:
    void setName(const std::string& newName)  // 由const左值设置
    { name = newName; }

    void setName(std::string&& newName)  // 由右值设置
    { name = std::move(newName); } 
    // ...
};
```
这种写法是没有问题的，左值由拷贝构造函数执行；右值由移动赋值构造函数执行。
但是重载代价比较大，合适的写法是这样的。
```cpp
class Widget {
public:
void setName(T&& newName)  // 由右值设置
    { name = std::forward<std::string>(newName); } 
    // ...
};
```
#### 安全的使用std::move和std::forward
```cpp
template<typename T>
void setSignText(T&& text)   // text是个通用引用
{ 
    sign.setText(text);      // 使用text，但不修改它

    auto now = std::chrono::system_clock::now();  // 获取当前时间

    signHistory.add(now, std::forward<T>(text));  // 有条件地把text转换为右值
}
```
#### 函数返回值不需要std::move
在分配给函数返回值的内存中直接构造w，从而避免拷贝局部变量w:return value optimization（RVO）.
这个在C++11以前就明文规定的了。因此放心的创建临时变量，返回吧。再通过std::move赋值给另一个它，只调用了一次构造，一次移动构造函数哦。


### 熟悉替代重载通用引用的方法
- 通过const T&传递参数  
可以的。// 一次拷贝构造函数
- 通过值传递对象，  
```
explicit Person(std::string n)  // 替换 T&& 构造函数
    : name(std::move(n)) {}    // 使用std::move` 
```
- 重载多一个参数的版本，避免匹配到通用引用  
这个写起来比较麻烦。item 27
- 约束接受通用引用的模板`std::enable_if`  
```cpp
class Person {
public:
    template<typename T,
             typename = typename std::enable_if<condition>::type>
    explicit Person(T&& n);

    ...
};
```
- `std::is_same`可以用来判断两个类型是不是一样。
- `std::decay<T>::type`用来去掉那些const、volatile、引用。其实，std::decay就像它的名字表示那样，也可以把数组和函数类型转换成指针类型（看条款1），不过对于这里的讨论，std::decay的行为和我描述的一样。
- `std::is_base_of<T1, T2>::value`,当T2是由T1派生而来时，结果为true。
上面的那个condition可以确定了，就是
`!std::is_same<Person, typename std::decay<T>::type>::value`.  

- 完美的person转发构造
```cpp
class Person {
public:
    template<
      typename T,
      typename = std::enable_if_t<
        !std::is_base_of<Person, std::decay_t<T>>::value
        &&
        !std::is_integral<std::remove_reference_t<T>>::value
      >
    >
    explicit Person(T&& n)       // 接受std::string类型
    : name(std::forward<T>(n))   // 和 可以转换为string的参数的构造函数
    { 
        // 断言可以从T对象构造一个std::string
        static_assert(
          std::is_constructible<std::string, T>::value,
          "Parameter n can't be used to construct a std::string"
        );
        // ...
     }

    explicit Person(int idx)     // 接受整型参数的构造函数
    : name(nameFromIdx(idx))
    { ... }

    ...    // 拷贝和移动构造， 等等
    
private:
    std::string name;
};
```

#### 总结：　　
设计类的时候如果有动态申请的资源，也应该设计转移构造函数和转移拷贝函数。  
在设计类库时，还应该考虑 std::move 的使用场景并积极使用它。  
std::move和std::forward在运行期都没有做任何事情。因为类型在编译期就已经确认，类型转换在编译期就已经完成。