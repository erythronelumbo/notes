# Notes for design of a virtual machine

- Type: Stack machine
- Assumes that signed integers are represented as two's complement.
- Also assumes IEEE 754 floating point types (32 and 64 bits).
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
- `-w`: Word (32 bits)
- `-h`: Half word (16 bits)
- `-b`: Byte (8 bits)
- `-i`: Immediate

Encoding immediates:
```
[opcode][x_byte0]...[x_byte{n-1}]
```
- Can be a packed representation.

To-do:
- Stack manipulation for the call stack (?)
  - *Prefix* opcode + stack manipulation opcode
    - Example: `csmanip dup-1`
- Floating-point operations (arithmetic, comparisons, etc.)
  - `f{add,sub,mul,div,neg,eq,neq,gt,lt,geq,neq}-{w,d}` (22 instrs.)
  - Also functions like `sin`, `cos` and the like?
- Arithmetic operations with an immediate operand (?)
  - With a *prefix* byte before the opcode
- More arithmetic and bitwise operations (like modular multiplication, modular
  inverse, modular exponentiation, popcount, counting trailing/leading zeros,
  absolute value and sign)?
  - Indicated with a *prefix* byte
- Interaction (I/O) with *external devices*:
  - Standard input and outputs, screen, mouse, keyboard, a (pseudo)random number
    generator (?), etc.
  - ~~Can it be done via memory-mapped I/O?~~ Port-mapped I/O
- Operations on the call stack
- An `eval` family of opcodes?
  - Treats the top of the data stack as an opcode.
  - Possible usages:
    - `lit[2] lit[3] lit[0x19] eval[1]` (same as `lit[2] lit[3] add-d`)
    - `lit[4] lit[2] lit[3] lit[0x19] lit[0x1c] eval[2]` (same as
      `lit[4] lit[2] lit[3] add-d mul-d`)
    - `lit[4] lit[2] lit[3] lit[0x19] lit[0x1c] lit[2] eval-t` (same as
      `... eval[2]`)
- Discard the operations for 8-bit and 16-bit integers?

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
- `exit`:
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
- `pc`:
  - `|...]` --> `|... PC]`
  - `PC` is the program counter.
- `pop-1`:
  - `|... x0]` --> `|...]`
- `pop-2`:
  - `|... x1 x0]` --> `|...]`
- `pop-4`:
  - `|... x3 x2 x1 x0]` --> `|...]`
- `pop-8`:
  - `|... x7 x6 x5 x4 x3 x2 x1 x0]` --> `|...]`
- `pickif`:
  - `|... x1 x0 c]` --> `|... x0]` if `c != 0`
  - `|... x1 x0 c]` --> `|... x1]` if `c == 0`
- `dup[i]`:
  - `|... x{i} ... x0]` --> `|... x{i} ... x0 x{i}]`
- `dup-t`:
  - `|... x{i} ... x0 i]` --> `|... x{i} ... x0 x{i}]`
- `swap[i]`:
  - `|... x{i} ... x0]` --> `|... x0 ... x{i+1}]`
- `swap-t`:
  - `|... x{i} ... x0 i]` --> `|... x0 ... x{i+1}]`
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
- `ddup-1`:
  - `|... x1 x0]` --> `|... x1 x0 x1 x0]`
- `ddup-2`:
  - `|... x3 x2 x1 x0]` --> `|... x3 x2 x1 x0 x3 x2]`
- `ddup-3`:
  - `|... x5 x4 x3 x2 x1 x0]` --> `|... x5 x4 x3 x2 x1 x0 x5 x4]`
- `ddup-4`:
  - `|... x7 x6 x5 x4 x3 x2 x1 x0]` --> `|... x7 x6 x5 x4 x3 x2 x1 x0 x7 x6]`
- `dswap-1`:
  - `|... x3 x2 x1 x0]` --> `|... x1 x0 x3 x2]`
- `dswap-2`:
  - `|... x5 x4 x3 x2 x1 x0]` --> `|... x1 x0 x3 x2 x5 x4]`
- `dswap-3`:
  - `|... x7 x6 x5 x4 x3 x2 x1 x0]` --> `|... x1 x0 x5 x4 x3 x2 x7 x6]`
- `dswap-4`:
  - `|... x9 x8 x7 x6 x5 x4 x3 x2 x1 x0]` -->
    `|... x1 x0 x7 x6 x5 x4 x3 x2 x9 x8]`
- `dgrab-3`:
  - `|... x5 x4 x3 x2 x1 x0]` --> `|... x3 x2 x1 x0 x5 x4]`
- `dgrab-4`:
  - `|... x7 x6 x5 x4 x3 x2 x1 x0]` --> `|... x5 x4 x3 x2 x1 x0 x7 x6]`
- `dbury-3`:
  - `|... x5 x4 x3 x2 x1 x0]` --> `|... x1 x0 x5 x4 x3 x2]`
- `dbury-4`:
  - `|... x7 x6 x5 x4 x3 x2 x1 x0]` --> `|... x1 x0 x7 x6 x5 x4 x3 x2]`


Bytecode table:

- The correspondent bytecode value is `(8*row + column)`.

```
+--------+--------+--------+--------+--------+--------+--------+--------+
|exit    |jump    |jump-i  |jz      |jz-i    |jnz     |jnz-i   |call    |
|call-i  |ret     |rz      |rnz     |...     |...     |...     |...     |
+--------+--------+--------+--------+--------+--------+--------+--------+
+--------+--------+--------+--------+--------+--------+--------+--------+
|ld-d    |ld-w    |ld-h    |ld-b    |st-d    |st-w    |st-h    |st-b    |
|sti-d[p]|sti-w[p]|sti-h[p]|sti-b[p]|ldi-d[p]|ldi-w[p]|ldi-h[p]|ldi-b[p]|
+--------+--------+--------+--------+--------+--------+--------+--------+
+--------+--------+--------+--------+--------+--------+--------+--------+
|sxt-bh  |sxt-bw  |sxt-bd  |sxt-hw  |sxt-hd  |sxt-wd  |...     |...     |
|trc-b   |trc-h   |trc-w   |...     |...     |...     |...     |...     |
+--------+--------+--------+--------+--------+--------+--------+--------+
+--------+--------+--------+--------+--------+--------+--------+--------+
|...     |...     |...     |...     |...     |...     |...     |...     |
|...     |...     |...     |...     |...     |...     |...     |...     |
+--------+--------+--------+--------+--------+--------+--------+--------+
+--------+--------+--------+--------+--------+--------+--------+--------+
|dup[i]  |swap[i] |grab[n] |bury[n] |ddup[i] |dswap[j]|dgrab[m]|dbury[m]|
|dup-t   |swap-t  |grab-t  |bury-t  |ddup-t  |dswap-t |dgrab-t |dbury-t |
+--------+--------+--------+--------+--------+--------+--------+--------+
|pop[i]  |pick[n] |lit[x]  |pc      |ctcs    |dtoc    |ctod    |...     |
|...     |...     |...     |...     |...     |...     |...     |...     |
+--------+--------+--------+--------+--------+--------+--------+--------+
+--------+--------+--------+--------+--------+--------+--------+--------+
|fadd-d  |fsub-d  |fneg-d  |fmul-d  |fdiv-d  |feq-d   |fneq-d  |flt-d   |
|fgt-d   |fleq-d  |fgeq-d  |fconv-d |ffun-d  |...     |...     |...     |
+--------+--------+--------+--------+--------+--------+--------+--------+
|fadd-d  |fsub-d  |fneg-d  |fmul-d  |fdiv-d  |feq-d   |fneq-d  |flt-d   |
|fgt-d   |fleq-d  |fgeq-d  |fconv-d |ffun-d  |...     |...     |...     |
+--------+--------+--------+--------+--------+--------+--------+--------+
+--------+--------+--------+--------+--------+--------+--------+--------+
|and-d   |or-d    |xor-d   |not-d   |sll-d   |slr-d   |sar-d   |inc-d   |
|dec-d   |add-d   |sub-d   |neg-d   |mul-d   |udiv-d  |sdiv-d  |umod-d  |
+--------+--------+--------+--------+--------+--------+--------+--------+
|smod-d  |eq-d    |neq-d   |isz-d   |isnz-d  |ult-d   |slt-d   |ugt-d   |
|sgt-d   |uleq-d  |sleq-d  |ugeq-d  |sgeq-d  |ifun-d  |...     |...     |
+--------+--------+--------+--------+--------+--------+--------+--------+
|and-w   |or-w    |xor-w   |not-w   |sll-w   |slr-w   |sar-w   |inc-w   |
|dec-w   |add-w   |sub-w   |neg-w   |mul-w   |udiv-w  |sdiv-w  |umod-w  |
+--------+--------+--------+--------+--------+--------+--------+--------+
|smod-w  |eq-w    |neq-w   |isz-w   |isnz-w  |ult-w   |slt-w   |ugt-w   |
|sgt-w   |uleq-w  |sleq-w  |ugeq-w  |sgeq-w  |ifun-w  |...     |...     |
+--------+--------+--------+--------+--------+--------+--------+--------+
|and-h   |or-h    |xor-h   |not-h   |sll-h   |slr-h   |sar-h   |inc-h   |
|dec-h   |add-h   |sub-h   |neg-h   |mul-h   |udiv-h  |sdiv-h  |umod-h  |
+--------+--------+--------+--------+--------+--------+--------+--------+
|smod-h  |eq-h    |neq-h   |isz-h   |isnz-h  |ult-h   |slt-h   |ugt-h   |
|sgt-h   |uleq-h  |sleq-h  |ugeq-h  |sgeq-h  |ifun-h  |...     |...     |
+--------+--------+--------+--------+--------+--------+--------+--------+
|and-b   |or-b    |xor-b   |not-b   |sll-b   |slr-b   |sar-b   |inc-b   |
|dec-b   |add-b   |sub-b   |neg-b   |mul-b   |udiv-b  |sdiv-b  |umod-b  |
+--------+--------+--------+--------+--------+--------+--------+--------+
|smod-b  |eq-b    |neq-b   |isz-b   |isnz-b  |ult-b   |slt-b   |ugt-b   |
|sgt-b   |uleq-b  |sleq-b  |ugeq-b  |sgeq-b  |ifun-b  |...     |...     |
+--------+--------+--------+--------+--------+--------+--------+--------+
```

`ifun-{d/w/h/b}` and `{fops/fcmp/fcnv/ffun}-{d/w}` are *prefixes*:
```
+-----------------------------------------+
|              ifun-{d/w/h/b}             |
+------+----+----+----+----+----+----+----+ (pcnt: counts the set bits [a.k.a.
|Suffix|abs |sign|rotl|rotr|pcnt|ctz |clz | "population count"]; ctz: counts the
+------+----+----+----+----+----+----+----+ trailing zeros; clz: counts the
|Hex   |0x00|0x01|0x02|0x03|0x04|0x05|0x06| leading zeros)
+------+----+----+----+----+----+----+----+

+------------+ +------------+ +-----------------+ +------------+
| fops-{d/w} | | fmcp-{d/w} | |   fcnv-{d/w}    | | ffun-{d/w} |
+-------+----+ +-------+----+ +------------+----+ +-------+----+
|Suffix |Hex | |Suffix |Hex | |Suffix      |Hex | |Suffix |Hex |
+-------+----+ +-------+----+ +------------+----+ +-------+----+
|add    |0x00| |eq     |0x00| |to-id       |0x00| |abs    |0x00|
+-------+----+ +-------+----+ +------------+----+ +-------+----+
|sub    |0x01| |neq    |0x01| |to-iw       |0x01| |sign   |0x01|
+-------+----+ +-------+----+ +------------+----+ +-------+----+
|mul    |0x02| |lt     |0x02| |to-ih       |0x02| |mod    |0x02|
+-------+----+ +-------+----+ +------------+----+ +-------+----+
|div    |0x03| |gt     |0x03| |to-ib       |0x03| |floor  |0x03|
+-------+----+ +-------+----+ +------------+----+ +-------+----+
|neg    |0x04| |leq    |0x04| |to-f{w/d}   |0x04| |round  |0x04|
+-------+----+ +-------+----+ +------------+----+ +-------+----+
|recip  |0x05| |geq    |0x05|      ^              |ceil   |0x05|
+-------+----+ +-------+----+      |              +-------+----+
                                   |              |sqrt   |0x06|
                                   |              +-------+----+
               `to-fw` if preceded by `fcnv-d`    |exp    |0x07|
               `to-fd` "     "     "  `fcnv-w`    +-------+----+
                                                  |log    |0x08|
                                                  +-------+----+
                                                  |sin    |0x09|
                                                  +-------+----+
                                                  |cos    |0x0a|
                                                  +-------+----+
                                                  |tan    |0x0b|
                                                  +-------+----+
                                                  |asin   |0x0c|
                                                  +-------+----+
                                                  |acos   |0x0d|
                                                  +-------+----+
                                                  |atan   |0x0e|
                                                  +-------+----+
                                                  |sinh   |0x0f|
                                                  +-------+----+
                                                  |cosh   |0x10|
                                                  +-------+----+
                                                  |tanh   |0x11|
                                                  +-------+----+
                                                  |asinh  |0x12|
                                                  +-------+----+
                                                  |acosh  |0x13|
                                                  +-------+----+
                                                  |atanh  |0x14|
                                                  +-------+----+

Organization:
- Control:             [ 0| 0| 0| 0|x3|x2|x1|x0] (        16) (0x00~0x0f)
- Memory operations:   [ 0| 0| 0| 1|x3|x2|x1|x0] (        16) (0x10~0x1f)
- Integer conversions: [ 0| 0| 1| 0|x3|x2|x1|x0] (        16) (0x20~0x2f)
- ...:                 [ 0| 0| 1| 1| ?| ?| ?| ?] (        16) (0x30~0x3f)
- Stack manipulation:  [ 0| 1| 0|x4|x3|x2|x1|x0] (        32) (0x40~0x5f)
- Floating point ops.: [ 0| 1| 1|t0|o3|o2|o1|o0] (2*23 =  26) (0x60~0x7f)
- Integral arithmetic: [ 1|t1|t0|o4|o3|o2|o1|o0] (4*30 = 120) (0x80~0xff)
```


## Assembly syntax

TODO: Complete

- It should make possible the writing of *one-liners*.
- Labels: `@[label-name [<ops>...]]`
  - Alternative: `@[label1] <ops1>... @[label2] <ops2>...`
- Macros: `#[MACRO-NAME [<ops>...]]`
- "Typed" integer literals:
  - `<integer_repr>_<type>` (example: `-1424553_i32`, `NaN_f64`)
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
@[rotr64] dup-2 dup-2 slr-d bury-3 neg-d lit[63] and-d sll-d or-d ret

>:
Labels - Fibonacci
f(0) = 0
f(1) = 1
f(n) = f(n-1)+f(n-2)

|... n]
|... n n] dup-1
|... n n 1] lit[1]
|... n (n<=1)] uleq-d (where (n<=1)==1)
|... n] rnz
|... (n-1)] dec-d
|... (n-1) (n-1)] dup-1
|... (n-1) (n-2)] dec-d
|... (n-1) f(n-2)] call[@fibonacci]
|... f(n-2) (n-1)] swap-1
|... f(n-2) f(n-1)] call[@fibonacci]
|... (f(n-2)+f(n-1))] add-d
|... (f(n-2)+f(n-1))] ret
<:
@[fibonacci]
  dup-1 lit[1] uleq-d rnz
  dec-d dup-1 call[@fibonacci]
  swap-1 call[@fibonacci] add-d
```
