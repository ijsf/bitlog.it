# Getting Started With The GA144 And ArrayForth

_2014/12/24 Oguz Meteer // guztech_

---

The GA144 chip by GreenArrays is unique in several ways:

* It has 144, 18-bit cores.
* It is asynchronous, so there is no clock.
* It is programmed in [Forth](https://en.wikipedia.org/wiki/Forth_%28programming_language%29), and GreenArrays provides colorForth, a version of arrayForth specifically for the GA144.

Even though there is adequate documentation, getting started with this chip can still be quite daunting. There are a couple of tutorials on the internet, but they are written for older versions of arrayForth and do not work with the latest version. Essential information is spread out throughout the documentation, so think again if you want to start with the GA144 quickly. Especially when you have to program the chip with an IDE that looks like this:

[![codeediting](http://bitlog.it/wp-content/uploads/2014/12/codeediting-300x225.jpg)](http://bitlog.it/wp-content/uploads/2014/12/codeediting.jpg)

Do not worry however, as I will explain in this post how to write a Hello World example and get it running on the simulator, with emphasis on the last part as I will not go into details of the chip works. I do expect you to have some experience with Forth and its stack based nature. Also, please read the [arrayForth user manual](http://www.greenarraychips.com/home/documents/greg/DB004-131030-aFUSER.pdf) as it is very useful for getting a feeling for arrayForth. More links to interesting information about the GA144 can be found at the end of this post (highly recommended!).

First launch
============

When you first launch arrayForth, you are greeted with the following screen:

[![startscreen](http://bitlog.it/wp-content/uploads/2014/12/startscreen-300x225.jpg)](http://bitlog.it/wp-content/uploads/2014/12/startscreen.jpg)

Something you should know about arrayForth is that it uses a specific keyboard layout for entering different kinds and colors of syntax (hence colorForth). On page 13, chapter 3 of the arrayForth user manual, you can find the layout used while you are in the editor. Before we start writing our Hello World example, we need to know how memory is organized, where we put our code, which nodes run which block of code, etc.

### Memory Organization

Memory in arrayForth is organized in 1400 blocks, some which contain system software such as the compiler and simulator, while others contain example code or are simply empty. A summary of how memory blocks are organized can be found on page 18, chapter 4 of the arrayForth user manual, while a detailed overview can be found in the `EVB001-02b.html` file located in the installation folder. We can see that blocks 840 to 1078 can be used to store user `code`. Note that it's not 1079 because only even numbered blocks can contain code, while uneven numbered blocks are used to store user comments.

### Node Layout

The GA144 chip consists of 144 F18 computers in a grid of 18x8:

[![GA144](http://bitlog.it/wp-content/uploads/2014/12/GA144-300x232.jpg)](http://bitlog.it/wp-content/uploads/2014/12/GA144.jpg)

Each computer, called a node, has an identifier that starts with 000 for the bottom left node and ends with 717 for the top right node. Each node is connected to its neighbor nodes. The outermost nodes are connected to the outside world, either directly, or through peripherals such as UART, SPI, ADC, etc. Therefore, you need to carefully plan the layout of your application.

Hello World
===========

Now that we know a bit about the GA144, let's write our first piece of code that performs the following calculation: `3(x+1)`. We will place our code at block 860, which lies in the area that is available for user code.

  860<space>edit<space>

This opens up the editor and greets us with an empty block:
  
[![block860](http://bitlog.it/wp-content/uploads/2014/12/block860-300x225.png)](http://bitlog.it/wp-content/uploads/2014/12/block860.png)

Type the following:

  <u>0<space>org<space><esc><x>br<space><esc>
  
Your screen should now look like this (bottom part is left out):

[![firstline](http://bitlog.it/wp-content/uploads/2014/12/firstline1.jpg)](http://bitlog.it/wp-content/uploads/2014/12/firstline1.jpg)

If you know assembly, then you will understand the yellow 0 org part. Whatever comes after that will be placed at memory location 0 and up. The blue br is an editor command that is not compiled into executable code, but rather tells the editor to insert two new lines. If it was cr, then only one line new line would have been added. Next up, we will define three words (functions if you will): setup, calc, and mul.

* setup will give the variable x in our formula 3(x+1) a value and store it in register a.
* calc will retrieve the value in register a, add one to it, and multiply it by three.
* mul is a function that performs the 18-bit multiplication as the F18 computer does not have a single multiply instruction. For more information, check the [excellent post about multiplication by Ashley Nathan Feniello](https://github.com/AshleyF/Color/blob/master/Docs/multiply_step.md).

Let's first code the mul word:

  <i>mul<space>a!<space>0<space>17<space><esc><u>for<space><esc><o>+\*
  <space><esc><u>unext<space>drop<space>drop<space><esc><o>a<space>;<space><esc>
  
[![mul](http://bitlog.it/wp-content/uploads/2014/12/mul.jpg)](http://bitlog.it/wp-content/uploads/2014/12/mul.jpg)

Next up is the calc word:

  <i>calc<space>a<space>1<space>+<space>3<space>mul<space>;<space><esc>

[![calc](http://bitlog.it/wp-content/uploads/2014/12/calc.jpg)](http://bitlog.it/wp-content/uploads/2014/12/calc.jpg)

The final word is setup:

  <i>setup<space>4<space>a!<space>calc<space>;<space><esc><x>br<space><esc>

[![setup](http://bitlog.it/wp-content/uploads/2014/12/setup.jpg)](http://bitlog.it/wp-content/uploads/2014/12/setup.jpg)

In this case, I have chosen x = 4, which will result in 3(4+1) = 15 = 0x0F in hex. We are almost done! Last part of our Hello World example is to make sure that the entry point of our program (setup) is called when a node is loaded with our code. We can set it like this:

  <u><F1>0a9<space><F1>org<space><esc><o>setup<space>;<space><esc>

Thıs gives us the final line of our code, and places the entry point to our code at address 0xA9 where setup will be called. 

[![full](http://bitlog.it/wp-content/uploads/2014/12/full.jpg)](http://bitlog.it/wp-content/uploads/2014/12/full.jpg)

Now press space to go out of editing mode (you will see that the red 860 in the bottom right will turn grey). Our code needs be compiled, which can be done by typing compile and pressing space. You can also save the state of all 1400 blocks with the "save" word (type save and space).

Setting Up Nodes
================

So we have written code, but how do we get it into one of the nodes? This is the part that is not up to date when you look at other, older tutorials, and it is also not very well documented. Block 200 contains instructions to load code blocks onto nodes, and this is where we can load our code. First change all instructions that are not white or blue to white (which are comments). You can do that by placing the cursor behind the instruction/number and pressing a until it turns white.

We will be placing the loading instructions after the first line:

  <u>400<space>node<space>860<space>load<space><esc><x>br<space><esc>

This line selects node 404, and loads block 860 in it. How it looks:

[![block 200](http://bitlog.it/wp-content/uploads/2014/12/block-200.jpg)](http://bitlog.it/wp-content/uploads/2014/12/block-200.jpg)

The reason why we place this in block 200 is because the softsim simulator, which resides within arrayForth (blocks 148-150), first sets up the simulator (block 148) and loads application code stored in block 200 (block 150).

You can repeat this step if you want to place the same block in another node (running the same code in parallel), or if you want to load another code block in another node. If we were to run the simulator now, we would see that node 404 is suspended and does not execute any code. To make sure the node starts with the program counter pointing at the correct entry point, we have to modify block 216. This block contains the configuration and testsbeds for the simulator.

Open up block 216:

  216<space>edit<space>

And add the following code before the line that starts with the comment "rom write test 200 +node 13 /p,":

  <u><F1>0a9<space><F1>404<space>enter<space><esc><x>br<space><esc>

Block 216 now looks like this:

[![block 216](http://bitlog.it/wp-content/uploads/2014/12/block-216-300x225.jpg)](http://bitlog.it/wp-content/uploads/2014/12/block-216.jpg)

Now the correct entry point for node 404 has been set, and we are ready to run our code in the simulator!

Running The Simulator
=====================

Type so (or softsim) en press space to start the simulator.

[![simulator](http://bitlog.it/wp-content/uploads/2014/12/simulator-300x225.jpg)](http://bitlog.it/wp-content/uploads/2014/12/simulator.jpg)

Page 33, chapter 7 of the arrayForth user manual explains how the simulator works, but I'll give a short explanation.

On the top right, there is an 18 by 8 grid representing all nodes of the GA144. A green symbol represents a node that is running and a grey one means that the node is suspended. If you would not have modified block 216 as explained above, node 404 (5th from the left and 4th from above) would be grey as it would not run our Hello World example.

There is also a red and a yellow X, which represent the "focus" node and the "other" node respectively. On the bottom right, the memory contents of the focus node is shown. If you press the "/" key, then the top right will be replaced with the memory contents of the "other" node. This is useful if you have two nodes the communicate with each other.

The left portion of the screen displays the internal state of nodes, such as register contents. This grid of 8 by 4 nodes is marked by a blue rectangle in the grid overview on the top right, and can be moved to select the nodes that we are interested in. An example of the internal state of a node:

[![internalstate](http://bitlog.it/wp-content/uploads/2014/12/internalstate.jpg)](http://bitlog.it/wp-content/uploads/2014/12/internalstate.jpg)

From top to bottom:

1.  Node number (grey), com port address (white)
2.  Slot number (white), opcode name (green)
3.  Instruction register (white)
4.  Memory timer (green), program counter (white)
5.  A, A register (white)
6.  B, B register (white)
7.  IO, IO register (cyan)
8.  R, top of the return stack (red)
9.  T, top of data stack (green)
10.  S, second on the data stack (green)
11.  @ represents a communication port that is being used

Running our code
----------------

If we place the red X on node 404, and adjust the blue rectangle to include node 404, we can see what happens with the node that we want to run our example on:

[![simexample](http://bitlog.it/wp-content/uploads/2014/12/simexample-300x225.jpg)](http://bitlog.it/wp-content/uploads/2014/12/simexample.jpg)

If we step through the code by pressing n repeatedly, we can see that node 404 executes our code:

[![example](http://bitlog.it/wp-content/uploads/2014/12/example.jpg)](http://bitlog.it/wp-content/uploads/2014/12/example.jpg)

Below I will shortly explain what happens chronologically. First, the memory address is stated, and next is a numbered list of instructions that correspond to instruction slots (see the F18 Technology Reference document for more information).

#### 0xA9

1.  jump to 0xA

#### 0xA

1.  put the value stored in 0xB (which is 4) on the stack
2.  store the top of the stack in register A
3.  NOP
4.  NOP

#### 0xC

1.  jump to 0x6

#### 0x6

1.  put the value stored in register A on the stack
2.  put the value stored in 0x7 (1) on the stack
3.  perform addition
4.  put the value stored in 0x8 (3) on the stack

#### 0x9

1.  jump to 0x0

#### 0x0

1.  store the top of the stack in register A
2.  put the value stored in 0x1 (0) on the stack
3.  put the value stored in 0x2 (0x11 = 17) on the stack
4.  NOP

#### 0x3

1.  store the top of the stack on the top of the return stack
2.  NOP
3.  NOP
4.  NOP

#### 0x4

1.  perform a step multiply
2.  skip to the next instruction if the return stack is 0, else go back to the previous slot
3.  drop the value stored on top of the data stack
4.  NOP

#### 0x5

1.  drop the value stored on the top of the data stack
2.  put the value stored in register A on the stack
3.  return

When the last instruction has exectued, we can see that the value 0xF or 15 in decimal on the top of the stack, confirming that our example works correctly! I hope that this post will help filling in the blanks after reading the GreenArrays documentation.

References
==========

GreenArrays documentation:
[http://www.greenarraychips.com/home/documents/index.html](http://www.greenarraychips.com/home/documents/index.html)

Ashley Nathan Feniello has written several excellent blog posts on the GA144:
[https://github.com/ashleyf/color](https://github.com/ashleyf/color)

Website of Chuck Moore (inventor of Forth and co-founder/CTO of GreenArrays):
[http://www.colorforth.com](http://www.colorforth.com)

F18 instruction set: 
[http://www.colorforth.com/inst.htm](http://www.colorforth.com/inst.htm)

Performing arithmetic on the F18:
[http://www.colorforth.com/arith.htm](http://www.colorforth.com/arith.htm)
