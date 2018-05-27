---
title: 左值、右值和引用
date: 2018-05-25 21:17:37
categories: c++
tags: c++11
---

## 左值、右值和引用
[TOC]

C++( 包括 C) 中所有的表达式和变量要么是左值，要么是右值。可以用以下方式理解左值和右值：
#### 左值
- 左值的定义就是非临时对象，那些可以在多条语句中使用的对象。所有定义的变量都满足这个定义。
- 一个有用的，有启发意义的判断一个表达式是左值的方法是取它的地址。如果可以取地址，它基本上就是一个左值。如果不行，通常来说是一个右值。（Effective Modern C++）
#### 右值
- 一般没有名字
- 计算的中间过程，价值只是为了`复制`到一个有名字的对象。
- 右值是指临时的对象，它们只在当前的语句中有效。
- 一般函数返回值是右值。
<!-- more -->
#### 一些举例
```cpp
A a1 = GetA();                                  // a1是左值，GetA()返回的结果是右值
int x, y;
x+y;                                            // x+y的结果是右值(甚至没有办法叫出名字)
```

### 左值引用和右值引用
#### 左值引用
```cpp
int& a = 5;              // a是一个左值引用
int GetX(int & x);      // x算是一个左值引用，虽然它是一个形参   L1
```
对于上面的`GetX`函数，显然
```cpp
int x = 0;
int y = Get(x);      // ok
int z = Get(Get(x))  // error，no matching，里面的函数返回的是int &&
```

#### 右值引用
左值的声明符号为”&”， 为了和左值区分，右值的声明符号为”&&”。
```cpp
int && a = 5;        // 右值引用，可以对一个右值引用取地址   L2
int GetX(int && x);
```
对于上面的`GetX`函数，也显然
```cpp
int x = 0;
int y = Get(x);      // error，no matching ，x：int &
int && a = 1;  // no matter
int && b = x; // error
```
#### 左值引用和右值引用
- 对函数的右值引用无论具名与否都将被视为左值
- 具名右值引用被视为左值，如`int & a = 666;`，显然，它可以match到`int fun(int x)`;
- 无名对对象的右值引用被视为x值
- 对函数的右值引用无论具名与否都将被视为左值,例如：`int && a = 5; int x = 1; int && b = x;`,a和b都可以作为左值。

##### 参数都是左值
不管他是一个左值还是一个右值。也就是说，给定一个类型T，你可以得到类型T的左值同时也可以得到它的右值。当处理一个有右值引用的参数时需要铭记于心，因为参数本身是个左值：可以取地址。
类似的原因，所有的参数都是左值。
实参和形参的区别是很重要的，因为形参只能是左值，但是给他们初始化的实参即有可能是右值也有可能是左值，例：实参`std::move(wid)`是右值，但是形参`wid`是左值，

#### 一个抄来的例子
```cpp
void process_value(int& i) { 
 std::cout << "LValue processed: " << i << std::endl; 
} 
 
void process_value(int&& i) { 
 std::cout << "RValue processed: " << i << std::endl; 
}  
int main() { 
 int a = 0; 
 process_value(a); 
 process_value(1); 
}
```
运行结果：
```cpp
LValue processed: 0 
RValue processed: 1
```
Process_value 函数被重载，分别接受左值和右值。由输出结果可以看出，临时对象是作为右值处理的。
#### 通用引用
注意下面这个模板函数
```cpp
template<typename T>
int fun(T&& x) {
    // ...
    return 0;
}
```
- 注意：`T &&` 不一定是一个右值引用，他其实是一个通用引用（叫法来自：Effective Modern C++ item24）
因为以下调用方式居然都是成立的：
```cpp
// fun
int x = 1;
int y = fun(x);
int & a = x;
y = fun(a);
int && b = 1;
y = fun(b);
```
编译器推导出了些什么鬼东西，请看引用折叠
### 引用折叠
对于引用有以下消除特性：
```
A& & 变成 A&
A& && 变成 A&
A&& & 变成 A&
A&& && 变成 A&&
```
简单结论：
- 就是左值引用会传染，只有纯右值&& && = &&，沾上一个左值引用就变左值引用了，根本原因就是：在一处声明为左值，编译器就必须保证此对象可靠（左值）。
- 引用折叠会出现在4中上下文：模板实例化，auto类型生成，typedef和类型别名声明的创建和使用，decltype。

那么，对于上面的`int (T&& x)`的几种调用方式就有：
```cpp
int x = 1;
int y = fun(x);   // T被推导为int &，int & && 折叠为int&
int & a = x;      // 
y = fun(a);       // T被推导为int &，int & && 折叠为 int &
int && b = 1;
y = fun(b);       // T被推导为int，int &&
```
