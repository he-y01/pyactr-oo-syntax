# An Object-Oriented Syntax for (PY)ACT-R

This is a proposal for a syntax that makes it possible to write [pyactr](https://github.com/jakdot/pyactr) productions in an object-oriented way instead of as strings.

## Definitions

In this document, there are a couple of terms used with a very specific meaning:
- rule: any interaction with a buffer (e.g., in LISP ACT-R `g`, `manual`, …) consisting of the type of interaction (e.g., in LISP ACT-R: `+`, `?`, `=`, `~`), the buffer's name and optionally a chunk.
- rule sequence: several rules stringed together
- production: definition of possible cognitive "actions" consisting of the left-hand side (LHS; antecedent) and a right-hand side (RHS; conclusion) each being made up of a single or multiple rules (i.e., a rule sequence)
- production sequence: a sequence of several productions
- chunk: a container of information consisting of a type (marked by the `isa` key, e.g., `isa goal`) and arbitrary properties each consisting of a key-value pair (e.g., in LISP ACT-R `phase start`)

## Motivation

First, let's talk about why to create such a syntax for ACT-R in the first place (this section does not talk about why to use pyactr over LISP ACT-R, but assumes that some reason is already given). Take the following example production in LISP ACT-R:

```lisp
(P start-something
   =goal>
      isa goal
      phase start_something
   ?visual-location>
      state full
   =visual-location>
      isa visual-location
      screen-x =X
==>
   =goal>
      isa goal
      phase finished_something
   =imaginal-item-pos>
      isa pos
      screen-x =X
   +manual>
      isa manual
      cmd press_key
      key W
   ~visual-location>)
```

which would be written as the following string in plain pyactr:

```python
"""
=g>
isa goal
phase start_something
?visual_location>
state full
=visual_location>
isa _visuallocation
screen_x =X
==>
=g>
isa goal
phase finished_something
=imaginal_item_pos>
isa pos
screen_x =X
+manual>
isa _manual
cmd press_key
key W
~visual_location>
"""
```

This has a couple of disadvantages:
- bad readability (e.g., rule headers like `=g>` are somewhat difficult to distinguish from corresponding the chunks)
- spelling mistakes are very likely due to missing checks by IDE and code editors
- comments for & commenting out specific parts is not possible inside a production or rule

Therefore, this project implements classes / objects as wrappers for certain elements of pyactr to allow for the following syntax (more on the notation in the section "How to use it"):

```python
from pyactr_oo_syntax.base.lisplike import e, p, q, t

(
    e.GOAL_(isa='goal',
            phase'start_something') &
    q.VISUAL_LOCATION_(state='full') &
    e.VISUAL_LOCATION_(isa='visual_location',
                       screen_x='=X')
    >>
    e.GOAL_(isa='goal',
            phase='finished_something') &
    e.IMAGINAL_(BUFFER_NAME='item_pos',
                isa='pos',
                screen_x='=X') &
    p.MANUAL_(isa='_manual',
              cmd='press_key',
              key='W') &
    t.VISUAL_LOCATION_()
)
```

Due to the object-based implementation, it opens up the possibilities to define custom rule classes (and chunk classes) which opens up the possibilities to benefit from the following advantages:
- restrict or predefine the properties / slots a certain chunk can have (e.g., a goal chunk `SimpleGoalChunk` should only have a single property with the name `phase`)
- restrict the data type of a property's value with type annotations (e.g., a `move_` production should only except a `dir` of type `MovementDirection` where `MovementDirection`  is a custom `Enum` containing the literals `W`, `A`, `S`, and `D`)
- create more complex productions (and chunks) due to generally easier reusability and inheritance (e.g., `move_` might be a subclass of `press_key_`)

Here a sneak peak of how the example production could look like (note: that several equivalent techniques are mixed to show the possibilities even if in a real application they should be more coherently, e.g., `is_simple_goal_(phase='end')` == `is_simple_goal_(**SimpleGoalChunk(phase='end'))` == `is_simple_goal_.from_chunk(SimpleGoalChunk(phase='end'))`):

```python
# Loading Predefined Syntax
from pyactr_oo_syntax.helpers.actr_data_types import BufferStatus
from pyactr_oo_syntax.base.chunk import AdvChunk
from pyactr_oo_syntax.base.lisplike import e, p, q, t
from pyactr_oo_syntax.convenience.rules import is_simple_goal_, press_key_

# Custom Extensions
class MovementDirections(enum.Enum): ## helper
    up = 'W'
    left = 'A'
    down = 'S'
    right = 'D'

class move_(press_key_): # custom rule
    def __init__(self, dir=MovementDirection):
        super().__init__(Buffer.MANUAL, key=dir.value)

# Definition of Chunk and rule_ objects
PredefinedChunk = AdvChunk(isa='visual_location', screen_x='=X')

production = (
    is_simple_goal_(phase='start')
    q.VISUAL_LOCATION_(state=BufferStatus.FULL) &
    e.VISUAL_LOCATION_(**PredefinedChunk)
    >>
    is_simple_goal_.from_chunk(SimpleGoalChunk(phase='start')) &
    e.IMAGINAL_(BUFFER_NAME=IMAGINALS.ITEM_POS.value,
                isa='pos',
                screen_x='=X') &
    move_(dir=MovementDirections.up) &
    t.VISUAL_LOCATION_()
)
```

where `str(production)` is equal to the string representation of plain pyactr above.


## How to use it

### Requirements & File Structure

This project is only a wrapper-like extension of Jakub Dotlačil's [pyactr](https://github.com/jakdot/pyactr) and therefore requires it to be installed.

For now, this project is not available as a python package. Please download the `pyactr_oo_syntax` project folder and place it into your own project.

This project is structured as follows:

```txt
★ = Class, ☆ = Enum

pyactr_oo_syntax/
├── base/
│   ├── chunk.py
│   │   └── ★ AdvChunk
│   ├── lisplike.py
│   │   ├── ★ p (ACT-R: +)
│   │   ├── ★ e (ACT-R: =)
│   │   ├── ★ q (ACT-R: ?)
│   │   └── ★ t (ACT-R: ~)
│   └── rule_and_production.py  
│       ├── ★ rule_
│       ├── ★ rule_sequence_
│       ├── ★ production
│       └── ★ production_sequence
├── convience/
│   ├── chunks.py
│   │   ├── ★ SimpleGoalChunk (isa='goal', phase=...)
│   │   ├── ★ VisuallocationChunk (isa='_visuallocation', screen_x=..., screen_y=..., attended=...)
│   │   ├── ★ VisualChunk (isa='_visual', value=..., cmd=..., screen_pos:VisuallocationChunk=...)
│   │   └── ★ ManualChunk (isa='_manual', cmd=..., key=...)
│   └── rules.py
│       ├── ★ request_ (ACT-R: +)
│       ├── ★ subsumption_ (ACT-R: =)
│       ├── ★ query_ (ACT-R: ?)
│       ├── ★ flush_ (ACT-R: ~)
│       ├── ★ is_simple_goal_
│       ├── ★ is retrieved_
│       ├── ★ retrieve_
│       └── ★ press_key
└── helpers/
    ├── actr_data_types.py
    │   ├── ☆ RuleType (+, =, ?, ~)
    │   ├── ☆ Buffer (GOAL, RETRIEVAL, VISUAL, VISUAL_LOCATION, MANUAL, IMAGINAL)
    │   ├── ☆ BufferQuery
    │   ├── ★ BufferStatus (holds BufferQuery enum for each buffer)
    │   └── ★ BufferExtraTest (holds BufferQuery enum for VISUAL_LOCATION buffer)
    └── enum_sequence.py
        └── ★ Sequence
```

### Naming Conventions

To bridge the gap between python, LISP, and the ACT-R framework, naming conventions are used differently than they usually are (and arguably should be) in python:

`ALL_UPPER_SNAKECASE` ❯ indicating buffers (even if its in a function or parameter name)
`lower_snakecase_with_trailing_underscore_` ❯ indicating rules (implemented as classes / objects; also for names of variable holding rule objects)
`lower_snakecase_without_trailing_underscore_production` ❯ indicating productions (implemented as classes / objects; also for names of variable holding production objects; ending in "production" / "prodseq" to distinguish from the notation for other variables)
`UpperCamelcaseChunk` ❯ chunks (implemented as classes / obect; ending in "Chunk" to distinguish from Enums and other class names)

The choice to write rules and production names in lower snakecase was made to make them very easy visually distinguishable from chunks (the lower snakecase form was chosen for the rules and productions since they have a stronger resemblance to functions that have lower snakecase as their usual notation).

### Understanding the syntax 

#### Rules & productions

There are two different notations for rules: a class-based ones and a more LISP-like (kind of functional-based) one that, in its core, also makes use of the first implementation.

##### Lisp-like notation

To make use of the notation as easy as possible for people already familiar with ACT-R notation, the following syntax tries to resemble usual ACT-R as close as possible.

Let's start with the rule headers, for example `=g>` or `+manual>`. As python does not allow the symbols specifying the type of interaction with the buffer (`+,=,?,~`) in class and variable names, the following substitution was made:

| LISP ACT-R | pyactr oo syntax |
| ---------- | ---------------- |
| +          | p                |
| =          | e                |
| ?          | q                |
| ~          | t                |

Only the first character of each symbols name was used to not increase character count. If this feels too minimalist, simply use aliases during the importing, for instance:

```python
from pyactr_oo_syntax.base.lisplike import p as request
```

After the symbol or character, the buffer name is specified. In this object-oriented syntax, the are functions returning `rule_` objects (i.e., similar to constructors) and are written like this:

```python
e.GOAL_() # =g>
p.MANUAL_() # +manual>
```

Inside the parentheses, chunks can now be specified by providing their properties / slots as key-value pairs:

```python
e.GOAL_(              # =g>
    isa='goal',       # isa goal
    phase='start'     # phase start
)
```

The `isa` property is always required – expect when no chunk information is given at all or for querying the buffer status. The trailing underscore after `GOAL` should indicate that a `rule_` obejct is returned. 

##### Class notation

The LISP-like is only for convenience and also makes use the abstract base class `rule_` which takes as parameters the rule type (`+,=,?,~`), the buffer (`g,retrieval,manual,visual,...`) – with an optional name for the imaginal buffer if it is used –, the type of the chunk (`isa`), and an arbitrary amount of keyword arguments as further chunk properties / slots.

Each of `p,e,q,t` has an equivalent class inheriting from `rule_` using the class notation:

```python
from pyactr_oo_syntax.helpers.actr_data_types import Buffer
from pyactr_oo_syntax.convenience.rules import (
    request_, # p / +
    subsumption_, # e / =
    query_, # q / ?
    flush_ # t / ~
)

# Example usage
subsumption_(buffer=Buffer.GOAL,
             isa='goal',
             phase='start')

request_(buffer=Buffer.IMAGINAL,
         imaginal_buffer_name='objects',
         isa='object',
         value=5)
```

Note: the `imaginal_buffer_name` is always automatically prepended with "imaginal_".


##### Combining rules

To combine several rules in the usual ACT-R notation, rules are just written after another, in python this will not work. Instead, rules are combined with an `&` (see `__and__` and `__rand__` methods of `rule_`, `rule_sequence_` and `production` for more on the implementation). The result will be a `rule_sequence_` object:

```python
(
    rule1_ &
    rule2_
)
```

Since they function like normal operators in python, they parentheses can be used as usual as well (this might help with structuring them for better readability):

```python
(
    (
        rule1a_ &
        rule1b_
    ) &
    rule2
)
```

Rules in LISP-like and class notation can be combined without any problem (because the LISP-like notation also returns object of type `rule_`).

##### Creating productions

Productions in ACT-R consist of a left- and a right-hand-side – each consisting of one or more rules – separated by the `==>` symbol. In the object-oriented python syntax, this symbol must be replaced with an already existing python operator. Simply due to the highest visual resemblance, the right-shift operator `>>` was chosen (see `__rshift__` method of `production_` for more on the implementation). Production can then be created as follows:

```python
(
    rule1_
    >>
    rule2_
)

(
    rule1_ &
    rule2_ &
    rule3_
    >>
    rule4_ &
    rule5_
)

rule_sequence_ =\
(
    rule1_ &
    rule2_
)

(
    rule_sequence_
    >>
    rule_3
)
```

Again, more parentheses can be used for structuring. Rule sequences might also be saved as variable to allow for reuse in multiple production.

For convenience and organisation, multiple production can be combined in a `production_sequence` by means of the `+` operator (see `__add__` and `__radd__` methods of `production_sequence_` and `production` for more on the implementation):

```python
production_sequence =\
(
    production1 +
    production2
)

production_sequence + production3
```

Productions and production sequences can then be added to the actr model via the `add_to_model` method, which also allows to specify utility and reward:

```python
(
    rule1
    >>
    rule2
).add_to_model(actr_model)

production.add_to_model(actr_model, utility=1, rewards=2)
```

To ensure easy debugging, `add_to_model` returns the added production in pyactr string form:

```python
prod = (rule1 >> rule2).add_to_model()
print(prod)
```

##### Reusing rules & productions

As the `+`, `>>`, and `+` operations for rules, rule sequences, productions, and production sequences always generate new object, they can be saved in a variable and used multiple times in other rule sequences, productions, or productions sequences without the original object changing -- see the following example:

```python
__my_rule_sequence_ =\
(
    rule1_ &
    rule2_
)

__my_rule_sequence_ & rule2b

my_production1 =\
(
    __my_rule_sequence_ &
    rule3_
    >>
    rule4_
)

my_productions2 =\
(
    __my_rule_sequence_
    >>
    rule5_
)

# __my_rule_sequence_ is still in the state as defined in the beginning; my_production1 and my_production2 do not contain rule2b_
```

By using the walrus operator (`:=`), they can also be defined in-place (note: assignment must be enclosed in parentheses to limit scope):

```python
my_production1 =\
(
    (
        __my_rule_sequence :=\
        rule1_ &
        rule2_
    ) &
    rule3_
    >>
    rule4_
)

my_production2 =\
(
    __my_rule_sequence_
    >>
    rule5_
)
```


#### Chunks

To make the already existing `Chunk` class of pyactr compatible with this object-oriented syntax, pyactr's `Chunk` class was extended to be "stringable" (necessary for later conversion of production into string form) – by means of (re)defining the `__str__` method – and to be usable with python's dictionary unpacking operator (`**`) – by means of defining the `__getitem__` and `keys` methods.

The unpacking allows for the following usage in connection with rules:

```python
from pyactr_oo_syntax.base.chunk import AdvChunk
from pyactr_oo_syntax.base.lisplike import e
from pyactr_oo_syntax.convience.rules import subsumption_

e.GOAL_(**AdvChunk(isa='goal', value='test'))
subsumption_(**AdvChunk(isa='goal', value='test'))
subsumption_.from_chunk(AdvChunk(isa='goal', value='test'))
```

Here, the AdvChunks are basically converted into dictionaries containing their properties / slots, which (due to the unpacking operator) will then be captured as keyword arguments (predefined and/or as `**kwargs`) by the respective rule. 

The `from_chunk` method might be specified in rules for convenience but does not have to be. In fact, rules inheriting from other rules that already specify a `from_chunk` method cannot override its parameters (and their types).

An `AdvChunk` can be added to declarative memory with the `add_to_decmem` method, like:

```python
AdvChunk(isa='goal', value='test').add_to_decmem(actr_model)
```

For more specialized chunks that already specify certain properties / slots, those slots should be made available as class properties, for example, as follows:

```python
from pyactr_oo_syntax.base.chunk import AdvChunk
from pyactr_oo_syntax.helpers.data_types import static_chunk_slot

class SimpleGoalChunk(AdvChunk):
    @static_chunk_slot
    def isa(cls) -> str:
        return 'goal'

    def __init__(self, phase:str):
        super().__init__(isa=SimpleGoalChunk.isa, phase=phase)

```

The definition of the property / slot uses `@static_chunk_slot` followed by a getter return the corresponding value directly instead of using a simple class variable `isa = 'goal'`, so it cannot be overwritten later.


### Tips on how to fully utilize this syntax

- use enums to create own data types (like `MovementDirection`)
- save rule sequences that are used in multiple productions as variables and use those variables for defining the respective productions
- define own rule and chunk classes (use previously created enums and type annotations while doing so)
- structure implementations by separating elements into several files, import productions and production sequences into a main file and only then add them to the model (make helper variables private, so they cannot be imported, which helps with decluttering auto-suggestions) 


## Some Notes

Currently, not everything that can be done with LISP ACT-R / pyactr can be done with this object-oriented syntax. For elements that are currently supported, please report any errors. 

As the productions are generally built before a simulation runs, the new syntax should not influence the time needed for executions during a simulation (however, the added conversions might increase reaction times if and while more productions are added during the simulation; this might have severe implications for certain experiments and their evaluation).


## Acknowledgements & License

As this project is only a wrapper-like extension of Jakub Dotlačil's [pyactr](https://github.com/jakdot/pyactr), it would not work or exist without it!

This project was inspired during the "Human-Aware AI" course (summer semester 2025) by Prof. Dr. Nele Rußwinkel, Rebecca von Engelhardt, Thomas Sievers, and Bastian Mannerow at the University of Lübeck.

This project is published under the GNU General Public License v3.0.
