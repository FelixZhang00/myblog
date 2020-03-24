# JVM的简单实现
title:  JVM的简单实现
tag: java

---
本文介绍java虚拟机的一些知识，并以[jvmgo](https://github.com/zxh0/jvmgo)为例介绍一些虚拟机的简单实现。jvmgo是用Go语言实现的java虚拟机，其作者说这个项目的主要目的是学习Go和JVM，所以只是一个toy，对于破除JVM的神秘感还是很有帮助的。

## class类文件结构
使用java编译器（java程序用javac，Groovy程序用groovyc编译器）可以把java代码编译位存储字节码的class文件，虚拟机并不关心class文件的来源是何种语言。这种做法达到了语言无关性的目的。另外有各种可以运行在不同操作系统上的虚拟机，都可以载入和执行同一种平台无关的字节码，实现了平台无关性。

class文件是一组以8位字节为基础单位的二进制流，占用8位字节以上空间的数据项时以大端方式存储，最高位字节在地址最低位。
Class文件格式采用下面伪结构来存储数据，只有两种数据类型：无符号数和表。无符号数可以作为指向表的索引，或者bitmask。
```c
ClassFile {
    u4             magic;
    u2             minor_version;
    u2             major_version;
    u2             constant_pool_count;
    cp_info        constant_pool[constant_pool_count-1];
    u2             access_flags;
    u2             this_class;
    u2             super_class;
    u2             interfaces_count;
    u2             interfaces[interfaces_count];
    u2             fields_count;
    field_info     fields[fields_count];
    u2             methods_count;
    method_info    methods[methods_count];
    u2             attributes_count;
    attribute_info attributes[attributes_count];
}
```
class字节码文件可以用图形化工具[classpy](https://github.com/zxh0/classpy)查看，比命令行工具javap更加方便参看。效果如下：
![](https://ws1.sinaimg.cn/large/8b331ee1gy1fk9xxqguobj20qo0fmqms.jpg)
在[jvmgo](https://github.com/zxh0/jvmgo)中用ClassFile结构体表示，把.class文件以字节流的方式读出，然后填到这个结构体中。

calss文件格式详情可以看《Java虚拟机规范》和jvm相关文档:[The class File Format](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-4.html).
这里简单举几个例子。

### 方法表
一个方法用如下数据结构表示后缀_index表示是指向常量池的索引。
name_index指出了方法名。
descriptor_index指出了方法返回值和参数列表信息，是java中方法重载的关键。
```cpp
method_info {
    u2             access_flags;
    u2             name_index;
    u2             descriptor_index;
    u2             attributes_count;
    attribute_info attributes[attributes_count];
}
```
如下是javac编译器为类自动生成的`<init>`默认构造函数，它的名称索引和描述符索引分别指向常量池中对应的位置。
方法表第0项：
![](https://ws1.sinaimg.cn/large/8b331ee1gy1fk9yuptfkyj20d4094q3p.jpg)
常量池第12、13项：
![](https://ws1.sinaimg.cn/large/8b331ee1gy1fk9yyrdm38j208o0b8t9b.jpg)

### 属性表
```cpp
attribute_info {
    u2 attribute_name_index;
    u4 attribute_length;
    u1 info[attribute_length];
}
```
在Class文件、字段表、方法表都可以携带自己的属性表结合，用于描述某些场景专有的信息。
属性是可以扩展的，不同的虚拟机实现可以定义自己的属性类型。由于这个原因，Java虚拟机规范没有使用tag，而是使用属性名来区别不同的属性

Code属性中存放字节码等方法相关信息。
Code是变长属性，只存在于method_info结构中。在classpy中观察main方法的code属性如下，其中max_stack代表了操作数栈(Operand Stacks)深度的最大值。max_locaks代表了局部变量表所需的存储空间（包括方法参数），code_length和code用来存储java程序编译后生成的字节码指令。
![](https://ws1.sinaimg.cn/large/8b331ee1gy1fk9z5t2wllj20j00qqn01.jpg)

---

## 运行时数据区
![运行时数据区](https://ws1.sinaimg.cn/large/8b331ee1gy1fk9zjn57t3j21fs0ya3zx.jpg)
在运行Java程序时，虚拟机需要使用内存来存放各种的数据，这个内存区域就是运行时数据区。
多线程共享的内存区域主要存放两类数据：类数据和类实例 （也就是对象Object）。对象数据存放在堆（Heap）中，类数据存放在方法区 （Method Area）中。堆由垃圾收集器GC定期清理。类数据包括字段和方法信息、方法的字节码、 运行时常量池，等等。
线程私有的运行时数据区用于辅助执行Java字节码。每个线程都有自己的pc寄存器（Program Counter）和Java虚拟机栈（JVM Stack）。Java虚拟机栈又由栈帧（Stack Frame）构成，帧中保存方法执行的状态，包括局部变量表（Local Variable）和操作数栈（Operand Stack）等。如果当前方法是Java方法，则 pc寄存器中存放当前正在执行的Java虚拟机指令的地址，否则，当前方法是本地方法，pc寄存器中的值没有明确定义。

## 解释器

[jvmgo](https://github.com/zxh0/jvmgo)中虚拟机字节码执行引擎代码如下：
```go
func interpret(method *heap.Method) {
	thread := rtda.NewThread()
	frame := thread.NewFrame(method)
	thread.PushFrame(frame)
	loop(thread)
}
	
func loop(thread *rtda.Thread) {
	reader := &base.BytecodeReader{}
	for {
		frame := thread.CurrentFrame()
		pc := frame.NextPC()
		thread.SetPC(pc)
		// decode
		reader.Reset(frame.Method().Code(), pc)
		opcode := reader.ReadUint8()
		inst := instructions.NewInstruction(opcode)
		inst.FetchOperands(reader)
		frame.SetNextPC(reader.PC())
		// execute
		inst.Execute(frame)
		if thread.IsStackEmpty() {
			break
		}
	}
}
```
interpret()法的参数是MemberInfo指针，调用MemberInfo结 构体的CodeAttribute()法可以获取它的Code属性，从class文件结构中得到bytecode、maxstack等信息后，创建一个Frame，在一个loop中不停循环解释字节码。

每个指令是一个u1类型的单字节，当虚拟机读取到code中的一个字节码时，就可以对应找出这个字节码代表什么指令，并且可以知道这条指令后面是否需要跟随参数，以及参数应当如何理解。
如iload指令根据第一个操作数作为索引从局部变量表取出一个int值，然后push到操作数栈。
指令比较多，做个总结的话，无非是从操作数栈或者局部变量表取出来，算一算，把结果再放回运行时数据区，如果遇到跳转指令就改变下frame上的pc。

## 类加载机制
虚拟机的类加载机制：虚拟机把class文件加载到内存，并对数据进行校验、转换解析和初始化，最终形成可以被虚拟机直接使用的java类型。

java代码在进行javac编译的时候，并没有链接这一步骤，而是在虚拟机加载Class文件的时候进行动态链接。所以在Class文件中不会保存各个方法、字段的最终内存布局信息，当虚拟机运行时，需要从常量池获得对应的符号引用，再在类创建时或运行时解析翻译到具体的内存地址之中。
在jvmgo中有class_loader的parseClass方法完成类和字段符号引用解析。

类的加载大致可以分为三个步骤：首先找到class文件并把数据读取到内存；然后解析class文件，生成虚拟机可以使用的类数据，并放入方法区；最后进行链接。
jvmgo中相关实现如下：
```go
type ClassLoader struct {
	cp          *classpath.Classpath
	classMap    map[string]*Class // loaded classes
}
func (self *ClassLoader) loadNonArrayClass(name string) *Class {
   data, entry := self.readClass(name)
   class := self.defineClass(data)
   link(class)
   return class
}
// jvms 5.3.5
func (self *ClassLoader) defineClass(data []byte) *Class {
   class := parseClass(data) //把class文件数据转换成Class结构体
   class.loader = self
   resolveSuperClass(class) //解析类符号引用
   resolveInterfaces(class)
   self.classMap[class.name] = class
   return class
}
// jvms 5.4.3.1
func resolveSuperClass(class *Class) {
   if class.name != "java/lang/Object" {
      class.superClass = class.loader.LoadClass(class.superClassName)
   }
}
func link(class *Class) {
   verify(class)
   prepare(class) //准备阶段主要是给类变量分配空间并给予初始值
}
func prepare(class *Class) {
	calcInstanceFieldSlotIds(class) 
	calcStaticFieldSlotIds(class) //计算并分配静态变量所需内存
	allocAndInitStaticVars(class)
}
```

## 方法调用
java虚拟机提供了5条方法调用字节码指令：
invokestatic指令：调用静态方法。
invokespecial指令：调用无须动态绑定的实例方法，包括构造函数、私有方法和通过super 关键字调用的超类方法。
invokevirtual指令：调用所有虚方法。
invokeinterface指令：调用接口方法，会在运行时再确定一个实现此接口的对象。
invokedynamic指令：先在运行时动态解析出调用点限定符所引用的方法，然后再自行该方法，分派的逻辑是由用户所设定的引导方法决定的。

方法调用参数传递如下，对于实例方法，Java编译器会在参数列表的前面添加一个参数，这个隐藏的参数就是this引用。
依次把这n个变量从调用者的操作数栈中弹出，放进被调用方法的局部变量表中，参数传递就完成了
![](https://ws1.sinaimg.cn/large/8b331ee1gy1fkalpzj75mj21ew0r0dgz.jpg)

在定位到需要调用的方法之后，Java虚拟机要给这个方法创建 一个新的帧并把它推入Java虚拟机栈顶，然后传递参数。
```go
func InvokeMethod(invokerFrame *rtda.Frame, method *heap.Method) {
   thread := invokerFrame.Thread()
   newFrame := thread.NewFrame(method)
   thread.PushFrame(newFrame)

   argSlotCount := int(method.ArgSlotCount())
   if argSlotCount > 0 {
      for i := argSlotCount - 1; i >= 0; i-- {
         slot := invokerFrame.OperandStack().PopSlot()
         newFrame.LocalVars().SetSlot(uint(i), slot)
      }
   }
```
解释器的loop下一个循环就会从New Frame的开头开始执行。

## 实例化对象

实例化对象主要通过new指令。
new指令的操作数是一个uint16索引，来自字节码。通过这个索引，
可以从当前类的运行时常量池中找到一个类符号引用。
解析这个类符号引用，拿到类数据，然后创建对象（根据类实例变量的个数分配空间），并把对象引用推入栈顶，new指令的工作就完成了。
```go
// Create new object
type NEW struct{ base.Index16Instruction }

func (self *NEW) Execute(frame *rtda.Frame) {
	cp := frame.Method().Class().ConstantPool()
	classRef := cp.GetConstant(self.Index).(*heap.ClassRef)
	class := classRef.ResolvedClass()
	if !class.InitStarted() { //类初始化
		frame.RevertNextPC()
		base.InitClass(frame.Thread(), class)
		return
	}
	ref := class.NewObject()
	frame.OperandStack().PushRef(ref)
}
//创建对象
func newObject(class *Class) *Object {
   return &Object{
      class:  class,
      fields: newSlots(class.instanceSlotCount),
   }
}
```

比如对于如下的java代码
```java
    public MyObject(int a) {
        this.instanceVar = a;
    }

    public MyObject(int a,int b) {
        a = a+b;
        this.instanceVar = a;
    }
    MyObject myObj = new MyObject(100); // new

```
编译后，再用javap反编译如下：
```powershell
 3: new           #4                  // class jvmgo/book/ch06/MyObject
 6: dup
 7: bipush        100
 9: invokespecial #5                  // Method "<init>":(I)V
```
先调用new指令，开辟一个Object的内存空间，再调用构造函数方法，这里jvm将通过索引5找到相关的构造函数

## 总结
jvmgo还实现了数组和字符串、本地方法调用、反射机制、自动装箱和拆箱、异常处理，感兴趣的看看。

---
## 参考
https://github.com/zxh0/jvmgo
https://github.com/zxh0/classpy
《Java虚拟机规范》
《深入理解java虚拟机》
《自己动手写Java虚拟机》

