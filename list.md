# 《STL源码剖析》学习笔记——List

[TOC]



### 概述

​	相较于Vector，List中元素删除和插入效率更高，其不需要拷贝元素；在存储空间的利用方面，List更加的精准，用多少申请多少，删除之后会将内存还给系统，不会像Vector产生空间浪费；另一方面，List不支持随机访问，需要使用迭代器访问。

## 数据结构

​	List的实现采用双向链表的数据结构，得益于此，List在元素的删除和插入操作上永远是常数时间。其节点的结构如下：

```c++
template <class T>
struct __list_node {
  typedef void* void_pointer;
  void_pointer next;//指向下一个节点
  void_pointer prev;//指向上一个节点
  T data;
};
```

节点中除了存放的数据外，还有两个指针，分别指向上一个节点和下一个节点。

## 数据管理

​	List的构造函数有多个版本，如下是无参形式的构造函数源码:

```c++
list() { empty_initialize(); }
void empty_initialize() { 
    node = get_node();
    node->next = node;
    node->prev = node;
}
```

List有一个节点`node`，节点内data为空，该节点作为List的入口，在构造函数中将`node->next`和`node->prev`都指向`node`，自身形成了一个闭环，此时List内没有其他节点，数据为空，其他重载的构造函数都是在调用empty_initialize()的基础上在插入数据。

​	作为List的入口，List的开始和结束节点位置是通过`node`获取的，如下所示，`node->next`指向了List的第一个节点，LIst最后一个元素的的`next`指向了`node`，如此形成了一个闭环。

```c++
iterator begin() { return (link_type)((*node).next); }
const_iterator begin() const { return (link_type)((*node).next); }
iterator end() { return node; }
const_iterator end() const { return node; }
```

获取List元素个数可以通过size接口，其实现是通过迭代器遍历List计数，empty接口可以用于List的判空操作，源码如下：

```c++
  bool empty() const { return node->next == node; } //只有头节点
  size_type size() const {
    size_type result = 0;
    distance(begin(), end(), result);
    return result;
  }
template <class InputIterator, class Distance>
inline void distance(InputIterator first, InputIterator last, Distance& n) {
  __distance(first, last, n, iterator_category(first));
}
template <class InputIterator, class Distance>
inline void __distance(InputIterator first, InputIterator last, Distance& n, 
                       input_iterator_tag) {
  while (first != last) { ++first; ++n; } //使用迭代器遍历List计数
}
```



### 迭代器

​	和其他容器一样，List内部实现了一个迭代器，其实现如下，其实际上维护了一个节点，重载了自加和自减操作，将下个或者上个节点赋值给当前节点。

```c++
template<class T, class Ref, class Ptr>
struct __list_iterator {
  typedef __list_iterator<T, T&, T*>             iterator;
  typedef __list_iterator<T, const T&, const T*> const_iterator;
  typedef __list_iterator<T, Ref, Ptr>           self;

  typedef bidirectional_iterator_tag iterator_category;
  typedef T value_type;
  typedef Ptr pointer;
  typedef Ref reference;
  typedef __list_node<T>* link_type;
  typedef size_t size_type;
  typedef ptrdiff_t difference_type;

  link_type node;//维护的节点

  __list_iterator(link_type x) : node(x) {}
  __list_iterator() {}
  __list_iterator(const iterator& x) : node(x.node) {}

  bool operator==(const self& x) const { return node == x.node; }
  bool operator!=(const self& x) const { return node != x.node; }
  reference operator*() const { return (*node).data; }

#ifndef __SGI_STL_NO_ARROW_OPERATOR
  pointer operator->() const { return &(operator*()); }
#endif /* __SGI_STL_NO_ARROW_OPERATOR */

  self& operator++() { 
    node = (link_type)((*node).next);//将下一个节点的值赋给迭代器
    return *this;
  }
  self operator++(int) { 
    self tmp = *this;
    ++*this;
    return tmp;
  }
  self& operator--() { 
    node = (link_type)((*node).prev); //将前一个节点的值赋给迭代器
    return *this;
  }
  self operator--(int) { 
    self tmp = *this;
    --*this;
    return tmp;
  }
}
```

### 插入

​	与其他容器类似，List支持insert接口进行插入操作，同样，inser他有多种形式，这里也仅讨论最简单的插入单个元素的形式，其源码如下，插入多个元素的实现也要调用单个元素版本的insert。

```c++
//在position位置插入元素x
iterator insert(iterator position, const T& x) {
    //申请空间，使用x的值构建一个节点，
    link_type tmp = create_node(x);
    //对新节点的前后指针赋值，使其位置在position之前，使其与其他容器的insert操作保持一致
    tmp->next = position.node;
    tmp->prev = position.node->prev;
    //配合新节点更新position及其前一个节点的值
    (link_type(position.node->prev))->next = tmp;
    position.node->prev = tmp;
    //返回新节点
    return tmp;
}
```

另外还有几个涉及插入的常用接口，它们都是借助了insert来实现：

```c++
  void push_front(const T& x) { insert(begin(), x); }//在第一个节点之前插入元素
  void push_back(const T& x) { insert(end(), x); }//在List最后新增节点
```

### 删除

​	与其他容器一致，List支持erase接口进行删除操作，erase重载了多个版本，这里仅讨论删除单个元素的版本，源码如下，删除多个元素的实现均依赖于删除单个元素。

```c++
iterator erase(iterator position) {
    link_type next_node = link_type(position.node->next);
    link_type prev_node = link_type(position.node->prev);
    //将position处前一个节点的下一节点指向position下一个节点
    prev_node->next = next_node;
    //将position处后一个节点的前一节点指向position上一个节点
    next_node->prev = prev_node;
    //销毁position处的节点
    destroy_node(position.node);
    //返回position处下一个节点的位置
    return iterator(next_node);
}
```

另外几个涉及删除节点的常用接口，他们都同故宫调用erase来实现：

```c++
void clear();
void pop_front() { erase(begin()); }
void pop_back() { 
    iterator tmp = end();
    erase(--tmp);
}
```

### 查找修改

​	与其他容器一样，可以调用STL算法结构Find来查找相应值的节点。通常调用begin等接口获取List的迭代器，通过遍历的方式查找和访问元素，修改元素的值也依赖于迭代器。

### transfer

​	transfer是list内部函数，作用是把List中的一段插入到某个位置，可以是两个List之间，也可以是同一个List，起源码如下，

```c++
void transfer(iterator position, iterator first, iterator last) {
    if (position != last) {
        (*(link_type((*last.node).prev))).next = position.node;
        (*(link_type((*first.node).prev))).next = last.node;
        (*(link_type((*position.node).prev))).next = first.node;  
        link_type tmp = link_type((*position.node).prev);
        (*position.node).prev = (*last.node).prev;
        (*last.node).prev = (*first.node).prev; 
        (*first.node).prev = tmp;
    }
}
```

​	可以将此过程理解为范围为[first, last)的List的删除后插入到另一段List的position处，具体可分为7步，分别对应与7行源码实现，如下图，摘自《STL源码剖析》

![](C:\Users\mo\Documents\GitHub\stlNote\无标题.png)

基于transfer即可实现两个List的插入拼接操作，即接口splice，起源吗如下，

```c++
void splice(iterator position, list& x) {
if (!x.empty()) 
transfer(position, x.begin(), x.end());
}
void splice(iterator position, list&, iterator i) {
iterator j = i;
++j;
if (position == i || position == j) return;
transfer(position, i, j);
}
void splice(iterator position, list&, iterator first, iterator last) {
if (first != last) 
transfer(position, first, last);
}
```

排序

​	由于LIst不支持随机访问，所以不适用STL算法中的sort，其实现是快速排序，所以List内部实现了sort接口，