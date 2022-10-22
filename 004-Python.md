# Python

[TOC]

## 0 inbox

- the difference between a method and a function

| method                                                      | function                           |
| ----------------------------------------------------------- | ---------------------------------- |
| are associated with the objects of the class they belong to | are not associated with any object |
| is called 'on' an object                                    | can invoke one just by its name    |

- [dictionary](https://docs.python.org/3.9/tutorial/datastructures.html#dictionaries)

  - indexed by key, which can be any **immutable** type
  - a pair of braces creates a empty dictionary: `{}`
  - placing a comma-separated list of key:value pairs within the braces adds initial pairs to dictionary 
  - store using a key that is already in use, the old value associated with that key is forgotten.
  - how to loop through a dictionary
    - `for key in dict`
    - `for value in dict.values()`
    - `for key, value in dict.items()`

- `object.__call__(self [,args])`

  > Called when the instance is "called" as a function; if this method is defined, `x(arg1, arg2, ...)` roughly translates to `type(x).__call__(x, arg1, ...)`.
  
- `sum()`: sum over a list

  

## 1 build-in Types

## 1.1 List

- If you use the **assignment operator(=)** to copy a list, you name a new variable that refers to the original list.
