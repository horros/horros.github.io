---
layout: post
title:  Z80 emulator in C# - Part 2
date:   2021-04-14 18:45:19 +0300
comments: true
---

Last time we got a rudimentary emulator up and running. In part 2 we'll add some more instructions that are necessary to be able to do anything remotely useful.

First things first, there is INDEED a ```LD a, val``` opcode, and it's ```0x3E```, I'm just - what we'd call in finnish - a wood eye. So let's implement that before we do anything else, as the A-register is the accumulator and is used in pretty much every arithmetic operation, so very important indeed!

Even _firster_, we can make our "program" in memory more readable by actually referencing the opcodes by name. So instead of using ```memory.WriteMemory(0x0001, 0x21);``` we can write ```memory.WriteMemory(0x0001, (byte)CPU.Instruction.LD_HL_VAL);``` which makes it feel almost assembler-y. We could change this to eg. an internal class with constants instead so we don't have to cast to a byte all the time, and I'll probably do that at some later stage.

Having changed those, let's implement ```LD a, val```. This should be pretty familiar by now, add another entry to the instruction-enum with ```LDA = 0x3E``` and let's add that as the second actual operator in our program.

```csharp
memory.WriteMemory(0x0000, (byte)CPU.Instruction.LD_HL_VAL); // Load into HL the following two bytes
memory.WriteMemory(0x0001, 0x42);  
memory.WriteMemory(0x0002, 0x42);
memory.WriteMemory(0x0003, (byte)CPU.Instruction.LDA); // Load the next byte into the accumulator
memory.WriteMemory(0x0004, 0xEA);
```

We also need the accumulator in the CPU, so let's add

```csharp
public static byte A; // 8-bit accumulator
```

below the other register definitions in the CPU class.

If we take a look at the instruction set we'll notice there are a bunch of ```INC <reg>``` and ```DEC <reg>```. These increment or decrement the register by one. These are operations that are very often used so they are optimised to be as fast as possible. Let's implement ```INC a``` (```0x3C```) and ```DEC a``` (```0x3D```) too while we're at it!

We'll add this to the Instruction-enum:

```csharp
LDA = 0x3E,
INC_A = 0x3C,
DEC_A = 0x3D
```

And the following to our instruction switch. This couldn't be simpler:

```csharp
case (byte)Instruction.LDA:
    A = memory.ReadMemory(PC);
    PC++;
    break;
case (byte)Instruction.INC_A:
    A++;
    break;
case (byte)Instruction.DEC_A:
    A--;
    break;
```

And for fun, let's increment twice and decrement once!

```csharp
memory.WriteMemory(0x0000, (byte)CPU.Instruction.LD_HL_VAL); // Load into HL the following two bytes
memory.WriteMemory(0x0001, 0x42);  
memory.WriteMemory(0x0002, 0x42);
memory.WriteMemory(0x0003, (byte)CPU.Instruction.LDA); // Load the next byte into the accumulator
memory.WriteMemory(0x0004, 0xEA);
memory.WriteMemory(0x0005, (byte)CPU.Instruction.INC_A); // Increment the A register by one
memory.WriteMemory(0x0006, (byte)CPU.Instruction.INC_A); // Increment the A register by one
memory.WriteMemory(0x0007, (byte)CPU.Instruction.DEC_A); // Decrement the A register by one
```

If we run this with a watch on the CPU registers we'll see that it first gets set to ```0xEA```, then incremented once to ```0xEB```, incremented again to ```0xEC``` and finally decremented once back to ```0xEB```! Nice, we can now do rudimentary arithmetic!

![1stinc](/assets/images/z80cpu-2/1stinc.png)

_We just ran the first increment operation, incrementing 0xEA to 0xEB._

![1stinc](/assets/images/z80cpu-2/2ndinc.png)

_Second increment operation, incrementing 0xEB to 0xEC._

![1stinc](/assets/images/z80cpu-2/3rddec.png)

_The decrement operation, decrementing 0xEC back to 0xEB._

_Note: I switched to Visual Studio because you can have the watch window format the watch as hexadecimal._

Let's move on to even more interesting things than adding or removing one from a number!

Computers use something called the _stack_. It's an area in the memory that works kinda like a stack of books. Let's say we push our first book, _JRR Tolkien: The Fellowship of the Ring_, onto our "stack of books" on the floor. We only have one book there now. Then we push our second book, _Raymond E. Feist: Magician_ onto the stack. We now have two books on top of each other. Then we push _George R.R. Martin: A Game of Thrones_ onto the stack, and we now have three books. If we want to remove books, we can't just pull _The Fellowship of the Ring_ from the stack, everything else would topple over. We must remove the one on top first, so we pop _A Game of Thrones_ off the stack. This is _literally_ _exactly_ how a stack in computer memory works.

A stack is (usually) a _LIFO stack_ (Last in, first out). The last thing you push onto the stack is the first thing that gets popped off the stack. Stacks are used in low level programming as a convenient way to store eg. addresses when we jump around in the code. So let's implement a stack and some operations!

There is nothing special about the stack memory otherwise, except the fact that it works backwards. So when you push things onto the stack you _decrement_ the stack pointer, and when you pop things off the stack you _increment_ the stack pointer. The stack on the Z80 can be anywhere you want, by convention it is usually set to the top of the memory map, so in our case we should set the stack pointer to ```0xFFFF```.

First things first, let's add the 16-bit stack pointer to the CPU class. Add the following below the program counter

```csharp
public static ushort SP; // stack pointer
```

```csharp
class Program
{
    static void Main(string[] args)
    {

        byte[] memory = new byte[0xFFFF]; // 64k address space
        memory.WriteMemory(0x0000, (byte)CPU.Instruction.LD_HL_VAL); // Load into HL the following two bytes
        memory.WriteMemory(0x0001, 0x42);  
        memory.WriteMemory(0x0002, 0x42);
        memory.WriteMemory(0x0003, (byte)CPU.Instruction.LDA); // Load the next byte into the accumulator
        memory.WriteMemory(0x0004, 0xEA);
        memory.WriteMemory(0x0005, (byte)CPU.Instruction.INC_A); // Increment the A register by one
        memory.WriteMemory(0x0006, (byte)CPU.Instruction.INC_A); // Increment the A register by one
        memory.WriteMemory(0x0007, (byte)CPU.Instruction.DEC_A); // Decrement the A register by one
        CPU.PC = 0x0000; // Initialize the program counter
        CPU.SP = 0xFFFF; // Initialize the stack pointer
        do {
            CPU.Execute(ref memory); // Let's goooo!
        } while (true);
                    
    }
}
```

If we look at the opcode tables, we find a ```PUSH HL``` opcode that pushes the value in the HL register onto the stack. Because the memory locations are 8 bit and HL is a "16 bit" register, we need to split the value and store them separately. The documentation says when we push HL onto the stack, we first _decrement_ SP, write the value in the H-register (remember HL is a fake 16-bit register, it's really two 8-bit registers) to the memory location that SP indicates, then we decrement SP again and write the value in the L-register to the memory location indicated by SP. Knowing this, and the fact that we know ```PUSH HL``` is 0xE5, we can now write the code to handle this!

First we'll add the new opcode to our enum that should at this point look something like this:

```csharp
public enum Instruction : byte {
    LD_HL_VAL = 0x21,
    JP_HL = 0xE9,
    LDA = 0x3E,
    INC_A = 0x3C,
    DEC_A = 0x3D,
    PUSH_HL = 0xE5
}
```

Then we'll add the case to our **BIG SWITCH O' DOOM** to handle the opcode. Here when we split the value in HL into two bytes, we'll use the very convenient BitConverter class. The GetBytes-method will return an array of bytes representing whatever you pass it as the parameter. This _could_ be done with some bitshifting and bitwise operator madness, but they are (for me) very hard to read and are error prone. This is way easier and neater.

```csharp
case (byte)Instruction.PUSH_HL:
    byte[] bytes = BitConverter.GetBytes(HL); // Convenience and clarity,
                                              // This could be done with bitshifting etc
    SP--;
    memory.WriteMemory(SP, bytes[0]);
    SP--;
    memory.WriteMemory(SP, bytes[1]);
    break;
```

If you're curious, you could do something like

```csharp
byte byte1 = (byte)(HL & 0x00ff); 
byte byte2 = (byte)(HL >> 8);
```

All that remains is adding the instruction to our little program in memory and change the second LD HL's arguments a little bit to easier see what's happening:

```csharp
memory.WriteMemory(0x0000, (byte)CPU.Instruction.LD_HL_VAL); // Load into HL the following two bytes
memory.WriteMemory(0x0001, 0x42);  
memory.WriteMemory(0x0002, 0x42);
memory.WriteMemory(0x0003, (byte)CPU.Instruction.LDA); // Load the next byte into the accumulator
memory.WriteMemory(0x0004, 0xEA);
memory.WriteMemory(0x0005, (byte)CPU.Instruction.INC_A); // Increment the A register by one
memory.WriteMemory(0x0006, (byte)CPU.Instruction.INC_A); // Increment the A register by one
memory.WriteMemory(0x0007, (byte)CPU.Instruction.DEC_A); // Decrement the A register by one
memory.WriteMemory(0x0008, (byte)CPU.Instruction.JP_HL); // Jump to the memory location in the HL register
memory.WriteMemory(0x4242, (byte)CPU.Instruction.LD_HL_VAL); // Load into HL the following two bytes
memory.WriteMemory(0x4243, 0x98);
memory.WriteMemory(0x4244, 0x76);
memory.WriteMemory(0x4245, (byte)CPU.Instruction.PUSH_HL); // Push the value from the HL register onto the stack
```

Visual Studio can already format the contents of watches as hexadecimal, but if you are using VS Code you can add some formatting by hand. This should work in any IDE / editor with decent debugger support. Add the following watches:

```text
CPU.HL.ToString("X")
CPU.PC.ToString("X")
CPU.SP.ToString("X")
CPU.A.ToString("X")
instruction.ToString("X")
memory[0xFFFE].ToSTring("X")
memory[0xFFFD].ToString("X")
```

Again setting a breakpoint on the first (executable) line in the Execute-method and stepping through the code with the debugger and our eyes on the watch-window, we see that our PUSH opcode is executed and 0x9876 is pushed onto the stack (backwards)! Hooray! Now let's also implement popping a value off the stack into the HL register! The opcode for this is 0xE1, and follows the same principle but the other way around; read the memory at the location SP is pointing to, store that value in the L register, increment the stack pointer, read the memory at the location SP is pointing to, store that value in the H register and increment the stack pointer.

Let's add the opcode:

```csharp
public enum Instruction : byte {
    LD_HL_VAL = 0x21,
    JP_HL = 0xE9,
    PUSH_HL = 0xE5,
    POP_HL = 0xE1
}
```

And the case (again, remember we have the 8-bit values stored "the other way around"):

```csharp
case (byte)Instruction.POP_HL:
    byte popFirstByte = memory.ReadMemory(SP);
    SP++;
    byte popSecondByte = memory.ReadMemory(SP);
    SP++;
    HL = (ushort)((popSecondByte << 8)+(popFirstByte));
    break;
```

and the instruction to our little application:

```csharp
memory.WriteMemory(0x0000, (byte)CPU.Instruction.LD_HL_VAL); // Load into HL the following two bytes
memory.WriteMemory(0x0001, 0x42);  
memory.WriteMemory(0x0002, 0x42);
memory.WriteMemory(0x0003, (byte)CPU.Instruction.LDA); // Load the next byte into the accumulator
memory.WriteMemory(0x0004, 0xEA);
memory.WriteMemory(0x0005, (byte)CPU.Instruction.INC_A); // Increment the A register by one
memory.WriteMemory(0x0006, (byte)CPU.Instruction.INC_A); // Increment the A register by one
memory.WriteMemory(0x0007, (byte)CPU.Instruction.DEC_A); // Decrement the A register by one
memory.WriteMemory(0x0008, (byte)CPU.Instruction.JP_HL); // Jump to the memory location in the HL register
memory.WriteMemory(0x4242, (byte)CPU.Instruction.LD_HL_VAL); // Load into HL the following two bytes
memory.WriteMemory(0x4243, 0x98);
memory.WriteMemory(0x4244, 0x76);
memory.WriteMemory(0x4245, (byte)CPU.Instruction.PUSH_HL); // Push the value from the HL register onto the stack
memory.WriteMemory(0x4246, (byte)CPU.Instruction.POP_HL); // Pop the value from the stack into the HL register
```

Running it and keeping an eye on our debugger watch window again shows us we are indeed popping the correct value off the stack and the stack pointer is back to where we expect it to be!

The complete emulator should now look something like this:

```csharp
using System;

namespace z80sharp
{
    class Program
    {
        static void Main(string[] args)
        {

            byte[] memory = new byte[0xFFFF]; // 64k address space
            memory.WriteMemory(0x0000, (byte)CPU.Instruction.LD_HL_VAL); // Load into HL the following two bytes
            memory.WriteMemory(0x0001, 0x42);  
            memory.WriteMemory(0x0002, 0x42);
            memory.WriteMemory(0x0003, (byte)CPU.Instruction.LDA); // Load the next byte into the accumulator
            memory.WriteMemory(0x0004, 0xEA);
            memory.WriteMemory(0x0005, (byte)CPU.Instruction.INC_A); // Increment the A register by one
            memory.WriteMemory(0x0006, (byte)CPU.Instruction.INC_A); // Increment the A register by one
            memory.WriteMemory(0x0007, (byte)CPU.Instruction.DEC_A); // Decrement the A register by one
            memory.WriteMemory(0x0008, (byte)CPU.Instruction.JP_HL); // Jump to the memory location in the HL register
            memory.WriteMemory(0x4242, (byte)CPU.Instruction.LD_HL_VAL); // Load into HL the following two bytes
            memory.WriteMemory(0x4243, 0x98);
            memory.WriteMemory(0x4244, 0x76);
            memory.WriteMemory(0x4245, (byte)CPU.Instruction.PUSH_HL); // Push the value from the HL register onto the stack
            memory.WriteMemory(0x4246, (byte)CPU.Instruction.POP_HL); // Pop the value from the stack into the HL register
            CPU.PC = 0x0000; // Initialize the program counter
            CPU.SP = 0xFFFF; // Initialize the stack pointer
            do {
                CPU.Execute(ref memory); // Let's goooo!
            } while (true);
                        
        }
    }

    public static class CPU {

        public static ushort HL; // 16-bit register
        public static ushort PC; // program counter
        public static ushort SP; // stack pointer

        public static byte A; // 8-bit accumulator

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
                case (byte)Instruction.LDA:
                    A = memory.ReadMemory(PC);
                    PC++;
                    break;
                case (byte)Instruction.INC_A:
                    A++;
                    break;
                case (byte)Instruction.DEC_A:
                    A--;
                    break;
                case (byte)Instruction.PUSH_HL:
                    byte[] bytes = BitConverter.GetBytes(HL); // Convenience and clarity,
                                                              // This could be done with bitshifting etc
                    SP--;
                    memory.WriteMemory(SP, bytes[1]); // bytes[1] here because we're working backwards
                    SP--;
                    memory.WriteMemory(SP, bytes[0]);
                    break;
                case (byte)Instruction.POP_HL:
                    byte popFirstByte = memory.ReadMemory(SP);
                    SP++;
                    byte popSecondByte = memory.ReadMemory(SP);
                    SP++;
                    HL = (ushort)((popSecondByte << 8)+(popFirstByte));
                    break;
            }
        }
        public enum Instruction : byte {
            LD_HL_VAL = 0x21,
            JP_HL = 0xE9,
            LDA = 0x3E,
            INC_A = 0x3C,
            DEC_A = 0x3D,
            PUSH_HL = 0xE5,
            POP_HL = 0xE1
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

The code is available in a gist here: <https://gist.github.com/horros/a979b6517a8c62baf7f30b2c58272e11>

That's it for part 2, let's see what we come up with for part 3!

(ps. I wrote the stack push/pop part before I noticed the LD A operator which I decided needs to go first, so things might be a little bit wonky.)