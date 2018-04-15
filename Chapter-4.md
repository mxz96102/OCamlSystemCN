# 第四章 标签与变体

本章节主要介绍OCaml 3中的新特性：标签和多态变体。

[TOC]



## 4.1 标签

如果你看过标准库中的那些以标签结尾的module，你将会看到这些函数与你定义的函数有些不同的类型声明。

```ocaml
# ListLabels.map;;
- : f:('a -> 'b) -> 'a list -> 'b list = <fun>
```

```ocaml
# StringLabels.sub;;
- : string -> pos:int -> len:int -> string = <fun>
```

这样的声明形式叫做 标签（label）。标签可以用来给代码加上文档，允许更多的检查，让函数应用更具灵活性。你可以通过给参数加上前缀 ~ 来加上这样的标签：

```ocaml
# let f ~x ~y = x - y;;
val f : x:int -> y:int -> int = <fun>
```

```ocaml
# let x = 3 and y = 2 in f ~x ~y;;
- : int = 1
```

当你想要让参数变为别名变量，让标签在类型中显示的时候，你可以使用 ~name: 这样的形式，你也可以通过这样的形式，格式化传参：

```ocaml
# let f ~x:x1 ~y:y1 = x1 - y1;;
val f : x:int -> y:int -> int = <fun>
```

```ocaml
# f ~x:3 ~y:2;;
- : int = 1
```

标签和其他OCaml标识符遵守相同的规则，他们不能和预留关键字重名。

格式化参数通过各自的标签来匹配，如果没有标签匹配的话，将会当作空标签来解释。这就允许了应用中的参数交流。一个函数也可以部分的接受一个函数作为任意参数，创造出剩下参数的新函数：

```ocaml
# let f ~x ~y = x - y;;
val f : x:int -> y:int -> int = <fun>
```

```ocaml
# f ~y:2 ~x:3;;
- : int = 1
```

```ocaml
# ListLabels.fold_left;;
- : f:('a -> 'b -> 'a) -> init:'a -> 'b list -> 'a = <fun>
```

```o cam l
# ListLabels.fold_left [1;2;3] ~init:0 ~f:( + );;
- : int = 6
```

```ocaml
# ListLabels.fold_left ~init:0;;
- : f:(int -> 'a -> int) -> 'a list -> int = <fun>
```

如果有些参数共享了标签的话，他们交流的顺序是按照传承顺序来决定的，但是其他的参数依然不影响：

```ocaml
# let hline ~x:x1 ~x:x2 ~y = (x1, x2, y);;
val hline : x:'a -> x:'b -> y:'c -> 'a * 'b * 'c = <fun>
```

```ocaml
# hline ~x:3 ~y:2 ~x:5;;
- : int * int * int = (3, 5, 2)
```

作为参数匹配的规则，如果一个应用接受了所有参数，那么标签可能能会被忽略。在实践中，许多应用都是接受了所有参数的，所以标签经常会被忽略：

```ocaml
# f 3 2;;
- : int = 1
```

```ocaml
# ListLabels.map succ [1;2;3];;
- : int list = [2; 3; 4]
```

但是要注意类似于ListLabels.fold_left这样的函数，这样的函数不会有接受完参数这样的状态：

```ocaml
# ListLabels.fold_left ( + ) 0 [1;2;3];;
Error: This expression has type int -> int -> int
       but an expression was expected of type 'a list
```

当一个函数作为参数传到了高阶函数中，那么标签必须在两者的类型中都匹配，不允许加入或者删除标签：

```ocaml
# let h g = g ~x:3 ~y:2;;
val h : (x:int -> y:int -> 'a) -> 'a = <fun>
```

```ocaml
# h f;;
- : int = 1
```

```ocaml
# h ( + ) ;;
Error: This expression has type int -> int -> int
       but an expression was expected of type x:int -> y:int -> 'a
```

不过需要注意的是，当你不需要某个参数的时候，你可以使用未知匹配，但是你必须对这个参数使用标签前缀：

```ocaml
# h (fun ~x:_ ~y -> y+1);;
- : int = 3
```

### 4.1.1  可选参数

标签参数的一个有趣的特性就是，他们可以被声明为可选的。对于可选参数，我们需要使用 ? 来代替 ~ 来声明参数，声明时必须给出默认值：

```Ocaml
# let bump ?(step = 1) x = x + step;;
val bump : ?step:int -> int -> int = <fun>
```

```Ocaml
# bump 2;;
- : int = 3
```

```Ocaml
# bump ~step:3 2;;
- : int = 5
```

一个接受可选参数的函数需要至少有一个不是可选的参数。判断一个可选参数是否被忽略的标准是，这个参数后有没有出现函数的类型。需要注意如果参数加上了?标签，你只能通过一些特殊的方法来让参数不被忽略：

```ocaml
# let test ?(x = 0) ?(y = 0) () ?(z = 0) () = (x, y, z);;
val test : ?x:int -> ?y:int -> unit -> ?z:int -> unit -> int * int * int =
  <fun>
```

```Ocaml
# test ();;
- : ?z:int -> unit -> int * int * int = <fun>
```

```Ocaml
# test ~x:2 () ~z:3 ();;
- : int * int * int = (2, 0, 3)
```

可选参数也可以和不可选参数或者没加标签的参数交流，前提是他们同时被传入。自然的，可选参数并不能和独自传入的没有标签的参数交流。

```ocaml
# test ~y:2 ~x:3 () ();;
- : int * int * int = (3, 2, 0)
```

```ocaml
# test () () ~z:1 ~y:2 ~x:3;;
- : int * int * int = (3, 2, 1)
```

```ocaml
# (test () ()) ~z:1 ;;
Error: This expression has type int * int * int
       This is not a function; it cannot be applied.
```

在上列例子中，(test () ()) 已经接受完了参数(0, 0, 0)，所以更多的参数不能再被传入。

可选参数一般会由可选类型来实现。如果你没有给予一个默认值，那么你就需要通过类型None 和 Some来判断有没有传入参数。你可以在传入参数不同的情况下表现不同的行为。

```ocaml
# let bump ?step x =
    match step with
    | None -> x * 2
    | Some y -> x + y
  ;;
val bump : ?step:int -> int -> int = <fun>
```

你也可以在一个可选参数的函数基础上来调用另一个函数，依然是通过在参数前加上?前缀。这个符号在使用后就不会造成参数被直接推导为可选类型。

```ocaml
# let test2 ?x ?y () = test ?x ?y () ();;
val test2 : ?x:int -> ?y:int -> unit -> int * int * int = <fun>
```

```ocaml
# test2 ?x:None;;
- : ?y:int -> unit -> int * int * int = <fun>
```

### 4.1.2  标签和类型推导

标签和可选参数在带给编程舒适性的同时，也蕴含着语言中完全无法推测的陷阱。

下面两个例子就是这样的：

```ocaml
# let h' g = g ~y:2 ~x:3;;
val h' : (y:int -> x:int -> 'a) -> 'a = <fun>
```

```ocaml
# h' f ;;
Error: This expression has type x:int -> y:int -> int
       but an expression was expected of type y:int -> x:int -> 'a
```

```ocaml
# let bump_it bump x =
    bump ~step:2 x;;
val bump_it : (step:int -> 'a -> 'b) -> 'a -> 'b = <fun>
```

```ocaml
# bump_it bump 1 ;;
Error: This expression has type ?step:int -> int -> int
       but an expression was expected of type step:int -> 'a -> 'b
```

在第一个例子中，g传递来先~y后~x两个参数，但是f确实期待着先~x后~y的。但是如果我们提前知道了类型为  x:int -> y:int -> int 就可以正确处理这种情况。最简单的变通方法就是按照顺序来传入参数。

第二个例子就更加微妙了，在我们试图让bump的类型为 ?step:int -> int -> int 的同时，它却被推导为了 step:int -> int -> 'a。这两个类型是不相容的（一般参数和可选参数是不同的），所以在我们使用真的bump类型的时候就会发生类型错误。

在这里提出这些的目的并不是要解释类型推导的工作细节。一个必须理解的事情是，不管是g还是bump在编写使用他们的函数的时候，都没有足够的信息来推导出他们的类型的。的确，在编写过程中是无法看一眼参数，就知道这些参数是否可选，他们的顺序是怎样的。编译器使用的策略就是假设参数里面没有可选参数，所有参数都按照正确顺序接受。

这个问题正确的解决方法是给函数加上类型声明：

```Ocaml
# let bump_it (bump : ?step:int -> int -> int) x =
    bump ~step:2 x;;
val bump_it : (?step:int -> int -> int) -> int -> int = <fun>
```

```Ocaml
# bump_it bump 1;;
- : int = 3
```

在实践中，这样的问题一般在使用对象作为可选参数时候出现，所以在参数中声明对象类型是个好办法。

如果你要把一个不同的类型传入参数，编译器通常会报错。但是，在一个特殊的情况，如果期待的类型是一个没有标签的函数类型，而且传入的参数是一个期待可选参数的函数，编译器就会尝试去把他们的类型转换为匹配的，对所有的可选参数传递None。

```ocaml
# let twice f (x : int) = f(f x);;
val twice : (int -> int) -> int -> int = <fun>
```

```ocaml
# twice bump 2;;
- : int = 8
```

这样的转换和预期的语义是一致的，包括在副作用方面。也就是说，如果在使用可选参数的时候产生了副作用，那么这些副作用会被延迟到函数接受了参数的时候。

### 4.1.3  对于标签的建议

和命名一样，选择正确的函数标签也不是一个容易的任务，一个好的标签满足：

- 让程序更具可读性
- 容易记住
- 在可能时，允许不完整的应用存在

我们可以通过OCaml库中的标签来解释这些情况。

为了以一个面向对象的角度来叙述，可以假设每一个函数都有一个主要的参数，它的对象，其他的参数都是和他的行为有关的。为了允许函数的功能性结合以及通过标签交流，对象本身并没有标签，它的角色从函数本身剥离了出来。那些有着标签的参数，标签代表了他们的性质或者作用。最好的标签结合了这两者。当不可能兼得的时候，因为性质一般从类型看出来，我们需要依照作用来作为标签。不过，要尽量避免使用过于复杂的缩写。

```ocaml
ListLabels.map : f:('a -> 'b) -> 'a list -> 'b list
UnixLabels.write : file_descr -> buf:bytes -> pos:int -> len:int -> unit
```

当几个对象有着同样的作用和性质的时候，一般不会有标签。

```ocmal
ListLabels.iter2 : f:('a -> 'b -> 'c) -> 'a list -> 'b list -> unit
```

当没有一个合适的对象的时候，所有的参数都有标签。

```ocaml
BytesLabels.blit :
  src:bytes -> src_pos:int -> dst:bytes -> dst_pos:int -> len:int -> unit
```

但是，只有一个参数的时候，通常没有标签：

```ocaml
BytesLabels.create : int -> bytes
```

只要参数之间的作用没有歧义，这些准则通常对于类型是一个类型变量的函数适用。标记这样的参数通常会在忽略导致一些尴尬的错误信息。比如我们在 ListLabels.fold_left 里面看到的那样。

这些是一些库中常用的标签：

| 标签  | 意义                                   |
| ----- | -------------------------------------- |
| f:    | 接受一个函数                           |
| pos:  | 在string array 或者bytes里面的一个位置 |
| len:  | 长度                                   |
| buf:  | 作为buffer的string或者bytes            |
| src:  | 操作源                                 |
| dst:  | 操作的目的地                           |
| init: | 初始值                                 |
| cmp:  | 比较函数                               |
| mode: | 操作模式，或是模式的一个list           |

当然，这些都只是建议，要铭记在心的是，选择标签是基于可读性的。越是古怪的选择，越是让程序难以维护。

在这些建议中，正确的函数名和正确的标签应该要能够理解函数的意义。因为这些信息可以通过OCamlBrowser和ocaml toplevel来得到，文档只是为了更细节的描述而使用的。

## 4.2  多态变体

如同在1.4节中展示的那样，变体是一个用来构建数据结构和算法的强有力的工具。但是，有些时候它们对于模型化编程缺乏灵活性。这是因为每个构造器都要被分配一个独特的类型来定义和使用美好。甚至是在一个多类型的定义中，每一个命名类型都会被赋予属于单个的构造器。于是，当不能决定构造器是属于哪个多类型的时候，或者一个类型的值同时也属于其他类型的时候，变成了很复杂的情况。

通过使用后多态变体，这个假设就被解决了。也就是说，一个变体标记不属于任何的类型，类型系统只会检查他能否采纳这个类型来使用。在使用变体标记之前，你不需要定义类型。变体类型会在每一次使用中被独立的推导。

### 基础用法

在程序中，多态变体和一般的变体差不多，你只需要在命名时前面加上`前缀。

```Ocaml
# [`On; `Off];;
- : [> `Off | `On ] list = [`On; `Off]
```

```Ocaml
# `Number 1;;
- : [> `Number of int ] = `Number 1
```

```Ocaml
# let f = function `On -> 1 | `Off -> 0 | `Number n -> n;;
val f : [< `Number of int | `Off | `On ] -> int = <fun>
```

```Ocaml
# List.map f [`On; `Off];;
- : int list = [1; 0]
```

[> \` Off| \`On] 的意思是，如果你要匹配这个list，你至少需要\` Off 或者 \`On来匹配。[<\`On|\`Off|\`Number of int] 的意思是，f 可以接受的参数是 不需要参数的 \`On | \`Off 或者是接受整型的 \`Number。在变体类型中的 > 和< 表示它们依然可以继续改变，可以接受更少的类型或者定义更多类型。如此，它们包含了一个内含的类型变量。因为每一个变体类型只在整个类型中出现一次，它们所隐含的类型变量就不会出现。

上面代码的变体类型都是多态，并且允许将来的重写的。当在书写类型声明的时候，如果描述的是固定的变体类型的时候，类型是不能被重写的。这对于缩略类型也依然适用。这样的类型不包含> 或者 <， 只是封装了相关类型好，就像普通的数据类型定义：

```Ocaml
# type 'a vlist = [`Nil | `Cons of 'a * 'a vlist];;
type 'a vlist = [ `Cons of 'a * 'a vlist | `Nil ]
```

```Ocaml
# let rec map f : 'a vlist -> 'b vlist = function
    | `Nil -> `Nil
    | `Cons(a, l) -> `Cons(f a, map f l)
  ;;
val map : ('a -> 'b) -> 'a vlist -> 'b vlist = <fun>
```

### 进阶用法

附带类型检查的多态变体是一种精妙的用法。一些这样的表达式会导致更加复杂的类型信息。

```
# let f = function `A -> `C | `B -> `D | x -> x;;
val f : ([> `A | `B | `C | `D ] as 'a) -> 'a = <fun>
```

```ocaml
# f `E;;
- : [> `A | `B | `C | `D | `E ] = `E
```

这里我们可以目睹两个现象。首先，因为这个匹配是开放的（最后一个可以匹配任意标签），我们有了类型 [> \`A | \`B] 而不是 [< \`A | \`B]的闭合匹配。其次，因为x返回自身，输入类型和输出类型都是独特的。>符号就是用来表示这样的类型拓展的，如果我们把 \`E加入到其中，会有新的一组类型。

```ocaml
# let f1 = function `A x -> x = 1 | `B -> true | `C -> false
  let f2 = function `A x -> x = "a" | `B -> true ;;
val f1 : [< `A of int | `B | `C ] -> bool = <fun>
val f2 : [< `A of string | `B ] -> bool = <fun>
```

```Ocaml
# let f x = f1 x && f2 x;;
val f : [< `A of string & int | `B ] -> bool = <fun>
```

这里的f1和f2都接受了变体\`A和\`B，但是\`A的声明在f1中是整数，而在f2中是string。在f的的类型中，\`C只被f1接受，所以消失了，而A由int和string来作为参数类型。这意味着，如果我们传递\`A 给变体f，它的必须既是int又是string，但是由于没有这样的值存在，f无法应用于\`A，也只接受输入为\`B。

甚至一个值有固定的变体类型，那么就不能通过更大的强制类型转换得到。强制类型转换，一般会书写源类型和目标类型，但是在例子中，源类型可以被忽略。

```
# type 'a wlist = [`Nil | `Cons of 'a * 'a wlist | `Snoc of 'a wlist * 'a];;
type 'a wlist = [ `Cons of 'a * 'a wlist | `Nil | `Snoc of 'a wlist * 'a ]
```

```ocaml
# let wlist_of_vlist  l = (l : 'a vlist :> 'a wlist);;
val wlist_of_vlist : 'a vlist -> 'a wlist = <fun>
```

```ocaml
# let open_vlist l = (l : 'a vlist :> [> 'a vlist]);;
val open_vlist : 'a vlist -> [> 'a vlist ] = <fun>
```

```ocaml
# fun x -> (x :> [`A|`B|`C]);;
- : [< `A | `B | `C ] -> [ `A | `B | `C ] = <fun>
```

你也可以使用模式匹配来选择性的强制转换类型：

```ocaml
# let split_cases = function
    | `Nil | `Cons _ as x -> `A x
    | `Snoc _ as x -> `B x
  ;;
val split_cases :
  [< `Cons of 'a | `Nil | `Snoc of 'b ] ->
  [> `A of [> `Cons of 'a | `Nil ] | `B of [> `Snoc of 'b ] ] = <fun>
```

当一群变体标签被包装在或模式，他们的别名只给出变体标签封装后的类型。这样的特性就可以书写很多用法，例如递增的定义函数：

```ocaml
# let num x = `Num x
  let eval1 eval (`Num x) = x
  let rec eval x = eval1 eval x ;;
val num : 'a -> [> `Num of 'a ] = <fun>
val eval1 : 'a -> [< `Num of 'b ] -> 'b = <fun>
val eval : [< `Num of 'a ] -> 'a = <fun>
```

```ocaml
# let plus x y = `Plus(x,y)
  let eval2 eval = function
    | `Plus(x,y) -> eval x + eval y
    | `Num _ as x -> eval1 eval x
  let rec eval x = eval2 eval x ;;
val plus : 'a -> 'b -> [> `Plus of 'a * 'b ] = <fun>
val eval2 : ('a -> int) -> [< `Num of int | `Plus of 'a * 'a ] -> int = <fun>
val eval : ([< `Num of int | `Plus of 'a * 'a ] as 'a) -> int = <fun>
```

为了让这种写法更舒适，你可以使用类型定义简写来做或匹配。这就是说，如果你定义了一个类型 myvariant = [\`Tag1 of int | \`Tag2 of bool]，那么类型#myvariant等于在写(\`Tag1 of int | \`Tag2 of bool)。

这用的缩写可以单独使用：

```ocaml
# let f = function
    | #myvariant -> "myvariant"
    | `Tag3 -> "Tag3";;
val f : [< `Tag1 of int | `Tag2 of bool | `Tag3 ] -> string = <fun>
```

或者和其他别名结合：

```Ocaml
# let g1 = function `Tag1 _ -> "Tag1" | `Tag2 _ -> "Tag2";;
val g1 : [< `Tag1 of 'a | `Tag2 of 'b ] -> string = <fun>
```

```Ocaml
# let g = function
    | #myvariant as x -> g1 x
    | `Tag3 -> "Tag3";;
val g : [< `Tag1 of int | `Tag2 of bool | `Tag3 ] -> string = <fun>
```

### 4.2.1  多态变体的弱点

在我们看到多态变体的威力以后，你也许会想为什么多态变体只是被加入核心语言变体，而没有代替全部核心类型。

答案是双方面的。一方面，虽然这样是高效的，但是缺乏静态类型信息会影响优化，多态变体会比核心语言的类型稍稍大一点。更不用提在使用更大型数据的时候出现的差别了。

更加重要的一点是，多态变体虽然是类型安全的，但是是一种更弱的类型准则。也就是说，核心语言类型做了比保证类型安全更多的工作，他们也检查了你是否只声明了构造器，所有在数据结构中呈现的构造器都是合适的，他们强制保证了参数的类型限制。

因为这样，你必须在使用多态变体的时候更加让类型明确。当你在写库的时候，因为要在借口中描述确切类型，所以不用担心这个，但是对于一般程序的书写，最好还是正对核心语言的变体来使用：

同时也要注意，一些用法会造成极难发现的错误，举个例子来说，下列的代码可能是错误的，但是编译器没办法发现：

```Ocaml
# type abc = [`A | `B | `C] ;;
type abc = [ `A | `B | `C ]
```

```Ocaml
# let f = function
    | `As -> "A"
    | #abc -> "other" ;;
val f : [< `A | `As | `B | `C ] -> string = <fun>
```

```Ocaml
# let f : abc -> string = f ;;
val f : abc -> string = <fun>
```

You can avoid such risks by annotating the definition itself.

```Ocaml
# let f : abc -> string = function
    | `As -> "A"
    | #abc -> "other" ;;
Error: This pattern matches values of type [? `As ]
       but a pattern was expected which matches values of type abc
       The second variant type does not allow tag(s) `As
```
