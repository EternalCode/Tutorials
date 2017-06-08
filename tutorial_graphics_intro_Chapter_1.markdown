# Chapter 1: Using C on the ABG

This is a tutorial series introducing hacking the GBA using the C programming language. In this introductionary chapter, I assume the reader knows the C language to an acceptable threshold and has previous experience programming on the GBA either with C or ASM. This chapter is about learning roughly about the technical aspects of the GBA, but not to the point of a resource document like GBATEK or Cowbite would be. Our goal for the first chapter is establishing a base foundation inorder to be able to do some of the cooler things in later chapters.

This tutorial will be pulling many references and relating closely to FireRed's engine, as the author's preference.



## The Screen

I know this is a hardware component in a tutorial that's supposed to cover software, but like I say, we need to build a foundation. After reading this section, you should understand the following terms and concepts:
- VBlank
- HBlank
- Vsync
- VCount
- Scanline

If you know what these are, you can skip this section.


> 
> ![Image from cornac](https://www.coranac.com/tonc/img/gba_draw.png)
> 
> Taken from Tonc
>

The GBA's screen resolution is 240 pixels wide and 160 pixels in height. Like traditional computers, the GBA's display control starts off at the top-left most pixel on the screen and then configures the next pixel in the row. We call the entire row of pixels a **Scanline**.

You'll notice in the image, on the Scanline, after the first 240 pixels there is a 68 pixel area indicated by the green horizontal indicator arrows. This area is called the **HBlank**. Every Scanline has an HBlank period of 68 pixels. When talking about the scanline, we include this Hblank period in addition to the expected 240 pixel length of the screen. The display controller will experience a pause long enough for approximately 68 pixels to be drawn before moving on to the next Scanline. Consider this metaphor. You are drawing a horizontal line on a piece of paper, when you reach the end of the sheet, before you start drawing on the next line you must lift your hand and move it there. Consider this pause for moving your hand the same as an HBlank period. Each scanline experiences this pause.

Moving on past the 160th Scanline, we reach another pause area labeled the "VBlank". Exactly like the Hblank, the **VBlank** is a pause which the display controller does before moving on to the top-most Scanline. The VBlank extends for exactly 68 Scanlines, which makes it last significantly longer than the HBlank (recall that a scanline contains the Hblank in it). However, the Vblank, unlike the HBlank, has a very unique and useful timing. By being at the bottom of the screen, it occurs exactly after the contents of the entire screen has been drawn and the additional pause length makes it an ideal position for configuring graphics data. For example, imagine if we had a picture of a ball moving across the screen, like this:
>
> ![img](http://k03.kn3.net/3857C867C.gif)
>
 
Imagine if we changed the position of this ball during the middle of a scanline, not during the VBlank. The ball's shape would become distorted and we'd get something that looked like this:
>
> ![img](http://i.imgur.com/wabQGdI.png)
>

This is why every game (including every modern game) syncs their graphics during the VBlank. Every Vblank, any graphical components of a game is updated. This syncing process is commonly known as Vertical Syncing, or **Vsync** for short.

Finally, to explain what the **Vcount** is, it's the number of the Scanline. For example, the first Scanline happens when VCount == 0, and the second Scanline occurs when VCount == 1, etc.

This is generally all you'll ever need to know about the GBA's screen. It's important to fully understand these concepts inorder to understand how we want to render graphics on the screen and why GameFreak renders graphics the way that they do. I encourage you reread the section if you cannot recall certain things. Together we will eliminate screen tearing from all of our games!



## Introduction to the Memory map

Extra readings:
- http://problemkaputt.de/gbatek.htm#gbamemorymap
- https://www.cs.rit.edu/~tjh8300/CowBite/CowBiteSpec.htm#Memory

Before reading the following paragraphs, I want to preface that some of this information you will not be able to understand right now. For example, I don't expect you to know what DMA is. However, I do reference these things, and so if you get a little bit confused on what those are, I suggest continueing to read and after we've covered those topics comming back to revisit this chapter. For the love of everything holy, don't skip this section, as it's essential for setting up some general understanding of areas on the GBA!

**Read Only Memory (ROM)**

The ROM on the GBA is held locally on a game catridge. As the name suggests, we cannot write code to write data to the ROM. What we can do is write changes to the ROM binary outside of runtime, and we do this using tools like HexEditors, map editors, script editors, table editors, compiler output etc. When you see an address, you can tell that it's in the ROM if it's prefaced with the digits `08` or `09`, but you wouldn't see `0C` or `0A` preface the address. This is because GBA catridges are generally _capped_ out at 32MB. There are cases in other games where there are game mirrors at higher addresses, but that doesn't matter to us, since we're not going to utilize ROM mirrors. 

The bus speed for reading from the ROM is exactly 16bits, or 2 bytes. That means the GBA is only capable of reading two bytes worth of data per "trip". This makes thumb instructions ideal for programs that are executed through the ROM. ARM on the other hand requires two trips from the ROM for a single instruction, and that makes it slower during execution than thumb code. This is why we'll almost always compile our C code with the -thumb flag. Another important thing about the ROM is that in the worst case, it's the _slowest_ piece of memory on the GBA. However, when it comes to reading a string of memory in sequence, it is actually _faster_ than reading data from EWRAM.

**External Working RAM (EWRAM)**

EWRAM, or External Work RAM is in reference to RAM that is not on the CPU chip of the GBA. Like the ROM, EWRAM has a 16bit bus. Most of our image mirrors and data structures that don't need DMA or immediate syncing are kept here. In terms of Pokémon games, the Pokémon data structure for your team is held in EWRAM. Additionally, a lot of Gen III software functions run using EWRAM, for example malloc. EWRAM is always denoted with the "02" prefix, and is relatively adbundant at 256kb!

**Internal Work RAM (IWRAM)**

Internal Work RAM, or IWRAM, is the fastest RAM on the GBA, but also more expensive than the others. With a 32bit bus speed, IWRAM is often used by the game to store structures like the Superstate and other configuration structures to be synced with other areas of RAM during the Vblank. Normally when we want to run a lot of ARM code, we'd set a DMA memory transfer to copy the ARM Code to IWRAM and then execute it there, taking full advantage of the bus speed and instruction complexit.In Pokémon hacking, we'll likely never have the opportunity to try that out, but it's nonetheless worth noting that it's possible to run code that is stored in the RAM just as well as code in the ROM.

**Palette RAM, Object Attribute Memory RAM, and Video RAM**

Exactly as their names imply, Pal RAM stores palette color data for use by BGs and Objs. Object Attribute Memory RAM stores data structures related to Object rendering as well as the Object transformation matrix. Video RAM is segmented into BG dedicated Graphics and the lower section belongs to Objects graphics. Here is a memory map of these three RAM segments:

>
>    `05000000-050003FF`   BG/OBJ Palette RAM        (1 Kbyte)
>
>    `06000000-06017FFF`   VRAM - Video RAM          (96 KBytes)
>
>    `07000000-070003FF`   OAM - OBJ Attributes      (1 Kbyte)
>
> From GBATEK
>


You may notice that there are gaps inbetween these RAM locations, no they're not secret extra RAM, but just unusable RAM mirrors. The reason I just quoted GBATEK here instead of explaining each of them specifically is because I wanted to talk about how FireRed interacts with these areas of RAM. In a typical scenario, if you wrote some C code to write into these RAM areas directly, it'd have no affect whatsoever during the game. Save perhaps a single frame "flash" of graphical mess. In actuallity, it _did_ write! It just so happens, that Gamefreak's system also writes to those areas of RAM every frame (VBlank). Thus the game simply overwrites whatever your changes, and that is why you will never see your direct changes ever presented on the GBA screen. There are advantages to doing what GameFreak does, and I'll go into explaining all of them in detail in upcomming chapters. For now, just know that every Vblank, data in these areas are synced to IWRAM and EWRAM mirrors. 

The GBA also contains IO registers in the 04000000 area. The GBA doesn't contain an Operating system, so to control it's many support chips we make use of these IO Registers. By setting bits in an IO register you can enable, disable or adjust functionality of the corresponding support chip. While this seems primitive, it also allows someone who knows all of these IO registers to basically control every aspect of the GBA. Most of us will strive to understand as much as we can about these IO registers, as they are probably the most important thing to understand before ROM hacking any game on the GBA. I suggest skimming http://problemkaputt.de/gbatek.htm#gbaiomap Note I said, "skim" basically you'd just want to see what kinds of things these IO registers control. Thankfully for you, GameFreak has done a lot of work to write higher level functions to handle interacting with the IO registers, so we don't need to mess on a bit level most of the time. However, interacting with these registers during graphics rendering is mostly unavoidable so familiarizing yourself with what's possible is a good idea. Regardless, I will cover some of the important ones as we continue on beyond this first chapter. It's also very important to note that the GBA's BIOS does Zero-fill all of it's RAM when it's booted up. Additionally, the GBA will load the address 08000000 in the program counter and start executing the code contained there.


## What is a callback and what is a task?

First of all, I'll be talking about GameFreak's implementation of these common programming practices/attributes. You can easily find general information about these sorts of concepts on Wikipedia or similar. I want to discuss the callbacks executed by the main loop. Additionally, I will also talk about GameFreak's task system, because that's similar to the callback system, running at a much lower priority. I will discuss the Game loop/main loop in the following section, and hopefully by then everything here will click nicely.

**Callback1**:
Callback1 (C1 for short) is one of the callbacks executed through the main loop quite arbitrarily. This is the highest priority callback and is not Vsynced, meaning it can be running at any arbitrary time regardless of what value Vcount is. It's not stretching the truth too far to say that the entire game is executed through C1. Quite literally everything except syncing is generally done by the game in C1. During the overworld, C1 is responsible for running the scripting engine, handling player inputs, updating positionary elements, updating the world etc. At other times, such as battling, C1 is what runs the battle engine core including the battle script engine. Hopefully this provides some light to how important C1 is for the execution of Gen III games. The important takeaways is probably that C1 is high priority, not Vsynced and executed though the main loop - this means if the game is not "frozen" C1 will be running!

**Callback2**:
Callback2 (C2 for short) is exactly the same as C1, however, it's lower in priority. It will only run after C1 has finished executing. The game normally uses C2 to execute and sync game structures like the task system, Object callback execution, etc. I'll talk more about the Task system soon and even more about the Object system in a later chapter. For now, understand that C1 and C2 are both executed through the main game loop and neither of them are Vsynced. Due to C2's lower priority nature, user input and other things that need to be executed or processed quickly are done in C1, while things like executing arbitrary systems and syncronizing structures from the changes C1 has made are perfect for C2.

Both of the callbacks have their function pointers stored in the superstate. You could easily write directly to the superstate like this:
```c
super.callback1 = my_func;
super.callback2 = my_func2;
```

But typically, it's better practice (also slower) to utilize the game functions to do this:
```c
set_callback1(my_func);
set_callback2(my_func2);
```

**Tasks**:
Tasks are a useful function Gamefreak's developers had decided to add into generation III games. They can basically be considered functions which run through C2. C2 normally contains the function "tasks_exec", (which as the name suggests, executes tasks) and that would be when your task function is ran. As a GBA programmer, you can utilize GameFreak's tasks system by calling the function task_add which looks like this:
```c
u8 task_add(TaskFunc func_ptr, u8 priority) {
    // adds the task
}
```

The function you pass into task_func is a special type, it's header looks like this:
```c
void task_example(u8 task_id);
```
As you can see, every task function is passed it's own task_id and it's return value is ignored. A task is executed through a priority queue system, the position in the tasks array is the task_id. The reason tasks are passed their task_id is because they normally contain many variables local and private to the task itself. You cannot easily access these variables without the task's ID and the task_id is also needed to delete a task. An example of a task is the HP bar depletion animation. That's a task who's duty is essentially to calculate a Pokémon's HP deficit, and animate the HP bar to reduce itself down to a certain amount. Once that amount is reached, the task deletes itself. The next time this Pokémon takes a hit, the task is created anew with new variables to consider. The game also utilizes tasks for things like overworld HM animations, whiteout after battle loss and other very useful features. Being able to use tasks yourself is a good starting point to being able to do some more interesting things in your hack. The entire Dexnav system I created runs off the task system, it's quite flexible ;)

To run a task you do this:
1) Make a function that returns nothing and takes a task id
2) call task_add passing the function pointer of your function and a priority value
3) Enjoy your new function that runs (almost) every frame


## The Game Loop - Basic edition

Every game in existence works utilizing a concept people in the game development industry call the Game Loop. Generally, the game loops work the same in every game:

```c
    while(true) {
        Run game code
        Wait for Vblank*
            Sync data to screen
    }
```
That's the general flow of a main loop you'd expect in almost any game. Like discussed in "The Screen" section, it's extremely important that any screen elements are written during the VBlank period. Because the GBA's only form of output is through the screen (and sound), as expected the main loop almost entirely revolves around this key VBlank timer. Obviously, this is a very naive loop and in actuallity, modern game code is very much more complex, but essentially it can be boiled down to this. Inorder to maintain a high frame count, an efficient main loop is neccisary!

How about we check out a game we'd actually hack, specifically for FireRed the game loop looks like this:

```c
    init_data_structures();
    while(true) {
        check_keypad_reset();
        if (*is_link_connected()) {
            execute_callbacks();
        } else {
            execute_callbacks();
            if (*connection_canceled()) {
                obj_sync_cancel();
                execute_callbacks();
            }
        }
        
    gametime_increment();
    wait_for_interrupt_exec_completion();
    }
```

Before explanations, the ones with '*' preceeding them means that I'm not 100% sure about their functionality. Unfortunately I don't have a console with a link cable to test if these names or even ideas are actually accurate, but at a glimpse it seems so. I'm fairly certain they deal with link connections. 

Quite interestingly enough, the "manual keypad reset" functionality is included in this main loop. That is Holding A+B+START+SELECT buttons at the same time would reset your game (try it!). That's what the "check_keypad_reset()" part is all about. This check is run every round of the main loop, it may thus seem inefficient, and it is. However, this is cleverly implemented as if the button "A" isn't being pressed the remainder of the checks aren't run either. A further optimization would have been to work this into the game's keypad interrupt functions, but I suppose the developers determined this wasn't too big of a deal. Taking out the link related things our loop looks more like this:

```c
    init_data_structures();
    while(true) {
        check_keypad_reset();
        execute_callbacks();
        gametime_increment();
        wait_for_interrupt_exec_completion();
    }
```

As you may have guessed, execute_callbacks is responsible for executing the C1 and C2 callbacks and the rest are self explanitory. Except for maybe wait_for_interrupt_exec_completion.

You may be asking yourself now, "Where is the part where the graphics get synced to the screen?". To understand how GameFreak handled this, and how you should too, we need to first learn about interrupts. Before doing that though, I'll mention that wait_for_interrupt_exec_completion() basically ensures that all enabled interrupts are executed before proceeding to the next iteration of the loop.

Interrupts are a shared concept in both modern computers and the GBA. Essentially an interrupt request, or IRQ is triggered by an "event" occuring. This event can be anything from a DPad button press, or when the scan line reaches the Vblank area. When the event-triggered interrupt occurs, the CPU switches to IRQ mode and the BIOS saves the contents of both the stack and the registers, then calls something called an "Interrupt Service Routine" by setting the Program Counter to the location of that IRQ Service routine. When switching to IRQ mode, the CPU is swapped into ARM mode, so IRQ Service routines are expected to be in ARM. However, Gamefreak's IRQ handler overhead has the functionality to activate an interrupt, and jump the CPU back to thumb mode before executing the IRQ service routine. So we can still use the -thumb flag to compile our C version of an IRQ service routine (if we're using Gamefreak's code to execute interrupts).

To set an interrupt, you need to modify the contents of a register called the "Interrupt Master Enable Register". But as I alluded to before, Gamefreak has a system to enable interrupts in thumb mode and much more easily for certain types of interrupts. Specifically, Gamefreak's code handles Vcount, Vblank, Keyboard, HBlank and probably more, but those are all the ones which I've actually used. If these are one of the interrupts you're looking to set, what you first do is set the service routine inside the superstate and then call the function "interrupt_enable(type)" where type is an enumumerator value with the type of interrupt you're trying to enable.

So now revisiting the question, "where is the part where the graphics get synced to the screen?", that's actually done in a Vblank interrupt! Since interrupts have the ability to make the CPU drop what it's doing and execute some arbitrary code, you always guarentee that even if your mainloop and game is doing some extremely complex things which run into the VBlank, the VBlank interrupt will still execute without fail. For instance, imagine if we made a big error and our C1 callback ended up being something like this:
```c
u8 x = 3;
while (x) {
    x += 3;
}
```

Clearly, 'x' will never be 0, so this loop will keep going forever. However, if we tried putting something like this in our C1, we'd observe that the game "soft locks" in which the music and the flower animations as well as the screen contents are still working, but we cannot interact with the game. The only reason that these are working is because of the interrupts. The IRQ mode drops the CPU out of this infinite loop to play the music, draw the animation, draw to the screen, and then....returns it back to this infinite loop. I'm sure many of you have experienced something similar to this before. Now you know why it happens, and hopefully you know how interrupts work.

To take away from this, know that the game executes the main callbacks C1 and C2 inside the main loop. The graphical syncing of the attributes in the game is done during the VBlank making use of an interrupt.


# Introductionary challenge

<mark>-TBD-</mark>
