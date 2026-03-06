# Python Style Guide

Style guide for Python projects. These rules represent conventions that can make code maintainable. Feel free to change it to your linkings. For everything else, follow [PEP 8](https://peps.python.org/pep-0008/).

Code is formatted with **black** and type-checked with **mypy**. This guide covers what automated tools cannot enforce.

---

**1)** Never use catch-all `except:` statements, or catch `Exception`, unless you are
- re-raising the exception, or
- creating an isolation point in the program where exceptions are not propagated but are recorded and suppressed instead, such as protecting a thread from crashing by
	  guarding its outermost block.

```
Yes: try:
         process(value)
     except ValueError:
         handle_value_error()

Yes: try:
         process(value)
     except Exception:
         logger.exception("Unexpected error")
         raise

Yes: try:
         run_worker()
     except Exception:
         logger.exception("Worker crashed")  # Isolation point: log and suppress.

No:  try:
         process(value)
     except:
         pass

No:  try:
         process(value)
     except Exception:
         pass
```

---
**2)** Use default iterators and operators for types that support them, like lists, dictionaries, and files.
The built-in types define iterator methods, too. Prefer these methods to methods that return lists, except that you should not mutate a container while iterating over it.

```
Yes:  for key in adict: ...
      if obj in alist: ...
      for line in afile: ...
      for k, v in adict.items(): ...

No:   for key in adict.keys(): ...
      for line in afile.readlines(): ...
```
---
**3)** Use generators as needed.  Use `Yields:` rather than `Returns:` in the docstring for generator functions.
If the generator manages an expensive resource, make sure to force the clean up.
A good way to do the clean up is by wrapping the generator with a context manager (PEP-0533).

```
def read_large_file(path: str) -> Iterator[str]:
    “””
    Reads a file line by line.

    Yields:
        Each line from the file.
    “””
    with open(path) as f:
        for line in f:
            yield line.strip()
```

When the generator manages an expensive resource, wrap it with a context manager to ensure cleanup:

```
@contextlib.contextmanager
def open_file(path):
    f = open(path, "w")
    try:
        yield f          # this becomes the `as` variable
    finally:
        f.close()

with open_file("test.txt") as f:
    f.write("hello")
```

---
**4)** Do not use mutable objects as default values in the function or method definition.

```
Yes: def foo(a, b=None):
         if b is None:
             b = []
			 
Yes: def foo(a, b: Sequence | None = None):
         if b is None:
             b = []
			 
Yes: def foo(a, b: Sequence = ()):  # Empty tuple OK since tuples are immutable.
         ...


No:  def foo(a, b=[]):
         ...
		 
No:  def foo(a, b=time.time()):  # Is `b` supposed to represent when this module was loaded?
         ...
		 
No:  def foo(a, b={}):
```

---
**5)** Properties are allowed, but, like operator overloading, should only be used when necessary and match the expectations of typical attribute access; 
follow the getters and setters rules otherwise.

For example, using a property to simply both get and set an internal attribute isn’t allowed: there is no computation occurring, so the property is unnecessary 
(make the attribute public instead). 
In comparison, using a property to control attribute access or to calculate a trivially derived value is allowed: the logic is simple and unsurprising.

Properties should be created with the `@property` decorator. Inheritance with properties can be non-obvious.
Do not use properties to implement computations a subclass may ever want to override and extend.

```
Yes: class Circle:
         def __init__(self, radius: float) -> None:
             self.radius = radius  # Simple attribute: just make it public.

         @property
         def area(self) -> float:  # Derived value: property is appropriate.
             return math.pi * self.radius ** 2

No:  class Circle:
         def __init__(self, radius: float) -> None:
             self._radius = radius

         @property
         def radius(self) -> float:  # No computation: unnecessary property.
             return self._radius

         @radius.setter
         def radius(self, value: float) -> None:
             self._radius = value
```

---
**6)** **True/False Evaluations**:

Use the “implicit” false if possible, e.g., `if foo:` rather than `if foo != []:`. There are a few caveats that you should keep in mind though:

- Always use `if foo is None:` (or `is not None`) to check for a None value. E.g., when testing whether a variable or argument that defaults to None was set to some other value. The other value might be a value that’s false in a boolean context!

- Never compare a boolean variable to False using `==`. Use `if not x:` instead. If you need to distinguish False from None then chain the expressions, such as `if not x and x is not None:`.

- For sequences (strings, lists, tuples), use the fact that empty sequences are false, so `if seq:` and `if not seq:` are preferable to `if len(seq):` and `if not len(seq):` respectively.

- When handling integers, implicit false may involve more risk than benefit (i.e., accidentally handling None as 0). You may compare a value which is known to be an integer (and is not the result of len()) against the integer 0.

```
Yes: if not users:
         print('no users')

     if i % 10 == 0:
         self.handle_multiple_of_ten()

     def f(x=None):
         if x is None:
             x = []

No:  if len(users) == 0:
         print('no users')

     if not i % 10:
         self.handle_multiple_of_ten()

     def f(x=None):
         x = x or []
```

- Note that '0' (i.e., 0 as string) evaluates to true.

- Note that Numpy arrays may raise an exception in an implicit boolean context. Prefer the `.size` attribute when testing emptiness of a np.array (e.g. `if not users.size`).
---
**7)** **Decorators**: Use decorators wisely when there is a clear advantage. Decorators should follow the same import and naming guidelines as functions. Decorator pydoc should clearly state that the function is a decorator. 
Write unit tests for decorators. Avoid external dependencies in the decorator itself (e.g. don’t rely on files, sockets, database connections, etc.), 
since they might not be available when the decorator runs (at import time, perhaps from pydoc or other tools). 
A decorator that is called with valid parameters should (as much as possible) be guaranteed to succeed in all cases.

```
import functools
import time

def retry(max_attempts: int = 3):
    """
    Decorator that retries a function on failure.

    Args:
        max_attempts: Maximum number of retry attempts.
    """
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(max_attempts):
                try:
                    return func(*args, **kwargs)
                except Exception:
                    if attempt == max_attempts - 1:
                        raise
                    time.sleep(2 ** attempt)
        return wrapper
    return decorator


@retry(max_attempts=3)
def fetch_data(url: str) -> dict:
    ...
```

---
**8)** **Threading**:

Do not rely on the atomicity of built-in types. While Python’s built-in data types such as dictionaries appear to have atomic operations, there are corner cases where they aren’t atomic 
(e.g. if `__hash__` or `__eq__` are implemented as Python methods) and their atomicity should not be relied upon. Neither should you rely on atomic variable assignment (since this in turn depends on dictionaries).

Use the queue module’s Queue data type as the preferred way to communicate data between threads. Otherwise, use the threading module and its locking primitives. 
Prefer condition variables and `threading.Condition` instead of using lower-level locks.

```
Yes: import queue

     def producer(q: queue.Queue) -> None:
         q.put("data")

     def consumer(q: queue.Queue) -> None:
         item = q.get()
         process(item)

No:  shared_data = []

     def producer() -> None:
         shared_data.append("data")  # Not thread-safe.

     def consumer() -> None:
         item = shared_data.pop()
         process(item)
```

---
**9)** All code must use type hints (`PEP-484`). Annotate all function signatures (parameters and return types). Code is type-checked at build time with `mypy`.
Type annotations should be in the source. Use pyi files only for third-party or extension modules.

```
def capture_frame(device_id: int, timeout: float = 5.0) -> NDArray:
    return camera.get_frame(device_id, timeout)

def load_calibration(path: str) -> dict[str, Any]:
    with open(path) as f:
        return yaml.safe_load(f)
```

Annotating `self` or `cls` is generally not necessary. `Self` can be used if it is necessary for proper type information, e.g.

```
from typing import Self

class BaseClass:
    @classmethod
    def create(cls) -> Self:
        ...

    def difference(self, other: Self) -> float:
        ...
```

If any other variable or a returned type should not be expressed, use `Any`.
Annotate code that is prone to type-related errors (previous bugs or complexity).
Annotate code that is hard to understand.
Annotate code as it becomes stable from a types perspective. In many cases, you can annotate all the functions in mature code without losing too much flexibility.

---
**10)** **Typing Variables**:

If an internal variable has a type that is hard or impossible to infer, specify its type with an annotated assignment - use a colon and type between the variable name and value
(the same as is done with function arguments that have a default value):

```
a: Foo = SomeUndecoratedFunction()
```

*Type Comments*: Though you may see them remaining in the codebase (they were necessary before Python 3.6), do not add any more uses of a `# type: <type name>` comment on the end of the line:

```
a = SomeUndecoratedFunction()  # type: Foo
```

---
**11)** It is fine, though not required, to use parentheses around tuples. Do not use them in return statements or conditional statements unless using parentheses for implied line continuation or to indicate a tuple.

```
Yes: if foo:
         bar()

Yes: return x, y

Yes: return (x,)  # Parentheses indicating a single-element tuple.

Yes: if (long_variable_name
             and another_long_name):  # Parentheses for line continuation.
         bar()

No:  if (foo):
         bar()

No:  return (x, y)
```

---
**12)** Use `4` spaces for indentation. Never use tabs.

When a line is too long and you need to wrap it, either:
- line up the continuation with the opening bracket, or
- use a 4-space hanging indent.

If a closing bracket is on its own line, it should line up with the line that has the opening bracket.

```
Yes: # Arguments are aligned with opening delimiter.
     foo = long_function_name(var_one, var_two,
                              var_three, var_four)

     # Hanging indent with 4 spaces.
     foo = long_function_name(
         var_one, var_two,
         var_three, var_four,
     )

     # Closing bracket matches opening line.
     result = [
         item_one,
         item_two,
         item_three,
     ]

No:  # Arguments not aligned or indented.
     foo = long_function_name(var_one, var_two,
         var_three, var_four)

No:  # Closing bracket not matching opening line.
     result = [
         item_one,
         item_two,
         ]
```

---
**13)** Use docstrings to document all packages, modules, classes, and functions. Always use the three-double-quote `"""` format (per PEP 257).

Both the opening `"""` and closing `"""` should be on their own lines. The docstring text starts on the line after the opening quotes. For multi-line docstrings, the summary line should be followed by a blank line, then the rest of the body.

```
def fetch_frameset(device_id: int, timeout: float = 5.0) -> Frameset:
    """
    Fetches a single frameset from the specified device.

    Waits for the device to produce a complete RGBD frameset within
    the given timeout period.

    Args:
        device_id: The camera device identifier.
        timeout: Maximum seconds to wait for a frame.

    Returns:
        A Frameset containing depth, color, and infrared images.

    Raises:
        TimeoutError: If no frame is received within the timeout.
    """
```

---
**14)** **Classes**:

Classes should have a docstring below the class definition describing the class.
Public attributes, excluding properties, should be documented in an `Args` section in the class docstring. Do not duplicate this in the `__init__` docstring.

```
class CameraConfig:
    """
    Configuration for a depth camera device.

    Holds connection parameters and capture settings used
    during camera initialization.

    Args:
        serial_number: The camera’s unique serial identifier.
        frame_rate: Capture rate in frames per second.
    """

    def __init__(self, serial_number: str, frame_rate: int = 30):
        self.serial_number = serial_number
        self.frame_rate = frame_rate

    @property
    def is_high_speed(self) -> bool:
        """
        Whether the camera is configured for high-speed capture.
        """
        return self.frame_rate >= 60
```


All class docstrings should start with a one-line summary that describes what the class instance represents.
The class docstring should not repeat unnecessary information, such as that the class is a class.

```
Yes:

class StoreLayout:
    """
    The spatial layout of a store's camera coverage area.

    ...
    """

No:

class StoreLayout:
    """
    Class that describes the spatial layout of a store's camera coverage area.

    ...
    """
```

---
**15)** **Block and Inline Comments**:

The final place to have comments is in tricky parts of the code. 
If you’re going to have to explain it at the next code review, you should comment it now. 
Complicated operations get a few lines of comments before the operations commence. Non-obvious ones get comments at the end of the line.

```
# We use a weighted dictionary search to find out where i is in
# the array.  We extrapolate position based on the largest num
# in the array and the array size and then do binary search to
# get the exact number.
if i & (i - 1) == 0:  # True if i is 0 or a power of 2.
```

Never describe the code. Assume the person reading the code knows Python (though not what you’re trying to do) better than you do.

```
# BAD COMMENT: Now go through the b array and make sure whenever i occurs
# the next element is i+1
```

---
**16)** Pay attention to punctuation, spelling, and grammar; it is easier to read well-written comments than badly written ones.

Comments should be as readable as narrative text, with proper capitalization and punctuation. In many cases, complete sentences are more readable than sentence fragments.
Shorter comments, such as comments at the end of a line of code, can sometimes be less formal, but you should be consistent with your style.

---
**17)** Use an f-string, even when the parameters are all strings. Use your best judgment to decide between string formatting options. A single join with `+` is okay but do not format with `+`.

```
Yes: x = f'name: {name}; score: {n}'
     
No: x = first + ', ' + second
    x = 'name: ' + name + '; score: ' + str(n)
```

Avoid using the `+` and `+=` operators to accumulate a string within a loop, as this can lead to quadratic running time.
Instead, collect substrings in a list and `''.join()` after the loop, or write to an `io.StringIO` buffer.

```
Yes:

parts = ['device_id,serial,status']
for device in devices:
    parts.append(f'{device.id},{device.serial},{device.status}')
report = '\n'.join(parts)

No:

report = 'device_id,serial,status'
for device in devices:
    report += f'\n{device.id},{device.serial},{device.status}'
```

Be consistent with your choice of string quote character within a file. Pick ' or " and stick with it. 
It is okay to use the other quote character on a string to avoid the need to backslash-escape quote characters within the string.

Prefer `"""` for multi-line strings rather than `'''`. 


```
No:
long_string = """This is pretty ugly.
Don't do this.
"""

Yes:
long_string = """This is fine if your use case can accept
    extraneous leading spaces."""


Yes:
long_string = ("And this too is fine if you cannot accept\n"
               "extraneous leading spaces.")

Yes:
import textwrap

long_string = textwrap.dedent("""\
    This is also fine, because textwrap.dedent()
    will collapse common leading spaces in each line.""")
```

---
**18)** The preferred way to manage files and similar resources is using the with statement:

```
with open("hello.txt") as hello_file:
    for line in hello_file:
        print(line)
```

For file-like objects that do not support the with statement, use contextlib.closing():

```
import contextlib

with contextlib.closing(open_camera(device_id)) as camera:
    for frame in camera.stream():
        process(frame)
```

In rare cases where context-based resource management is infeasible, code documentation must explain clearly how resource lifetime is managed.

---
**19)** **TODO Comments**:  

Use TODO comments for code that is temporary, a short-term solution, or good-enough but not perfect.

A TODO comment begins with the word TODO in all caps, a following colon, and probably a link to a resource that contains the context, if available. 
Follow this piece of context with an explanatory string introduced with a hyphen -. The purpose is to have a consistent TODO format that can be searched to find out how to get more details.

```
# TODO: https://digital-711.atlassian.net/browse/TICKET-123 - Investigate shared memory optimizations.
```

Avoid adding TODOs that refer to an individual or team as the context:

```
# TODO: @yourusername - File an issue and use a '*' for repetition.
```

If your TODO is of the form “At a future date do something” make sure that you either include a very specific date (“Fix by November 2009”) or 
a very specific event (“Remove this code when all clients can handle XML responses.”) that future code maintainers will comprehend. Issues are ideal for tracking this.

---
**20)** Imports should be on separate lines; there are exceptions for `typing` and `collections.abc` imports.

E.g.:

```
Yes: from collections.abc import Mapping, Sequence
     import os
     import sys
     from typing import Any, NewType

No:  import os, sys
```

Imports are always put at the top of the file, just after any module comments and docstrings and before module globals and constants. Imports should be grouped from most generic to least generic:

Python standard library imports. For example:
```
import sys
```

Third-party module or package imports. For example:

```
import numpy as np
```

Code repository sub-package imports. For example:

```
from rgbd.camera import RealSenseCamera
```

Within each grouping, imports should be sorted lexicographically, ignoring case, according to each module’s full package path (the path in from path import ...). 
Code may optionally place a blank line between import sections.

```
import logging
import pathlib
import sys

import boto3
import cv2
import numpy as np
import redis

from qcomponents.pipes import Pipe
from qcore.camera import CameraManager
from qcore.processors import DepthProcessor
from rgbd.frameset import Frameset
```

---
**21)** **Naming**:

- module_name, 
- package_name, 
- ClassName, 
- method_name, 
- ExceptionName, 
- function_name, 
- GLOBAL_CONSTANT_NAME, 
- global_var_name, 
- instance_var_name, 
- function_parameter_name, 
- local_var_name, 
- query_proper_noun_for_thing, 
- send_acronym_via_https.

Function names, variable names, and filenames should be descriptive; avoid abbreviation. In particular, do not use abbreviations that are ambiguous or unfamiliar to readers outside your project, 
and do not abbreviate by deleting letters within a word.

---
**22)** **Names to Avoid**:

Single character names, except for specifically allowed cases:

- counters or iterators (e.g. `i`, `j`, `k`, `v`, et al.)
- `e` as an exception identifier in try/except statements.
- `f` as a file handle in with statements
- private type variables with no constraints (e.g. `_T = TypeVar("_T")`, `_P = ParamSpec("_P")`)

Please be mindful not to abuse single-character naming. Generally speaking, descriptiveness should be proportional to the name’s scope of visibility. 
For example, `i` might be a fine name for 5-line code block but within multiple nested scopes, it is likely too vague.

Some don'ts:

- dashes (-) in any package/module name

- `__double_leading_and_trailing_underscore__` names (reserved by Python)

- offensive terms

- names that needlessly include the type of the variable (for example: `id_to_name_dict`)

---
**23)** **Naming Conventions**:

Prepending a single underscore ( _ ) has some support for protecting module variables and functions (linters will flag protected member access). 
Note that it is okay for unit tests to access protected constants from the modules under test.

Prepending a double underscore (__ aka “dunder”) to an instance variable or method effectively makes the variable or method private to its class (using name mangling);
we discourage its use as it impacts readability and testability, and isn’t really private. Prefer a single underscore.

Name mangling renames `__attr` to `_ClassName__attr` under the hood, which prevents accidental access:

```
class Camera:
    def __init__(self) -> None:
        self.__serial = “ABC123”

cam = Camera()
cam.__serial       # AttributeError: ‘Camera’ object has no attribute ‘__serial’
cam._Camera__serial  # “ABC123” — still accessible, just harder to reach accidentally.
```

Where name mangling is genuinely useful is avoiding attribute collisions in inheritance hierarchies (for both method and variable):

```
class Base:
    def __init__(self) -> None:
        self.__counter = 0  # Becomes _Base__counter

class Child(Base):
    def __init__(self) -> None:
        super().__init__()
        self.__counter = 100  # Becomes _Child__counter — no collision with Base’s counter.
```

Even so, prefer a single underscore unless you specifically need to avoid name collisions in a class hierarchy.

Place related classes and top-level functions together in a module. Unlike Java, there is no need to limit yourself to one class per module.

Use CapWords for class names, but `lower_with_under.py` for module names. `CapWords.py` module names are discouraged because it’s confusing when the module is named after a class.

New unit test files follow PEP 8 compliant `lower_with_under` method names, for example, `test_<method_under_test>_<state>`.

---
**24)** **Mathematical Notation**:

For mathematically heavy code, short variable names that would otherwise violate the style guide are preferred when they match established notation in a reference paper or algorithm. 
When doing so, reference the source of all naming conventions in a comment or docstring or, if the source is not accessible, clearly document the naming conventions. 
Prefer PEP8-compliant descriptive_names for public APIs, which are much more likely to be encountered out of context.

```
# Internal: short names matching the Kabsch algorithm notation.
# Reference: https://en.wikipedia.org/wiki/Kabsch_algorithm
def _kabsch(P: NDArray, Q: NDArray) -> NDArray:
    H = P.T @ Q
    U, S, Vt = np.linalg.svd(H)
    d = np.sign(np.linalg.det(Vt.T @ U.T))
    R = Vt.T @ np.diag([1, 1, d]) @ U.T
    return R

# Public API: use descriptive names.
def compute_optimal_rotation(
    source_points: NDArray,
    target_points: NDArray,
) -> NDArray:
    """
    Computes the optimal rotation matrix between two point sets.
    """
    return _kabsch(source_points, target_points)
```

---
**25)** **Main**:

In Python, pydoc as well as unit tests require modules to be importable. If a file is meant to be used as an executable, its main functionality should be in a `main()` function, 
and your code should always check `if __name__ == '__main__'` before executing your main program, so that it is not executed when the module is imported.

Use:

```
def main():
    ...

if __name__ == '__main__':
    main()
```

---
**26)** **Function length**:

Prefer small and focused functions. Long functions are sometimes appropriate, so no hard limit on function length. If a function exceeds about 40 lines, 
think about whether it can be broken up without harming the structure of the program.

Even if your long function works perfectly now, someone modifying it in a few months may add new behavior. This could result in bugs that are hard to find. 
Keeping your functions short and simple makes it easier for other people to read and modify your code.

You could find long and complicated functions when working with some code. Do not be intimidated by modifying existing code: if working with such a function proves to be difficult, 
you find that errors are hard to debug, or you want to use a piece of it in several different contexts, consider breaking up the function into smaller and more manageable pieces.

---
**27)** **Forward Declarations**:

If you need to use a class name (from the same module) that is not yet defined, for example, if you need the class name inside the declaration of that class, 
or if you use a class that is defined later in the code, import annotations.

```
Yes:
from __future__ import annotations

class MyClass:
    def __init__(self, stack: list[MyClass], item: OtherClass) -> None:

class OtherClass:
    ...
```

---
**28)** **NoneType**:

In the Python type system, NoneType is a “first class” type, and for typing purposes, None is an alias for NoneType. If an argument can be None, it has to be declared! 
Use `|` union type expressions.

Use explicit `X | None` instead of implicit. Earlier versions of PEP 484 allowed `a: str = None` to be interpreted as `a: str | None = None`, but that is no longer the preferred behavior.

```
Yes:
def modern_or_union(a: str | int | None, b: str | None = None) -> str:
    ...

No:
def union_optional(a: Union[str, int, None], b: Optional[str] = None) -> str:
    ...
def nullable_union(a: Union[None, str]) -> str:
    ...
def implicit_optional(a: str = None) -> str:
    ...
```

---
**29)** **Type Aliases**:

You can declare aliases of complex types. The name of an alias should be CapWorded. If the alias is used only in this module, it should be _Private.

Note that the : TypeAlias annotation is only supported in versions 3.10+.


```
from typing import TypeAlias

_Point: TypeAlias = tuple[float, float]
PointCloud: TypeAlias = list[_Point]
```

From Python 3.12+, prefer the `type` statement instead of `TypeAlias`:

```
type _Point = tuple[float, float]
type PointCloud = list[_Point]
```

---
**30)** **Tuples vs Lists**:

Typed lists can only contain objects of a single type. Typed tuples can either have a single repeated type or a set number of elements with different types. 
The latter is commonly used as the return type from a function.

```
a: list[int] = [1, 2, 3]
b: tuple[int, ...] = (1, 2, 3)
c: tuple[int, str, float] = (1, "2", 3.5)
```


**31)** **String types**:

Use str for string/text data. For code that deals with binary data, use bytes.

```
def deals_with_text_data(x: str) -> str:
    ...
def deals_with_binary_data(x: bytes) -> bytes:
    ...
```

If all the string types of a function are always the same, for example if the return type is the same as the argument type 
and the function can take both str and bytes in the code above, then use `AnyStr`.

