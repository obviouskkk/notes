## More_Effective_C++
@(C++)

[TOC]

### 基础
#### 指针和引用
- 指针可以指向空值，可以被重新赋值；引用不可以
- 存在不指向任何对象的可能，或者需要在不同的时刻指向不同的内容，这个时候用指针。
- 重载操作符时或者不改变指向的时候，应该使用引用。
#### 尽量使用C++的类型转换
安全，编译器会帮你检查。
#### 不要对数组使用多态
- 如果函数参数为父类型A，实际传入参数为子类型B，那么编译器不会报错或者警告
- 但是因为数组每次向后，都跳过一个sizeof(A)，但数组元素实际类型为B，造成越界。

一个具体的例子：[数组和多态](https://github.com/obviouskkk/codes_cplusplus/blob/master/More_effective_cpp/item_3_ArrayPolymorphism.cpp)

#### 避免无用的缺省构造函数
- 没有默认构造函数的话，`B arrb[10]`或者`B * pb = new B[10]` 不能调用构造函数。解决方法：`B b[] = {B(1), b(2)...}`或者利用指针数组来代替对象数组，即
```
typedef B* pb;
pb arrayPb[10];      //不调用构造函数
// 或者 pb * arraypb = new pb[10]; // 也不调用构造函数
for (int i = 0; i < 10; ++i)
{
	arraypb[i] = new B(i); // 调用构造函数
} 
```
上面的第二种方法容易内存泄露。更好的方案是operator new[]

- 没有默认构造函数的第二个问题：无法在很多基于模板的容器类中使用，要求`data = new T[size]`,当然`vector`不需要。


### 运算符
