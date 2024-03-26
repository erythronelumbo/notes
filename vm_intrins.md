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
- ~~Can the function signatures be
  `vm_value (*primfunn_sig)(vm_state*, <<inputs>>...)`?~~
  - ~~Should error handling be done with `setjmp` and the like?~~
- The function result can be treated as an error code too.
  - Use a special tag for error codes.
  - Only to be used internally by the VM.

Alternative:
```c
typedef vm_value (*primfun0_sig)();
typedef vm_value (*primfun1_sig)(
  const vm_value*
);
typedef vm_value (*primfun2_sig)(
  const vm_value*, const vm_value*
);
typedef vm_value (*primfun3_sig)(
  const vm_value*, const vm_value*, const vm_value*
);
typedef vm_value (*primfun4_sig)(
  const vm_value*, const vm_value*, const vm_value*, const vm_value*
);
typedef vm_value (*primfun5_sig)(
  const vm_value*, const vm_value*, const vm_value*, const vm_value*,
  const vm_value*
);
```

Pseudo-example primitive:
```c
vm_value add3(
  vm_state *vmst,
  const vm_value *x2, const vm_value *x1, const vm_value *x0
)
{
  if (x2->tag != x1->tag && x1->tag != x0->tag)
  {
    // Probably some setjmp tricks
    VM_PRIMITIVE_ERROR(vmst, func_status_incompatible_types);

    // Should return or just jump?
  }

  vm_value res;
  switch (x2->tag)
  {
    case vm_tagval_ib: res.as.ib = x2->as.ib + x1->as.ib + x0->as.ib; break;
    case vm_tagval_ih: res.as.ih = x2->as.ih + x1->as.ih + x0->as.ih; break;
    case vm_tagval_iw: res.as.iw = x2->as.iw + x1->as.iw + x0->as.iw; break;
    case vm_tagval_id: res.as.id = x2->as.id + x1->as.id + x0->as.id; break;
    case vm_tagval_fw: res.as.fw = x2->as.fw + x1->as.fw + x0->as.fw; break;
    case vm_tagval_fd: res.as.fd = x2->as.fd + x1->as.fd + x0->as.fd; break;
    default:
      VM_PRIMITIVE_ERROR(vmst, func_status_incompatible_types);
  }
  res.tag = x2->tag;
  return res;
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
