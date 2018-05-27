---
title: std::move
date: 2018-05-25 21:28:37
categories: c++
tags: C++11
---

### 新特性的目的
右值引用 (Rvalue Referene) 是 C++11中引入的新特性 , 它实现了转移语义 (Move Sementics) 和精确传递 (Perfect Forwarding)。它的主要目的有两个方面：
- 消除两个对象交互时不必要的对象拷贝，节省运算存储资源，提高效率。
- 能够更简洁明确地定义泛型函数。
<!-- more -->
### std::move
首先，std::move() 并没有实际移动任何东西。
std::move的大致实现：
```cpp
template<typename T>
typename remove_reference<T>::type&& move(T&& param) {
    using ReturnType = typename remove_reference<T>::type&&;
    return static_cast<ReturnType>(param);
}
```
可以看到：
- `std::move`仅仅是进行了类型转换
- 用`std::remove_reference`去掉T身上的引用，请看[引用折叠](https://blog.csdn.net/o_bvious/article/details/80315806#%E5%BC%95%E7%94%A8%E6%8A%98%E5%8F%A0)。
- move的名字其实更接近于`rvalue_cast`,把值转换为左值。
- 请不要误解为move接受的参数类型就是右值引用，实际上它是一个[通用引用](https://blog.csdn.net/o_bvious/article/details/80315806#%E9%80%9A%E7%94%A8%E5%BC%95%E7%94%A8)。

避免了不必要的深拷贝。
当需要将某个对象的内容“转移”到其他位置时，您可以使用移动，而不进行复制（例如内容不重复，这就是为什么它可以用于某些不可复制的对象，如unique_ptr）。  
也可以使用std :: move来获取临时对象的内容，而无需复制（并节省大量时间）。  
它只是向编译器发出信号，程序员不再关心该对象发生了什么。

##### 意义
想一下上面的`string a(x);`想一下下面的场景：
- x是string类型，而构造x的目的如果仅仅是为了拷贝构造a
- 构造玩a，x就失去了意义，可以被随意折腾
- 现在有一个高效的移动构造函数：
```cpp 
string(string&& that)   // rvalue reference
{
    data = that.data;
    that.data = nullptr;
}
```
那么显然x目前是一个左值（俺有名字），无法match到参数为(string &&x)，到了`std::move`大显神威的时候了。
##### 功能
类型转换：获得一个右值，右值意味着告诉编译器：你可以使用、移动、侵占我所有的资源，因为我只是个右值。

#### 移动构造函数
工作是将资源从一个对象移到另一个对象而不是复制它们。
复制构造函数会生成深层副本，因为源必须保持不变。另一方面，移动构造函数只能复制指针，然后将源中的指针设置为null。
移动对象意味着将其管理的某些资源的所有权转移给另一个对象。想想智能指针。
- 下面是`std::string`的可能声明，注意move构造不带const（编译器要把参数掏空无法const）
- 加了const，就会调用上面的copy构造。
```cpp
string(const string& rhs);        //copy ctor
string(string&& rhs);            //move ctor
```
#### 其他问题

- C++11中`return local_var`与`return std::move(local_var) `效果是否相同？

编译器就把优化几乎做到了极致——局部变量返回到函数外部并赋值给外部变量这个过程基本上不存在任何多余的临时变量构造和析构，这比move机制更加高效。



