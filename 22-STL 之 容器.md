# 22-STL 之 容器

## 22.0 目录

1. STL的六大部件介绍
2. 容器分类
3. 序列式容器介绍（vector、list、deque）
4. 关联式容器
5. 资料
6. 收获

## 22.1 STL六大部件

STL：cpp standard library cpp标准库

STL的六大部件compounts：

1. 容器 （Containers）
2. 分配器（Allocators）
3. 算法（Algorithms）
4. 迭代器（Iteratros）
5. 适配器（Adapters）
6. 仿函数（Functors）

<img src="https:////upload-images.jianshu.io/upload_images/1791669-28e7c4a9836e0d2e.png?imageMogr2/auto-orient/strip|imageView2/2/w/1006/format/webp" alt="img" style="zoom:80%;" />

<img src="https:////upload-images.jianshu.io/upload_images/1791669-0b329362a6bd16d7.png?imageMogr2/auto-orient/strip|imageView2/2/w/906/format/webp" alt="img" style="zoom: 80%;" />

其中比较重要或者常用的部件是：容器、算法、迭代器、仿函数
 迭代器是连接容器和算法的桥梁，仿函数是对“（）”的操作符重载。接下来我们会重点学习实践容器和算法。而本篇主要介绍各种容器的特性。

## 22.2 容器分类

容器从内存分配是否是连续上来划分，可以分为序列式容器和关联式容器以及无序容器。

**序列式容器**：强调值的排序，每个元素都有固定的位置

包含以下几种容器类型：

1. Array（数组 固定大小）
2. Vector（向量 会自动扩充）、
3. Deque（双向队列，前后都可以扩充）
4. List：链表（双向链表）
5. Forward-list：（单项链表）

**关联式容器**：二叉树结构，各元素之间没有严格的物理上的顺序关系。
 使用场景：大量查找

包含以下几种容器：

1. Set/Multiset
2. Map/Multimap ：key-Value

其中 Set 和Map 元素内容不可以重复，Multiset和Multimap 元素内容可重复

内部是高度平衡的树即红黑树结构



![img](https:////upload-images.jianshu.io/upload_images/1791669-cc6ed6a2df3df7d0.png?imageMogr2/auto-orient/strip|imageView2/2/w/858/format/webp)

**无序容器**
 HashTable Separate Chaining

![img](https:////upload-images.jianshu.io/upload_images/1791669-838833e448f51d14.png?imageMogr2/auto-orient/strip|imageView2/2/w/748/format/webp)

## 22.3 序列式容器介绍（vector、list、deque）

### 22.3.1 vector

<img src="https:////upload-images.jianshu.io/upload_images/1791669-d748fc4e958ac9fc.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp" alt="img" style="zoom: 50%;" />

Vector和普通数组的区别在于，数组是静态空间，vector是动态扩展。这里的动态扩展并不是在原空间之后续接新空间，而是寻找更大的内存空间，以两倍增长的方式 ，将原数据拷贝过去。

获取数据的方式

```csharp
[]
at
front
back
```

容器互换:  swap()
 查看容器的容量:   capacity()
 重新制定vector容器大小:  resize()

### 22.3.2 list 链表

链表是由一个个节点组成，节点包含 数据域和指针域。

优点：对任意位置可以快速添加删除数据，只需要改变节点指针域的指向，而不用移动后续位置；大小比较灵活，不用想vector的双倍扩展
 缺点：遍历速度没有连续的array块；由于指针域的存在，占用的空间比array大。

Stl中的链表是一个双向循环链表。链表的迭代器只能前移或后移，不能跳跃

![img](https:////upload-images.jianshu.io/upload_images/1791669-53dfc271e87e7cc8.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)



```swift
前加加： operator++（） 返回值是self&
后加加： operator++（int） 返回值是self
```

### 22.3.3 deque容器

双端数组，可以对头端进香插入删除操作，deque内存上是分段连续的



<img src="https:////upload-images.jianshu.io/upload_images/1791669-b60645b4922c7550.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp" alt="img" style="zoom: 80%;" />

顺序容器适配器栈stack 内部是deque实现
 先进后出



![img](https:////upload-images.jianshu.io/upload_images/1791669-5cd0290d37e7f5e3.png?imageMogr2/auto-orient/strip|imageView2/2/w/508/format/webp)

顺序容器适配器 队列queue  先进先出，内部也是deque实现

![img](https:////upload-images.jianshu.io/upload_images/1791669-0d54016451a0b0d8.png?imageMogr2/auto-orient/strip|imageView2/2/w/506/format/webp)

**常用的方法如下：**

赋值操作



```objectivec
=
assign(begin,end)
assign(n,num)
```

大小



```cpp
deque.empty() 
deque.size()
deque.resize(num)
deque.resize(num,elem)

没有容量capacity
```



```cpp
void printDeuqe(const deque<int &d>)
{
    for(deque<int>::const_iterator it =d.begin();it !=d.end();it++)
    {
        cout << *it << "";
    }
    cout <<endl;
}
```

插入删除



```cpp
//在容器尾部添加一个数据
push_back(elem);

//在容器头部添加一个容器
push_front(elem);

//删除容器的最后一个数据
pop_back();

//删除容器的第一个数据
pop_front();

//在pos位置插入一个elem元素的拷贝，返回新数据的位置
insert(pos,elem);
//在pos位置插入【beg,end）区间的数据
insert(pos,beg,end);

clear();
erase(pos);
erase(beg,end);
```

数据存取



```cpp
//返回索引pos所指的数据
at(int pos);
opeartor[];
//返回容器中第一个数据元素
front();
//返回容器最后一个数据元素
back();
```

排序



```swift
//对beg和end区间内元素进行排序
sort(iterator beg, iterator end)
```

## 22.4 关联性容器

可以理解为 小型的关联型数据库

### 22.4.1  set/multiset容器

容器set中所有的元素插入时候会自动排序，其中set不允许重复值；multiset可以重复。

赋值
 插入数据使用insert

赋值构造
 拷贝构造

查找和统计



```swift
//查找key是否存在，如存在，返回该键的元素的迭代器；如不存在返回set.end()

find(key);

//统计key的元素个数,对于set返回值是0或1. 
count(key);
```

排序



```cpp
利用仿函数，改变排序规则

仿函数 重载（），记得加const
如： bool opeartor（）（int x, int y） const;

class Mycompare
{
public:
    bool operator()(int v1,int v2)
    {
        return v1 >v2;
    }
}

对于自定义的数据类型排序，要使用仿函数制定排序规则。
set<Object,MyCompare>
```

### 22.4.2 map/multimap容器

map中所有元素都是pair，其中Pair的第一元素是key，起到索引作用，第二个元素是value，所有元素都会根据元素的key自动排序。

另外容器的迭代器采用左闭右开的区间编程



![img](https:////upload-images.jianshu.io/upload_images/1791669-cf14d53d72ea07fe.png?imageMogr2/auto-orient/strip|imageView2/2/w/536/format/webp)

这篇就到这里，了解stl的六大组件，容器的分类，以及各种容器的特点，常用方法以及部分使用场景。从下一篇开始开始学习实践算法，一起来学习吧。

## 22.5 资料

1. 《Tinking in C++》
2. 《C++Primer》
3. [侯捷C++ STL 体系结构与内核分析] : [https://www.bilibili.com/video/BV1db411q7B8?from=search&seid=653949808386505225](https://links.jianshu.com/go?to=https%3A%2F%2Fwww.bilibili.com%2Fvideo%2FBV1db411q7B8%3Ffrom%3Dsearch%26seid%3D653949808386505225)  上面的图片均来自于该资源。

## 22.6 收获

1. 了解stl的六大组件
2. 了解序列式容器和关联式容器的分类
3. 了解各种容器的特点，常用方法，和优缺点。

 