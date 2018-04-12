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

### 4.1.1  Optional arguments

An interesting feature of labeled arguments is that they can be made optional. For optional parameters, the question mark ? replaces the tilde ~ of non-optional ones, and the label is also prefixed by ? in the function type. Default values may be given for such optional parameters.

```
 let bump ?(step = 1) x = x + step;;

val bump : ?step:int -> int -> int = <fun>

```

```
 bump 2;;

- : int = 3

```

```
 bump ~step:3 2;;

- : int = 5

```

A function taking some optional arguments must also take at least one non-optional argument. The criterion for deciding whether an optional argument has been omitted is the non-labeled application of an argument appearing after this optional argument in the function type. Note that if that argument is labeled, you will only be able to eliminate optional arguments through the special case for total applications.

```
 let test ?(x = 0) ?(y = 0) () ?(z = 0) () = (x, y, z);;

val test : ?x:int -> ?y:int -> unit -> ?z:int -> unit -> int * int * int =
  <fun>

```

```
 test ();;

- : ?z:int -> unit -> int * int * int = <fun>

```

```
 test ~x:2 () ~z:3 ();;

- : int * int * int = (2, 0, 3)

```

Optional parameters may also commute with non-optional or unlabeled ones, as long as they are applied simultaneously. By nature, optional arguments do not commute with unlabeled arguments applied independently.

```
 test ~y:2 ~x:3 () ();;

- : int * int * int = (3, 2, 0)

```

```
 test () () ~z:1 ~y:2 ~x:3;;

- : int * int * int = (3, 2, 1)

```

```
 (test () ()) ~z:1 ;;

Error: This expression has type int * int * int
       This is not a function; it cannot be applied.

```

Here (test () ()) is already (0,0,0) and cannot be further applied.

Optional arguments are actually implemented as option types. If you do not give a default value, you have access to their internal representation, type 'a option = None | Some of 'a. You can then provide different behaviors when an argument is present or not.

```
 let bump ?step x =
    match step with
    | None -> x * 2
    | Some y -> x + y
  ;;

val bump : ?step:int -> int -> int = <fun>

```

It may also be useful to relay an optional argument from a function call to another. This can be done by prefixing the applied argument with ?. This question mark disables the wrapping of optional argument in an option type.

```
 let test2 ?x ?y () = test ?x ?y () ();;

val test2 : ?x:int -> ?y:int -> unit -> int * int * int = <fun>

```

```
 test2 ?x:None;;

- : ?y:int -> unit -> int * int * int = <fun>

```

### 4.1.2  Labels and type inference

While they provide an increased comfort for writing function applications, labels and optional arguments have the pitfall that they cannot be inferred as completely as the rest of the language.

You can see it in the following two examples.

```
 let h' g = g ~y:2 ~x:3;;

val h' : (y:int -> x:int -> 'a) -> 'a = <fun>

```

```
 h' f ;;

Error: This expression has type x:int -> y:int -> int
       but an expression was expected of type y:int -> x:int -> 'a

```

```
 let bump_it bump x =
    bump ~step:2 x;;

val bump_it : (step:int -> 'a -> 'b) -> 'a -> 'b = <fun>

```

```
 bump_it bump 1 ;;

Error: This expression has type ?step:int -> int -> int
       but an expression was expected of type step:int -> 'a -> 'b

```

The first case is simple: g is passed ~y and then ~x, but f expects ~x and then ~y. This is correctly handled if we know the type of g to be x:int -> y:int -> int in advance, but otherwise this causes the above type clash. The simplest workaround is to apply formal parameters in a standard order.

The second example is more subtle: while we intended the argument bump to be of type ?step:int -> int -> int, it is inferred as step:int -> int -> 'a. These two types being incompatible (internally normal and optional arguments are different), a type error occurs when applying bump_it to the real bump.

We will not try here to explain in detail how type inference works. One must just understand that there is not enough information in the above program to deduce the correct type of g or bump. That is, there is no way to know whether an argument is optional or not, or which is the correct order, by looking only at how a function is applied. The strategy used by the compiler is to assume that there are no optional arguments, and that applications are done in the right order.

The right way to solve this problem for optional parameters is to add a type annotation to the argument bump.

```
 let bump_it (bump : ?step:int -> int -> int) x =
    bump ~step:2 x;;

val bump_it : (?step:int -> int -> int) -> int -> int = <fun>

```

```
 bump_it bump 1;;

- : int = 3

```

In practice, such problems appear mostly when using objects whose methods have optional arguments, so that writing the type of object arguments is often a good idea.

Normally the compiler generates a type error if you attempt to pass to a function a parameter whose type is different from the expected one. However, in the specific case where the expected type is a non-labeled function type, and the argument is a function expecting optional parameters, the compiler will attempt to transform the argument to have it match the expected type, by passing None for all optional parameters.

```
 let twice f (x : int) = f(f x);;

val twice : (int -> int) -> int -> int = <fun>

```

```
 twice bump 2;;

- : int = 8

```

This transformation is coherent with the intended semantics, including side-effects. That is, if the application of optional parameters shall produce side-effects, these are delayed until the received function is really applied to an argument.

### 4.1.3  Suggestions for labeling

Like for names, choosing labels for functions is not an easy task. A good labeling is a labeling which

- makes programs more readable,
- is easy to remember,
- when possible, allows useful partial applications.

We explain here the rules we applied when labeling OCaml libraries.

To speak in an “object-oriented” way, one can consider that each function has a main argument, its *object*, and other arguments related with its action, the *parameters*. To permit the combination of functions through functionals in commuting label mode, the object will not be labeled. Its role is clear from the function itself. The parameters are labeled with names reminding of their nature or their role. The best labels combine nature and role. When this is not possible the role is to be preferred, since the nature will often be given by the type itself. Obscure abbreviations should be avoided.

```
ListLabels.map : f:('a -> 'b) -> 'a list -> 'b list
UnixLabels.write : file_descr -> buf:bytes -> pos:int -> len:int -> unit

```

When there are several objects of same nature and role, they are all left unlabeled.

```
ListLabels.iter2 : f:('a -> 'b -> 'c) -> 'a list -> 'b list -> unit

```

When there is no preferable object, all arguments are labeled.

```
BytesLabels.blit :
  src:bytes -> src_pos:int -> dst:bytes -> dst_pos:int -> len:int -> unit

```

However, when there is only one argument, it is often left unlabeled.

```
BytesLabels.create : int -> bytes

```

This principle also applies to functions of several arguments whose return type is a type variable, as long as the role of each argument is not ambiguous. Labeling such functions may lead to awkward error messages when one attempts to omit labels in an application, as we have seen with ListLabels.fold_left.

Here are some of the label names you will find throughout the libraries.

| Label | Meaning                                  |
| ----- | ---------------------------------------- |
| f:    | a function to be applied                 |
| pos:  | a position in a string, array or byte sequence |
| len:  | a length                                 |
| buf:  | a byte sequence or string used as buffer |
| src:  | the source of an operation               |
| dst:  | the destination of an operation          |
| init: | the initial value for an iterator        |
| cmp:  | a comparison function, e.g. Pervasives.compare |
| mode: | an operation mode or a flag list         |

All these are only suggestions, but keep in mind that the choice of labels is essential for readability. Bizarre choices will make the program harder to maintain.

In the ideal, the right function name with right labels should be enough to understand the function’s meaning. Since one can get this information with OCamlBrowser or the ocaml toplevel, the documentation is only used when a more detailed specification is needed.

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