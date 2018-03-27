# Chapter 6  Advanced examples with classes and modules

- [6.1  Extended example: bank accounts](https://caml.inria.fr/pub/docs/manual-ocaml/advexamples.html#sec62)
- [6.2  Simple modules as classes](https://caml.inria.fr/pub/docs/manual-ocaml/advexamples.html#sec63)
- [6.3  The subject/observer pattern](https://caml.inria.fr/pub/docs/manual-ocaml/advexamples.html#sec68)

(Chapter written by Didier Rémy)

In this chapter, we show some larger examples using objects, classes and modules. We review many of the object features simultaneously on the example of a bank account. We show how modules taken from the standard library can be expressed as classes. Lastly, we describe a programming pattern known as *virtual types* through the example of window managers.

## 6.1  Extended example: bank accounts

In this section, we illustrate most aspects of Object and inheritance by refining, debugging, and specializing the following initial naive definition of a simple bank account. (We reuse the module Euro defined at the end of chapter [3](https://caml.inria.fr/pub/docs/manual-ocaml/objectexamples.html#c%3Aobjectexamples).)

```
 let euro = new Euro.c;;

val euro : float -> Euro.c = <fun>

```

```
 let zero = euro 0.;;

val zero : Euro.c = <obj>

```

```
 let neg x = x#times (-1.);;

val neg : < times : float -> 'a; .. > -> 'a = <fun>

```

```
 class account =
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

```
 let c = new account in c # deposit (euro 100.); c # withdraw (euro 50.);;

- : Euro.c = <obj>

```

We now refine this definition with a method to compute interest.

```
 class account_with_interests =
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

We make the method interest private, since clearly it should not be called freely from the outside. Here, it is only made accessible to subclasses that will manage monthly or yearly updates of the account.

We should soon fix a bug in the current definition: the deposit method can be used for withdrawing money by depositing negative amounts. We can fix this directly:

```
 class safe_account =
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

However, the bug might be fixed more safely by the following definition:

```
 class safe_account =
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

In particular, this does not require the knowledge of the implementation of the method deposit.

To keep track of operations, we extend the class with a mutable field history and a private method trace to add an operation in the log. Then each method to be traced is redefined.

```
 type 'a operation = Deposit of 'a | Retrieval of 'a;;

type 'a operation = Deposit of 'a | Retrieval of 'a

```

```
 class account_with_history =
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

One may wish to open an account and simultaneously deposit some initial amount. Although the initial implementation did not address this requirement, it can be achieved by using an initializer.

```
 class account_with_deposit x =
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

A better alternative is:

```
 class account_with_deposit x =
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

Indeed, the latter is safer since the call to deposit will automatically benefit from safety checks and from the trace. Let’s test it:

```
 let ccp = new account_with_deposit (euro 100.) in
  let _balance = ccp#withdraw (euro 50.) in
  ccp#history;;

- : Euro.c operation list = [Deposit <obj>; Retrieval <obj>]

```

Closing an account can be done with the following polymorphic function:

```
 let close c = c#withdraw c#balance;;

val close : < balance : 'a; withdraw : 'a -> 'b; .. > -> 'b = <fun>

```

Of course, this applies to all sorts of accounts.

Finally, we gather several versions of the account into a module Account abstracted over some currency.

```
 let today () = (01,01,2000) (* an approximation *)
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

This shows the use of modules to group several class definitions that can in fact be thought of as a single unit. This unit would be provided by a bank for both internal and external uses. This is implemented as a functor that abstracts over the currency so that the same code can be used to provide accounts in different currencies.

The class bank is the *real* implementation of the bank account (it could have been inlined). This is the one that will be used for further extensions, refinements, etc. Conversely, the client will only be given the client view.

```
 module Euro_account = Account(Euro);;


```

```
 module Client = Euro_account.Client (Euro_account);;


```

```
 new Client.account (new Euro.c 100.);;


```

Hence, the clients do not have direct access to the balance, nor the history of their own accounts. Their only way to change their balance is to deposit or withdraw money. It is important to give the clients a class and not just the ability to create accounts (such as the promotional discount account), so that they can personalize their account. For instance, a client may refine the deposit and withdraw methods so as to do his own financial bookkeeping, automatically. On the other hand, the function discountis given as such, with no possibility for further personalization.

It is important to provide the client’s view as a functor Client so that client accounts can still be built after a possible specialization of the bank. The functor Client may remain unchanged and be passed the new definition to initialize a client’s view of the extended account.

```
 module Investment_account (M : MONEY) =
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

The functor Client may also be redefined when some new features of the account can be given to the client.

```
 module Internet_account (M : MONEY) =
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

## 6.2  Simple modules as classes

One may wonder whether it is possible to treat primitive types such as integers and strings as objects. Although this is usually uninteresting for integers or strings, there may be some situations where this is desirable. The class money above is such an example. We show here how to do it for strings.

### 6.2.1  Strings

A naive definition of strings as objects could be:

```
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

However, the method escaped returns an object of the class ostring, and not an object of the current class. Hence, if the class is further extended, the method escaped will only return an object of the parent class.

```
 class sub_string s =
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

As seen in section [3.16](https://caml.inria.fr/pub/docs/manual-ocaml/objectexamples.html#ss%3Abinary-methods), the solution is to use functional update instead. We need to create an instance variable containing the representation s of the string.

```
 class better_string s =
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

As shown in the inferred type, the methods escaped and sub now return objects of the same type as the one of the class.

Another difficulty is the implementation of the method concat. In order to concatenate a string with another string of the same class, one must be able to access the instance variable externally. Thus, a method repr returning s must be defined. Here is the correct definition of strings:

```
 class ostring s =
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

Another constructor of the class string can be defined to return a new string of a given length:

```
 class cstring n = ostring (String.make n ' ');;

class cstring : int -> ostring

```

Here, exposing the representation of strings is probably harmless. We do could also hide the representation of strings as we hid the currency in the class money of section [3.17](https://caml.inria.fr/pub/docs/manual-ocaml/objectexamples.html#ss%3Afriends).

#### Stacks

There is sometimes an alternative between using modules or classes for parametric data types. Indeed, there are situations when the two approaches are quite similar. For instance, a stack can be straightforwardly implemented as a class:

```
 exception Empty;;

exception Empty

```

```
 class ['a] stack =
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

However, writing a method for iterating over a stack is more problematic. A method fold would have type ('b -> 'a -> 'b) -> 'b -> 'b. Here 'a is the parameter of the stack. The parameter 'b is not related to the class 'a stack but to the argument that will be passed to the method fold. A naive approach is to make 'b an extra parameter of class stack:

```
 class ['a, 'b] stack2 =
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

However, the method fold of a given object can only be applied to functions that all have the same type:

```
 let s = new stack2;;

val s : ('_weak1, '_weak2) stack2 = <obj>

```

```
 s#fold ( + ) 0;;

- : int = 0

```

```
 s;;

- : (int, int) stack2 = <obj>

```

A better solution is to use polymorphic methods, which were introduced in OCaml version 3.05. Polymorphic methods makes it possible to treat the type variable 'b in the type of fold as universally quantified, giving fold the polymorphic type Forall 'b. ('b -> 'a -> 'b) -> 'b -> 'b. An explicit type declaration on the method fold is required, since the type checker cannot infer the polymorphic type by itself.

```
 class ['a] stack3 =
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

### 6.2.2  Hashtbl

A simplified version of object-oriented hash tables should have the following class type.

```
 class type ['a, 'b] hash_table =
    object
      method find : 'a -> 'b
      method add : 'a -> 'b -> unit
    end;;

class type ['a, 'b] hash_table =
  object method add : 'a -> 'b -> unit method find : 'a -> 'b end

```

A simple implementation, which is quite reasonable for small hash tables is to use an association list:

```
 class ['a, 'b] small_hashtbl : ['a, 'b] hash_table =
    object
      val mutable table = []
      method find key = List.assoc key table
      method add key valeur = table <- (key, valeur) :: table
    end;;

class ['a, 'b] small_hashtbl : ['a, 'b] hash_table

```

A better implementation, and one that scales up better, is to use a true hash table… whose elements are small hash tables!

```
 class ['a, 'b] hashtbl size : ['a, 'b] hash_table =
    object (self)
      val table = Array.init size (fun i -> new small_hashtbl)
      method private hash key =
        (Hashtbl.hash key) mod (Array.length table)
      method find key = table.(self#hash key) # find key
      method add key = table.(self#hash key) # add key
    end;;

class ['a, 'b] hashtbl : int -> ['a, 'b] hash_table

```

### 6.2.3  Sets

Implementing sets leads to another difficulty. Indeed, the method union needs to be able to access the internal representation of another object of the same class.

This is another instance of friend functions as seen in section [3.17](https://caml.inria.fr/pub/docs/manual-ocaml/objectexamples.html#ss%3Afriends). Indeed, this is the same mechanism used in the module Set in the absence of objects.

In the object-oriented version of sets, we only need to add an additional method tag to return the representation of a set. Since sets are parametric in the type of elements, the method tag has a parametric type 'a tag, concrete within the module definition but abstract in its signature. From outside, it will then be guaranteed that two objects with a method tag of the same type will share the same representation.

```
 module type SET =
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

```
 module Set : SET =
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

## 6.3  The subject/observer pattern

The following example, known as the subject/observer pattern, is often presented in the literature as a difficult inheritance problem with inter-connected classes. The general pattern amounts to the definition a pair of two classes that recursively interact with one another.

The class observer has a distinguished method notify that requires two arguments, a subject and an event to execute an action.

```
 class virtual ['subject, 'event] observer =
    object
      method virtual notify : 'subject ->  'event -> unit
    end;;

class virtual ['subject, 'event] observer :
  object method virtual notify : 'subject -> 'event -> unit end

```

The class subject remembers a list of observers in an instance variable, and has a distinguished method notify_observers to broadcast the message notify to all observers with a particular event e.

```
 class ['observer, 'event] subject =
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

The difficulty usually lies in defining instances of the pattern above by inheritance. This can be done in a natural and obvious manner in OCaml, as shown on the following example manipulating windows.

```
 type event = Raise | Resize | Move;;

type event = Raise | Resize | Move

```

```
 let string_of_event = function
      Raise -> "Raise" | Resize -> "Resize" | Move -> "Move";;

val string_of_event : event -> string = <fun>

```

```
 let count = ref 0;;

val count : int ref = {contents = 0}

```

```
 class ['observer] window_subject =
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

```
 class ['subject] window_observer =
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

As can be expected, the type of window is recursive.

```
 let window = new window_subject;;

val window : < notify : 'a -> event -> unit; _.. > window_subject as 'a =
  <obj>

```

However, the two classes of window_subject and window_observer are not mutually recursive.

```
 let window_observer = new window_observer;;

val window_observer : < draw : unit; _.. > window_observer = <obj>

```

```
 window#add_observer window_observer;;

- : unit = ()

```

```
 window#move 1;;

{Position = 1}
- : unit = ()

```

Classes window_observer and window_subject can still be extended by inheritance. For instance, one may enrich the subject with new behaviors and refine the behavior of the observer.

```
 class ['observer] richer_window_subject =
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

```
 class ['subject] richer_window_observer =
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

We can also create a different kind of observer:

```
 class ['subject] trace_observer =
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

and attach several observers to the same object:

```
 let window = new richer_window_subject;;

val window :
  < notify : 'a -> event -> unit; _.. > richer_window_subject as 'a = <obj>

```

```
 window#add_observer (new richer_window_observer);;

- : unit = ()

 window#add_observer (new trace_observer);;

- : unit = ()

 window#move 1; window#resize 2;;

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