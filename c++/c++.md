# C++ 11中的某些“概念”

记录一些学习中遇到的概念，比较[基础的C++知识](./C_plus_exp.md)，尽量从汇编角度解释。

## 0x00 nullptr和NULL

C语言中一般将NULL宏定义为:
```C
#define NULL ((void*)0)
```
C++中将NULL宏定义为:
```C++
#define NULL 0
```

关于0和nullptr以及NULL的区别可以用这段程序测试出来

```C++
void f(int target){
    std::cout << "f(int target)" << std::endl;
}
void f(void* target){
    std::cout << "f(void* target)" << std::endl;
}
int main(){
    f(nullptr); // f(void* target)
    f(0); // f(int target)
    f(10000); // f(int target)
    f(NULL); // 报错，存在歧义
}
```

nullptr的底层是std::nullptr_t，而且NULL在C++中一般定义为0，不过上述程序中f(NULL)确确实实报存在歧义的错误，但是如果按照定义为0的情况是应该调用f(int)的函数的。

## 0x01 dynamic_cast和down cast

首先是多态，这个都清楚，比如:

```c++
class Base{
public:
 int x;
  virtual void show(){
    std::cout << "Base::show()" << std::endl;
  }
};
class Derived : public Base{
public:
  int y;
  virtual void show(){
    std::cout << "Deviced::show()" << std::endl;
  }
};

int main(){
  Base* p1 = new Base;
  Base* p2 = new Derived;
  p1->show(); // Base::show()
  p2->show(); // Deviced::show()
	delete p1;
  delete p2;
}
```

其通过内存中的虚表来进行调用，会调用到真正的属于自己类型的方法，但现在如果有一个需求，我们需要将`Base*`转为`Derived*`进行使用，对于以上例子，`p2`指针肯定可以通过，而`p1`就不行了，`p1`就是`Base*`类型，如果强制转为`Derived*`并且进行使用，会出现内存的错误。

这种将基类指针转为派生类的操作，我们称为`Downcast`。

具体在写程序的时候，我们如何判别这种downcast？有的donwcast是安全的，有的downcast则是错误的，这里可以使用C++中给出的`dynamic_cast`，动态转换，如果失败，则直接返回`nullptr`。

同样的代码，如果使用`dynamic_cast`，则:

```c++
int main(){
  Base* p1 = new Base;
  Base* p2 = new Derived;
  Derived* p3 = dynamic_cast<Derived*>(p1); // nullptr
  Derived* p4 = dynamic_cast<Derived*>(p2); // 成功
}
```

## 0x02 右值引用 or 将亡值


### 初识

一些代码就可以比较直观的看出左值和右值的区别，比如：

```C++
int a = 0;
int &a_lref = a; // 左值引用
int &&rref = 10; // 右值引用
```

概念的话，可以理解为： 

__能取地址的，能放到等式左边的，为左值__ 。

__不能取地址，不能放等式左边的，为右值__ 。

这里还有一条规则： __常引用可以绑定到右值__ ，如何理解？即：

```C++
const int& const_ref = 10; 
```

可以试着从汇编角度理解，有这么几行代码：

```C++
const int& r = 10;
int &&rr = 255;
int aaa = 255;
```

得到汇编：

```assembly
    push    rbp
    mov     rbp, rsp
    
    # const int& r = 10;
    mov     eax, 10
    mov     DWORD PTR [rbp-28], eax
    lea     rax, [rbp-28]
    mov     QWORD PTR [rbp-8], rax
    
    # int &&r = 255;
    mov     eax, 255
    mov     DWORD PTR [rbp-24], eax
    lea     rax, [rbp-24]
    mov     QWORD PTR [rbp-16], rax
    
    # int aaa = 255;
    mov     DWORD PTR [rbp-20], 255
    mov     eax, 0
    pop     rbp
    ret
```

可以看到，10，255都是右值，但是得到了不同的待遇， __编译器对常引用和右值引用都开辟了空间，然后再对其取地址得到引用(在汇编看来引用就是指针)，可以得出，当右值被常引用和右值引用"绑定"后，右值就真的存在于内存中了，换句话说，其生命周期被延长了，不再是“将亡值”__ 。

### 近一步

如果我们只使用POD类型的变量，右值的作用似乎没那么重要，但事实上我们经常会使用一些结构体、类等自定义类型，而变量免不了拷贝复制等等操作，这时候右值引用就起作用了。

考虑这样的代码：

```C++
class A {};

A getA() {
    A a;
    return a;
}

void handleA(A& a){}

int main() {
    A a = getA();
    handleA(a);         // pass
    handleA(getA());    // error 右值->将亡值
}
```

上述的例子中，12行通过的例子需要拷贝(尽管在现在的编译器的优化下，上述例子可直接构建在main栈帧上来避免拷贝，但这不是不使用右值的理由)，而 __13行不能通过编译的例子显然更符合我们“希望”的，因为这样看起来可以减少一次拷贝__ 。

getA()的返回值显然是右值，既然是右值，就不能取到地址。怎么做？很简单，就像一开始所说的那样，使用常引用或者右值引用，可以将handleA修改一下就可以通过，比如：

```C++
void handleA(const A& a){}

// 又比如我们讨论的主角，在C++11中加入的“右值引用”
void handleA(A&& a){} 
```

经过绑定的右值寿命有多长？可以试着编译后看汇编，我能给出的答案是， __将亡值将被当作函数调用的参数，直接构建在栈帧中，也就是handleA可通过rdi寄存器(x64)来访问这个“将亡值”，当handleA结束后，其对应的栈帧被销毁，那么这个“将亡值”的寿命也就到头了__ 。

__当某个右值被绑定到一个“右值引用”上后，这个右值引用就成了“左值”__ 试着这样理解：

```C++
int &&a = 10;
```

__10是右值，在被a这个右值引用绑定后，10存在于内存中了，但a这个“右值引用”本身也是存在内存中的，它不会即刻消亡，事实上，a是左值，或者你可以称“类型为右值引用的左值”__ 。

### 作用

它们的存在都是为了提高运行效率的，比较常用在的地方就是深拷贝和浅拷贝，某些即将被销毁的值(将亡值)中的一些指针，我们可以直接拿来使用，从而提高程序的运行效率。

常引用和右值引用都能延长将亡值寿命，但在C++类中存在一些区别，[具体](./TODO)

可以简单的写出一个Buffer类用于存储数据：

```C++
class Buffer {
public:
  Buffer()
  : data_(new char[1024])
  {
  }

  Buffer(const Buffer& buf) {
    data_ = new char[1024];
    memcpy(data_, buf.data_, 1024);
  }
  
  Buffer(Buffer&& buf) {
    data_ = buf.data_;
    buf.data_ = nullptr;
  }

  ~Buffer() {
    delete [] data_;
  }

  char* data_;
};

void handleBuffer(Buffer buf) {
  // do something
}

int main() {
    Buffer b;
    handleBuffer(b);            // call Buffer(const Buffer& buf)
    handleBuffer(std::move(b)); // call Buffer(Buffer&& buf)
}
```

可以看到，当我们调用handleBuffer时采用的是值传递方式，进而触发拷贝函数，生成2个副本，不过我们可以通过std::move将其转为“右值”，然后再通过移动构造函数(13行)将其“偷掉”，在32行之后，b变量就不能够再使用了，其内部的data_指针已为空。

// TODO: wait for update...


## 0x03 回调函数问题


### 多线程内存管理问题

在C++中对内存的管理一直是一个比较重要的问题，如果写的程序是单线程，那管理内存似乎会简单些许，但现实中我们总会写多线程程序，在多线程下对一块内存的管理就复杂了许多，特别是关乎类的构造及析构，情况会更加复杂。

竞态问题我们会马上想到某个东西，锁，是啊，锁可以解决竞态问题，比如这样一个类:

```C++
class Obj {
public:
    void read(){
        // lock
        std::unique_lock<std::mutex> lockGuard(mutex_);
        // do something...
    }

private:
    std::mutex mutex_;
};
```

看起来解决了，但忽略了构造和析构，构造好解决，或许我们可以用二段式构造？或许还有其他解决方法，但析构是个麻烦事，mutex_会在析构中被删除，而mutex_是应该保护函数安全运行的啊！似乎进入了死循环。

### 使用智能指针

我在看muduo网络库的时候就遇到了这个问题，代码中，TcpServer拥有TcpConnection，并且TcpConnection中拥有Channel，&Channel::handleEvent会在事件触发时被调用(epoll)，这本身没问题，问题发生在Tcp连接关闭事件，&Channel::handleEvent会企图调用关闭函数，关闭函数中需要删除TcpConnection，结果灾难发生了，TcpConnection拥有Channel，这意味着TcpConnection被删除后，Channel也不复存在，但我们现在正在使用Channel中的handleEvent函数啊！

原先，TcpConnection中拥有Channel的shared_ptr，引用为1，当TcpConnection被析构后，Channel也被析构，所以我们需要想办法延长Channel的寿命使得handleEvent顺利结束，怎么做？想办法让TcpConnection不要那么快的消亡，在handleEvent中获得一个指向TcpConnection的shared_ptr，使得引用最少为1，并且持续到&Channel::handleEvent结束为止，这听起来有点像左脚踩右脚上天，但这个方法确实可行，muduo就是这样解决的。

缩减部分代码得到如下例子：

```C++
/* Shared_test.h */
#include <functional>
#include <iostream>
#include <map>

using std::cout;
using std::endl;

class TcpConnection;
class Channel;

typedef std::function<void(int)> TcpServerCloseCallback;

class TcpServer {
public:

  // 将被当成回调注册进Channel，目的是删除TcpConnection
  void closeConnections(int fd);
  void newConnection(int fd);

private:
  // fd -> TcpConnection
  std::map<int, std::shared_ptr<TcpConnection>> connections_;

};

class TcpConnection : public std::enable_shared_from_this<TcpConnection>{
public:
  void setTcpServerCloseCallback(TcpServerCloseCallback);
  void init(int fd);

private:
  std::shared_ptr<Channel> channel_;
  TcpServerCloseCallback tcpServerCloseCallback_;
};

class Channel {
public:

  // 事件触发时调用，可能会调用关闭回调
  void handleEvent();
  void setCloseEventCallback(TcpServerCloseCallback cb);
  void setFd(int fd);
  // important
  void tie(const std::shared_ptr<void>& conn);

private:
  int fd_;
  TcpServerCloseCallback tcpServerCloseEventCallback_;
  
  // for tie
  std::weak_ptr<void> tie_;
};
```

```C++
/* Shared_test.cc */
#include "Shared_test.h"
#include <cassert>
#include <cstring>

void TcpServer::closeConnections(int fd) {
  auto it = connections_.find(fd);
  assert ( it != connections_.end() );
  connections_.erase(it); // remove it!
}

void TcpServer::newConnection(int fd) {
  TcpConnection* conn = new TcpConnection;
  std::shared_ptr<TcpConnection> pConn(conn);
  pConn->setTcpServerCloseCallback(
          std::bind(&TcpServer::closeConnections, this, std::placeholders::_1)
          );
  pConn->init(fd);
  connections_[fd] = pConn;
}

bool TcpServer::hasChannel(int fd) {
  auto it = connections_.find(fd);
  if (it == connections_.end()) {
    return false;
  }
  return true;
}


// ************** TcpConnection **************
void TcpConnection::init(int fd) {
  rawPointer_ = new Channel;
  channel_ = std::shared_ptr<Channel>(rawPointer_);
  channel_->setCloseEventCallback(tcpServerCloseCallback_);
  channel_->setFd(fd);
  channel_->tie(shared_from_this()); // for tie
}

void TcpConnection::setTcpServerCloseCallback(TcpServerCloseCallback cb) {
  tcpServerCloseCallback_ = std::move(cb);
}

// ************** Channel **************
void Channel::handleEvent() {
  std::shared_ptr<void> lock = tie_.lock(); // tie! count=2
  tcpServerCloseEventCallback_(fd_); // remove ok, count=1
}

void Channel::setCloseEventCallback(TcpServerCloseCallback cb) {
  tcpServerCloseEventCallback_ = std::move(cb);
}

void Channel::setFd(int fd) {
  fd_ = fd;
}

void Channel::tie(const std::shared_ptr<void>& conn) {
  tie_ = conn; // weak_ptr
}
```

具体调用栈则为&Channel::handleEvent被触发，当发生关闭事件时，提升其中的weak_ptr，使TcpConnection的引用计数变为2，然后进入TcpServer的删除回调，从connctions_中擦除对应的TcpConnction，此时计数降至1，回调结束后函数栈跳回&Channel::handleEvent，当handleEvent退出后，引用计数降至0，TcpConnection被释放。

### std::bind的中的this裸指针问题？

// TODO: wait for update...







