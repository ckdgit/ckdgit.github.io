---
layout:     post
title:      flutter mixin探秘
subtitle:   mixin的使用和原理探究
date:       2020-01-19
author:     CK
header-img: img/post-bg-flutter-mixin.jpg
catalog: true
tags:
    - flutter
---


# flutter mixin探秘
---
**转载请标明来源**

**本文是根据flutter v1.9.1版本分析编写。依赖的dart版本是V2.5.0**

本文分为两个部分，第一部分介绍mixin的使用，第二部分是with的实现原理。
### 一、mixin的用法

在最近看flutter SDK中的源码时对其中复杂的mixin和on搞的糊里糊涂的，所以就去详细了解了mixin的用法。

在了解如何使用mixin之前，先简单介绍下mixin，下面是官方的解释：
```
Mixins are a way of reusing a class’s code in multiple class hierarchies.
```
mixin提供了一种在复杂类层次结构中复用代码的方法。

我们知道，Flutter使用的Dart语言是一种面向对象语言，在面向对象中可以复用代码的方式有继承和组合两种常用的方式，那么mixin本质是继承还是组合还是它们的变体或者其它新颖的实现方式呢？这里卖个关子，我们继续看下去。

下面首先介绍和mixin使用相关的一些dart保留字。
#### 1.1 保留字介绍
- mixin：

 mixin用来声明一个“类”，其中除了不能声明构造函数，其它和一个普通类没有区别。
 ```
mixin mixinM {
  //error
  mixinM();

  int i;

  void test() {}
}
 ```
- with：

使用with关键字，后跟一个或多个mixin或者普通类。

**ps**：本文分析时，with后面可以跟mixin或者类（类中不能有构造方法），但在官方更新计划文档中，会在后续版本中把mixin和class进一步隔离开来，with后面不能在后面声明类，为了减少后续代码的适配成本，大家开发时多加注意（不过适配成本并不大）
```
class ClassDemo with mixinM {
  String name;

  ClassDemo();

  void testClass() {}
}
```
- on的解释

要指定只有某些特定类型可以使用mixin，使用on来指定所需的超类或者mixin，可以让你编写的mixin可以调用它未定义的方法，并且可以使用super像继承一样调用父类方法。

如下面代码所示，详细看代码注释。
```
mixin mixinA {
  testA() {}
}

mixin mixinM on mixinA {

  int fieldM;

  void test() {
    testA();//调用mixinA的testA方法
  }
}
//要使用mixinM，必须要继承mixinM on的类或者with mixinM on的mixin/类
class ClassDemo with mixinA, mixinM {
  String name;

  ClassDemo();

  void testClass() {
     test();// 使用mixinM的test方法
  }
}
```
#### 1.2 使用详解
在上面我们介绍了mixin使用中所用到的一些保留字，并在一些简单的代码中看到了mixin作为一种代码逻辑复用的方式的使用。我们看到mixin和普通继承的使用非常像，我们定义一个mixin，在要使用的地方对其进行with，就可以使用mixin中声明的方法或者变量，那么他具体和继承有什么区别呢？

首先，当你的类去with一个mixin/类时，他并不影响你再去继承一个其它的类，如下：

**class A extends T with B, C {}**

其次在上面我们也说了，mixin本身是个打了引号的类，他不能声明构造方法，说明官方不希望我们编写初始化它的代码，我们不能通过初始化获得它的引用，并通过引用在其他的地方调用它的方法；要使用它我们只能通过with的方式，把他混入到某个类上。

这个是因为如果这样使用是不安全的，上面我们说到，mixin中可以调用在其本身未声明的方法，可以通过on的方式带来来了一种扩展mixin本身能力的方式，但是前提是我们混入的类需要继承或者with mixin上on的类型，当我们直接引用而不是通过with的方式，这种调用可以能会使用到未声明的方法，产生运行安全问题;
二是因为因为在底层的实现上不允许这样的使用，文章第二部分会分析到。

with后面是可以添加多个mixin类型的，那么它是一种“多继承”么？我们看下面这个例子。

```
mixin A {
  test() {
    print('A');
  }
}
mixin B {
  test() {
    print('B');
  }
}

class C {
  test() {
    print('C');
  }
}

class D extends C with A, B {}

class E extends C with B, A {}

void main() {
  D d = D();
  d.test();

  E e = E();
  e.test();
}

```
上面的main方法最终print的是什么呢？
**BA**

这是什么情况呢，好像我们第一印象中应该是“CC”才是，这个with带来的效果好像是“远者近也”，什么意思呢？我们知道在继承中，在类的层次结构中方法的父类方法调用实际上调用的谁离类本身父类层级中最近的类中的方法，但是with相反，难道它颠覆了面向对象中继承的实现么，起码表象看来如此，但是实际如此么？

这个mixin到底是什么的变种呢？其实还是离不开继承与实现，mixin本身以及with和on的底层实现都是通过继承以及实现接口的方式实现的，那么这种“远者近也”是怎么产生的呢，在本文下面第二个部分“脱糖”中我们详细描述。

### 二、mixin的脱糖
承接上文，我们知道了mixin的使用，但是对于其中的一些具体细节还是有一些疑惑，接下来我们会具体介绍其底层原理，也即mixin的脱糖过程。

在flutter的编译过程中，dart代码在编译过程中会会被先编译成dill文件，分析这个dill文件我们发现了mixin以及with脱糖后的字节码，下面我们一一介绍。
#### 2.1 mixin“类”
在编译产物中，程序中mixin代码会做如下的转化：
源码：
```
mixin B {
  testB() {}
}
mixin C {
  testC() {}
}
mixin D {
  testD() {}
}
mixin A on B, C, D {
  testA() {
    testB();
    testC();
    testD();
  }
}
```
转化后：

```
abstract class B extends Object {
  abstract testB() {}
}

abstract class C extends Object {
  abstract testC() {}
}

abstract class D extends Object {
  abstract testD() {}
}
//mixin A转化
abstract class _A&B&C extends Object implements B, C{}
abstract class _A&B&C&D extends Object implements _A&B&C ,D {}
abstract class A extends _A&B&C&D {
  abstract testA() {
    testB();
    testC();
    testD();
  }
}
```
这就是为什么当on后面限定了类型，在使用mixin时需要继承或者实现on后面的类型,它们都被当作接口implements。mixin上省略的on子句等效于on Object。

mixin会被转化为抽象类，其on的类型会被以普通接口的方式实现，但是为了字节码的大小问题，我们用抽象类，可以不用实现on的类型中的方法，这也说明了上面我们说为什么我们不能直接使用mixin以及它没有构造方法的问题，因为它的底层实现是不允许的，mixin只有被混入到普通类上，才能建立完整的类结构，下面我们就说下混入到的普通类上的mixin形成的混合类是如何脱糖的，以及怎样建立完整的可使用的类结构的。

#### 2.2 混合类
上面是mixin的脱糖后的代码，那么混入mixin的普通类会被转化为什么样的代码呢？

源码：

```
mixin B {
  testB() {}
}
mixin C {
  testC() {}
}

mixin A on B, C {
  testA() {
    testB();
    testC();
  }
}

class E {}

class F extends E with B, C, A {
    testF(){}
}
```
脱糖后：

```
//类F转化后的代码，上面的同上省略了
abstract class _F&E&B extends E implements B{
    testB() {}
}
abstract class _F&E&B&C extends _F&E&B implements C{
    restC() {}
}
abstract class _F&E&B&C&A extends _F&E&B&C implements A{
    testA() {
        testB();
        testC();
    }
}
class F extends _F&E&B&C&A {
    testF(){}
}
```

从上面我们可以看到，dart把我们的继承以及with形成的混合类给捋直了，还是倒着捋的，这也回答了上面的例子的问题，“远者近也”确实只是表象内容，其内在的实现还是继承和实现那一套。

只是为什么是倒着来的呢，为了保证mixin“类”代码调用不属于它的扩展的代码，也即on的类型的代码，我么需要把on的对象放到类层级上层（也包括extends的对象），那么即使with是正的顺序，和extends的顺序就会分割开来，导致with是正的，但是extends在前，但实际在类层级结构中的上面，就会带来更多的歧义，所以还不如统一倒着来。

从上面也可以看到，在脱糖过程中为了捋顺代码层级结构，会增加许多私有的抽象类，它们主要负责连接mixin，以及承担在implements中方法的实现，其实就是方法原样的拷贝。

在我阅读sdk中使用的一些复杂的mixin使用，它的易读性非常差，因为他本质是继承，所以mixin中可以重写on后面的类或者mixin中方法，也可以通过super调用on中被重写的方法，这进一步增难了代码的阅读以及理解；并且在使用mixin时，形成的继承结构导致，with的对象要按照mixin本身在后，其on的类/mixin在前的顺序排列，其实也增加了维护和迭代的成本。

我本身觉得mixin确实在编写简单的逻辑上上可以减少代码的编写，比接口好用，能像继承一样复用代码，且有种多继承的功能感觉，但这也都是增加了一些理解成本和复杂功能的迭代成本的。

所以我编写这篇文章，希望能帮助你在使用以及在阅读别人的mixin代码时，更加轻松，对其真正的实现也有个整体了解。


