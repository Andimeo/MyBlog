title: Java Generics
date: 2015-05-12 21:39:00
categories: Java
tags: 
- 泛型
- Generics
- Wildcard
---

今天看了下Java的官方教程中关于泛型的部分。泛型引起我的注意是因为微博上一篇比较`List<?>`和`List<Object>`的文章。最近看Lucene的代码，其中Util部分大量的使用了泛型，今天刚好浮生修得一日闲便全面的学习一下Java的泛型。其中很多内容都与直觉相符，所以我觉得不值得通篇翻译，这里仅就我觉得有趣也反直觉的地方做一个总结。

[The Java™ Tutorials Lesson: Generics](http://docs.oracle.com/javase/tutorial/java/generics/index.html)

## 命名规则
C++和Java中都学习过泛型，可是从来觉得写T就跟写for循环中的i一样，也就是约定俗成，谁知这里还是有一些规则的。
+ E - Element (used extensively by the Java Collections Framework)
+ K - Key
+ N - Number
+ T - Type
+ V - Value
+ S, U, V etc. - 2nd, 3rd, 4th types

<!-- more -->

## The Diamond
在Java 7及以后的版本中，当调用泛型类的构造函数时，可以省去类型参数，只使用一组空的尖括号，只要在编译期类型可以被编译器确定或者推导出来。这对尖括号，就叫作diamond。比如我们可以像如下这样声明一个链表：

```
List<String> list = new ArrayList<>();
```

diamond在英文中可以指扑克牌中的方片，两个尖括号放在一起<>，的确像一个方片。

## Multiple Bounds
`List<? extends A>` 和 `List<? super A>`代表类型为A的子类和A的父类的链表类型。这里的语法称为upper bound和lower bound。这些都是非常基础的知识了。但是类型参数可以有多个bounds，这样的写法并不常见。如:

```
<T extends A & B & C>
```

需要注意的是，如果A、B、C中有一个为类，其余为接口的话，类必须写到第一个的位置，否则会在编译时报错。

## Unbounded Wildcard
unbounded wildcard在两种情形下很有用：
+ 当你在写一个只需要类Object提供的功能就能够完成的方法时
+ 当使用的泛型类中的方法并不依赖于泛型参数时。比如`List.clear`或`List.size`。实际上，我们经常使用`Class<?>`，因为`Class<T>`
中的大部分方法都和T无关。

考虑下面的方法：

```
public static void printList(List<Object> list) {
    for (Object elem : list)
        System.out.println(elem + " ");
    System.out.println();
}
```

printList的目的是打印任意类型的列表，但是上面的函数却做不到。它只能打印Object的List，无法打印`List<Integer>`，`List<String>`或`List<Double>`，因为它们都不是`List<Object>`的子类。为了写一个泛型的printList，需要使用`List<?>`：

```
public static void printList(List<?> list) {
    for (Object elem: list)
        System.out.print(elem + " ");
    System.out.println();
}
```

因为对任何具体类型`A`，`List<A>`是`List<?>`的子类。

需要特别注意的是，`List<Object>`和`List<?>`是不一样的。你可以往`List<Object>`插入`Object`和其他Object的子类。但是你只能往`List<?>`中插入`null`。

## Lower Bounded Wildcard
你可以为一个Wildcard指定Upper Bound或者Lower Bound，但是不能同时指定。

## Wildcards and Subtyping
The common parent is List&lt;?&gt;  | A hierarchy of several generic List class declarations.
-----|------
{% img /img/generics-listParent.jpg 500 %}| {% img /img/generics-wildcardSubtyping.jpg 500 %}  
  
## Guidelines for Wildcard Use
为了讨论的方便，我们假设一个变量提供下面的两种功能：
+ "In"变量，为函数提供输入数据
+ "Out"变量，为函数提供输出

Wildcard Guidelines：
+ "In"变量用Upper Bounded Wildcard定义，使用extends关键字
+ "Out"变量用Lower Bounded Wildcard定义，使用super关键字
+ 当"In"变量可以用Object中定义的方法访问时，使用Unbounded Wildcard
+ 当变量同时作为"In"和"Out"时，不要使用Wildcard

这份Guideline并不适用于函数的返回值类型，应该避免使用wildcard作为函数的返回值类型。

## 泛型的限制
** 不能用基本类型实例化泛型 **

**不能创建类型参数的实例**

你不能创建类型参数的实例，但是可以使用反射创建。

**不能创建包含类型参数的static成员**

因为所有类的实例都共同拥有static成员，但是可以创建和类的类型参数不同的泛型static函数

**不能创建参数类型的数组**

```
Object[] stringLists = new List<String>[];  // compiler error, but pretend it's allowed
stringLists[0] = new ArrayList<String>();   // OK
stringLists[1] = new ArrayList<Integer>();  // An ArrayStoreException should be thrown,
                                            // but the runtime can't detect it.
```

如果允许创建类型参数的数组，上面的代码将会抛出`ArrayStoreException`

**不能创建，catch或throw参数类型的对象**

泛型类不能直接或间接继承`Throwable`。

```
// Extends Throwable indirectly
class MathException<T> extends Exception { /** ... **/ }    // compile-time error

// Extends Throwable directly
class QueueFullException<T> extends Throwable { /** ... **/ // compile-time error
```

一个方法不能catch类型参数的实例

```
public static <T extends Exception, J> void execute(List<J> jobs) {
    try {
        for (J job : jobs)
            // ...
    } catch (T e) {   // compile-time error
        // ...
    }
}
```

然而，可以在`throws`子句中使用类型参数

```
class Parser<T extends Exception> {
    public void parse(File file) throws T {     // OK
        // ...
    }
}
```

**不能拥有在Type Erasure之后签名一样的重载函数**

```
public class Example {
    public void print(Set<String> strSet) { }
    public void print(Set<Integer> intSet) { }
}
```
