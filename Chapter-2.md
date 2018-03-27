# 第二章 module 系统

[TOC]



## 2.1  结构体 STRUCT

module是为了将有关的定义打包在一起（比如数据类型和它的相关操作），并对这些定义做出统一的命名规划。为了避免用完名字或者变量名的混淆，OCaml引入了struct关键字。这样的一个包是使用struct……end关键字来声明的，它包含了一系列的定义。结构体一般会和一个module名称绑定。这是一个优先级队列以及相关操作的打包：

```ocaml
# module PrioQueue =
    struct
      type priority = int
      type 'a queue = Empty | Node of priority * 'a * 'a queue * 'a queue
      let empty = Empty
      let rec insert queue prio elt =
        match queue with
          Empty -> Node(prio, elt, Empty, Empty)
        | Node(p, e, left, right) ->
            if prio <= p
            then Node(prio, elt, insert right p e, left)
            else Node(p, e, insert right prio elt, left)
      exception Queue_is_empty
      let rec remove_top = function
          Empty -> raise Queue_is_empty
        | Node(prio, elt, left, Empty) -> left
        | Node(prio, elt, Empty, right) -> right
        | Node(prio, elt, (Node(lprio, lelt, _, _) as left),
                          (Node(rprio, relt, _, _) as right)) ->
            if lprio <= rprio
            then Node(lprio, lelt, remove_top left, right)
            else Node(rprio, relt, left, remove_top right)
      let extract = function
          Empty -> raise Queue_is_empty
        | Node(prio, elt, _, _) as queue -> (prio, elt, remove_top queue)
    end;;
module PrioQueue :
  sig
    type priority = int
    type 'a queue = Empty | Node of priority * 'a * 'a queue * 'a queue
    val empty : 'a queue
    val insert : 'a queue -> priority -> 'a -> 'a queue
    exception Queue_is_empty
    val remove_top : 'a queue -> 'a queue
    val extract : 'a queue -> priority * 'a * 'a queue
  end
```

在结构体外部，他的组件可以通过点操作符被引用，通过结构体名称被唯一确认，就像例子里面的优先级队列，PrioQueue.insert定义了队列的插入操作，PrioQueue.queue是一个variant类型定义。

```ocaml
# PrioQueue.insert PrioQueue.empty 1 "hello";;

- : string PrioQueue.queue =
PrioQueue.Node (1, "hello", PrioQueue.Empty, PrioQueue.Empty)
```

OCaml还提供了一种open操作，它允许全部的包内定义释放到现在的结构作用域里面。

```ocaml
#  open PrioQueue;;

#  insert empty 1 "hello";;
- : string PrioQueue.queue = Node (1, "hello", Empty, Empty)
```

打开一个module轻量化了对于它组件的获取方式，但同时会让已经在环境中的定义产生冲突。在一些特殊情况下，打开module可能产生命名冲突，很容易导致一些让人疑惑的错误：

```ocaml
#   let empty = []
#   open PrioQueue;;
val empty : 'a list = []
```

```ocaml
#   let x = 1 :: empty ;;
Error: This expression has type 'a PrioQueue.queue
       but an expression was expected of type int list
```

有一种解决方案是在作用域里面打开modules，让module组件只集中在部分表达式中可用。这样也使代码更加容易阅读，open声明离它被使用的地方更近了，代码片段也实现了自包含。我们可以使用两种操作来实现：

```ocaml
#   let open PrioQueue in
    insert empty 1 "hello";;
- : string PrioQueue.queue = Node (1, "hello", Empty, Empty)
```

和

```ocaml
#   PrioQueue.(insert empty 1 "hello");;
- : string PrioQueue.queue = Node (1, "hello", Empty, Empty)
```

在第二种形式里面，本地作用域被圆括号界定，这种圆括号也是可以被省略的，例如：

```ocaml
#   PrioQueue.[empty] = PrioQueue.([empty]);;
- : bool = true
```

```ocaml
#   PrioQueue.[|empty|] = PrioQueue.([|empty|]);;
- : bool = true
```

```ocaml
#    PrioQueue.{ contents = empty } = PrioQueue.({ contents = empty });;
- : bool = true
```

上面的代码就可以简写为：

```ocaml
#   PrioQueue.[insert empty 1 "hello"];;
- : string PrioQueue.queue list = [Node (1, "hello", Empty, Empty)]
```

我们也可以在module之中用include引入其他module的定义，这对于拓展现有的module是特别有效的。我们可以在上面的例子中加入一个不会抛出异常，而是返回可选值的None的函数：

```ocaml
#   module PrioQueueOpt =
    struct
      include PrioQueue

      let remove_top_opt x =
        try Some(remove_top x) with Queue_is_empty -> None

      let extract_opt x =
        try Some(extract x) with Queue_is_empty -> None
    end;;
module PrioQueueOpt :
  sig
    type priority = int
    type 'a queue =
      'a PrioQueue.queue =
        Empty
      | Node of priority * 'a * 'a queue * 'a queue
    val empty : 'a queue
    val insert : 'a queue -> priority -> 'a -> 'a queue
    exception Queue_is_empty
    val remove_top : 'a queue -> 'a queue
    val extract : 'a queue -> priority * 'a * 'a queue
    val remove_top_opt : 'a queue -> 'a queue option
    val extract_opt : 'a queue -> (priority * 'a * 'a queue) option
  end

```

## 2.2  签名

签名相当于结构体的交互界面，它具体说明了结构体的哪些部分可以被外部访问，以及它们的类型。签名可以用来隐藏一些结构体内部组件，或者将重定义类型导出组件。举个例子来说，下面的签名详细说明了优先级队列的empty，insert和extract的类型，但是没有声明内部辅助功能remove_top，也让queue的类型抽象了（它没有提供一个指定的类型）：

```ocaml
# module type PRIOQUEUE =
    sig
      type priority = int         (* still concrete *)
      type 'a queue               (* now abstract *)
      val empty : 'a queue
      val insert : 'a queue -> int -> 'a -> 'a queue
      val extract : 'a queue -> int * 'a * 'a queue
      exception Queue_is_empty
    end;;
module type PRIOQUEUE =
  sig
    type priority = int
    type 'a queue
    val empty : 'a queue
    val insert : 'a queue -> int -> 'a -> 'a queue
    val extract : 'a queue -> int * 'a * 'a queue
    exception Queue_is_empty
  end
```

通过签名将PrioQueue结构体重新定向，创建了PrioQueue的另一个视图，在这个视图中，remove_top在实际的队列中是无法被获取的：

```ocaml
# module AbstractPrioQueue = (PrioQueue : PRIOQUEUE);;
module AbstractPrioQueue : PRIOQUEUE
```

```ocaml
# AbstractPrioQueue.remove_top ;;
Error: Unbound value AbstractPrioQueue.remove_top
```

```ocaml
# AbstractPrioQueue.insert AbstractPrioQueue.empty 1 "hello";;
- : string AbstractPrioQueue.queue = <abstr>
```

这样的重定向也同样可以用在定义结构体的时候：

```ocaml
module PrioQueue = (struct ... end : PRIOQUEUE);;
```

也可以和module名绑定：

```ocaml
module PrioQueue : PRIOQUEUE = struct ... end;;
```

类似于module，签名也可以通过include关键字从其他的签名引入，在下面的例子，我们拓展了 PRIOQUEUE 的定义：

```ocaml
# module type PRIOQUEUE_WITH_OPT =
    sig
      include PRIOQUEUE
      val extract_opt : 'a queue -> (int * 'a * 'a queue) option
    end;;
module type PRIOQUEUE_WITH_OPT =
  sig
    type priority = int
    type 'a queue
    val empty : 'a queue
    val insert : 'a queue -> int -> 'a -> 'a queue
    val extract : 'a queue -> int * 'a * 'a queue
    exception Queue_is_empty
    val extract_opt : 'a queue -> (int * 'a * 'a queue) option
  end
```

## 2.3  函子

函子是从结构体到结构体的特殊函数，他们被用来表达一些被参数化的结构体：类似于结构体A是通过结构体B作为参数定义的（通过B的签名定义的类型），函子通过接受参数B返回了A结构体本身。函子F可以应用于一类抽象的B结构，并产出相应的A结构体。

举个例子，结构体set通过排序过的list来定义，通过结构体参数来提供元素的集合和这种类型的排序函数（用来保持set是排过序的）：

```Ocaml
# type comparison = Less | Equal | Greater;;
type comparison = Less | Equal | Greater
```

```Ocaml
# module type ORDERED_TYPE =
    sig
      type t
      val compare: t -> t -> comparison
    end;;
module type ORDERED_TYPE = sig type t val compare : t -> t -> comparison end
```

```Ocaml
# module Set =
    functor (Elt: ORDERED_TYPE) ->
      struct
        type element = Elt.t
        type set = element list
        let empty = []
        let rec add x s =
          match s with
            [] -> [x]
          | hd::tl ->
             match Elt.compare x hd with
               Equal   -> s         (* x 已经包含在 s 里面了 *)
             | Less    -> x :: s    (* x 比 s 所有元素都小 *)
             | Greater -> hd :: add x tl
        let rec member x s =
          match s with
            [] -> false
          | hd::tl ->
              match Elt.compare x hd with
                Equal   -> true     (* x 属于 s *)
              | Less    -> false    (* x 比 s 所有元素都小 *)
              | Greater -> member x tl
      end;;
module Set :
  functor (Elt : ORDERED_TYPE) ->
    sig
      type element = Elt.t
      type set = element list
      val empty : 'a list
      val add : Elt.t -> Elt.t list -> Elt.t list
      val member : Elt.t -> Elt.t list -> bool
    end
```

通过向Set函子里面输入一个包含了类型和比较函数的结构体，我们得到了相应类型的结构体：

```ocaml
# module OrderedString =
    struct
      type t = string
      let compare x y = if x = y then Equal else if x < y then Less else Greater
    end;;
module OrderedString :
  sig type t = string val compare : 'a -> 'a -> comparison end
```

```ocaml
# module StringSet = Set(OrderedString);;
module StringSet :
  sig
    type element = OrderedString.t
    type set = element list
    val empty : 'a list
    val add : OrderedString.t -> OrderedString.t list -> OrderedString.t list
    val member : OrderedString.t -> OrderedString.t list -> bool
  end
```

```ocaml
# StringSet.member "bar" (StringSet.add "foo" StringSet.empty);;
- : bool = false
```

## 2.4  函子和类型抽象

在PrioQueue例子中，为了让用户使用结构体的时候，不需要被限制于它是基于list，我们将实际的类型抽象过程隐藏起来，用一个函子来抽象这个过程，用户使用的时候就不用改变源代码就可以改变底层实现的结构类型。以上情况可以通过使用函子的签名来重定向实现：

```ocaml
# module type SETFUNCTOR =
    functor (Elt: ORDERED_TYPE) ->
      sig
        type element = Elt.t      (* concrete *)
        type set                  (* abstract *)
        val empty : set
        val add : element -> set -> set
        val member : element -> set -> bool
      end;;
module type SETFUNCTOR =
  functor (Elt : ORDERED_TYPE) ->
    sig
      type element = Elt.t
      type set
      val empty : set
      val add : element -> set -> set
      val member : element -> set -> bool
    end
```

```ocaml
# module AbstractSet = (Set : SETFUNCTOR);;
module AbstractSet : SETFUNCTOR
```

```ocaml
# module AbstractStringSet = AbstractSet(OrderedString);;
module AbstractStringSet :
  sig
    type element = OrderedString.t
    type set = AbstractSet(OrderedString).set
    val empty : set
    val add : element -> set -> set
    val member : element -> set -> bool
  end
```

```ocaml
# AbstractStringSet.add "gee" AbstractStringSet.empty;;
- : AbstractStringSet.set = <abstr>
```

为了优雅的加上类型，我们希望返回的结构体的签名也可以通过函子来得到，用这个签名来做类型限制：

```ocaml
# module type SET =
    sig
      type element
      type set
      val empty : set
      val add : element -> set -> set
      val member : element -> set -> bool
    end;;
module type SET =
  sig
    type element
    type set
    val empty : set
    val add : element -> set -> set
    val member : element -> set -> bool
  end
```

```ocaml
# module WrongSet = (Set : functor(Elt: ORDERED_TYPE) -> SET);;
module WrongSet : functor (Elt : ORDERED_TYPE) -> SET
```

```ocaml
# module WrongStringSet = WrongSet(OrderedString);;
module WrongStringSet :
  sig
    type element = WrongSet(OrderedString).element
    type set = WrongSet(OrderedString).set
    val empty : set
    val add : element -> set -> set
    val member : element -> set -> bool
  end
```

```ocaml
# WrongStringSet.add "gee" WrongStringSet.empty ;;
Error: This expression has type string but an expression was expected of type
         WrongStringSet.element = WrongSet(OrderedString).element
```

上述代码出现问题的原因是，因为SET抽象的定义了类型element，所以在函子返回的结果中缺失了基础类型的定义，导致最后类型等价失败。因此，WrongStringSet.element 的类型并不是string，WrongStringSet的操作也并不能在string 上使用。作为样板，在SET的签名中element的类型被声明为Elt.t是非常重要的，但不幸的是，SET定义的上下文中并没有Elt。为了解决这个问题，OCaml提供了带有类型的共同构建，提供额外的类型等价来共同构造签名：

```ocaml
# module AbstractSet2 =
    (Set : functor(Elt: ORDERED_TYPE) -> (SET with type element = Elt.t));;
module AbstractSet2 :
  functor (Elt : ORDERED_TYPE) ->
    sig
      type element = Elt.t
      type set
      val empty : set
      val add : element -> set -> set
      val member : element -> set -> bool
    end
```

在这个情况下，我们可以使用一个可选的标记，提供函子定义和类型重定向来获取结果：

```ocaml
module AbstractSet2(Elt: ORDERED_TYPE) : (SET with type element = Elt.t) =
  struct ... end;;
```

在函子中抽象的组件类型是有强大的技术支持的，它提供了高度的类型安全保障。考虑有序字符串的情况，我们使用同标准不同的方式来比较字符串，比如，我们不分大小写来比较字符串：

```ocaml
# module NoCaseString =
    struct
      type t = string
      let compare s1 s2 =
        OrderedString.compare (String.lowercase_ascii s1) (String.lowercase_ascii s2)
    end;;
module NoCaseString :
  sig type t = string val compare : string -> string -> comparison end
```

```ocaml
# module NoCaseStringSet = AbstractSet(NoCaseString);;
module NoCaseStringSet :
  sig
    type element = NoCaseString.t
    type set = AbstractSet(NoCaseString).set
    val empty : set
    val add : element -> set -> set
    val member : element -> set -> bool
  end
```

```ocaml
# NoCaseStringSet.add "FOO" AbstractStringSet.empty ;;
Error: This expression has type
         AbstractStringSet.set = AbstractSet(OrderedString).set
       but an expression was expected of type
         NoCaseStringSet.set = AbstractSet(NoCaseString).set
```

需要注意的是AbstractStringSet.set和NoCaseStringSet.set类型是不能共用的，这两种类型的值也不相匹配。正确的做法应该是：虽然两种类型都是基于一个类型的，但是他们对类型的排序是不同的，这种不同的特性也需要在操作中体现（严格的按照标准排序增长，从而构成一种不受实际案例影响的顺序）。将AbstractStringSet的操作应用到NoCaseStringSet.set上会导致错误的结果，或者构建一个不符合NoCaseStringSet规则的list。

## 2.5  Modules 和 多文件编辑

以上所有的例子都是在交互系统中运行的。但是，module在批编译程序中有着更大的用处。这些程序实际上都很有必要将源代码分割到不同的文件中，可以被分别编译，在组合的时候将变化最小化。

在OCaml中，单独可编辑的单元是结构体和签名组合的特殊情况，他们之间的关系在module系统中可以被容易的解释。一个可编辑的单元有两部分组成：

- 包含着实现的文件A.ml，包含了一系列的定义，类似于由struct……end构建的结构。
- 包含着接口的文件A.mli，包含了一系列的详细说明，类似于sig……end构建的签名。

这两个文件定义了一个结构体名为A，这样下面的定义就进入了顶级作用域：

```ocaml
module A: sig (* A.mli 的内容 *) end
        = struct (* A.ml 的内容 *) end;;
```

这些文件定义了一个可以被ocamlc -c命令单独编译的可编辑单元（ocamlc -c 意思是只做编译，不作连接）。这样会编译出.cmi的接口文件和.cmo的二进制文件。当所有的单元被编译的时候，他们的.cmo文件会被使用ocamlc连接在一起。菊科例子来说，下面的命令编译和连接了一个包含了Aux和Main的文件：

```bash
$ ocamlc -c Aux.mli                     # 输出 aux.cmi
$ ocamlc -c Aux.ml                      # 输出 aux.cmo
$ ocamlc -c Main.mli                    # 输出 main.cmi
$ ocamlc -c Main.ml                     # 输出 main.cmo
$ ocamlc -o theprogram Aux.cmo Main.cmo
```

这个程序运行起来相当于下面的代码被放到了顶级作用域：

```ocaml
module Aux: sig (* Aux.mli 的内容 *) end
          = struct (* Aux.ml 的内容 *) end;;
module Main: sig (* Main.mli 的内容 *) end
           = struct (* Main.ml 的内容 *) end;;
```

特殊地，Main可以引用Aux：在Main.ml和Main.mli的定义和声明可以引用Aux.ml里面的定义，通过使用Aux.ident记号可以使用Aux.mli导出的接口。

在camlc中输入的.cmo文件的顺序决定了module定义的顺序。在上述例子中，Aux在Main之前出现，所里Main可以引用Aux，但是反过来不可以。

值得注意的是，只有顶级作用域的结构体可以被单独编译，而单独的函子或者module类型不可以。但是，所有module级的对象都可以作为结构体编译，所以解决方法是把函子放在结构体里，这样就能在一个文件里面单独编译了。