
Shader Cache - General Knowledge
===

## Introduction

Two words, shader and cache.
In order to understand what a shader cache is, a little background on what shaders and caches are is in order.
First term, cache; caching is a technique in programming where a program will store data in such a way that the program can avoid the cost of generating that data again, by loading it from the place where you "cached" it.
Second term, shaders; just like how the CPU inside your computer runs programs, the GPU inside your computer runs specialized programs known as shaders.
One of the hallmarks of modern consoles is also have a GPU that works in a similar fashion to the GPU inside your PC or phone.
Developers on these consoles can take advantage of GPUs in order to have precise control over how their 3D models and scenes are rendered, and use the GPU to accelerate the math behind it all.
In order to really get a feel for what a shader is, let's take a second to look at the evolution of GPUs.

## Before programmable pipelines there was 

Historically, 3D rendering on consoles used hardware that performed rendering with a fixed function pipeline, a dramatic step up from the custom pixel processing chips present in older generations.
As a quick tangent, a pipeline is a combination stages that run step by step, going from raw vertex coordinates and data, to pixels that are displayed on the screen.
The game developer will configure the pipelines in different ways to tweak the output of each pipeline, which is how games create everything in the 3D scene from the 3D camera, to animations, to bloom effects, lighting, cel shading, and so on.
A fixed function pipeline means the steps that each pipeline takes will be the same for every game that runs them, so then how could games ever look different?
Well, the answer is surprisingly simple, while the steps of the pipeline are fixed, they are still highly configurable.
Think of it like I told you that you need to draw a line on a graph and you only have two variables X and Y.
The simple answer is to take X and Y and make a straight line out of it. (Y = MX + B)
But you really have a lot more control over it.
You can make a curve by raising X to a power of 2.
You can make an oscillating line by using trignometric functions.
You can even draw the batman logo if you really go crazy with it.
With a little finesse, you can do just about anything with it, but are ultimately limited to drawing lines that you can describe with two variables, which makes it more complicated to do any sort of exotic lines.

## Enter programmable pipelines

Modern GPUs work differently.
Instead of having a predefined operation for the pipeline stages, modern GPU's are programmable, meaning for several of the pipeline steps, you write a program in a language and compile and run it.
This turns out to be very flexible, enabling you to configure the pipeline with insane amounts of customization.
The programming language that shaders are written in vary based on the graphics API you use, but they all get compiled into an instruction set that is specific to the vendor that makes the GPU.
Much like how EXEs are typically compiled from a high level language into machine code, shaders are compiled into GPU specific machine code.
Now here's where the fun begins.
Modern emulators need to process the game's programmable shaders, but more importantly, if they want run the game at full speed, they'll need to run the game's shaders on your computer's GPU.

## Recompiling game shaders to run on your GPU

So, recap.
The game has a shader program, and the game's shader is shipped with the game, precompiled for the console's GPU, and the console's GPU is not the same as the GPU in your PC.
The enterprising emulator developer must write a decompiler for that specific GPU assembly code, and turn it into code that can be compiled and run on your PC's GPU.
First problem, figuring out what the console's GPU does - we can talk more about this another day.
Second problem, now that you know more about what the GPU does, how does the efficient emulator developer *actually* convert console's code into code for your GPU?
Good question, but it's a long story that I'll cover another day.
Woo hoo, now that we hand waved away many many hundreds of hours of work, time to get to the original topic, reusing shaders.

## Storing them for reuse

For those that actually play games on modern emulators, instead of spending their time tweaking settings and complaining on the forums about their favorite pet issues, a well known problem many modern emulators all seem to share is occassional stuttering.
Load a new zone? Pauses.
Slice your sword for the first time? Freezes.
These stutters are usually caused by the game submitting the shader program to the console's GPU for the first time, which means the emulator has to decompile it and then recompile it.
Decompiling is usually fast enough to not cause lag, but ohhh boy, compiling takes a noticeable amount of time.
Its not something an emulator can avoid either; the emulator doesn't know anything about the shaders before the game sends it, so you can't just scan the game code and find all of the shaders.
And if the emulator wants to be fast, running the shaders on your CPU isn't a good option either, since shaders are meant to be run on hundreds of cores at the same time, and thats just what GPU's do best.
There's a few different ways to tackle the stutter issue, all of them have different trade-offs and there's no perfect solution.
Some emulators do asynchronous shader compilation, which means the emulator will put shader compilation onto a separate thread, and run the compiling in the background while continuing the draw call.
This is pretty cool when avoiding stutter, but also causes "pop-in" where objects textures and effects will be missing until suddenly it pops into view.
As I explained previously, shaders are responsible for generating, positioning, and animating just about everything in the 3D world, meaning if the emulator chooses to continue after skipping that draw, then it makes sense for stuff to be missing in the world until the shader is finished compiling and "pop" it starts showing up again.
Another technique is to just not compile the game's shaders at all.
Dolphin is a prime example of this technique, where instead of recompiling the game shader into your GPU's shader, it generates a few extremely massive shaders that act as a shader interpreter.
This means that dolphin doesn't have to wait for a shader to compile because you've already compiled the only shaders you'll need.
They call it Ubershaders, and its probably worthy of a whole 'nother video about it.
The final and most common option is to write the shaders to your hard drive, so when the emulator starts loading a game, it can also preload all of the shaders.
Since every shader is stored on the disk and preloaded when launching the game, the gamer will only encounter that specific stutter once throughout their entire playthrough.
This option is still compatible with asynchronous shader compile too, making it very attractive as you can completely remove all shader compilation stutter entirely, while having a mostly glitch free playthrough.

## What's in the ~~wonder ball~~ shader cache?

So whats inside those scrumptious shader storages that make play so silky smooth?
By now, we understand that there are three major transformations of the game's shader programs, the original binary shaders for the console's GPU, the decompiled source code for your GPU, and the compiled binary for your GPU, so you might be wondering which of these are stored in the shader cache?
Drum roll please.
All of them. (confetti)
I gave a quick definition of caching at the beginning of the video, but now we need a quick chat about cache hits (data is in the cache), cache misses (data is not in the cache), and cache invalidation (data is out of date and needs reloaded).
Inside the emulator's shader cache, a cache hit means the shader has already been recompiled, so we can reuse that shader instead.
A cache miss means that this is the first time the game is uploading that particular shader, so we need to recompile it.
And lastly, there really isn't anything that would cause a cache invalidation, as anything that game can do to change the shader is really just a new shader at that point, so it might as well just be treated as a cache miss.
That's how the cache works inside the emulator, but when you write it to disk and reload it from disk, there's a few new problems we need to deal with when considering cache invalidation.
When it comes to shaders, the resulting shader binary that you run on your GPU is all that matters, but while reloading the compiled binaries from your GPU, the GPU can reject them for whatever reason (the most common cause is driver updates).
This means that if the disk cache just stored compiled binaries, one day you'll have a full cache, then update your GPU drivers, and then your cache is all gone.
So the emulator needs to store more information in order to recompile the shaders if the driver rejects them, and thats exactly what the decompiled source code is all about.
Using the decompiled source code, the emulator can recompile it 

## Can't share shaders 

A quick notice about legality of shaders.
Shaders are game code, and code is protected by copyright, and copyright laws prohibit redistribution of their work.
**I'M NOT A LAWYER**
So I'm not going to even pretend that this is some authoritative legal discussion, but the gist is emulator devs can't redistribute 




