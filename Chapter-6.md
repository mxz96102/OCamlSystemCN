# Chapter 6  类与模块进阶

在这一章，我们将会展示一些使用类，对象，模块的更大型的例子。我们将通过银行账户的例子来回顾对象的特性。展示标准库的模型怎样用类来表示。最后，我们通过窗口管理器来描述一种被称为虚拟类型的编程模式。

## 6.1  类拓展示例：银行账户

在这一节，我们讲通过下面这个简单的英航账户例子来描述对象的大部分对象特性以及继承特效。（我们将重新使用我们在第三章使用过的Euro 模块）

```ocaml
# let euro = new Euro.c;;
val euro : float -> Euro.c = <fun>
```

```ocaml
# let zero = euro 0.;;
val zero : Euro.c = <obj>
```

```ocaml
# let neg x = x#times (-1.);;
val neg : < times : float -> 'a; .. > -> 'a = <fun>
```

```ocaml
# class account =
    object
      val mutable balance = zero
      method balance = balance
      method deposit x = balance <- balance # plus x
      method withdraw x =
        if x#leq balance then (balance <- balance # plus (neg x); x) else zero
    end;;
class account :
  object
    val mutable balance : Euro.c
    method balance : Euro.c
    method deposit : Euro.c -> unit
    method withdraw : Euro.c -> Euro.c
  end
```

```ocaml
# let c = new account in c # deposit (euro 100.); c # withdraw (euro 50.);;
- : Euro.c = <obj>
```

我们现在重新定义了方法计算利息：

```ocaml
# class account_with_interests =
    object (self)
      inherit account
      method private interest = self # deposit (self # balance # times 0.03)
    end;;
class account_with_interests :
  object
    val mutable balance : Euro.c
    method balance : Euro.c
    method deposit : Euro.c -> unit
    method private interest : unit
    method withdraw : Euro.c -> Euro.c
  end
```

我们将让interest方法私有，这个方法看起来不该让外部随意调用。在这里，它只能被子类调用用于每年/每月来更新账户。

我们需要处理一个当前定义的bug：存款方法可以被用来传入负数参数来取钱。我们可以直接修复：
```ocaml
# class safe_account =
    object
      inherit account
      method deposit x = if zero#leq x then balance <- balance#plus x
    end;;
class safe_account :
  object
    val mutable balance : Euro.c
    method balance : Euro.c
    method deposit : Euro.c -> unit
    method withdraw : Euro.c -> Euro.c
  end
```

但是，下面这个定义的解决方法更加安全：

```ocaml
# class safe_account =
    object
      inherit account as unsafe
      method deposit x =
        if zero#leq x then unsafe # deposit x
        else raise (Invalid_argument "deposit")
    end;;
class safe_account :
  object
    val mutable balance : Euro.c
    method balance : Euro.c
    method deposit : Euro.c -> unit
    method withdraw : Euro.c -> Euro.c
  end
```

这样定义的方法做到了与deposit方法的实现无关。

为了持续记录操作，我们将一个可变域用来跟踪调用历史，加上一个方法来记录操作的log。这样一来，每一个要追踪的方法都要被重新定义。

```ocaml
# type 'a operation = Deposit of 'a | Retrieval of 'a;;
type 'a operation = Deposit of 'a | Retrieval of 'a
```

```ocaml
# class account_with_history =
    object (self)
      inherit safe_account as super
      val mutable history = []
      method private trace x = history <- x :: history
      method deposit x = self#trace (Deposit x);  super#deposit x
      method withdraw x = self#trace (Retrieval x); super#withdraw x
      method history = List.rev history
    end;;
class account_with_history :
  object
    val mutable balance : Euro.c
    val mutable history : Euro.c operation list
    method balance : Euro.c
    method deposit : Euro.c -> unit
    method history : Euro.c operation list
    method private trace : Euro.c operation -> unit
    method withdraw : Euro.c -> Euro.c
  end
```

有人也许希望能够在开账户的同时存进一定的初始金额。虽然初始化定义也许不能满足这个需求，但是我们可以通过初始化器来实现

```ocaml
# class account_with_deposit x =
    object
      inherit account_with_history
      initializer balance <- x
    end;;
class account_with_deposit :
  Euro.c ->
  object
    val mutable balance : Euro.c
    val mutable history : Euro.c operation list
    method balance : Euro.c
    method deposit : Euro.c -> unit
    method history : Euro.c operation list
    method private trace : Euro.c operation -> unit
    method withdraw : Euro.c -> Euro.c
  end
```

一种更好的选择是：

```ocaml
# class account_with_deposit x =
    object (self)
      inherit account_with_history
      initializer self#deposit x
    end;;
class account_with_deposit :
  Euro.c ->
  object
    val mutable balance : Euro.c
    val mutable history : Euro.c operation list
    method balance : Euro.c
    method deposit : Euro.c -> unit
    method history : Euro.c operation list
    method private trace : Euro.c operation -> unit
    method withdraw : Euro.c -> Euro.c
  end
```

的确，因为调用deposit自带了安全检查和增加了追踪，所以后者更加安全，让我们来测试一下：

```ocaml
# let ccp = new account_with_deposit (euro 100.) in
  let _balance = ccp#withdraw (euro 50.) in
  ccp#history;;
- : Euro.c operation list = [Deposit <obj>; Retrieval <obj>]
```

关闭账户可以被下面的多态方法来解决：

```ocaml
# let close c = c#withdraw c#balance;;
val close : < balance : 'a; withdraw : 'a -> 'b; .. > -> 'b = <fun>
```

当然，这个适用于所有类型的账户。

最后，我们将多个版本的账户集中到一个模块，并且抽象出一些货币。

```ocaml
# let today () = (01,01,2000) (* an approximation *)
  module Account (M:MONEY) =
    struct
      type m = M.c
      let m = new M.c
      let zero = m 0.

      class bank =
        object (self)
          val mutable balance = zero
          method balance = balance
          val mutable history = []
          method private trace x = history <- x::history
          method deposit x =
            self#trace (Deposit x);
            if zero#leq x then balance <- balance # plus x
            else raise (Invalid_argument "deposit")
          method withdraw x =
            if x#leq balance then
              (balance <- balance # plus (neg x); self#trace (Retrieval x); x)
            else zero
          method history = List.rev history
        end

      class type client_view =
        object
          method deposit : m -> unit
          method history : m operation list
          method withdraw : m -> m
          method balance : m
        end

      class virtual check_client x =
        let y = if (m 100.)#leq x then x
        else raise (Failure "Insufficient initial deposit") in
        object (self)
          initializer self#deposit y
          method virtual deposit: m -> unit
        end

      module Client (B : sig class bank : client_view end) =
        struct
          class account x : client_view =
            object
              inherit B.bank
              inherit check_client x
            end

          let discount x =
            let c = new account x in
            if today() < (1998,10,30) then c # deposit (m 100.); c
        end
    end;;
```

这里展示了该怎样使用模块将多各类成组的集中到一个单元中。这个单元可以提供给银行内部或者外部使用。实现了这样一个函子，就将货币抽象出来，相同的代码可以用于不同的货币。

bank类是一个真实的银行账户实现，这里的实现可以被用于未来的拓展，改进等等。相反的，客户只会被给与客户端的视图：

```ocaml
# module Euro_account = Account(Euro);;
```

```ocaml
# module Client = Euro_account.Client (Euro_account);;
```

```ocaml
# new Client.account (new Euro.c 100.);;
```

因此，客户并没有直接接触存款或者自己账户历史的权限。他们唯一可以改变自己存款的方法就是存钱或者取钱。非常重要的一点是：让客户有一个类而不是由能力去创建账户（例如推广用的打折账户），所以他们可以自己定义一些账户信息。举个例子来说，客户也许希望自己给劲存款和取款方法来自动记录他的经济流动。另一方面，我们提供了discountis方法，这个方法是没有办法被个人定义的。

给客户视图提供一个Client函子也是很重要的，这样之后不同银行可以有不同的定制化。Client函子在之后也许会保留不变，或者被传向一个新的定义来初始化拓展后的账户用户视图。

```ocaml
# module Investment_account (M : MONEY) =
    struct
      type m = M.c
      module A = Account(M)

      class bank =
        object
          inherit A.bank as super
          method deposit x =
            if (new M.c 1000.)#leq x then
              print_string "Would you like to invest?";
            super#deposit x
        end

      module Client = A.Client
    end;;
```

函子Client也可以在一些新的特性加入账户的时候被改进后给用户。

```ocaml
# module Internet_account (M : MONEY) =
    struct
      type m = M.c
      module A = Account(M)

      class bank =
        object
          inherit A.bank
          method mail s = print_string s
        end

      class type client_view =
        object
          method deposit : m -> unit
          method history : m operation list
          method withdraw : m -> m
          method balance : m
          method mail : string -> unit
        end

      module Client (B : sig class bank : client_view end) =
        struct
          class account x : client_view =
            object
              inherit B.bank
              inherit A.check_client x
            end
        end
    end;;
```

## 6.2  简单的模块即是类

也许有人会想，我们是不是能把一些原生类型，比如整数，字符串来当做对象处理呢。虽然这样处理整数和字符串通常很无趣，但是的确在一些情况下是有这样的需求的。下面的money类就是一个例子，我们将在这里展示我们该怎样处理字符串。

### 6.2.1  字符串

将字符串作为对象处理的原生定义可以是这样：

```ocaml
 class ostring s =
    object
       method get n = String.get s n
       method print = print_string s
       method escaped = new ostring (String.escaped s)
    end;;
class ostring :
  string ->
  object
    method escaped : ostring
    method get : int -> char
    method print : unit
  end
```

但是，escaped返回的是一个ostring类的对象，而并非当前类的对象。因此，如果一个类是由多次的extend的到的，那么escaped方法只会返回这个对象的父类。

```ocaml
# class sub_string s =
    object
       inherit ostring s
       method sub start len = new sub_string (String.sub s  start len)
    end;;
class sub_string :
  string ->
  object
    method escaped : ostring
    method get : int -> char
    method print : unit
    method sub : int -> int -> sub_string
  end
```

正如在3.16中我们看到过的那样，解决方案就是使用函数，我们需要创建一个包含字符串s引用的实例变量。

```ocaml
 # class better_string s =
    object
       val repr = s
       method get n = String.get repr n
       method print = print_string repr
       method escaped = {< repr = String.escaped repr >}
       method sub start len = {< repr = String.sub s start len >}
    end;;
class better_string :
  string ->
  object ('a)
    val repr : string
    method escaped : 'a
    method get : int -> char
    method print : unit
    method sub : int -> int -> 'a
  end
```

正如在推断类型中显示的那样，escaped和sub方法现在返回值类型是和输入类相同的类型。

另一个很难实现的地方就是concat方法。为了确定一个字符串和另一个字符串是同一个类的，我们需要去从外部访问实例变量。这样以来，我们需要定义一个返回s的方法repr。下面是正确的字符串类定义：

```ocaml
# class ostring s =
    object (self : 'mytype)
       val repr = s
       method repr = repr
       method get n = String.get repr n
       method print = print_string repr
       method escaped = {< repr = String.escaped repr >}
       method sub start len = {< repr = String.sub s start len >}
       method concat (t : 'mytype) = {< repr = repr ^ t#repr >}
    end;;
class ostring :
  string ->
  object ('a)
    val repr : string
    method concat : 'a -> 'a
    method escaped : 'a
    method get : int -> char
    method print : unit
    method repr : string
    method sub : int -> int -> 'a
  end
```

另一个string类的构造器可以被定义为返回一个定长的字符串。

```ocaml
# class cstring n = ostring (String.make n ' ');;
class cstring : int -> ostring
```

在这里，将字符串的引用暴露出来基本上没有影响，我们也可以使用3.17的方法把引用隐藏起来。

#### 栈

有时候，我们在表述参数方程的数据类型的时候，模块和类都是可选的。所以，在一些情况下，这两种方法非常相似。举个例子，一个栈可以直接通过类来实现：

```ocaml
# exception Empty;;
exception Empty
```

```ocaml
# class ['a] stack =
    object
      val mutable l = ([] : 'a list)
      method push x = l <- x::l
      method pop = match l with [] -> raise Empty | a::l' -> l <- l'; a
      method clear = l <- []
      method length = List.length l
    end;;
class ['a] stack :
  object
    val mutable l : 'a list
    method clear : unit
    method length : int
    method pop : 'a
    method push : 'a -> unit
  end
```

但是，在书写一个遍历栈的方法时候，我们会遇到一些问题。fold方法的类型是('b -> 'a -> 'b) -> 'b -> 'b。在这里，'a是栈的类型参数，但是'b并不和'a类相关但是却作为参数传给了fold。一个初步的方法就是将'b作为额外参数加入stack类中：

```ocaml
# class ['a, 'b] stack2 =
    object
      inherit ['a] stack
      method fold f (x : 'b) = List.fold_left f x l
    end;;
class ['a, 'b] stack2 :
  object
    val mutable l : 'a list
    method clear : unit
    method fold : ('b -> 'a -> 'b) -> 'b -> 'b
    method length : int
    method pop : 'a
    method push : 'a -> unit
  end
```

可是，fold方法对于给定的对象只能将方法应用于相同的类型：

```ocaml
# let s = new stack2;;
val s : ('_weak1, '_weak2) stack2 = <obj>
```

```ocaml
# s#fold ( + ) 0;;
- : int = 0
```

```ocaml
# s;;
- : (int, int) stack2 = <obj>
```

在OCaml3.05版本中加入的多态方法，就可以很好的解决这个问题。

多态方法让fold方法的类型变量'b可以从全局上验证，并且给予所有fold方法 'b. ('b -> 'a -> 'b) -> 'b -> 'b多态类型。但是在这里，我们需要给fold书写一个明确的类型声明，因为类型检查起不能自己推断多态类型。

```ocaml
# class ['a] stack3 =
    object
      inherit ['a] stack
      method fold : 'b. ('b -> 'a -> 'b) -> 'b -> 'b
                  = fun f x -> List.fold_left f x l
    end;;
class ['a] stack3 :
  object
    val mutable l : 'a list
    method clear : unit
    method fold : ('b -> 'a -> 'b) -> 'b -> 'b
    method length : int
    method pop : 'a
    method push : 'a -> unit
  end
```

### 6.2.2  哈希表

一个简单版本的面向对象哈希表应该具有如下类的类型：

```ocaml
# class type ['a, 'b] hash_table =
    object
      method find : 'a -> 'b
      method add : 'a -> 'b -> unit
    end;;
class type ['a, 'b] hash_table =
  object method add : 'a -> 'b -> unit method find : 'a -> 'b end
```

一个简单的实现例子，对于小规模的哈希表我们可以使用相关列表：

```ocaml
# class ['a, 'b] small_hashtbl : ['a, 'b] hash_table =
    object
      val mutable table = []
      method find key = List.assoc key table
      method add key valeur = table <- (key, valeur) :: table
    end;;
class ['a, 'b] small_hashtbl : ['a, 'b] hash_table
```

一个更好的实现就是，实现一个真正哈希表并且他的元素是小规模哈希表，

```ocaml
# class ['a, 'b] hashtbl size : ['a, 'b] hash_table =
    object (self)
      val table = Array.init size (fun i -> new small_hashtbl)
      method private hash key =
        (Hashtbl.hash key) mod (Array.length table)
      method find key = table.(self#hash key) # find key
      method add key = table.(self#hash key) # add key
    end;;
class ['a, 'b] hashtbl : int -> ['a, 'b] hash_table
```

### 6.2.3  集合

实现集合将问题带向了新的难度。因为在做并运算的时候，我们需要访问同一个类的其他对象引用。

这是在3.17节所提到的友函数的一个实例，这是一种在集合模块里面普遍使用的，用于确定对象缺失的方法。

在面向对象版本的集合中，我们只需要添加tag方法，来返回集合的类型引用。因为集合是依赖元素的的类型来计算类型的，tag方法也拥有一个计算类型'a tag，虽然在模块里是明确的，但是依然是抽象的签名。在外部看来，这样保证了两个拥有同样tag类型的对象会共用一个模块引用。

```ocaml
# module type SET =
    sig
      type 'a tag
      class ['a] c :
        object ('b)
          method is_empty : bool
          method mem : 'a -> bool
          method add : 'a -> 'b
          method union : 'b -> 'b
          method iter : ('a -> unit) -> unit
          method tag : 'a tag
        end
    end;;
```

```ocaml
# module Set : SET =
    struct
      let rec merge l1 l2 =
        match l1 with
          [] -> l2
        | h1 :: t1 ->
            match l2 with
              [] -> l1
            | h2 :: t2 ->
                if h1 < h2 then h1 :: merge t1 l2
                else if h1 > h2 then h2 :: merge l1 t2
                else merge t1 l2
      type 'a tag = 'a list
      class ['a] c =
        object (_ : 'b)
          val repr = ([] : 'a list)
          method is_empty = (repr = [])
          method mem x = List.exists (( = ) x) repr
          method add x = {< repr = merge [x] repr >}
          method union (s : 'b) = {< repr = merge repr s#tag >}
          method iter (f : 'a -> unit) = List.iter f repr
          method tag = repr
        end
    end;;
```

## 6.3  观察者模式

接下来的例子是关于观察者模式的，观察者模式常用于解决有内部关联的复杂继承问题。这种情况的一个普遍模式就是两个互相递归依赖的类。

observer类有一个独立的notify方法，要求传入主体和事件来执行一个动作。

```ocaml
# class virtual ['subject, 'event] observer =
    object
      method virtual notify : 'subject ->  'event -> unit
    end;;
class virtual ['subject, 'event] observer :
  object method virtual notify : 'subject -> 'event -> unit end
```

这个类在实例变量中记录了一个observer列表，还拥有一个独立方法notify_observers，在特定事件e触发时候来广播消息。

```ocaml
# class ['observer, 'event] subject =
    object (self)
      val mutable observers = ([]:'observer list)
      method add_observer obs = observers <- (obs :: observers)
      method notify_observers (e : 'event) =
          List.iter (fun x -> x#notify self e) observers
    end;;
class ['a, 'event] subject :
  object ('b)
    constraint 'a = < notify : 'b -> 'event -> unit; .. >
    val mutable observers : 'a list
    method add_observer : 'a -> unit
    method notify_observers : 'event -> unit
  end
```

复杂的地方在于怎样定义继承了这种模式的实例。在OCaml里，可以通过一种简单的方式进行，正如下面的窗口操作例子一样：

```ocaml
# type event = Raise | Resize | Move;;
type event = Raise | Resize | Move
```

```ocaml
# let string_of_event = function
      Raise -> "Raise" | Resize -> "Resize" | Move -> "Move";;
val string_of_event : event -> string = <fun>
```

```ocaml
# let count = ref 0;;
val count : int ref = {contents = 0}
```

```ocaml
# class ['observer] window_subject =
    let id = count := succ !count; !count in
    object (self)
      inherit ['observer, event] subject
      val mutable position = 0
      method identity = id
      method move x = position <- position + x; self#notify_observers Move
      method draw = Printf.printf "{Position = %d}\n"  position;
    end;;
class ['a] window_subject :
  object ('b)
    constraint 'a = < notify : 'b -> event -> unit; .. >
    val mutable observers : 'a list
    val mutable position : int
    method add_observer : 'a -> unit
    method draw : unit
    method identity : int
    method move : int -> unit
    method notify_observers : event -> unit
  end
```

```ocaml
# class ['subject] window_observer =
    object
      inherit ['subject, event] observer
      method notify s e = s#draw
    end;;
class ['a] window_observer :
  object
    constraint 'a = < draw : unit; .. >
    method notify : 'a -> event -> unit
  end
```

正如所预料的，window的类型是递归定义的

```ocaml
# let window = new window_subject;;
val window : < notify : 'a -> event -> unit; _.. > window_subject as 'a =
  <obj>
```

However, the two classes of window_subject and window_observer are not mutually recursive.

```ocaml
# let window_observer = new window_observer;;
val window_observer : < draw : unit; _.. > window_observer = <obj>
```

```ocaml
# window#add_observer window_observer;;
- : unit = ()
```

```ocaml
# window#move 1;;
{Position = 1}
- : unit = ()
```

window_observer和window_subject两个类依然可以通过继承拓展。下面这个例子，丰富了窗口主体，改进了observer的行为。

```
# class ['observer] richer_window_subject =
    object (self)
      inherit ['observer] window_subject
      val mutable size = 1
      method resize x = size <- size + x; self#notify_observers Resize
      val mutable top = false
      method raise = top <- true; self#notify_observers Raise
      method draw = Printf.printf "{Position = %d; Size = %d}\n"  position size;
    end;;
class ['a] richer_window_subject :
  object ('b)
    constraint 'a = < notify : 'b -> event -> unit; .. >
    val mutable observers : 'a list
    val mutable position : int
    val mutable size : int
    val mutable top : bool
    method add_observer : 'a -> unit
    method draw : unit
    method identity : int
    method move : int -> unit
    method notify_observers : event -> unit
    method raise : unit
    method resize : int -> unit
  end
```

```ocaml
# class ['subject] richer_window_observer =
    object
      inherit ['subject] window_observer as super
      method notify s e = if e <> Raise then s#raise; super#notify s e
    end;;
class ['a] richer_window_observer :
  object
    constraint 'a = < draw : unit; raise : unit; .. >
    method notify : 'a -> event -> unit
  end
```

我们也可以创建另一种observer：

```ocaml
# class ['subject] trace_observer =
    object
      inherit ['subject, event] observer
      method notify s e =
        Printf.printf
          "<Window %d <== %s>\n" s#identity (string_of_event e)
    end;;
class ['a] trace_observer :
  object
    constraint 'a = < identity : int; .. >
    method notify : 'a -> event -> unit
  end
```

并且可以将多个observer连接到同一个对象上：

```ocaml
# let window = new richer_window_subject;;
val window :
  < notify : 'a -> event -> unit; _.. > richer_window_subject as 'a = <obj>
```

```ocaml
# window#add_observer (new richer_window_observer);;
- : unit = ()
# window#add_observer (new trace_observer);;
- : unit = ()
# window#move 1; window#resize 2;;
<Window 1 <== Move>
<Window 1 <== Raise>
{Position = 1; Size = 1}
{Position = 1; Size = 1}
<Window 1 <== Resize>
<Window 1 <== Raise>
{Position = 1; Size = 3}
{Position = 1; Size = 3}
- : unit = ()
```