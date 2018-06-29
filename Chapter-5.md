# 第五章 多态及其限制

这一章主要描述了一些关于多态函数和多态类型的进阶问题。在OCaml中，一些类型由类型检查推导而出的类型，适用性可能不如期望的那么强。这样的不适用性会影响函数副作用的交互以及多态递归确定，高阶多态的阻塞。

这一张对于这些情形给出细节，并且如果可能的话，解决了这些适用性的问题。

[TOC]

## 5.1  弱多态以及数据变化

### 5.1.1  弱多态类型

常见的不适用的例子，大多来源于多态类型与数据变化。下面的代码就是一个普通的例子：

```ocaml
# let store = ref None ;;
val store : '_weak1 option ref = {contents = None}
```

因为None的类型是'a option，函数的引用类型是'b -> 'b ref，吧么store的类型会化简为：'a option ref。但是，在推导类型中的 ' _weak1 option ref，是一种不同的类型。以 _weak前缀出现的类型变量是弱多态类型变量，可以简称为弱类型变量。弱类型变量起到了单个未知类型变量的占位符的作用。一旦 ' _weak1 代表的类型确定了，所有 ' _weak1 的相关的类型都会被所替代的类型代替。举个例子来说，我们可以定义另一个储存整数的引用：

```Ocaml
# let another_store = ref None ;;
val another_store : '_weak2 option ref = {contents = None}
```

```Ocaml
# another_store := Some 0;
  another_store ;;
- : int option ref = {contents = Some 0}
```

在another_store储存了一个整数之后，another_store的类型变为了int option ref。一般多态类型变量和弱类型之间的划分让OCaml不易发生运行错误。为了理解为什么这样会让OCaml更加稳定，考虑一个简单的函数，他是用来交换值x和store引用的值：

```ocaml
# let swap store x = match !store with
    | None -> store := Some x; x
    | Some y -> store := Some x; y;;
val swap : 'a option ref -> 'a -> 'a = <fun>
```

我们将我们的方法应用到store：

```ocaml
# let one = swap store 1
  let one_again = swap store 2
  let two = swap store 3;;
val one : int = 1
val one_again : int = 1
val two : int = 2
```

在这三次交换之后，储存的值应该是3，直到现在，问题都不大。我们可以尝试用更加有趣的值来替换3，例如一个函数：

```ocaml
# let error = swap store (fun x -> x);;
Error: This expression should not be a function, the expected type is
int
```

在这一点上，类型检查正确的说出了，不能用函数交换整数，整数应该用另一个整数来交换。类型检查让我们可以规范的改变出存在store的值：

```ocaml
# store := Some (fun x -> x);;
Error: This expression should not be a function, the expected type is
int
```

看一看store的类型，我们可以发现，store的类型'_weak1已经被int替换了。

```ocaml
# store;;
- : int option ref = {contents = Some 3}
```

所以，在我们在store里面储存了整数后，你不能用它来储存其他类型的值。更广泛的，弱类型避免了程序对于多态类型的值的过多的变化。

此外，若类型不能出现在顶级module的签名中，类型必须在编译时明确。另一方面，不同的编译单元可以用不同甚至不相容的类型来取代弱类型。出于这个原因，编译下面这段代码：

```ocaml
# let option_ref = ref None
```

会报一个编译错误：

```ocaml
Error: The type of this expression, '_weak1 option ref,
       contains type variables that cannot be generalized
```

为了解决这个错误，我们必须加上明确的类型声明：

```ocaml
# let option_ref: int option ref = ref None
```

这样的写法，是对于全局可变变量一个理想的实现。此外，这样也可以一开始确定使用的类型。如果这一点上出了错，那么会造成令人迷惑的类型错误，纠正的方法就是把其他用法都标为错误。

### 5.1.2  值的限制

判断在确切上下文的module中，哪一个多态类型应该替代哪个弱类型时一个复杂的问题。为了实现这个问题，类型系统必须处理函数隐性可变的状态。举个例子来说，下面的函数使用一个内部引用来实现一个延迟的身份函数。

```ocaml
# let make_fake_id () =
    let store = ref None in
    fun x -> swap store x ;;
val make_fake_id : unit -> 'a -> 'a = <fun>
```

```ocaml
# let fake_id = make_fake_id();;
val fake_id : '_weak3 -> '_weak3 = <fun>
```

把fake_id函数应用到不同类型时是不安全的。fake_id函数被正确的赋予了类型 '\_weak3 -> '\_weak3 而不是 'a -> 'a。与此同时，fake_id也应该也能允许使用一个本地可变的状态而不引发冲突。

为了规避这些复杂的情况，类型检查必须根据函数背后一直依赖的可变状态，考虑每一种可能返回的值，这个值会被给予弱类型。这样可变值的类型限制及其函数应用被称为值限制。需要注意的是，值限制是一种保守策略：在一些情况下，本应该可以被安全赋予多态类型的值，会被给予弱类型：

```ocaml
# let not_id = (fun x -> x) (fun x -> x);;
val not_id : '_weak4 -> '_weak4 = <fun>
```

通常，这种情况通常发生在高阶函数的使用。为了避免这个问题，我们可以通过为函数添加明确的参数来解决：

```ocaml
# let id_again = fun x -> (fun x -> x) (fun x -> x) x;;
val id_again : 'a -> 'a = <fun>
```

通过这个参数，id_again 发定义被看作是一个可以计算类型的函数。这样的操作在lambda演算中叫做eta-拓展。

### 5.1.3  松弛值限制

有另外一种通过使用类型检查实现的方法，可以从某种程度上解决不必要的弱类型。坦白来讲，我们可以证明若弱类型只出现在协变体位置，弱类型可以安全的转化为多态类型。举个例子来说，'a list 是 'a 的协变体：

```ocaml
# let f () = [];;
val f : unit -> 'a list = <fun>
```

```ocaml
# let empty = f ();;
val empty : 'a list = []
```

这里需要注意的是，empty类型推测结果是'a list而不是'_weak5 list，是因为f()函数应用的值的限制。

这种在协变体位置的类型限制转化，叫做松弛值限制。

### 5.1.4  分歧与值限制

分歧描述了类型构造器与副类型交互的行为。考虑这样一个例子，一对类型x和xy，x是xy的副类型，表示为x:>xy

```Ocaml
# type x = [ `X ];;
type x = [ `X ]
```

```Ocaml
# type xy = [ `X | `Y ];;
type xy = [ `X | `Y ]
```

因为x是xy的副类型，我们可以把一个x类型的值转化为xy：

```ocaml
# let x:x = `X;;
val x : x = `X
```

```ocaml
# let x' = ( x :> xy);;
val x' : xy = `X
```

相似的，如果我们有一个x类型的list，那么可以转化为xy类型的list，因为我们可以一个一个的转化其中的元素：

```ocaml
# let l:x list = [`X; `X];;
val l : x list = [`X; `X]
```

```ocaml
# let l' = ( l :> xy list);;
val l' : xy list = [`X; `X]
```

换句话来说，x:>xy 暗示着 x list :> xy list，所以，类型构造器 'a list 是一个参数'a 的协变体。

相反的，如果我们有一个可以接受xy类型的函数：

```ocaml
# let f: xy -> unit = function
    | `X -> ()
    | `Y -> ();;
val f : xy -> unit = <fun>
```

它也能接受x类型的值：

```ocaml
# let f' = (f :> x -> unit);;
val f' : x -> unit = <fun>
```

需要注意的是，我们可以把类型f和f'重写为下面这样：

```ocaml
# type 'a proc = 'a -> unit
    let f' = (f: xy proc :> x proc);;
type 'a proc = 'a -> unit
val f' : x proc = <fun>
```

在这个情况下，我们把x :> xy 实现成了 xy proc :> x proc。在这里，第二个子类型关系逆转了x和xy的顺序，因为类型构造器'a proc是一个参数'a 的逆变体。更一般的情况下，函数类型构造器'a -> 'b 在返回类型'b的时候是一个协变体，作为参数类型'a的时候，就是一个逆变体。

一个类型构造器也可以在一些类型参数中保持不变，不论是协变体还是逆变体。下面就是一个典型的例子

```ocaml
# let x: x ref = ref `X;;
val x : x ref = {contents = `X}
```

如果我们将x强制赋予xy ref类型的变量xy，我们可以使用xy来储存的引用'Y，然后将内容当做类型x读取其中的值，这样就会破坏类型系统。

更广泛的，一旦类型变量出现在了可变状态的位置，他就会固定下来。最终，协变体变量将永远不会表示一个可变的位置，协变体也可以被安全的被包括，如果你对这部分描述有更多的兴趣，可以咨询这一篇文章：<http://www.math.nagoya-u.ac.jp/~garrigue/papers/morepoly-long.pdf>

就这样，放松的值限制和类型参数协变量帮助我们在很多情况下避免了eta拓展。

### 5.1.5  抽象数据类型

更多地，当类型定义暴露的时候，类型检查器可以自己推断变量信息，甚至在未知类型的情况下，可以从放松的值限制里获益。但是，这样的情况不适用于我们自己定义的新的抽象类型。为了方便描述，我们定义一个类型collection：

```ocaml
# module type COLLECTION = sig
    type 'a t
    val empty: unit -> 'a t
  end
  module Implementation = struct
    type 'a t = 'a list
    let empty ()= []
  end;;

# module type COLLECTION = sig type 'a t val empty : unit -> 'a t end
module Implementation :
  sig type 'a t = 'a list val empty : unit -> 'a list end
```

```ocaml
# module List2: COLLECTION = Implementation;;
module List2 : COLLECTION
```

在这种情况下，当你强制把List1转换为COLLECTION的时候，类型检查忘记了'a List.t不是一个'a的协变体，结果，放松的值限制就不起效了。

```ocaml
#  List2.empty ();;
- : '_weak5 List2.t = <abstr>
```

为了保持放松的值限制，我们需要声明抽象类型'a COLLECTION为'a的协变体：

```ocaml
# module type COLLECTION = sig
    type +'a t
    val empty: unit -> 'a t
  end

#  module List2: COLLECTION = Implementation;;
module type COLLECTION = sig type +'a t val empty : unit -> 'a t end
module List2 : COLLECTION
```

这样我们就能恢复这个多态性质：

```ocaml
# List2.empty ();;
- : 'a List2.t = <abstr>
```

## 5.2  多态强制转换

第二类不寻常的问题直接与多态函数的类型推断有关。在一些情况下，OCaml的类型推断也许不能有足够的广度，来允许一些递归函数的定义，特别是在不寻常代数数据类型的递归函数中。

在常规的多态代数数据类型中，类型构造器的类型参数是依据声明写死的。举个例子，下面我们定义了的嵌套List：

```ocaml
#  type 'a regular_nested = List of 'a list | Nested of 'a regular_nested list
    let l = Nested[ List [1]; Nested [List[2;3]]; Nested[Nested[]] ];;
type 'a regular_nested = List of 'a list | Nested of 'a regular_nested list
val l : int regular_nested =
  Nested [List [1]; Nested [List [2; 3]]; Nested [Nested []]]
```

需要注意的是，类型构造器 regular_nested 总是作为 'a regular_nested 出现在定义附近。有了 'a 作为类型参数，就可以计算出这个递归函数的最大深度了。

```ocaml
# let rec maximal_depth = function
    | List _ -> 1
    | Nested [] -> 0
    | Nested (a::q) -> 1 + max (maximal_depth a) (maximal_depth (Nested q));;

val maximal_depth : 'a regular_nested -> int = <fun>
```

非常规的递归代数数据类型意味着，定义多态代数数据类型的类型参数左右两边是不同的。举个例子来说，定义一个对于所有嵌套的list都有着相同的深度：

```ocaml
# type 'a nested = List of 'a list | Nested of 'a list nested;;
type 'a nested = List of 'a list | Nested of 'a list nested
```

直觉上，一个类型'a nested 是一个包含list ... 的list中包含了k个嵌套list的元素。我们可以将maximal_depth函数简历在regular_depth基础上来计算他的深度k，所以首次尝试，我们这样定义：

```ocaml
# let rec depth = function
    | List _ -> 1
    | Nested n -> 1 + depth n;;
Error: This expression has type 'a list nested
       but an expression was expected of type 'a nested
       The type variable 'a occurs inside 'a list
```

这里的类型错误证实了，在定义深度期间，类型检查首先把深度赋予类型 'a -> 'b，当类型模式匹配的时候，'a -> 'b 变为了 'a nested -> 'b，然后在List分支被赋予类型后变为了 'a nested -> int。但是，在把depth类型应用于Nested分支的时候，类型检查碰到了一个错误：depth n 应用于了 'a list nested，那么它的类型一定是 'a list nested -> 'b。将这个限制与前面的限制统一的过程中发生了错误。换一句话说，这样的顶一下，递归函数depth因为非常规的类型构造器嵌套，被应用于'a 不同的类型。这就创造了一个错误，因为类型检查器中，新的类型变量'a 只是在函数depth中定义了，然而，在这里每一个depth的应用都是不同的类型变量。

### 5.2.1  明确的多态注解

对于这个复杂问题的解决方法就是为'a 写一个明确的多态注解：

```ocaml
# let rec depth: 'a. 'a nested -> int = function
    | List _ -> 1
    | Nested n -> 1 + depth n;;
val depth : 'a nested -> int = <fun>
```

```ocaml
# depth ( Nested(List [ [7]; [8] ]) );;
- : int = 2
```

在depth的类型里，'a.'a nested -> int 的类型变量 'a 是全局验证的，换句话说，'a.'a nested -> int 被所有'a解释为 ，对于所有的类型'a，深度遍历一个'a 嵌套值到整数。然而标准的'a nested -> int 类型会被解释为：定义一个类型变量'a ，然后深度遍历一个嵌套值到整数。这两种解释有两点主要的不同。首先，明确多态注解指示了类型检查器，每次都引入一个新的类型变量，这样就解决了depth函数定义的问题。

其次，这样也提醒类型检查器，这个函数的类型应该是多态的。必要的，在没有明确的多态类型注解的时候，下面的类型声明也是能完美运行的：

```ocaml
# let sum: 'a -> 'b -> 'c = fun x y -> x + y;;
val sum : int -> int -> int = <fun>
```

因为在这里'a, 'b 和 'c 表示了一个可能不是多态的类型变量。DansGuardian，这里如果你明确了一个多态类型不是多态的，这里会出错。

```ocaml
# let sum: 'a 'b 'c. 'a -> 'b -> 'c = fun x y -> x + y;;
Error: This definition has type int -> int -> int which is less general than
         'a 'b 'c. 'a -> 'b -> 'c
```

一个值得注意的地方时，你不需要完全明确depth的类型，只要将全局保证的类型变量加上注解就可以了：

```ocaml
# let rec depth: 'a. 'a nested -> _ = function
    | List _ -> 1
    | Nested n -> 1 + depth n;;
val depth : 'a nested -> int = <fun>
```

```ocaml
# depth ( Nested(List [ [7]; [8] ]) );;
- : int = 2
```

### 5.2.2  更多的例子

在明确的多态注解下，你可以实现一个只关注嵌套结构而不是元素类型的递归函数。举个例子，下面是一个更加复杂的例子来计算嵌套list的元素数：

```ocaml
# let len nested =
      let map_and_sum f = List.fold_left (fun acc x -> acc + f x) 0 in
      let rec len: 'a. ('a list -> int ) -> 'a nested -> int =
      fun nested_len n ->
        match n with
        | List l -> nested_len l
        | Nested n -> len (map_and_sum nested_len) n
      in
    len List.length nested;;
val len : 'a nested -> int = <fun>
```

```ocaml
# len (Nested(Nested(List [ [ [1;2]; [3] ]; [ []; [4]; [5;6;7]]; [[]] ])));;
- : int = 7
```

类似地，使用多于一个明确的多态类型变量有可能是必要的，就像计算一个嵌套list的长度的嵌套list：

```ocaml
# let shape n =
    let rec shape: 'a 'b. ('a nested -> int nested) ->
      ('b list list -> 'a list) -> 'b nested -> int nested
      = fun nest nested_shape ->
        function
        | List l -> raise
         (Invalid_argument "shape requires nested_list of depth greater than 1")
        | Nested (List l) -> nest @@ List (nested_shape l)
        | Nested n ->
          let nested_shape = List.map nested_shape in
          let nest x = nest (Nested x) in
          shape nest nested_shape n in
    shape (fun n -> n ) (fun l -> List.map List.length l ) n;;
val shape : 'a nested -> int nested = <fun>
```

```ocaml
# shape (Nested(Nested(List [ [ [1;2]; [3] ]; [ []; [4]; [5;6;7]]; [[]] ])));;
- : int nested = Nested (List [[2; 1]; [0; 1; 3]; [0]])
```

## 5.3  高阶多态函数

明确多态注解解决的类型推断问题只是一小部分。一个更加普遍的问题出现在多态函数作为参数传入高阶函数的时候。举个例子来说，我们可能想要计算两个嵌套list平均深度和长度：

```ocaml
# let average_depth x y = (depth x + depth y) / 2;;
val average_depth : 'a nested -> 'b nested -> int = <fun>
```

```ocaml
# let average_len x y = (len x + len y) / 2;;
val average_len : 'a nested -> 'b nested -> int = <fun>
```

```ocaml
# let one = average_len (List [2]) (List [[]]);;
val one : int = 1
```

一般情况下，我们会将上述定义分解为：

```ocaml
# let average f x y = (f x + f y) / 2;;
val average : ('a -> int) -> 'a -> 'a -> int = <fun>
```

但是average的类型比起average_len范围窄了许多，因为它需要第一个参数和第二个参数的类型都是相同的：

```ocaml
# average_len (List [2]) (List [[]]);;
- : int = 1
```

```ocaml
# average len (List [2]) (List [[]]);;
Error: This expression has type 'a list
       but an expression was expected of type int
```

就如前面多态递归的问题一样，问题主要来自于类型变量只在let定义开始的时候定义了。在我们计算f x和f y的时候，x 和 y的类型是一起明确的。为了避免这样的问题，我们需要指示类型检查器，f在第一个参数里时是多态的。也就是，我们希望average有下面的类型：

```ocaml
val average: ('a. 'a nested -> int) -> 'a nested -> 'b nested -> int
```

但是现阶段，这样的标记在OCaml里面并不存在：average已经在一个参数中有了全局验证类型'a，然而对于多态递归来说，全局验证类型是比其余类型先被引入的。这个位置的全局验证类型意味着，averange是一个二级多态函数。这样的高阶函数并不能被OCaml直接支持：二阶多态函数类型无法被类型推断确定，于是我们需要使用高阶函数来处理这些全局验证类型。

在OCaml里，有两种针对全局验证类型的解决方案，一个是全局验证的记录域：

```ocaml
# type 'a nested_reduction = { f:'elt. 'elt nested -> 'a };;
type 'a nested_reduction = { f : 'elt. 'elt nested -> 'a; }
```

```ocaml
# let boxed_len = { f = len };;
val boxed_len : int nested_reduction = {f = <fun>}
```

一个是全局验证对象方法：

```ocaml
# let obj_len = object method f:'a. 'a nested -> 'b = len end;;
val obj_len : < f : 'a. 'a nested -> int > = <obj>
```

为了解决问题，我们可以使用记录解决方案：

```ocaml
# let average nsm x y = (nsm.f x + nsm.f y) / 2 ;;
val average : int nested_reduction -> 'a nested -> 'b nested -> int = <fun>
```

或者对象解决方案：

```ocaml
# let average (obj:<f:'a. 'a nested -> _ > ) x y = (obj#f x + obj#f y) / 2 ;;
val average : < f : 'a. 'a nested -> int > -> 'b nested -> 'c nested -> int =
  <fun>
```
