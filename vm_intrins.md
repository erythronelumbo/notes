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
  vm_value *in_n_minus_1, ..., vm_value *in_2, vm_value *in_1, vm_value *in_0
)
{
  {
    if (!TYPE_TAGS_OK(in_n_minus_1, ..., in_2, in_1, in_0))
      return func_status_incompatible_types;
  }
  {
    // the function's code...
    *out = SOME_FUNCTION(*in_n_minus_1, ..., *in_2, *in_1, *in_0);
  }
  return func_status_ok;
}
```

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
