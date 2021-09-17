---
title: 结构体速记
date: 2020-07-25 13:29:02
tags: C
categories: 
    - C
    - 重学C语言
---

平时阅读 C 语言的代码，少不了要在各种形式的 struct 中周旋，特此记录，以备查阅。

## 声明

`struct{ ... } x, y, z;`

此种方式指明了类型，并为其声明了变量，分配了存储空间。

但是这种方式没有对结构体类型命名，假如在程序的其它地方再次声明此种类型时会使程序膨胀极难维护。

因此，C 语言提供了两种方式来命名结构体类型：

1. 结构标记
2. typedef 定义

## 结构标记

```c
struct part {
    int number;
    char *name;
    int on_hand;
};
```

`part` 就是创建的标记，之后可以使用它来声明变量了：

```c
struct part part1, part2;
```

**`part` 不是类型名，因此 `struct` 关键字不能省略！**

另外，结构标记的声明可以和结构体变量的声明合并在一起，比如：

```c
struct part {
    int number;
    char *name;
    int on_hand;    
} part1, part2;
```

## typedef 定义

```c
typedef struct {
    int number;
    char *name;
    int on_hand;
} Part;
```

类型 `Part` 的名字必须出现在定义的末尾，此后便可以像内置类型一样使用 `Part` 了：

```c
Part part1, part2;
```

有两种使用 `typedef` 定义结构体类型的方法.

第一种：

```c
#include<stdio.h>

struct Point{
  int x;
  int y;
};
typedef struct Point Point;
int main() {
    Point p1;
    p1.x = 1;
    p1.y = 3;
    printf("%d \n", p1.x);
    printf("%d \n", p1.y);
    return 0;
}
```

第二种：

```c
#include<stdio.h>

typedef struct Point{
  int x;
  int y;
} Point;
int main() {
    Point p1;
    p1.x = 1;
    p1.y = 3;
    printf("%d \n", p1.x);
    printf("%d \n", p1.y);
    return 0;
}
```

## 一些问题

为什么 C 语言会提供两种方式的类型命名呢？其实 C 语言早期并没有 `typedef` ，所以标记是结构类型命名唯一的方法。当加入 typedef 时已经太晚了，标记已经无法删除了。

虽然我们可以使用任意一种方式来命名结构体类型，甚至可以同时具有标记和typedef：

```c
typedef struct part{
    int number;
    char *name;
    int on_hand;
} Part;
```

甚至标记的名字和typedef的名字都可以一样，但通常并不这么做，因为这在早期的编译器中会出现问题。

但是，有时候我们只能使用结构体标记，那就是结构体成员有自引用的时候，一个典型的例子就是链表：

```c
struct node {
    int value;
    struct node *next;
};
```

这种情况下，如果没有标记 `node` 就无法声明 `next` 的类型。

## 结构体初始化

### 声明时初始化

```c
// 声明同时进行初始化
struct student_st{
    char c;
    int score;
    const char *name;
} s1 = {'A', 89, "hello"};

// 使用标记声明并初始化
struct student_st s2 = {'A', 91, "Alan"};
```

### 指定初始化

`C99` 中允许指定初始化，将点号和成员名称组合起来称为`指示符`，指定初始化的优点是初始化值的顺序不需要和声明时一致。如果初始化中有没有指定的成员，那么这些成员将被设为0。

```c
struct student_st s2 =
    {
        .name = "YunYun",
        .c = 'B',
        .score = 92,
    };
```

结构体数组

```c
struct student_st stus[2] =
{
    {
        .c = 'D',
        .score = 94,
        /*也可以只初始化部分成员*/
    },
    {
        .c = 'D',
        .score = 94,
        .name = "Xxx"
    },
```

### 复合字面量

`C99` 中可以使用`复合字面量`来创建没有名字的数组，这通常用于函数的参数传递，以避免先创建变量。

```c
total = sum_array((int []){1, 2, 3, 4, 5});
```

复合字面量的格式为：**先在一对圆括号内给定类型名，随后在一对花括号内设定所包含的元素的值。**

同样，复合字面量也可以用于”实时“创建一个结构，而不需要将其存储在变量中。生成的结构也可以像参数一样传递，可以被函数返回，也可以赋值给变量。

```c
// 看如下函数调用
print_part((struct part){528, "Disk drive", 100});

// 变量赋值
part1 = (struct part){528, "Disk drive", 100}；
```

细细体会一下变量赋值的例句，它有些类似于含有初始化的声明，但是并不完全一样，初始化只能出现在声明中，不能出现在赋值语句中。

```c
struct part1 = {528, "Disk drive", 100}; // ok
part1 = {528, "Disk drive", 100}; // error
```