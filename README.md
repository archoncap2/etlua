# etlua

Embedded Lua templating

## Install

```bash
$ luarocks install --server=http://rocks.moonscript.org etlua
```

## Tutorial

```lua
local etlua = require "etlua"
local template = etlua.compile([[
  Hello <%= name %>,
  Here are your items:
  <% for i, item in pairs(items) do %>
   * <%= item -%>
  <% end %>
]])

print(template({
  name = "leafo",
  items = { "Shoe", "Reflector", "Scarf" }
}))

```

## Reference

The following tags are supported

* `<% lua_code %>` runs lua code verbatim
* `<%= lua_expression %>` writes result of expression to output, HTML escaped
* `<%- lua_expression %>` same as above but with no HTML escaping

Any of the embedded Lua tags can use the `-%>` closing tag to suppress a
following newline if there is one, for example: `<%= 'hello' -%>`.

The module can be loaded by doing:

```lua
local etlua = require "etlua"
```

### Methods

#### `func = etlua.compile(template_string)`

Compiles the template into a function, the returned function can be called to
render the template. The function takes one argument: a table to use as the
environment within the template. `_G` is used to look up a variable if it can't
be found in the environment.

#### `result = etlua.render(template_string, env)`

Compiles and renders the template in a single call. If you are concerned about
high performance this should be avoided in favor of `compile` if it's possible
to cache the compiled template.

### Errors

If any of the methods fail they will return `nil`, followed by the error
message.

### How it works

* Templates are transparently translated into Lua code and then loaded as a
  function. Rendering a compiled template is very fast.
* Any compile time errors are rewritten to show the original source position in
  the template.
* The parser is aware of strings so you can put closing tags inside of a string
  literal without any problems.

## Raw API

The raw API is a bit more complicated but it lets you insert code between the
compile stages in addition to exposing the internal buffer of the template.

All methods require a parser object:

```lua
local parser = etlua.Parser()
```

#### `lua_code, err = parser.compile_to_lua(etlua_code)`

Parses a string of etlua code, returns the compiled Lua version as a
string.

Here's an example of the generated Lua code:

```lua
local parser = etlua.Parser()
print(parser:compile_to_lua("hello<%= world %>"))
```

```lua
local _b, _b_i, _tostring, _concat, _escape = ...
_b_i = _b_i + 1
_b[_b_i] = "hello"
_b_i = _b_i + 1
--[[9]] _b[_b_i] = _escape(_tostring( world ))
_b_i = _b_i + 1
_b[_b_i] = ""
return _b
```

There are a few interesting things, there are no global variable references,
all required values are passed in as arguments. Comments are inserted to
annotate the positions of where code originated from. `_b` is expected to be a
regular Lua table that is the buffer where chunks of the template are inserted
into.

#### `fn, err = parser.load(lua_code)`

Converts the Lua code returned by `parser.compile_to_lua` into an actual
function object. If there are any syntax errors then `nil` is returned along
with the error message. At this stage, syntax errors are rewritten to point to
the original location in the etlua code and not the generated code.

#### `result = parser.run(fn, env={}, buffer={})`

Executes a loaded function returned by `parser.load` with the specified buffer
and environment. Returns the result of fn, which is typically the buffer. The
environment is applied to `fn` with `setfenv` (a version is included for Lua
5.2).

### Example

For example we can render multiple templates into the same buffer:

```lua
parser = etlua.Parser()

first_fn = parser:load(parser:compile_to_lua("Hello "))
second_fn = parser:load(parser:compile_to_lua("World"))

buffer = {}
parser:run(first_fn, nil, buffer)
parser:run(second_fn, nil, buffer)

print(table.concat(buffer)) -- print 'Hello World'
```

## License

MIT, Copyright (C) 2013 by Leaf Corcoran

