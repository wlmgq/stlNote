迭代器模式：提供一种方法，使之能够依序访问某个容器中的各个元素，而又无需暴露该容器的内部表述方式。
迭代器是容器和算法之间的胶合剂。



1.如何判断迭代器的类型
    1)通过编译器对函数模版的实参推倒可以解决参数类别的判定,如下


template<class I, class T>
	  void func_imp1(I Iter, T t)
	  {
	    T tmp=t;//由于func_imp1是一个函数模版，一旦被调用，编译器会自动进行参数推倒，导出参数类型为T（改例为int）
		//...这里为func应该做的工作
	  }
    
	  template<class I>
	  inline void func(I Iter)
	  {
	    func_imp1(Iter,*Iter);
	  }
	  
	  int main()
	  {
	    int i;
		func(&i);
	  }
	2)通过内嵌类型声明解决返回值类型的判定，如下

template<class T>
	  struct MyIter//迭代器类
	  {
	    typedef T value_type; //内嵌类型声明，（类类型可以自己声明，但是内置类型没有办法声明）
	    T* ptr；
		MyIter（T* p=0）:ptr(p){}
		T& operator*(){return *ptr;}
		//.....
	  };
	  
	  template<class I>
	  typename I::value_type func(I Iter)//算法类    !！typename I::value_type为返回值的类型声明
	  {
	    return *Iter;
	  }
   3)通过Traits解决原生指针的返回类型问题
       *不同的容器有不同的专属迭代器，而不同的迭代器具有不同的特性；但是算法的接口确实统一的，
        算法通过Traits技术获取不同的迭代器属性，然后选择正确的流程；
        Traits依靠显式模版特殊化把代码中因类型不同而发生变化的片段提取出来，用统一的接口来包装；
        Traits允许系统在编译的时候根据类型做一些判断，遵循另增一个间接层的宗旨。
       例如：

//泛化版本
        template<class I>//I为迭代器类，迭代器类要内嵌类型声明
        struct iterator_traits//相当于一个中间层
        {
		  typedef typename I::value_type value_type;
		};		
		
	   //特化版本1
	    template<class I>
		struct iterator_traits<I*>//迭代器是个原生指针
		{
		  typedef I value_type;
		}
	   
	   //特化版本2
	    template<class I>
		struct iterator_traits< const I*>//迭代器是个原生指针
		{
		  typedef I value_type;
		}
		
		//2）的修改版本
		template<class I>
	    typename iterator_traits<I>::value_type func(I Iter)//通过iterator_traits类实现接口统一
	    {
	      return *Iter;
	    }
2.迭代器内嵌相应类型
    1）value_type:表示迭代器所指对象的类型
    2）difference_type:表示两个迭代器之间的距离
    3）reference:传递迭代器所指对象的引用
    4）pointer:传递迭代器所指对象的地址
    5）iterator_category(迭代器种类):

            a.Input Iterator:只读，支持p++
            b.Output Iterator:只写，支持p++
            c.Forward Iterator:允许写入型算法在此种迭代器所形成的区间上进行读写操作，支持p++
            d.Bidirectionl Iterator:可双向移动，支持p++，p--
            e.Random Access Iterator:支持p++,p--,p[n],p+n,p-n,p1-p2,p1<p2
           *任何一个迭代器，其类型永远应该落在“该迭代器所隶属的各种类型中，最强化的那个”；
           *STL算法的一个命名规则：以算法所能接收的最低阶迭代器类型，来为其迭代器类型参数命名；template<class InputIterator>
           *迭代器类型之间存在继承关系 InputIterator<-ForwardIterator<-Bidirectionl Iterator<-Random Access Iterator
    注意：任何迭代器都应该提供五个内嵌相应类型，以利于Traits萃取类型。继承STL提供的iterator类可以满足这个要求。
3.STL迭代器关键声明代码

1)//五种迭代器类型(只是用来区分类型，方便实现函数重载)
       struct input_iterator_tag{};	
	   struct output_iterator_tag{};
	   struct forward_iterator_tag : public input_iterator_tag {};
	   struct bidirectionl_iterator_tag : public forward_iterator_tag {};
	   struct random_access_iterator_tag : public bidirectionl_iterator_tag {};
2)//iterator类，供客户自定义的迭代器类继承
	   template<class Category, class T, class Distance = ptrdiff_t, class Pointer = T*, class Reference = T&>
	   struct iterator
	   {
	      typedef Category  iterator_category;
		  typedef T         value_type;
		  typedef Distance  difference_type;
		  typedef Pointer   pointer;
		  typedef Reference reference;
	   }
3)//iterator_traits类，用来判定类型
	   //泛化版本
	   template<class iterator>
	   struct iterator_traits
	   {
	      typedef typename iterator::iterator_category  iterator_category;
		  typedef typename iterator::value_type         value_type;
		  typedef typename iterator::difference_type    difference_type;
		  typedef typename iterator::pointer            pointer;
		  typedef typename iterator::reference          reference;
	   }
	   //特化版本，针对原生指针T*
	   template<class iterator>
	   struct iterator_traits<T*>
	   {
	      typedef random_access_iterator_tag  iterator_category;
		  typedef T                           value_type;
		  typedef ptrdiff_t                   difference_type;
		  typedef T*                          pointer;
		  typedef T&                          reference;
	   }

4.++、--
    iterator<>& operator ++();//表示前置，++it;



在开始讲迭代器之前，先列举几个例子，由浅入深的来理解一下为什么要设计迭代器。

//对于int类的求和函数
int sum(int *a , int n)
{
    int sum = 0 ;
    for (int i = 0 ; i < n ; i++) {
        sum += *a++;
    }
    return sum;
}
//对于listNode类的求和函数
struct ListNode {
    int val;
    ListNode * next;
};
int sum(ListNode * head) {
    int sum = 0;
    ListNode *p = head;
    while (p != NULL) {
        sum += p->val;
        p=p->next;
    }
    return sum;
}
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
针对如上两个函数，我们来谈谈这种设计的缺陷所在：

遍历int数组和单链表listNode时，需要设计两份不一样的sum求和函数，对于STL这样含有大量容器的代码库，针对每一种容器都设计sum的话，过于冗杂
在sum函数中暴露了太多设计细节，如ListNode的节点值类型int，和指向下一个节点的指针next
对于int数组来说，还必须知道数组的大小，以免越界访问
算法的设计过多的依赖容器，容器的改动会造成大量算法函数的随之改动
那么，如何设计才能使得算法摆脱特定容器？如何让算法和容器各自独立设计，互不影响又能统一到一起？本篇博客就带你一窥STL的迭代器设计。

迭代器概述
迭代器是一种抽象的设计概念，在设计模式中是这么定义迭代器模式的，即提供一种方法，使之能够巡访某个聚合物所含的每一个元素，而无需暴露该聚合物的内部表述方式。

不论是泛型思维或STL的实际运用，迭代器都扮演着重要的角色，STL的中心思想就在于：将数据容器和算法分开，彼此独立设计，最后再以一帖胶着剂将它们撮合在一起。

谈到迭代器需要遍历容器就想到指针，的确，迭代器就是一种类似指针的对象，而指针的各种行为中最常见也最重要的就是内容提领(dereference)和成员访问(member access)，因此，迭代器最重要的编程工作就是对operator*和operator->进行重载工作。

Traits编程技法
在介绍STL迭代器之前，先来看看Traits编程技法，通过它你能够更好的理解迭代器设计。

template参数推导机制
我们先回过头去看看sum函数，在sum函数中，我们必须知道容器的元素类型，这关系到函数返回值。既然迭代器设计中不能暴露容器的实现细节，那么我们在算法中是不可能知道容器元素的类型，因此，必须设计出一个机制，能够提取出容器中元素的类型。看看如下示例代码：

#include <stdio.h>
#include <iostream>

using namespace std;

//此函数中并不知道iter所指的元素型别，而是通过模板T来获取的
template <class I, class T1 ,class T>
T sum_impl(I iter ,T1 n , T t) {
    T sum = 0;//通过模板的特性，可以获取I所指之物的型别,此处为int

    //这里做func应该做的工作
    for(int i = 0 ; i < n ;i++){
        sum+=*iter++;
    } 
    return sum;
}

template <class I , class T1>
inline T1 sum(I iter , T1 n) {//此处暴露了template参数推导机制的缺陷，在型别用于返回值时便束手无策
    return sum_impl(iter , n ,*iter);
}

int main() {
    int a[5] = {1,2,3,4,5};
    int sum1 = sum(a , 5);
    cout<<sum1<<endl;
}
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
上例中通过模板的参数推导机制推导出了iter所指之物的型别(int)，于是可以定义T sum变量。

然而，迭代器所指之物的型别并非只是”迭代器所指对象的型别“，最常用的迭代器型别有五种，并非任何情况下任何一种都可利用上述的template参数推导机制来获取，而且对于返回值类型，template参数推导机制也束手无策，因此，Traits技法应运而生，解决了这一难题！

内嵌型别机制
Traits技法采用内嵌型别来实现获取迭代器型别这一功能需求，具体怎么实现的呢？我们看下面的代码：

#include <stdio.h>
#include <iostream>

using namespace std;
//定义一个简单的iterator
template <class T>
struct MyIter{
    typedef T value_type; // 内嵌型别声明
    T* ptr;
    MyIter(T* p =0):ptr(p){}
    T& operator*() const {return *ptr;}
};

template <class I>
typename I::value_type // func返回值型别
func(I iter){
    return *iter;
}

int main(){
    MyIter<int> iter(new int(8));
    cout<<func(iter)<<endl;
}
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
注意，func()函数的返回值型别必须加上关键词typename，因为T是一个template参数，在它被编译器具现化之前，编译器对其一无所知，关键词typename的用意在于告诉编译器这是一个型别，如此才能通过编译。

内嵌型别看起来不错，可以很顺利的提取出迭代器所指型别并克服了template参数推导机制不能用于函数返回值的缺陷。可是，并不是所有的迭代器都是class type，原声指针就不是！如果不是class type，那么就无法声明内嵌型别。但是STL绝对必须接受原生指针作为一个迭代器。因此，必须另行它法！

iterator_traits萃取机
针对原生指针这类特殊情况，我们很容易想到利用模板偏特化的机制来实现特殊声明，在泛化设计中提供一个特化版本。偏特化的定义是：针对任何template参数更进一步的条件限制所设计出来的一个特化版本。这里，针对上面的MyIter设计出一个适合原生指针的特化版本，如下：

template <class T>
struct MyIter <T*>{   //T*代表T为原生指针，这便是T为任意型别的一个更进一步的条件限制
    typedef T value_type; // 内嵌型别声明
    T* ptr;
    MyIter(T* p =0):ptr(p){}
    T& operator*() const {return *ptr;}
};
1
2
3
4
5
6
7
有了上述的介绍，包括template参数推导，内嵌型别，模板偏特化等，下面STL的真正主角要登场了，STL专门设计了一个iterator_traits模板类来”萃取“迭代器的特性。其定义如下：

// 用于traits出迭代其所指对象的型别
template <class I>
struct iterator_traits
{
  typedef typename I::value_type        value_type;
};
1
2
3
4
5
6
这个类如何完成迭代器的型别萃取呢？我们继续往下看：

template <class I>
typename iterator_traits<I>::value_type // 通过iterator_traits类萃取I的型别
func(I iter){
    return *iter;
}
1
2
3
4
5
从表面上来看，这么做只是多了一层间接性，但是带来的好处是极大的！iterator_traits类可以拥有特化版本，如下：

//原生指针特化版本
template <class T>
struct iterator_traits <T*>
{
  typedef T  value_type;
}
//const指针特化版本
template <class T>
struct iterator_traits <const T*>
{
  typedef T  value_type;
}
1
2
3
4
5
6
7
8
9
10
11
12
于是，原生指针int*虽然不是一种class type，也可以通过traits取其value，到这里traits的思想就基本明了了！下面就来看看STL只能中”萃取机“的源码

// 用于traits出迭代其所指对象的型别
template <class Iterator>
struct iterator_traits
{
  // 迭代器类型, STL提供五种迭代器
  typedef typename Iterator::iterator_category iterator_category;

  // 迭代器所指对象的型别
  // 如果想与STL算法兼容, 那么在类内需要提供value_type定义
  typedef typename Iterator::value_type        value_type;

  // 这个是用于处理两个迭代器间距离的类型
  typedef typename Iterator::difference_type   difference_type;

  // 直接指向对象的原生指针类型
  typedef typename Iterator::pointer           pointer;

  // 这个是对象的引用类型
  typedef typename Iterator::reference         reference;
};
// 针对指针提供特化版本
template <class T>
struct iterator_traits<T*>
{
  typedef random_access_iterator_tag iterator_category;
  typedef T                          value_type;
  typedef ptrdiff_t                  difference_type;
  typedef T*                         pointer;
  typedef T&                         reference;
};

// 针对指向常对象的指针提供特化
template <class T>
struct iterator_traits<const T*>
{
  typedef random_access_iterator_tag iterator_category;
  typedef T                          value_type;
  typedef ptrdiff_t                  difference_type;
  typedef const T*                   pointer;
  typedef const T&                   reference;
};
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
对于源码中的五种迭代器型别在下一小节中会有详细说明。

迭代器设计
迭代器型别
value_type
所谓value_type，是指迭代器所指对象的型别，在之前的示例中已经介绍得很清楚了，这里就不再赘述。

difference_type
difference_type用来表示两个迭代器之间的距离，因此它也可以用来表示一个容器的最大容量。下面以STL里面的计数功能函数count()为例，来介绍一下difference_type的用法。

template <class I,class T>
typename iterator_traits<I>::difference_type   //返回值类型
count(I first , I end , const T& value){
    typename iterator_traits<I>::difference_type n = 0;  
    for( ; first!=end ; ++first){
        if(*first == value) ++n;
    }
    return n;
}
1
2
3
4
5
6
7
8
9
针对原生指针和原生的const指针，iterator_traits的difference_type为ptrdiff_t(long int)，特化版本依然可以采用iterator_traits::difference_type来获取型别。

reference_type
从“迭代器所指之物的内容是否可以改变”的角度可以将迭代器分为两种：

const迭代器：不允许改变“所指对象之内容”，例如const int* p
mutable迭代器：允许改变“所指对象之内容”，例如int* p
当我们要对一个mutable迭代器进行提领(reference)操作时，获得的不应该是一个右值(右值不允许赋值操作)，而应该是一个左值，左值才允许赋值操作。

故： 
+ 当p是一个mutable迭代器时，如果其value type是T，那么*p的型别应该为T&； 
+ 当p是一个const迭代器时，*p的型别为const T&

pointer type
迭代器可以传回一个指针，指向迭代器所指之物。再迭代器源码中，可以找到如下关于指针和引用的实现：

Reference operator*() const { return *value; }
pointer operator->() const { return &(operator*()); }
1
2
在iterator_traits结构体中需要加入其型别，如下：

template <class Iterator>
struct iterator_traits
{
  typedef typename Iterator::pointer           pointer;
  typedef typename Iterator::reference         reference;
}
//针对原生指针的特化版本
template <class T>
struct iterator_traits<T*>
{
  typedef T*                         pointer;
  typedef T&                         reference;
};
//针对原生const指针的特化版本
template <class T>
struct iterator_traits<const T*>
{
  typedef const T*                   pointer;
  typedef const T&                   reference;
};
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
iterator_category
这一型别代表迭代器的类别，一般分为五类：

Input Iterator：只读(read only)
Output Iterator：只写(write only)
Forward Iterator：允许“写入型”算法在此迭代器所形成的区间上进行读写操作
Bidirectional Iterator：可双向移动的迭代器
Random Access Iterator：前四种迭代器都只供应一部分指针的算数能力(前三种支持operator++，第四种支持operator–)，第五种则涵盖所有指针的算数能力，包括p+n,p-n,p[n],p1-p2,p1
// 用于标记迭代器类型
struct input_iterator_tag {};
struct output_iterator_tag {};
struct forward_iterator_tag : public input_iterator_tag {};
struct bidirectional_iterator_tag : public forward_iterator_tag {};
struct random_access_iterator_tag : public bidirectional_iterator_tag {};
1
2
3
4
5
6
迭代器的保证
为了符合规范，任何迭代器都应该提供五个内嵌型别，以利于Traits萃取，否则就自别与整个STL架构，可能无法与其他STL组件顺利搭配。STL提供了一份iterator class如下：

//为避免挂一漏万，每个新设计的迭代器都必须继承自它
template <class Category, class T, class Distance = ptrdiff_t,
          class Pointer = T*, class Reference = T&>
struct iterator {
  typedef Category  iterator_category;
  typedef T         value_type;
  typedef Distance  difference_type;
  typedef Pointer   pointer;
  typedef Reference reference;
};
1
2
3
4
5
6
7
8
9
10
迭代器源码(完整版)
/**
 * 用于标记迭代器类型
    */
    struct input_iterator_tag {};
    struct output_iterator_tag {};
    struct forward_iterator_tag : public input_iterator_tag {};
    struct bidirectional_iterator_tag : public forward_iterator_tag {};
    struct random_access_iterator_tag : public bidirectional_iterator_tag {};

/**
 * 用于traits出迭代其所指对象的型别
    */
    template <class Iterator>
    struct iterator_traits
    {
      // 迭代器类型, STL提供五种迭代器
      typedef typename Iterator::iterator_category iterator_category;

  // 迭代器所指对象的型别
  // 如果想与STL算法兼容, 那么在类内需要提供value_type定义
  typedef typename Iterator::value_type        value_type;

  // 这个是用于处理两个迭代器间距离的类型
  typedef typename Iterator::difference_type   difference_type;

  // 直接指向对象的原生指针类型
  typedef typename Iterator::pointer           pointer;

  // 这个是对象的引用类型
  typedef typename Iterator::reference         reference;
};

/**
 * 针对指针提供特化版本
    */
    template <class T>
    struct iterator_traits<T*>
    {
      typedef random_access_iterator_tag iterator_category;
      typedef T                          value_type;
      typedef ptrdiff_t                  difference_type;
      typedef T*                         pointer;
      typedef T&                         reference;
    };
    /**
 * 针对指向常对象的指针提供特化
    */
    template <class T>
    struct iterator_traits<const T*>
    {
      typedef random_access_iterator_tag iterator_category;
      typedef T                          value_type;
      typedef ptrdiff_t                  difference_type;
      typedef const T*                   pointer;
      typedef const T&                   reference;
    };
    /**
 * 返回迭代器类别
    */
    template <class Iterator>
    inline typename iterator_traits<Iterator>::iterator_category
    iterator_category(const Iterator&)
    {
      typedef typename iterator_traits<Iterator>::iterator_category category;
      return category();
    }
    /**
 * 返回表示迭代器距离的类型
    */
    template <class Iterator>
    inline typename iterator_traits<Iterator>::difference_type*
    distance_type(const Iterator&)
    {
      return static_cast<typename iterator_traits<Iterator>::difference_type*>(0);
    }
    /**
 * 返回迭代器所指对象的类型
    */
    template <class Iterator>
    inline typename iterator_traits<Iterator>::value_type*
    value_type(const Iterator&)
    {
      return static_cast<typename iterator_traits<Iterator>::value_type*>(0);
    }
    1
    2
    3
    4
    5
    6
    7
    8
    9
    10
    11
    12
    13
    14
    15
    16
    17
    18
    19
    20
    21
    22
    23
    24
    25
    26
    27
    28
    29
    30
    31
    32
    33
    34
    35
    36
    37
    38
    39
    40
    41
    42
    43
    44
    45
    46
    47
    48
    49
    50
    51
    52
    53
    54
    55
    56
    57
    58
    59
    60
    61
    62
    63
    64
    65
    66
    67
    68
    69
    70
    71
    72
    73
    74
    75
    76
    77
    78
    79
    80
    81
    82
    83
    84
    迭代器相关函数设计
    distance函数
    distance函数用来计算两个迭代器之前的距离。

针对Input Iterator，Output Iterator，Forward Iterator，Bidirectional Iterator来说，必须逐一累加计算距离
针对random_access_iterator来说，支持两个迭代器相减，所以直接相减就能得到距离
其具体实现如下:

/**
 * 针对Input Iterator，Output Iterator，Forward Iterator，Bidirectional Iterator
 * 这四种函数，需要逐一累加来计算距离
    */
    template <class InputIterator>
    inline iterator_traits<InputIterator>::difference_type
    __distance(InputIterator first, InputIterator last, input_iterator_tag)
    {
      iterator_traits<InputIterator>::difference_type n = 0;
      while (first != last) {
    ++first; ++n;
      }
      return n;
    }
    /**
 * 针对random_access_iterator，可直接相减来计算差距
    */
    template <class RandomAccessIterator>
    inline iterator_traits<RandomAccessIterator>::difference_type
    __distance(RandomAccessIterator first, RandomAccessIterator last,
           random_access_iterator_tag)
    {
      return last - first;
    }
    // 入口函数，先判断迭代器类型iterator_category，然后调用特定函数
    template <class InputIterator>
    inline iterator_traits<InputIterator>::difference_type
    distance(InputIterator first, InputIterator last)
    {
      typedef typename iterator_traits<InputIterator>::iterator_category category;
      return __distance(first, last, category());
    }
    1
    2
    3
    4
    5
    6
    7
    8
    9
    10
    11
    12
    13
    14
    15
    16
    17
    18
    19
    20
    21
    22
    23
    24
    25
    26
    27
    28
    29
    30
    31
    32
    Advance函数
    Advance函数有两个参数，迭代器p和n，函数内部将p前进n次。针对不同类型的迭代器有如下实现：

针对input_iterator和forward_iterator,单向，逐一前进
针对bidirectional_iterator，双向，可以前进和后退，n>0和n<0
针对random_access_iterator，支持p+n，可直接计算
其代码实现如下：

/**
 * 针对input_iterator和forward_iterator版本
    */
    template <class InputIterator, class Distance>
    inline void __advance(InputIterator& i, Distance n, input_iterator_tag)
    {
      while (n--) ++i;
    }
    /**
 * 针对bidirectional_iterator版本
    */
    template <class BidirectionalIterator, class Distance>
    inline void __advance(BidirectionalIterator& i, Distance n,
                      bidirectional_iterator_tag)
    {
      if (n >= 0)
    while (n--) ++i;
      else
    while (n++) --i;
    }
    /**
 * 针对random_access_iterator版本
    */
    template <class RandomAccessIterator, class Distance>
    inline void __advance(RandomAccessIterator& i, Distance n,
                      random_access_iterator_tag)
    {
      i += n;
    }
    /**
 * 入口函数，先判断迭代器的类型
    */
    template <class InputIterator, class Distance>
    inline void advance(InputIterator& i, Distance n)
    {
      __advance(i, n, iterator_category(i));
    }
    1
    2
    3
    4
    5
    6
    7
    8
    9
    10
    11
    12
    13
    14
    15
    16
    17
    18
    19
    20
    21
    22
    23
    24
    25
    26
    27
    28
    29
    30
    31
    32
    33
    34
    35
    36
    37
    附加：type_traits设计
    iterator_traits负责萃取迭代器的特性；type_traits负责萃取型别的特性。

在带你深入理解STL之空间配置器(思维导图+源码)一篇博客中，讲到需要根据对象构造函数和析构函数的trivial和non-trivial特性来采用最有效的措施，例如：如果构造函数是trivial的，那么可以直接采用如malloc()和memcpy()等函数，来提高效率。

例如：如果拷贝一个未知类型的数组，如果其具有trivial拷贝构造函数，那么可以直接利用memcpy()来拷贝，反之则要调用该类型的拷贝构造函数。

type_traits的源代码实现如下：

/**
 * 用来标识真/假对象,利用type_traits响应结果来进行参数推导，
 * 而编译器只有面对class object形式的参数才会做参数推导，
 * 这两个空白class不会带来额外负担
    */
    struct __true_type{};
    struct __false_type{};

/**
 * type_traits结构体设计
   */
   template <class type>
   struct __type_traits
   {
   // 不要移除这个成员
   // 它通知能自动特化__type_traits的编译器, 现在这个__type_traits template是特化的
   // 这是为了确保万一编译器使用了__type_traits而与此处无任何关联的模板时
   // 一切也能顺利运作
   typedef __true_type     this_dummy_member_must_be_first;

   // 以下条款应当被遵守, 因为编译器有可能自动生成类型的特化版本
   //   - 你可以重新安排的成员次序
   //   - 你可以移除你想移除的成员
   //   - 一定不可以修改下列成员名称, 却没有修改编译器中的相应名称
   //   - 新加入的成员被当作一般成员, 除非编译器提供特殊支持

   typedef __false_type    has_trivial_default_constructor;
   typedef __false_type    has_trivial_copy_constructor;
   typedef __false_type    has_trivial_assignment_operator;
   typedef __false_type    has_trivial_destructor;
   typedef __false_type    is_POD_type;
   };
   // 特化类型:
   //         char, signed char, unsigned char,
   //         short, unsigned short
   //         int, unsigned int
   //         long, unsigned long
   //         float, double, long double
   /**
 * 以下针对C++内置的基本数据类型提供特化版本, 
 * 使其具有trivial default constructor,
 * copy constructor, assignment operator, destructor并标记其为POD类型
   */
   __STL_TEMPLATE_NULL struct __type_traits<char>
   {
   typedef __true_type    has_trivial_default_constructor;
   typedef __true_type    has_trivial_copy_constructor;
   typedef __true_type    has_trivial_assignment_operator;
   typedef __true_type    has_trivial_destructor;
   typedef __true_type    is_POD_type;
   };
   //针对char的特化版本
   __STL_TEMPLATE_NULL struct __type_traits<signed char>
   {
   typedef __true_type    has_trivial_default_constructor;
   typedef __true_type    has_trivial_copy_constructor;
   typedef __true_type    has_trivial_assignment_operator;
   typedef __true_type    has_trivial_destructor;
   typedef __true_type    is_POD_type;
   };
   //针对unsigned char的特化版本
   __STL_TEMPLATE_NULL struct __type_traits<unsigned char>
   {
   typedef __true_type    has_trivial_default_constructor;
   typedef __true_type    has_trivial_copy_constructor;
   typedef __true_type    has_trivial_assignment_operator;
   typedef __true_type    has_trivial_destructor;
   typedef __true_type    is_POD_type;
   };
   //针对short的特化版本
   __STL_TEMPLATE_NULL struct __type_traits<short>
   {
   typedef __true_type    has_trivial_default_constructor;
   typedef __true_type    has_trivial_copy_constructor;
   typedef __true_type    has_trivial_assignment_operator;
   typedef __true_type    has_trivial_destructor;
   typedef __true_type    is_POD_type;
   };
   //针对unsigned short的特化版本
   __STL_TEMPLATE_NULL struct __type_traits<unsigned short>
   {
   typedef __true_type    has_trivial_default_constructor;
   typedef __true_type    has_trivial_copy_constructor;
   typedef __true_type    has_trivial_assignment_operator;
   typedef __true_type    has_trivial_destructor;
   typedef __true_type    is_POD_type;
   };
   //针对int的特化版本
   __STL_TEMPLATE_NULL struct __type_traits<int>
   {
   typedef __true_type    has_trivial_default_constructor;
   typedef __true_type    has_trivial_copy_constructor;
   typedef __true_type    has_trivial_assignment_operator;
   typedef __true_type    has_trivial_destructor;
   typedef __true_type    is_POD_type;
   };
   //针对unsigned int的特化版本
   __STL_TEMPLATE_NULL struct __type_traits<unsigned int>
   {
   typedef __true_type    has_trivial_default_constructor;
   typedef __true_type    has_trivial_copy_constructor;
   typedef __true_type    has_trivial_assignment_operator;
   typedef __true_type    has_trivial_destructor;
   typedef __true_type    is_POD_type;
   };
   //针对long的特化版本
   __STL_TEMPLATE_NULL struct __type_traits<long>
   {
   typedef __true_type    has_trivial_default_constructor;
   typedef __true_type    has_trivial_copy_constructor;
   typedef __true_type    has_trivial_assignment_operator;
   typedef __true_type    has_trivial_destructor;
   typedef __true_type    is_POD_type;
   };
   //针对unsigned long的特化版本
   __STL_TEMPLATE_NULL struct __type_traits<unsigned long>
   {
   typedef __true_type    has_trivial_default_constructor;
   typedef __true_type    has_trivial_copy_constructor;
   typedef __true_type    has_trivial_assignment_operator;
   typedef __true_type    has_trivial_destructor;
   typedef __true_type    is_POD_type;
   };
   //针对float的特化版本
   __STL_TEMPLATE_NULL struct __type_traits<float>
   {
   typedef __true_type    has_trivial_default_constructor;
   typedef __true_type    has_trivial_copy_constructor;
   typedef __true_type    has_trivial_assignment_operator;
   typedef __true_type    has_trivial_destructor;
   typedef __true_type    is_POD_type;
   };
   //针对double的特化版本
   __STL_TEMPLATE_NULL struct __type_traits<double>
   {
   typedef __true_type    has_trivial_default_constructor;
   typedef __true_type    has_trivial_copy_constructor;
   typedef __true_type    has_trivial_assignment_operator;
   typedef __true_type    has_trivial_destructor;
   typedef __true_type    is_POD_type;
   };
   //针对long double的特化版本
   __STL_TEMPLATE_NULL struct __type_traits<long double>
   {
   typedef __true_type    has_trivial_default_constructor;
   typedef __true_type    has_trivial_copy_constructor;
   typedef __true_type    has_trivial_assignment_operator;
   typedef __true_type    has_trivial_destructor;
   typedef __true_type    is_POD_type;
   };

// 针对指针提供特化
template <class T>
struct __type_traits<T*>
{
   typedef __true_type    has_trivial_default_constructor;
   typedef __true_type    has_trivial_copy_constructor;
   typedef __true_type    has_trivial_assignment_operator;
   typedef __true_type    has_trivial_destructor;
   typedef __true_type    is_POD_type;
};

// 针对char *, signed char *, unsigned char *提供特化

struct __type_traits<char*>
{
   typedef __true_type    has_trivial_default_constructor;
   typedef __true_type    has_trivial_copy_constructor;
   typedef __true_type    has_trivial_assignment_operator;
   typedef __true_type    has_trivial_destructor;
   typedef __true_type    is_POD_type;
};

struct __type_traits<signed char*>
{
   typedef __true_type    has_trivial_default_constructor;
   typedef __true_type    has_trivial_copy_constructor;
   typedef __true_type    has_trivial_assignment_operator;
   typedef __true_type    has_trivial_destructor;
   typedef __true_type    is_POD_type;
};

struct __type_traits<unsigned char*>
{
   typedef __true_type    has_trivial_default_constructor;
   typedef __true_type    has_trivial_copy_constructor;
   typedef __true_type    has_trivial_assignment_operator;
   typedef __true_type    has_trivial_destructor;
   typedef __true_type    is_POD_type;
};
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52
53
54
55
56
57
58
59
60
61
62
63
64
65
66
67
68
69
70
71
72
73
74
75
76
77
78
79
80
81
82
83
84
85
86
87
88
89
90
91
92
93
94
95
96
97
98
99
100
101
102
103
104
105
106
107
108
109
110
111
112
113
114
115
116
117
118
119
120
121
122
123
124
125
126
127
128
129
130
131
132
133
134
135
136
137
138
139
140
141
142
143
144
145
146
147
148
149
150
151
152
153
154
155
156
157
158
159
160
161
162
163
164
165
166
167
168
169
170
171
172
173
174
175
176
177
178
179
180
181
182
183
184
185
186
187
188
189
190
后记
本篇博客比较枯燥，STL的迭代器中有很多优秀的值得学习的设计方式，如萃取机制，用类继承来定义迭代器类型等，通篇代码比较多，图片讲解较少，只能说迭代器的设计中基本上都是较为繁琐的型别声明和定义，认真看下去的话会收获很多！若有疑惑可以在博文下方留言，我看到会及时帮大家解答
--------------------- 
作者：ZeeCoder 
来源：CSDN 
原文：https://blog.csdn.net/terence1212/article/details/52287762 
版权声明：本文为博主原创文章，转载请附上博文链接！