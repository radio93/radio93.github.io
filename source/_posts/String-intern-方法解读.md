---
title: String.intern()方法解读
date: 2020-11-18 16:58:56
tags:
- java
---

首先因为jdk1.7对该方法进行了改动，所以我们只分析1.7的现象。

在分析这个方法前，我们先来看看HotSpot的**字符串常量池**

他的存在，是为了减小对象的创建，我们不要把它想的过于复杂，比如我们在程序中写了一个`String a = "abc";`那么这个abc就会存储在字符串常量池中，下次再遇到这个，就不会再创建一个abc了，而是直接去字符串常量池中去找，所以`String a = "abc";`和`String b = "abc";`这两个变量，`a==b`是`true`的。因为他们都指向同一个字符串常量池。（字符串常量池1.7之前在方法区，1.7之后被挪到了堆，方法区也被挪到了元空间）。

简单了解了一下字符串常量池后，我们再来看一个经典的面试题

### String s1 = new String("abc");这句话创建了几个字符串对象？

答案是 1个或者2个，为啥呢，如果**字符串常量池**中已存在字符串常量（或者引用，后面再说）“abc”，则只会在**堆**空间创建一个字符串常量“abc”。如果池中没有字符串常量“abc”，那么它将首先在池中创建，然后在堆空间中创建，因此将创建总共 2 个字符串对象。

比如下面这个示例

```java
@Test
public void test1() {
    String s1 = new String("abc");// 堆内存的地址值
    String s2 = "abc";
    System.out.println(s1 == s2);// 输出 false,因为一个是堆内存，一个是常量池的内存，故两者是不同的。
}
```

这个如果比较好懂，我们就再接着看一个稍微复杂一点的

```java
 @Test
public void test2() {
    String str1 = "str";
    String str2 = "ing";

    String str3 = "str" + "ing";//常量池中的对象
    String str4 = str1 + str2; //在堆上创建的新的对象
    String str5 = "string";//常量池中的对象
    System.out.println(str3 == str4);//false
    System.out.println(str3 == str5);//true
    System.out.println(str4 == str5);//false
}
```

我们来解读一下这个test，第一行str1和第二行str2 毫无疑问，在字符串常量池中产生，第三行和第四行有什么区别呢，第三行 两个字符串常量相加，依然在字符串常量池，第四行了两个变量相加，那么就会在堆中产生了。这里我们做一个说明：

>String str3 = "str" + "ing"; 在编译期间，这种拼接会被优化，编译器直接帮你拼好，所以str3相当于直接赋值为“string”
>
>而相加的过程中一旦出现了对象，就不会做优化，因为这是一个对象，内存不是确定的，没有写死，无法实现优化。而且在相加的过程中，java会先new出一个StringBuilder，然后调用append()方法来将+号两遍的字符串拼接起来，然后toString()之后返回给=号左边的变量，也就是说，最后得到的是一个new出来的字符串

这样一来，结果就很好理解了。

我们再看一个例子

```java
@Test
public void intern4() {
    String a="a";
    String ab =a+"b";
    System.out.println(ab == "ab");//false
}
```

我们分析一下，ab是堆，“ab”是常量池，所以false很合理，如果我们吧a变成final的，那么结果就是true了。

```java
@Test
public void intern4() {
    final String a="a";
    String ab1 =a+"b";
    System.out.println(ab1 == "ab");//true
}
```

这就是我们刚才所说的优化了，因为a是final不可变的，所以编译的时候，a+"b"="a"+"b"="ab";

为啥说intern要先举一下上面的例子呢，因为String的intern方法就是针对这个来设计的优化。

**intern是一个本地方法，它的作用是如果字符串常量池中已经包含一个等于此String对象的字符串，则返回代表池中这个字符串的String对象的引用；否则，会将此String对象包含的字符串添加到常量池中，并且返回此String对象的引用。**

jdk1.7后，如果对象只存在堆中，会拷贝对象的引用到常量池中

了解了基本定义之后，我们再来看下面这个例子

```java
@Test
public void test3(){
    String s1 = new String("计算机");
    String s2 = s1.intern();
    String s3 = "计算机";
    System.out.println(s1 == s2);//false，因为一个是堆内存中的String对象,一个是常量池中的 String 对象，
    System.out.println(s3 == s2);//true，因为两个都是常量池中的String对象
}
```

第一行会在字符串常量池和堆中分别创建两个对象，s1指向堆中的引用，没有问题，s1.intern()指向常量池，此时常量池中已经有了“计算机”，所以s2此时就等于常量池中的引用，s3直接指向常量池。

看了这个简单的例子后，我们稍微看个复杂点的

```java
@Test
public void intern3() {
    String str1 = new String("abc") + new String("def");
    System.out.println(str1.intern() == str1);//true
    System.out.println(str1 == "abcdef");//true
}
```

这个时候我们可能会有疑问str1 == "abcdef" 怎么可能为true呢，一个是堆，一个是常量池。我们一步步看，第一句话，我们分析一下，创建了几个对象。

答案是5个，常量池中有两个，分别是abc和def，堆中有三个，abc，def和abcdef，这个没啥疑问吧，然后第二行，调用了str1.intern()。这个时候发生了什么，它去常量池中找abcdef，发现没有，做了个什么操作呢，把堆中的引用存到了常量池中，（1.7之前是拷贝值，1.7之后是拷贝引用，因为都在堆里面了，为了优化，没必要存两份了）。这个时候，常量池中存在了abcdef，这个abcdef就是str1中堆的值。所以 这个时候我们再来比较str1和abcdef，其实都是指的堆。我们把这个例子加一句话

```java
@Test
public void intern3() {
    String str2 = "abcdef";
    String str1 = new String("abc") + new String("def");
    System.out.println(str1.intern() == str1);//false
    System.out.println(str1 == "abcdef");//false
}
```

我们只在第一行加一句话，其他的都不动。结果却是翻天覆地的。如果你对intern有了一个差不多的了解后，我们再来分析一下，第一句话，在常量池中创建了一个abcdef，第二句话效果不变，我们再来看第三句话，此时str1.intern()去常量池中找，发现有了一个常量了，这个时候 它就指向了常量池中的abcdef了，所以结果不难理解了。

了解了这个之后，我们再来看一个有趣的例子

```java
@Test
public void intern() {
    String str1 = new StringBuilder("计算机").append("软件").toString();
    String str2 = new StringBuilder("ja").append("va").toString();

    System.out.println(str1.intern() == str1);
    System.out.println(str2.intern() == str2);
}
```

根据我们刚才的推测，这俩应该都是true，因为都是指向的堆中的引用。但是实际上，结果是：

```java
true
false
```

为啥第二个是false呢，这是因为“java”是个关键字， 这个字符串在执行StringBuilder.toString()之前就已经出现过了，字符串常量池中已经有它的引用，不符合intern()方法要求“首次遇到”的原则，“计算机软件”这个字符串则是首次出现的，因此结果返回true。

最后，我们再看一个例子

```java
@Test
public void intern2() {
    String a="a";
    String b="b";
    String ab1 = new String(a+b);
    String ab2 = new String(a+b);
    System.out.println(ab1.intern() == ab1);//true
    System.out.println(ab2.intern() == ab2);//false
    System.out.println(ab1.intern() == ab2.intern());//true
}
```

这个例子再次给我们证明了，如果常量池中不存在，返回对象的引用，所以第一个是true，第二个ab2.intern()的时候，发现已经存在了ab1的引用，所以指向了ab1，这两行，谁在前面，谁是true。



