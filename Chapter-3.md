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

隐藏在父类里面的私有方法是不可见的，这样它也不会被重载。因为初始化方法也是作为私有方法存在，所有的私有方法会按照继承顺序执行。

需要注意的是，为了使逻辑清晰，被重载的方法必须用!标志标记，如果之前没有定义过这个方法，那么会导致错误。

```ocaml
#   object
      method! m = ()
    end;;
Error: The method `m' has no previous definition
```

这种明确重载也可以使用在继承和值上：

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

Subtyping is never implicit. There are, however, two ways to perform subtyping. The most general construction is fully explicit: both the domain and the codomain of the type coercion must be given.

子类型是永远都是

We have seen that points and colored points have incompatible types. For instance, they cannot be mixed in the same list. However, a colored point can be coerced to a point, hiding its color method:

```
 let colored_point_to_point cp = (cp : colored_point :> point);;

val colored_point_to_point : colored_point -> point = <fun>

```

```
 let p = new point 3 and q = new colored_point 4 "blue";;

val p : point = <obj>
val q : colored_point = <obj>

```

```
 let l = [p; (colored_point_to_point q)];;

val l : point list = [<obj>; <obj>]

```

An object of type t can be seen as an object of type t' only if t is a subtype of t'. For instance, a point cannot be seen as a colored point.

```
 (p : point :> colored_point);;

Error: Type point = < get_offset : int; get_x : int; move : int -> unit >
       is not a subtype of
         colored_point =
           < color : string; get_offset : int; get_x : int;
             move : int -> unit > 

```

Indeed, narrowing coercions without runtime checks would be unsafe. Runtime type checks might raise exceptions, and they would require the presence of type information at runtime, which is not the case in the OCaml system. For these reasons, there is no such operation available in the language.

Be aware that subtyping and inheritance are not related. Inheritance is a syntactic relation between classes while subtyping is a semantic relation between types. For instance, the class of colored points could have been defined directly, without inheriting from the class of points; the type of colored points would remain unchanged and thus still be a subtype of points.

The domain of a coercion can often be omitted. For instance, one can define:

```
 let to_point cp = (cp :> point);;

val to_point : #point -> point = <fun>

```

In this case, the function colored_point_to_point is an instance of the function to_point. This is not always true, however. The fully explicit coercion is more precise and is sometimes unavoidable. Consider, for example, the following class:

```
 class c0 = object method m = {< >} method n = 0 end;;

class c0 : object ('a) method m : 'a method n : int end

```

The object type c0 is an abbreviation for <m : 'a; n : int> as 'a. Consider now the type declaration:

```
 class type c1 =  object method m : c1 end;;

class type c1 = object method m : c1 end

```

The object type c1 is an abbreviation for the type <m : 'a> as 'a. The coercion from an object of type c0 to an object of type c1 is correct:

```
 fun (x:c0) -> (x : c0 :> c1);;

- : c0 -> c1 = <fun>

```

However, the domain of the coercion cannot always be omitted. In that case, the solution is to use the explicit form. Sometimes, a change in the class-type definition can also solve the problem

```
 class type c2 = object ('a) method m : 'a end;;

class type c2 = object ('a) method m : 'a end

```

```
 fun (x:c0) -> (x :> c2);;

- : c0 -> c2 = <fun>

```

While class types c1 and c2 are different, both object types c1 and c2 expand to the same object type (same method names and types). Yet, when the domain of a coercion is left implicit and its co-domain is an abbreviation of a known class type, then the class type, rather than the object type, is used to derive the coercion function. This allows leaving the domain implicit in most cases when coercing form a subclass to its superclass. The type of a coercion can always be seen as below:

```
 let to_c1 x = (x :> c1);;

val to_c1 : < m : #c1; .. > -> c1 = <fun>

```

```
 let to_c2 x = (x :> c2);;

val to_c2 : #c2 -> c2 = <fun>

```

Note the difference between these two coercions: in the case of to_c2, the type #c2 = < m : 'a; .. > as 'a is polymorphically recursive (according to the explicit recursion in the class type of c2); hence the success of applying this coercion to an object of class c0. On the other hand, in the first case, c1 was only expanded and unrolled twice to obtain < m : < m : c1; .. >; .. >(remember #c1 = < m : c1; .. >), without introducing recursion. You may also note that the type of to_c2 is #c2 -> c2 while the type of to_c1 is more general than #c1 -> c1. This is not always true, since there are class types for which some instances of #c are not subtypes of c, as explained in section [3.16](https://caml.inria.fr/pub/docs/manual-ocaml/objectexamples.html#ss%3Abinary-methods). Yet, for parameterless classes the coercion (_ :> c) is always more general than (_ : #c :> c).

A common problem may occur when one tries to define a coercion to a class c while defining class c. The problem is due to the type abbreviation not being completely defined yet, and so its subtypes are not clearly known. Then, a coercion (_ :> c) or (_ : #c :> c) is taken to be the identity function, as in

```
 function x -> (x :> 'a);;

- : 'a -> 'a = <fun>

```

As a consequence, if the coercion is applied to self, as in the following example, the type of self is unified with the closed type c (a closed object type is an object type without ellipsis). This would constrain the type of self be closed and is thus rejected. Indeed, the type of self cannot be closed: this would prevent any further extension of the class. Therefore, a type error is generated when the unification of this type with another type would result in a closed object type.

```
 class c = object method m = 1 end
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

However, the most common instance of this problem, coercing self to its current class, is detected as a special case by the type checker, and properly typed.

```
 class c = object (self) method m = (self :> c) end;;

class c : object method m : c end

```

This allows the following idiom, keeping a list of all objects belonging to a class or its subclasses:

```
 let all_c = ref [];;

val all_c : '_weak3 list ref = {contents = []}

```

```
 class c (m : int) =
    object (self)
      method m = m
      initializer all_c := (self :> c) :: !all_c
    end;;

class c : int -> object method m : int end

```

This idiom can in turn be used to retrieve an object whose type has been weakened:

```
 let rec lookup_obj obj = function [] -> raise Not_found
    | obj' :: l ->
       if (obj :> < >) = (obj' :> < >) then obj' else lookup_obj obj l ;;

val lookup_obj : < .. > -> (< .. > as 'a) list -> 'a = <fun>

```

```
 let lookup_c obj = lookup_obj obj !all_c;;

val lookup_c : < .. > -> < m : int > = <fun>

```

The type < m : int > we see here is just the expansion of c, due to the use of a reference; we have succeeded in getting back an object of type c.

The previous coercion problem can often be avoided by first defining the abbreviation, using a class type:

```
 class type c' = object method m : int end;;

class type c' = object method m : int end

```

```
 class c : c' = object method m = 1 end
  and d = object (self)
    inherit c
    method n = 2
    method as_c = (self :> c')
  end;;

class c : c'
and d : object method as_c : c' method m : int method n : int end

```

It is also possible to use a virtual class. Inheriting from this class simultaneously forces all methods of c to have the same type as the methods of c'.

```
 class virtual c' = object method virtual m : int end;;

class virtual c' : object method virtual m : int end

```

```
 class c = object (self) inherit c' method m = 1 end;;

class c : object method m : int end

```

One could think of defining the type abbreviation directly:

```
 type c' = <m : int>;;


```

However, the abbreviation #c' cannot be defined directly in a similar way. It can only be defined by a class or a class-type definition. This is because a #-abbreviation carries an implicit anonymous variable .. that cannot be explicitly named. The closer you get to it is:

```
 type 'a c'_class = 'a constraint 'a = < m : int; .. >;;


```

with an extra type variable capturing the open object type.

## 3.13  Functional objects

It is possible to write a version of class point without assignments on the instance variables. The override construct {< ... >} returns a copy of “self” (that is, the current object), possibly changing the value of some instance variables.

```
 class functional_point y =
    object
      val x = y
      method get_x = x
      method move d = {< x = x + d >}
    end;;

class functional_point :
  int ->
  object ('a) val x : int method get_x : int method move : int -> 'a end

```

```
 let p = new functional_point 7;;

val p : functional_point = <obj>

```

```
 p#get_x;;

- : int = 7

```

```
 (p#move 3)#get_x;;

- : int = 10

```

```
 p#get_x;;

- : int = 7

```

Note that the type abbreviation functional_point is recursive, which can be seen in the class type of functional_point: the type of self is 'a and 'a appears inside the type of the method move.

The above definition of functional_point is not equivalent to the following:

```
 class bad_functional_point y =
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

While objects of either class will behave the same, objects of their subclasses will be different. In a subclass of bad_functional_point, the method move will keep returning an object of the parent class. On the contrary, in a subclass of functional_point, the method move will return an object of the subclass.

Functional update is often used in conjunction with binary methods as illustrated in section [6.2.1](https://caml.inria.fr/pub/docs/manual-ocaml/advexamples.html#module%3Astring).

## 3.14  Cloning objects

Objects can also be cloned, whether they are functional or imperative. The library function Oo.copy makes a shallow copy of an object. That is, it returns a new object that has the same methods and instance variables as its argument. The instance variables are copied but their contents are shared. Assigning a new value to an instance variable of the copy (using a method call) will not affect instance variables of the original, and conversely. A deeper assignment (for example if the instance variable is a reference cell) will of course affect both the original and the copy.

The type of Oo.copy is the following:

```
 Oo.copy;;

- : (< .. > as 'a) -> 'a = <fun>

```

The keyword as in that type binds the type variable 'a to the object type < .. >. Therefore, Oo.copy takes an object with any methods (represented by the ellipsis), and returns an object of the same type. The type of Oo.copy is different from type < .. > -> < .. > as each ellipsis represents a different set of methods. Ellipsis actually behaves as a type variable.

```
 let p = new point 5;;

val p : point = <obj>

```

```
 let q = Oo.copy p;;

val q : point = <obj>

```

```
 q#move 7; (p#get_x, q#get_x);;

- : int * int = (5, 12)

```

In fact, Oo.copy p will behave as p#copy assuming that a public method copy with body {< >} has been defined in the class of p.

Objects can be compared using the generic comparison functions = and <>. Two objects are equal if and only if they are physically equal. In particular, an object and its copy are not equal.

```
 let q = Oo.copy p;;

val q : point = <obj>

```

```
 p = q, p = p;;

- : bool * bool = (false, true)

```

Other generic comparisons such as (<, <=, ...) can also be used on objects. The relation < defines an unspecified but strict ordering on objects. The ordering relationship between two objects is fixed once for all after the two objects have been created and it is not affected by mutation of fields.

Cloning and override have a non empty intersection. They are interchangeable when used within an object and without overriding any field:

```
 class copy =
    object
      method copy = {< >}
    end;;

class copy : object ('a) method copy : 'a end

```

```
 class copy =
    object (self)
      method copy = Oo.copy self
    end;;

class copy : object ('a) method copy : 'a end

```

Only the override can be used to actually override fields, and only the Oo.copy primitive can be used externally.

Cloning can also be used to provide facilities for saving and restoring the state of objects.

```
 class backup =
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

The above definition will only backup one level. The backup facility can be added to any class by using multiple inheritance.

```
 class ['a] backup_ref x = object inherit ['a] oref x inherit backup end;;

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

```
 let rec get p n = if n = 0 then p # get else get (p # restore) (n-1);;

val get : (< get : 'b; restore : 'a; .. > as 'a) -> int -> 'b = <fun>

```

```
 let p = new backup_ref 0  in
  p # save; p # set 1; p # save; p # set 2;
  [get p 0; get p 1; get p 2; get p 3; get p 4];;

- : int list = [2; 1; 1; 1; 1]

```

We can define a variant of backup that retains all copies. (We also add a method clear to manually erase all copies.)

```
 class backup =
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

```
 class ['a] backup_ref x = object inherit ['a] oref x inherit backup end;;

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

```
 let p = new backup_ref 0  in
  p # save; p # set 1; p # save; p # set 2;
  [get p 0; get p 1; get p 2; get p 3; get p 4];;

- : int list = [2; 1; 0; 0; 0]

```

## 3.15  Recursive classes

Recursive classes can be used to define objects whose types are mutually recursive.

```
 class window =
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

Although their types are mutually recursive, the classes widget and window are themselves independent.

## 3.16  Binary methods

A binary method is a method which takes an argument of the same type as self. The class comparable below is a template for classes with a binary method leq of type 'a -> bool where the type variable 'a is bound to the type of self. Therefore, #comparable expands to < leq : 'a -> bool; .. > as 'a. We see here that the binder as also allows writing recursive types.

```
 class virtual comparable =
    object (_ : 'a)
      method virtual leq : 'a -> bool
    end;;

class virtual comparable : object ('a) method virtual leq : 'a -> bool end

```

We then define a subclass money of comparable. The class money simply wraps floats as comparable objects. We will extend it below with more operations. We have to use a type constraint on the class parameter x because the primitive <= is a polymorphic function in OCaml. The inherit clause ensures that the type of objects of this class is an instance of #comparable.

```
 class money (x : float) =
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

Note that the type money is not a subtype of type comparable, as the self type appears in contravariant position in the type of method leq. Indeed, an object m of class money has a method leqthat expects an argument of type money since it accesses its value method. Considering m of type comparable would allow a call to method leq on m with an argument that does not have a method value, which would be an error.

Similarly, the type money2 below is not a subtype of type money.

```
 class money2 x =
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

It is however possible to define functions that manipulate objects of type either money or money2: the function min will return the minimum of any two objects whose type unifies with#comparable. The type of min is not the same as #comparable -> #comparable -> #comparable, as the abbreviation #comparable hides a type variable (an ellipsis). Each occurrence of this abbreviation generates a new variable.

```
 let min (x : #comparable) y =
    if x#leq y then x else y;;

val min : (#comparable as 'a) -> 'a -> 'a = <fun>

```

This function can be applied to objects of type money or money2.

```
 (min (new money  1.3) (new money 3.1))#value;;

- : float = 1.3

```

```
 (min (new money2 5.0) (new money2 3.14))#value;;

- : float = 3.14

```

More examples of binary methods can be found in sections [6.2.1](https://caml.inria.fr/pub/docs/manual-ocaml/advexamples.html#module%3Astring) and [6.2.3](https://caml.inria.fr/pub/docs/manual-ocaml/advexamples.html#module%3Aset).

Note the use of override for method times. Writing new money2 (k *. repr) instead of {< repr = k *. repr >} would not behave well with inheritance: in a subclass money3 of money2 the timesmethod would return an object of class money2 but not of class money3 as would be expected.

The class money could naturally carry another binary method. Here is a direct definition:

```
 class money x =
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

## 3.17  Friends

The above class money reveals a problem that often occurs with binary methods. In order to interact with other objects of the same class, the representation of money objects must be revealed, using a method such as value. If we remove all binary methods (here plus and leq), the representation can easily be hidden inside objects by removing the method value as well. However, this is not possible as soon as some binary method requires access to the representation of objects of the same class (other than self).

```
 class safe_money x =
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

Here, the representation of the object is known only to a particular object. To make it available to other objects of the same class, we are forced to make it available to the whole world. However we can easily restrict the visibility of the representation using the module system.

```
 module type MONEY =
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

Another example of friend functions may be found in section [6.2.3](https://caml.inria.fr/pub/docs/manual-ocaml/advexamples.html#module%3Aset). These examples occur when a group of objects (here objects of the same class) and functions should see each others internal representation, while their representation should be hidden from the outside. The solution is always to define all friends in the same module, give access to the representation and use a signature constraint to make the representation abstract outside the module.