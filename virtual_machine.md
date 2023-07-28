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
    #define STACK_CAPACITY 256
  
    typedef struct vm_state
    {
      // Data/operand and call/return stacks
      uint_least64_t data_stk[STACK_CAPACITY], call_stk[STACK_CAPACITY];
  
      // Stack pointers (data and call stacks)
      uint_least32_t ds_ptr, cs_ptr;
  
      // Program counter
      uint_least64_t pc;
  
      // Pointer to memory
      uint_least64_t* mem;

      // Devices
      void (*devices_fn)(struct vm_state*, uint_least8_t);
    } vm_state;
    ```
  - Would be a good idea to also use *tags* to indicate if the values of the
    data stack are *plain* values, addresses to labels/"subroutines" and
    undefined values, among others?
    - Implementation: in a separate array (i.e. in a struct-of-arrays fashion).
    - Examples:
      - Picking one of three "subroutines" and applying them to two values:
        ```
        Data stack: |... 0x02  0x03  0x143a 0x143b 0x1448 0x01 ]
        Tags:       |... VALUE VALUE ADDR   ADDR   ADDR   VALUE]
                    (ADDR: address of label/subroutine)
        ```
      - Undefined value after a division by zero or some other bad operation:
        ```
        Data stack: |... 0x00     ]
        Tags:       |... UNDEFINED]
        ```
      - *Sends* a value to the device `0x10`:
        ```
        Data stack: |... 0x01020304 0x10]
        Tags:       |... VALUE      PORT]
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
  - *Prefix* opcode + stack manipulation opcode or *begin-end* opcodes
    - Possible usages:
      - `cstkm dig-3 cstkm dup-1`
      - `cstkmb dig-3 dup-1 cstkme`
- Arithmetic operations with an immediate operand (?)
  - With a *prefix* byte before the opcode
- Extra arithmetic and bitwise operations (like modular multiplication, modular
  inverse, modular exponentiation, popcount, counting trailing/leading zeros,
  absolute value and sign)
  - `ctz` (count trailing zeros) can be used for binary GCD, for example
  - Indicated with a *prefix* opcode (for example: `ifun-w[abs]`)
- Interaction (I/O) with *external devices*:
  - Standard input and outputs, screen, mouse, keyboard, a (pseudo)random number
    generator (?), etc.
  - Also command line arguments?
    - Similar to `argc` and `argv` from C's `main` function.
  - Uses port-mapped I/O
- Operations on the call stack
- An `eval` family of opcodes?
  - Treats the top of the data stack as an opcode.
  - Possible usages:
    - `lit[2] lit[3] lit[0x19] eval[1]` (same as `lit[2] lit[3] add-d`)
    - `lit[4] lit[2] lit[3] lit[0x19] lit[0x1c] eval[2]` (same as
      `lit[4] lit[2] lit[3] add-d mul-d`)
    - `lit[4] lit[2] lit[3] lit[0x19] lit[0x1c] lit[2] eval-x` (same as
      `... eval[2]`)

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
- `extcall`:
  - `|... out{m-1} ... out{1} out{o} port]` --> `|... in{n-1} ... in{1} in{o}]`
  - Reads from and/or writes to an external *device* indicated by `port`. The
    number of inputs (`n`) and outputs (`m`) depends on the device.

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
- `pick[n]`:
  - `|... x{n-1} ... x1 x0 i]` --> `|... x{i}]`
  - `1 <= i <= 255`
- `dup[i]`:
  - `|... x{i} ... x0]` --> `|... x{i} ... x0 x{i}]`
- `dup-x`:
  - `|... x{i} ... x0 i]` --> `|... x{i} ... x0 x{i}]`
  - `0 <= i <= 255`
- `swap[i]`:
  - `|... x{i} ... x0]` --> `|... x0 ... x{i+1}]`
- `swap-x`:
  - `|... x{i} ... x0 i]` --> `|... x0 ... x{i+1}]`
  - `0 <= i <= 254`
- `dig-3`:
  - `|... x2 x1 x0]` --> `|... x1 x0 x2]`
- `dig-4`:
  - `|... x3 x2 x1 x0]` --> `|... x2 x1 x0 x3]`
- `dig-5`:
  - `|... x4 x3 x2 x1 x0]` --> `|... x3 x2 x1 x0 x4]`
- `dig-6`:
  - `|... x5 x4 x3 x2 x1 x0]` --> `|... x4 x3 x2 x1 x0 x5]`
- `dig-7`:
  - `|... x6 x5 x4 x3 x2 x1 x0]` --> `|... x5 x4 x3 x2 x1 x0 x6]`
- `dig-8`:
  - `|... x7 x6 x5 x4 x3 x2 x1 x0]` --> `|... x6 x5 x4 x3 x2 x1 x0 x7]`
- `dig-x`:
  - `|... x{n} x{n-1} ... x1 x0 n]` --> `|... x{n-1} ... x1 x0 x{n}]`
  - `3 <= n <= 255`
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
- `bury-x`:
  - `|... x{n} x{n-1} ... x1 x0 n]` --> `|... x0 x{n} x{n-1} ... x1]`
  - `3 <= n <= 255`
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
- `ddig-3`:
  - `|... x5 x4 x3 x2 x1 x0]` --> `|... x3 x2 x1 x0 x5 x4]`
- `ddig-4`:
  - `|... x7 x6 x5 x4 x3 x2 x1 x0]` --> `|... x5 x4 x3 x2 x1 x0 x7 x6]`
- `dbury-3`:
  - `|... x5 x4 x3 x2 x1 x0]` --> `|... x1 x0 x5 x4 x3 x2]`
- `dbury-4`:
  - `|... x7 x6 x5 x4 x3 x2 x1 x0]` --> `|... x1 x0 x7 x6 x5 x4 x3 x2]`


Bytecode table:

- The correspondent bytecode value is `(row + column)`.

```
+----++--------+--------+--------+--------+--------+--------+--------+--------+
|    ||  0x00  |  0x01  |  0x02  |  0x03  |  0x04  |  0x05  |  0x06  |  0x07  |
+----++--------+--------+--------+--------+--------+--------+--------+--------+
|0x00||exit    |jump    |jump-i  |jz      |jz-i    |jnz     |jnz-i   |call    |
|0x08||call-i  |ret     |rz      |rnz     |extcall |...     |...     |...     |
+----++--------+--------+--------+--------+--------+--------+--------+--------+
|0x10||ld-d    |ld-w    |ld-h    |ld-b    |st-d    |st-w    |st-h    |st-b    |
|0x18||ldi-d[p]|ldi-w[p]|ldi-h[p]|ldi-b[p]|sti-d[p]|sti-w[p]|sti-h[p]|sti-b[p]|
+----++--------+--------+--------+--------+--------+--------+--------+--------+
|0x20||sxt-b-h |sxt-b-w |sxt-b-d |sxt-h-w |sxt-h-d |sxt-w-d |...     |...     |
|0x28||trc-b   |trc-h   |trc-w   |...     |...     |...     |...     |...     |
+----++--------+--------+--------+--------+--------+--------+--------+--------+
|0x30||...     |...     |...     |...     |...     |...     |...     |...     |
|0x38||...     |...     |...     |...     |...     |...     |...     |...     |
+----++--------+--------+--------+--------+--------+--------+--------+--------+
|0x40||dup-1   |dup-2   |dup-3   |dup-4   |dup-5   |dup-6   |dup-7   |dup-8   |
|0x48||swap-1  |swap-2  |swap-3  |swap-4  |swap-5  |swap-6  |swap-7  |swap-8  |
+----++--------+--------+--------+--------+--------+--------+--------+--------+
|0x50||dig-3   |dig-4   |dig-5   |dig-6   |dig-7   |dig-8   |dup-x   |swap-x  |
|0x58||bury-3  |bury-4  |bury-5  |bury-6  |bury-7  |bury-8  |dig-x   |bury-x  |
+----++--------+--------+--------+--------+--------+--------+--------+--------+
|0x60||ddup-1  |ddup-2  |ddup-3  |ddup-4  |ddig-3  |ddig-4  |ddup-x  |dswap-x |
|0x68||dswap-1 |dswap-2 |dswap-3 |dswap-4 |dbury-3 |dbury-4 |ddig-x  |dbury-x |
+----++--------+--------+--------+--------+--------+--------+--------+--------+
|0x70||pop-1   |pop-2   |pop-4   |pop-8   |pick[n] |lit[x]  |pc      |...     |
|0x78||ctcs    |dtoc    |ctod    |...     |...     |...     |...     |...     |
+----++--------+--------+--------+--------+--------+--------+--------+--------+
|0x80||and-d   |or-d    |xor-d   |not-d   |sll-d   |slr-d   |sar-d   |inc-d   |
|0x88||dec-d   |add-d   |sub-d   |neg-d   |mul-d   |udiv-d  |sdiv-d  |umod-d  |
+----++--------+--------+--------+--------+--------+--------+--------+--------+
|0x90||smod-d  |eq-d    |neq-d   |isz-d   |isnz-d  |ult-d   |slt-d   |ugt-d   |
|0x98||sgt-d   |uleq-d  |sleq-d  |ugeq-d  |sgeq-d  |ifun-d  |...     |...     |
+----++--------+--------+--------+--------+--------+--------+--------+--------+
|0xa0||and-w   |or-w    |xor-w   |not-w   |sll-w   |slr-w   |sar-w   |inc-w   |
|0xa8||dec-w   |add-w   |sub-w   |neg-w   |mul-w   |udiv-w  |sdiv-w  |umod-w  |
+----++--------+--------+--------+--------+--------+--------+--------+--------+
|0xb0||smod-w  |eq-w    |neq-w   |isz-w   |isnz-w  |ult-w   |slt-w   |ugt-w   |
|0xb8||sgt-w   |uleq-w  |sleq-w  |ugeq-w  |sgeq-w  |ifun-w  |...     |...     |
+----++--------+--------+--------+--------+--------+--------+--------+--------+
|0xc0||fadd-d  |fsub-d  |fneg-d  |fmul-d  |fdiv-d  |feq-d   |fneq-d  |flt-d   |
|0xc8||fgt-d   |fleq-d  |fgeq-d  |ffun-d  |...     |...     |...     |...     |
+----++--------+--------+--------+--------+--------+--------+--------+--------+
|0xd0||fadd-w  |fsub-w  |fneg-w  |fmul-w  |fdiv-w  |feq-w   |fneq-w  |flt-w   |
|0xd8||fgt-w   |fleq-w  |fgeq-w  |ffun-w  |...     |...     |...     |...     |
+----++--------+--------+--------+--------+--------+--------+--------+--------+
|0xe0||fic-d-d |fic-d-w |fic-w-d |fic-w-w |ifc-d-d |ifc-d-w |ifc-w-d |ifc-w-w |
|0xe8||ffc-d-w |ffc-w-d |...     |...     |...     |...     |...     |...     |
+----++--------+--------+--------+--------+--------+--------+--------+--------+
|0xf0||...     |...     |...     |...     |...     |...     |...     |...     |
|0xf8||...     |...     |...     |...     |...     |...     |...     |...     |
+----++--------+--------+--------+--------+--------+--------+--------+--------+
```


`ifun-{d/w/h/b}` and `{fops/fcmp/fcnv/ffun}-{d/w}` are *prefixes*:
```
+------------------------------------------+
|              ifun-{d/w/h/b}              |
+------++----+----+----+----+----+----+----+ (pcnt: counts the set bits [a.k.a.
|Suffix||abs |sign|rotl|rotr|pcnt|ctz |clz | "population count"]; ctz: counts
+------++----+----+----+----+----+----+----+ the trailing zeros; clz: counts the
|Hex   ||0x00|0x01|0x02|0x03|0x04|0x05|0x06| leading zeros)
+------++----+----+----+----+----+----+----+

+------------------------------------------------------+
|                      ffun-{d/w}                      |
+-------+----++-------+----++-------+----++-------+----+
|Suffix |Hex ||Suffix |Hex ||Suffix |Hex ||Suffix |Hex |
+-------+----++-------+----++-------+----++-------+----+
|abs    |0x00||sign   |0x01||mod    |0x02||floor  |0x03|
+-------+----++-------+----++-------+----++-------+----+
|round  |0x04||ceil   |0x05||sqrt   |0x06||exp    |0x07|
+-------+----++-------+----++-------+----++-------+----+
|log    |0x08||sin    |0x09||cos    |0x0a||tan    |0x0b|
+-------+----++-------+----++-------+----++-------+----+
|asin   |0x0c||acos   |0x0d||atan   |0x0e||sinh   |0x0f|
+-------+----++-------+----++-------+----++-------+----+
|cosh   |0x10||tanh   |0x11||asinh  |0x12||acosh  |0x13|
+-------+----++-------+----++-------+----++-------+----+
|atanh  |0x14||       |    ||       |    ||       |    |
+-------+----++-------+----++-------+----++-------+----+

Organization:
- Control:                    [ 0| 0| 0| 0|x3|x2|x1|x0] (0x00~0x0f)
- Memory operations:          [ 0| 0| 0| 1|x3|x2|x1|x0] (0x10~0x1f)
- Integer conversions:        [ 0| 0| 1| 0|x3|x2|x1|x0] (0x20~0x2f)
- Stack manipulation:         [ 0| 1|o5|o4|o3|o2|o1|o0] (0x40~0x7f)
- Integral operations:        [ 1| 0|tp|o4|o3|o2|o1|o0] (0x80~0xbf)
- Floating-point operations:  [ 1| 1| 0|o4|o3|o2|o1|o0] (0xc0~0xdf)
- Floating-point conversions: [ 1| 1| 1| 0|o3|o2|o1|o0] (0xe0~0xef)
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
- Directives:
  - `.<dir-name>(<args>...)`
  - Example: `.str["Hello, world!]" .table-h[0b1001011010100101 0xabcd -12345]`

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
