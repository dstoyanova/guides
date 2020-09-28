# Development 

## Style guides

Python is the main language used at Scaleout. This style guide is a list of __dos__ and __don'ts__ for Python programs.

### Python - Language rules


#### 1. Imports

Use ```import``` statements for packages and modules only, not for individual classes or functions to keep the namespace management convention simple. The source of each identifier is indicated in a consistent way: ```x.Obj``` says that object ```Obj``` is defined in module ```x```.

- Use ```import x``` for importing packages and modules.
- Use ```from x import y``` where ```x``` is the package prefix and ```y``` is the module name with no prefix.
- Use ```from x import y as z``` if two modules named ```y``` are to be imported or if ```y``` is an inconveniently long name.
- Use ```import y as z``` only when ```z``` is a standard abbreviation (e.g., np for numpy).

Example:
```python
from sound.effects import echo

echo.EchoFilter(input, output, delay=0.7, atten=4)
```


#### 2. Packages

Import each module using the full pathname location of the module.

This will help to avoid conflicts in module names or incorrect imports due to the module search path not being what the author expected. Last but not least, it makes it easier to find modules.

Yes:
```python
import absl import flags
from doctor.who import jodie

FLAGS = flags.FLAGS
```

No:
```python
import absl import flags
import jodie

FLAGS = flags.FLAGS
```


#### 3. Exceptions

Exceptions are allowed but must be used carefully.

- Raise exceptions like this: ```raise MyError('Error message')``` or ```raise MyError()```. Do not use the two-argument form (```raise MyError, 'Error message'```).
- Make use of built-in exception classes when it makes sense. For example, raise a ```ValueError``` to indicate a programming mistake like a violated precondition (such as if you were passed a negative number but required a positive one).
- Libraries or packages may define their own exceptions. When doing so they must inherit from an existing exception class. Exception names should end in ```Error```.
- Avoid using catch-all ```except:``` statements, or catch ```Exception``` or ```StandardError```.
- Minimize the amount of code in a ```try/except``` block. The larger the body of the ```try```, the more likely that an exception will be raised by a line of code that you didn’t expect to raise an exception. In those cases, the ```try/except``` block hides a real error.
- Use the ```finally``` clause to execute code whether or not an exception is raised in the ```try``` block. This is often useful for cleanup, i.e., closing a file.


#### 4. Global variables

Avoid global variables.

Even though they're occasionally useful, they also have the potential to change module behavior during the import, because assignments to global variables are done when the module is first imported.

If needed, globals should be declared at the module level and made internal to the module by prepending an ```_``` to the name. External access must be done through public module-level functions


#### 5. Nested/Local/Inner Classes and Functions

Nested local functions or classes are fine when used to close over a local variable. Inner classes are fine.

Avoid nested functions or classes except when closing over a local value. Do not nest a function just to hide it from users of a module. Instead, prefix its name with an ```_``` at the module level so that it can still be accessed by tests.


#### 6. Comprehensions & Generator Expressions

List, Dict, and Set comprehensions as well as generator expressions provide a concise and efficient way to create container types and iterators without resorting to the use of traditional loops, ```map()```, ```filter()```, or ```lambda```.

They are good to use for simple cases. Each portion must fit on one line: mapping expression, ```for``` clause, filter expression. Multiple ```for``` clauses or filter expressions are not permitted. Use loops instead when things get more complicated since complicated comprehensions or generator expressions can be hard to read.

```lambda``` functions are good to use for one-liners. If the code inside the ```lambda``` function is longer than 60-80 chars, it’s probably better to define it as a regular nested function.

For common operations like multiplication, use the functions from the operator module instead of ```lambda``` functions. For example, prefer ```operator.mul``` to ```lambda x, y: x * y```.


#### 7. Conditional Expressions

Conditional expressions (sometimes called a “ternary operator”) are mechanisms that provide a shorter syntax for if statements. For example: ```x = 1 if cond else 2```.

Okay to use for simple cases. Each portion must fit on one line: true-expression, if-expression, else-expression. Use a complete if statement when things get more complicated.


#### 8. Properties

Use properties for accessing or setting data where you would normally have used simple, lightweight accessor or setter methods.

Readability is increased by eliminating explicit get and set method calls for simple attribute access. Allows calculations to be lazy. Considered the Pythonic way to maintain the interface of a class. In terms of performance, allowing properties bypasses needing trivial accessor methods when a direct variable access is reasonable. This also allows accessor methods to be added in the future without breaking the interface.

Properties should be created with the ```@property``` decorator.

Example:
```python
 import math

 class Square(object):
     """A square with two properties: a writable area and a read-only perimeter.

     To use:
     >>> sq = Square(3)
     >>> sq.area
     9
     >>> sq.perimeter
     12
     >>> sq.area = 16
     >>> sq.side
     4
     >>> sq.perimeter
     16
     """

     def __init__(self, side):
         self.side = side

     @property
     def area(self):
         """Area of the square."""
         return self._get_area()

     @area.setter
     def area(self, area):
         return self._set_area(area)

     def _get_area(self):
         """Indirect accessor to calculate the 'area' property."""
         return self.side ** 2

     def _set_area(self, area):
         """Indirect setter to set the 'area' property."""
         self.side = math.sqrt(area)

     @property
     def perimeter(self):
         return self.side * 4
```


#### 9. True/False Evaluations

Python evaluates certain values as ```False``` when in a boolean context. A quick “rule of thumb” is that all “empty” values are considered false, so ```0, None, [], {}, ''``` all evaluate as false in a boolean context.

- Always use ```if foo is None:``` (or ```is not None```) to check for a ```None``` value-e.g., when testing whether a variable or argument that defaults to `None` was set to some other value. The other value might be a value that’s false in a boolean context!
- Never compare a boolean variable to ```False``` using ```==```. Use ```if not x:``` instead. If you need to distinguish ```False``` from ```None``` then chain the expressions, such as ```if not x and x is not None:```.
- For sequences (strings, lists, tuples), use the fact that empty sequences are false, so ```if seq:``` and ```if not seq:``` are preferable to if ```len(seq):``` and ```if not len(seq):``` respectively.
- When handling integers, implicit false may involve more risk than benefit (i.e., accidentally handling ```None``` as 0). You may compare a value which is known to be an integer (and is not the result of ```len()```) against the integer 0.
- Note that ```'0'``` evaluates to true.

### Python - Style rules

One of the recommended IDEs when working with Python is PyCharm. It includes functionality for formatting your Python code. If you prefer to use something else, you can always use [yapf](https://github.com/google/yapf/) auto-formatter to format nicely your Python files.

Keep the code clean from unnecessary comments. However, make sure you write comments if you feel like the next developer, who is going to read your code, needs an explanation. 

Use ```TODO``` comments for code that is temporary, a short-term solution, or good-enough but not perfect.
```python
# TODO(morgan@scaleoutsystems.com): Use a "*" here for string repetition.
# TODO(Mattias) Change this to use relations.
```

Naming conventions:

| type | public | internal | 
| --- | --- | --- |
| packages | `lower_with_under` |  |
| modules | `lower_with_under` | `_lower_with_under` |
| classes | `CapWords` | `_CapWords` |
| exceptions | `CapWords` |  |
| functions | `lower_with_under()` | `_lower_with_under()` |
| global/class constants | `CAPS_WITH_UNDER` | `_CAPS_WITH_UNDER` |
| global/class variables | `lower_with_under` | `_lower_with_under` |
| instance variables | `lower_with_under` | `_lower_with_under` |
| method names | `lower_with_under()` | `_lower_with_under()` |
| function/method parameters | `lower_with_under` |  |
| local variables | `lower_with_under` |  |

## Source Control

### Branching strategy

**GitFlow** branching model is very well suited to collaboration and scaling the development team. The main idea is to make parallel development very easy, by isolating new development from finished work.

![](https://datasift.github.io/gitflow/GitFlowHotfixBranch.png)

- New development (such as features and non-emergency bug fixes) is done in **feature** branches, and is merged back into main body of code when the developer(s) is happy with the code. In case of interuptions, if you are asked to switch to another task, commit your changes and create a new feature branch for the new task. Once you are done, you can simply switch back to the original feature branch where you left off.
- Feature branches are branched off of a **develop** branch, which is a staging area for all completed features that haven't yet been released. As new development is completed, it gets merged back into the develop branch.
- When it is time to make a release, a **release** branch is created off of the develop branch.
- The code in the release branch is deployed onto a suitable test environment, tested, and any problems are fixed directly in the release branch. This deploy -> test -> fix -> re-deploy -> re-test cycle continues until the code is good enough to be released to customers.
- When the release is finished, the **release** branch is merged into **master** and into **develop** too, to make sure that any changes made in the release branch aren’t accidentally lost by new development.
- The master branch tracks released code only. The only commits to master are merges from **release** branches and **hotfix** branches. The hotfix branches are used to create emergency fixes. They are branched directly from a tagged release in the master branch, and when finished are merged back into both **master** and **develop** to make sure that the hotfix isn’t accidentally lost when the next regular release occurs.

### Naming conventions

- master branch - **master**
- develop branch - **develop**
- feature branch - **feature/[JIRA-KEY]**
- bug fix branch - **issue/[JIRA-KEY]**
- hotfix branch - **hotfix/[JIRA-KEY]** 
- release branch - **release/[release version]**

Even though it is recommended to have the ongoing work in JIRA, sometimes a JIRA-KEY is not available. In cases like this, include a very short description of what the branch is intended to solve.

### Peer review

Create **pull requests** when you need to merge code! Try to avoid direct merging of code unless the changes are minor and you are 100% sure that the rest of the code would not be affected (for example, changing a caption string might not need an approval every time). Create a pull request and collect **at least** one approval when:

- merging **feature** or **issue** branch into **develop**
- merging **hotfix** branch into **master**
- merging **release** branch into **master**

### Useful GUIs

- GitHub Desktop - if you are new to git or tired of fighting with it, this GUI simplifies the development workflow.
- Visual Studio Code and PyCharm both have integrated source control GUIs which make it easier to reveiw changes, commit and push them to the origin.

## Testing

### Module tests

### Unit tests

### Integration tests

## Documentation strategy

## Rollout
