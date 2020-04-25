---
title: c++基础
date: 2019-01-04 20:07:06
tags: 基础
categories: C
---

# c++基础
## 一、常见问题
### 指针 数组
关于指针,实际就是addr,在汇编代码中对应着M[R[xx]];引用类型,只是语法对指针做的一层封装,在传递值的语言以及函数中,拥有对象控制权限,那么传递的就是引用.

1.指针的实质:

- 对于指针的类型决定了其每次进行移位操作时,内存地址的偏移大小,而任意类型指针其所占的内存大小咋取决于程序和机器内存地址位数[^1]
- void \*类型指针可以代表任意指针类型,作为形参可以接受任意指针,指针之间的互相转换也是非常合理的.
- 指向指针指针类型如 int\*\*p,从语法角度来理解p是一个指针,其指向了一个int\*指针,也就是说该指针内存存放这一个指针的地址 `int \*p1=0; int \*\*p=&p1`;
- 只想指针的引用类型 \*&ref
[^1]:对于熟悉的我就不会列出例子

2.数组的实质:

数组表示一段连续的空间,c++很容易将指针和数组概念不能完全分清楚.
```c++
//1.栈上分配数组
  int a[3] = { 1,2,3 };
	int *a1= a;
	int a2 = *a;
	cout << a << endl;
	cout << &a << endl;
	cout << a1<<endl;
	system("pause");
```
汇编代码:
首先[addr]不是间接取址 而是直接取址
```c++
	int a[3] = { 1,2,3 };
00C64CB8  mov         dword ptr [ebp-14h],1  //此处的地址就是a符号
00C64CBF  mov         dword ptr [ebp-10h],2  
00C64CC6  mov         dword ptr [ebp-0Ch],3  
	int *a1 = a;                            //将a赋予一个指针,就是将该数组首位内存地址计算出来并且给了指针
00C64CCD  lea         eax,[ebp-14h]  
00C64CD0  mov         dword ptr [ebp-20h],eax  
	int a2 = *a;
00C64CD3  mov         eax,4  
00C64CD8  imul        ecx,eax,0   //计算偏移
00C64CDB  mov         edx,dword ptr [ebp+ecx-14h]  ecx=0 [ebp-14h]=a
00C64CDF  mov         dword ptr [ebp-2Ch],edx  
```
    
```c++
2.堆上分配数组
int *a=new int[4]{1,2,3,4} //c++中使用new关键字后,该函数本身就会返回一个指针
//此中情况下a指针指向的就是数组元素的地址

//以上结论可以通过反汇编自行证明
```
结论:

- 对于数组如果是局部变量其本身就是通过栈维护的几个连续元素,语法上使用首元素[index]的形式进行访问;
- 如果将局部数组a符号赋值给数组,那么相当于引用了新的指针变量,指针变量内容为数组元素地址
- 使用堆数组则表现的如同java语言  

3.关于数组和指针在语法上的一些使用:
- c++中语法中数组名=指针
- 既然使用指针来控制数组,那么每次进行的偏移量是根据指针类型确定的,而数组长度则未知,在传递参数时,要告知数组长度  
```c++
//基本类型指针和数组参数传递
    void point(int*p)
    {
        std::cout<<"int*p";
    }
    //对于数组名[xx] 和指针 对于编译器来说当作参数时相同对待 看作一个指针值
    //这两个函数重复定义
//    void point(int array[])
//    {
//        std::cout<<"int*p";
//    }
//    void point(int array[10])
//    {
//        std::cout<<"int*p";
//    }
    /**
     *
     * @param p 指向int[10] 这种类型数组名,每次偏移为type*length ,接受类型为数组指针
     */
    void pointArrayFun(int (*p)[10])
    {
        std::cout<<p[1]-p[0];
        //指针的类型就是控制每次偏移的位置,对于p来说每次偏移的量就是10*4
    }
    /**
     *
     * @param p为指向指针的指针,可以接受类型为 type *array[xx] 指针数组
     */
    void arrayPointFun(int**p){
        std::cout<<p<<std::endl; //本例子中代表了arrayPoint数组的首位地址
        std::cout<<*p<<std::endl;//代表了a的地址
    }
    


    void pointAndArray(){
        int a=3;
        int b=3;
        int array[10]={1,2,3,4,5,6,7,8};

        int* arrayPoint[10]={&a,&b};
        //匹配指针
        arrayPointFun(arrayPoint);

        int (*pointArray)[10]=&array;
        pointArrayFun(pointArray);
    }
```
4.函数指针和函数名
- 函数名等同于函数指针,也就是说在作为形参,函数名可以匹配函数参数和函数指针参数,无非是在使用的时候是否要使用\*符号
- 当返回复杂类型 使用 auto fun()->return type

------  



### 作用域和生命周期
c++的作用域有编译单元即文件,函数,函数内部出现的{}符号 

- 全a局符号:
 - delclear 声明 如外部函数声明,外部变量声明属于`弱符号`,对于c++未使用extern关键的外部变量声明会被编译器解释为全局变量这样的强符号
 - define 全局函数=类函数,全局变量定义都属于`强符号`
 - static 修饰静态全局变量,为当前编译单元所有,静态变量可以声明在函数体中
- 局部变量:
定义在函数体作用域中,编译过程分配在栈上的变量,当作用域结束后随着栈回收而回收
- 关于类:
c++将类作为了全局符号,类中的函数作为了全局函数,static 类函数只是从编译器的角度来看是静态类函数,而不是和默认static 作为静态单元关键字来使用
```c++
//t.h
#ifndef ELFREAD_T_H
#define ELFREAD_T_H
class T{
    void test();
};
#endif //ELFREAD_T_H
```
```c++
#include "t.h"
#include <iostream>
void T::test() {
    std::cout<<123;
}
```
```c++
#ifndef ELFREAD_T2_H
#define ELFREAD_T2_H
class T{
    void test();
};
#endif //ELFREAD_T2_H
```
```c++
#include "t2.h"
#include <iostream>
void T::test() {
    std::cout<<456;
}
```
    这两个cpp在链接过程就会出现全局符号重复定义

头文件在c++语言中的作用就是将所谓的声明和定义分开,去看一下链接过程就能完全明白,按照上述定义的符号来理解,头文件的用法就是用来写外部符号,也就是所谓的 declear声明.
```
CMakeFiles\elfRead.dir/objects.a(test2.cpp.obj): In function `ZN1T4testEv':
F:/code/clionProject/elfRead/test2.cpp:6: multiple definition of `T::test()'
CMakeFiles\elfRead.dir/objects.a(test.cpp.obj):F:/code/clionProject/elfRead/test.cpp:6: first defined here
```
    这是minGW链接时提示的错误,说明类函数的确是被汇编器当作全局符号的,链接过程在符号解析的过程中抛出错误;
    即使此处不适用头文件,直接使用两个cpp,里边写相同的class定义,也会如此.
补充:
对于语法
```c++
class A{
type fun(){
}
}
//至少在minGW编译器中会将之看作声明,就是说在头文件中,可以进行类函数的实现,当该函数被引入其实现cpp中,如果在cpp再次实现该函数,会在`预处理`过程中提示`重复定义`
//当这种类头文件被引入其他使用者的编译头中,并不会在链接过程中出现任何问题,按照逻辑来说t.h 进行声明,t.cpp中进行定义,如果t.h中实现了一个类函数,就相当于一个全局符号,当xx.cpp引如t.h,在预处理展开后,也就出现了一个全局符号,链接时就应该出错.
//但是事实与之相反,并不会出现链接错误,就说明上述语法被编译器作为声明而非定义
```
```c++
//main.cpp
#include <iostream>
class T{
public:
    void test(){};
    void x();
};
int main(){
    T t;
    t.test();
    t.x();
}
//此时 t.test t.x
//t.h
#ifndef ELFREAD_T_H
#define ELFREAD_T_H

#include <iostream>
class T{
public:
    void test(){
        std::cout<<123;
    };
    void x();
};

#endif //ELFREAD_T_H
//test.cpp
#include <iostream>
#include "t.h"
void T:: x(){
    std::cout<<"xx";
}

```
---------
### 构造器相关
1.默认构造器
默认构造器不完成任何内容,定义了其他构造器后,编译器不会再创建默认构造器,可以通过default关键字保留默认构造器  XX()=default; 
2.构造器初始化列表顺序
```c++
 class F{
  int a;
  int b;
  public:
    //构造器初始化顺序时按照变量位置进行的
    F(int c,int d):b(c),a(b){
        std::cout<<a<<std::endl;
        std::cout<<b<<std::endl;
        //4201358
        //1
    }
 }
```
3.拷贝构造函数和赋值操作符完成了深拷贝工作
- 拷贝构造函数的调用时机:
```c++
class Person
{
private:
    int age;
    string name;
    F(const F&f)//默认的拷贝
    F&operator=(const F&f)// 默认的赋值运算符
};
void test(){
Person p;
Person p2=p; //创造新的对象时调用 也就是这条语句编译器认为是调用了拷贝构造器 参数是p2的地址和p的地址
Pserson p=p2; //赋值运算符
}
//发生值传递/返回值也会调用拷贝构造函数
/*
* 对于这种返回值的情况
*  1.如果Person重写了析构函数,也就是说明Person含有指针对象,并且是在析构中delete了指针对象,那么对于形参p的拷贝对象实参p,就会在结束时造成实参p内存指针内存被回收
   2.编译器优化会自动使用移动语义,就会避免返回值的析构发生
/
void test2(Person p){
 return p; //此时在AX寄存器中又创建了一个p对象
 //当该函数结束时,会调用参数p对象的析构和返回p对象的析构
}
void test3(){
 Person p;
 Pserson px;
 Person p2=test2(p); //拷贝构造
 px=test2(p); //赋值运算符
}
```
了解一个事实,编译器对于形参无论是X86还是IA32都创建了一个`新的`,无论是任何类型,形参所占用的空间和实参是相同
 - 基本数据类型形参在栈或者寄存器拷贝一份当作形参
 - 指针类型相同,指针本身对于汇编来说也只是一个 `值`而已
 - 类类型 分为值传递和引用传递, 前者对于栈对象,如果直接传递对象本身,相当于调用构造器创建新的形参对象;后者指栈对象传递引用或者取地址,都会传递该对象首地址,堆对象和传递引用和指针是相同的.
   
```c++
 class F{
  int a;
  int b;
  public:
  F(int c,int d):a(c),d(b){}
  F(F&f){a=f.a;b=f.b;} //拷贝构造函数,参数必须是引用类型,否则相当于不断的调用拷贝构造函数
 }
```

4.析构函数
- 会在栈对象销毁后调用,堆对象调用delete过程中free函数调用前调用
- 析构函数本身{}中由程序员编写,用来释放类中指针成员,指针成员编译器自动释放
- 如果重写了析构函数,那么相对来说就要重写一个拷贝构造函数和拷贝赋值运算符,因为拷贝对于指针成员来说是浅拷贝,指针会指向同一块,重写的拷贝函数应该完成指针对象构造,具体看3.
- 指针作为基本数据类型,并不会触发析构函数,所以传递类对象时,传递引用就不会发生上述重写析构导致的内存回收

5.委托构造函数,构造器调用前调用另一个构造完成初始化,以及其中的逻辑代码

```c++
 class F{
  int a;
  int b;
  public:
  F(int c,int d):a(c),d(b){}
  F():F(1,2){} //委托构造
 }
```
6.隐式类型转换可以通过explicit关键字屏蔽
7.仅仅使用数据的类使用struct
8.关于c++ 指针 引用 值 类构造器
- 首先将指针和引用看作相同的,两者都占用空间,并且其值都是地址,只是c++编译器在语法上对两者进行了区别,属于不同类型,可以参与重载
- 值传递:参数在汇编指令下会进行一次值复制操作,无论该实参的类型,只是在c++语法层面将这个问题复杂化了
1.关于基本类值传递,就是简单的复制操作;对于简单类型的引用或者指针传递,被称之为引用传递,实际对于引用/指针来说还是存在一次地址值的复制操作,作为寄存器或栈值进行函数调用,只是该参数的行为改动的是原值的
2.对于复杂类型类的值传递问题,对于语法层面的值传递,会导致拷贝构造函数的调用,返回值也是,都存在了创建一份新的对象的操作;当传递指针和引用时候,值传递的是地址值,此时的参数复制工作并不会涉及原对象的拷贝工作
3.关于析构函数,析构函数本身并不做什么工作,对于类成员变量的清除,默认析构函数也不会进行;对于汇编代码的堆栈指针变化造成的所谓`回收堆栈`也不会对内存本身数据造成改变,只是在语法上不让代码访问,形参也是如此,不过X86形参会放在寄存器中;这里要说的意思是,只有你的析构函数主动清除了指针成员指向的内存,此时返回临时引用才会造成问题.

---------

## 二、关键字
### const
1. 基础用法:
```c++
const int const* a //前者为顶层const  后者底层const
```

顶层const 表示 a不能再被赋值 ,底层const 表示不能通过*a进行赋值操作
### typedef和using
1. 基础用法:
```c++
//1.使用别名
typedef typeName type;
using typeName=type;
//2.using 使用命名空间
using namespace xx; //不推荐这样使用会导致命名空间污染
```
### auto
1. 基础用法:
```
//1.自动推断类型
auto returnFun a;
//2.后置返回类型
auto fun(paras)->return Type
```
### decltype
1. 基础用法:
```
//获得类型
decltype (expression) 
```
### 命名空间namespace
1. 基础用法:
```
//1.定义命名空间
namesapce xx{
code
}
//2.使用命名空间
xx::fun
//3.匿名命名空间
namesapce{
}
匿名命名空间和static关键字都可以将该变量 函数的作用域控制在当前文件中
```
关于匿名命名空间可以参看[google c++ style](https://zh-google-styleguide.readthedocs.io/en/latest/google-cpp-styleguide/scoping/#unnamed-namespace-and-static-variables#)
### class struct union
1. 基础用法:
    class struct只是默认权限不同,一般仅仅当不含有成员函数的情况使用struct;
    union {
    int x
    double y
    } 同时只能一个变量占用内存,也就是说其内存大小= `sizeof (int)` /`sizeof (double)`
    
## 三、宏
### 1.使用宏的保护机制
```
#ifndef xx
#define xx
#endif
```
- 关于xx的命名方式要体现项目的全路径,如 FOO_BAR_BAZ_H_
- 关于c++的编译单元是以单个文件进行的,宏保护就是对于单个文件进行的保护
### 2.使用\表示宏的换行
```
//测试函数的宏定义
#define Test(Name) \
extern void testFunction(void);\
class TestClass    \
{                  \
  public:          \
    TestClass()    \
 {                 \
   cout<<#Name;    \
   testFunction(); \
 }                 \
}instance;          \
void testFunction(void)
//如何使用
Test(fun)
{
std::cout<<"test";
}
```
这个例子还能看出来extern 关系不仅仅在链接时取链接外部函数,还能链接同一个文件中的函数
## 四、其他特性
### 1.使用{}进行初始化
```
 int array[3]={1,2,3}
 std::vector<int>vec={1,2,3} //google中提倡使用这个替代可变参数
```
### 2.关于c++内存管理
1. 编译器只会主动调用栈对象的析构函数,而程序员要自己确定的事情是当该类中含有堆内存的时候要在析构函数中进行delete,delete的作用是调用析构函数,并且释放该指针指向的堆内存
```c++
class Innner{
type * var;
~(){
if(var)
delete var;
}
}
class F{
Inner * in;
~(){
if(in)
delete in;
}
}
void test2(F f){ //如果如此直接传递值,则会引起临时对象析构,导致内存问题
return f;
}
void test()
{
//仅仅当`句柄`为栈对象的时候才会编译器才会按照如此进行,在于该内存在函数作用域结束后就无法生存
F f;// 当该函数结束自动释放内存
//如果时堆指针
F *f=new F()//函数结束后则不会调用析构函数
}
```
对于栈对象
- 又因为值传递问题,引出了 拷贝构造函数在 栈对象参数传递  栈对象返回时导致的析构,可能会引起内部指针指向同一处造成的内存问题,如果非要进行值传递,那么就要定义值传递的拷贝构造函数,以及值传递的赋值运算符
- 关于栈对象中共享堆成员内存,那么就要定义引用传递的拷贝构造函数和赋值运算符,以及析构函数
{%codeblock 值行为的类 lang:cpp%}
Class Inner{
public:
int a;
Inner(int a):a(a){}
}
class F{
Inner* in;
F(Inner* in):in(in){}
//值传递的拷贝构造函数
F(const &F f){
  this->in=new Inner(f.in.a);
}
F& oprator=(const &F f)
this->in=new Inner(f.in.a);
}
//此时的析构函数按照正常的就可以
~F(){
if(in)
delete in;
}
{%endcodeblock%}
{%codeblock 引用行为的类 lang:cpp%}
Class Inner{
public:
int a;
Inner(int a):a(a){}
}
class F{
Inner* in;
std::size_t* count;
F(Inner* in):in(in),count(1){}
//引用行为的拷贝构造函数
F(const &F f){
  this->in=f.in;
  this.count=f.count;
  this.count++;
}
//引用行为的赋值运算符
F& oprator=(const &F f)
 count--;
 clear();
 this.in=f.in;
 this.count=f.count;
 count++;
}
//此时的析构函数按照正常的就可以
~F(){
count--;
clear();
}
void clear(){
 if(count==0)
 delete in;
}
void test(){

}
//如此就完成了对于中栈对象值行为和拷贝行为的管理
{%endcodeblock%}
对于堆对象
- 还是上述的问题,堆对象句柄c++编译器不会为之进行主动析构,只有程序员主动调用析构函数,理解一下无论时栈还是堆句柄,我们所做的都应该是在其句柄声明周期结束的时候去判断是否应该是否该内存,对于分配在栈上的内存,编译器就是很自信的直接清除了,而令人困惑的就是堆这部分内存.
- 我的理解是,如果你创建了一个栈对象,并且保持值行为,那么这个对象的内存无论是栈成员,还是指针堆成员,都应该伴随着该句柄的生命而结束;如果定义的是引用行为,那么就要使用引用计数来判定何时释放堆成员.
- 使用原生指针作为堆句柄时,问题在于你没有办法使用c++本身语法进行引用计数判定
- 一般来说存在new 就要有与之匹配的delete
{%codeblock 堆句柄 lang:cpp%}
Class A{
};
void test(){
A *a=NEW A();
A *a2=NEW A();
//对于指针数据类型本身来说仅仅是一次值覆盖过程,但是造成了a本身指向的内存没有办法释放
//原因在于type*这个类型实质还是指针,没有办法执行 所谓的引用行为赋值运算符,通过智能指针将指针封装成对象类型就能够限制此行为
a=a2; 
}
{%endcodeblock%}
智能指针的实质就是通过将指针封装成栈对象,此时编译器对于智能指针这种栈对象就会按照一般情况来处理
### 3. 左右值
 将没有指向`句柄`的量称之为右值
 - 如临时变量 1 a 等,
 - 由左值运算出的结果未赋值的情况
   string a="123";string b=123"; a+b(右值)

引用函数
```c++
 class Temp{
  void test() &; //匹配this为左值的情况
  void test()&& //匹配this是右值的情况,&和const一样可以参与函数重载
  Temp test2();//该函数函数的就是右值
  Temp& test3();//返回左值
  void test4(){
  test2().test()//匹配左值
  test3().test() //匹配右值函数
  }
 }
```