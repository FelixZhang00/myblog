
# C++与汇编小结
title:  C++与汇编小结
tag: c++ 
---

本文通过C++反编译，帮助理解C++中的一些概念（指针引用、this指针、虚函数、析构函数、lambda表达式），
希望能在深入理解C++其它一些高级特性（多重继承、RTTI、异常处理）能起到抛砖引玉的作用吧

常用反汇编工具有：objdump、IDA Pro、[godbolt](https://gcc.godbolt.org/)
以下代码均使用x86-64 gcc 6.3编译。

## 指针和引用
引用类型的存储方式和指针是一样的，都是使用内存空间存放地址值。
只是引用类型是通过编译器实现寻址，而指针需要手动寻址。

```c++
void funRef(int &ref){
	ref++;
}

int main(){
	//定义int类型变量
	int var = 0x41;
	//int指针变量，初始化为变量var的地址
	int *pnVar = &var;
	//取出指针pcVar指向的地址内容并显示
	char *pcVar = (char*)&var;
	printf("%s",pcVar);

	//引用作为参数，即把var的地址作为参数
	funRef(var);

	return 0;
}
```
  用godbolt查看的效果如图，C++代码与对应的汇编代码用相同的颜色标注，非常方便查看。
![](https://ws1.sinaimg.cn/large/8b331ee1gy1fhkgjfj3g5j21b01864b0.jpg)

---

## switch

在分支数少的情况下可以用if--else if模拟，
但是分支比较大的情况下，需要比较的次数太多，
如果是有序线性的数值，可将每个case语句块的地址预先保存在数组中，
考察switch语句的参数，并依次查询case语句块地址的数组，
从而得到对应case语句块的首地址，
这样可以降低比较的次数，提升效率。

```c++
int main(){
	int nIdx=1;
	scanf("%d",&nIdx);

	int result = 0;
	switch(nIdx){
		case 1:
			result = 1;
			break;
		case 2:
			result = 2;
			break;	
		case 3:
			result = 3;
			break;
		case 5:
			result = 3;
			break;	
		case 7:
			result = 3;
			break;		
	}

	return result;
}
```
如下图，编译器把switch跳转表放到了.L4所指向的区域，其中的元素.L2、.L3 ... .L8指向case对应代码地址。
![](https://ws1.sinaimg.cn/large/8b331ee1gy1fhkglbho4lj21ba1m4ws4.jpg)

---

## this指针

this指针中保存了所属对象的首地址。

在调用成员函数的过程中，编译器利用rdi寄存器保存了对象的首地址，
并以寄存器传参的方式传递到成员函数中。

```c++
#include <stdio.h>

class Location{
public:
	Location(){
		//this指针指向一块16字节的内存区域
		m_x = 1;
		//m_x是一个8字节类型，所以mov一个4字
		//mov     DWORD PTR [rax], 1
		m_y = 2;
		//mov     WORD PTR [rax+4], 2
	}

	short getY(){
		//获取this指针（对象首地址）偏移4处的数据，即m_y的值
		//movzx   eax, WORD PTR [rbp-8+4]
		return m_y;
	}

private:
	int m_x; //占4字节
	short m_y; //占2字节
	//由于内存对齐，整个对象占8字节
};


int main(){
	//在栈上分配16字节，其中有8字节分配给是loc
	//把栈上loc的内存地址（即this指针）作为参数调用Location构造函数。
	Location loc;

	//把栈上loc的内存地址（即this指针）作为参数调用getY成员函数。
	short y = loc.getY();
	//y变量位于[rbp-2]处
	//mov     WORD PTR [rbp-2], ax
	return 0;
}
```
对应的汇编如下：
![](https://ws1.sinaimg.cn/large/8b331ee1gy1fhkglyyv96j21xy174qh5.jpg)

## 虚函数和虚表

编译器会为每一个包含虚函数的类（或通过继承得到的子类）生成一个表，其中包含指向类中每一个虚函数的指针。
这样的表就叫做虚表（vtable）。
此外，每个包含虚函数的类都获得另外一个数据成员，用于在运行时指向适当的虚表。
这个成员通常叫做虚表指针（vtable pointer），并且是类中的第一个数据成员。

在运行时创建对象时，对象的虚表指针将设置为指向合适的虚表。
如果该对象调用一个虚函数，则通过在该对象的虚表中进行查询来选择正确的函数。

代码举例如下，详细代码在[这里](https://gist.github.com/FelixZhang00/173be139404ec703a47dd4d1e52137ad)。
```c++
class BaseClass {
public:
   BaseClass(){x=1;y=2;};
   virtual void vfunc1() = 0;
   virtual void vfunc2(){};
   virtual void vfunc3();
   virtual void vfunc4(){};
   void hello(){printf("hello，y=%d",this->y);};
private:
   int x;//4字节
   int y;
};

class SubClass : public BaseClass {
public:
   SubClass(){z=3;};
   virtual void vfunc1(){};
   virtual void vfunc3();
   virtual void vfunc5(){};
private:
   int z;
};
```

### 虚表布局
下图是一个简化后的内存布局，它动态分配了一个SubClass类型的对象，编译器会确保该对象的第一个字段虚表指针指向正确的虚表。虚表指向编译器为每个类在只读段创建的一块区域，即虚表，类似于数组，其中的大部分元素指向在代码段中的成员函数地址。C++编译器会在编译阶段给这些函数名做name mangling，以实现c++中函数重载、namespace等标准。
![虚表布局](https://ws1.sinaimg.cn/large/8b331ee1gy1fhkgbscy1sj20yk0ysn3h.jpg)

```x86asm
vtable for SubClass:
        .quad   0
        .quad   typeinfo for SubClass ;RTTI相关
        .quad   SubClass::vfunc1() ;this指针中的虚表指针一般指向这个偏移处
        .quad   BaseClass::vfunc2()
        .quad   SubClass::vfunc3()
        .quad   BaseClass::vfunc4()
        .quad   SubClass::vfunc5()
vtable for BaseClass:
        .quad   0
        .quad   typeinfo for BaseClass ;RTTI相关
        .quad   __cxa_pure_virtual ;vfunc1是纯虚函数
        .quad   BaseClass::vfunc2()
        .quad   BaseClass::vfunc3()
        .quad   BaseClass::vfunc4()
```

 SubClass 中包含两个指向属于BaseClass的函数（ BaseClass::vfunc2 和 BaseClass::vfunc4）的指针。
 这是因为 SubClass 并没有重写这2个函数，而是直接继承自BaseClass 。
由于没有针对纯虚函数BaseClass::vfunc1的实现，因此，在 BaseClass的虚表中并没有存储 vfunc1 的地址。
这时，编译器会插入一个错误处理函数的地址，名为 purecall，万一被调用，它会令程序终止或者其他编译器想要发生的行为。
另外，一般的成员函数不在虚表里面，因为不涉及动态调用，如BaseClass中的hello()函数。

###创建对象
这里已在堆上动态创建对象为例。
调用new操作符，在堆上动态分配一块SubClass大小的内存，rax指向这块内存的开始。
SubClass需要的内存大小为`24字节=8(虚表指针)+4*3(3个int类型的成员变量)+4(内存对齐)`
对象首地址的值作为参数调用SubClass构造函数。 
```c++
   BaseClass *a_ptr = new SubClass();
```
```x86asm
main:
		;...
        mov     edi, 24 ;SubClass需要24字节的内存
        call    operator new(unsigned long)
        mov     rbx, rax
        mov     rdi, rbx ;this指针作为参数
        call    SubClass::SubClass()
```

SubClass的构造函数，在完成自身的任务之前会调用基类的构造函数，然后对this指针的内存的虚表指针修改为指向SubClass自身的虚表。
```x86asm
SubClass::SubClass():
        push    rbp
        mov     rbp, rsp
        sub     rsp, 16
        mov     QWORD PTR [rbp-8], rdi
        mov     rax, QWORD PTR [rbp-8]
        mov     rdi, rax ;this指针
        call    BaseClass::BaseClass()
        mov     edx, OFFSET FLAT:vtable for SubClass+16 ;指向SubClass虚表
        mov     rax, QWORD PTR [rbp-8]
        mov     QWORD PTR [rax], rdx ;this指针的虚表指针字段赋值
        mov     rax, QWORD PTR [rbp-8]
        mov     DWORD PTR [rax+16], 3 ;z=3
        nop
        leave
        ret
```
BaseClass的构造函数:
```x86asm
BaseClass::BaseClass():
        push    rbp
        mov     rbp, rsp
        mov     QWORD PTR [rbp-8], rdi ;this指针
        mov     edx, OFFSET FLAT:vtable for BaseClass+16 ;指向BaseClass虚表
        mov     rax, QWORD PTR [rbp-8]
        mov     QWORD PTR [rax], rdx ;this指针的虚表指针字段赋值
        mov     rax, QWORD PTR [rbp-8]
        mov     DWORD PTR [rax+8], 1 ;x=1
        mov     rax, QWORD PTR [rbp-8]
        mov     DWORD PTR [rax+12], 2 ;y=2
        nop
        pop     rbp
        ret
```

### 调用成员函数

#### 1、非虚函数
hello()是类BaseClass中的非虚成员函数，不需要通过虚表查找，编译器直接生成调用语句`call    BaseClass::hello()`，并且第一个参数默认为this指针。
```c++
	BaseClass *a_ptr = new SubClass();
   //一般的成员函数，不在虚表里
   a_ptr->hello();
```
```x86asm
main:
		;...		
        mov     rdi, rax ;参数:this指针
        call    BaseClass::hello()

.LC0:
        .string "hello\357\274\214y=%d"
BaseClass::hello():
        push    rbp
        mov     rbp, rsp
        sub     rsp, 16
        mov     QWORD PTR [rbp-8], rdi  ;this指针放到栈上
        mov     rax, QWORD PTR [rbp-8]
        mov     eax, DWORD PTR [rax+12] ;this指针偏移12处，即成员变量y的位置
        mov     esi, eax              ;参数:format的数据，即y的值
        mov     edi, OFFSET FLAT:.LC0 ;参数:format string
        mov     eax, 0                ;参数:fd，指向stdout
        call    printf
        nop
        leave
        ret
```

#### 2、虚函数
a_ptr是BaseClass类型的指针，动态分配的是SubClass类型的内存。
call_vfunc函数的参数是基类BaseClass，再调用vfunc3函数时需要先根据虚表指针定位到虚表，再通过偏移，解引用找到vfunc3的代码段地址，完成调用。
```c++
int main(){
	BaseClass *a_ptr = new SubClass();
	//对象首地址作为参数调用函数call_vfunc
   call_vfunc(a_ptr);
}
void call_vfunc(BaseClass *a) {
   a->vfunc3();
}  
```
```x86asm
main:
	;...	
	mov     rax, QWORD PTR [rbp-24]
    mov     rdi, rax ;rax为this指针
    call    call_vfunc(BaseClass*)
 
call_vfunc(BaseClass*):
        push    rbp
        mov     rbp, rsp
        sub     rsp, 16
        mov     QWORD PTR [rbp-8], rdi ;把this指针放到栈上
        mov     rax, QWORD PTR [rbp-8]
        mov     rax, QWORD PTR [rax]   ;this指向的内存的开头8个字节的数据复制给rax,即虚表指针
        add     rax, 16                ;找到虚表指针偏移16
        mov     rax, QWORD PTR [rax]   ;虚表指针偏移16处解引用，得到函数的SubClass::vfunc3的地址
        mov     rdx, QWORD PTR [rbp-8]
        mov     rdi, rdx              ;rdi为this指针，作为参数
        call    rax                   ;调用vfunc3
        nop
        leave
        ret   
```

---

## 析构函数

这里以堆分配的对象析构为例，完整代码在[这里](https://gist.github.com/FelixZhang00/4bcfeec2ba3f8568a6da8e4151379800)。
堆分配的对象的析构函数在分配给对象的内存释放之前通过 delete 操作符调用。
其过程如下：
1、如果类拥有任何虚函数，则还原对象的虚表指针，使其指向相关类的虚表。如果一个子类在创建过程中覆盖了虚表指针，就需要这样做。
2、执行程序员为析构函数指定的代码。
3、如果类拥有本身就是对象的数据成员，则执行这些成员的析构函数。
4、如果对象拥有一个超类，则调用超类的析构函数
5、如果是释放堆的对象，则用一个代理析构函数执行1~4步骤，并在最后调用delete操作符释放堆上的对象。

```c++
class BaseClass {
public:
   BaseClass(){x=1;y=2;};
   virtual ~BaseClass(){printf("~BaseClass()\n");};
   virtual void vfunc1() = 0;
private:
   int x;//4字节
   int y;
};

class SubClass : public BaseClass {
public:
   SubClass(){z=3;};
   virtual ~SubClass(){printf("~SubClass()\n");};
   virtual void vfunc1(){};
private:
   int z;
};


int main() {
   BaseClass *a_ptr = new SubClass();
   //触发析构 
   delete a_ptr; 
} 
```

```x86asm
;只读段中的虚表结构
vtable for SubClass:
    .quad   0
    .quad   typeinfo for SubClass
    .quad   SubClass::'scalar deleting destructor' ;代理析构函数的地址
    .quad   SubClass::~SubClass() ;析构函数的地址，这里godbolt没有把它们区分出来
    .quad   SubClass::vfunc1()
        
main:
     mov     QWORD PTR [rbp-24], rbx  ;rbx为a_ptr的指针
     cmp     QWORD PTR [rbp-24], 0    ;判断a_ptr是否为null，这是编译器加的。
     je      .L9                      ;如果为null直接跳过析构
     mov     rax, QWORD PTR [rbp-24]
     mov     rax, QWORD PTR [rax]
     add     rax, 8                   ;this指针偏移8处，即指向代理析构函数
     mov     rax, QWORD PTR [rax]     ;rax为代理析构函数的地址
     mov     rdx, QWORD PTR [rbp-24] 
     mov     rdi, rdx                 ;参数：this指针
     call    rax                      ;调用代理析构函数

;SubClass的代理析构函数
SubClass::'scalar deleting destructor':
        push    rbp
        mov     rbp, rsp
        sub     rsp, 16
        mov     QWORD PTR [rbp-8], rdi
        mov     rax, QWORD PTR [rbp-8]
        mov     rdi, rax              ;参数:this指针
        call    SubClass::~SubClass() ;调用析构函数
        mov     rax, QWORD PTR [rbp-8]
        mov     esi, 24               ;参数:释放24字节大小的堆空间
        mov     rdi, rax              ;参数:堆空间的首地址
        call    operator delete(void*, unsigned long) ;释放堆空间
        leave
        ret
        
 .LC1:
        .string "~SubClass()"       
;SubClass的析构函数，执行析构函数中的代码
SubClass::~SubClass():
    push    rbp
    mov     rbp, rsp
    sub     rsp, 16
    mov     QWORD PTR [rbp-8], rdi
    mov     edx, OFFSET FLAT:vtable for SubClass+16
    mov     rax, QWORD PTR [rbp-8]
    mov     QWORD PTR [rax], rdx      ;还原对象的虚表指针，使其指向相关类的虚表
    mov     edi, OFFSET FLAT:.LC1
    call    puts                      ;调用puts函数，这里编译器把printf调用转换成puts了。
    mov     rax, QWORD PTR [rbp-8]
    mov     rdi, rax                  ;参数：this指针 
    call    BaseClass::~BaseClass()   ;调用基类的析构函数
    nop
    leave
    ret

.LC0:
        .string "~BaseClass()"
;BaseClass的析构函数，执行析构函数中的代码        
BaseClass::~BaseClass():
        push    rbp
        mov     rbp, rsp
        sub     rsp, 16
        mov     QWORD PTR [rbp-8], rdi
        mov     edx, OFFSET FLAT:vtable for BaseClass+16
        mov     rax, QWORD PTR [rbp-8]
        mov     QWORD PTR [rax], rdx
        mov     edi, OFFSET FLAT:.LC0
        call    puts
        nop
        leave
        ret

```
通过分析C++析构函数的调用过程，我们就知道了为什么C++基类的析构函数要声明为virtual了。我们希望当调用C++基类BaseClass的析构函数时能够触发动态绑定，能够找到当前对象所属类的虚函数表中的析构函数。
如果不声明BaseClass的析构函数为virtual，那么在调用`delete a_ptr`时，将只会释放BaseClass大小的内存，给SubClass中成员变量分配的内存将得不到释放，从而导致内存泄漏。

## C++11中的Lambda表达式
lambda表达式表示一个可调用的代码单元。可以理解为一个未命名的内联函数。
lambda表达式具有如下形式：
```
[capture list](parameter list) -> return type {function body}
```
下面定义了一个C++函数，其中有一个lambda表达式。v1之前的&符号指出v1是以引用方式捕获，当lambda返回v1时，它返回的是v1指向对象的值，所以j的值是0，而不是42.
```
void fcn1(){
    int v1 =42;
    auto f= [&v1] {return v1;};
    v1 = 0;
    auto j = f();
}
```
对应的反汇编代码如下，可以看到编译器为fcn1中的lambda表达式在代码段中生成了一段指令，当调用这个lambda时就会执行到这段指令，跟普通的函数调用一致。
可以看出传递给`fcn1()::{lambda()#1}`函数的参数rdi的值其实就是v1变量的地址，所以这个lambda是是采用引用方式捕获变量的。
```
.Ltext0:
fcn1()::{lambda()#1}::operator()() const:

        push    rbp
        mov     rbp, rsp
        mov     QWORD PTR [rbp-8], rdi
        mov     rax, QWORD PTR [rbp-8]
        mov     rax, QWORD PTR [rax]
        mov     eax, DWORD PTR [rax]
        pop     rbp
        ret
fcn1():
        push    rbp
        mov     rbp, rsp
        sub     rsp, 16
        mov     DWORD PTR [rbp-8], 42
        lea     rax, [rbp-8]
        mov     QWORD PTR [rbp-16], rax
        mov     DWORD PTR [rbp-8], 0
        lea     rax, [rbp-16]
        mov     rdi, rax
        call    fcn1()::{lambda()#1}::operator()() const
        mov     DWORD PTR [rbp-4], eax
        nop
        leave
        ret
```

---
## 参考
《IDA Pro权威指南》
《C++反汇编与逆向分析技术揭秘》
《C++ Primer（第5版）》