---
layout: post
title:  Z80 emulator in C# - Part 3
date:   2021-04-22 18:45:19 +0300
comments: true
---
Now that we have already implemented a few opcodes and such, it _might_ be a good idea to actually test the code on an actual working emulator to make sure we're doing the correct thing. Googling around I found an Amstrad emulator. The Amstrad was quite a popular home computer in the mid- and late 80's. You can download the emulator here: <http://www.winape.net/>. Installing is just unzipping the package where ever you feel like. It needs DirectPlay (anyone still remember that?) but at least my Windows 10 machine asked if I want to install it and happily complied when I said yes.

Firing up the emulator presents us with this beautiful screen:

![WinAPE](/assets/images/z80cpu-3/winape.png)

If you hit F3 you open the assembler. We need to know a few basic things.

- To input hexadecimal values we prepend the value with the &-sign. So ```0x8000``` is ```&8000```.
- Also in Z80 assembler, to reference the memory location pointed at by a register we give the register in parenthesis. So eg ```JP (LH)``` which means "jump to the memory location that is stored in the register LH", because "jump to the memory location LH" doesn't make any sense.
- The Amstrad has a block of RAM starting at address 0x8000 that we can use. Below there's some ROM and other stuff we can't poke around in.
- To tell the assembler where we want to start in memory, we use the keyword "Org".

So let's try to write our app in actual Z80 assembler on an actual working Z80 emulator!

Hit F3 in WinAPE and then File -> New to start a new assembly file.

If we look at our Z80 program we see that we are jumping to address ```0x4242```. Since this is below our ```0x8000``` starting point, we'll use ```0x9999``` instead. Let's see what we can put together.

{% highlight nasm %}
Org &8000
LD hl, &9999
LD a, &EA
INC a
INC a
DEC a
JP (hl)

Org &9999
LD hl, &9876
PUSH hl
LD HL, &0000
POP hl
RET
{% endhighlight %}

This seems ok. Start at memory address 0x8000, load 0x9999 into the HL register, load 0xEA into the A-register, increment A twice, decrement once, and jump to memory location 0x9999. At 0x9999 store 0x9876 in HL, push HL onto the stack and then pop the stack and place it in HL. I added an extra ```LD HL, &0000``` to set HL to 0 so it's easier to see that we're actually popping the value off the stack and into HL.

Hit F9 to assemble the program. A window will open where you can see that the hex opcodes that the assembler produces is almost 1:1 with our application! The only thing that differs is memory location 8009 that contains "(9999)" and the sequence ```0x21 0x00 0x00``` which is ```LD hl, 0x0000```! I'm not quite sure what the (9999) is about, obviously it's the jump address, but the documentation for ```JP (HL)``` just says "Loads the value of HL into the PC" so it is probably related to the ```Org &9999``` telling the assembler where the rest of our code goes.

If you click on the gutter (the gray bar) to the left of the assembler code window you can set breakpoints. Set one on the first ```LD``` operation and in the Amstrad emulator window type in ```call &8000``` and hit enter. This starts executing the code in ```&8000```, and should immediately stop and open the debugger window. You can keep an eye on the registers in the right hand window (keep in mind, A is an 8-bit register, and the debugger shows AF, so the combined 16-bit register of A and F - the two first bytes are the A-register). F7 steps through the code, and if we step through line by line we see that the emulator matches our puny emulator!

Note that last ```RET``` instruction in our assembly code? That instructs the CPU to pop the last thing off the stack and put it in the PC. This basically means jumping back to whatever called our program in the first place, which is some form of Amstrad operating system (or BASIC or something) that sits in a loop and waits for user input, and that ```call &8000``` is opcode ```0xCD```. That pushes the current PC plus three onto the stack (which makes stack point to the memory location 3 bytes after the CALL operation, so right after the 16 bit address or ```0x8000``` in this case), then sets the PC to that value, causing the CPU to jump to that memory location. When the program (or subroutine if you will) in that memory location finishes executing and executes the ```RET```-instruction, it will pop the last thing off the stack - which is the "PC + 3" address we pushed onto it - and place that in PC, thus causing the CPU to resume execution right after the ```CALL``` operation which returns us back to that input loop. _Note, this is my speculation, I've not actually checked what the Amstrad does._

Why don't we implement the ```RET``` stuff in our emulator!

```RET``` is opcode ```0xC9```, and we'll add that and ```CALL``` (```0xCD```) to our Instruction enum. For our little subroutine, we'll set the A-register to ```0x0A```, jump to the subroutine, increment the A-register a few times and then return and maybe increment a few times again.

```csharp
memory.WriteMemory(0x4247, (byte)CPU.Instruction.LDA);
memory.WriteMemory(0x4248, 0x0A);
memory.WriteMemory(0x4249, (byte)CPU.Instruction.CALL);
memory.WriteMemory(0x424A, 0x88);
memory.WriteMemory(0x424B, 0x44);
memory.WriteMemory(0x424C, (byte)CPU.Instruction.INC_A);
memory.WriteMemory(0x424D, (byte)CPU.Instruction.INC_A);
memory.WriteMemory(0x424E, (byte)CPU.Instruction.INC_A);
memory.WriteMemory(0x8844, (byte)CPU.Instruction.INC_A);
memory.WriteMemory(0x8845, (byte)CPU.Instruction.INC_A);
memory.WriteMemory(0x8846, (byte)CPU.Instruction.INC_A);
memory.WriteMemory(0x8847, (byte)CPU.Instruction.RET);
```

So what we do is load ```0x0A``` into the A-register, then call (read: jump to) ```0x8844```, and if we look at what we put in memory location ```0x8844-0x8847```, it's three ```INC a``` instructions and the ```RET``` instruction, which returns us to address ````0x424C``` and increment the A register three more times, so at the end of it all our A-register should contain ```0x10```.

The case handling the two new instructions looks like this:

```csharp
case (byte)Instruction.CALL:
    byte[] PCbytes = BitConverter.GetBytes(PC+2); // we have already incremented PC once
    SP--;
    memory.WriteMemory(SP, PCbytes[1]);
    SP--;
    memory.WriteMemory(SP, PCbytes[0]);
    byte callByte1 = memory.ReadMemory(PC);
    PC++;
    byte callByte2 = memory.ReadMemory(PC);
    PC = (ushort)((callByte1 << 8) + (callByte2));
    break;
case (byte)Instruction.RET:
    byte retFirstByte = memory.ReadMemory(SP);
    SP++;
    byte retSecondByte = memory.ReadMemory(SP);
    SP++;
    PC = (ushort)((retSecondByte << 8) + (retFirstByte));
    break;
```

The call instruction first gets the value of the program counter + 2 (we already incremented the PC by one after we read the instruction at the top of the switch statement), so as our program counter is at ```0x4249```, we'll get out ```0x4249 + 3 = 0x424C```. We then push that address onto the stack and set PC to the next two bytes (addresses ```0x424A = 0x88``` and ```0x424B = 0x44```) causing execution to jump to that address.

The ```RET``` instruction reads whatever is on top of the stack, which is our return address ```0x424C``` and sets the program counter to that, causing us to jump back to that address and continue execution (incrementing the A-register three more times).

Setting a watch on ```CPU.A``` and stepping through the code, we se that we indeed jump to the correct place in memory increment the A-register, jump back and increment the A-register again, ending up with ```0x10``` in the register! I'll leave it as an exercise for the reader to try it out in the Z80 emulator, the full C# code is available in this gist: <https://gist.github.com/horros/3b85c850af3c5087dd4cef4aff8df148>.

That's it for this article, next time we'll have a look at the registers and implement some *MAFFS*, maybe even figure out how to do multiplication and division which aren't a part of the Z80's instruction set!

<div id="disqus_thread"></div>
<script>

    var disqus_config = function () {
    this.page.url = "{{site.url}}{{page.url}}";
    this.page.identifier = "{{page.id}}";
    };
    
    (function() { // DON'T EDIT BELOW THIS LINE
    var d = document, s = d.createElement('script');
    s.src = 'https://markusblog-2.disqus.com/embed.js';
    s.setAttribute('data-timestamp', +new Date());
    (d.head || d.body).appendChild(s);
    })();
</script>
<noscript>Please enable JavaScript to view the <a href="https://disqus.com/?ref_noscript">comments powered by Disqus.</a></noscript>