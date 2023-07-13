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

Control:
```
+---+------+------+------+------+------+------+------+------+
|\\\| +0   | +1   | +2   | +3   | +4   | +5   | +6   | +7   |
+---+------+------+------+------+------+------+------+------+
|  0|halt  |jump  |jump-i|jz    |jz-i  |jnz   |jnz-i |call  |
+---+------+------+------+------+------+------+------+------+
|  8|call-i|ret   |...   |...   |...   |...   |...   |...   |
+---+------+------+------+------+------+------+------+------+
```

Integer arithmetic and comparisons, halt:
```
+---+------+------+------+------+------+------+------+------+
|\\\| +0   | +1   | +2   | +3   | +4   | +5   | +6   | +7   |
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
```

Memory:
```
+---+------+------+------+------+------+------+------+------+
|\\\| +0   | +1   | +2   | +3   | +4   | +5   | +6   | +7   |
+---+------+------+------+------+------+------+------+------+
|144|ld-d  |ld-w  |ld-h  |ld-b  |ldi-d |ldi-w |ldi-h |ldi-b |
+---+------+------+------+------+------+------+------+------+
|152|st-d  |st-w  |st-h  |st-b  |sti-d |sti-w |sti-h |sti-b |
+---+------+------+------+------+------+------+------+------+
```

- `ld-*` and `st-*` take the adresses from the top of the stack.
- `ldi-*` and `sti-*` use an immediate value for the address.
  - 32 or 64 bits? A packed integer encoding?
    - `[<opcode>][addr3][addr2][addr1][addr0]`
  - Relative or absolute?

Sign extension (`sxt`) and truncation (`trc`) (between 64-, 32-, 16- and 8- bit
integers):
```
+---+------+------+------+------+------+------+------+------+
|\\\| +0   | +1   | +2   | +3   | +4   | +5   | +6   | +7   |
+---+------+------+------+------+------+------+------+------+
|160|sxt-bd|sxt-bw|sxt-bh|sxt-hd|sxt-hw|sxt-wd|trc-w |trc-h |
+---+------+------+------+------+------+------+------+------+
|168|trc-b |...   |...   |...   |...   |...   |...   |...   |
+---+------+------+------+------+------+------+------+------+
```

Stack manipulation:
```
+---+------+------+------+------+------+------+------+------+
|\\\| +0   | +1   | +2   | +3   | +4   | +5   | +6   | +7   |
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
```

- `{dup/swap/grab/bury}-t` operate according to the value in the top of the
  data stack.
- `lit` pushes an immediate value


Notes:
- `-d`: Double word (64 bits)
- `-w`: Word (64 bits)
- `-h`: Half word (64 bits)
- `-b`: Byte (8 bits)
- `-i`: Immediate

To-do:
- Floating-point operations (arithmetic, comparisons, etc.)
  - Also functions like `sin`, `cos` and the like?
- Interaction (I/O) with *external devices*:
  - Standard input and outputs
  - Screen
  - Mouse, keyboard
  - A (pseudo)random number generator?
  - Files?
- Operations on the call stack
- An `eval` opcode?
  - Treats the top of the data stack as an opcode

### Stack effects

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
