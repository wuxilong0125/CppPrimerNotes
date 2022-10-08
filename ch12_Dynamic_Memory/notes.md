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

## 直接管理内存

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

##### 动态分配的const对象

```cpp
const int *pci = new const int(1024);
const string *pcs = new const string;
```

**动态分配const对象必须初始化，有默认构造函数的类类型可以隐式初始化，其他类型需要显示初始化**

##### 内存耗尽

```cpp
int *p1 = new int; // 如果分配失败，new抛出std::bad_alloc
int *p2 = new (nothrow) int; // 如果分配失败，new返回空指针
```

##### 释放动态内存

```cpp
delete p;
// p必须指向一个动态分配的对象或是一个空指针
```

##### 指针值和delete

传递给delete的指针必须指向动态分配的内存，或者是一个空指针。

```cpp
int i, *pi1 = &i, *pi2 = nullptr;
double *pd = new double(33), *pd2 = pd;
delete i; // 错误，i不是一个指针
delete pi1; // 未定义，pi1指向一个局部变量
delete pd; // 正确
delete pd2; // 未定义，pd2指向的内存已经被释放了
delete pi2; // 正确，释放一个空指针总是没有错误的
```

##### 动态对象的生存期直到被释放时为止

由shared_ptr管理的内存在最后一个shared_ptr销毁时会被自动释放。但对于通过内置指针类型来管理的内存就不是这样了，**内置指针管理的动态对象，直到被显式释放之前它都是存在的。**

```cpp
Foo* factory(T arg)
{
    ...
    return new Foo(arg); 
}

void use_factory(T arg)
{
    Foo *p = factory(arg);
    // 使用p但不delete它
}// p离开了它的作用域，但它所指向的内存没有被释放！



//修正
void use_factory(T arg)
{
    Foo *p = factory(arg);
    // 使用p
    delete p;
}
```

**由内置指针（而不是智能指针）管理的动态内存在被显式释放前一直都会存在**

## shared_ptr 和 new 结合使用

```cpp
shared_ptr<double> p1; // shared_ptr 可以指向一个double
shared_ptr<int> p2(new int(42)); // p2 指向一个值为42的int
```

**接收指针参数的智能指针构造函数是explicit的，因此，不能将一个内置指针隐式转换为一个智能指针，必须使用直接初始化形式来初始化一个智能指针**

```cpp
shared_ptr<int> p1 = new int(1024); //错误：必须使用直接初始化形式
shared_ptr<int> p2(new int(1024)); // 正确
```

`shared_ptr<T> p(q)`：p管理内置指针q所指向的对象；q必须指向new分配的内存，且能够转换为T*类型

`shared_ptr<T> p(u)`：p从unique_ptr u那里接管了对象的所有权；将u置为空

`shared_ptr<T> p(q, d)`：p接管了内置指针q所指向的对象的所有权。必须能转换为T*类型。p将使用可调用对象d来代替delete

`shared_ptr<T> p(p2, d)`：p是shared_ptr p2的拷贝，唯一的区别是p将用可调用对象d来代替delete

`p.reset()`：若p是唯一指向其对象的shared_ptr，reset会释放此对象。

`p.reset(q)`：若传递了可选的参数内置指针q，会令p指向q，否则会将p置为空。

`p.reset(q,d)`：若传递了参数d，将会调用d而不是delete来释放q

#### 智能指针的get方法

get用来讲指针的访问权限传递给代码，你只有在确定代码不会delete指针的情况下，才能使用get。特别是，永远不要用get初始化另一个智能指针或者为另一个智能指针赋值。

```cpp
shared_ptr<int> p(new int(42)); // 引用计数为1
int *q = p.get(); //正确，但使用q时要注意，不要让它管理的指针被释放
{
    未定义：两个独立的shared_ptr指向相同的内存
    shared_ptr<int>(q);
}// 程序块结束，q被销毁，它指向的内存被释放
int foo = *p; // 未定义：p指向的内存已经被释放
```

## unique_ptr

一个unique_ptr “拥有” 它所指向的对象。unique_ptr 不支持普通的拷贝和赋值操作。

初始化unique_ptr必须采用直接初始化形式：

```cpp
unique_ptr<double> p1; // 可以指向一个double的unique_ptr

unique_ptr<int> p2(new int(42)); //p2指向一个值为42的int


unique_ptr<string> p1(new string("Stegosaurus"));
unique_ptr<string> p2(p1); // 错误：unique_ptr 不支持拷贝
unique_ptr<string> p3;
p3 = p2; //错误：unique_ptr 不支持赋值
```

### unique_ptr操作

```cpp
unique_ptr<T> u1; //空unique_ptr，可以指向类型为T的对象，
                  //u1会使用delete来释放它的指针
unique_ptr<T, D> u2; // u2会使用一个类型为D的可调用对象来释放它的指针
unique_ptr<T, D> u(d); // 空 unique_ptr，指向类型为T的对象，
                       // 用类型为D的对象代替delete
u = nullptr // 释放u指向的对象，将u置为空

u.release() // u放弃对指针的控制权，返回指针，并将u置为空

u.reset() //释放u指向的对象
u.reset(q) // 如果提供了内置指针q，令u指向这个对象，否则u置为空
```



虽然我们不能拷贝或赋值unique_ptr，但可以通过调用release或reset将指针的所有权从一个(非const)unique_ptr转移给另一个unique：

```cpp
// 将所有权从p1转给p2
unique_ptr<string> p2(p1.release()); // release将p1置为空
unique_ptr<string> p3(new string("Trex")); 
p2.reset(p3.release()); // 将所有权从p3转移给p2，
                        // reset释放了p2原来指向的内存
```

调用release会切断unique_ptr和它原来管理的对象间的联系。release返回的指针通常被用来初始化另一个智能指针或给另一个智能指针赋值。

`p2.release()` 错误，p2不会释放内存，而且我们丢失了指针

`auto p = p2.release()` 正确，但我们必须记得delete(p)



### 传递unique_ptr参数和返回unique_ptr

不能拷贝unique_ptr的规则有一个例外：我们可以拷贝或赋值一个将要被销毁的unique_ptr。最常见的例子是从函数返回一个unique_ptr:

```cpp
unique_ptr<int> clone(int p) {
    // 正确：从int*创建一个unique_ptr<int>
    return unique_ptr<int>(new int(p));
}
```

还可以返回一个局部对象的拷贝：

```cpp
unique_ptr<int> clone(int p) {
    unique_ptr<int> ret(new int (p)); 
    // ...
    return ret;
}
```

### 向unique_ptr传递删除器

默认情况下，unique_ptr用delete释放它指向的对象。在创建或reset一个这种unique_ptr类型的对象时，必须提供一个指定类型的可调用对象(**删除器**)

```cpp
// p 指向一个类型为objT的对象，
// 并使用第一个类型为delT的对象释放objT对象
// 它会调用一个名为fcn的delT类型对象。
unique_ptr<objT, delT> p (new objT, fcn);
```

## weak_ptr

weak_ptr是一种不控制所指向对象生存期的智能指针，它指向由一个shared_ptr管理的对象。将一个weak_ptr绑定到一个shared_ptr不会改变shared_ptr的引用计数。一旦最后一个指向对象的shared_ptr被销毁，对象就会被释放。即使有weak_ptr指向对象，对象也还是会被释放，因此，weak_ptr的名字抓住了这种智能指针“弱”共享对象的特点。

```cpp
weak_ptr<T> w; // 空weak_ptr可以指向类型为T的对象
weak_ptr<T> w(sp); // 与shared_ptr sp指向相同对象的weak_ptr。
                   // T必须能转换为sp指向的类型
w = p; //p可以是一个shared_pte 或 一个weak_ptr。赋值后w与p共享对象。
w.reset() // 将w置为空
w.use_count() // 与w共享对象的shared_ptr的数量
w.expired() // 若w.use_count() 为0，返回true，否则返回false
w.lock() // 如果expired为true，返回一个空shared_ptr；
         // 否则返回一个指向w的对象的shared_ptr
```

当我们创建一个weak_ptr时，要用一个shared_ptr来初始化它：

```cpp
auto p = make_shared<int>(42);
weak_ptr<int> wp(p); // wp弱共享p；p的引用计数未改变
```

### 核查指针类
