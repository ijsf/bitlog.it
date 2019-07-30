# A Basic Introduction to Clash for FPGA Development - Part 1

_2018/03/20 Oguz Meteer // guztech_

---

***This post is part of a series:***

* [A Basic Introduction to Clash for FPGA Development - Part 1](20180320_a_basic_introduction_to_clash_for_fpga_development_part_1.md)
* [A Basic Introduction to Clash for FPGA Development - Part 2](20180514_a_basic_introduction_to_clash_for_fpga_development_part_2.md)

For FPGA development, VHDL was my default language for a while. That started to shift towards Verilog when I discovered [Yosys-SMTBMC](http://www.clifford.at/papers/2016/yosys-smtbmc/) created by [Clifford Wolf](http://www.clifford.at/), which is a formal verification tool for hardware. However, recently I've been exploring other HDL languages and one that peaked my interest is [Clash](http://www.clash-lang.org/). It stands for *[CAES](https://www.utwente.nl/en/eemcs/caes/) Language for Synchronous Hardware*, and it is built upon Haskell. Its compiler is [open-source](https://github.com/clash-lang/clash-compiler) and is mainly developed by the people at [QBayLogic](https://qbaylogic.com). Since I am doing my PhD at the [CAES](https://www.utwente.nl/en/eemcs/caes/) group at the [University of Twente](https://www.utwente.nl/), it might seem as an obvious choice to explore Clash. But having very little experience with functional programming, this seemed challenging to me. Could I really develop hardware in a language that is very foreign to me?

I think it is safe to say that not many hardware developers for FPGAs have lots of functional programming experience. But don't let that stop you! I want to show you that it is totally doable. You only need to understand a few gotcha's which I will describe and this and future blog posts.

***Disclaimer:*** I'm not a Haskell expert at all, and my explanations will most likely be oversimplified and perhaps not entirely correct for the sake of not over-complicating things. Even so, for this first post on Clash, I do try to explain the implementation details very thoroughly.

***Reminder:*** All the code for this post and future posts can be found on [GitLab](https://gitlab.com/GuzTech/bitlog_clash_tutorial) and [Github](https://github.com/GuzTech/bitlog_clash_tutorial/tree/master/). The code for this part is in the [part\_1](https://github.com/GuzTech/bitlog_clash_tutorial/tree/master/part_1) folder.

Why a functional programming language as an HDL?
================================================

A pure functional programming language guarantees that functions have no side-effects, i.e. they do not alter nor depend on some ***global state*** or time. This means that each time that you call a function with the same input, you will always get the same output, which is a nice property to have. If we look at hardware, you often describe combinatorial and synchronous processes that take some input, and produce some output, i.e. they take some state, and produce a new state. Each input and output is explicitly defined (think of the sensitivity list of your processes/always blocks), and if you give it a specific input each time, you get the same output every single time. This kind of sounds like a pure functional programming language, doesn't it?

When developing in Clash (and Haskell), think of functions as things that transform one to state to another state. What this means is that you can separate your ***specification*** from your ***implementation***. In other words, you describe ***what*** a function must do instead of ***how*** it must do it. When you specify something, you abstract away the details of the platform it runs on, whereas with an implementation you have to take the platform into account. They might seem the same, but hopefully I can show the differences in this and future posts.

Setting things up
=================

When exploring something new, it is often a good idea to start with the basics. Since Clash is built upon Haskell, we will need to understand the syntax first. I can definitely recommend [Learn You A Haskell For Great Good](http://learnyouahaskell.com/), [A brief introduction to Haskell](https://wiki.haskell.org/A_brief_introduction_to_Haskell), and [Functional Programming lecture recordings](https://vimeo.com/groups/460834/sort:alphabetical/format:thumbnail) by Jan Kuper, one of the originators of Clash.

Next we need to install Clash. For this post I will be using the latest release as of this writing which is 0.7.2. It requires version 8.0.2 of the Glasgow Haskell Compiler (GHC) but because I'm running Arch Linux, the version of GHC in the repositories is 8.2.2-1. That's why I use [Stack](https://docs.haskellstack.org/en/stable/README/) to create a sandbox with the correct version of GHC and Clash. First, install Stack using your package manager. Using [stackage.org](http://stackage.org) we can find the lts version that contains the latest GHC 8.0 release which is 9.21. We can now use Stack to create a sandbox by running the following commands:

```
stack setup --resolver lts-9.21
stack install --resolver lts-9.21 clash-ghc-0.7.2
```

To start the Clash interactive environment, run:

```
stack exec clashi
```

You should be greeted with the following prompt:

```
CLaSHi, version 0.7.2 (using clash-lib, version 0.7.1):
http://www.clash-lang.org/  :? for help
CLaSH.Prelude>
```

A Simple Example
================

***Recommended reading material: [Starting Out](http://learnyouahaskell.com/starting-out), [Types and Typeclasses](http://learnyouahaskell.com/types-and-typeclasses)***

Let's start with a simple example. Create a project folder and start Clash in that folder. Then create a file named ***Example1.hs*** with the following contents:

```
module Example1 where
import CLaSH.Prelude

counter val = val + 1
counter2 val = adder val 1

adder val1 val2 = val1 + val2

topEntity = counter
```

You can load and reload a file using `:l Example1` and `:r` respectively.

Haskell programs consist of modules, and in line 1 we describe the name of the module. It is recommended for the module name to match the name of the file (Example1.hs in this example). The second line imports the [***CLaSH.Prelude***](http://hackage.haskell.org/package/clash-prelude-0.11.2/docs/CLaSH-Prelude.html) module which can be thought of the standard library of Clash. That module contains types and functions that can be used to generate synthesizable hardware. The other lines create four functions named ***counter***, ***counter2***, ***adder***, and ***topEntity*** (functions start with a lower-case letter).

In this example, both functions describe combinatorial circuits, with ***counter*** and ***counter2*** having one input and one output, and ***adder*** having two inputs and an output. If you have noticed, we haven't described the types of the inputs and output. We can ask the type of our functions using `:t <function name>`:

```
*Example1> :t counter
counter :: Num a => a -> a
*Example1> :t counter2
counter2 :: Num a => a -> a
*Example1> :t adder
adder :: Num a => a -> a -> a
*Example1> :t topEntity
topEntity :: Num a => a -> a
```

Let's dissect the results by breaking them up into three parts.

1.  The part ***before*** the `::` is the name of the function.
2.  The part ***after*** the `=>` describes the inputs and the output. In Haskell, a function can have zero or more inputs, and only one output. The inputs and output are seperated with a singular arrow (`->`), and since there can only be one output, the very last thing after the last `->` is the type of the output.
3.  The part between the `::` and the `=>` describe to which class(es) a type belongs to.

If we look at the type definition of the ***counter*** function, we see that it has one input of type `a`, and one output which is also of type `a`. Also, type `a` belongs to the `Num` class of types, which is any numeric type. The ***adder*** function is very similar, except that it has two inputs (denoted by the `a -> a ->` part). ***counter2*** is an alternative version that uses the ***adder*** function, where the input ***val*** is passed on to ***adder*** and the second input to ***adder*** is fixed at 1. ***topEntity*** should look familiar to those that use VHDL/Verilog. It is used by Clash to define the top entity that will be evaluated when we want to generate VHDL and (System)Verilog. In our example, this means that our top entity is the ***counter*** function, and therefore it has the same type. You can see that I've not specified the input and output for ***topEntity***, because Haskell understands that the type definition of ***topEntity*** is the same as that of ***counter***. We could have also written:

```
topEntity val = counter val
```

It is functionally equivalent to writing it the way we wrote it before, and depends on your preference. In this case, the type of counter is so trivial, that I didn't bother writing it more explicitly.

Haskell programmers will most likely say "That's not a hardware description, that's just a normal Haskell function!", and they are right. Since Clash is built upon Haskell, it basically **is** Haskell with additional functions and types that can be synthesized. Now, Haskell actually managed to deduce that ***a*** should be a numeric type since we use addition in the function, but it keeps the type as "generic" as possible. This is called ***polymorphic*** in Haskell, since it can accept multiple types. We could have specified by hand that our functions only take `Float`s  and returns a `Float`, but then they would not be able to add `Integer`s.

While it is nice that our functions are polymorphic, they cannot be synthesized because Clash needs to know the exact type of `a`, i.e. it has to be in ***normal form***. `Num` is a class of numbers, but it isn't a specific type. What should it generate? Something that takes signed/unsigned integers? How many bits should be used to represent the numbers? Since there is (luckily?) no support for mind reading in Clash, we have specify it ourselves. In other words, the highest function (***topEntity*** in our case) ***cannot be polymorphic***, as Clash needs to know what to generate. Change the file to:

```
module Example1 where
import CLaSH.Prelude

counter val = val + 1
counter2 val = adder val 1

adder val1 val2 = val1 + val2

topEntity :: Integer -> Integer
topEntity = counter
```

Now we specify that the input and output of our top entity are of type `Integer`. Why did add the type specification to ***topEntity*** and not ***counter***? Sure, we could have done that, but in general it is better to specify concrete types in the outer layers of your design. Changing the types throughout your design becomes easier since you have to change it in fewer places, while still keeping the inner layers polymorphic.

`Integer` is a Haskell type which is 64 bits on my machine, and if we were to generate VHDL/Verilog by issuing `:vhdl` and `:verilog` respectively, we would see that the input and output type would be 64 bit signed numbers. However, you most likely want to be able to control the type and size of the inputs and outputs. Luckily, Clash has some [types built in](http://hackage.haskell.org/package/clash-prelude-0.11.2/docs/CLaSH-Prelude.html#g:13) that we can use:

-   `Bit` and `BitVector n`: these are similar to std\_(u)logic and std\_(u)logic\_vector in VHDL, and represent raw bits. Here, the `n` specifies the size of the`BitVector`.
-   `Signed n` and `Unsigned n`: as the names imply, they represent `n` bit signed and unsigned integers.
-   `SFixed int frac` and `UFixed int frac`: these are signed and unsigned fixed-point representations with `int` bits for the integer part and `frac` bits for the fractional part respectively.
-   `Vec n t`: it is a vector of size `n` that stores things of type `t`.

If we now change our file to:

```
module Example1 where
import CLaSH.Prelude

counter val = val + 1
counter2 val = adder val 1

adder val1 val2 = val1 + val2

topEntity :: Unsigned 10 -> Unsigned 10
topEntity = counter
```

and generate VHDL/Verilog, then we would see that the input and output type would be 10 bit unsigned numbers.

If we wanted to use the ***adder*** function as our top entity, how should we change the type definition of ***topEntity***? If you recall, the type definition of the ***adder*** function is `adder :: Num a => a -> a -> a`. Let's say we want to use 16 bit signed values for the inputs and output. We can accomplish this by changing the type and implementation of ***topEntity*** to:

```
topEntity :: Signed 16 -> Signed 16 -> Signed 16
topEntity = adder
```

Evaluating our functions
========================

One of the benefits of Clash is that you can evaluate your functions using the [REPL](https://en.wikipedia.org/wiki/Read–eval–print_loop) provided by the interactive environment. This might not seem useful in our example since the functions are trivial, but when you have larger functions then its usefulness increases a lot.

Let's evaluate our ***counter*** function first:

```
*Example1> counter 4
5
```

Since we didn't provide a type definition for ***counter***, it will take anything that belongs to the `Num` class, which ***4*** is:

```
*Example1> :t 4
4 :: Num t => t
```

If we want to test it with a different type (such as an `Unsigned 3` for example), then we can specify what the type of our input is like this:

```
*Example1> counter (4 :: Unsigned 3)
5
```

We can check what the type of the result is, even though we know it has to be the same type as that of the input:

```
*Example1> :t counter (4 :: Unsigned 3)
counter (4 :: Unsigned 3) :: Unsigned 3
```

This shows us that if we invoke ***counter*** with an input that has the type `Unsigned 3`, then the output will also have the type `Unsigned 3`. We could have also chosen to specify a type definition for ***counter*** itself as we did for ***topEntity***. Because the types for ***topEntity*** have been defined, we do not have to specify them when evaluating the function:

```
*Example1> topEntity 3
4
*Example1> topEntity (-1)
0
*Example1> :t topEntity (-1)
topEntity (-1) :: Signed 16
```

Note that for negative numbers, we have to place the number between brackets [because the minus sign is a unary function](https://stackoverflow.com/questions/26073878/why-it-is-impossible-to-multiply-negative-numbers-in-haskell-without-brackets).

In the same way, we can also evaluate ***adder***:

```
*Example1> adder (3 :: Signed 3) (5 :: Signed 3)
0
*Example1> :t adder (3 :: Signed 3) (5 :: Signed 3)
adder (3 :: Signed 3) (5 :: Signed 3) :: Signed 3
*Example1> adder (3 :: Signed 4) (5 :: Signed 4)
-8
*Example1> :t adder (3 :: Signed 4) (5 :: Signed 4)
adder (3 :: Signed 4) (5 :: Signed 4) :: Signed 4
```

Here we can see that if we add 3 and 5 together, the result is zero, since a 3 bit signed number can only represent numbers between [-4, 3]. Similarly, the result for a 4 bit signed number would be -8 because of the wrap around.

Pattern matching
================

***Recommended reading material: [Syntax in Functions](http://learnyouahaskell.com/syntax-in-functions)***

We have defined two very simple functions, but now we want to use an enable signal to control the counter. Here is an implementation that will do just that:

```
counter2 :: Num t => t -> Bool -> t
counter2 val enable = case enable of
  True  -> val + 1
  False -> val
```

I've also added the type definition that Haskell deduced. Since we pattern match ***enable*** with ***True*** and ***False***, Haskell understands that enable must be of type ***Bool***. Pattern matching is very powerful, even if it doesn't really show when using it for a Bool. However, in future posts we will use types that have more than two values and hopefully then it will become apparent that they are very useful.

If we want our top entity to use our new counter, the resulting file becomes:

```
module Example1 where
import CLaSH.Prelude

counter val = val + 1
counter2 val = adder val 1

counter3 val enable = case enable of
  True  -> val + 1
  False -> val

counter4 val enable = case enable of
  True  -> adder val 1
  False -> val

adder val1 val2 = val1 + val2

topEntity :: Signed 16 -> Bool -> Signed 16
topEntity = counter3
```

We can evaluate our new counter (I will use ***topEntity*** so that I don't have to specify the type of the inputs):

```
*Example1> topEntity 3 True
4
*Example1> :t topEntity 3 True
topEntity 3 True :: Signed 16
*Example1> topEntity 6 False
6
```

Just to introduce some useful syntax (the ***where*** clause), we could also write the same function like this:

```
counter3 val enable = o
  where
    o = case enable of
      True  -> val + 1
      False -> val
```

In this version we introduce ***o***, where ***o*** is either ***val + 1*** or ***val*** depending on ***enable***. In this function, it doesn't make much sense to do this, but in functions that are more complex the ***where*** clause becomes very useful. We can demonstrate this with a function that either adds or subtracts based on a boolean:

```
addsub :: Num t => t -> t -> Bool -> t
addsub val1 val2 a_ns = o
  where
    res_add = val1 + val2
    res_sub = val1 - val2
    o = case a_ns of
      True  -> res_add
      False -> res_sub
```

Again, I've added the type specification that Haskell deduced. We have three inputs (`t -> t -> Bool`) where ***t*** is a ***Num***, ***a\_ns*** is a ***Bool*** that determines if we should add or subtract, and the output is also of type ***t***. We could have written is more concisely, but some might consider this more readable. We use the ***where*** clause to bind the two possible results to the names ***res\_add*** and ***res\_sub***.

A circular stack
================

Let's create something a little bit more complex, but still trivial enough to understand by simply looking at the function. We will create a [stack](https://en.wikipedia.org/wiki/Stack_(abstract_data_type)), which is a piece of memory where you can push thing onto it, and pop things off of it. A stack pointer is used to keep track of where in memory data is pushed or popped. More specifically, we will implement a circular stack, where the stack pointer simply wraps around when push or pop more times back to back than the size of the stack. Here is an initial, ugly version:

```
stack1 (mem, sp) push pop value = ((mem', sp'), o)
  where
    sp' = case push of
      True  -> case pop of
        True  -> sp
        False -> sp + 1
      False -> case pop of
        True  -> sp - 1
        False -> sp
    mem' = case push of
      True  -> replace sp value mem
      False -> mem
    o = case pop of
      True -> mem !! sp'
      _    -> 0
```

Let's first look at the type:

```
*Example1> :t stack1
stack1
  :: (Enum t1, KnownNat n, Num t1, Num t) =>
     (Vec n t, t1) -> Bool -> Bool -> t -> ((Vec n t, t1), t)
```

Now don't let that overwhelm you! Let's dissect the type of the ***stack1*** function. The part after the `=>` shows that we have 4 inputs of types (given as `name` - `type`):

1.  (`(mem, sp)` - `(Vec n t, t1)`): The first input is a ***tuple***, which is denoted by comma separated values within brackets. It holds two things:
    1.  (`mem - Vec n t)`, which is a vector of size `n` that stores things of type `t`. `n` belongs to the class `KnownNat n`, which is a natural number that has to have an actual value when we want to generate VHDL/Verilog. `t` belongs to the class `Num`, which we have seen before.
    2.  (`sp` - `t1`): the stack pointer `sp` belongs to the classes , `Enum t1`, and `Num t1`. Again, we have seen `Num` already, and `Enum` is something that is enumerable.

2.  (`push` - `Bool`): this determines if we want to push something onto the stack.
3.  (`pop` - `Bool`): this determines if we want to pop something off of the stack.
4.  (`value` - `t`): `value` is what we push onto the stack if `push` is ***True***. Its type is `t`, which is the same type as the things that are in `mem`.

The return type is a ***tuple***:

1.  `(mem', sp')`: the first element of the output is also a tuple and contains two things:
    1.  (`mem'` - `Vec n t1`): `mem'` (note the prime) is the new state of the memory and has the same type as `mem`. This of course makes sense since it is the same ***thing*** except that it might contain other values in its new state.
    2.  (`sp'` - `t`): `sp'` (again, note the prime) is the new state of the stack pointer and has the same type as `sp`. Note that it is not necessary to give the outputs the same name as the inputs appended with a prime, we can give them any unused name we want.

Here it becomes much more clear what I meant in the beginning where Haskell (and therefore Clash) allows you to separate your ***specification*** from your ***implementation***. ***stack1*** is a function that takes a certain state and returns a new state. We do not specify what kind of memory we are using, how big it is, what the type of the values we want to push onto our stack is, and so on, because those are implementation details. We only specify ***what*** this function should do, which is to conditionally update a value in `mem`, and conditionally return the top value on the stack. And *when* we actually choose what the types of our functions will be, *then* we obtain an implementation.

Let's go back to the function itself. I have grouped the `mem` and `sp` together in a tuple, because I consider those the *internal state* of the stack. This doesn't mean that the function stores any internal state though, because this function describes a combinatorial circuit. Think of it as a logical grouping of things that belong together. In later posts I will cover registers that can store state, and it will become much more apparent why I did it like this (**hint**: look up a Mealy machine). The other inputs are self explanatory I think. For the output, I again combined the new state of the memory and stack pointer in a tuple `(mem', sp')`, and also we have some output `o`.

The first two lines

```
stack1 (mem, sp) push pop value = ((mem', sp'), o)
  where
```

describe a function ***stack1*** that takes 4 inputs (1 tuple, 2 bools and a value), and introduces three new names `mem'`, `sp'`, and `o`. Next, we bind a value to `sp'`depending on the state of `push` and `pop`:

```
    sp' = case push of
      True  -> case pop of
        True  -> sp
        False -> sp + 1
      False -> case pop of
        True  -> sp - 1
        False -> sp
```

If we only want to push or pop, then the stack pointer will be increased or decreased by one respectively. But if we want to push and pop at the same time, then the stack pointer will remain the same because the increase and decrease of the stack pointer would cancel each other out.

Next up is the memory:

```
    mem' = case push of
      True  -> replace sp value mem
      False -> mem
```

If we don't push anything on the stack then we just bind `mem` to `mem'`, since it doesn't change the state of the memory. However, when we push then we have to modify the state of the memory. That is what the [***replace***](http://hackage.haskell.org/package/clash-prelude-0.11.2/docs/CLaSH-Sized-Vector.html#v:replace) function does:

```
*Example1> :t replace
replace :: (Enum i, KnownNat n) => i -> a -> Vec n a -> Vec n a
```

It takes an index `i`, a value `a`, and a vector, and returns a new vector where the value at index `i` is replaced by value `a`. And that is exactly what we want!

Finally, we bind a value to `o`:

```
    o = case pop of
      True  -> mem !! sp'
      _     -> 0
```

If we want to pop, then `o` gets bound the value of the memory location pointed to by the new stack pointer `sp'`. It will become apparent why I didn't use the old stack pointer `sp` to index the memory when we evaluate our function in a bit. To actually get the value at a specific index of a vector (remember, `mem` is a vector), we use the ***!!*** function:

```
*Example1> :t (!!)
(!!) :: (Enum i, KnownNat n) => Vec n a -> i -> a
```

It takes a vector of size `n` that holds things of type `a`, an index `i`, and it returns something of type `a`. The reason why I place brackets around the function when I ask for the type of the ***!!*** function is because it is an *infix* function whereas normally, functions in Haskell are *prefix* functions.

Evaluating our stack
====================

We can evaluate our stack like so:

```
*Example1> stack1 ((0 :> 0 :> 0 :> 0 :> Nil), 0 :: Unsigned 2) True False 42
((<42,0,0,0>,1),0)
*Example1> stack1 (((0 :: Signed 16) :> 0 :> 0 :> 0 :> Nil), 0 :: Unsigned 2) True False 42
((<42,0,0,0>,1),0)
```

The first input is a tuple, and the first element of that tuple is a vector. We can create vectors by hand using the following syntax:

```
(<value1> :> <value2> :> ... :> <valueN> :> Nil)
```

As you can see, I've evaluated the function twice, with a small difference between both. Why did I do this? Well, let's look at the types of both:

```
*Example1> :t stack1 ((0 :> 0 :> 0 :> 0 :> Nil), 0 :: Unsigned 2) True False 42
stack1 ((0 :> 0 :> 0 :> 0 :> Nil), 0 :: Unsigned 2) True False 42
  :: Num t => ((Vec 4 t, Unsigned 2), t)

*Example1> :t stack1 (((0 :: Signed 16) :> 0 :> 0 :> 0 :> Nil), 0 :: Unsigned 2) True False 42
stack1 (((0 :: Signed 16) :> 0 :> 0 :> 0 :> Nil), 0 :: Unsigned 2) True False 42
  :: ((Vec 4 (Signed 16), Unsigned 2), Signed 16)
```

In the first version, I didn't specify the type of the values in the vector so Haskell deduced that its type is a `Num`. We have seen before that we cannot generate hardware if we do not specify a concrete type for our vector. In the second version, I only specify the type for the first value in the vector `(0 :: Signed 16)`, because Haskell knows that the types of all the elements in a vector are the same.

Because our vector has a size of 4, I specified the type of the stack pointer to be `Unsigned 2` so that we cannot address beyond its size. Furthermore, we set `push` to ***True*** and `pop` to ***False***, and the value that we want to push is 42 (very original, I know).

The resulting new state is `((<42,0,0,0>,1),0)`:

1.  `<42, 0, 0, 0>` is the contents of the vector
2.  `1` is the value of the stack pointer after we have applied the ***stack1*** function, i.e. it is the next state of the stack pointer `sp'`.
3.  `0` is the new value of `o`. Since we didn't pop anything, it is bound to 0 just like we specified.

But what if we want to test with a bigger vector? Do we have to type a bunch of zeroes or other values? Luckily, there is a function [***repeat***](http://hackage.haskell.org/package/clash-prelude-0.11.2/docs/CLaSH-Sized-Vector.html#v:repeat) that takes care of this for us:

```
*Example1> :t repeat
repeat :: KnownNat n => a -> Vec n a
```

It takes something of type `a` and returns a vector of size `n` storing things of type `a`. Let's use the ***repeat*** function to make our lives easier, and let's also bind the new state to something (***x*** in this case):

```
*Example1> x = stack1 (repeat 0 :: Vec 8 (Signed 16), 0 :: Unsigned 3) True False 3
*Example1> x
((<3,0,0,0,0,0,0,0>,1),0)
*Example1> :t x
x :: ((Vec 8 (Signed 16), Unsigned 3), Signed 16)
```

We can also go from this new state stored in ***x*** to a different state. Let's try it by popping the value that we just pushed:

```
*Example1> y = stack1 (fst x) False True 0
*Example1> y
((<3,0,0,0,0,0,0,0>,0),3)
```

Because the output of the ***stack1*** function is a tuple, and the type of the ***first*** element of that tuple `(mem', sp')` matches the type of one of its inputs, we can simply take the first element of x, and use it as the first input:

```
*Example1> :t stack1
stack1
  :: (Enum t1, KnownNat n, Num t1, Num t) =>
     (Vec n t, t1) -> Bool -> Bool -> t -> ((Vec n t, t1), t)
     \-----------/                          \-----------/
           |                                      |
           ----------------------------------------
                         Same type!
```

From the result, we can see that popping doesn't change the contents of the memory, but it decreases the stack pointer by one, and output the the value stored at the decreased stack pointer. In this implementation, the stack pointer always points to the memory location where a new value would be pushed. So when we pop, we want to output the value of the current stack pointer minus one, which is what we bind to `sp'`:

```
Idx |-----|
 3  |  0  |
    |-----|
 2  |  0  |
    |-----|
 1  |  0  | <- sp This is where the next value will be pushed.
    |-----|
 0  |  3  | <- sp' (= sp - 1) This is the value we just pushed.
    |-----|
```

Now for those playing along, there is a bug in my function when you push and pop at the same time. The correct behavior should be that the previous value should be popped and replaced with the new value, but this doesn't happen. Can you figure out why, and how can you fix it? I will explain the bug and fix it in the next post, but for now I leave it as homework ;)

Conclusion
==========

We have gone over the very basics of Clash and created a few simple functions. What I have noticed while writing this post is that I have shown a lot of types. Some of them were scary looking for non functional programmers, which might give you the impression that you have wrestle with them all the time. However, types are very useful and will help you when you will be debugging! And luckily, you don't have to deal with that a majority of the time. Also, our stack implementation looks very ugly, and doesn't show the power of Clash at all. In the next post, we will improve our stack function and make it much more readable and concise. We will also introduce registers so that we can create hardware that has state.

All the code we wrote in this blog post and will write in future posts can be found in the [Bitlog Clash tutorial Github repository](https://github.com/GuzTech/bitlog_clash_tutorial/tree/master). The code for this part is in the [part\_1](https://github.com/GuzTech/bitlog_clash_tutorial/tree/master/part_1) folder in the repo, and I have added some additional things like the ***stack2*** function that looks a little bit nicer (but still has the same bug!), and a modified ***topEntity*** that uses the stack. The type definition of ***topEntity*** looks very convoluted, but see if you understand why it is correct. At the end of the next post it will look something like this:

```
*Stack> :t topEntity 
topEntity :: Signal SInstr -> Signal Value
```

Play around with it, and generate some hardware. If you have questions and/or constructive criticism, let me know on Twitter [@BitlogIT](https://twitter.com/BitlogIT)
