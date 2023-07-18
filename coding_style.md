# Coding style - C and C++

- Width: 80 characters
- Indentation: 2 spaces

## Naming

- Use `lowercase_with_underscore` for variables, functions, structs, classes
  and other types.
- Use `UPPERCASE_WITH_UNDERSCORE` for macros.
- (C++) For template arguments and concepts, use `PascalCase`.
- Macro names MUST have a maximum length of 64 characters.
  - For include guard definitions, the respective macro names MUST have a
    maximum length of 70 characters.
- Other names MUST have a maximum length of 32 characters.
  - This also applies to source file names, without counting the file extension.

## Indentation

- Use Allman style for code blocks:
  ```c
  // some statement
  {
    // code...
  }
  ```
  - For *if-else* statements:
    ```c
    if (predicate1)
    {
      // code...
    }
    else if (predicate2)
    {
      // code...
    }
    else
    {
      // code...
    }
    ```
- For functions:
  ```c
  my_function(argument1, arg2);
  my_function(
    arg1, arg2, arg3,
    arg4
  );
  ```
- For function-like macros:
  ```c
  MY_MACRO(arg1, arg2, arg3, arg4, arg5, arg6)
  MY_MACRO( \
    arg1, arg2, \
    arg3, arg4, \
    fifth_argument, \
    sixth_argument \
  )
  ```
- For arrays:
  ```c
  /*statements...*/ {item1, item2, item3, item4, item5, item6};
  /*statements...*/ {
    item1, item2, item3, item4,
    item5, item6
  };
  /*statements...*/ {
    item1,
    item2,
    item3,
    item4,
    item5,
    item6
  };
  ```
  - Multidimensional arrays:
    ```c
    // Example with 2D arrays
    /*statements...*/ {{x00, x01, x02}, {x10, x11, x12}, {x20, x21, x22}};
    /*statements...*/ {
      {x00, x01, x02}, {x10, x11, x12}, {x20, x21, x22},
      {x30, x31, x32}, {x40, x41, x42}
    };
    /*statements...*/ {
      {x00, x01, x02},
      {x10, x11, x12},
      {x20, x21, x22}
    };
    /*statements...*/ {
      {
        x00, x01, x02
      },
      {
        x10, x11, x12
      },
      {
        x20, x21, x22
      }
    };
    /*statements...*/ {
      {
        x00,
        x01,
        x02
      },
      {
        x10,
        x11,
        x12
      },
      {
        x20,
        x21,
        x22
      }
    };
    ```
- (C++) For templates:
  ```c++
  templated<param1, param2, param3, param4, param5, param6>;
  templated<
    param1,
    param2,
    param3,
    param4, param5, param6,
    param7, param8
  >;
  ```

### Function declaration and definition

Here are some ways to indent the declaration and definition of functions:

```c
/*qualifiers...*/ result_type function(type1 arg1, type2 arg2, type3 arg3)
{
  // code...
}
```

```c
/*qualifiers...*/ result_type function1(
  type1 arg1, type2 arg2, type3 arg3, type4 arg4
)
{
  // code...
}

/*qualifiers...*/ result_type function2(
  type1 arg1, type2 arg2,
  type3 arg3,
  type4 arg4
)
{
  // code...
}
```

```c
/*qualifiers...*/ result_type
function1(type1 arg1, type2 arg2, type3 arg3, type4 arg4)
{
  // code...
}

/*qualifiers...*/
result_type
function1(type1 arg1, type2 arg2, type3 arg3, type4 arg4)
{
  // code...
}
```

### Macros

```c
#define MY_MACRO_1 /* macro definition... */

#define MY_MACRO_2 \
/* macro definition - first line */ \
/* macro definition - second line */ \
/* macro definition - third line */
```

```c
#define FUNCTION_LIKE_MACRO_1(arg1, arg2, arg3, arg4) /* macro definition... */

#define FUNCTION_LIKE_MACRO_2(arg1, arg2, arg3, arg4) \
/* macro definition - first line */ \
/* macro definition - second line */
```

```c
#define FUNCTION_LIKE_MACRO_1( \
  arg1, arg2, arg3, arg4 \
) \
/* macro definition - first line */ \
/* macro definition - second line */

#define FUNCTION_LIKE_MACRO_2( \
  first_argument, second_argument, third_argument, \
  fourth_argument, fifth_argument \
) \
/* macro definition - first line */ \
/* macro definition - second line */
```

### (C++) Templates

```cpp
template <TemplateInput1 X1, TemplateInput2 X2, TemplateInput3 X3>
// templated declaration/definition to follow
```

```cpp
template <
  TemplateInput1 X1, TemplateInput2 X2, TemplateInput3 X3,
  TemplateInput4 X4
>
// templated declaration/definition to follow
```

`TemplateInput<n>` can be `typename`, `class`, a template template or any type
(i.e. integral or pointer) that can be used in such context.
