# Old Diatom

This is the first version of diatom, it was created for an task list app also called Diatom, the app was abandoned.

It is similar to current diatom, differences are:

- no maps (except top level)
- no tags
- only raw strings
- arrays are written with `{}`

Example:

```diatom
# Menu
id file
value `File`
popup { # menu items
  { `New` `Open` `Close` } # value
  { `CreateNewDoc()` `OpenDoc()` `CloseDoc()` } # onclick
}
```

## Values

The first level of the file is a mapping of symbols to values, values can be:

- Number: integers `-1 42` and decimals `3.14 -1.44`
- String: <code>\`Hello, world\`</code> has no escape character
- Symbol: an identifier made of alphanumerics and the symbols `_-/?!*`, must start with a letter
- Array: a list of values `{ 1 2 "hi" no-commas }`

Everything else works the same as modern Diatom.
