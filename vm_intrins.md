# Virtual machine - "Intrinsic" functions

- Assumes that the arguments are *pairs* of the form
  `{value_of_any_type, type_tag}`.

## Interface (temptative)

```c
DEFINE_INTRINSIC( \
  name, (in_type_n_minus_1, ..., in_type_1, in_type_0), output_type, \
  { \
    /* the function's code...*/ \
  } \
)
```

- `in_type_*` can be one of the following:
  - `tag`: Type tag
  - `bn`: Boolean
  - `ib`, `ih`, `iw`, `id`: Integers
  - `ix`: Any integer
  - `fw`, `fd`: Floats
  - `fx`: Any float
  - `nx`: Any numeric
  - `func`: Compiled function
  - `clos`: Closure
  - `strn`: String
  - `list`: Linked list
  - `hmar`: Homogeneous array
  - `htar`: Heterogeneous array
  - `rsig`: Record signature
  - `rval`: Record value
  - `asig`: Algebraic type signature
  - `aval`: Algebraic type value
- Should it have *only* one output?
  - "Forth-style" notation: `(in{n-1} ... in{1} in{0} -- out)`
  - `out = f(in{n-1} ... in{1} in{0})`

## "Translation" to C function:

```c
func_status name(
  vm_value *out,
  const vm_value *in0, const vm_value *in1, const vm_value *in2,
  const vm_value *in3
  /* , maybe some more arguments... */
)
{
  {
    if (!TYPE_TAGS_OK(in0, in1, in2, in3, ...))
      return func_status_incompatible_types;
  }
  {
    // the function's code...
    *out = SOME_FUNCTION(in0, in1, in2, in3, ...);
  }
  return func_status_ok;
}
```

Prototype:
```c
typedef func_status (*primfun0_sig)(
  vm_value*
);
typedef func_status (*primfun1_sig)(
  vm_value*,
  const vm_value*
);
typedef func_status (*primfun2_sig)(
  vm_value*,
  const vm_value*, const vm_value*
);
typedef func_status (*primfun3_sig)(
  vm_value*,
  const vm_value*, const vm_value*, const vm_value*
);
typedef func_status (*primfun4_sig)(
  vm_value*,
  const vm_value*, const vm_value*, const vm_value*, const vm_value*
);
typedef func_status (*primfun5_sig)(
  vm_value*,
  const vm_value*, const vm_value*, const vm_value*, const vm_value*,
  const vm_value*
);
// ...and so on

typedef union func_ptr
{
  primfun0_sig f0;
  primfun1_sig f1;
  primfun2_sig f2;
  primfun3_sig f3;
  primfun4_sig f4;
  primfun5_sig f5;
  // ...and so on
} func_ptr;
```
- This should ensure that **only** certain kinds of functions are accepted.

## Temptative "translation" for VM's execution loop

Assuming a *big switch*:
```c
// ...
case vm_op_name:
  {
    // Checks the type tags
    if (!TYPE_TAGS_OK(in_type_n_minus_1, ..., in_type_1, in_type_0))
      return exec_status_incompatible_types;
  }
  {
    // the function's code...
  }
  {
    // operand stack pointer modification...
  }
  break;
// ...
```
