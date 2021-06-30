---
title: 策略模式（上）
date: 2021-01-03 20:42:35
tags: 
	- strategy
	- 策略模式
categories: 
	- DesignPatterns	
	- 设计模式
---

> 今年开始深入学习设计模式，并把所思所想记录下来，以备查阅。因为我以《Head First 设计模式》一书为导向，所以第一个学习的模式是`策略模式`。本篇为策略模式的上篇，我以**传统的严格意义上的面向对象语言 **`Java`为例来说明此模式；我会在下一篇用**非严格意义上的OO语言** `Go`基于同样的例子进行说明。

## 有一个游戏

​	假设我们在设计一款鸭子游戏。玩家可以通过按钮选择任意一款鸭子，使得对应的鸭子可以在屏幕上展现，并且做相应的动作。为此我们设计了如下的高层代码：

```java
public class Game {
    Duck duck;
    public Game(String type){
        if (type.equals("picnic")) {
            duck = new MallardDuck();
        } else if (type.equals("hunting")) {
            duck = new DecoyDuck();
        } else if(type.equals("inBathTub")) {
            duck = new RubberDuck();
        }
    }

    public play() {
        duck.display();
        duck.swim();
    }
}
```

​	我们希望`Duck`是一个超类，并且在这里起到多态的作用。通过`type`输入参数来模拟用户通过按钮选择鸭子这一行为，并且在构造函数中初始化相应的鸭子实例，以便于后续调用`play`函数来在屏幕上展示。

​	我们假定**每种鸭子都有独一无二的外观，并且每种鸭子都有相同的游泳方式**。记住我们的这两个假设，因为我们马上要据此设计超类。

## 超类

​	我们之前假设了鸭子的两种特征：

1. 每种鸭子都有独一无二的外观
2. 每种鸭子的游泳方式都相同

​    我们把`Duck`设计为一个抽象类，因为外观是独一无二的，所以`display`设计为抽象方法，具体的外观由每一个具体的子类去实现；`swim`的行为每种鸭子都一样，所以我们在抽象类里将其实现，以达到所有子类共享的目的。

​	但是，我们忽然又意识到鸭子还有`quack`叫的特征，但是目前又不确定quack是不是有变化，所以暂时将其也在抽象类里实现。我们的超类代码大概如下：

```java
public abstract class Duck {

    public abstract void display();

    public void quack() {
        System.out.println("quack quack quack!");
    }

    public void swim() {
        System.out.println("All ducks float, even decoys!");
    }

}
```

我们来总结一下如此设计的目的：

1. 抽象类`Duck`用于实现多态
2. 抽象方法`display`用于统一接口，让子类分别实现
3. `quack`和`swim`分别用于继承，以达到代码复用的目的

我们用一张图来表示这种继承关系：

![duck](https://qiniu.liupzmin.com/duck)

我们的设计是利用继承达到多态和代码复用的目的。绿头鸭和红头鸭分别是具体的实现，它们继承了`quack`和`swim`代码，又各自实现了`display`的方法。目前来看，我们的设计还算完美，如果不是一只`橡皮鸭子`出现的话。

## 现在要让鸭子飞

​	现在游戏上线了一种新的鸭子—**橡皮鸭**，橡皮鸭与之前的鸭子不同的是，它不会“呱呱”叫，只会“吱吱”叫。我们所有种类的鸭子都是`继承`自超类`Duck`，所以我们可以在橡皮鸭的子类中覆盖`Duck`的`quack`方法，向下面这样：

```java
public class RubberDuck extends Duck{

    public void display(){
        System.out.println("I am a rubber duck!");
    }

    public void quack() {
        System.out.println("squeak squeak squeak!");
    }

}
```

​	因为橡皮鸭子不会呱呱叫，所以我们覆写了`quack`方法。子类重写虽然能解决当前的问题，但也势必会引入新的问题，我们接着往下看。

​	现在有一个新的需求要加入到游戏当中去，那就是我们需要让鸭子展示`飞行`的动作。基于我们目前的设计，很容易想到的是：在`Duck`中加入`fly()`方法：

![fly](https://qiniu.liupzmin.com/duck-fly)

​	那么，问题来了，我们是将`fly()`设计成抽象方法呢，还是将其在超类里实现呢？在超类里实现就会出现橡皮鸭子会飞的情况，我们自然会想到子类重写，就像重写`quack`函数一样。但是，如果以后又加入了其它的不会飞也不会叫的鸭子怎么办？比如诱饵鸭是木头鸭，不会飞也不会叫；橡皮鸭不会飞但是会叫。长此以往，我们的代码会充斥着各种子类、各种重写的方法，这显然是有问题的。换句话说，我们之前的重写`quack`是不明智之举。

​	其实，问题的本质是**继承不适用于目前的场景。**

​	使用继承的问题：

- 如果是抽象的方法，代码在多个子类中重复，即代码无法复用。试想一下：一部分鸭子的飞行代码相同，而又有很多种类的鸭子飞行行为各有特色。
- 每次新增行为都要修改抽象类和子类，包括以后面临的修改，这都违反了开闭原则，且不可能预知全部的行为。
- 如果超类实现子类继承，复写子类方法同样无法达到代码复用，因为可能有多个种类的鸭子飞行行为相同，但又不同于超类里的实现。如果一旦涉及到修改，势必会带来维护上的噩梦。
- 改变会牵一发而动全身，造成其他鸭子不想要的改变。

## 接口能解决问题么

​	`Java`为我们提供了`interface`来实现多态。既然不能使用继承和重写来实现对应的功能，那么我们很容易想到用定义接口的方式让子类分别实现`quack`和`fly`接口：

![duck interface](https://qiniu.liupzmin.com/duck-interface)

像图中那样将`quack`和`fly`作为单独的接口去实现。这可以解决部分问题，也就是不会有鸭子和行为不符合的情况，但依然面临严峻的问题：

- 行为只是可能会发生变化的话，每个子类都实现接口，会造成代码无法复用，继而也会带来维护上的噩梦。
- 如果我想在运行中随时改变飞行的动作呢？继承超类和子类实现单独的接口都无法有效的解决问题。

其实，单独的接口和抽象方法所面临的问题是一致的，即代码复用和维护变更的问题。试想一下，如果有48个Duck子类的飞行行为相同，代码实现就会有48份，而你恰巧在某个时刻需要修改这个行为...

这时，你肯定期待着`设计模式`能骑着白马来解救你！

## 分开变化和不变的部分

幸运的是，有一个设计原则，恰好适用于此状况。

> 找出应用中可能需要变化之处，把它们独立出来，不要和那些不需要变化的代码混在一起。

这样的概念很简单，几乎是每个设计模式背后的精神所在。所有的模式都提供了一套方法让**“系统中的某部分改变不会影响其它部分”**。

我们试分析一下，关于变和不变存在以下三种情况：

1. 一定不变的
2. 一定会变的
3. 可能会变的

对于**一定会变的**，我们很容易用**抽象方法或者接口**来解决（`display`）。对于**一定不变**的（`swim`），我们就用**继承**，把实现放在超类里。

棘手的是可能会变化的部分，比如这里的`fly`和`quack`，这正是让我们左支右绌、进退维谷的根源。根据上面提到的原则，我们应该把这两个部分从`Duck`类中取出来，建立一组新类来代表每个行为。

![separate behavior](https://qiniu.liupzmin.com/separate-behavior)

我们该如何实现新的行为类呢？我们希望每种Duck在使用行为类的时候具有弹性和灵活性，比如可以动态改变，也就是说我们可以在Duck类中增加设定行为的方法，这样就可以在运行时动态的改变鸭子的行为了。

那么，有什么设计原则可以指导我们么？

> 针对接口编程，而不是针对实现编程。

我们理应在Duck类中使用接口，而不是具体的实现。也就是说Duck类不负责实现`fly`和`quack`，具体的实现交给新的类去实现`FlyBehavior`和`QuackBehavior`接口。

Duck类应该只针对接口编程，**“针对接口编程”真正的意思是“针对超类型编程”。**这里所谓的“接口”有多个含义，关键在于使用`多态`。利用多态，程序可以针对超类型编程，执行时会根据实际状况执行到真正的行为，不会被绑死在超类型的行为上。“针对超类型编程”这句话，可以明确的说成“变量的声明应该是超类型，通常是一个抽象类或者是一个接口。如此，只要是具体实现次超类型的类所产生的对象，都可以指定给这个变量。这也意味着，声明类时不用理会以后执行时的真正对象类型！”

**也就是说，高层代码应该针对行为编程！**

## 实现行为

![implement duck's behavior](https://qiniu.liupzmin.com/impl-duck-behavior)

​	我们设计了两个接口**FlyBehavior**和**QuackBehavior**，还有它们对应的类，负责实现具体的行为。这样的设计，可以让飞行和呱呱叫的动作被其它的对象复用，因为这些行为已经与鸭子类无关了。而我们可以新增一些行为，不会影响到既有的行为类，也不会影响到使用飞行行为的鸭子类。这么一来，有了继承的代码复用好处，却没有了继承所带来的包袱。

用代码实现一下：

```java
public interface FlyBehavior {
	public void fly();
}

public class FlyWithWings implements FlyBehavior {
	public void fly() {
		System.out.println("I'm flying!!");
	}
}

public class FlyNoWay implements FlyBehavior {
	public void fly() {
		System.out.println("I can't fly");
	}
}

public class FlyRocketPowered implements FlyBehavior {
	public void fly() {
		System.out.println("I'm flying with a rocket");
	}
}

public interface QuackBehavior {
	public void quack();
}

public class Quack implements QuackBehavior {
	public void quack() {
		System.out.println("Quack");
	}
}

public class Squeak implements QuackBehavior {
	public void quack() {
		System.out.println("Squeak");
	}
}

public class MuteQuack implements QuackBehavior {
	public void quack() {
		System.out.println("<< Silence >>");
	}
}
```



把行为整合进抽象类duck

![new duck](https://qiniu.liupzmin.com/new-duck.png)



上图是新的鸭子类，实现代码如下：

```java
public abstract class Duck {

    FlyBehavior flyBehavior;
    QuackBehavior quackBehavior;

    public abstract void display();

    public void performQuack() {
        flyBehavior.quack();
    }

    public void performFly() {
        flyBehavior.fly();
    }

    public void swim() {
        System.out.println("All ducks float, even decoys!");
    }

}
```



我们重新实现一下绿头鸭：

```java
public class MallardDuck extends Duck {

	public MallardDuck() {

		quackBehavior = new Quack();
		flyBehavior = new FlyWithWings();

	}

	public void display() {
		System.out.println("I'm a real Mallard duck");
	}
}
```

​	我们的绿头鸭代码里有针对实现的代码，就是实例化行为的两行，这貌似违背了了我们之前所述，**针对接口编程，而不是针对实现**；其实，这里可以改为工厂模式，使得我们的代码彻底的面向接口。但工厂模式不是我们本次的重点，关于这一点我会稍后再略作解释，让我们继续完善我们的代码吧！

之前我们说要把鸭子的行为设计成可以动态改变，现在貌似还差点火候。那么，让我们在Duck类中再加入两个设定行为的方法吧：

```java
public void setFlyBehavior(FlyBehavior fb) {
		flyBehavior = fb;
}

public void setQuackBehavior(QuackBehavior qb) {
		quackBehavior = qb;
}
```

​	从此以后，我们可以随时调用这两个方法来改变鸭子的行为。来做个模拟吧：

```java
public class MiniDuckSimulator {
 
	public static void main(String[] args) {
 
		Duck mallard = new MallardDuck();
		mallard.performQuack();
		mallard.performFly();
   
		Duck model = new ModelDuck();
		model.performFly();
		model.setFlyBehavior(new FlyRocketPowered());
		model.performFly();

	}
}
```

output：

```shell
Quack
I'm flying!!
I can't fly
I'm flying with a rocket!
```

​	可见，在运行时想改变鸭子的行为，只需要调用`setter`方法即可。我们通过把可能变化的行为抽象为接口，使用单独的类去实现它。这样即解决了代码复用的问题，又使得维护变得简单。

## 是的，这就是策略模式

***没错，我们刚刚完成一个策略模式！***

是时候了解一下`策略模式`的具体定义了：

> Define a family of algorithms, encapsulate each one, and make them interchangeable. Strategy lets the algorithm vary independently from clients that use it.

翻译一下：**定义一族算法类，将每个算法分别封装起来，让它们可以互相替换。策略模式可以使算法的变化独立于使用它们的客户端（这里的客户端代指使用算法的代码）。**

让我们把描述问题的措辞稍作改变，不再把鸭子的行为说成是“一组行为”，我们开始把行为想成“一族算法”，算法代表了鸭子能做的事。如今算法和鸭子类之间不再是`IS-A`的关系，而是`HAS-A`的关系。

​	`“有一个”`关系相当有趣：每一个鸭子都有一个`FlyBehavior`和一个`QuackBehavior`，好将飞行和呱呱叫的行为委托给它们代为处理。当你将两个类结合起来使用，如同本例一般，这就是**“组合”**。这种做法和“继承”的不同之处在于，鸭子的行为不是继承来的，而是和合适的行为对象“组合”来的。

​	这也是我们通常所说的另一个设计原则：**多用组合，少用继承**。

## 策略模式就是简单的多态么

​	纵观`策略模式`的定义以及各种围绕`策略模式`的示例，我们很容易产生一个疑问：**策略模式就是简单的多态么？策略模式的定义何其标题看不出任何联系，到底何为策略？**

​	从维基百科上策略模式的定义可以看出，策略模式乃是一个方法论，不拘于多态一种实现方式：

> Typically, the strategy pattern stores a reference to some code in a data structure and retrieves it. This can be achieved by mechanisms such as the native [function pointer](https://en.wikipedia.org/wiki/Function_pointer), the [first-class function](https://en.wikipedia.org/wiki/First-class_function), classes or class instances in [object-oriented programming](https://en.wikipedia.org/wiki/Object-oriented_programming) languages, or accessing the language implementation's internal storage of code via [reflection](https://en.wikipedia.org/wiki/Reflection_(computer_science)).

​	`stackoverflow`上亦有相同的发问 [Is 'Strategy Design Pattern' no more than the basic use of polymorphism?](https://stackoverflow.com/questions/37670842/is-strategy-design-pattern-no-more-than-the-basic-use-of-polymorphism) 得票最高的回答也阐述了相同的意思：**策略模式，或者说设计本身，它不是指细节代码，而是一种思维方式。**

​	我们虽然在这里用多态的方式实现了策略模式，但策略模式的实现方式绝非多态一种。

​	另外我观 [设计模式之美](https://time.geekbang.org/column/intro/100039001) 中对策略模式的阐述，其称：**不使用工厂模式而直接实例化行为对象的情况为简单的面向接口编程，并非严格意义的策略模式。**如此观点虽有启发，但仍未消除心中疑虑，因为**“策略”**一词在我心中还有另一个概念。

​	策略模式的概念与定义难以让我释怀。我必须自己寻找一个答案，并说服自己去相信它，即便它可能不那么正确。因为我们每个人之所以要不断的思考，就是要缝合自我认知体系里矛盾的地方。

​	而`策略`一词，在我的知识体系里，是矛盾的！

## 从Unix设计哲学中取经

​	在最初接触`策略模式`时，我很自然的就联想到`Unix设计哲学`中的一条原则：**分离原则。**

​	***Rule of Separation: Separate policy from mechanism; separate interfaces from engines.***

​	**分离原则：策略同机制分离，接口同引擎分离**

​	起初，我认为同是`策略`，思想肯定也很接近。但是`Gof`中描述的算法族，每个具体的算法就是一个策略，让我一度觉得这两个原则之间可能没有关系。

​	的确，按照定义，这两个原则对`策略`的理解不是一个层面的东西。

​	先来了解一下分离原则。其实它比策略模式要更加普世，虽然分离原则和策略模式同为软件设计原则，但分离原则要更加抽象，而策略模式更贴近于代码，分离原则更偏向于架构。

​	分离原则讲究把策略同机制分离，策略是针对使用方，机制则说的是实现方。分离原则认为，策略和机制是按照不同的时间尺度变化的，策略的变化要远远快于机制。所以，把策略和机制揉成一团有两个负面影响：一来会使策略变的死板，难以适应用户需求的改变；二来也意味着任何策略的改变都极有可能动摇机制。

​	相反，将两者剥离，就有可能**在探索新策略的时候不会打破机制**。另外也更容易为机制写出较好的测试，因为策略都太过短命，不值得花太多经历在上面。《UNIX编程艺术》一书中，举了x图形引擎的例子。让X成为一个通用的图形引擎，而将用户界面风格留给工具包或者系统其它层来决定。GUI工具包的观感时尚来去匆匆，而光栅操作和组合却是永恒的。

​	让我们回想一下策略模式中的策略，`Gof`说每个具体的算法类就是策略。我认为以此起**"策略模式"**这个名字多少有些以偏概全，未准确传达此模式使**“算法族可以互相替换”**的主旨。分离原则和策略模式中的策略是不同场景下对不同内容的描述：**分离原则的策略是指使用多种不同机制的方法，让机制与使用分离，本质上还是抽离不变与变化的东西**；而策略模式之策略是指一系列做同类事的算法，因为同类，所以可互换，可多态！

​	所以，忘掉`策略模式`概念本身吧，它或许不是一个好名字，但却是一个好的、常用的、易用的代码设计方法。只要心中牢记“面向接口，而不是实现编程”，也许一不小心就会写出**策略模式**了！

## 总结学到的思想

1. **面向接口编程，而不是面向实现编程**
2. **“针对接口编程”真正的意思是“针对超类型编程”**
3. **“针对超类型编程”这句话，可以明确的说成“变量的声明应该是超类型，通常是一个抽象类或者是一个接口。如此，只要是具体实现次超类型的类所产生的对象，都可以指定给这个变量。这也意味着，声明类时不用理会以后执行时的真正对象类型”**
4. **多用组合，少用继承**
5. **唯一不变的就是变化本身**
6. **所有的模式都提供了一套方法让“系统中的某部分改变不会影响其它部分”**



***参考文献：***



1. [设计模式：可复用面向对象软件的基础](https://book.douban.com/subject/34262305/)
2. [Head First 设计模式](https://book.douban.com/subject/2243615/)
3. [设计模式之美](https://time.geekbang.org/column/intro/100039001) 