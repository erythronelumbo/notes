# Notes for design of a virtual machine

- Type: Stack machine
- Assumes that signed integers are represented as two's complement.
- State:
  - Data/operand stack: 256 64-bit elements.
  - Call/return stack: 256 64-bit elements.
  - Stack pointers (for both stacks): At least 16-bit integers.
  - Program counter: 64 bit integer.
    - Can it be a `size_t`?
  - Pointer to memory
  - As a C struct:
    ```c
    #define STACK_SIZE 256
  
    typedef struct
    {
      // Data/operand and call/return stacks
      uint_least64_t data_stk[STACK_SIZE], call_stk[STACK_SIZE];
  
      // Stack pointers (data and call stacks)
      uint_least32_t ds_ptr, cs_ptr;
  
      // Program counter
      uint_least64_t pc;
  
      // Pointer to memory
      uint_least64_t* mem;
    } vm_state;
    ```


## Opcodes

Notes:
- `-d`: Double word (64 bits)
- `-w`: Word (64 bits)
- `-h`: Half word (64 bits)
- `-b`: Byte (8 bits)
- `-i`: Immediate

Encoding immediates:
```
[opcode][x_byte0]...[x_byte{n-1}]
```
- Can be a packed representation.

To-do:
- Floating-point operations (arithmetic, comparisons, etc.)
  - Also functions like `sin`, `cos` and the like?
- Arithmetic operations with an immediate operand (?)
  - With a *prefix* byte before the opcode
- More arithmetic and bitwise operations (like modular multiplication, modular
  inverse, modular exponentiation, popcount, counting trailing/leading zeros,
  absolute value and sign)?
  - Indicated with a *prefix* byte
- Interaction (I/O) with *external devices*:
  - Standard input and outputs
  - Screen
  - Mouse, keyboard
  - A (pseudo)random number generator?
  - Files?
- Operations on the call stack
- An `eval` opcode?
  - Treats the top of the data stack as an opcode.

Stack notation:
```
+------------------------- Bottom of the stack
|
|                      +-- Top of the stack
|                      |
v                      v
|x{n-1} x{n-2} ... x1 x0]
```

Control:
- `halt`:
  - Stops the program. It has no effect on the data and return stacks.
- `jump`:
  - `|... p]` --> `|...]`
  - Sets the value of the program counter to `p` and jumps to the point in the
    program indicated by this value.
- `jump-i[p]`:
  - Sets the value of the program counter to `p` and jumps to the point in the
    program indicated by this value.
  - It has no effect on the data and return stacks.
- `jz`:
  - `|... x p]` --> `|...]`
  - If `x == 0`, sets the value of the program counter to `p` and jumps to the
    point in the program indicated by this value; otherwise, increments the
    program counter.
- `jz-i[p]`:
  - `|... x]` --> `|...]`
  - Same as `jz`, but `p` is an immediate.
- `jnz`:
  - `|... x p]` --> `|...]`
  - If `x != 0`, sets the value of the program counter to `p` and jumps to the
    point in the program indicated by this value; otherwise, increments the
    program counter.
- `jnz-i[p]`:
  - `|... x]` --> `|...]`
  - Same as `jnz`, but `p` is an immediate.
- `call`:
  - Data stack: `|... p]` --> `|...]`
  - Call stack: `|...]` --> `|... PC]`
  - Pushes the current values of the program counter onto the call stack and
    then jumps to the point in the program given by `p`.
- `call-i[p]`:
  - Same as `call`, but `p` is an immediate.
- `ret`:
  - Call stack: `|... PC]` -> `|...]`
  - Pops the value of the top of the call stack and sets the program counter to
    it.
- `rz`:
  - Data stack: `|... x]` --> `|...]`
  - Call stack:
    - `|... PC]` --> `|...]` if `x == 0`
    - Unchanged if `x != 0`
  - If the top of the data stack is zero, pops the value on the top of the call
    stack and sets the program counter to it; otherwise, the call stack will be
    unchanged.
- `rnz`:
  - Data stack: `|... x]` --> `|...]`
  - Call stack:
    - `|... PC]` --> `|...]` if `x != 0`
    - Unchanged if `x == 0`
  - If the top of the data stack is not zero, pops the value on the top of the
    call stack and sets the program counter to it; otherwise, the call stack
    will be unchanged.

Arithmetic and bitwise integer operations:
- `and-{d/w/h/b}`:
  - `|... x1 x0]` --> `|... (x1 & x0)]`
- `or-{d/w/h/b}`:
  - `|... x1 x0]` --> `|... (x1 | x0)]`
- `xor-{d/w/h/b}`:
  - `|... x1 x0]` --> `|... (x1 ^ x0)]`
- `not-{d/w/h/b}`:
  - `|... x]` --> `|... ~x]`
- `sll-{d/w/h/b}`:
  - `|... x s]` --> `|... (x << s)]`
- `slr-{d/w/h/b}`:
  - `|... x s]` --> `|... (x >> s)]`
  - Shifts *all* the bits (included the *sign* bit if signed).
- `sar-{d/w/h/b}`:
  - `|... x s]` --> `|... (x >> s)]`
  - Shifts all the bits except the sign bit, doing a *sign extension*.
  - A.K.A. *arithmetic shift*.
- `inc-{d/w/h/b}`:
  - `|... x]` --> `|... (x + 1)]`
- `dec-{d/w/h/b}`:
  - `|... x]` --> `|... (x - 1)]`
- `add-{d/w/h/b}`:
  - `|... x1 x0]` --> `|... (x1 + x0)]`
- `sub-{d/w/h/b}`:
  - `|... x1 x0]` --> `|... (x1 - x0)]`
- `neg-{d/w/h/b}`:
  - `|... x]` --> `|... (-x)]`
- `mul-{d/w/h/b}`:
  - `|... x1 x0]` --> `|... (x1 * x0)]`
- `udiv-{d/w/h/b}`:
  - `|... x1 x0]` --> `|... (x1 / x0)]`
  - `x1` and `x0` are taken as unsigned values.
- `sdiv-{d/w/h/b}`:
  - `|... x1 x0]` --> `|... (x1 / x0)]`
  - `x1` and `x0` are taken as signed values.
- `umod-{d/w/h/b}`:
  - `|... x1 x0]` --> `|... (x1 % x0)]`
  - `x1` and `x0` are taken as unsigned values.
- `smod-{d/w/h/b}`:
  - `|... x1 x0]` --> `|... (x1 % x0)]`
  - `x1` and `x0` are taken as signed values.

Integer predicates:
- The result of each one of these operations is `1` when true or `0` when false.
- `isz-{d/w/h/b}`:
  - `|... x]` --> `|... (x == 0)]`
- `isnz-{d/w/h/b}`:
  - `|... x]` --> `|... (x != 0)]`
- `eq-{d/w/h/b}`:
  - `|... x1 x0]` --> `|... (x1 == x0)]`
- `neq-{d/w/h/b}`:
  - `|... x1 x0]` --> `|... (x1 != x0)]`
- `ult-{d/w/h/b}`:
  - `|... x1 x0]` --> `|... (x1 < x0)]`
  - The operands are read as unsigned values.
- `slt-{d/w/h/b}`:
  - `|... x1 x0]` --> `|... (x1 < x0)]`
  - The operands are read as signed values.
- `ugt-{d/w/h/b}`:
  - `|... x1 x0]` --> `|... (x1 > x0)]`
  - The operands are read as unsigned values.
- `sgt-{d/w/h/b}`:
  - `|... x1 x0]` --> `|... (x1 > x0)]`
  - The operands are read as signed values.
- `uleq-{d/w/h/b}`:
  - `|... x1 x0]` --> `|... (x1 <= x0)]`
  - The operands are read as unsigned values.
- `sleq-{d/w/h/b}`:
  - `|... x1 x0]` --> `|... (x1 <= x0)]`
  - The operands are read as signed values.
- `ugeq-{d/w/h/b}`:
  - `|... x1 x0]` --> `|... (x1 >= x0)]`
  - The operands are read as unsigned values.
- `sgeq-{d/w/h/b}`:
  - `|... x1 x0]` --> `|... (x1 >= x0)]`
  - The operands are read as signed values.

Memory:
- `ld-{d/w/h/b}`:
  - `|... addr]` --> `|... memory[addr]]`
- `ldi-{d/w/h/b}[addr]`:
  - `|...]` --> `|... memory[addr]]`
- `st-{d/w/h/b}`:
  - `|... x addr]` --> `|...]`
  - `x` will be stored in `memory[addr]`.
- `sti-{d/w/h/b}[addr]`:
  - `|... x]` --> `|...]`
  - `x` will be stored in `memory[addr]`.

"Conversion" between integers:
- `sxt-bd`:
  - `|... x]` --> `|... sign_extend_8to64bits(x)]`
  - Extends the sign of an 8-bit integer to 64 bits.
- `sxt-bw`:
  - `|... x]` --> `|... sign_extend_8to32bits(x)]`
  - Extends the sign of an 8-bit integer to 32 bits.
- `sxt-bh`:
  - `|... x]` --> `|... sign_extend_8to16bits(x)]`
  - Extends the sign of an 8-bit integer to 16 bits.
- `sxt-hd`:
  - `|... x]` --> `|... sign_extend_16to64bits(x)]`
  - Extends the sign of a 16-bit integer to 64 bits.
- `sxt-hw`:
  - `|... x]` --> `|... sign_extend_16to32bits(x)]`
  - Extends the sign of a 16-bit integer to 32 bits.
- `sxt-wd`:
  - `|... x]` --> `|... sign_extend_32to64bits(x)]`
  - Extends the sign of a 32-bit integer to 64 bits.
- `trc-w`:
  - `|... x]` --> `|... (x & 0xffffffff)]`
  - Truncation to 32 bits.
- `trc-h`:
  - `|... x]` --> `|... (x & 0xffff)]`
  - Truncation to 16 bits.
- `trc-b`:
  - `|... x]` --> `|... (x & 0xff)]`
  - Truncation to 8 bits.

Stack manipulation:
- `lit[x]`:
  - `|...]` --> `|... x]`
  - `x` is an immediate.
- `pop`:
  - `|... x0]` --> `|...]`
- `dpop`:
  - `|... x1 x0]` --> `|...]`
- `pickif`:
  - `|... x1 x0 c]` --> `|... x0]` if `c != 0`
  - `|... x1 x0 c]` --> `|... x1]` if `c == 0`
- `dup-1`:
  - `|... x0]` --> `|... x0 x0]`
- `dup-2`:
  - `|... x1 x0]` --> `|... x1 x0 x1]`
- `dup-3`:
  - `|... x2 x1 x0]` --> `|... x2 x1 x0 x2]`
- `dup-4`:
  - `|... x3 x2 x1 x0]` --> `|... x3 x2 x1 x0 x3]`
- `dup-5`:
  - `|... x4 x3 x2 x1 x0]` --> `|... x4 x3 x2 x1 x0 x4]`
- `dup-6`:
  - `|... x5 x4 x3 x2 x1 x0]` --> `|... x5 x4 x3 x2 x1 x0 x5]`
- `dup-7`:
  - `|... x6 x5 x4 x3 x2 x1 x0]` --> `|... x6 x5 x4 x3 x2 x1 x0 x6]`
- `dup-8`:
  - `|... x7 x6 x5 x4 x3 x2 x1 x0]` --> `|... x7 x6 x5 x4 x3 x2 x1 x0 x7]`
- `dup-t`:
  - `|... x{n} ... x0 n]` --> `|... x{n} ... x0 x{n}]`
- `swap-1`:
  - `|... x1 x0]` --> `|... x0 x1]`
- `swap-2`:
  - `|... x2 x1 x0]` --> `|... x0 x1 x2]`
- `swap-3`:
  - `|... x3 x2 x1 x0]` --> `|... x0 x2 x1 x3]`
- `swap-4`:
  - `|... x4 x3 x2 x1 x0]` --> `|... x0 x3 x2 x1 x4]`
- `swap-5`:
  - `|... x5 x4 x3 x2 x1 x0]` --> `|... x0 x4 x3 x2 x1 x5]`
- `swap-6`:
  - `|... x6 x5 x4 x3 x2 x1 x0]` --> `|... x0 x5 x4 x3 x2 x1 x6]`
- `swap-7`:
  - `|... x7 x6 x5 x4 x3 x2 x1 x0]` --> `|... x0 x6 x5 x4 x3 x2 x1 x7]`
- `swap-8`:
  - `|... x8 x7 x6 x5 x4 x3 x2 x1 x0]` --> `|... x0 x7 x6 x5 x4 x3 x2 x1 x8]`
- `swap-t`:
  - `|... x{n} ... x0 n]` --> `|... x0 ... x{n+1}]`
- `grab-3`:
  - `|... x2 x1 x0]` --> `|... x1 x0 x2]`
- `grab-4`:
  - `|... x3 x2 x1 x0]` --> `|... x2 x1 x0 x3]`
- `grab-5`:
  - `|... x4 x3 x2 x1 x0]` --> `|... x3 x2 x1 x0 x4]`
- `grab-6`:
  - `|... x5 x4 x3 x2 x1 x0]` --> `|... x4 x3 x2 x1 x0 x5]`
- `grab-7`:
  - `|... x6 x5 x4 x3 x2 x1 x0]` --> `|... x5 x4 x3 x2 x1 x0 x6]`
- `grab-8`:
  - `|... x7 x6 x5 x4 x3 x2 x1 x0]` --> `|... x6 x5 x4 x3 x2 x1 x0 x7]`
- `grab-t`:
  - `|... x{n} x{n-1} ... x1 x0 n]` --> `|... x{n-1} ... x1 x0 x{n}]`
- `bury-3`:
  - `|... x2 x1 x0]` --> `|... x0 x2 x1]`
- `bury-4`:
  - `|... x3 x2 x1 x0]` --> `|... x0 x3 x2 x1]`
- `bury-5`:
  - `|... x4 x3 x2 x1 x0]` --> `|... x0 x4 x3 x2 x1]`
- `bury-6`:
  - `|... x5 x4 x3 x2 x1 x0]` --> `|... x0 x5 x4 x3 x2 x1]`
- `bury-7`:
  - `|... x6 x5 x4 x3 x2 x1 x0]` --> `|... x0 x6 x5 x4 x3 x2 x1]`
- `bury-8`:
  - `|... x7 x6 x5 x4 x3 x2 x1 x0]` --> `|... x0 x7 x6 x5 x4 x3 x2 x1]`
- `bury-t`:
  - `|... x{n} x{n-1} ... x1 x0 n]` --> `|... x0 x{n} x{n-1} ... x1]`


Bytecode representation:

- The correspondent bytecode value is `(column + row)`.

```
+---+------+------+------+------+------+------+------+------+
|\\\| +0   | +1   | +2   | +3   | +4   | +5   | +6   | +7   |
+---+------+------+------+------+------+------+------+------+
|  0|halt  |jump  |jump-i|jz    |jz-i  |jnz   |jnz-i |call  |
+---+------+------+------+------+------+------+------+------+
|  8|call-i|ret   |rz    |rnz   |...   |...   |...   |...   |
+---+------+------+------+------+------+------+------+------+
| 16|and-d |or-d  |xor-d |not-d |sll-d |slr-d |sar-d |inc-d |
+---+------+------+------+------+------+------+------+------+
| 24|dec-d |add-d |sub-d |neg-d |mul-d |udiv-d|sdiv-d|umod-d|
+---+------+------+------+------+------+------+------+------+
| 32|smod-d|eq-d  |neq-d |isz-d |isnz-d|ult-d |slt-d |ugt-d |
+---+------+------+------+------+------+------+------+------+
| 40|sgt-d |uleq-d|sleq-d|ugeq-d|sgeq-d|...   |...   |...   |
+---+------+------+------+------+------+------+------+------+
| 48|and-w |or-w  |xor-w |not-w |sll-w |slr-w |sar-w |inc-w |
+---+------+------+------+------+------+------+------+------+
| 56|dec-w |add-w |sub-w |neg-w |mul-w |udiv-w|sdiv-w|umod-w|
+---+------+------+------+------+------+------+------+------+
| 64|smod-w|eq-w  |neq-w |isz-w |isnz-w|ult-w |slt-w |ugt-w |
+---+------+------+------+------+------+------+------+------+
| 72|sgt-w |uleq-w|sleq-w|ugeq-w|sgeq-w|...   |...   |...   |
+---+------+------+------+------+------+------+------+------+
| 80|and-h |or-h  |xor-h |not-h |sll-h |slr-h |sar-h |inc-h |
+---+------+------+------+------+------+------+------+------+
| 88|dec-h |add-h |sub-h |neg-h |mul-h |udiv-h|sdiv-h|umod-h|
+---+------+------+------+------+------+------+------+------+
| 96|smod-h|eq-h  |neq-h |isz-h |isnz-h|ult-h |slt-h |ugt-h |
+---+------+------+------+------+------+------+------+------+
|104|sgt-h |uleq-h|sleq-h|ugeq-h|sgeq-h|...   |...   |...   |
+---+------+------+------+------+------+------+------+------+
|112|and-b |or-b  |xor-b |not-b |sll-b |slr-b |sar-b |inc-b |
+---+------+------+------+------+------+------+------+------+
|120|dec-b |add-b |sub-b |neg-b |mul-b |udiv-b|sdiv-b|umod-b|
+---+------+------+------+------+------+------+------+------+
|128|smod-b|eq-b  |neq-b |isz-b |isnz-b|ult-b |slt-b |ugt-b |
+---+------+------+------+------+------+------+------+------+
|136|sgt-b |uleq-b|sleq-b|ugeq-b|sgeq-b|...   |...   |...   |
+---+------+------+------+------+------+------+------+------+
|144|ld-d  |ld-w  |ld-h  |ld-b  |ldi-d |ldi-w |ldi-h |ldi-b |
+---+------+------+------+------+------+------+------+------+
|152|st-d  |st-w  |st-h  |st-b  |sti-d |sti-w |sti-h |sti-b |
+---+------+------+------+------+------+------+------+------+
|160|sxt-bd|sxt-bw|sxt-bh|sxt-hd|sxt-hw|sxt-wd|trc-w |trc-h |
+---+------+------+------+------+------+------+------+------+
|168|trc-b |...   |...   |...   |...   |...   |...   |...   |
+---+------+------+------+------+------+------+------+------+
|176|dup-1 |dup-2 |dup-3 |dup-4 |dup-5 |dup-6 |dup-7 |dup-8 |
+---+------+------+------+------+------+------+------+------+
|184|lit   |...   |...   |...   |...   |...   |...   |...   |
+---+------+------+------+------+------+------+------+------+
|192|swap-1|swap-2|swap-3|swap-4|swap-5|swap-6|swap-7|swap-8|
+---+------+------+------+------+------+------+------+------+
|200|pop   |...   |...   |...   |...   |...   |...   |...   |
+---+------+------+------+------+------+------+------+------+
|208|grab-3|grab-4|grab-5|grab-6|grab-7|grab-8|dup-t |swap-t|
+---+------+------+------+------+------+------+------+------+
|216|dpop  |...   |...   |...   |...   |...   |...   |...   |
+---+------+------+------+------+------+------+------+------+
|224|bury-3|bury-4|bury-5|bury-6|bury-7|bury-8|grab-t|bury-t|
+---+------+------+------+------+------+------+------+------+
|232|pickif|...   |...   |...   |...   |...   |...   |...   |
+---+------+------+------+------+------+------+------+------+
|240|...   |...   |...   |...   |...   |...   |...   |...   |
+---+------+------+------+------+------+------+------+------+
|248|...   |...   |...   |...   |...   |...   |...   |...   |
+---+------+------+------+------+------+------+------+------+
```

## Assembly syntax

TODO: Complete

- It should make possible the writing of *one-liners*.
- Labels: `@[label-name [<ops>...]]`
- Macros: `#[MACRO-NAME [<ops>...]]`
- "Typed" integer literals:
  - `<integer_repr>_<type>` (example: `-1424553_i32`)
- Comments:
  - Multiple lines: `>: ...content... <:`

*Mock-up* example:

```
>:
Macros - logical operators
|... x1 x0]
|... x1 (x0 != 0)] (isnz-d)
|... (x0 != 0) (x1 != 0)] (swap-1)
|... (x0 != 0) (x1 != 0)] (isnz-d)
|... ((x0 != 0) & (x1 != 0))] (and-b)
<:
#[LOGICAL-AND [isnz-d swap-1 isnz-d and-b]]
#[LOGICAL-OR [isnz-d swap-1 isnz-d or-b]]

>:
Labels - bitwise 64-bit right rotation

|... x r]
|... x r x]
|... x r x r]
|... x r (x>>r)]
|... (x>>r) x r]
|... (x>>r) x (-r)]
|... (x>>r) x (-r) 63]
|... (x>>r) x ((-r)&63)]
|... (x>>r) (x<<((-r)&63))]
|... (x>>r)|(x<<((-r)&63))]
<:
@[
  rotr64
  [dup-2 dup-2 slr-d bury-3 neg-d lit[63] and-d sll-d or-d ret]
]

>:
Labels - Fibonacci
f(0) = 0
f(1) = 1
f(n) = f(n-1)+f(n-2)

|... n]
|... n n] dup-1
|... n n 1] lit[1]
|... n (n<=1)] uleq-d (where (n<=1)==1)
|... n] jnz-i[@fib-end]
|... (n-1)] dec-d
|... (n-1) (n-1)] dup-1
|... (n-1) (n-2)] dec-d
|... (n-1) f(n-2)] call[@fibonacci]
|... f(n-2) (n-1)] swap-1
|... f(n-2) f(n-1)] call[@fibonacci]
|... (f(n-2)+f(n-1))] add-d
|... (f(n-2)+f(n-1))] ret
<:
@[
  fibonacci
  [
    dup-1 lit[1] uleq-d jnz-i[@fibonacci-end]
    dec-d dup-1 call[@fibonacci]
    swap-1 call[@fibonacci] add-d
  ]
]
@[fibonacci-end [ret]]
```
