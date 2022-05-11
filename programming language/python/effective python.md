# Pythonic
- Read *<The Zen of Python>* through executing `import this` in python interpreter.

## String & Bytes
- Prioritize the use of f-string.

## Iterator
- Introduce itertools and implement iterators to achieve less memory footprint.

## Assignment Expression
- Use **:=** (walrus operator) for better readability.

# List & Dictionary
- Prefered catch-all unpacking than slicing while fetching value from composite type object (e.g. list, tuple).

## Missing Key
- Handle missing key with object method get and or operarator, or overwriting dunder method `__missing__` in advanced, instead of catching KeyError.
