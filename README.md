# Diatom

Diatom is a very simple data serialization language, it tries to be human readable and gets rid of a lot of useless punctuation.

> This spec is a newer version of Diatom, designed to be more "general use" than the older format

A simple Diatom file is very similar to JSON, but commas and colons are not required, they are just cosmetics used for readability.

```diatom
menu {
  id file
  value "File"
  popup {
    menuitem [
      { value "New", onclick "CreateNewDoc()" }
      { value "Open", onclick "OpenDoc()" }
      { value "Close", onclick "CloseDoc()" }
    ]
  }
}
```

## Values

Diatom has 5 value types:

- Number: integers `-1 42` and decimals `3.14 -1.44`
- String: normal strings `"Hello, world"` can use escape sequences like `\n` `\t` etc.; raw strings <code>\`Hello, world\`</code> cannot
- Symbol: an identifier made of alphanumerics and the symbols `_-/?!*`, must start with a letter
- Array: a list of values `[ 1 2 "hi" no-commas ]` 
- Map: a mapping of identifiers to values `{ name "Diatom", amount 14, is-boxed? no }`

There are no booleans or null.

## Tags

Any value can have a tag before it, tags are symbols preceded by `@`.

Tags are like type annotations to be used during parsing.

```
items [
  @Potion { name "Grass" description `turns you into grass` }
  @Ring { name "Brass Ring" description `its very cold, will dedigitate your finger` }
]
```

## Parsing

### `Error() → error`

Returns the last error or `ok` if there was none.

### `Tag() → string`

Returns the last tag read or empty if there was none.

### `NextKind() → string`

Return if the next thing is a `tag` or `symbol`, `number`, `string`, `array` or `map`.

### `ReadTag() → string`

Reads a tag, returns empty string if there is none.

Does not error if tag is absent.

### `ReadSymbol() → string`

Reads a symbol, returns empty string on failure.

- error `not a symbol` if there is no symbol

### `ReadString() → string`

Reads a string, returns empty string on failure.

- error `not a string` if there is no string
- error `unterminated string` if unterminated

### `ReadInteger() → integer`

Reads a number and expects it to be an integer. If possible should be a big integer, otherwise signed 64 bits should be the default.

- Input `42` → Output `42`
- Input `42.5` → Output `42`, error `not an integer`
- Input `name` → Output `0`, error `not a number`
- (no big int) Input `49340983473984832` → Output `0`, error `integer overflow`

#### Variations

- `ReadInteger32() → i32`
- `ReadInteger64() → i64`
- `ReadBigInteger() → bigint`

### `ReadNumber() → (float, error)`

Reads a number. Should be 64 bit floating point.

- Input `42` → Output `42.0`
- Input `42.5` → Output `42.5`
- Input `name` → Output `0`, error `not a number`

### `ReadArray(inside: bool) → (inside: bool)` or `ReadArray(inside: ref bool)`

Used to iterate through an array:

- `ReadArray(inside) →`
  - if `not inside`: try to consume `[`, otherwise error `not an array` and `→ inside = false`
  - if input has `]` consume and `→ inside = false`
  - else `→ inside = true`

Python example (in python the iterator variant is preferred):

```py
list = []
i = False
while diatom.read_array(i):
  n = diatom.read_number()
  list.append(n)
```

C example:

```c
double numbers[6] = {0};
int inside = 0;
for (int i = 0; diatomReadArray(diatom, &inside); ++i) {
  if (diatomError(diatom)) return false;
  if (i >= 6) {
    printf("Too many items\n");
    return false;
  }
  double n = diatomReadNumber(diatom);
  if (diatomError(diatom)) return false;
  numbers[i-1] = n;
}
return true;
```

#### Variations

- `IterArray() → Iterator` for languages with iterators

### `ReadKey(inside: bool) → (key: string, inside: bool)` or `ReadKey(inside: ref bool) → string`

Used to iterate through a map:

- `ReadKey(inside) →`
  - if `not inside`: try to consume `{`, otherwise error `not a map` and `→ "", inside = false`
  - if input has `}` consume and `→ "", inside = false`
  - try `key = ReadSymbol()`, otherwise `→ "", inside = false`
  - if input has `}` error `map missing value` and `→ key, inside = false`
  - else `→ key, inside = true`

Python example (in python the iterator variant is preferred):

```py
dict = {}
i = False
key, i = diatom.read_key(i)
while i:
  n = diatom.read_number()
  dict[key] = n
  key, i = diatom.read_key(i)
```

C example:

```c
int inside = 0;
int len;
char *key;
for (int i = 0; key = diatomReadKey(diatom, &inside, &len); ++i) {
  if (diatomError(diatom)) return false;
  double n = diatomReadNumber(diatom);
  if (diatomError(diatom)) return false;
  printf("%.*s => %f\n", len, key, n);
}
return true;
```

#### Variations

- `IterMap() → Iterator` for languages with iterators

### Things to consider

Some functions can take an optional argument to expect a value and error if not found, like:

- `ReadString("name")`
- `ReadSymbol("name")`
- `ReadInteger(2)`
- `ReadNumber(2.5)`

## Example

```
items [
  @Potion { name "Grass" description `turns you into grass` }
  @Ring { name "Brass Ring" description `its very cold, will dedigitate your finger` }
]
```

Python, with exceptions and expectations.

```python
ITEM_SCHEMA = {
  "name": diatom.read_string,
  "description": diatom.read_string,
}
ITEMS_NAME = {
  "Potion": Potion,
  "Ring": Ring,
}

def read_item():
  try:
    diatom.read_key("items")
    for i in diatom.iter_array():
      tag = diatom.read_tag()
      constructor = ITEMS_NAME.get(tag, None)
      if constructor is None:
        raise Exception("Invalid Item")
  
      data = {}
      for key in diatom.iter_map():
        f = ITEM_SCHEMA.get(key, None)
        if f is None: raise Exception("Invalid field")
        dict[key] = f()
      
      items.register(constructor(**data))
  except:
    print("Error parsing document")
```
