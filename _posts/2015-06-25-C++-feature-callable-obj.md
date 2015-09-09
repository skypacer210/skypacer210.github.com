---
layout: post
title: "C++11可调用对象模板类初窥"  
image: /image/reactor_figure_1.png  
categories:
- C++

tags:
- 可调用对象
- 迭代器

---


###1、初识可调用对象模板类
 

{% highlight objc %}
 79 typedef boost::function<bool (const TcpConnectionPtr&,
 80 									StringPiece,
 81                         		Timestamp)> RawMessageCallback;
 82                  
 83 typedef boost::function<void (const TcpConnectionPtr&,
 84                         		const MessagePtr&,
 85                         		Timestamp)> ProtobufMessageCallback;
 86                  
 87 typedef boost::function<void (const TcpConnectionPtr&,
 88                         		Buffer*,
 89                         		Timestamp,
 90                         		ErrorCode)> ErrorCallback;
{% endhighlight %}

这里用到了C++11的新特性——**可调用对象模板类**，可调用对象理解为函数，该特性可以用于定义函数指针，相当于是把函数本身做进一步抽象。这里利用该特性定义了三个回调函数，以第一个`typedef boost::function<bool (const TcpConnectionPtr&, StringPiece，Timestamp)> RawMessageCallback`为例：

 - 其中`function<bool (const TcpConnectionPtr&, StringPiece，Timestamp)>`意思是定义了一个可调用对象（函数指针），参数是三个`const TcpConnectionPtr&, StringPiece，Timestamp`。
 - 再利用typedef，定义了一个函数指针`RawMessageCallback `，或者说是一个可调用对象。

该上下文的调用如下：     

{% highlight objc %}
134		private:
135   		const ::google::protobuf::Message* prototype_;
136       const string tag_;
137       ProtobufMessageCallback messageCallback_;
138       RawMessageCallback rawCb_;
139       ErrorCallback errorCallback_;
{% endhighlight %}

###2、用法实例

可以结合`map`容器和该特性定义一张函数表，如下示例：  

{% highlight objc %}
  1 #include <string>                                                                                                                                                                               
  2 #include <vector>
  3 #include <map>   
  4 #include <functional>
  5 #include <set>      
  6 #include <iostream> 
  7 #include <sstream>  
  8 #include <fstream>  
  9 #include <cstring>  
 10 #include <cctype>   
 11 #include <stdexcept>
 12                     
 13 #include <typeinfo> 
 14                     
 15                     
 16 using namespace std;
 17                     
 18 int add(int i, int j)
 19 {                   
 20     return i + j;   
 21 }                   
 22 auto mod = [ ](int i, int j)
 23 {                   
 24     return i % j;   
 25 };                  
 26                     
 27                     
 28 struct divide       
 29 {                   
 30     int operator() (int denom, int diveisor)
 31     {               
 32         return denom * diveisor;           
 33     }               
 34 };                  
 35                     
 36                     
 37 int main(int argc, char *argv[])           
 38 {                   
 39     map<char, int(*)(int, int)> binops_limit;
 40     binops_limit.insert( {'+', add} );     
 41     binops_limit.insert( {'%', mod} );     
 42                     
 43     map<char, function<int(int, int)> > binops = 
 44     {               
 45         { '+', add },      
 46         { '-', minus<int>() },             
 47         { '*', [](int i, int j) { return i - j; } },
 48         { '/', divide() }, 
 49         { '%', mod },      
 50     }; 
 51             
 52     map<char, int(*)(int, int)>::const_iterator iter =  
 53                                         binops_limit.begin();      
 54             
 55     for (; iter != binops_limit.end(); ++iter)          
 56     {       
 57         cout << "Type of iterator: " << typeid(iter).name() << endl;
 58         cout << "Type of iterator->first : " << typeid(iter->firstea.name() << " " << "value: " << iter->first  << endl;
 59         cout << "Type of iterator->second: " << typeid(iter->second).name() << " " << "value: " << iter->second  << endl;
 60     }       
 61             
 62     cout << "FuncTable: + "<< binops['+'](10, 5) <<endl;
 63     cout << "FuncTable: - "<< binops['-'](10, 5) << endl;
 64     cout << "FuncTable: * "<< binops['*'](10, 5) << endl;
 65     cout << "FuncTable: / "<< binops['/'](10, 5) << endl;
 66     cout << "FuncTable: % "<< binops['%'](10, 5) << endl;
 67             
 68     return 0;
 69 }
{% endhighlight %}

运行结果如下：

{% highlight objc %}
Type of iterator: NSt3__120__map_const_iteratorINS_21__tree_const_iteratorINS_12__value_typeIcPFiiiEEEPNS_11__tree_nodeIS5_PvEElEEEE
Type of iterator->first : c value: %
Type of iterator->second: PFiiiE value: 1
Type of iterator: NSt3__120__map_const_iteratorINS_21__tree_const_iteratorINS_12__value_typeIcPFiiiEEEPNS_11__tree_nodeIS5_PvEElEEEE
Type of iterator->first : c value: +
Type of iterator->second: PFiiiE value: 1
FuncTable: + 15
FuncTable: - 5
FuncTable: * 5
FuncTable: / 50
FuncTable: % 0
{% endhighlight %}
 
这里面有几个坑要注意： 

 - `auto mod = [ ](int i, int j)`定义了一个lamba表达式，即没有函数名的函数，这是什么鬼？注意函数实现最后有分号`；`
 - 头文件一定包含全面，否则出来一堆错误，特别是`#include <stdexcept>`；
 - 如果想打印类型名怎么办？就像gdb中的命令ptype，C++也有类似的方法，首先包含头文件`#include <typeinfo>`，然后直接`typeid(var)`即可。从运行结果可以看到迭代器本身的类型和迭代器成员的类型区别。
 - 迭代器本身为指针，因此访问其成员用`->`操作符。

TODO: 容器遍历与访问的两种方法：迭代器和容器内部方法之区别。