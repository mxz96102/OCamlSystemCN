# 第一章 核心语言部分

本章节主要介绍OCaml语言的基础入门部分，从最简单的数学运算，到最后的一个简单的编译器，读者可以通过实践对OCaml语言有一个基础认识。

## 1.1  基础部分

首先，为了让大家对OCaml语言有一个很直接的了解，本书中的大部分代码都可以通过命令行工具来实时实践。如果你使用的是Linux或者Unix，那么在命令行中输入`ocaml`命令就可以开始实践，若你使用的是windows系统，则是通过OCamlwin.exe来运行相应的功能。

本教程的代码段将会同时展示输入和输出的代码。`#`开头的代码段落为输入，随后的段落表示相应的输出。在这个命令行工具中，用户输入的OCaml语句会被`;;`结束，然后输出相应的结果。这个工具是实时编译执行，实时输出结果的。

在OCaml语言中，语句一般是表达式或者是`let`定义式（用于定义值或者函数）。

```ocaml
# 1+2*3;;
- : int = 7
```

```ocaml
# let pi = 4.0 *. atan 1.0;;
val pi : float = 3.14159265358979312
```

```Ocaml
# let square x = x *. x;;
val square : float -> float = <fun>
```

```ocaml
# square (sin pi) +. square (cos pi);;
- : float = 1.
```

OCaml 系统会得出每一个语句的值及其类型，包括函数的参数都会有详细的类型声明。系统根据他们在函数之中的使用来推断类型，或者根据你书写的声明来得出类型。需要注意的是，在OCaml中，整数和浮点数是两个分开的类型，我们使用`+`和`*`来操作整数，`+.`和`*.`来操作浮点数。

```Ocaml
# 1.0 * 2;;
Error: This expression has type float but an expression was expected of type int
(*表达式包含float类型，但是表达式要求int类型的参数*)
```

使用`rec`关键字来声明递归函数：

```ocaml
# let rec fib n =
    if n < 2 then n else fib (n-1) + fib (n-2);;
val fib : int -> int = <fun>
```

```ocaml
# fib 10;;
- : int = 55
```

## 1.2  数据类型

除了整数和浮点数，OCaml还提供了一些其他的基本数据类型：布尔值，字符，和字符串（不可改变的）。

```ocaml
# (1 < 2) = false;;
- : bool = false
```

```ocaml
# 'a';;
- : char = 'a'
```

```ocaml
# "Hello world";;
- : string = "Hello world"
```

OCaml内建的数据结构包括了`turples`，`array`，`list`。你也可以定义你自己的数据结构，在之后的章节将会有更多相关细节。但是现在，我们先来看看`list`。

list一般是用中括号括起来，用分号彼此分开声明的一串相同数据类型的数据，或者直接用`[]`（称作‘nil’），通过`::` 操作符加上数据。

```ocaml
# let l = ["is"; "a"; "tale"; "told"; "etc."];;
val l : string list = ["is"; "a"; "tale"; "told"; "etc."]
(*直接书写的模式*)
```

```ocaml
# "Life" :: l;;
- : string list = ["Life"; "is"; "a"; "tale"; "told"; "etc."]
(*通过::来创建的模式*)
```

其他OCaml 数据结构和`list`类似，都不需要单独的为他们申请内存空间，内存管理也自动的由OCaml执行。OCaml也不用单独处理指针，在编译的时候，OCaml会在必要时静默的引入指针。

其他数据结构也能和`list`一样，通过模式识别来监听和销毁。list的模式和list的表达式是一样的，可以通过一些标识符来代表一些泛指的区域。

下面是一个插入排序的例子：

```ocaml
# let rec sort lst =
    match lst with
      [] -> []
    | head :: tail -> insert head (sort tail)
  and insert elt lst =
    match lst with
      [] -> [elt]
    | head :: tail -> if elt <= head then elt :: lst else head :: insert elt tail
  ;;
val sort : 'a list -> 'a list = <fun>
val insert : 'a -> 'a list -> 'a list = <fun>
```

```ocaml
# sort l;;
- : string list = ["a"; "etc."; "is"; "tale"; "told"]
```

上面所写排序函数的类型推断为：`'a list -> 'a list`，这就意味着`sort`函数可以应用于任何包含同类型变量的`list`，类型 `'a` 是一个 `a` 类型变量，可以代表任何一个给定的类型。这个排序可以使用到任意类型`list`的原因是，比较操作符在OCaml里面实现了多态：他可以接收任意两个相同类型的变量，并返回布尔值。这就使得排序函数也是多态的。

```ocaml
# sort [6;2;5;3];;
- : int list = [2; 3; 5; 6]
```

```ocaml
# sort [3.14; 2.718];;
- : float list = [2.718; 3.14]
```

排序算法并没有修改原来的`list`，它返回了一个包含了和输入`list`具有同样元素的`list`。实际上在OCaml之中，`list`在创建之后就无法再被改变了，是一种不可变的数据结构。大部分OCaml数据结构都是不可变的，但是在OCaml中也存在一些数据结构（主要是数组）是可变的。



## 1.3  函数亦是值

OCaml是一门函数式编程语言：函数应该和其他值一样，可以作为数据，并且可以被自由的传递。举个例子来说，这里有一个 `deriv` 函数接收任意浮点数函数作为参数，并且返回计算结果：

```Ocaml
# let deriv f dx = function x -> (f (x +. dx) -. f x) /. dx;;
val deriv : (float -> float) -> float -> float -> float = <fun>
```

```Ocaml
# let sin' = deriv sin 1e-6;;
val sin' : float -> float = <fun>
```

```Ocaml
# sin' pi;;
- : float = -1.00000000013961143
```

就连函数也可以直接作为值定义：

```ocaml
# let compose f g = function x -> f (g x);;
val compose : ('a -> 'b) -> ('c -> 'a) -> 'c -> 'b = <fun>
```

```ocaml
# let cos2 = compose square cos;;
val cos2 : float -> float = <fun>
```

接受函数作为参数的函数叫做“高阶函数”，高阶函数在遍历和操作数据结构方面有很大的用处。举个例子，OCaml标准库提供了一个`List.map` 函数来接受一个函数遍历`List`，并返回结果。

```ocaml
# List.map (function n -> n * 2 + 1) [0;1;2;3;4];;
- : int list = [1; 3; 5; 7; 9]
```

这个实用的高阶函数，已经被预定义为`list`和`array`的库函数，但是其实它也不是很高深的技巧，可以很简单的定义为：

```ocaml
# let rec map f l =
    match l with
      [] -> []
    | hd :: tl -> f hd :: map f tl;;
val map : ('a -> 'b) -> 'a list -> 'b list = <fun>
```

## 1.4  Records 和 变体

用户自定义的数据结构主要包含 records 和 变体（variant），他们都是使用type来声明的，例如定义一个record类型来代表无理数。

```ocaml
# type ratio = {num: int; denom: int};;
type ratio = { num : int; denom : int; }
```

```Ocaml
# let add_ratio r1 r2 =
    {num = r1.num * r2.denom + r2.num * r1.denom;
     denom = r1.denom * r2.denom};;
val add_ratio : ratio -> ratio -> ratio = <fun>
```

```Ocaml
# add_ratio {num=1; denom=3} {num=2; denom=5};;
- : ratio = {num = 11; denom = 15}
```

Record同样也能被模式识别。

```OCaml
# let integer_part r =
    match r with
      {num=num; denom=denom} -> num / denom;;
val integer_part : ratio -> int = <fun>
```

因为模式识别已经指出了这是唯一的一种情况，我们可以把r代替为record：

```ocaml
# let integer_part {num=num; denom=denom} = num / denom;;
val integer_part : ratio -> int = <fun>
```

没有用到的值可以被省略：

```ocaml
# let get_denom {denom=denom} = denom;;
val get_denom : ratio -> int = <fun>
```

同样的，没有用到的值也可以通过 _ 通配符来省略：

```ocaml
# let get_num {num=num; _ } = num;;
val get_num : ratio -> int = <fun>
```

如果 = 两边都变量名称是相同的，我们可以采用更简单的写法：

```ocaml
# let integer_part {num; denom} = num / denom;;
val integer_part : ratio -> int = <fun>
```

这种简写的方法同样适用于使用函数来构造record：

```ocaml
# let ratio num denom = {num; denom};;
val ratio : int -> int -> ratio = <fun>
```

你也可以一次性的改变record的内容：

```ocaml
# let integer_product integer ratio = { ratio with num = integer * ratio.num };;
val integer_product : int -> ratio -> ratio = <fun>
```

`with`这个函数符号，接受左边的值并且复制后更新右边的量，最终返回新的record。

变体的声明则是列出了该类型所有可能的值，每一种都由构造器和它所对应的实际类型来对应，构造器名称通过开头大写来与变量名做出区分。

```ocaml
# type number = Int of int | Float of float | Error;;
type number = Int of int | Float of float | Error
```

上面的声明表示number类型的值可以是整型，浮点数或者一个常量Error（通常是非法操作获得的，比如除以0）。

枚举类型是一种变体的特殊用法：

```ocaml
# type sign = Positive | Negative;;
type sign = Positive | Negative
```

```ocaml
# let sign_int n = if n >= 0 then Positive else Negative;;
val sign_int : int -> sign = <fun>
```

为了定义number类型的算术操作，我们使用模式识别来判断两个输入number：

```ocaml
# let add_num n1 n2 =
    match (n1, n2) with
      (Int i1, Int i2) ->
        (* 检查加法有没有溢出 *)
        if sign_int i1 = sign_int i2 && sign_int (i1 + i2) <> sign_int i1
        then Float(float i1 +. float i2)
        else Int(i1 + i2)
    | (Int i1, Float f2) -> Float(float i1 +. f2)
    | (Float f1, Int i2) -> Float(f1 +. float i2)
    | (Float f1, Float f2) -> Float(f1 +. f2)
    | (Error, _) -> Error
    | (_, Error) -> Error;;
val add_num : number -> number -> number = <fun>
```

```ocaml
# add_num (Int 123) (Float 3.14159);;
- : number = Float 126.14159
```

另一个有趣的例子就是，OCaml内建的可选类型，它用于表示是返回了一个值还是没有返回值：

```ocaml
# type 'a option = Some of 'a | None;;
type 'a option = Some of 'a | None
```

这个类型对于有异常情况的函数有很大作用，例如：

```ocaml
# let safe_square_root x = if x > 0. then Some(sqrt x) else None;;
val safe_square_root : float -> float option = <fun>
(*这里避免了开方负数所会抛出的错误*)
```

变体类型经常被用在描述递归数据结构上，例如一个二叉树：

```ocaml
# type 'a btree = Empty | Node of 'a * 'a btree * 'a btree;;
type 'a btree = Empty | Node of 'a * 'a btree * 'a btree
```

这个定义的意思是：一个二叉树节点可以是`Empty`，也可以是一个可选类型的值，和两个该类型的节点。

同样对于二叉树的操作本来就是以递归的方式定义的，举个例子，在二分树进行查找和插入：

```ocaml
# let rec member x btree =
    match btree with
      Empty -> false
    | Node(y, left, right) ->
        if x = y then true else
        if x < y then member x left else member x right;;
val member : 'a -> 'a btree -> bool = <fun>
```

```ocaml
# let rec insert x btree =
    match btree with
      Empty -> Node(x, Empty, Empty)
    | Node(y, left, right) ->
        if x <= y then Node(y, insert x left, right)
                  else Node(y, left, insert x right);;
val insert : 'a -> 'a btree -> 'a btree = <fun>
```

## 1.5  命令式编程

到此为止的很多例子，我们都是使用纯声明式编程来写的。OCaml同时也具有完整的命令式编程的特性。包括但不限于，while和for循环，可变的数据结构array。array被以`[|`和`|]`声明，他也可以使用`Array.make`函数来创建一个array。举个例子，下面就是两个向量的加法：

```ocaml
# let add_vect v1 v2 =
    let len = min (Array.length v1) (Array.length v2) in
    let res = Array.make len 0.0 in
    for i = 0 to len - 1 do
      res.(i) <- v1.(i) +. v2.(i)
    done;
    res;;
val add_vect : float array -> float array -> float array = <fun>
```

```ocaml
# add_vect [| 1.0; 2.0 |] [| 3.0; 4.0 |];;
- : float array = [|4.; 6.|]
```

record类型同样也可以被改变，这里提供了可变化的声明来定义：

```OCaml 
# type mutable_point = { mutable x: float; mutable y: float };;

type mutable_point = { mutable x : float; mutable y : float; }
```

```Ocaml
# let translate p dx dy =
    p.x <- p.x +. dx; p.y <- p.y +. dy;;
val translate : mutable_point -> float -> float -> unit = <fun>
```

```Ocaml
# let mypoint = { x = 0.0; y = 0.0 };;
val mypoint : mutable_point = {x = 0.; y = 0.}
```

```Ocaml
# translate mypoint 1.0 2.0;;
- : unit = ()
```

```Ocaml
# mypoint;;
- : mutable_point = {x = 1.; y = 2.}
```

OCaml 有内建的变量概念，变量可以被赋值（let 绑定并不是一种赋值语句，他只是把新的标记符带进作用域）。然而，标准库提供了引用，它是一个不确定的单元（可以理解为单元素数组），通过 `! `来得到变量的内容，通过 `:=` 来赋值变量的内容。通过引用来获取变量内容并且修改内容，下面是一个插入排序的例子：

```ocaml
# let insertion_sort a =
    for i = 1 to Array.length a - 1 do
      let val_i = a.(i) in
      let j = ref i in
      while !j > 0 && val_i < a.(!j - 1) do
        a.(!j) <- a.(!j - 1);
        j := !j - 1
      done;
      a.(!j) <- val_i
    done;;
val insertion_sort : 'a array -> unit = <fun>
```

引用对于包含外来状态的函数有很大作用，例如下面这个生成伪随机数的例子，最后返回的是引用的值：

```ocaml
# let current_rand = ref 0;;
val current_rand : int ref = {contents = 0}
```

```ocaml
# let random () =
    current_rand := !current_rand * 25713 + 1345;
    !current_rand;;
val random : unit -> int = <fun>
```

当然，引用也不是什么奇技淫巧，我们也可以通过可变变量的record来实现：

```ocaml
# type 'a ref = { mutable contents: 'a };;
type 'a ref = { mutable contents : 'a; }
```

```ocaml
# let ( ! ) r = r.contents;;
val ( ! ) : 'a ref -> 'a = <fun>
```

```ocaml
# let ( := ) r newval = r.contents <- newval;;
val ( := ) : 'a ref -> 'a -> unit = <fun>
```

在一些特殊情况下，你需要储存一些多态的函数在数据结构中，以保证它的多态。但是，OCaml不允许没有类型声明的值，这时候你可以使用多态的类型来达到这个目的：

```ocaml
# type idref = { mutable id: 'a. 'a -> 'a };;
type idref = { mutable id : 'a. 'a -> 'a; }
```

```ocaml
# let r = {id = fun x -> x};;
val r : idref = {id = <fun>}
```

```ocaml
# let g s = (s.id 1, s.id true);;
val g : idref -> int * bool = <fun>
```

```ocaml
# r.id <- (fun x -> print_string "called id\n"; x);;
- : unit = ()
```

```ocaml
# g r;;
called id
called id
- : int * bool = (1, true)
```

## 1.6  异常

OCaml 提供异常来作为信号，以应对非常规的情况。异常同样也可以用在一些远程请求控制结构上。异常通过`exception`构造器构造，通过`raise`操作符来发出信号。举个例子，下面的函数对空`list`进行了处理：

```ocaml
# exception Empty_list;;
exception Empty_list
```

```ocaml
# let head l =
    match l with
      [] -> raise Empty_list
    | hd :: tl -> hd;;
val head : 'a list -> 'a = <fun>
```

```ocaml
# head [1;2];;
- : int = 1
```

```ocaml
# head [];;
Exception: Empty_list.
```

异常被用于整个标准库，用来警示那些库函数不能正常完成的情况。举个例子，`List.assoc` 函数，返回在pair list中指定键的值，如果没有找到，就会有异常：`Not_found`：

```ocaml
# List.assoc 1 [(0, "zero"); (1, "one")];;
- : string = "one"
```

```ocaml
# List.assoc 2 [(0, "zero"); (1, "one")];;
Exception: Not_found.
```

异常可以被`try`关键字捕获：

```ocaml
# let name_of_binary_digit digit =
    try
      List.assoc digit [0, "zero"; 1, "one"]
    with Not_found ->
      "not a binary digit";;
val name_of_binary_digit : int -> string = <fun>
```

```ocaml
# name_of_binary_digit 0;;
- : string = "zero"
```

```ocaml
# name_of_binary_digit (-1);;
- : string = "not a binary digit"
```

`with`部分实际上还是一般的对异常值进行模式匹配，我们可以通过下面这种方式来捕获所有异常，进行操作，最终再raise异常：

```ocaml
# let temporarily_set_reference ref newval funct =
    let oldval = !ref in
    try
      ref := newval;
      let res = funct () in
      ref := oldval;
      res
    with x ->
      ref := oldval;
      raise x;;
val temporarily_set_reference : 'a ref -> 'a -> (unit -> 'b) -> 'b = <fun>
```

## 1.7   表达式词法分析（*）

我们将以更具代表性的例子来解释OCaml中词法分析的编写：我们将编写一个计算方程的程序，下列是将会用到的符号定义：

```ocaml
# type expression =
      Const of float
    | Var of string
    | Sum of expression * expression    (* e1 + e2 *)
    | Diff of expression * expression   (* e1 - e2 *)
    | Prod of expression * expression   (* e1 * e2 *)
    | Quot of expression * expression   (* e1 / e2 *)
  ;;
type expression =
    Const of float
  | Var of string
  | Sum of expression * expression
  | Diff of expression * expression
  | Prod of expression * expression
  | Quot of expression * expression
```

我们首先定义了一个变体来对应变量和词性之间的关系，为了简单，我们把未知数的上下文用一个list来表示：

```ocaml
# exception Unbound_variable of string;;
exception Unbound_variable of string
```

```ocaml
# let rec eval env exp =
    match exp with
      Const c -> c
    | Var v ->
        (try List.assoc v env with Not_found -> raise (Unbound_variable v))
    | Sum(f, g) -> eval env f +. eval env g
    | Diff(f, g) -> eval env f -. eval env g
    | Prod(f, g) -> eval env f *. eval env g
    | Quot(f, g) -> eval env f /. eval env g;;
val eval : (string * float) list -> expression -> float = <fun>
```

```ocaml
# eval [("x", 1.0); ("y", 3.14)] (Prod(Sum(Var "x", Const 2.0), Var "y"));;
- : float = 9.42
```

在符号解析部分，我们通过定义一个递归函数来针对`dv`进行处理：

```ocaml
# let rec deriv exp dv =
    match exp with
      Const c -> Const 0.0
    | Var v -> if v = dv then Const 1.0 else Const 0.0
    | Sum(f, g) -> Sum(deriv f dv, deriv g dv)
    | Diff(f, g) -> Diff(deriv f dv, deriv g dv)
    | Prod(f, g) -> Sum(Prod(f, deriv g dv), Prod(deriv f dv, g))
    | Quot(f, g) -> Quot(Diff(Prod(deriv f dv, g), Prod(f, deriv g dv)),
                         Prod(g, g))
  ;;
val deriv : expression -> string -> expression = <fun>
```

```ocaml
# deriv (Quot(Const 1.0, Var "x")) "x";;
- : expression =
Quot (Diff (Prod (Const 0., Var "x"), Prod (Const 1., Const 1.)),
 Prod (Var "x", Var "x"))
```

## 1.8  漂亮的输出

在上面的例子中，我们将符号抽象表达，但是这些符号表达使得表达式变得生硬难懂，我们需要一个打印函数来将这些抽象符号转换为我们便于理解的数学表达式（例如 2*x+1）

在打印函数中，我们将优先级规则引入，来避免一些没有必要的圆括号。在最后，产出的表达式将会有着更少的括号：

```ocaml
# let print_expr exp =
    (* Local function definitions *)
    let open_paren prec op_prec =
      if prec > op_prec then print_string "(" in
    let close_paren prec op_prec =
      if prec > op_prec then print_string ")" in
    let rec print prec exp =     (* prec is the current precedence *)
      match exp with
        Const c -> print_float c
      | Var v -> print_string v
      | Sum(f, g) ->
          open_paren prec 0;
          print 0 f; print_string " + "; print 0 g;
          close_paren prec 0
      | Diff(f, g) ->
          open_paren prec 0;
          print 0 f; print_string " - "; print 1 g;
          close_paren prec 0
      | Prod(f, g) ->
          open_paren prec 2;
          print 2 f; print_string " * "; print 2 g;
          close_paren prec 2
      | Quot(f, g) ->
          open_paren prec 2;
          print 2 f; print_string " / "; print 3 g;
          close_paren prec 2
    in print 0 exp;;
val print_expr : expression -> unit = <fun>
```

```ocaml
# let e = Sum(Prod(Const 2.0, Var "x"), Const 1.0);;
val e : expression = Sum (Prod (Const 2., Var "x"), Const 1.)
```

```ocaml
# print_expr e; print_newline ();;
2. * x + 1.
- : unit = ()
```

```ocaml
# print_expr (deriv e "x"); print_newline ();;
2. * 1. + 0. * x + 0.
- : unit = ()
```

## 1.9  独立运行的OCaml程序

我们所给的所有例子都是通过交互系统来运行的。OCaml的代码同样也可以通过编译成一个单独的二进制文件来运行，代码文件必须放在一个后缀名为.ml的文件中。这个过程包括了一系列解析，所以会实时输出相应的信息。不像在交互模式下面，类型，推导都被自动打印，程序只有使用打印函数才能输出，下面是个例子：

```ocaml
(* File fib.ml *)
let rec fib n =
  if n < 2 then 1 else fib (n-1) + fib (n-2);;
let main () =
  let arg = int_of_string Sys.argv.(1) in
  print_int (fib arg);
  print_newline ();
  exit 0;;
main ();;

```

`Sys.argv` 是一个包含命令行参数的数组，`Sys.argv.(1)`代表第一个参数。下面这个例子展示了程序通过命令行编译运行。

```bash
$ ocamlc -o fib fib.ml
$ ./fib 10
89
$ ./fib 20
10946
```

更多复杂的独立OCaml程序，包含多个源文件，引入库函数等将会在后面的文章里面介绍，第9和12章解释了怎样使用批编译指令ocamlc和ocamlopt。多文件的OCaml项目可以通过使用第三方的构建系统来编译，例如：[ocamlbuild](https://github.com/ocaml/ocamlbuild/) 