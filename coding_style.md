# Coding style - C and C++

- Width: 80 characters
- Indentation: 2 spaces

## Naming

- Use `lowercase_with_underscore` for variables, functions, structs, classes
  and other types.
- Use `UPPERCASE_WITH_UNDERSCORE` for macros.
- (C++) For template arguments and concepts, use `PascalCase`.
- Macro names MUST have a maximum length of 64 characters.
- Other names MUST have a maximum length of 32 characters.
  - This also applies to source file names, without counting the file extension.

## Indentation

- Use Allman style for code blocks:
  ```
  // some statement
  {
    // code...
  }
  ```
- For functions:
  ```
  my_function(argument1, arg2);
  my_function(
    arg1, arg2, arg3,
    arg4
  );
  ```
