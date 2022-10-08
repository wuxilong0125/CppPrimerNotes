# 程序内存分配

## 静态内存

静态内存用来保存局部static对象、类static数据成员以及定义在任何函数之外的变量。

static对象在使用之前分配，在程序结束时销毁。

## 栈内存

栈内存用来保存定义在函数内的非static对象。

栈对象仅在其定义的程序块运行时才存在。

**分配在静态内存或栈内存中的对象由编译器自动创建和销毁。**

## 自由空间（堆）

程序用堆来存储动态分配的对象。动态对象的生存期由程序来控制，也就是说当动态对象不再使用时，我们的代码必须显式地销毁它们。

# 动态内存与智能指针

c++中的动态内存管理是用**new**、**delete**运算符来完成的。

**new**，在动态内存中为对象分配空间并返回一个指向该对象的指针。

**delete**，接受一个动态对象的指针，销毁该对象，并释放与之关联的内存。

为了更容易更安全管理动态内存，新的标准库提供了两种智能指针**shared_ptr**和**unique_str**类型来管理动态对象。智能指针的行为类似常规指针，区别在于智能指针负责自动释放所指向的对象。

**shared_ptr**：允许多个指针指向同一个对象；

**unique_ptr**：“独占”所指向的对象。

**week_ptr**：弱引用，指向shared_ptr所管理的对象。

**三种类型都定义在头文件memory中**

## shared_ptr类

### 用法

```cpp
shared_ptr<string> p1;  //shared_ptr 指向string

shared_ptr<list<int>> p2; //shared_ptr 指向int的list

// 解引用*p1返回p1所指的对象
// 条件判断中使用智能指针，效果就是检测它是否为空
if (p1 && p1->empty()) //若p1不为空，检查它是否指向一个空string
{
    *p1 = "hi";
}
```

默认初始化的智能指针中保存着一个空指针。

| share_ptr  和 unique_ptr都支持的操作        |                                        |
|:------------------------------------:| -------------------------------------- |
| shared_ptr<T> sp    unique_ptr<T> up | 空智能指针，可以指向类型为T的对象                      |
| p                                    | 将p应作一个条件判断，若p指向一个对象，则为true             |
| *p                                   | 解引用p，获得它指向的对象                          |
| p->mem                               | 等价于(*p).mem                            |
| p.get()                              | 返回p中保存的指针。若指针指针释放了其对象，返回的指针指向的对象也就消失了。 |
| swap(p, q)    p.swap(q)              | 交换p和q中的指针                              |

| shared_ptr独有的操作       |                                                                              |
| --------------------- | ---------------------------------------------------------------------------- |
| make_shared<T> (args) | 返回一个shared_ptr，指向一个动态分配的类型为T的对象，使用args初始化此对象                                 |
| shared_ptr<T>p (q)    | p是shared_ptr q的拷贝；此操作会递增q中的计数器。q中的指针必须能转换为T*                                 |
| p = q                 | p和q都是shared_ptr， 所保存的指针必须能互相转换，此操作会递减p的引用计数，递增q的引用次数；若p的引用计数变为0，则将其管理的原内存释放。 |
| p.unique()            | 若p.use_count()为1，返回true，否则返回false                                            |
| p.use_count()         | 返回与p共享对象的智能指针数量，可能很慢，主要用于调试                                                  |

### make_shared函数

最安全的分配和使用动态函数的方法是调用一个名为make_shared的标准库函数。此函数在动态内存中分配一个对象并初始化它，返回指向此对象的shared_ptr。与智能指针一样，make_shared也定义在头文件memory中。

```cpp
//指向一个值为42的int的shared_ptr
shared_ptr<int> p3 = make_shared<int>(42);

shared_ptr<string> p4 = make_shared<string>(10, '9');

shared_ptr<int> p5 = make_shared<int>(); //值初始化，值为0

auto p6 = make_shared<vector<string>>();
```

### shared_ptr的拷贝与赋值

每个shared_ptr都有一个关联的计算器，通常称为**引用计数(reference count)**。拷贝一个shared_ptr，计数器都会递增。

```cpp
auto p = make_shared<int> (42); //p指向的对象只有一个引用者
auto q(p) //p和q指向相同对象，此对象有两个引用者

auto r = make_shared<int> (42);
r = q; 
// 给r赋值，令它指向另外一个地址，地址q指向的对象的引用计数，
// 递减r原来指向的对象的引用计数，r原来指向的对象的引用计数为0，自动释放
```

当shared_ptr对象的引用计数为0时会调用析构函数释放对象占用的内存， 同时释放关联的内存。

```cpp
shared_ptr<Foo> factory(T arg)
{
    //处理arg
    //shared_ptr 负责释放内存
    return make_shared<Foo> (arg);
}
void use_factory(T arg)
{
    shared_ptr<Foo> p = factory(arg);
    //使用 p

}    //p离开了作用域，它指向的内存会被自动释放掉
```

p是use_factory的局部变量，在use_factory结束时它将被销毁，当p被销毁时，将递减其引用计数检查它是否为0，在此例中，p是唯一引用factory返回的内存的对象，由于p将要销毁，p指向的这个对象也会被销毁，所占用的内存会被释放。如果有其他shared_ptr也指向这个块内存，它就不会被释放。

#### 使用了动态生存期的资源的类

程序使用动态内存出于以下三种原因之一：

1、程序不知道自己需要使用多少对象

2、程序不知道所需对象的准确类型

3、程序需要在多个对象间共享数据

**一般而言，如果两个对象共享底层的数据，当某个对象被销毁时，我们不能单方面地销毁底层数据**

```cpp
Blob<string> b1; //空Blob
{// 新作用域
    Blob<string> b2 = {"a", "an", "the"};
    b1 = b2;
} //b2被销毁了，但b2中的元素不能销毁

//b1指向最初由b2创建的元素。
```

#### 直接管理内存

##### 使用new动态分配和初始化对象

自由空间分配的内存是无名的，因此new无法为其分配的对象命名，而是返回一个指向该对象的指针`int *pi = new int;//pi指向一个动态分配的，未初始化的无名对象`

```cpp
string *ps = new string; // 初始化为空string
int *pi = new int;  // pi指向一个未初始化的int


int *pi = new int(1024);
string *ps = new string(10, '9');

// 花括号初始化
vector<int> *pv = new vector<int>{0, 1, 2, 3, 4, 5, 6, 7, 8, 9};

string *ps1 = new string; // 默认初始化为空string
string *ps = new string(); // 初始值为空string


int *pi1 = new int; // 默认初始化，*pi1的值未定义
int *pi2 = new int(); // 值初始化为0; *pi2为
```

使用auto从此初始化器来推断我们想要分配的对象的类型，但是，由于编译器要用初始化器的类型来推断要分配的类型，只有当括号中仅有单一初始化器时才可以使用auto：

```cpp
auto p1 = new auto(obj);
auto p2 = new auto{a, b, c};
```
