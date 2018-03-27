> 这学期的算法，编译这些不限语言的课，我基本都用了ocaml自己写（这样就不会被同学要代码了）
>
> 之前我在JavaScript写函数式，总觉着怪怪的，直到我找到了OCaml
>
> 虽然这个语言和我一样大，但是普及度堪忧。。。这学期一份安利都没有卖出去
>
> 为了深入学习OCaml，我翻译了文档

# 第一章 核心语言部分

[TOC]



## 1.1  基础部分

为了对OCaml语言有一个很直接的了解，我们使用交互系统来实践代码。如果是在Linux，Unix下，则是在ocaml的命令行界面运行，如果是在Windows下，则是使用 OCamlwin.exe 来运行。本教程将会展示所有输入以及输出来展示结果，#开头的语句代表用户的输入，紧接着没有#的段落则是输出。

在这个交互系统中，用户输入的OCaml语句会被;;结束，然后输出相应的结果。系统是实时编译执行，并打印出结果。语句一般是表达式或者是let定义式（定义值或者函数）。

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

OCaml 系统会在解析的时候分析每一个短语的类型和值，就连函数的参数都需要详细的类型声明：系统也会根据他们在函数之中的使用来推断他的类型。需要注意的是，整数和浮点数是两个分开的类型，我们使用+和*来操作整数，+.和\*.来操作浮点数。

```Ocaml
# 1.0 * 2;;
Error: This expression has type float but an expression was expected of type int
```

递归函数需要在声明时候绑定rec关键字：

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

除了整数和浮点数，OCaml还提供了一些其他的基本数据类型：布尔值，字符，和不可改变的字符串。

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

预定义的数据结构包括了turples，array，list。当然你也可以定义你自己的数据结构，这在之后的内容中会有更多细节。但是现在，我们先来看看list。list一般是用中括号括起来，用分号彼此分开声明的，或者是通过\[\](叫做‘nil’)，通过 :: 操作符加上数据来创建。

```ocaml
# let l = ["is"; "a"; "tale"; "told"; "etc."];;
val l : string list = ["is"; "a"; "tale"; "told"; "etc."]
```

```ocaml
# "Life" :: l;;
- : string list = ["Life"; "is"; "a"; "tale"; "told"; "etc."]
```

其他OCaml 数据结构和list类似，都不需要分别的为他们申请内存空间，内存管理机制也自动的由OCaml执行。类似的，OCaml也不用单独处理指针，在编译的时候，OCaml会在必要时静默的引入指针。

同样的，这些数据结构也会和list一样，通过模式识别来监听和销毁。list的模式和list的表达式是一样的，通过一些标识符来代表一些非特定的区域，我们可以操作list。下面是一个插入排序的例子：

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

排序的类型推断是：'a list -> 'a list，这就是说这个排序算法可以应用于任何包含同类型变量的list，类型 'a 是一个 a 类型变量，可以代表任何一个给定的type。这个排序可以使用到任意类型list的原因是，比较操作符在OCaml里面实现了多态：他可以接收任意两个相同类型的变量，并返回布尔值。这就使得排序函数也是多态的。

```ocaml
# sort [6;2;5;3];;
- : int list = [2; 3; 5; 6]
```

```ocaml
# sort [3.14; 2.718];;
- : float list = [2.718; 3.14]
```

排序算法并没有修改原来的list，它返回了一个包含了和输入list具有同样元素的list。实际上在OCaml之中，list在创建之后就无法再被改变了，是一种不可变的数据结构。大部分OCaml数据结构都是不可变的，但是一些数据结构（主要是数组）是可变的，也就是说他们可以随时被改变。



## 1.3  函数亦是值

OCaml是一门函数式编程语言：函数应该和其他数学值一样，可以作为一部分数据，并且可以被自由的传递。举个例子来说，这里有一个 deriv 函数接收任意浮点数函数作为参数，并且返回计算结果：

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

接受函数作为参数的函数叫做“高阶函数”，高阶函数在遍历和操作数据结构方面有很大的用处。举个例子，OCaml标准库提供了一个List.map 函数来接受一个函数遍历List，并返回结果。

```ocaml
# List.map (function n -> n * 2 + 1) [0;1;2;3;4];;
- : int list = [1; 3; 5; 7; 9]
```

这个有用的高阶函数，已经被预定义为list和array的库函数，但是其实它也不是很高深的技巧，可以很简单的定义为：

```ocaml
# let rec map f l =
    match l with
      [] -> []
    | hd :: tl -> f hd :: map f tl;;
val map : ('a -> 'b) -> 'a list -> 'b list = <fun>
```

## 1.4  Records 和 variants

用户自定义的数据结构包括 records 和 variants，他们都是使用type来声明的，这里我们定义了一个record类型来代表无理数。

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

没有用到的量可以被省去：

```ocaml
# let get_denom {denom=denom} = denom;;
val get_denom : ratio -> int = <fun>
```

同样的，没有用到的量也可以通过 _ 通配符来舍去

```ocaml
# let get_num {num=num; _ } = num;;
val get_num : ratio -> int = <fun>
```

如果 = 两边都变量名称是相同的，我们可以不用那么麻烦的写法：

```ocaml
# let integer_part {num; denom} = num / denom;;
val integer_part : ratio -> int = <fun>
```

这种话简写的方法同样适用于使用函数来构造record：

```ocaml
# let ratio num denom = {num; denom};;
val ratio : int -> int -> ratio = <fun>
```

你也可以一次性的改变record的内容：

```ocaml
# let integer_product integer ratio = { ratio with num = integer * ratio.num };;
val integer_product : int -> ratio -> ratio = <fun>
```

with这个函数符号，接受左边的值并且copy后更新右边的量，最终返回这个record。

variant的声明则是列出了所有可能的该类型的值，每一种都由构造器和它所对应的实际类型来对应，构造器通过开头大写来与变量名做出区分。

```ocaml
# type number = Int of int | Float of float | Error;;
type number = Int of int | Float of float | Error
```

这个声明表达了number类型的值可以是整型，浮点数或者一个常量成为Error（通常是非法操作获得的，比如除以0）。

枚举类型是一种variant的特殊用法：

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

另一个有趣的例子就是，内建的可选类型，它代表了是返回了一个值还是没有返回值：

```ocaml
# type 'a option = Some of 'a | None;;
type 'a option = Some of 'a | None
```

这个类型在判断一些会失败的函数上有很大作用，例如：

```ocaml
# let safe_square_root x = if x > 0. then Some(sqrt x) else None;;
val safe_square_root : float -> float option = <fun>
```

最常见的variant类型使用还是在描述递归数据结构上，例如一个二叉树：

```ocaml
# type 'a btree = Empty | Node of 'a * 'a btree * 'a btree;;
type 'a btree = Empty | Node of 'a * 'a btree * 'a btree
```

这个定义应该这样解释：一个二叉树节点可以是Empty，特可以是一个可选类型的值，和两个该类型的节点。

对于二叉树的操作本来就是以递归的方式定义的，举个例子，在二分树进行查找和插入：

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

## 1.5  指令式编程

到此为止的很多例子，我们都是使用纯声明式编程来写的。OCaml同时也具有完整的指令式编程的特性。包括但不限于，while和for循环，可变的数据结构array。array被以[|和|]声明，他也可以使用Array.make函数来创建一个array。举个例子，下面就是两个向量的加法：

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

OCaml 有内建的变量概念，变量可以被赋值（let 绑定并不是一种赋值语句，他只是把新的标记符带进作用域）。然而，标准库提供了参照，它是一个不确定的单元（或者是单元素数组），通过 ! 来得到变量的内容，通过 := 来赋值变量的内容。通过参照来获取变量内容并且修改内容，下面是一个插入排序的例子：

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

参照对于包含外来状态的函数有很大作用，例如一下生成伪随机数的例子，最后返回的是参照的值：

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

当然，参照也不是什么奇技淫巧，我们也可以通过可变变量的record来实现：

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

在一些特殊情况下，你需要储存一些多态的函数在数据结构中，以保证它的多态。但是，OCaml不允许没有用户提供的类型声明。这时候你可以使用多态的类型来达到这个目的：

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

OCaml 提供异常来作为信号，应对非常规的情况。异常同样也可以用在一些远程请求控制结构上。异常通过exception构造器构造，通过raise操作符来发出信号。举个例子，下面的函数对空list进行了处理：

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

异常被用于整个标准库，用来警示那些库函数不能正常完成的情况。举个例子，List.assoc 函数，返回在pair list中指定键的值，如果没有找到，就会有异常：Not_found：

```ocaml
# List.assoc 1 [(0, "zero"); (1, "one")];;
- : string = "one"
```

```ocaml
# List.assoc 2 [(0, "zero"); (1, "one")];;
Exception: Not_found.
```

异常可以被try关键字捕获：

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

with部分实际上还是普通的对异常值的模式匹配，我们可以通过下面这种方式来捕获所有异常，进行操作，最终再raise异常：

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

我们将以一些更具代表性的例子来解释OCaml的词法分析：对于包含变量的格式操作，下列是描述符的定义：

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

我们首先定义了一个variant来对应变量和词性之间的关系，为了简单，我们把环境用list来表示：

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

对于正式的符号解析，我们定义了一个派生表达式来针对dv进行处理：

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

在上面的例子中，我们有了抽象的符号表达，但是这些符号表达使得表达式变得生硬难懂，我们需要一个打印函数来吧这些抽象符号转换为我们便于理解的数学表达式（例如 2*x+1）

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

Sys.argv 是一个包含命令行参数的数组，Sys.argv.(1)代表第一个参数。下面这个例子展示了程序通过命令行编译运行。

```bash
$ ocamlc -o fib fib.ml
$ ./fib 10
89
$ ./fib 20
10946
```

更多复杂的独立OCaml程序，包含多个源文件，引入库函数等将会在后面的文章里面介绍，第9和12张解释了怎样使用批编译指令ocamlc和ocamlopt。多文件的OCaml项目可以通过使用第三方的构建系统来编译，例如：[ocamlbuild](https://github.com/ocaml/ocamlbuild/) 。