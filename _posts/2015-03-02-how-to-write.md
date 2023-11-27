---
layout: post
title: 模板匹配
date: 2022-3-02
categories: blog
tags: [C++]
description: C++如何实现函数匹配
---

由一个[cppquiz](https://cppquiz.org/quiz/question/233)的问题引入
```
#include <type_traits>
#include <iostream>

using namespace std;

struct X {
    int f() const&&{
        return 0;
    }
};

int main() {
    auto ptr = &X::f;
    cout << is_same_v<decltype(ptr), int()>
         << is_same_v<decltype(ptr), int(X::*)()>;
}
```
第一反应是应该输出01，但是实际上函数在匹配的时候会根据返回值类型，参数列表，引用修饰符，cv修饰符，异常修饰符来进行匹配，这里const &&明显是没有匹配上的。
>The return type, the parameter-type-list, the ref-qualifier, the cv-qualifier-seq, and the exception specification, but not the default arguments, are part of the function type. [ Note: Function types are checked during the assignments and initializations of pointers to functions, references to functions, and pointers to member functions.  — end note ]

而且这个&&的修饰符让我也很困惑，这到底表达了什么意思，以前确实没有注意过这种修饰。
查了一下发现这是C++11引入的新标准
默认情况下，对于类中用 public 修饰的成员函数，既可以被左值对象调用，也可以被右值对象调用。举个例子：
```
#include <iostream>
using namespace std;
class demo {
public:
    demo(int num):num(num){}
    int get_num(){
        return this->num;
    }
private:
    int num;
};
int main() {
    demo a(10);
    cout << a.get_num() << endl;
    cout << move(a).get_num() << endl;
    return 0;
}
```
可以看到，demo 类中的 get_num() 成员函数既可以被 a 左值对象调用，也可以被 move(a) 生成的右值 demo 对象调用，运行程序会输出两个 10。

某些场景中，我们可能需要限制调用成员函数的对象的类型（左值还是右值），为此 C++11 新添加了引用限定符。所谓引用限定符，就是在成员函数的后面添加 "&" 或者 "&&"，从而限制调用者的类型（左值还是右值）。

修改上面程序：
```
#include <iostream>
using namespace std;
class demo {
public:
    demo(int num):num(num){}
    int get_num()&{
        return this->num;
    }
private:
    int num;
};
int main() {
    demo a(10);
    cout << a.get_num() << endl;          // 正确
    //cout << move(a).get_num() << endl;  // 错误
    return 0;
}
```
和之前的程序相比，我们仅在 get_num() 成员函数的后面添加了 "&"，它可以限定调用该函数的对象必须是左值对象。因此第 16 行代码中，move(a) 生成的右值对象是不允许调用 get_num() 函数的。

同理，我们再次修改程序：
```
#include <iostream>
using namespace std;
class demo {
public:
    demo(int num):num(num){}
    int get_num()&&{
        return this->num;
    }
private:
    int num;
};
int main() {
    demo a(10);
    //cout << a.get_num() << endl;      // 错误
    cout << move(a).get_num() << endl;  // 正确
    return 0;
}
```
和先前程序不同的是，get_num() 函数后根有 "&&" 限定符，它可以限定调用该函数的对象必须是一个右值对象。
注意，引用限定符不适用于静态成员函数和友元函数。

const和引用限定符
我们知道，const 也可以用于修饰类的成员函数，我们习惯称为常成员函数，例如：
class demo{
public:
    int get_num() const;
}
这里的 get_num() 就是一个常成员函数。

const 和引用限定符修饰类的成员函数时，都位于函数的末尾。C++11 标准规定，当引用限定符和 const 修饰同一个类的成员函数时，const 必须位于引用限定符前面。

需要注意的一点是，当 const && 修饰类的成员函数时，调用它的对象只能是右值对象；当 const & 修饰类的成员函数时，调用它的对象既可以是左值对象，也可以是右值对象。无论是 const && 还是 const & 限定的成员函数，内部都不允许对当前对象做修改操作。

举个例子：
```
#include <iostream>
using namespace std;
class demo {
public:
    demo(int num,int num2) :num(num),num2(num2) {}
    //左值和右值对象都可以调用
    int get_num() const &{
        return this->num;
    }
    //仅供右值对象调用
    int get_num2() const && {
        return this->num2;
    }
private:
    int num;
    int num2;
};
int main() {
    demo a(10,20);
    cout << a.get_num() << endl;        // 正确
    cout << move(a).get_num() << endl;  // 正确
   
    //cout << a.get_num2() << endl;     // 错误 
    cout << move(a).get_num2() << endl; // 正确
    return 0;
}
```







