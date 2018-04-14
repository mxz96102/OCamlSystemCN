# 第四章 标签与变体

本章节主要介绍OCaml 3中的新特性：标签和多态变体。

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

## 4.2  Polymorphic variants

Variants as presented in section [1.4](https://caml.inria.fr/pub/docs/manual-ocaml/coreexamples.html#s%3Atut-recvariants) are a powerful tool to build data structures and algorithms. However they sometimes lack flexibility when used in modular programming. This is due to the fact that every constructor is assigned to a unique type when defined and used. Even if the same name appears in the definition of multiple types, the constructor itself belongs to only one type. Therefore, one cannot decide that a given constructor belongs to multiple types, or consider a value of some type to belong to some other type with more constructors.

With polymorphic variants, this original assumption is removed. That is, a variant tag does not belong to any type in particular, the type system will just check that it is an admissible value according to its use. You need not define a type before using a variant tag. A variant type will be inferred independently for each of its uses.

### Basic use

In programs, polymorphic variants work like usual ones. You just have to prefix their names with a backquote character `.

```
 [`On; `Off];;

- : [> `Off | `On ] list = [`On; `Off]

```

```
 `Number 1;;

- : [> `Number of int ] = `Number 1

```

```
 let f = function `On -> 1 | `Off -> 0 | `Number n -> n;;

val f : [< `Number of int | `Off | `On ] -> int = <fun>

```

```
 List.map f [`On; `Off];;

- : int list = [1; 0]

```

[>`Off|`On] list means that to match this list, you should at least be able to match `Off and `On, without argument. [<`On|`Off|`Number of int] means that f may be applied to `Off, `On (both without argument), or `Number n where n is an integer. The >and < inside the variant types show that they may still be refined, either by defining more tags or by allowing less. As such, they contain an implicit type variable. Because each of the variant types appears only once in the whole type, their implicit type variables are not shown.

The above variant types were polymorphic, allowing further refinement. When writing type annotations, one will most often describe fixed variant types, that is types that cannot be refined. This is also the case for type abbreviations. Such types do not contain < or >, but just an enumeration of the tags and their associated types, just like in a normal datatype definition.

```
 type 'a vlist = [`Nil | `Cons of 'a * 'a vlist];;

type 'a vlist = [ `Cons of 'a * 'a vlist | `Nil ]

```

```
 let rec map f : 'a vlist -> 'b vlist = function
    | `Nil -> `Nil
    | `Cons(a, l) -> `Cons(f a, map f l)
  ;;

val map : ('a -> 'b) -> 'a vlist -> 'b vlist = <fun>

```

### Advanced use

Type-checking polymorphic variants is a subtle thing, and some expressions may result in more complex type information.

```
 let f = function `A -> `C | `B -> `D | x -> x;;

val f : ([> `A | `B | `C | `D ] as 'a) -> 'a = <fun>

```

```
 f `E;;

- : [> `A | `B | `C | `D | `E ] = `E

```

Here we are seeing two phenomena. First, since this matching is open (the last case catches any tag), we obtain the type [> `A | `B] rather than [< `A | `B] in a closed matching. Then, since x is returned as is, input and return types are identical. The notation as 'a denotes such type sharing. If we apply f to yet another tag `E, it gets added to the list.

```
 let f1 = function `A x -> x = 1 | `B -> true | `C -> false
  let f2 = function `A x -> x = "a" | `B -> true ;;

val f1 : [< `A of int | `B | `C ] -> bool = <fun>
val f2 : [< `A of string | `B ] -> bool = <fun>

```

```
 let f x = f1 x && f2 x;;

val f : [< `A of string & int | `B ] -> bool = <fun>

```

Here f1 and f2 both accept the variant tags `A and `B, but the argument of `A is int for f1 and string for f2. In f’s type `C, only accepted by f1, disappears, but both argument types appear for `A as int & string. This means that if we pass the variant tag `A to f, its argument should be *both* int and string. Since there is no such value, f cannot be applied to `A, and `B is the only accepted input.

Even if a value has a fixed variant type, one can still give it a larger type through coercions. Coercions are normally written with both the source type and the destination type, but in simple cases the source type may be omitted.

```
 type 'a wlist = [`Nil | `Cons of 'a * 'a wlist | `Snoc of 'a wlist * 'a];;

type 'a wlist = [ `Cons of 'a * 'a wlist | `Nil | `Snoc of 'a wlist * 'a ]

```

```
 let wlist_of_vlist  l = (l : 'a vlist :> 'a wlist);;

val wlist_of_vlist : 'a vlist -> 'a wlist = <fun>

```

```
 let open_vlist l = (l : 'a vlist :> [> 'a vlist]);;

val open_vlist : 'a vlist -> [> 'a vlist ] = <fun>

```

```
 fun x -> (x :> [`A|`B|`C]);;

- : [< `A | `B | `C ] -> [ `A | `B | `C ] = <fun>

```

You may also selectively coerce values through pattern matching.

```
 let split_cases = function
    | `Nil | `Cons _ as x -> `A x
    | `Snoc _ as x -> `B x
  ;;

val split_cases :
  [< `Cons of 'a | `Nil | `Snoc of 'b ] ->
  [> `A of [> `Cons of 'a | `Nil ] | `B of [> `Snoc of 'b ] ] = <fun>

```

When an or-pattern composed of variant tags is wrapped inside an alias-pattern, the alias is given a type containing only the tags enumerated in the or-pattern. This allows for many useful idioms, like incremental definition of functions.

```
 let num x = `Num x
  let eval1 eval (`Num x) = x
  let rec eval x = eval1 eval x ;;

val num : 'a -> [> `Num of 'a ] = <fun>
val eval1 : 'a -> [< `Num of 'b ] -> 'b = <fun>
val eval : [< `Num of 'a ] -> 'a = <fun>

```

```
 let plus x y = `Plus(x,y)
  let eval2 eval = function
    | `Plus(x,y) -> eval x + eval y
    | `Num _ as x -> eval1 eval x
  let rec eval x = eval2 eval x ;;

val plus : 'a -> 'b -> [> `Plus of 'a * 'b ] = <fun>
val eval2 : ('a -> int) -> [< `Num of int | `Plus of 'a * 'a ] -> int = <fun>
val eval : ([< `Num of int | `Plus of 'a * 'a ] as 'a) -> int = <fun>

```

To make this even more comfortable, you may use type definitions as abbreviations for or-patterns. That is, if you have defined type myvariant = [`Tag1 of int | `Tag2 of bool], then the pattern #myvariant is equivalent to writing (`Tag1(_ : int) | `Tag2(_ : bool)).

Such abbreviations may be used alone,

```
 let f = function
    | #myvariant -> "myvariant"
    | `Tag3 -> "Tag3";;

val f : [< `Tag1 of int | `Tag2 of bool | `Tag3 ] -> string = <fun>

```

or combined with with aliases.

```
 let g1 = function `Tag1 _ -> "Tag1" | `Tag2 _ -> "Tag2";;

val g1 : [< `Tag1 of 'a | `Tag2 of 'b ] -> string = <fun>

```

```
 let g = function
    | #myvariant as x -> g1 x
    | `Tag3 -> "Tag3";;

val g : [< `Tag1 of int | `Tag2 of bool | `Tag3 ] -> string = <fun>

```

### 4.2.1  Weaknesses of polymorphic variants

After seeing the power of polymorphic variants, one may wonder why they were added to core language variants, rather than replacing them.

The answer is twofold. One first aspect is that while being pretty efficient, the lack of static type information allows for less optimizations, and makes polymorphic variants slightly heavier than core language ones. However noticeable differences would only appear on huge data structures.

More important is the fact that polymorphic variants, while being type-safe, result in a weaker type discipline. That is, core language variants do actually much more than ensuring type-safety, they also check that you use only declared constructors, that all constructors present in a data-structure are compatible, and they enforce typing constraints to their parameters.

For this reason, you must be more careful about making types explicit when you use polymorphic variants. When you write a library, this is easy since you can describe exact types in interfaces, but for simple programs you are probably better off with core language variants.

Beware also that some idioms make trivial errors very hard to find. For instance, the following code is probably wrong but the compiler has no way to see it.

```
 type abc = [`A | `B | `C] ;;

type abc = [ `A | `B | `C ]

```

```
 let f = function
    | `As -> "A"
    | #abc -> "other" ;;

val f : [< `A | `As | `B | `C ] -> string = <fun>

```

```
 let f : abc -> string = f ;;

val f : abc -> string = <fun>

```

You can avoid such risks by annotating the definition itself.

```
 let f : abc -> string = function
    | `As -> "A"
    | #abc -> "other" ;;

Error: This pattern matches values of type [? `As ]
       but a pattern was expected which matches values of type abc
       The second variant type does not allow tag(s) `As

```

------

- [1](https://caml.inria.fr/pub/docs/manual-ocaml/lablexamples.html#text1)

  This correspond to the commuting label mode of Objective Caml 3.00 through 3.02, with some additional flexibility on total applications. The so-called classic mode (-nolabels options) is now deprecated for normal use.