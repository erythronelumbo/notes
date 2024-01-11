# Coding style - C and C++

- Width: 80 characters
  - Makes the code easier to read, especially in devices with smaller screens.
  - Useful when working with multiple windows (for example, working with two
    files side-by-side) and when necessary to work with a bigger font size (for
    example, at night, when using a single screen to show the code to other
    people or if/when visually impaired).
  - Long lines of code tend to be hard to read and understand.
    - It is likely to lose the focus on such lines of code when reading them,
      especially when the code is complex.
    - Reading such lines also implies repeated scrolling, which can be a waste
      of time.
    - This width constraint also allows other people (including the programmer
      themselves) to review the code easily.
- Indentation: 2 spaces
  - Useful when some code blocks have to be overly nested (for example, nested
    *for* loops where multidimensional data structures are involved).
  - Consistent across any facility for editing and reading plain text.
    - Tabs can have different widths across different platforms and can vary
      depending of the preference of other individuals or organizations.
  - For consistency, this indentation MUST NOT be mixed with tabs.

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
  - This constraint prevents the programmer from using cumbersome, verbose
    names.

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
  - For *switch* statements:
    ```c
    switch (value)
    {
      case value0:
        // do something
        break;
      case value1:
        {
          // code
        }
        break;
      default:
        // do something
        break;
    }
    ```
    - `case`s can be stacked in the following manners, as long as the 80-column
      boundary is respected:
      - Horizontal:
        ```c
        case VALUE0: case VALUE1: case VALUE 2:
        // do stuff...
        ```
      - Vertical:
        ```c
        case VALUE0:
        case VALUE1:
        case VALUE2:
        // do something
        ```
      - Mixed:
        ```c
        case VALUE0: case VALUE1: case VALUE2:
        case VALUE3: case VALUE4:
        // the remaining code
        ```
  - For *while* loops:
    ```c
    while (condition)
    {
      // code
    }
    ```
  - For *do-while* loops:
    ```c
    do
    {
      // code
    } while (condition);
    ```
  - For *for* loops:
    ```c
    for (/*declarations*/; /*condition*/; /*update*/)
    {
      // code
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
- For long definitions and assignments:
  ```c
  /*qualifiers...*/ /*value_type*/ my_value = (
    /* a really long expression */
    /* another line from the really long expression */
    /* more lines... */
  );
  ```
  - TODO: String literals with multiple lines.

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


## Structs and C++ classes

Struct definition in C:

```c
struct my_data_type
{
  // data members
};

typedef struct my_data_type_1
{
  // data members
} my_data_type_1;
```

Struct and class definition in C++:
```cpp
struct my_data_type
{
  // data members
};

class my_class
{
  public:
    // public members
  protected:
    // protected members
  private:
    // private members
};
```

### (C++) Inheritance:

In a same line:
```cpp
class derived_1 : /*attr*/ base_1, /*attr*/ base_2, /*attr*/ base_3
{
  // code
};
```

In the following lines:
```cpp
class derived_2 :
  /*attr*/ base_1, /*attr*/ base_2, /*attr*/ base_3
{
  // code
};

class derived_3 :
  /*attr*/ base_1,
  /*attr*/ base_2,
  /*attr*/ base_3
{
  // code
};

class derived_4 :
  /*attr*/ base_1, /*attr*/ base_2,
  /*attr*/ base_3
{
  // code
};
```

- Notes:
  - `/*attr*/` refers to any of the usual C++ inheritance attributes.
  - This also applies to `struct`s.

With templates:
```cpp
class my_class_1 : /*attr*/ tmp_class1<p1, p2>, /*attr*/ tmp_class2<p3, p4>
{
  // code...
};

class my_class_2 :
  /*attr*/ tmp_class1<prm1, prm2, ...>, /*attr*/ tmp_class2<prm1, prm2, ...>
{
  // code...
};
```

- If the template parameters for at least one of the base `class`es/`struct`s
  need to be spread in multiple lines, such classes MUST be in separate
  *sections* and **all** the base classes to be inherited from MUST be placed in
  the line next to the class declaration:
  ```cpp
  class my_class_3 :
    /*attr*/ base_1, /*attr*/ base_2, /*attr*/ base_3,
    /*attr*/ base_4,
    /*attr*/ tmp_class1<
      param1, param2, param3, param4,
      param5, param6
    >,
    /*attr*/ base_5, /*attr*/ tmp_base_a<tp1, tp2, tp3>,
    /*attr*/ tmp_base_b<int, float, my_pair<ta, tb>>,
    /*attr*/ tmp_class_2<
      my_type_a, my_type_b, my_type_c, some_parameter,
      some_tmp_thing<ta, tb, tc>, some_tmp_thing<td, te, tf>,
      some_tmp_thing<
        ta, tb, tc
      >,
      t00, t01, t02, t03
    >,
    /*attr*/ tmp_class2<prm1, prm2>, /*attr*/ tmp_class3<prm3, prm4, prm5>,
    /*attr*/ tc<tc<int, 3>, 4>, /*attr*/ something<0, 1, 1, 2, 3, 5, 13, 21>
  {
    // code...
  };
  ```
