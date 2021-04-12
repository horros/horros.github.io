---
layout: post
title:  Z80 emulator in C#
date:   2021-04-12 17:45:19 +0300
---

Some time ago I bought a Z80 processor to see if I could make it blink some leds and just maybe make it interact with something. Needless to say, life and 100 other projects came in the way and I never got around to doing that. I'm also not at all familiar with the Z80 having been a 6502-kid with my Commodore 64, so now that I got an inspiration boost I figured "what better way to understand how the Z80 works than write an emulator for it in C#"? The answer I reckon is "loads of better ways", but I'm never one to do things the easy way. Keep in mind I'm writing this as I go along, so huge refactorings are probably inevitable.

The Z80 is an 8-bit microprocessor by Zilog from the mid-70s, and was designed by the legendary Federico Faggin, and is in some shape or form still in production to this day. It's got a 16-bit wide memory address and can use a total of 64kB of memory (without bank switching I think, but that's out of scope for now).

So how does a CPU work on a fundamental level? This of course depends on the CPU, but _usually_ the CPU has a built-in instruction set, and when it is powered on it jumps to a predefined location in memory called the _reset vector_, where it tries to execute whatever instructions it find there. Once it executes an instruction, it increments the _program counter_ which points to the next address in memory from where the CPU tries to read and execute the next instruction and so on and so forth. If there's nothing in the memory where the CPU tries to read, it will probably read junk until the program counter overflows, and start again from zero. For the 6502 the reset vector is at address 0xFFFC, but for the Z80 things are even simpler, it's 0x0000.

CPUs execute instructions in what's called "cycles". Simplified these are controlled by a crystal oscillator that generates a pulse at a steady interval. For old 8-bit computers this was usually in the range of 4 to 10 million times per second, or 4 to 10MHz. When the CPU gets a signal it does *one* thing. This can be "fetch an instruction from memory", "decode an instruction", "execute an instruction" and so on. Each _instruction cycle_ takes one or more _clock cycles_. For this first example I don't think we actually need to bother with this, but it's good to know.

So what do we need to get started? Obviously we need the CPU. The CPU has a bunch of registers that are used for different things like storing data, storing the _program counter_ and the _stack pointer_, and loads of other things. The registers on a Z80 are a bit weird, they can either function as one 8-bit register or two of them can be cobbled together to form a faux 16-bit register. Eg. the H and L registers are usually used together as a 16-bit HL-register, and the A and F registers are usually combined to an AF-register, where the A register is an 8-bit accumulator and the F register is a flag-register. For our CPU I reckon for now we need a 16-bit program counter which will point at the next memory location for the CPU to read, and the HL-register. We'll start with a default C# console app and add the CPU to the same file.

```csharp

namespace z80sharp
{
    class Program
    {
        static void Main(string[] args)
        {
        }
    }

    
    public static class CPU 
    {
        public static ushort HL; // "16-bit" register
        public static ushort PC; // program counter
    }
}
```

We obviously also need some memory. We obviously can't read from memory if we don't have any memory! We also need to initialize the program counter to point to the start of the memory space since that's where the Z80 starts executing.

```csharp

namespace z80sharp
{
    class Program
    {
        static void Main(string[] args)
        {
            byte[] memory = new byte[0xFFFF]; // 64k memory space
            CPU.PC = 0x0000; // point to the beginning of our memory space
        }
    }

    
    public static class CPU 
    {
        public static ushort HL; // 16-bit register
        public static ushort PC; // program counter
    }
}
```
For good measure, let's add some extension methods to the memory just to make it a tiny bit easier to read.

```csharp
public static class MemoryExtensions {
    public static byte ReadMemory(this byte[] memory, ushort address) {
        return memory[address];
    }
    
    public static void WriteMemory(this byte[] memory, ushort address, byte value) {
        memory[address] = value;
    }
}
```

The instruction set for the Z80 is a bit odd IMO, as I can't seem to find a way to eg. load an 8-bit value into the A-register, but maybe we'll figure this out during our journey. Looking through the instruction set of the Z80 makes me wish I'd started this with the 6502 instead. A list of instructions can be found here: <https://clrhome.org/table/#>. To find the hexadecimal opcode, you read the vertical row first, and then the horizontal row. Armed with this we can see that the opcode for ```ld hl,**``` is 0x21. This using what is called "extended immediate addressing mode" as it instructs the CPU to load into the HL register whatever 16-bits we provide, instead of loading eg. contents of other registers or so. For more information on the Z80 addressing modes, have a peek at this <https://8bitnotes.com/2017/05/z80-addressing-modes/>. If I try to think of the simplest program that actually does *something*, I'd say it would be to load some value into a register, so ```ld hl,**``` sounds like a good candidate.

To execute opcodes from memory we obviously need to let the CPU access it somehow. How the Z80 (or the 6502) does this in "real life" is outside the scope of this blog post, but a 1000 mile overview is the CPU has some address pins and a memory bus, and tries to read the memory bus at the location the addressing pins indicate. For our needs we'll just pass in a reference to the memory array for our upcoming Execute-method. As the first thing the CPU does is read instructions from the location pointed at by the Program Counter, we'll do just that! And since every instruction needs to increment the program counter so the CPU knows where the next instruction is, we'll do that too.

```csharp
namespace z80sharp
{
    class Program
    {
        static void Main(string[] args)
        {
            byte[] memory = new byte[0xFFFF]; // 64k memory space
            CPU.PC = 0x0000; // point to the beginning of our memory space
        }
    }

    public static class CPU {
        public static ushort HL; // 16-bit register
        public static ushort PC; // program counter
        
        public static void Execute(ref byte[] memory) {
            byte instruction = memory.ReadMemory(PC);
            PC++;
        }
    }

}
```

This won't be very interesting yet as we have nothing in memory. The opcode for LD HL, xx was 0x21, so we could put that in memory loaction 0x0000, and the 16 bits of data in the next two 8-bit memory locations. Something like

```csharp
    memory.WriteMemory(0x0000, 0x21);
    memory.WriteMemory(0x0001, 0x42);
    memory.WriteMemory(0x0002, 0x42);
```

Now we should have the memory contain the representation of ```LD HL, 0x4242```. However, we still can't do much with them since our CPU doesn't know what 0x21 means. There are tons of ways to solve this, but we'll just use a simple enum for our instructions for now. I think all opcodes are 8-bit, so we'll make a byte enum.

```csharp
public enum Instruction : byte {
    LD_HL_VAL = 0x21 // load a value into the HL register
}
```

Then we can use a BIG SWITCH O' DOOM in the execute method to figure out what instruction we just read, read the next two bytes, cobble them together to get a 16-bit word, and put that in the HL-register. Sidenote, the Z80 is a little-endian CPU, and as far as I know all Windows machines are little-endian. If you're running on a big-endian system, you need to swap the first and second bytes, otherwise it'll be wrong.

```csharp
public static void Execute(ref byte[] memory) {
    byte instruction = memory.ReadMemory(PC);
    PC++;
    switch (instruction) {
        case (byte)Instruction.LD_HL_VAL:
            // read the next two 8-bit memory locations
            byte firstByte = memory.ReadMemory(PC);
            PC++; // Increment the program counter
            byte secondByte = memory.ReadMemory(PC);
            PC++; // Increment the program counter again
            HL = (ushort)((firstByte << 8)+(second));
            break;
    }
}
```

Now we should have something like this:

```csharp
using System;

namespace z80sharp
{
    class Program
    {
        static void Main(string[] args)
        {

            byte[] memory = new byte[0xFFFF]; // 64k address space
            memory.WriteMemory(0x0000, 0x21);
            memory.WriteMemory(0x0001, 0x42);
            memory.WriteMemory(0x0002, 0x42);
            CPU.PC = 0x0000; // Initialize the program counter
            CPU.Execute(ref memory); // Let's goooo!
            
        }
    }

    public static class CPU {

        public static ushort HL; // 16-bit register
        public static ushort PC; // program counter

        public static void Execute(ref byte[] memory) {

            byte instruction = memory.ReadMemory(PC);
            PC++;
            switch (instruction) {
                case (byte)Instruction.LD_HL_VAL: // extended load immediate 16-bit value into HL
                    byte firstByte = memory.ReadMemory(PC);
                    PC++;
                    byte secondByte = memory.ReadMemory(PC);
                    PC++;
                    HL = (ushort)((firstByte << 8)+(secondByte));
                    break;
            }
        }
        public enum Instruction : byte {
            LD_HL_VAL = 0x21
        }
    }

     public static class MemoryExtensions {
        public static byte ReadMemory(this byte[] memory, ushort address) {
            return memory[address];
        }

        public static void WriteMemory(this byte[] memory, ushort address, byte value) {
            memory[address] = value;
        }
    }
}
```

If we set a breakpoing inside the Execute method and set a watch on CPU.HL and CPU.PC and step through, we'll se that the first instruction we read is 33 decimal, or 0x21 hex. Both firstByte and secondByte contain the decimal value 66 (or 0x42 hex), and the HL register contains the value 16962 which is indeed 0x4242 hex! Success!

![SUCCESS](/assets/images/z80cpu-1/cpuregisters.png)

Now, another simple instruction that we can use is the "JP (HL)" instruction, which causes the CPU to jump to the memory location set in the HL register. The opcode for this instruction is 0xE9, so let's add that to the Instruction-enum. This instruction doesn't really need to do anything other than set the PC to the value of HL, so this is easy enough. Note that we don't increment the program counter, as the JP opcode sets the program counter to what it needs to be.

```csharp
        public static void Execute(ref byte[] memory) {

            byte instruction = memory.ReadMemory(PC);
            PC++;
            switch (instruction) {
                case (byte)Instruction.LD_HL_VAL: // extended load immediate 16-bit value into HL
                    byte firstByte = memory.ReadMemory(PC);
                    PC++;
                    byte secondByte = memory.ReadMemory(PC);
                    PC++;
                    HL = (ushort)((firstByte << 8)+(secondByte));
                    break;
                case (byte)Instruction.JP_HL:
                    PC = HL;
                    break;
            }
        }
        public enum Instruction : byte {
            LD_HL_VAL = 0x21,
            JP_HL = 0xE9
        }
```

Currently we only run the Execute method once, so we'll only read one opcode and then stop. We can wrap this in a loop and just keep executing like a normal CPU would. We'll of course crash when we try to read off of the end of the memory location, but that'll do for now. We have now set the program counter to point to 0x4242, so let's add some new instructions at that memory location too. We'll just use the same LD HL but instead of 0x4242 we'll use 0x9999. When we put all of this together we should have the following little app

```csharp
using System;

namespace z80sharp
{
    class Program
    {
        static void Main(string[] args)
        {

            byte[] memory = new byte[0xFFFF]; // 64k address space
            memory.WriteMemory(0x0000, 0x21); // Load into HL the following two bytes
            memory.WriteMemory(0x0001, 0x42);  
            memory.WriteMemory(0x0002, 0x42);
            memory.WriteMemory(0x0003, 0xE9); // Jump to the memory location in the HL register
            memory.WriteMemory(0x4242, 0x21); // Load into HL the following two bytes
            memory.WriteMemory(0x4243, 0x99);
            memory.WriteMemory(0x4244, 0x99);
            CPU.PC = 0x0000; // Initialize the program counter

            do {
                CPU.Execute(ref memory); // Let's goooo!
            } while (true);
                        
        }
    }

    public static class CPU {

        public static ushort HL; // 16-bit register
        public static ushort PC; // program counter

        public static void Execute(ref byte[] memory) {

            byte instruction = memory.ReadMemory(PC);
            PC++;
            switch (instruction) {
                case (byte)Instruction.LD_HL_VAL: // extended load immediate 16-bit value into HL
                    byte firstByte = memory.ReadMemory(PC);
                    PC++;
                    byte secondByte = memory.ReadMemory(PC);
                    PC++;
                    HL = (ushort)((firstByte << 8)+(secondByte));
                    break;
                case (byte)Instruction.JP_HL:
                    PC = HL;
                    break;
            }
        }
        public enum Instruction : byte {
            LD_HL_VAL = 0x21,
            JP_HL = 0xE9
        }
    }

     public static class MemoryExtensions {
        public static byte ReadMemory(this byte[] memory, ushort address) {
            return memory[address];
        }

        public static void WriteMemory(this byte[] memory, ushort address, byte value) {
            memory[address] = value;
        }
    }
}
```

Setting a breakpoint on like 34 and stepping through we see that we do indeed first load 0x4242 into the HL register, execute the JP opcode and jump to memory location 0x4242, and again load the value 0x9999 into the HL register! Having a watch on CPU.HL we see that it contains the decimal value 39321 which is indeed 0x9999! We have successfully written a tiny Z80 emulator that - granted - doesn't do anything terribly useful!

![SUCCESS AGAIN](/assets/images/z80cpu-1/cpuregisters2.png)

If you let the program run it will keep incrementing the program counter and attempting to find an instruction to execute, until we try to read outside our memory array and crash with

![CRASH](/assets/images/z80cpu-1/exception.png)

That's the end of part 1, I'm really quite satisfied with what we've achieved so far!