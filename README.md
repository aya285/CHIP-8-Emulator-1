# Basic Implementation of Emulator/Interpreter on Python


As we saw in the ICS course, it is possible to build our own virtual machine in our computers just by programming, 
so I got the idea to try to implement a theoretical machine with less than 50 instructions.
So CHIP8 was the best choice. It is not really a physical machine but an interpreter which runs its programs
in a virtual machine.
It was developed by Joseph Weisbecker and mostly used on graphing calculators.

## Machine Components
Every, as treated in Christoph KIRSCH's book, is made out of 3 mains parts:
- Input/Output or as referred to by I/O
- CPU
- Memory

***This emulator is basically meant for Raspberry Pi, a single-board computer(SBC)***

*Input*

As for the input, for the CHIP8 virtual machine, the input comes from a 16-button keyboard. It is also fed with
the program it is supposed to run.

*Output*

For output,the machine uses a 64x32 display, and a simple sound buzzer.
The display is basically just an array of pixels that are either in the on or off state.

*Memory*

CHIP8 has memory that can hold up to **40496** bytes.
This includes the interpreter itself, the fonts, and where it loads the program it is supposed to run (from input).

*Registers*

The CHIP8 has **16 8-bit registers** (usually referred to by Vi where i is the register's number so essentially from 0 to 15). 
There are used as mentioned in the book to store values for operations and procedures.
The last register is mostly used for flags and should be avoided for use in programs (works in the same way as in the zero register in selfie.c)
There is also **2 time registers**, that just cause delays by decrementing to 0 for each cycle,
wasting an operation in the process (like the *nop* one).
It as additionally an index register which is **16-bit**.
The program counter is also 16-bit.
And of course there is a stack pointer which, as for selfie program, stores the address of the highest stack element.
It has at most 16 elements in it at any given time. It can be implemented in python just by using a list.

## Code implementation

We used [pyglet](http://www.pyglet.org/) module in Python because it can easily output graphics as well as handling
sound output along with keyboard input.

Basically all emulators have a main loop structure like this one: 

    def main(self):
    
        self.initialize()
        self.load_rom(sys.argv[1])
        while not self.has_exit:
            self.dispatch_events()    
            self.cycle()
            self.draw()

So the first step is to initialize our variables.We set all the registers to zero (as to avoid garbage values and allow the program to work properly)
, reset all the key inputs to zero and also the program counter **self.pc**.
We then load a ROM(read-only memory) and perform each cycle and finally update the display as needed
>Loading means copying machine word from memory to one of the 8-bit registers to make available to the CPU to access and process instructions

> Program counter is a dedicated register in the CPU that points to the address of the instruction we're going tp process next in the memory.

So, so far we talked about the code, registers, the CPU but what about fonts as we mentioned them earlier ? 
Basically CHIP8 uses "sprites" for graphics. They are simply a set of bits that indicates a corresponding pixel is on or off.
To learn more about this check 
[CHIP-8 Technical Reference](http://devernay.free.fr/hacks/chip8/C8TECH10.HTM). (They are pretty similar to sprites in Scratch)

The CHIP8 as in the following implementation has sprites in memory for the 16 hexadecimal digits.
And each font character is 8x5 bits, therefore we need 5 bytes of memory to store the font per character.
Our init function will be like this, with font loading included:

    def initialize(self):
        self.clear()
        self.memory = [0]*4096 # max 4096
        self.gpio = [0]*16 # max 16
        self.display_buffer = [0]*64*32 # 64*32
        self.stack = []
        self.key_inputs = [0]*16  
        self.opcode = 0
        self.index = 0
    
    
        self.delay_timer = 0
        self.sound_timer = 0
        self.should_draw = False
        
        self.pc = 0x200
        
        i = 0
        while i < 80:
          # load 80-char font set
          self.memory[i] = self.fonts[i]
          i += 1

The should_draw method is by default False, just to update the display when needed and not everytime.
The self.clear() procedure is only to clear the pyglet window.

All of this program executed, we can now start loading our ROM into memory, It can be read into memory by simply opening
them as a binary(machine code), to allow the CPU to read them: 
    
    def load_rom(self, rom_path):
        log("Loading %s..." % rom_path)
        binary = open(rom_path, "rb").read()
        i = 0
        while i < len(binary):
          self.memory[i+0x200] = ord(binary[i])
          i += 1

The log() is just a log messages function to allow the user to know that the ROM is loading in a user-friendly way.

After all those instructions executed, we can now start processing the program.

<ins>1. The cycle()

Each cycle is made this way: 
    
    def cycle(self): 
        self.opcode = self.memory[self.pc]
    
        #
        # TODO: process this opcode here
        #
    
    
        # After 
        self.pc += 2
      
        # decrement timers
        if self.delay_timer > 0:
          self.delay_timer -= 1
        if self.sound_timer > 0:
          self.sound_timer -= 1
          if self.sound_timer == 0:
            # Play a sound here with pyglet!


We keep talking about opcode but what is opcode ?
> Opcode is just an abbreviation of **Operation Code** aka instruction machine code, instruction code, instruction 
> syllable, instruction parcel or opstring, which is the portion of a machine language instruction that specifies the 
> operation to be executed or performed.

So what that portion of code do is just basically reading the binary, line by line, and performing associated actions 
per line. The time-space complexity of this is very high, therefore slowing the execution, but as the CHIP8 very 
simple, we won't notice it, but for others emulators , this could be tricky and needs other faster approches.

Each opcode is 2-byte long(the same as the registers). The program counter points to the current opcode we want to execute,
then after processing it the program counter is incremented by 2 bytes, to move to the next opcode, until it reaches the end of the program.

The next step now is to implement an if-else block to perform switches on the opcode (as opposed to C where we can freely do create an instruction to switch statements for opcode).
We will use eventually dictionaries to store opcode methods as following: 

    self.vx = (self.opcode & 0x0f00) >> 8
    self.vy = (self.opcode & 0x00f0) >> 4
    self.pc += 2


    # 2. check ops, lookup and execute
    extracted_op = self.opcode & 0xf000
    try:
      self.funcmap[extracted_op]() # call the associated method
    except:
      print "Unknown instruction: %X" % self.opcode

Quick note about CHIP8 opcodes:
they are usually in the format XXXX. So basically we can decipher (aka decrypt) the instruction from its first (leftmost) nibble, so we extract the op with a bitwise AND
, and look it up in our function map (self.funcmap). If it isn't there, we will show that there is an error.

>**Note**
>
> self.pc is incremented BEFORE cycle(), so any modification to it are retained (This basically works like the **jal, jalr and beq**)

<ins>2. Sample Instruction: clear a screen with 00E0 and return with 00EE

Here, we are going to implement the first and simplest instruction: clear the screen.
This corresponds to the opcode0x00E0.
But there is a problem here, in our cycle(), we just used the leftmost nibble,and this might create a conflict with another 
instruction in the format **0x0XXX**: 0x00EE, or the "return from subroutine" opcode. The solution is so simple: we are 
going to an additional extraction: 

    def _0ZZZ(self):
        extracted_op = self.opcode & 0xf0ff
        try:
          self.funcmap[extracted_op]()
        except:
          print "Unknown instruction: %X" % self.opcode

Here, when we encounter an opcode which has 0 as the leftmost nibble, we extract the first and last 2 nibbles and look the associated method in the function map. 0x00E0 should map to CLS, and 0x00EE should map to the return subroutine:

def _0ZZ0(self):
    log("Clears the screen")
    self.display_buffer = [0]*64*32 # 64*32
    self.should_draw = True

To return from a subroutine, we just need to pop the topmost address from our stack:

      def _0ZZE(self):
        log("Returns from subroutine")
        self.pc = self.stack.pop()

And in self.funcmap:

    self.funcmap = {0x0000: self._0ZZZ,
                    0x00e0: self._0ZZ0,
                    0x00ee: self._0ZZE,
    ....

Here, the self._0ZZE will be the "return" method. Why don't we jus set 0x00E0 and 0x00EE to the methods
needed ?
Well, we actually could have, but 0x0xxx are special instructions, as they don't have inputs.
The last three nibbles are the ones which contains inputs to the operation, and they aren't constants, so they can be mapped as well.
For instance, let dig further in the JUMP opcode: 

<ins>3. Sample instruction: JUMPing with 1XXX:

The JUMP opcode works exactly as jal and jarl as we saw with selfie.c
The opcode starts with 1, and the following three nibbles are corresponding to the address of the instruction to be executed next.
Here , we can see why we do the extraction bitwise, as the jump address changes but the opcode identifier doesn't, hence we can identify the instruction.

This is the implementation of jump ( we just are dealing with the program counter):

     def _1ZZZ(self):
        log("Jumps to address NNN.")
        self.pc = self.opcode & 0x0fff

And in our function map: 

    self.funcmap = {0x0000: self._0ZZZ,
                        0x00e0: self._0ZZ0,
                        0x00ee: self._0ZZE,
                        0x1000: self._1ZZZ,
        ....

That's it ! We do the same procedure for all the rest opcodes, and we can proudly say we have now our own CHIP8 emulator.

### Important Opcodes : 

1. **Opcodes using the registers V0 to VF**

In the cycle() implementation, we can notice these two lines: 

    self.vx = (self.opcode & 0x0f00) >> 8
        self.vy = (self.opcode & 0x00f0) >> 4

As in the model we saw with selfie.c, some opcodes are general purpose registers.
For example, 0x4XNN, which "Skips the next instruction if VX doesn't equal NN". To handle this, the above instructions
just extracts the nibbles with bitwise operations to get the associated registers(which are usually stored in the second and third nibble).

The following code is used to perform the skip: 

    def _4ZZZ(self):
        log("Skips the next instruction if VX doesn't equal NN.")
        if self.gpio[self.vx] != (self.opcode & 0x00ff):
          self.pc += 2

The same goes for comparing: 

    def _5ZZZ(self):
        log("Skips the next instruction if VX equals VY.")
        if self.gpio[self.vx] == self.gpio[self.vy]:
          self.pc += 2

2. **Using the carry flag**

Some operations require setting the VF flag, as it is mentioned in the Technical Reference.
For instance, operation 0x8XX4 uses VF as a carry flag in addition. Why we need this ? Remember when we said that the
registers only stores 8-bit values ? If we add 2 8-bit values, there is a possibility of overflow.
So is the result is greater than the maximum, the carry flag is set 

    def _8ZZ4(self):
        log("Adds VY to VX. VF is set to 1 when there's a carry, and to 0 when there isn't.")
        if self.gpio[self.vx] + self.gpio[self.vy] > 0xff:
          self.gpio[0xf] = 1
        else:
          self.gpio[0xf] = 0
        self.gpio[self.vx] += self.gpio[self.vy]
        self.gpio[self.vx] &= 0xff


We can also use this feature in subtraction with a *borrow flag* :


    def _8ZZ5(self):
        log("VY is subtracted from VX. VF is set to 0 when there's a borrow, and 1 when there isn't")
        if self.gpio[self.vy] > self.gpio[self.vx]:
          self.gpio[0xf] = 0
        else:
          self.gpio[0xf] = 1
        self.gpio[self.vx] = self.gpio[self.vx] - self.gpio[self.vy]
        self.gpio[self.vx] &= 0xff

3. **Drawing**

Drawing is pretty straightforward in CHIP8.
It's just a pixel that is on or off, and we store the states in the *display_buffer* we defined earlier.
There are two opcodes that do drawing: **0xDXXX**, which draw sprites loaded from the ROM, and **0xFZ29**, which just 
draws a character.

      def _FZ29(self):
        log("Set index to point to a character")
        self.index = (5*(self.gpio[self.vx])) & 0xfff

This op just sets the index register to the address of the first byte of the corresponding characters (they are just
arrays like a string in c is just *char).This is because the main drawing code, which we will implement, 
uses the register to find the sprite it needs to draw. For further understanding of this, please check the Technical 
Reference. DXYN means according to the Reference: 

`Display n-byte sprite starting at memory location I at (Vx, Vy), set VF = collision.

This applies also to custom sprites, since they are too made out of pixels and bytes.
But here, instead of being a fixed 5 bytes, the N value can correspond to a custom sprite height. Also, the bytes are 
*XOR'ed* onto the screen.

The CHIP8 resources are limited, so we use the actual display to store sprites states. XOR'ing a pixel to the new value 
will tell you if the state changed, and therefore indicate a collision. Here, a collision is flagged in VF: if a pixel
changes from 1 to 0, then VF is set to 1.



Now we are going to implement the buffer update code: 

      def _DZZZ(self):
    log("Draw a sprite")
    self.gpio[0xf] = 0
    x = self.gpio[self.vx] & 0xff
    y = self.gpio[self.vy] & 0xff
    height = self.opcode & 0x000f
    row = 0
    while row < height:
      curr_row = self.memory[row + self.index]
      pixel_offset = 0
      while pixel_offset < 8:
        loc = x + pixel_offset + ((y + row) * 64)
        pixel_offset += 1
        if (y + row) >= 32 or (x + pixel_offset - 1) >= 64:
          # ignore pixels outside the screen
          continue
        mask = 1 << 8-pixel_offset
        curr_pixel = (curr_row & mask) >> (8-pixel_offset)
        self.display_buffer[loc] ^= curr_pixel
        if self.display_buffer[loc] == 0:
          self.gpio[0xf] = 1
        else:
          self.gpio[0xf] = 0
      row += 1
    self.should_draw = True

Basically, we are going to the pixel indicated at *x,y*, then we compare each pixel in our sprite to that in our buffer,
and XOR as needed. We set then our draw flag to True, so draw() function call catches it.

Pyglet is not efficient in terms of time when doing pixel-wise drawing. So to push this to be faster, we will do the 
drawing **pseudo-pixelwise**. The window will be **640x320** and the pixel will be drawn at **x*10** and **y*10**: 

    def draw(self):
    if self.should_draw:
      # draw
      self.clear()
      line_counter = 0
      i = 0
      while i < 2048:
        if self.display_buffer[i] == 1:
          # draw a square pixel
          self.pixel.blit((i%64)*10, 310 - ((i/64)*10))
        i += 1
      self.flip()
      self.should_draw = False

For the sake of simplicity, we will not optimize this for other applications.

4.**Keyboard Input**

Now we can implement the keyboard input methods: 


       def on_key_press(self, symbol, modifiers):
        log("Key pressed: %r" % symbol)
        if symbol in KEY_MAP.keys():
          self.key_inputs[KEY_MAP[symbol]] = 1
          if self.key_wait:
            self.key_wait = False
        else:
          super(cpu, self).on_key_press(symbol, modifiers)
    
      def on_key_release(self, symbol, modifiers):
        log("Key released: %r" % symbol)
        if symbol in KEY_MAP.keys():
          self.key_inputs[KEY_MAP[symbol]] = 0 

Here, we are just updating the key statues as inputted in the keyboard. This is very straightforward.
In the emulator, we can just read the values as necessary. The input will be handled by *0xEXXE and 0xEXX1* : 

    def _EZZE(self):
        log("Skips the next instruction if the key stored in VX is pressed.")
        key = self.gpio[self.vx] & 0xf
        if self.key_inputs[key] == 1:
          self.pc += 2
          
    def _EZZ1(self):
        log("Skips the next instruction if the key stored in VX isn't pressed.")
        key = self.gpio[self.vx] & 0xf
        if self.key_inputs[key] == 0:
          self.pc += 2

And that's everything. This might be incomplete but this is my first semester as a computer science engineer.
Thanks for my teachers **Dr Loubna MEKOUAR**, my Software Development Fundamentals course teacher, and **Mr Christoph KIRSCH**, my Introduction to Computer Science course teacher.
Please feel free to address any issue in this email : **Aya.HOUSSAINI@um6p.ma**.


___

*For reference, please check the [CHIP-8 information page](https://chip-8.github.io/links/)














