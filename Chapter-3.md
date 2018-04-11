# 第三章 OCaml 中的对象

[TOC]



## 3.1  类和对象

下面的point类定义了一个距离变量x和get_x，move两个方法。距离变量的初始值为0。这边被声明为可变的，所以方法可以改变他的值：

```ocaml
# class point =
    object
      val mutable x = 0
      method get_x = x
      method move d = x <- x + d
    end;;
class point :
  object val mutable x : int method get_x : int method move : int -> unit end
```

我们现在来创建一个新的point p，一个point类的实例：

```ocaml
# let p = new point;;
val p : point = <obj>
```

需要注意的是p的类型是point。\<obj>是一种类定义的省略定义。它代表了\<get_x : int; move : int -> unit>，列出了point类的方法以及方法的类型。

我们现在使用p的一些方法：

```ocaml
# p#get_x;;
- : int = 0
```

```ocaml
# p#move 3;;
- : unit = ()
```

```ocaml
# p#get_x;;
- : int = 3
```

类的解析只发生在对象产生的时候，所以，在下面的例子里面，实例变量x在每次创建的时候都有不同的值：

```ocaml
# let x0 = ref 0;;
val x0 : int ref = {contents = 0}
```

```ocaml
# class point =
    object
      val mutable x = incr x0; !x0
      method get_x = x
      method move d = x <- x + d
    end;;
class point :
  object val mutable x : int method get_x : int method move : int -> unit end
```

```ocaml
# new point#get_x;;
- : int = 1
```

```Ocaml
# new point#get_x;;
- : int = 2
```

point类也可以被抽象的定义，根据情况传入x的值创建实例：

```ocaml
# class point = fun x_init ->
    object
      val mutable x = x_init
      method get_x = x
      method move d = x <- x + d
    end;;
class point :
  int ->
  object val mutable x : int method get_x : int method move : int -> unit end
```

类似于函数定义，上面的定义可以简略为：

```ocaml
# class point x_init =
    object
      val mutable x = x_init
      method get_x = x
      method move d = x <- x + d
    end;;
class point :
  int ->
  object val mutable x : int method get_x : int method move : int -> unit end
```

现在point类变为了一个需要初始化参数来创建对象的函数：

```
# new point;;
- : int -> point = <fun>
```

```ocaml
# let p = new point 7;;
val p : point = <obj>
```

理所应当地，参数x_init是对于整个定义可见的。举个例子，下面的例子包含来方法get_offset来获取位置和初始位置的偏移量：

```ocaml
# class point x_init =
    object
      val mutable x = x_init
      method get_x = x
      method get_offset = x - x_init
      method move d = x <- x + d
    end;;
class point :
  int ->
  object
    val mutable x : int
    method get_offset : int
    method get_x : int
    method move : int -> unit
  end
```

这样的表达式可以被解析，并且会在定义对象的时候被绑定。这种方式有利于强制设定常量。举个例子，这些point会被自动调整到网格中最近的点：

```ocaml
# class adjusted_point x_init =
    let origin = (x_init / 10) * 10 in
    object
      val mutable x = origin
      method get_x = x
      method get_offset = x - origin
      method move d = x <- x + d
    end;;
class adjusted_point :
  int ->
  object
    val mutable x : int
    method get_offset : int
    method get_x : int
    method move : int -> unit
  end
```

（我们也可以在x_init不在网格内的时候抛出异常）事实上，这样的定义同样可以被包含在调用point来初始化的类：

```ocaml
# class adjusted_point x_init =  point ((x_init / 10) * 10);;
class adjusted_point : int -> point
```

作为一种可选的方式，我们也可以这样构建调整过的点：

```ocaml
# let new_adjusted_point x_init = new point ((x_init / 10) * 10);;
val new_adjusted_point : int -> point = <fun>
```

然而，前者的形式更加适合，因为调整是point一部分的定义并且会被继承。

具有这种能力的类构造器在其他语言中也有。一些类构造器可以通过不同的初始化方式来创建对象，另一个选择时使用initializer，将会在后面的3.4介绍。



## 3.2  立即对象

我们还有一种更直接的方式可以不用类来创建对象。

它和类定义是类似的，但是最后得到的是单个对象而不是一个类，所有在里面描述的定义会直接应用到立即对象中：

```ocaml
# let p =
    object
      val mutable x = 0
      method get_x = x
      method move d = x <- x + d
    end;;
val p : < get_x : int; move : int -> unit > = <obj>
```

```ocaml
# p#get_x;;
- : int = 0
```

```ocaml
# p#move 3;;
- : unit = ()
```

```ocaml
# p#get_x;;
- : int = 3
```

不同于类的是，类不能在表达式中出现，但是立即对象可以在任意地方出现，并且使用相应环境中的量：

```ocaml
# let minmax x y =
    if x < y then object method min = x method max = y end
    else object method min = y method max = x end;;
val minmax : 'a -> 'a -> < max : 'a; min : 'a > = <fun>
```

立即对象比起勒来说有两个缺点：他们的类型不能被简略，你也不能继承他们。但是这两个缺点有些时候也可以是有点，就像我们在3.3和3.10中即将看到的例子。

## 3.3  自参照

一个方法或者初始化方法具有向自己（这个对象本身）发送消息的能力。为了实现这个，自身必须被单独的绑定，就像下面的变量s（s可以是任何标识符，因此我们通常使用self来命名它）

```ocaml
# class printable_point x_init =
    object (s)
      val mutable x = x_init
      method get_x = x
      method move d = x <- x + d
      method print = print_int s#get_x
    end;;
class printable_point :
  int ->
  object
    val mutable x : int
    method get_x : int
    method move : int -> unit
    method print : unit
  end
```

```ocaml
# let p = new printable_point 7;;
val p : printable_point = <obj>
```

```ocaml
# p#print;;
7- : unit = ()
```

变量在声明方法的时候会动态绑定。特别的是，在printable_point被继承的时候，他的变量也会正确的绑定到子类的对象上。

在使用self大家有一个共通的问题，在子类里面类型它会被拓展，而且你无法改善这个错误，这里是个例子：

```OCaml
# let ints = ref [];;
val ints : '_weak1 list ref = {contents = []}
```

```ocaml
# class my_int =
    object (self)
      method n = 1
      method register = ints := self :: !ints
    end ;;
Error: This expression has type < n : int; register : 'a; .. >
       but an expression was expected of type 'weak1
       Self type cannot escape its class
```

你可以直接忽略前两行的错误信息。我们的重点是最后一行信息，当你把self作为一个外部引用的时候，它不可能通过继承被拓展。在3.12我们会对这个问题有更深入的了解。鉴于立即对象是不能被拓展的，所以这个问题不会发生。

```ocaml
# let my_int =
    object (self)
      method n = 1
      method register = ints := self :: !ints
    end;;
val my_int : < n : int; register : unit > = <obj>
```

## 3.4  初始化器

在类定义中的let绑定会在对象构造的时候被赋值。但是也有方法在对象创建之后立即就执行一个表达式。这样的匿名方法叫做初始化器。它同样可以获取self和实例变量。

```ocaml
# class printable_point x_init =
    let origin = (x_init / 10) * 10 in
    object (self)
      val mutable x = origin
      method get_x = x
      method move d = x <- x + d
      method print = print_int self#get_x
      initializer print_string "new point at "; self#print; print_newline ()
    end;;
class printable_point :
  int ->
  object
    val mutable x : int
    method get_x : int
    method move : int -> unit
    method print : unit
  end
```

```ocaml
# let p = new printable_point 17;;
new point at 10
val p : printable_point = <obj>
```

初始化器不能被重写。相反的是，所有的初始化器都将按顺序被执行。初始化器对于非变量的定义也是很有用的。在6.1中我们会看到另一个例子。

## 3.5  虚拟方法

使用关键字virtual可以声明一个方法却不定义它。这种方法将会在之后的子类中更详细的介绍。一个包含了虚拟方法的类必须被标记为virtual，而且不能被实例化。但是它依然定义了类型的缩略形式（将虚拟方法和其他方法同等对待）。

```ocaml
# class virtual abstract_point x_init =
    object (self)
      method virtual get_x : int
      method get_offset = self#get_x - x_init
      method virtual move : int -> unit
    end;;
class virtual abstract_point :
  int ->
  object
    method get_offset : int
    method virtual get_x : int
    method virtual move : int -> unit
  end
```

```ocaml
# class point x_init =
    object
      inherit abstract_point x_init
      val mutable x = x_init
      method get_x = x
      method move d = x <- x + d
    end;;
class point :
  int ->
  object
    val mutable x : int
    method get_offset : int
    method get_x : int
    method move : int -> unit
  end
```

实例变量也同样可以virtual声明，和虚拟方法是一样的。

```ocaml
# class virtual abstract_point2 =
    object
      val mutable virtual x : int
      method move d = x <- x + d
    end;;
class virtual abstract_point2 :
  object val mutable virtual x : int method move : int -> unit end
```

```ocaml
# class point2 x_init =
    object
      inherit abstract_point2
      val mutable x = x_init
      method get_offset = x - x_init
    end;;
class point2 :
  int ->
  object
    val mutable x : int
    method get_offset : int
    method move : int -> unit
  end
```

## 3.6  私有方法

私有方法是那些不在对象接口中出现的方法。他们可以被类里面其他方法访问。

```ocaml
# class restricted_point x_init =
    object (self)
      val mutable x = x_init
      method get_x = x
      method private move d = x <- x + d
      method bump = self#move 1
    end;;
class restricted_point :
  int ->
  object
    val mutable x : int
    method bump : unit
    method get_x : int
    method private move : int -> unit
  end
```

```ocaml
# let p = new restricted_point 0;;
val p : restricted_point = <obj>
```

```ocaml
# p#move 10 ;;
Error: This expression has type restricted_point
       It has no method move
```

```ocaml
# p#bump;;
- : unit = ()
```

需要注意的是他和Java或者C++里面的私有方法或者protected方法不同（可以被其他该类的实例访问）。这是OCaml类型与类之间依赖的直接结果：两个不关联的类可以产生很多同类型的对象，但是在类型层面是无法判断一个对象具体来源于哪一个类。但是在3.17里面介绍的友方法是可以实现这样的需求的。

私有方法是可以被继承的（它们通常在子类里面是可见的），除非它们被签名隐藏了。就像下面的例子一样：

私有方法也可以在子类里面声明：

```ocaml
# class point_again x =
    object (self)
      inherit restricted_point x
      method virtual move : _
    end;;
class point_again :
  int ->
  object
    val mutable x : int
    method bump : unit
    method get_x : int
    method move : int -> unit
  end
```

在这里的virtual声明只是为了声明一个方法而不提供定义。因为我们没有加上private声明，这使得方法是公有的。

一种可选的声明方式是：

```ocaml
# class point_again x =
    object (self : < move : _; ..> )
      inherit restricted_point x
    end;;
class point_again :
  int ->
  object
    val mutable x : int
    method bump : unit
    method get_x : int
    method move : int -> unit
  end
```

由于self类型是恒定的，所以他需要一个公有的move方法，这就足以导致private的声明重写。

一些人需要私有方法在子类中需要继续私有。因为在子类中方法是可见的，我们永远可以写一个同样的方法，然后在过程中调用上一层的方法，这样传递下去：

```ocaml
# class point_again x =
    object
      inherit restricted_point x as super
      method move = super#move
    end;;
class point_again :
  int ->
  object
    val mutable x : int
    method bump : unit
    method get_x : int
    method move : int -> unit
  end
```

当然，私有方法也可以是虚拟方法，只要以private virtual方法定义就可以了。

## 3.7  类接口

类接口参照了类的定义。它可以被直接用来重定向类的类型，就像类的声明一样，类同样可以定义新的类型。

```ocaml
# class type restricted_point_type =
    object
      method get_x : int
      method bump : unit
  end;;
class type restricted_point_type =
  object method bump : unit method get_x : int end
```

```ocaml
# fun (x : restricted_point_type) -> x;;
- : restricted_point_type -> restricted_point_type = <fun>
```

另外，对于程序文档来说，类接口可以被用来限制类的类型。不论是实例变量，还是私有方法，都可以被类的类型限制。然而，公有方法和虚拟的成员是不能被限制的。

```ocaml
# class restricted_point' x = (restricted_point x : restricted_point_type);;
class restricted_point' : int -> restricted_point_type
```

或者，这样写亦可以：

```Ocaml
# class restricted_point' = (restricted_point : int -> restricted_point_type);;
class restricted_point' : int -> restricted_point_type
```

类接口同样也可以在module签名里面说明，可以用来针对module来重定义类类型。

```ocaml
# module type POINT = sig
    class restricted_point' : int ->
      object
        method get_x : int
        method bump : unit
      end
  end;;
module type POINT =
  sig
    class restricted_point' :
      int -> object method bump : unit method get_x : int end
  end
```

```ocaml
# module Point : POINT = struct
    class restricted_point' = restricted_point
  end;;
module Point : POINT
```

## 3.8  继承

我们现在通过继承以前的point类来定义新的类colored point。这个类拥有point的所有变量和方法，并且加上了新的变量c和方法color。

```ocaml
# class colored_point x (c : string) =
    object
      inherit point x
      val c = c
      method color = c
    end;;

class colored_point :
  int ->
  string ->
  object
    val c : string
    val mutable x : int
    method color : string
    method get_offset : int
    method get_x : int
    method move : int -> unit
  end
```

```ocaml
# let p' = new colored_point 5 "red";;
val p' : colored_point = <obj>
```

```ocaml
# p'#get_x, p'#color;;
- : int * string = (5, "red")
```

point和colored point的类型是不同的，因为point没有color方法。但是，get_x方法却是对于两个类的实例来说都是有效的，所以get_x可以应用于这两个类。

```ocaml
# let get_succ_x p = p#get_x + 1;;
val get_succ_x : < get_x : int; .. > -> int = <fun>
```

```ocaml
# get_succ_x p + get_succ_x p';;
- : int = 8
```

这样使用有一个前提就是，这个方法不能在之前被定义过，不然会出现以下情况：

```ocaml
# let set_x p = p#set_x;;
val set_x : < set_x : 'a; .. > -> 'a = <fun>
```

```ocaml
# let incr p = set_x p (get_succ_x p);;
val incr : < get_x : int; set_x : int -> 'a; .. > -> 'a = <fun>
```

## 3.9  多重继承

在OCaml里是可以多重继承的。但是对于子类方法的重写会覆盖父类的方法，最终只会留下方法最近声明的定义。之前对于方法的定义可以通过绑定相关祖先类来复用。在下面这个例子中，super是被绑定在祖先类printable_point上的。super是一个只能用来在类中调用父类方法的伪值。

```ocaml
# class printable_colored_point y c =
    object (self)
      val c = c
      method color = c
      inherit printable_point y as super
      method! print =
        print_string "(";
        super#print;
        print_string ", ";
        print_string (self#color);
        print_string ")"
    end;;

class printable_colored_point :
  int ->
  string ->
  object
    val c : string
    val mutable x : int
    method color : string
    method get_x : int
    method move : int -> unit
    method print : unit
  end
```

```ocaml
# let p' = new printable_colored_point 17 "red";;

new point at (10, red)
val p' : printable_colored_point = <obj>
```

```ocaml
# p'#print;;

(10, red)- : unit = ()
```

隐藏在父类里面的私有方法是不可见的，这样它也不会被重写。因为初始化方法也是作为私有方法存在，所有的私有方法会按照继承顺序执行。

需要注意的是，为了使逻辑清晰，被重写的方法必须用!标志标记，如果之前没有定义过这个方法，那么会导致错误。

```ocaml
#   object
      method! m = ()
    end;;
Error: The method `m' has no previous definition
```

这种明确重写也可以使用在继承和值上：

```ocaml
# class another_printable_colored_point y c c' =
    object (self)
    inherit printable_point y
    inherit! printable_colored_point y c
    val! c = c'
    end;;
class another_printable_colored_point :
  int ->
  string ->
  string ->
  object
    val c : string
    val mutable x : int
    method color : string
    method get_x : int
    method move : int -> unit
    method print : unit
  end
```

## 3.10  参数初始化类

引用单元可以通过对象实现，但是比较不成熟的类定义是过不了类型检查的：

```OCaml
# class oref x_init =
    object
      val mutable x = x_init
      method get = x
      method set y = x <- y
    end;;
Error: Some type variables are unbound in this type:
         class oref :
           'a ->
           object
             val mutable x : 'a
             method get : 'a
             method set : 'a -> unit
           end
       The method get has type 'a where 'a is unbound
```

过不了的原因是，这个类里有至少一个方法拥有多态参数，这样的类必须有参数声明，或者里面的方法必须是确定类型的。如果确定了类型，那么类就可以这样定义：

```OCaml
# class oref (x_init:int) =
    object
      val mutable x = x_init
      method get = x
      method set y = x <- y
    end;;
class oref :
  int ->
  object val mutable x : int method get : int method set : int -> unit end
```

需要注意的是，立即对象并没有这样的限制：

```OCaml
# let new_oref x_init =
    object
      val mutable x = x_init
      method get = x
      method set y = x <- y
    end;;
val new_oref : 'a -> < get : 'a; set : 'a -> unit > = <fun>
```

另一方面，多态类声明必须明确的给出相应的类型参数。类的类型参数通过[]来声明。类参数必须对类中方法的参数类型有着明确的类型限制。

```Ocaml
# class ['a] oref x_init =
    object
      val mutable x = (x_init : 'a)
      method get = x
      method set y = x <- y
    end;;
class ['a] oref :
  'a -> object val mutable x : 'a method get : 'a method set : 'a -> unit end
```

```OCaml
# let r = new oref 1 in r#set 2; (r#get);;
- : int = 2
```

类型参数声明也可以被类中的方法推导出，在如下的例子中，展示了一种通过类声明反推类型的例子：

```OCaml
# class ['a] oref_succ (x_init:'a) =
    object
      val mutable x = x_init + 1
      method get = x
      method set y = x <- y
    end;;
class ['a] oref_succ :
  'a ->
  object
    constraint 'a = int
    val mutable x : int
    method get : int
    method set : int -> unit
  end
```

考虑更复杂的情况，定义一个圆，中心可以是任何一种点，在move方法加上了额外的类型限制，因为我们不能定义一个无法通过类的类型定义确定的方法：

```OCaml
# class ['a] circle (c : 'a) =
    object
      val mutable center = c
      method center = center
      method set_center c = center <- c
      method move = (center#move : int -> unit)
    end;;
class ['a] circle :
  'a ->
  object
    constraint 'a = < move : int -> unit; .. >
    val mutable center : 'a
    method center : 'a
    method move : int -> unit
    method set_center : 'a -> unit
  end
```

另外一种定义的方法，可以通过使用constraint句型来定义类型。在下面这个例子里，类型#point是通过之前定义point类而来的。这样的缩略定义，实际上是把#point的类型声明< get_x : int; move : int -> unit; .. >拓展到了a'。下面的例子展示了这种方法：

```OCaml
# class ['a] circle (c : 'a) =
    object
      constraint 'a = #point
      val mutable center = c
      method center = center
      method set_center c = center <- c
      method move = center#move
    end;;
class ['a] circle :
  'a ->
  object
    constraint 'a = #point
    val mutable center : 'a
    method center : 'a
    method move : int -> unit
    method set_center : 'a -> unit
  end
```

colored_circle类是一个特殊版的circle类，所以我们使用继承来实现。需要注意的是，当继承一个参数初始化类的时候，类型参数都需要明确的给出：

```Ocaml
# class ['a] colored_circle c =
    object
      constraint 'a = #colored_point
      inherit ['a] circle c
      method color = center#color
    end;;
class ['a] colored_circle :
  'a ->
  object
    constraint 'a = #colored_point
    val mutable center : 'a
    method center : 'a
    method color : string
    method move : int -> unit
    method set_center : 'a -> unit
  end
```

## 3.11  多态方法

参数初始化类虽然可以解决内容多态的问题，但是这任然不能解决方法多态的问题。

一个经典的例子就是迭代器：

```OCaml
# List.fold_left;;
- : ('a -> 'b -> 'a) -> 'a -> 'b list -> 'a = <fun>
```

```OCaml
# class ['a] intlist (l : int list) =
    object
      method empty = (l = [])
      method fold f (accu : 'a) = List.fold_left f accu l
    end;;
class ['a] intlist :
  int list ->
  object method empty : bool method fold : ('a -> int -> 'a) -> 'a -> 'a end
```

这个看起来是一个多态的迭代器，但是实际上在使用的时候却不是这样的。

```OCaml
# let l = new intlist [1; 2; 3];;
val l : '_weak2 intlist = <obj>
```

```OCaml
# l#fold (fun x y -> x+y) 0;;
- : int = 6
```

```OCaml
# l;;
- : int intlist = <obj>
```

```OCaml
# l#fold (fun s x -> s ^ string_of_int x ^ " ") "" ;;
Error: This expression has type int but an expression was expected of type
         string
```

在第一个叠加的应用中，我们的迭代器是有效的。但是，因为对象本身不是多态的（构造的时候类型已经固定），使用fold方法因为特定实例的类型描述而被制约了。所以我们把它作为string的迭代器的尝试失败了。

这里的问题在于我们关注点错了，我们需要的不是一个多态的类，而是一个多态的方法。所以我们要在定义方法的时候给丁特定的多态类型。

```ocaml
# class intlist (l : int list) =
    object
      method empty = (l = [])
      method fold : 'a. ('a -> int -> 'a) -> 'a -> 'a =
        fun f accu -> List.fold_left f accu l
    end;;
class intlist :
  int list ->
  object method empty : bool method fold : ('a -> int -> 'a) -> 'a -> 'a end
```

```ocaml
# let l = new intlist [1; 2; 3];;
val l : intlist = <obj>
```

```ocaml
# l#fold (fun x y -> x+y) 0;;
- : int = 6
```

```ocaml
# l#fold (fun s x -> s ^ string_of_int x ^ " ") "";;
- : string = "1 2 3 "
```

参考编译器显示的类类型，一个多态的方法类型，必须在类定义过程中完全明确，但是确定了类型的值不用那么急着确定。为什么类型要被确定呢？为题在于 (int -> int -> int) -> int -> int这样的类型也可以用于fold，但是这样就和我们定义的多态类型就不匹配了。（自动推导只对顶级类型变量有效，对内部的量却是不进行的，变成了一个无法确定的问题）。所以编译器不能在两种类型之间抉择，所以必须需要类型标注。

但是，类型可以被已经知道类型的类触发，下面就是一个例子：

```OCaml
# class intlist_rev l =
    object
      inherit intlist l
      method! fold f accu = List.fold_left f accu (List.rev l)
    end;;
```

下面是类型定义和描述分开的示例：

```OCaml
# class type ['a] iterator =
    object method fold : ('b -> 'a -> 'b) -> 'b -> 'b end;;
```

```OCaml
# class intlist l =
    object (self : int #iterator)
      method empty = (l = [])
      method fold f accu = List.fold_left f accu l
    end;;
```

需要注意的是这里的 `self : int #iterator ` 语句确定了这个对象实现的迭代器借口类型。

多态方法和普通方法的调用方法是相同的，但是你需要知道类型引用的限制。另一方面，这些方法会被认为是确认类型的，而且声明了会发生冲突的类型。

```OCAMl
# let sum lst = lst#fold (fun x y -> x+y) 0;;
val sum : < fold : (int -> int -> int) -> int -> 'a; .. > -> 'a = <fun>
```

```ocaml
# sum l ;;
Error: This expression has type intlist
       but an expression was expected of type
         < fold : (int -> int -> int) -> int -> 'a; .. >
       Types for method fold are incompatible
```

使用的时候，只要加上类型描述就可以了：

```ocaml
# let sum (lst : _ #iterator) = lst#fold (fun x y -> x+y) 0;;
val sum : int #iterator -> int = <fun>
```

当然，类型描述也可以是对方法的类型描述，但是这样的情况只发生在需要确定变量的时候：

```ocaml
# let sum lst =
    (lst : < fold : 'a. ('a -> _ -> 'a) -> 'a -> 'a; .. >)#fold (+) 0;;
val sum : < fold : 'a. ('a -> int -> 'a) -> 'a -> 'a; .. > -> int = <fun>
```

另一种多态方法的用法允许我们在函数中允许多种子类型的参数。我们在3.8节已经展示了这些多态方法在类中是怎样声明的，这样的方法可以拓展到其他方法：

```ocaml
# class type point0 = object method get_x : int end;;
class type point0 = object method get_x : int end
```

```ocaml
# class distance_point x =
    object
      inherit point x
      method distance : 'a. (#point0 as 'a) -> int =
        fun other -> abs (other#get_x - x)
    end;;
class distance_point :
  int ->
  object
    val mutable x : int 
    method distance : #point0 -> int
    method get_offset : int
    method get_x : int
    method move : int -> unit
  end
```

```ocaml
# let p = new distance_point 3 in
  (p#distance (new point 8), p#distance (new colored_point 1 "blue"));;
- : int * int = (5, 2)
```

需要注意的是#point0 as 'a这种用法把#point0部分类型拓展到了'a。如果你要在对象内实现多态，那么他应该要被单独的描述类型。

```ocaml
# class multi_poly =
    object
      method m1 : 'a. (< n1 : 'b. 'b -> 'b; .. > as 'a) -> _ =
        fun o -> o#n1 true, o#n1 "hello"
      method m2 : 'a 'b. (< n2 : 'b -> bool; .. > as 'a) -> 'b -> _ =
        fun o x -> o#n2 x
    end;;
class multi_poly :
  object
    method m1 : < n1 : 'b. 'b -> 'b; .. > -> bool * string
    method m2 : < n2 : 'b -> bool; .. > -> 'b -> bool
  end
```

在方法m1中，o必须是一个至少有方法n1的对象，但是其本身多态。在方法m2中，x和n2必须拥有同样的类型，并且这个类型和‘a是在同等级别的。

## 3.12  使用强制转换

子类型在大部分情况下都是不确定的，但是，我们有两种方法来表现子类型。最常见的方法就是，我们给定函数域和函数目标域的强制类型。

我们已经知道了point和colored point有类型冲突，举个例子来说，他们不能在同一个list里面出现。但是，一个colored point可以强制转换为point：

```Ocaml
# let colored_point_to_point cp = (cp : colored_point :> point);;
val colored_point_to_point : colored_point -> point = <fun>
```

```Ocaml
# let p = new point 3 and q = new colored_point 4 "blue";;
val p : point = <obj>
val q : colored_point = <obj>
```

```Ocaml
# let l = [p; (colored_point_to_point q)];;
val l : point list = [<obj>; <obj>]
```

一个t类型的对象只有在t是t'的子类型的时候才能被视为t'类型的对象。举个例子来说，一个point不能被视为一个colored point。

```ocaml
# (p : point :> colored_point);;
Error: Type point = < get_offset : int; get_x : int; move : int -> unit >
       is not a subtype of
         colored_point =
           < color : string; get_offset : int; get_x : int;
             move : int -> unit > 
```

这样不带实时类型检查的类型转换是不安全的。因为在运行时类型检查可能会报错，并且会要求运行时的上下文类型信息来处理错误，但是OCaml 系统中没有这样的设计。因此，我们并不能在语言内进行这样的操作。

需要注意的是，子类型和和继承是没有关系的。继承是类与类之间的关系，而子类型是类型与类型之间的关系。举个例子来讲，colored point类可以直接不用继承point类建立，但是colord point的类型不会因此改变，而且它的类型依然是point的子类型。

强制类型转换的域常常可以被忽略，就像下面这个例子，我们可以这样定义：

```ocaml
# let to_point cp = (cp :> point);;
val to_point : #point -> point = <fun>
```

在这个例子中，函数colored_point_to_point是函数to_point的一个实例。但是这样的情况不会总是发生。便携更加完整的强制类型转换就不可避免了。考虑下面这个类：

```ocaml
# class c0 = object method m = {< >} method n = 0 end;;
class c0 : object ('a) method m : 'a method n : int end
```

对象类型c0是对类型<m : 'a; n : int> 的简写。考虑以下的类型声明：

```ocaml
# class type c1 =  object method m : c1 end;;
class type c1 = object method m : c1 end
```

对象类型c1是对类型 <m : 'a> 的简写，c0到c1的强制类型转换是正确的：

```o cam l
# fun (x:c0) -> (x : c0 :> c1);;
- : c0 -> c1 = <fun>
```

但是，强制类型转换域并不能一直被省略。在那种情况下，我们需要书写类型的明确形式。有时，我们也可以通过改变类类型定义来解决问题：

```o cam l
# class type c2 = object ('a) method m : 'a end;;
class type c2 = object ('a) method m : 'a end
```

```ocaml
# fun (x:c0) -> (x :> c2);;
- : c0 -> c2 = <fun>
```

在类类型c1和c2是不同的同时，对象类型c1和c2可以被拓展为同样的对象类型（拥有同样的方法和类型）。当强制类型转换域是左包含，而且在其到达域（目标类型集）是一个已知类类型的缩写，那么会优先选择类类型而不是对象类型来作为函数的类型源。这样就使我们可以在大部分情况下忽略强制转换域，直接把子类转换到超类。类型的强制转换如下所示：

```ocaml
# let to_c1 x = (x :> c1);;
val to_c1 : < m : #c1; .. > -> c1 = <fun>
```

```ocaml
# let to_c2 x = (x :> c2);;
val to_c2 : #c2 -> c2 = <fun>
```

需要注意的是两个类型转换之间的差别：一方面，在to_c2的情况下，#c2 = < m : 'a; .. > 类型是多态相关的（根据在c2类类型中的明确的相关），因此可以成功的接受类c0对象进行强制类型转换。另一方面，在第一种情况中，c1只是被拓展和展开两次来包含< m : < m : c1; .. >; .. >类型，而没有引入相关来定义类型。你也许会注意到 to_c2 的类型是 #c2 -> c2 而 to_c1 的类型是比起#c1 -> c1 却更加广泛。但是这种情况并不会一直发生，因为有些类的#c 的实例并不是c的子类型（在3.16里面有解释）。总的来说，对于无参数类的强制类型转换 (_ :> c)总是比(_ : #c :> c)更加广泛的。

在我们在定义类c的同时定义一个类c强制类型转换的时候普遍会发生一个问题。这个问题是因为类型缩写还没完全定义，所以它的子类型也不是完全清楚的，所以 (_ :> c) 或者 (_ : #c :> c)会被定义为一个函数，如下：

```
# function x -> (x :> 'a);;
- : 'a -> 'a = <fun>
```

在下面的例子中，强制类型转换应用到了self上，但是self的类型与闭类型的c（一个对象的闭类型是没有省略的对象类型）不统一，这样会限制self的类型成为闭类型，是不被允许的。实际上，self的是不会成为闭类型的：这样可以阻止未来所有对于类拓展的可能。所以，当两个类型统一后产生了一个对象闭类型的时候，会产生类型错误。

```ocaml
# class c = object method m = 1 end
  and d = object (self)
    inherit c
    method n = 2
    method as_c = (self :> c)
  end;;
Error: This expression cannot be coerced to type c = < m : int >; it has type
         < as_c : c; m : int; n : int; .. >
       but is here used with type c
       Self type cannot escape its class
```

但是，这个问题最通用的实例，把self强制转换为当前类，是一个被类型检查认可的特殊例子，会被正确的赋予类型。

```ocaml
# class c = object (self) method m = (self :> c) end;;
class c : object method m : c end
```

这样就允许了下面的用法，用来保存全部类及其子类的对象list：

```ocaml
# let all_c = ref [];;
val all_c : '_weak3 list ref = {contents = []}
```

```ocaml
# class c (m : int) =
    object (self)
      method m = m
      initializer all_c := (self :> c) :: !all_c
    end;;
class c : int -> object method m : int end
```

这个用法也可用来取回一个类型已经被弱化的对象：

```ocaml
# let rec lookup_obj obj = function [] -> raise Not_found
    | obj' :: l ->
       if (obj :> < >) = (obj' :> < >) then obj' else lookup_obj obj l ;;
val lookup_obj : < .. > -> (< .. > as 'a) list -> 'a = <fun>
```

```ocaml
# let lookup_c obj = lookup_obj obj !all_c;;
val lookup_c : < .. > -> < m : int > = <fun>
```

因为使用了引用，这里我们看到的类型< m : int >只是c的一个拓展，我们成功的得到了一个类型c的对象。

之前的强制类型转换问题通常可以通过先使用类类型定义类型缩写来避免：

```
# class type c' = object method m : int end;;
class type c' = object method m : int end
```

```ocaml
# class c : c' = object method m = 1 end
  and d = object (self)
    inherit c
    method n = 2
    method as_c = (self :> c')
  end;;
class c : c'
and d : object method as_c : c' method m : int method n : int end
```

也可以通过使用虚拟类来解决。继承这个类，同时再强制让所有c的方法类型和c'相同：

```ocaml
# class virtual c' = object method virtual m : int end;;
class virtual c' : object method virtual m : int end
```

```ocaml
# class c = object (self) inherit c' method m = 1 end;;
class c : object method m : int end
```

也可以考虑直接定义类型缩略：

```ocaml
# type c' = <m : int>;;
```

但是，#c'的缩略却不能被简单这样定义。它智能通过类或者类类型来定义。因为 #-缩略 会带有一个不能被明确命名的匿名变量，你最多只能这样定义：

```ocaml
# type 'a c'_class = 'a constraint 'a = < m : int; .. >;;
```

通过使用一个额外的类型变量来捕捉对象开放类型。

## 3.13 函数式对象

我们可以写一个没有带有实例变量绑定的point类。通过使用重写结构{< ... >} 返回一个self，这样可以改变一些对象变量的值：

```ocaml
# class functional_point y =
    object
      val x = y
      method get_x = x
      method move d = {< x = x + d >}
    end;;
class functional_point :
  int ->
  object ('a) val x : int method get_x : int method move : int -> 'a end
```

```ocaml
# let p = new functional_point 7;;
val p : functional_point = <obj>
```

```ocaml
# p#get_x;;
- : int = 7
```

```ocaml
# (p#move 3)#get_x;;
- : int = 10
```

```ocaml
# p#get_x;;
- : int = 7
```

需要注意的是functional_point两个类型缩略是相关的，因为self 的类型是'a而'，a在方法move中出现了。

上面的functional_point定义和下面这个是不等价的：

```ocaml
# class bad_functional_point y =
    object
      val x = y
      method get_x = x
      method move d = new bad_functional_point (x+d)
    end;;
class bad_functional_point :
  int ->
  object
    val x : int
    method get_x : int
    method move : int -> bad_functional_point
  end
```

即使一些类的对象会有相同的表现，但是他们的子类的对象也会不同。在子类bad_functional_point中，move方法依然会返回父类的对象。相反的是， functional_point的子类中，move方法将会返回子类的对象。

函数式更新一般会用于结合两个方法（如同在6.2.1中描述的那样）。

## 3.14  克隆对象

对象可以被克隆，不管它是函数式的还是命令式的。库函数 Oo.copy 会浅克隆一个对象，也就是说，方法会返回一个同对象声明一样的，包含着相同的方法以及实例变量的新的对象。虽然实例变量被复制了，但是他们的内容是共享的。修改克隆对象的实例变量值不会影响原来的对象，反之亦然。但是一个深绑定（比如说一个实例变量为引用单元）的改变，必然会影响两者。

Oo.copy 的类型如下：

```ocaml
# Oo.copy;;
- : (< .. > as 'a) -> 'a = <fun>
```

这个类型里的关键字绑定了对象类型< .. >作为'a的类型。于是，Oo.copy 接受含有任何方法的对象，然后返回相同的类型。Oo.copy 的类型有别于 < .. > -> < .. > ，因为每一个省略代表的是不同的方法集合。省略说实话，是作为一个实际类型来表现的。

```ocaml
# let p = new point 5;;
val p : point = <obj>
```

```ocaml
# let q = Oo.copy p;;
val q : point = <obj>
```

```ocaml
# q#move 7; (p#get_x, q#get_x);;
- : int * int = (5, 12)
```

事实上，Oo.copy p 将会表现为 p#copy （当作一个有 {< >}定义的公有方法在p里定义）。

对象可以通过操作符=和<>来进行一般的比较。两个对象相等指的是物理上的（同一个储存空间），一个对象和他的克隆是不相等的。

```ocaml
# let q = Oo.copy p;;
val q : point = <obj>
```

```ocaml
# p = q, p = p;;
- : bool * bool = (false, true)
```

其他的比较操作符依然可以用在对象上（<, <=, …）。关系< 被定义为一个未确定但是严格的对象顺序。这种两个对象的顺序关系在两个对象创建以后是固定的，不受域的变化影响。

克隆和重写都有一个非空的交集。两个对象在没有重写任何域的时候是可以交换的：

```ocaml
# class copy =
    object
      method copy = {< >}
    end;;
class copy : object ('a) method copy : 'a end
```

```ocaml
# class copy =
    object (self)
      method copy = Oo.copy self
    end;;
class copy : object ('a) method copy : 'a end
```

只有重写才会用于重写域，只有 Oo.copy 才能从外部访问原始值。

克隆也可以用于为储存对象状态提供容器：

```ocaml
# class backup =
    object (self : 'mytype)
      val mutable copy = None
      method save = copy <- Some {< copy = None >}
      method restore = match copy with Some x -> x | None -> self
    end;;
class backup :
  object ('a)
    val mutable copy : 'a option
    method restore : 'a
    method save : unit
  end
```

以上的定义只会备份一层，备份的容器可以通过继承加到任何一个类上。

```ocaml
# class ['a] backup_ref x = object inherit ['a] oref x inherit backup end;;
class ['a] backup_ref :
  'a ->
  object ('b)
    val mutable copy : 'b option
    val mutable x : 'a
    method get : 'a
    method restore : 'b
    method save : unit
    method set : 'a -> unit
  end
```

```Ocaml
# let rec get p n = if n = 0 then p # get else get (p # restore) (n-1);;
val get : (< get : 'b; restore : 'a; .. > as 'a) -> int -> 'b = <fun>
```

```ocaml
# let p = new backup_ref 0  in
  p # save; p # set 1; p # save; p # set 2;
  [get p 0; get p 1; get p 2; get p 3; get p 4];;
- : int list = [2; 1; 1; 1; 1]
```

我们可以定义一个变体来储存所有的备份。（同时也加入了删除备份的方法）

```ocaml
# class backup =
    object (self : 'mytype)
      val mutable copy = None
      method save = copy <- Some {< >}
      method restore = match copy with Some x -> x | None -> self
      method clear = copy <- None
    end;;
class backup :
  object ('a)
    val mutable copy : 'a option
    method clear : unit
    method restore : 'a
    method save : unit
  end
```

```ocaml
# class ['a] backup_ref x = object inherit ['a] oref x inherit backup end;;
class ['a] backup_ref :
  'a ->
  object ('b)
    val mutable copy : 'b option
    val mutable x : 'a
    method clear : unit
    method get : 'a
    method restore : 'b
    method save : unit
    method set : 'a -> unit
  end
```

```ocaml
# let p = new backup_ref 0  in
  p # save; p # set 1; p # save; p # set 2;
  [get p 0; get p 1; get p 2; get p 3; get p 4];;
- : int list = [2; 1; 0; 0; 0]
```

## 3.15  相关类

相关类可以用于定义类型是互相相关定义的对象。

```ocaml
# class window =
    object
      val mutable top_widget = (None : widget option)
      method top_widget = top_widget
    end
  and widget (w : window) =
    object
      val window = w
      method window = window
    end;;
class window :
  object
    val mutable top_widget : widget option
    method top_widget : widget option
  end
and widget : window -> object val window : window method window : window end
```

尽管他们的类型是互相相关定义的，但是widget类和window类是各自独立的。

## 3.16  二元方法

一个接受一个相同类型参数作为self的方法叫做二元方法。下面的comparable类是一个含有二元方法leq的类，在'a -> bool中的类型变量'a和self的类型是相关联的。于是#comparable 展开为了< leq : 'a -> bool; .. >来作为 'a的类型。这里的结合也可以接受相关类型。

```ocaml
# class virtual comparable =
    object (_ : 'a)
      method virtual leq : 'a -> bool
    end;;
class virtual comparable : object ('a) method virtual leq : 'a -> bool end
```

定义comparable的一个子类money。money类简单的把浮点数包装成了comparable对象。我们将类拓展到如下代码方便有更多操作。我们必须在类参数上使用类型限制，因为在OCaml中 <= 是一个多态的函数。inherit 保证了对象的类型是#comparable类型的实例。

```ocaml
# class money (x : float) =
    object
      inherit comparable
      val repr = x
      method value = repr
      method leq p = repr <= p#value
    end;;
class money :
  float ->
  object ('a)
    val repr : float
    method leq : 'a -> bool
    method value : float
  end
```

需要注意的是，money不是comparable的一个子类型，因为self在leq的类型是逆向表示的。money类的实例m的方法leq 需要money的类型声明，因为方法需要获得值的方法。类型为comparable的m将会允许没有方法值的参数调用leq方法，但是这样会发生错误。

相似的，如下定义的类型money2不是money类型的子类型。

```ocaml
# class money2 x =
    object
      inherit money x
      method times k = {< repr = k *. repr >}
    end;;
class money2 :
  float ->
  object ('a)
    val repr : float
    method leq : 'a -> bool
    method times : float -> 'a
    method value : float
  end
```

但是我们可以定义一个方法允许接受money或者money2作为参数：下面的函数min将会返回两个对象中较小的那个，两个对象的类型是通过#comparable来定义的。但是min的类型并不是#comparable -> #comparable -> #comparable，而是只接受一次#comparable 类型后确定为'a。每一次调用函数的时候，'a都是一个重新计算的类型变量。

```ocaml
# let min (x : #comparable) y =
    if x#leq y then x else y;;
val min : (#comparable as 'a) -> 'a -> 'a = <fun>
```

这个函数可以用于money或者money2：

```Ocaml
# (min (new money  1.3) (new money 3.1))#value;;
- : float = 1.3
```

```Ocaml
# (min (new money2 5.0) (new money2 3.14))#value;;
- : float = 3.14
```

更多的二元方法的例子可以在6.2.1和6.2.3找到。

需要注意的是在多次重写方法后，当你定义了new money2 (k *. repr)而不是 {< repr = k *. repr >} ，在继承中不会正确的表现： 在money2的子类money3中，times方法将会返回一个money2的对象，而不是如所期待的返回money3的对象。

money类还有另外的二元方法，这里是直接的定义：

```Ocaml
# class money x =
    object (self : 'a)
      val repr = x
      method value = repr
      method print = print_float repr
      method times k = {< repr = k *. x >}
      method leq (p : 'a) = repr <= p#value
      method plus (p : 'a) = {< repr = x +. p#value >}
    end;;
class money :
  float ->
  object ('a)
    val repr : float
    method leq : 'a -> bool
    method plus : 'a -> 'a
    method print : unit
    method times : float -> 'a
    method value : float
  end
```

## 3.17  友类

上面的money类揭露了一个二元方法会产生的普遍问题。为了和同一个类的其他对象交互，money对象需要有一个暴露在域中的代表（一个方法值）。当我们把二元方法删除之后，这个代表可以很简单的隐藏在一个移除了方法值的类中，但是，这个方法是不可行的，因为一些二元方法要求我们在其他同个类的对象中可以访问这个代表（而不是self）。

```ocaml
# class safe_money x =
    object (self : 'a)
      val repr = x
      method print = print_float repr
      method times k = {< repr = k *. x >}
    end;;
class safe_money :
  float ->
  object ('a)
    val repr : float
    method print : unit
    method times : float -> 'a
  end
```

在这里，对象的代表只被特殊的对象知道。为了让他可以被同个类的其他对象使用， 我们只能让它对全局开放。但是，我们可以通过使用module系统来重新规划代表的可见性。

```ocaml
# module type MONEY =
    sig
      type t
      class c : float ->
        object ('a)
          val repr : t
          method value : t
          method print : unit
          method times : float -> 'a
          method leq : 'a -> bool
          method plus : 'a -> 'a
        end
    end;;
 module Euro : MONEY =
    struct
      type t = float
      class c x =
        object (self : 'a)
          val repr = x
          method value = repr
          method print = print_float repr
          method times k = {< repr = k *. x >}
          method leq (p : 'a) = repr <= p#value
          method plus (p : 'a) = {< repr = x +. p#value >}
        end
    end;;
```

另一个友方法的例子在6.2.3中可以找到。这些例子是在对象和函数需要看到彼此的代表的，但是这些代表却需要对外界隐藏的时候产生的，这些解决方法通常将所有的友类在同一个module里面定义，通过签名来限制这些代表对内对外的可见性。