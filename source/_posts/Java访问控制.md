---
title: Java访问控制
tags: Java
date: 2017-08-27 22:26:53
---
（本文是阅读《Java核心技术卷1》4.7和《Java编程思想》第5章有关访问控制部分的读书笔记）
访问权限控制（隐藏具体实现）实际上与“最初的实现并不恰当”有关。

好文不怕千回改。代码也如同文章。写完代码之后过一段时间再审视，往往会发现有更好的方式来处理。这也是重构代码的基础，正是因为不断地审视、重构代码，才会让代码更可读，更容易理解，因此更可维护。
但是代码重构会遇到一个问题，就是客户端程序员希望你的某些代码不要发生变化。所以你在重构的时候，希望自己在改动代码的时候，不会影响到客户端程序员。这也是OOP的一个基本问题：如何把变和不变区别出来。这种情况特别适用于类库，类库的使用者希望类库在更新的时候自己的代码不需要重写，另一方面类库的开发者希望自己的改动不会造成期待之外的影响。

<!--more-->

但是代码重构会遇到一个问题，就是客户端程序员希望你的某些代码不要发生变化。所以你在重构的时候，希望自己在改动代码的时候，不会影响到客户端程序员。

这种情况可以通过约定来解决，比如说类库开发者与使用者达到某种协定，不改动某些部分的代码。但是类库开发者怎么知道该方法或者域究竟有没有被客户端程序员使用。而且当要移除旧的实现，添加新的实现时，类库开发者改动任何的代码都有可能对现有的使用造成影响。因此类库的开发者手脚被束缚，没有办法去修改代码。

访问控制就是为了解决这样的问题，通过设置“权限”，使得类库开发者和使用者可以强制在某种协定下很好地工作。在Java中，具体就是提供了访问权限修饰词，规定哪些方法或者类可用与不可用，并且分为不同的等级：public、protected、包访问权限和private。Java还使用package来解决如何将构件捆绑到一个内聚的类库单元中的问题。

# Packages
* Group classes in a collection called a package
* organizing your work and separating your work from code libraries provided by others
  hierarchical packages
* guarantee the uniqueness of class names
* no relationship between nested packages for compiler
* reversed-dns name reduce name-space collisions。虽然正序的DNS名称也可以避免类名冲突，但是逆序代表的是一种自顶向下的逻辑，更符合包命名的规范，例如：
  * org.mycompany.accounting
  * org.mycompany.graphics
  * 上面的命名代表的是从org包中查看mycompany的包，在mycompany包中查看accounting和graphics的子包

在打包的时候，并不是将所有的class文件合并为单一的class文件，而是合成一个压缩好的，包含了很多class文件的文件包。这可能会造成凌乱和冲突。假如我将所有的.class文件，不分package如何放在同一个目录下，很可能会造成.class文件名字的冲突。因为Java编译单元（Java源文件）的生成文件与Java编译单元名称是相同，而Java编译单元的名称即某一个类名。如果定义了两个名字相同、包名不同的类，.class文件名称却是相同的，将所有.class文件放在一起，有可能会造成名字的冲突。而符合逻辑的解决办法就是将所有相同的.class文件置于同一个目录下，利用操作系统的层次化文件结构，解决了.class文件名称冲突的问题。这种解决方式同时还使得Java解释器可以根据包名和CLASSPATH找到该类所在的目录，从而加载.class文件。Java解释器会将包名的每个句点替换为斜杠或者反斜杠，从CLASSPATH根中产生一个路径名称。

# Class Importation
之所以要导入，实际上就是要提供一个管理名字空间的机制。由于类名可能存在冲突，因此必须控制名称空间并且为每个类创建唯一的标识符。

一个类可以使用所属包下的所有类和其他包的所有公共类。
访问其他包的公共类有两种方法：
每当要使用类时，在类名前加上完整的包名字，例如： java.time.LocalDate today = java.time.LocalDate.now();
使用import语句
import语句使得引用包中的类有更简便的方式，只要用了一次import，以后就无需在类名前加上完整包名。在导入的时候，可以选择导入一个指定的类还是整个包。例如，
import java.util.*;
将导入java.util包下的所有类，然后就可以直接使用包中的类： LocalDate today = LocalDate.now();
也可以导入指定的类
import java.util.LocalDate;
同样然后就可以直接使用该类。

要注意的是使用星号（*）来导入类时，只能导入一个包，不可以导入多个包，例如：
import java.*.*; 或者import java.*; 
在使用包导入的时候，唯一要担心的是导入的多个包中有类名冲突。比如java.util和java.sql包下都有名为Date的类。如果同时导入两个包，在使用Date类的时候就会出现问题，编译器不知道该选择哪个包的Date类。可以指定要使用的Date类：

```Java
import java.util.*;

import java.sql.*;

import java.util.Date;
```

如果两个包的Date类都需要，就必须在使用Date类时在类名前加上完整的包名以区别：
java.util.Date deadline = new java.util.Date();
java.sql.Date today = new java.sql.Date();
区分不同包中的类是编译器的职责，编译后的字节码会在类前加上完整的包名以区分。
Static Imports
使用static import可以导入类的静态方法和静态域，在使用的时候不需指明类，例如

```java
import static java.lang.System.*;

out.println("Hello world");

exit(0);
```

有些程序员可能不太喜欢这导入方式，会让代码不够清晰。
Addition of a Class into a Package
要把类放在某个包中，必须在源码的第一行就写package语句，在声明类之前：
package com.huxuecong.learnjava;
如果不指明package语句，该类就在默认包中，默认包是没有包名的。
要把源码文件放在能够对应上完整包名的子目录中，例如com.huxuecong.learnjava包必须在子目录com/huxuecong/learnjava的目录下。
编译器会根据包的结构在对应的目录中查找源码文件，然后编译，在同一个目录下生成字节码文件。

编译器在编译某个源码文件的时候并不会检查当前的目录结构，例如
```java
package com.mycompany;

public class PackageTest {
    public static void main(String[] args) {
        System.out.println("hello world");
    }
}
```
使用javac命令编译没有问题，即使该文件不在com/mycompany目录下，也会编译通过，能够生成class文件：

但是程序无法运行，因为VM找不到对应的类。


# Package Scope
* public访问控制符表示可以被任何的类使用
* private访问控制符表示仅可以被定义的类内部使用
* 默认访问控制符表示可以被同一个包中的任何类使用。包访问权限将包内所有的相关的类都聚合起来，彼此之间可以轻松地相互作用。注意必须是同一个包中，不如hxc.learningjava包下面有一个类A，hxc.learningjava.subpackage包下有另外一个包访问权限的类B，那么A是无法使用类B，因为hxc.learningjava和hxc.learningjava.subpackage不是同一个包，尽管看起来两个包存在某种层级关系。
  protected访问控制符将访问权限赋予派生类同一个包中。
  默认上来说，包并不是一个封闭的实体，任何人都可以在包中添加代码，所以可能会有别有用心或者大意的程序员就会在包范围内直接修改变量。例如早期的Java存在了某些方式可以添加类到java.awt包中，只需声明自定义的类的包是java.awt，并且把该类文件放在class path中的某个java/awt目录下，就可以获得对java.awt包的访问权限。后面JDK就直接明确不允许用户自定义类的包名是以java.开头。
  要使得包成为一个封闭的实体，不让别人随意添加代码，将包“封印”起来（seal package)，可以生成一个JAR文件来保护包。

# 类路径
从上面已经看出，class文件必须存储在文件系统上对应的子目录中，类文件的目录路径必须与包名对得上。
class文件除了存储在文件系统上，还能以Jar(Java archive)文件的方式存储。Jar文件包含了压缩好的多个class文件和对应的子目录，能减少空间，改进性能。在使用第三方库的时候，一般都需要加入一个或多个Jar文件。JDK本身也提供了Jar文件，例如jre/lib/rt.jar就包含了很多class文件。
Jar文件使用来ZIP格式来组织文件和子目录。

类路径就是一个包含了所有类文件的抽象路径，实际上是这些类文件所在路径的集合。(class path is the collection of all locations that can contain class files)
如果要在不同的程序中都能找到或加载类文件，必须把这些类文件所在的路径或者Jar文件加进class path中，例如在unix中的classpath：/home/user/classdir:.:/home/user/archives/archive.jar
该class path表示home/user/classdir下的所有类文件，当前目录（.)，/home/user/archives/archive.jar文件中包含的所有类文件都加载到了类路径中。

运行时库文件(rt.jar和其他在jre/lib、jre/lib/ext下的jar包）都会自动加载进来，不必明显地指定。

以/home/user/classdir:.:/home/user/archives/archive.jar类路径为例，展示vm寻找类文件的顺序。
假设要加载com.huxuecong.learnjava.Employee类，VM会按照以下顺序加载：

1. 在jre/lib和jre/lib/ext目录下寻找
2. 查找/home/user/classdir/com/huxuecong/learnjava/Employee.class
3. 以当前目录为起点查找com/huxuecong/learnjava/Employee.java
4. 在/home/user/archives/archive.jar内部查找com/huxuecog/learnjava/Employee.class

但编译器就比vm加载的时候要做了更多事情。如果指定类的时候并没有指明所属的包，编译器需要在每个import导入的包中去寻找有没有该类。例如：

```java
import java.util.*;

import com.huxuecong.learnjava.*;

```

源码使用了一个名为Employee的类，编译器会在class path中查找java.lang.Employee、java.util.Employee、com.huxuecong.learnjava.Employee和当前包的Employee，如果找到一个以上的匹配类，编译器就会报错。由于类名必须唯一，import语句的顺序并不会有任何影响。
除此之外，编译器还会看源码文件是否比class文件更新，如果是的话，就要重新编译该源码文件。

在编译的时候最好添加-classpath(or -cp)选项来指定类路径，而不是设置CLASSPATH环境变量或其他办法。

# 总结
包用于解决Java的名称冲突。

访问权限控制使得用户不能访问不应该访问的部分，让类库使用者可以更改类的内部工作方式而不必担心会对客户端程序员造成影响，有了犯错的可能性。

访问控制实际上是类库设计者与使用者之间的关系说明，这种关系是一种通信方式。