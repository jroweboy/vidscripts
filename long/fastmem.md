
Fastmem - Segfaulting for faster memory access
===

## Sections i want to talk about

- How does a JIT work? (Maybe too much for this vid?)
- If theres fastmem, then whats slow mem?
- How do we handle callbackst then?
- Can we do better? (Semifastmem)
- Limitations of fastmem
  - Requires host page table size to match guest size if emulating guest page access
  - Need to be able to decode any instructions that you encode in order to finish running the block


Script
===

## $MOTIVANTIO$?

Sometimes fast just isn't fast enough.
When optimizing CPU emulation, the general goal is to run as many guest CPU instructions as you can, with the fewest and fastest host CPU instructions possible.
(bucket graphic)
Small optimizations are just drops in the bucket for overall emulator speed.
(full bucket graphic)
Each minor change sounds so insignificant, but they all add up quickly to fill the bucket and make it *fast*.
(big drop falling into the bucket)
Fastmem can be a rather large drop in the bucket if done correctly, bringing a noticeable performance boost to emulators.


## Speed up with Just in Time compilation (Quick Overview)

(Without going into details about JITs in general)

(graphic of consoles -> magic -> pc/phone)
For most emulators, the overarching goal is simple: take a program that runs on that device you are emulating, and write a program to run it a different device.
While every device is different, from a very abstract point of view, each emulator will need to tackle similar problems.
(picture of binary bits -> magic -> picture of binary bits)
First and foremost, the emulator must be able to read in the compiled game binary, and accomplish the same actions that the game binary says to do.
A binary is very often comprised of several sections, which the emulator will need to parse and handle.
The important section today is the code section, which contains compiled CPU instructions for the guest machine.
(Interpreter diagram of read eval loop)
A simple way to handle CPU emulation is as follows:
Decode the guest CPU instruction, perform the action on the host, jump to the next instruction, and repeat.
This basic execution model for running guest CPU instructions on the host machine is called an interpreter.
Interpreters are both straightfoward and simple, but also too slow for emulators that demand high performance.
There is a faster way to execute guest code; let me introduce you to Just In Time recompilation, often called JIT for short.

(segue into JIT. most every modern emu uses it to some extent. including urmom xDDDDD)

Today I will not cover the details of recompiling code in depth, but in order to understand fastmem, we need at least a cursory understanding of why JITing is faster than interpreters.
When looking at the common interpreter diagram, there's a big ineffeciency staring right at our faces.
Every time we execute an instruction, we have to decode the instruction, and then emulate what it does.
But what if we just read a bunch of instructions at once, and then turned it directly into code that does the same thing?
Well, for starters, it's going to take some extra time to read and decode all of those instruction, and then some more time to generate native code that does the same thing on your computer.
So why is it so much faster?
(sherlock holmes)
Elementary my dear watson, applications tend to run the same code multiple times!
The savings really comes into play the more times the recompiled code runs, cutting out each of those reads will save a ton of time!
There's still some major problems we glossed over when talking about recompiling code, though.
(simple memory access code)
Here is a very simple example of a made up guest application that loads some memory from RAM into a register.
This instruction tells the memory address that it wants to load the data from, and what register to put it into.
Now, in our JIT recompiler, we want to convert this into instructions that will run on your host machine.
For desktop PCs, we would use the `mov` instruction which looks like
(mov instruction)(car tire screech)
What should we use for the memory address that we should load?

## The pretty picture of the high level overview without ugly details

In a simplified world, we could emulate memory access with some very straightforward code.
(Graphic of guest ram on host by making a simple array the size of the guest ram)
When the guest code needs to read or write to the device's RAM, we can simply put the values in this pretty little RAM container we created on the host machine with the same addresses that the guest uses.
If the guest says load 4 bytes from address 0x1000, we also load 4 bytes from address 0x1000.
But alas! Our pretty picture will soon get muddied as soon as we start adding in some crucial details.

## Warning: Shared memory architectures ahead

For many very good technical reasons, several consoles use a shared memory architecture, where the CPU and GPU share access to a single phsyical RAM device.
On the flip side, the devices you run the emulator on, usually has separate RAM for both the CPU and GPU, and constantly transferring data between the two is slow!
(Side note: phones and integrated GPUs are usually shared memory devices, but when programming for them, you end up programming for them as separate memory devices anyway)
Well we *could* solve the issue by emulating everything on the host CPU and emulate the RAM in a single array like we did in our pretty little picture.
Sadly, this doesn't cut it for a modern emulator where they need to go fast, and in order to go fast, they need to use the host graphics card as much as possible.
Since the RAM is separate between CPU and GPU, we have to manage two emulated RAM chunks, and we need to devise a way to copy as little as possible between the two, while keeping them in sync.
Enter memory access callbacks.
Whenever the CPU or the GPU write to their copy of RAM, you simply copy the data back to the other RAM.
In order to do this, we'll need to update our pretty picture of memory access to add a special handler to the memory read and write methods, so that we can sync it with the GPU memory.
(Change mov instruction in host code to push stack and call function read memory)
Things are getting a little dicey now.
In order to guarantee that our emulated RAM on the CPU and the GPU are kept in sync, we gotta do a whole lot of instructions for each and every guest memory access.

## Guest Page tables and Host page tables for 4x the fun

Now that we've got two separate copies of the RAM, time add in some more complexity, known as virtual memory.
A modern console wouldn't be so dang modern if it didn't also include an operating system and kernel, complete with multi process support.
That's right, many modern console games no longer run on bare metal, but instead run through the console's operating system which provides many nice features that your PC also provides.
(picture of virt mem?)
Virtual memory is one such hallmark feature which, without going on a tangent and diving into that rabbit hole, is how an operating system is able to run multiple processes while giving each process access to the entire RAM space and keeping each processes memory isolated.
The birds eye view of virtual memory looks like the following: whenever a process writes to memory, its really writing to something called a page.
This page could be backed by real location in memory, or maybe its backed by the hard drive.
Either way, the program thats running doesn't know or care, it just writes to a memory address and lets someone else handle that dirty work.
When the process attempts to write to a spot in memory, its up to the kernel to find page that it was meaning to write to, and swap it into place if it was moved by a different running process.
The kernel can figure out which process owns which page by doing some book keeping on the side.
This is called the page table, and it keeps track of which pages belong to which process, whether or not the process has permissions to read/write/execute that page, and any other attributes that the kernel needs to keep track of.
At the end of all this, the process will write to the page that they intended to, and everyone is happy.
Both process A and B are able to write to address 0x1000 without any problems.
Uh oh, wait we have to emulate this, don't we!
(optimistic emu dev stock image)
"Can't we just use the host computers page table" the optimistic upcoming emulator developer might respond.
(NO! graphic)
NO!
Well, sometimes you can. It depends on the requirements of the device that you are emulating.
If the device has its own virtual memory, and the size of the guest page table doesn't match the host, then chances are you won't be getting anywhere with that.
The rabbit hole goes deeper and deeper, but we'll stop here for now!
Let's look again at our slowly-getting-uglier generated code for memory access.
(Display graphic again, this time with virt mem included)
Now we need to take into account accessing the page (highlight each instruction as we mention it), finding out if the page even exists, checking if its accessible, 
For just one small memory instruction, we end up generating tons of instructions, we gotta do better!

## If theres fastmem, there must be slowmem

Everything I just described explains some of the challenges with emulating memory accesses, complicating our pretty little perfect-world vision of memory access.
This code represents the naive version of memory access in recompilers for emulators.
Each piece is necessary for accurate emulation, but is also very costly.
While no one has really named this pattern anything special, I like to think of it as slowmem.
It is always there for us, it has our back when our shortcuts fail, its slow, but steady.
Its sure to work.
But speed. 
(zoom in on the word speed)
Speed is what we crave.
We need to cut corners and shave off as mush as we can from the generated code, while detecting cases where we can't.
When our fast path fails, we can always fallback to good ol' slowmem.

## Maybe a little faster. Semifastmem

The first trick we can do is eliminate the function call.
Instead of storing the current register state, calling a function, and running the memory handling code somewhere else, we can inline the function inside our generated code in the best conditions.
Recall that the reason we want to jump to a function call is to handle cases where we need to sync memory between our two RAMs copies on the host CPU and GPU.
But the GPU doesn't need a full copy of the RAM, it only needs to know if the CPU is writing to memory that it actually cares about.
The host GPU only needs to know about things like textures, vertex buffers, and framebuffers, not everything else that the CPU would use RAM for.
So let's be a little bit smarter about what memory we copy over to the GPU.
We can accomplish this by marking the pages as something special, and then when a special page is accessed, the memory access function will tell the GPU to update their copy of the RAM as well.
Now that we don't copy data between the host GPU and CPU every time, in the generated code, we can check to see if the page is marked as special.
If it is, then we need to call the function for slowmem access.
But if its not, we can jump into semifastmem.
Instead of calling the function, we can keep a register around that stores the address of the page table, indirectly access the underlying memory, and then read/write to it.
The end result is we inlined reading the page table for almost all memory accesses, but still fallback to slowmem when needed.
Speed gains for everyone!
(Zoom in on speed with dramatic music)
GAH! Fine! We need more speed!

## 2 Fast 2 Mem for emulators that don't emulate guest page tables

Alright, alright. 
Back to the original idea we had, what if we could somehow magically line up the addresses such that when we generate a memory access, we can emulate it with a single host instruction.
Keep in mind, we still have a hard requirement to keep our CPU and GPU memory in sync AND we still need to emulate page table access.
Earlier I mentioned that we couldn't use the host page table to emulate the guest page table, but there are some caveats that I need to explain.
If the guest doesn't use a page table because the games run on bare metal, then that simplifies memory access a lot, as we don't need to jump through hoops to emulate page tables.
Several emulators such as PPSSPP, Dolphin, and Redream to name a few, are able to take advantage of the host page table to achieve the dream of fastmem.
Since games on these consoles accessed memory directly, they don't have to fiddle around with handling page tables in their emulation code and can write directly into some emulated RAM (unless the game was particularly evil and turned on the MMU but lets ignore that for now).
(picture with page table stuff scratched out)
Now that we don't have to worry about guest page tables, we can focus in on the core problem, converting a guest memory address to a host memory address.
In modern host operating systems, like your PC, the operating system will randomize all memory addresses when launching your emulator, which hardens your program some against malicious attackers who want to break your emulator.
Address Space Layout Randomization (ASLR) means that we can't just hard code the address that our emulated RAM will be at, since this address changes everytime.
(show offset + addr)
But we *can* figure out what address our RAM is located at runtime, and store it in a register so that we can access the memory at offset + address.
Problem solved?
Not quite, we haven't talked about *addressable memory space* yet.
Redream, a dreamcast emulator, is emulating a console with only 16MB of CPU RAM, which means you would expect that games only have 24 bits of memory addresses to work with because 2^24 is 16MB.
Rephrased a bit more simply, a console with only 16MB of ram should in theory only have 16MB of addresses that you can use.
From the first address 0x0 all the way to 0x (whatever that is), the address that corresponds to 16MB worth of RAM.
But memory access doesn't work that way in reality, it turns out that the actual ~~retail price~~ actual addressable memory space is dependent on different factors, and in the case of the dreamcast, there's 29 bits of addressable space, or 512MB of addresses that can be used.
So what does all these extra address map to if there isn't physical RAM for them?
(few seconds of jepordy theme)
// TODO verify the following claims about memory mirrors.
Addresses outside of the normal RAM range map back to the physical RAM!
These memory mirrors allow compilers to encode extra data in the upper bits of the address, which can be used as optimization hints for the memory controller.
So when Redream attempts to emulate dreamcast memory, it suddenly needs 32 copies of CPU RAM in order to emulate all of the different memory mirrors and then add code to sync them up, right?!
(show adding an extra instruction to mask the upper bits in generated code)
Well, no. The simple solution is just to add an extra instruction when generating guest memory access to mask off the upper bits of the address and voila!
(silence followed by a quiet "Boo! We are trying to get rid of extra instructions!")
Oh fine, we can do better.
Time to break out the host operating system's page table.
Using the host operating system's page table functionality, we can actually reserve large swathes of RAM, without actually commiting to use it until its needed.
This means we can ask the operating system to set up some pages in *its* page table for us to work with.
From there, we can reserve as much RAM as we need, and find inside of that a contiguous chunk of at least 512MB of RAM, get the offest to this RAM, and then use that as the offset when generating the code.
The advantage of using the host page table is now we don't actually have to commit to using 512MB of ram when the guest really only has 16MB of RAM.
For each of those other mirrored memory maps, we can tell the host operating system to commit to using the first 16MB of RAM, and then use that 16MB as the backing for all of the other mirrors.
Now its the host operating system's job to get the correct backing memory in place when accessing an address that falls into one of those mirrors, and we can get rid of that extra masking operation.
Hurray! We've finally reached our dream of emulating guest memory access with a single instruction on the host!
...

## Segfaulting for great justice

But... back to reality.
We still don't handle keeping the CPU and GPU memory in sync at this point.
What will we do when the game code writes to memory thats supposed to be synced with our GPU RAM?
Host page table to the rescue again!
Recall earlier, in our emulated page table, we could mark the page as "special" which would indicate that we needed to fallback to slowmem.
Well, marking a page as special isn't really a property of the real page tables you find on operating systems today, but we can accomplish the same thing by using page protection.
Protecting a memory page means that whenever any instruction tries to read or write or execute memory thats on this page, the kernel will segfault, throwing an error, and getting all huffy with us.
But segfaulting and causing the kernel to yell at us is just what we wanted.
Instead of checking all of the conditions to see if we should fallback to slowmem, we instead protect the memory pages that need slowmem, and then always access the memory directly in the generated code.
The dream is here, we finally emulate guest memory access with a single host instruction!... well, as long as we don't have to fall back to slowmem, and we kinda handwaved over what the consequences of protecting specific memory pages are.
(picture of computer with a knife knocking on a door)
So as we just learned, protecting the page means the kernel will send a segfault signal, and segfaulting means the host machines kernel is on its way to our door with the intent to kill our program!
This means when we run our beautiful single instruction generated code, there's a chance that its going to access protected memory, and when it does, the kernel kills the entire application.
(yells in the background: Its worth it for speeeeed)
But we can handle this!
Before running the generated code, we can tell the host operating system "Hey, ur gunna get mad at me for this, but i plan on causing some access violations later. I want you to call this function instead of killing me, please. Can you do that? Thx bby"
This is known as setting up signal handlers, a mechanism for letting the emulator handle the access violation on the protected page before the kernel slaugthers our poor emulator.
Signal handlers tend to be limited in what you can do inside of them though, its not safe to do too much, so we want to do as little as possible here and move on before things start getting too sketchy.
The first thing we want to do is setup a flag on the generated code block that caused the segfault.
We know that as long as that memory page is protected, running this generated code block is likely to cause segfaults, so we want to avoid this next time.
The flag we setup will tell the JIT that next time this generated code block should be run, don't actually run it.
Instead, recompile the code block but this time don't generate fastmem for memory instructions, just use slowmem.
Now we get fastmem for almost all of the cases, and also fallback to slowmem whenever we need to, thanks to those protected memory pages and segfaults.

## Decoding host instructions and bringing it downtown

Hmm, anyone else feel a bit of looming regret, as if we glossed over another important detail?
Oh, right.
If we segfault in the middle of a generated code block, that means we stopped running that code block, and that also means we didn't *finish* running the code either.
From the beginning, I mentioned that for all of these optimizations, its important that we accomplish the same end result while also doing less work.
Not running some of the code straight up means we didn't get the same result, so back to the drawing board!
Alright, lets look at the details again.
When the signal handler is called, we are told the address of the offending instruction.
We already have some sort of mechanism in place to tell ourselves what block of code we were running when we crashed, we needed this in order to flag that block for recompilation.
And with the address, we also know where in the block of code the error occurred.
But we can't simply just set everything back up and run that instruction again, as we know it's guaranteed to segfault.
(picture of looming kernel with knife slides in)
And we don't want that again.
Instead, its time to break out our toolkit again, and this time add an instruction decoder for our host platform.
If we decode the instruction at this address, we can find out what address the generated code was attempting to access.
With that knowledge, we can call directly into the method that we wrote for slowmem, which will safely emulate the failing instruction without causing another segfault.
From there, we can set another flag and store the cpu context as well, this time on the code dispatcher, telling the dispatcher that before it continues running anything else, it will first need to finish running this code block, and heres the values for the registers since you'll need that in order to run the same as well.
Now when we segfault, we set everything up in the signal handler so that the rest of the code block will run to completion.
And our goal of using a single instruction for emulating a memory access is now complete, but only when the guest isn't using virtual memory.

## Back to emulators with guest page tables

Fastmem, as we just worked out, is totally possible to do when the guest application doesn't have a distinction between virtual and physical addresses.
But now we need to look at what happens when the guest application IS using virtual memory.
For starters, the guest memory access instructions that we will be recompiling are using virtual addresses as their address, not physical addresses.
This means that for a console like the 3DS where it supports multiple processes running at the same time, you will need a slab of virtual memory on the host for each of the processes.
In general, most applications aren't running multiple processes though, and when they do, they tend on only have 2 total, so its not like we need too many host pages for this.
But things are about to get tricky again, and completely fall apart in some cases.
Now that we have to deal with guest virtual memory, we no longer have fixed memory mirrors that we can set up anymore.
When the guest directly accesses the RAM, we know what the memory mirrors are going to be, since thats general knowledge about the console that you are emulating

## summary cause no one watched for this long

In the quest for 

