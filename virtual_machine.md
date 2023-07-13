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

Integer arithmetic and comparisons, halt:
```
+---+------+------+------+------+------+------+------+------+
|\\\| +0   | +1   | +2   | +3   | +4   | +5   | +6   | +7   |
+---+------+------+------+------+------+------+------+------+
|  0|halt  |and-d |or-d  |xor-d |not-d |sll-d |slr-d |sar-d |
+---+------+------+------+------+------+------+------+------+
|  8|inc-d |dec-d |add-d |sub-d |neg-d |mul-d |udiv-d|sdiv-d|
+---+------+------+------+------+------+------+------+------+
| 16|umod-d|smod-d|eq-d  |neq-d |isz-d |isnz-d|ult-d |slt-d |
+---+------+------+------+------+------+------+------+------+
| 24|ugt-d |sgt-d |uleq-d|sleq-d|ugeq-d|sgeq-d|...   |...   |
+---+------+------+------+------+------+------+------+------+
| 32|...   |and-w |or-w  |xor-w |not-w |sll-w |slr-w |sar-w |
+---+------+------+------+------+------+------+------+------+
| 40|inc-w |dec-w |add-w |sub-w |neg-w |mul-w |udiv-w|sdiv-w|
+---+------+------+------+------+------+------+------+------+
| 48|umod-w|smod-w|eq-w  |neq-w |isz-w |isnz-w|ult-w |slt-w |
+---+------+------+------+------+------+------+------+------+
| 56|ugt-w |sgt-w |uleq-w|sleq-w|ugeq-w|sgeq-w|...   |...   |
+---+------+------+------+------+------+------+------+------+
| 64|...   |and-h |or-h  |xor-h |not-h |sll-h |slr-h |sar-h |
+---+------+------+------+------+------+------+------+------+
| 72|inc-h |dec-h |add-h |sub-h |neg-h |mul-h |udiv-h|sdiv-h|
+---+------+------+------+------+------+------+------+------+
| 80|umod-h|smod-h|eq-h  |neq-h |isz-h |isnz-h|ult-h |slt-h |
+---+------+------+------+------+------+------+------+------+
| 88|ugt-h |sgt-h |uleq-h|sleq-h|ugeq-h|sgeq-h|...   |...   |
+---+------+------+------+------+------+------+------+------+
| 96|...   |and-b |or-b  |xor-b |not-b |sll-b |slr-b |sar-b |
+---+------+------+------+------+------+------+------+------+
|104|inc-b |dec-b |add-b |sub-b |neg-b |mul-b |udiv-b|sdiv-b|
+---+------+------+------+------+------+------+------+------+
|112|umod-b|smod-b|eq-b  |neq-b |isz-b |isnz-b|ult-b |slt-b |
+---+------+------+------+------+------+------+------+------+
|120|ugt-b |sgt-b |uleq-b|sleq-b|ugeq-b|sgeq-b|...   |...   |
+---+------+------+------+------+------+------+------+------+
```

Memory:
```
+---+------+------+------+------+------+------+------+------+
|\\\| +0   | +1   | +2   | +3   | +4   | +5   | +6   | +7   |
+---+------+------+------+------+------+------+------+------+
|128|ld-d  |ld-w  |ld-h  |ld-b  |ldi-d |ldi-w |ldi-h |ldi-b |
+---+------+------+------+------+------+------+------+------+
|136|st-d  |st-w  |st-h  |st-b  |sti-d |sti-w |sti-h |sti-b |
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
|144|sxt-bd|sxt-bw|sxt-bh|sxt-hd|sxt-hw|sxt-wd|trc-w |trc-h |
+---+------+------+------+------+------+------+------+------+
|152|trc-b |...   |...   |...   |...   |...   |...   |...   |
+---+------+------+------+------+------+------+------+------+
```

Stack manipulation:
```
+---+------+------+------+------+------+------+------+------+
|\\\| +0   | +1   | +2   | +3   | +4   | +5   | +6   | +7   |
+---+------+------+------+------+------+------+------+------+
|160|dup-1 |dup-2 |dup-3 |dup-4 |dup-5 |dup-6 |dup-7 |dup-8 |
+---+------+------+------+------+------+------+------+------+
|168|lit   |...   |...   |...   |...   |...   |...   |...   |
+---+------+------+------+------+------+------+------+------+
|176|swap-1|swap-2|swap-3|swap-4|swap-5|swap-6|swap-7|swap-8|
+---+------+------+------+------+------+------+------+------+
|184|pop   |...   |...   |...   |...   |...   |...   |...   |
+---+------+------+------+------+------+------+------+------+
|192|grab-3|grab-4|grab-5|grab-6|grab-7|grab-8|dup-t |swap-t|
+---+------+------+------+------+------+------+------+------+
|200|dpop  |...   |...   |...   |...   |...   |...   |...   |
+---+------+------+------+------+------+------+------+------+
|208|bury-3|bury-4|bury-5|bury-6|bury-7|bury-8|grab-t|bury-t|
+---+------+------+------+------+------+------+------+------+
|216|pickif|...   |...   |...   |...   |...   |...   |...   |
+---+------+------+------+------+------+------+------+------+
```

- `{dup/swap/grab/bury}-t` operate according to the value in the top of the
  data stack.
- `lit` pushes an immediate value

Control:
```
+---+------+------+------+------+------+------+------+------+
|\\\| +0   | +1   | +2   | +3   | +4   | +5   | +6   | +7   |
+---+------+------+------+------+------+------+------+------+
|224|jump  |jump-i|jz    |jz-i  |jnz   |jnz-i |call  |call-i|
+---+------+------+------+------+------+------+------+------+
|232|ret   |...   |...   |...   |...   |...   |...   |...   |
+---+------+------+------+------+------+------+------+------+
|240|...   |...   |...   |...   |...   |...   |...   |...   |
+---+------+------+------+------+------+------+------+------+
|248|...   |...   |...   |...   |...   |...   |...   |...   |
+---+------+------+------+------+------+------+------+------+
```

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
