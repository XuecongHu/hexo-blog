---
title: 浅谈Java多态
date: 2017-10-11 14:44:03
tags: Java
---

多态，英语Polymorphism，由希腊语的两个单词polys（意为many, much)和morphē（意为form, shape)组成。从英文单词也可知道polymorphism的意思是“有着多样的形态”。多态表示的是同一个事物具有的不同形态。

<!--more-->

# 引子
在日常使用的语言中，我们随时使用到多态，也就是一字多义。举“洗”（wash)为例，“洗”可以表达多种不同含义的“洗”。洗衣服、洗澡、洗车中的”洗“实际上都不一样，都是不尽相同的动作。但是我们无需专门为了这些情景中的”洗“专门定义一个字或词。例如不必为”洗车“的”洗“而专门造一个字。

通过消除文字之间的耦合，极大地减少了语言的文字数量，提高了语言的简洁性、可读性。消除文字之间的耦合是指自然语言中的文字可以单独拿出来看待，比如”洗“这个字，单独拿出来看我们也知道是什么意思，而不是要从”洗车“整个词理解才能知道”洗“是什么意思。如果字与字之间的耦合度很高，只要我改变了一整段话的某一个字，就有可能要改掉整段话中的所有字了，会牵一发而动全身。比如说”我在室外洗自行车“。如果“洗”和“车”的耦合度很高，例如为不同的车“洗”都有专门的字，有为单车的“洗”，摩托车的“洗”，轿车的“洗”。这样只要我把”自行车“改为”轿车“，就要把自行车的“洗”换为轿车的“洗”了。我们希望不管是洗什么车，都是同一个洗，甚至是不管是洗什么物体，都是同一个“洗”。

而在面向对象的程序设计中，多态就是指同一个接口在不同的导出类中具有不同的行为表现方式，其意义与自然语言中的多态十分相似。

# 继承与多态
在OOP中，没有继承就没有多态(严格上这里的多态是指动态多态)。
要理解多态，必须结合面向对象中的继承来看，它并不是一个可以单独隔离来看的概念。

继承在程序设计中最主要并不是为了复用父类的代码，组合也可以完成代码的复用，而继承更多是表现出一种类与类之间的关系，这种关系就是子类是父类的一种类型，也就是经常提到的"is-a"关系。而这种关系正是多态存在的前提。
由于导出类复用了父类的接口（具有相同的方法），同一个消息可以发送给这些不同的导出类，使得相同的接口具有不同的行为表现。

借用《Java编程思想》的简单例子
```java
class Instrument {
	public void play(Note n) {
		System.out.println("Instrument.play()");
	}
}

class Wind extends Instrument {
	public void play(Note n) {
		System.out.println("Wind()");
	}
}

class Violin extends Instrument {
	public void play(Note n) {
		System.out.println("Violin()");
	}
}

public class Music {
	public static void tune(Instrument i) {
		i.play(Note.MIDDLE_C);
	}
	
	public static void main(String[] args) {
		Wind flute = new Wind();
		tune(flute);
		
		Violin violin = new Violin();
		tune(violin);
	}
}
```
在上面的例子中，Wind类和Violin类是Instrument类的导出类，有其独特的play方法实现。Music.tune()方法中调用的是Instrument类的play方法。只需要给tune方法传入Instrument类或其导出类，Java就会根据Instrument类的实际类型使用对应的play方法。同一个play方法，根据对象的类型具有不同的实现。
综合来看，在OOP中，多态的“同一个东西”就是指有同一个父类的同一个方法，而“不同的形态”是说这些子类的方法可以有自己不同的实现。
继承是多态的前提，并且是其实现的条件。

# 类型解耦
程序设计语言中多态的作用与自然语言的非常相似。
多态的本质在于消除了类型之间的耦合。简而言之，即一个类的代码改变尽量少影响另外一个类。如同上文阐述的自然语言中字与字之间的解耦。不希望一个类的改变导致另外一个类的改变，从而使得整个代码都大幅度的的改动。
使用在上一节中的代码例子，就是希望Music类中的tune方法是一个不受具体乐器而改变的方法，不想为了每一种具体的乐器都特地写一个tune方法，如tune(Violin)，tune(Wind)等等，只需要一个tune(Instrument)即可。
通过类型的解耦，使得改变的事物与不变的事物区别开来，不管新增还是减少乐器，都是使用Music.tune方法。
而之所以可以解耦，原因在于将what与how区别出来。Music.tune表示的是what，仅仅是一个抽象的概念，正如“洗”本身是一个抽象的“洗”。而具体的how，则由更细节的子类来表达，正如“洗车”中的“洗”。
通过多态，程序将变得更可扩展，代码也变得更加的简练。

# 后期绑定
在程序设计
多态是如何做到区别不同的子类型，调用正确的方法呢？
```java
public static void tune(Instrument i) {
	i.play(Note.MIDDLE_C);
}
```
在tune方法中，它只接受一个Instrument类的引用。但是实际上编译器如何知道这个Instrument引用指向的具体对象呢？是指向Violin对象还是Wind对象呢？实际上Java编译器无法得知，只能是在运行时得知。
实际上这个过程称为绑定，也就是将方法和一个方法主体（对象）关联起来。多态的实现依赖于后期绑定，即在运行时根据对象的类型进行绑定。后期绑定的“后期”与“前期”是一个相对的概念，区别在于是运行前还是运行时。

# 并非所有的都是多态
并非所有的东西都能是多态。正如在自然语言中，并非所有的字都会有多义。例如“人”，人的本意只能表达人类这种动物，并不会用来表示其他的动物或者事物，除非是后来的引申义。而往往谓词，可以有多义，如上文提及的“洗”，是一个动词。

在程序设计语言中，多态当然也有限制——多态只能是针对类的非static和final方法。换句话说，就是类的static和final方法以及类的域不能多态。private方法实际上是final方法，因此private方法也不能实现多态。
类域的多态并不是“多态”。域表示的是类的状态数据，与自然语言中的体词类似，状态数据不可能有多个，例如boolean类型的成员变量只能是true或者false。如果子类的域和父类的域值发生了改变，那不是多义，而是值发生了变化。
final方法表示的是不可覆写，自然就无法做到每个子类有不同的实现了。
static方法表示的该方法属于类，而非对象。多态的根据具体子类调用不同的方法变得毫无意义，因为向上转型后调用的总会是基类的方法。例如：
```java
class Super {
	public static staticMethod() {
		System.out.println("Super static method");
	}
}

class Sub extends Super {
	public static staticMethod() {
		System.out.println("Sub static method");
	}
}

public class StaticMethodPolymorphismTest {
	public static void main(String[] args) {
		Super super = new Sub();
		super.staticMethod();
	}
}
```
这段代码的输出例子是"Super Static method"而不是"Sub static method"。原因很简单，static方法是属于类的，所以调用staticMethod方法肯定是调用Super类，而非Sub类。顺带一提，在实践中，不建议使用对象实例来调用static方法，而是直接使用类来调用静态方法，可以减少混淆，如：
```java
Super.staticMethod();
```


# 构造器中的多态陷阱
值得一提的是，如果在多态中使用多态，很可能会造成一些意想不到的问题。这是因为在构造器初始化的时候，导出类的数据还没有构造完毕，如果多态的方法使用了导出类的数据，会造成意想不到的问题。
借用《Java编程思想》的简单例子。
```
class Glyph {
    void draw() {
        System.out.println("Glyph.draw");
    }

    public Glyph() {
        System.out.println("Glyph before draw()");
        draw();
        System.out.println("Glyph after draw()");
    }
}

class RoundGlyph extends Glyph {
    private int radius = 1;

    void draw() {
        System.out.println("RoundGlyph.draw(), radiu = " + radius);
    }

    public RoundGlyph(int radius) {
        this.radius = radius;
        System.out.println("RoundGlyph.RoundGlyph(), radius = " + radius);
    }
}

public class PolyConstructors {
    public static void main(String[] args) {
        new RoundGlyph(5);
    }
}
```
输出结果是：
Glyph before draw()
RoundGlyph.draw(), radiu = 0
Glyph after draw()
RoundGlyph.RoundGlyph(), radius = 5

在调用RoundGlyph构造器时，会首先隐式地调用Glyph构造器。在Glyph方法中会调用draw方法，而由于后期绑定，Java会调用RoundGlyph的draw方法。RoundGlyph的draw方法会使用到radius成员变量，而由于此时radius成员变量值只是初始化的零值，所以就打印出来0了。
所以多态并不建议在构造器中使用，我们甚至建议在构造器中尽可能简单地初始化对象，唯一安全使用的就是final方法。

# 结束
从根本上来说，OOP中的多态消除了类型之间的耦合，使得“变”与“不变”区别开来，提高了程序的可扩展性，使得代码更可读和更可维护，是面向对象中的基本特性。