---
title: 策略模式（下）
date: 2021-01-15 18:22:55
tags: 
	- strategy
	- 策略模式
categories: 
	- DesignPatterns	
	- 设计模式
---

​	我在上一篇 [策略模式（上）](https://liupzmin.com/2021/01/03/design_patterns/strategy/) 中介绍了`策略模式`的概念，并使用传统的`OO`语言`Java`实现了一个模拟鸭子游戏的例子。而当今一些新兴的编程语言，并不算严格意义上的**面向对象**语言，比如`Go`、`Rust` 等。若从面向对象严格的定义上讲，它们并没有**类、继承**等特性，没有严格遵守`OO`的四大特性，也就是非严格意义上的面向对象语言。

​	实际上，对于什么是面向对象编程、什么是面向对象编程语言，并没有一个官方的、统一的定义。而且，从 `1960` 年，也就是 `60` 年前面向对象编程诞生开始，这两个概念就在不停地演化，所以，也无法给出一个明确的定义，也没有必要给出一个明确定义。

​	在此，我们不做学院派，也不替圣人争长短，揪住概念不放。我们仅仅从语言的特性上来考虑问题，看同样的设计模式，在新时代的语言中会带给我们怎样的惊喜。接下来我以`Go`语言为例，重新使用`策略模式`实现我们上一篇中的模拟鸭子的游戏。

## 超类在哪里

​	我们在上一篇文章中使用抽象类和继承来实现多态，超类就是抽象类。但是`Go`并没有提供经典`OO`语言中的类和继承，那么要实现多态我们只有`接口`可用，因此这里可以将接口视为`超类`。但是`Go`的接口又不完全等同于`Java`的接口，**Go的接口实现是完全隐式的**。

​	`interface`是`Go`语言中真正的魔法，是`Go`语言的一个创新设计，它只是方法集合，并且它与实现者之间的关系是隐式的。它让程序内部各部分之间的耦合降至最低，同时它也是连接程序各个部分之间“纽带”。隐式的`interface`实现会不经意间满足：依赖抽象、里氏替换、接口隔离等原则，这在其他语言中是需要很"刻意"的设计谋划才能实现的，但在`Go interface`来看，一切却是自然而然的。

​	我们首先定义一个鸭子接口：

```go
type Duck interface {
    Display()
    Fly()
    Quack()
    Swim()
}
```

​	按照鸭子的抽象，我们定义了`Display`、`Swim`、`Fly`和`Quack`四个方法，意在每个接口的实现者都提供自己独立的方法实现。根据上一篇的讨论，这里面混合了**不变的、可能会变的、一定会变**的内容，根据这样的接口去实现会带来代码无法复用和维护上的难题。

​	这里面只有`Display`是一定会变的，所以可以继续留在`Duck`接口中。而`Fly`和`Quack`是可能会变的，根据我们上一篇总结的原则，应该把可能会变的内容抽离出去，用单独的类实现：

```go
// 定义 Flier 接口
type Flier interface {
	Fly()
}
// FlyWithWings 实现了 Flier 接口
type FlyWithWings struct{}

func (f FlyWithWings) Fly() {
	fmt.Println("I am flying with my  wings!")
}

// FlyNoWay 实现了 Flier 接口
type FlyNoWay struct{}

func (f FlyNoWay) Fly() {
	fmt.Println("I can not fly!")
}

// FlyWithRocket 实现了 Flier 接口
type FlyWithRocket struct{}

func (f FlyWithRocket) Fly() {
	fmt.Println("I am flying with a rocket!")
}

// 定义 Quacker 接口
type Quacker interface {
	Quack()
}
// QuackNormal 实现了 Quacker 接口
type QuackNormal struct{}

func (q QuackNormal) Quack() {
	fmt.Println("Quack Quack!")
}
// Squeak 实现了 Quacker 接口
type Squeak struct{}

func (s Squeak) Quack() {
	fmt.Println("Squeak Squeak!")
}
// QuackMute 实现了 Quacker 接口
type QuackMute struct{}

func (q QuackMute) Quack() {
	fmt.Println("I can not Quack!")
}

// 目前的 Duck 接口
type Duck interface {
    Display()
}
```

​	我们分别定义了`Flier`和`Quacker`接口，并分别为之提供了多种对应的实现。根据`Java`的经验， 这时候应该在抽象类里面声明接口成员变量，将行为委托出去，但是`Go`中的接口仅仅定义了行为方法，无法定义成员，我们该如何将`Fly`和`Quack`的行为委托出去呢？

​	而且，接口中也无法定义方法实现，那么我们又该如何继承`Swim`的行为呢？

## 忘掉继承，使用组合

​	`Go`语言提供了的最为直观的组合的语法元素就是**type embedding**，即**类型嵌入**。通过类型嵌入，我们可以将已经实现的功能嵌入到新类型中，以快速满足新类型的功能需求，这种方式有些类似经典`OO`的“继承”，但在原理上与经典`OO`的继承完全不同。这是一种`Go`精心设计的“语法糖”，被嵌入的类型和新类型两者之间没有任何关系，甚至相互完全不知道对方的存在，更没有经典`OO`那种父类、子类的关系以及向上、向下转型(`type casting`)。

​	通过新类型实例调用方法时，**方法的匹配取决于方法名字，而不是类型**。这种组合方式，我称之为“垂直组合”，即通过类型嵌入，快速让一个新类型“复用”其他类型已经实现的能力，实现功能的垂直扩展。

​	利用类型嵌入，我们可以定义一个基础的鸭子结构，并在其中嵌入`Flier`和`Quacker`接口，以达到行为委托的目的。而在接下来实现具体鸭子时将基础结构体嵌入，即可达到了垂直组合的效果：

```go
type DuckBase struct {
	Flier
	Quacker
}

func (d *DuckBase) SetFlyBehaviro(f Flier) {
	d.Flier = f
}

func (d *DuckBase) SetQuackBehaviro(q Quacker) {
	d.Quacker = q
}
```

​	上述代码定义了`DuckBase`结构体，并在其中嵌入了`Flier`和`Quacker`接口（我们还提供了设置行为的方法），如果要初始化`DuckBase`就必须使用`Flier`和`Quacker`的实现来构造；根据类型嵌入的规则：**将已经实现的功能嵌入到新类型中，新类型便会获得被嵌入类型的功能**，这相当于`DuckBase`也实现了`Flier`和`Quacker`接口。

​	我们来看一个具体的鸭子实现：

```go
type MallardDuck struct {
	*DuckBase
}

func (m MallardDuck) Display() {
	fmt.Println("my head is green!")
}

type RedheadDuck struct {
	*DuckBase
}

func (r RedheadDuck) Display() {
	fmt.Println("my head is red!")
}

type RubberDuck struct {
	*DuckBase
}

func (r RubberDuck) Display() {
	fmt.Println("I am rubber duck!")
}
```

​	上述代码片段提供了三种鸭子的实现，它们都嵌入了`*DuckBase`结构（因为我们的行为设置方法是指针接收器），并且分别实现了`Duck`接口。

​	既然可以通过类型嵌入来达到类似于“继承”的效果，那么我们完全可以让`DuckBase`结构体实现`Swim`方法，这样一来，每一个鸭子的实现都可以复用`Swim`的代码：

```go
func (d DuckBase) Swim() {
	fmt.Println("I am swimming!")
}
```

​	现在，请仔细回想，让我们沿着回忆的小路再多走几遍：

1. `*DuckBase` 嵌入了`Flier`、`Quacker`接口，并实现了`Swim`方法
2. 等于`*DuckBase` 实现了`Flier`、`Quacker`接口和`Swim`方法
3. `具体的鸭子实现`嵌入了`*DuckBase` 结构
4. 等于`具体的鸭子实现`实现了`Flier`和`Quacker`接口，以及`Swim`方法

我们就可以回到最初的`Duck`接口了：

```go
type Duck interface {
    Display()
    Fly()
    Quack()
    Swim()
}
```

`Go`提供了接口组合的功能，也就是可以通过接口嵌入达到用小接口组合为大接口的目的，所以我们可以修改为接口嵌入：

```go
type Duck interface {
	Flier
	Quacker
	Display()
	Swim()
	SetFlyBehaviro(Flier)
	SetQuackBehaviro(Quacker)
}
```

我们通过组合和类型嵌入的方式构建了一个大而全、抽象度低的`Duck`接口，且完全避免了之前用继承带来的弊端。我们可以自由替换行为，也可以利用多态，且合理的实现了代码复用，同时又不会有维护上的困扰。最关键的是，我们有了一个无比自然的`Duck`接口，我们上一篇所有的努力都是在构建一个我们逻辑认知上的一个自然的`Duck`接口。

现在，`Go`很轻松就做到了！

## 游戏时刻

继续使用上一篇的例子来测试一下我们的成果吧：

```go
func main() {
	mo := MallardDuck{&DuckBase{Flier: FlyWithWings{}, Quacker: QuackNormal{}}}

	mo.Display()
	mo.Fly()
	mo.Quack()
	mo.Swim()

	mo.SetFlyBehaviro(FlyWithRocket{})
	mo.Fly()
}

my head is green!
I am flying with my  wings!
Quack Quack!
I am swimming!
I am flying with a rocket!
```

文章中的代码见：[strategy-go](https://github.com/liupzmin/Learning/blob/master/design-patterns/strategy/strategy.go)

有一点差点忘记，我们实现了**策略模式**！

## 接下来是什么

还记得我们的“针对接口编程，而不是针对实现编程”的设计准则么？

我们在上一篇`Java`的实现中使用多态时是通过new来实例化一个实现的：
```java
Duck mallard = new MallardDuck();
mallard.performQuack();
mallard.performFly();
   
Duck model = new ModelDuck();
model.performFly();
model.setFlyBehavior(new FlyRocketPowered());
model.performFly();
```
在本篇中，我们直接使用实现类型的字面量初始化的：
```go
mo := MallardDuck{&DuckBase{Flier: FlyWithWings{}, Quacker: QuackNormal{}}}
```

这似乎违反了针对接口编程的设计准则，因为我们的代码里包含了具体实现的代码。所以，接下来我们会进入`工厂模式`的学习，在此之前先解释一点：虽然是针对接口编程，但是我们不能把实现完全消灭，因为程序的运行最后依然靠的是具体实现。那么，我们就需要有一个很好的方法将其组织起来，这就是`工厂模式`！