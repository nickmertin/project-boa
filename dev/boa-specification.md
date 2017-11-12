# Boa Language and Platform Specification

Copyright (c) 2017 Nicholas Mertin.

## 1 Syntax

The Boa language is designed to be extremely simple, yet extensible; most features that would traditionally be part of the language, such as object-oriented programming, are implemented instead in the standard library. To make these features, as well as user-created ones, more intrinsic to the language, there is no syntactical difference between "builtins", or pieces built into the language itself, and separately implemented features.

### 1.1 Expressions

Many parts of the Boa language depend on expressions, so it makes sense to define them first. Most elements of expressions should be familiar from other programming languages. All expressions have a value, which is either defined at compile time, or computed at runtime. There are a few types of expressions:

Expression Type|Description|Regular Expression|Examples
---------------|-----------|------------------|--------
Integer literal | An integer of any length, positive or negative. Integers are in decimal (base-10) by default, but can be prefixed by `0x` to be interpreted as hexidecimal (base-16), `0` to be interpreted as octal (base-8), or `0b` to be interpreted as binary (base-2). | `^([-+]?)(0|0x|0b|)([0-9]+)$` | `17`, `-0x4F`, `077`, `-0b1101`
Fractional number literal | A number, positive or negative, with a whole and fractional part of any length; a "decimal" number. Always in base-10. | `^([-+]?)([0-9]*)\.([0-9]+)$` | `-0.15`, `12.0`
Character literal | A single character, surrounded by single quotation marks (`'`). Any character can be represented using an escape sequence, which always starts with a backslash (`\`). | `^'([^\\]|\\(?:[\\'"abfnrtv]|[0-7]{3}|x[0-9a-f]{2,8}))'$` | `'a'`, `'!'`, `'\''`, `'\x3f7a'`
String literal | A series of zero or more characters, surrounded by double quotation marks (`"`). The same escape sequences can be used as with characters. |  `^"((?:[^\\]|\\(?:[\\'"abfnrtv]|[0-7]{3}|x[0-9a-f]{2,8}))*)"$` | `"Hello"`, `"1\n2\n3\n4"`
Token literal | A series of letters, numbers, periods, and underscores, that does not begin with a number or period. Note that tokens are not surrounded by any delimiter. | `^([a-zA-Z_][a-zA-Z0-9_\.]*)$` | `test`, `foo`
Variable | A stored variable, referenced by its name prefixed with a dollar sign (`$`) | `^\$([a-zA-Z0-9_]+(?:\.[a-zA-Z0-9_])*)$` | `$temp`, `$7`
Variable address | The address of a stored variable, referenced by its name prefixed with an ampersand (`&`) | `^&([a-zA-Z0-9_]+(?:\.[a-zA-Z0-9_])*)$` | `&temp`, `&7`
Complex expression | A series of expressions, surrounded by parentheses, and with various prefix unary operators and infix binary or ternary operators | N/A | `($x + 1)`, `(12 / $y + $offset)`
Array expression | A series of expressions inside square brackets. | N/A | `[1 2 7 test $foo -3.5]`, `[&x &y &z]`
Directive expression | A directive; see section 1.2. | N/A | `@sqrt $x`, `@log ($x + 5)`
Lambda expression | A series of directives inside curly braces. Note: no directives or expressions contained inside a lambda expression are evaluated until the lambda is evaluated; see section 2.7. | N/A | `{ @print $element }`

Complex expressions can any of the following unary (prefix) operators:

Unary Operator|Name|Result
--------------|----|------
`+` | Identity | The original value.
`-` | Negate | The original value, with the opposite sign.
`~` | Bitwise NOT | The original value, with all bits flipped.
`!` | Logical NOT | The opposite of the implied Boolean conversion of the original value.

Complex expressions can contain any of the following binary (infix) operators:

Binary Operator|Precedence|Name|Result
---------------|----------|----|------
`->` | 1 | Dereference Member | A member or other object, identified by the right value, that the left value has access to.
`*` | 2 | Multiplication | The product of the original values.
`/` | 2 | Division | The quotient of the division of the left value by the right value.
`%` | 2 | Remainder | The remainder of the division of the left value by the right value.
`+` | 3 | Addition | The sum of the original values.
`-` | 3 | Subtraction | The difference of the right value from the left value.
`<<` | 4 | Left Bit Shift | The left value shifted left by a number of bits specified by the right value.
`>>` | 4 | Right Bit Shift | The left value shifted right by a number of bits specified by the right value.
`<:` | 4 | Left Bit Rotate | The left value rotated left by a number of bits specified by the right value.
`>:` | 4 | Left Bit Rotate | The left value rotated right by a number of bits specified by the right value.
`<` | 5 | Less Than | True if the left value is less than the right value, otherwise false.
`>` | 5 | Greater than | True if the left value is greater than the right value, otherwise false.
`<=` | 5 | Less Than Or Equal To | True if the left value is less than or equal to the right value, otherwise false.
`>=` | 5 | Greater Than Or Equal To | True if the left value is greater than or equal to the right value, otherwise false.
`==` | 6 | Equal To | True if the original values are equal, otherwise false.
`!=` | 6 | Not Equal To | True if the original values are not equal, otherwise false.
`&` | 7 | Bitwise AND | The combination of the original values by binary AND of each bit.
`^` | 8 | Bitwise XOR | The combination of the original values by binary XOR of each bit.
`|` | 9 | Bitwise OR | The combination of the original values by binary OR of each bit.
`&&` | 10 | Logical AND | The logical AND result of the implied Boolean conversions of the original values.
`||` | 11 | Logical OR | The logical OR result of the implied Boolean conversions of the original values.

Additionally, complex expressions may contain the ternary operator, which consists of a Boolean expression, followed by a question mark (`?`), followed by an expression to evaluate if the condition is true, followed by a colon (`:`), followed by an express to evaluate if the condition is false.

All unary operators are evaluated with higher precedence than all binary operators, and the ternary operator with lower precedence than all binary operators.

### 1.2 Directives

The Boa language is built around the concept of directives, which consist of an "at" sign (`@`), followed immediately by the name of the directive, followed by zero or more expressions, called arguments, separated only by spaces. For example, using a directive called `print` with the argument `"Hello, world"`:

    @print "Hello, world"

Directives are a more abstract version of what other programming languages might call "functions"; they are significantly more extensible, and have a syntax that allows for more idiomatic or intrinsic code. For example, via directive expressions, we can "chain" directives:

    @print @sqrt 7

This evaluates thes `sqrt` directive with the single argument `7`, then the `print` directive with the result of the `sqrt` directive.

In order for this syntax to work, all directives must take a defined, constant number of arguments; otherwise, it would be impossible to determine, for example, which arguments belong to which directives in the following line:

    @complex @sqrt 2 1

It's possible that the `complex` directive takes 2 arguments, and the `sqrt` directive takes 1 argument, or that the `complex` directive takes 1 argument, and the `sqrt` directive takes 2 arguments. It's also possible that the `complex` directive does not take any argument, while the `sqrt` driective takes 2 arguments; in this case, `complex` would be evaluated first, followed by `sqrt`; the result of `sqrt` would be ignored.

## 2 Builtins

There are several directives, called builtins, which are built into the Boa language itself; they are provided by the compiler/interpreter, as opposed to library or user-defined directives, which are implemented in Boa.

### 2.1 `def`

This is the most essential directive: it defines another directive. It should be obvious why this must be provided as a builtin.

#### 2.1.1 Usage

    @def <name> <arg_types> <body>

#### 2.1.2 Arguments

- `name` - the name of the new directive, as a token.
- `arg_types` - an array of type specifications for the arguments; see section 3.1.2.

#### 2.1.3 Result

The result is the value of the argument `name`.

### 2.2 `export`

This directive allows a module to provide directives to be used by other modules.

#### 2.2.1 Usage

    @export <name>

#### 2.2.2 Arguments

- `name` - the name(s) of the directives to export; can either be a token, or an array of tokens.

#### 2.2.3 Result

The result is the name of the module, as a token.

### 2.3 `import`

This directive imports directives that are exported from another module.

#### 2.3.1 Usage

    @import <name>

#### 2.3.2 Arguments

- `name` - the name of the module to import directives from, as a token. The order in which the filesystem is searched for modules is described in section 4.2.

#### 2.3.3 Result

The result is the value of the argument `name`.

### 2.4 `expand`

This directive makes all directives from a module or submodule available without using the module as a prefix. It also imports the module if it is not already imported.

#### 2.4.1 Usage

    @expand <name>

#### 2.4.2 Arguments

- `name` - the name of the module or submodule to expand, as a token.

#### 2.4.3 Result

The result is the value of the argument `name`.

### 2.5 `label`

This directive defines a label that may be jumped to by a `goto` directive. Only one label with any given name may be declared within any lambda, or at the global level of any module.

#### 2.5.1 Usage

    @label <name>

#### 2.5.2 Arguments

- `name` - the name of the new label, as a token.

### 2.6 `goto`

This directive jumps immediately to a label named by its argument. To resolve the location of this label, the compiler/interpreter will search each frame of the call stack, until and including the global scope of a module; the search will not cross the boundary of a module import.

#### 2.6.1 Usage

    @goto <name>

#### 2.6.2 Arguments

- `name` - the name of the label to jump to, as a token.

### 2.7 `lambda`

This directive evaluates a lambda expression, with additional variables optionally created.

#### 2.7.1 Usage

    @lambda <lambda> <args>

#### 2.7.2 Arguments

- `lambda` - the lambda expression to evaluate.
- `args` - an array of values to pass as arguments to the lambda body.

#### 2.7.3 Result

The result is the value returned by the lambda body, or null if it does not return a value.

### 2.8 `null`

This directive always evaluates to null.

#### 2.8.1 Usage

    @null

#### 2.8.2 Result

The result is always null.

### 2.9 `context`

This directive evaluates a lambda, with no extra variables, in the context of the calling scope; for example, this can be used to define additional directives in that scope.

#### 2.9.1 Usage

    @context <lambda>

#### 2.9.2 Arguments

- `lambda` - the lambda to evaluate.

### 2.10 `symbol.new`

This directive defines a named symbol that can be used elsewhere, even in other modules or before it is defined here, so long as there are no cyclic dependencies.

#### 2.10.1 Usage

    @symbol.new <name> <value>

#### 2.10.2 Arguments

- `name` - the name of the new symbol, as a token. This must be unique across all symbols in the program.
- `value` - the value of the new sybbol.

#### 2.10.3 Result

The result is the value of the argument `name`.

### 2.11 `symbol.use`

This directive evaluates a named symbol created with the directive `symbol.new`.

#### 2.11.1 Usage

    @symbol.use <name>

#### 2.11.2 Arguments

- `name` - the name of the symbol, as a token.

#### 2.11.3 Result

The result is the value of the named symbol.

### 2.12 `return`

This directive exits a lambda, providing a result.

#### 2.12.1 Usage

    @return <result>

#### 2.12.2 Arguments

- `result` - the result of the lambda.

### 2.13 `arg_types`

This directive gets the type specifications of the parameters of a directive.

#### 2.13.1 Usage

    @arg_types <name>

#### 2.13.2 Arguments

- `name` - the name of the directive, as a token.

#### 2.13.3 Result

The result is an array of type specifications, equivalent to the `arg_types` argument provided to the directive `def` upon the creation of the directive.

### 2.14 `asm`

This directive allows the code to use native assembly instructions. Non-JIT intepreter implementations are not required to implement this directive.

#### 2.14.1 Usage

    @asm <mnemonic> <args>

#### 2.14.2 Arguments

- `mnemonic` - the name of the assembly instruction, as a token.
- `args` - an array of the arguments to provide to the instruction. For x86-like platforms, this must be in the Intel syntax order.

#### 2.14.3 Result

If a destination-type argument is given the value null, then a destination will be allocated, and the result placed in it will the result of the directive; otherwise, the result will be null.

### 2.15 `asm.reg`

This directive gets the value of a native CPU register. Non-JIT intepreter implementations are not required to implement this directive.

#### 2.15.1 Usage

    @asm.reg <register>

#### 2.15.2 Arguments

- `register` - The name of the CPU register to read from, as a token.

#### 2.15.3 Result

The result is the value of the register.

### 2.16 `token.combine`

This directive combines an ordered series of tokens to create a new one, by joining them with periods (`.`).

#### 2.16.1 Usage

    @token.combine <tokens>

#### 2.16.2 Arguments

- `tokens` - an array of tokens to combine.

#### 2.16.3 Result

The result is a token that is the combination of the other given tokens.

### 2.17 `token.string`

This directive converts a token to a string.

#### 2.17.1 Usage

    @token.string <token>

#### 2.17.2 Arguments

- `token` - the token to convert to a string.

#### 2.17.3 Result

The result is the token as a string.

### 2.18 `thread.put`

This directive sets a value in the thread-local data dictionary.

#### 2.18.1 Usage

    @thread.put <key> <value>

#### 2.18.2 Arguments

- `key` - the key of the value in the dictionary, as a token.
- `value` - the new value.

#### 2.18.3 Result

The result is the value of the argument `value`.

### 2.19 `thread.get`

This directive gets a value in the thread-local data dictionary.

#### 2.19.1 Usage

    @thread.get <key>

#### 2.19.2 Arguments

- `key` - the key of the value in the dictionary, as a token.

#### 2.19.3 Result

The result is the value from the dictionary.

### 2.20 `var`

This directive creates a new local variable.

#### 2.20.1 Usage

    @var <type> <name> <value>

#### 2.20.2 Arguments

- `type` - the type specification for the new variable.
- `name` - the name of the new variable, as a token. This must be unique across the current scope.
- `value` - the initial value of the variable.

#### 2.20.3 Result

The result is a reference to the variable.

### 2.21 `defined`

This directive checks if a directive is defined.

#### 2.21.1 Usage

    @defined <name>

#### 2.21.2 Arguments

- `name` - the directive name to check for.

#### 2.21.3 Result

The result is true if the directive is defined, otherwise false.

### 2.22 `set`

This directive sets the value of a variable.

#### 2.22.1 Usage

    @set <name> <value>

#### 2.22.2 Arguments

- `name` - the name of the variable.
- `value` - the new value of the variable.

#### 2.22.3 Result

The result is the a reference to the variable.

### 2.23 `primitive`

This directive produces the type information object for a given primitive type.

#### 2.23.1 Usage

    @primitive <name>

#### 2.23.2 Arugments

- `name` - the name of the primitive type; see section 3.1.1.

#### 2.23.3 Result

The result is the type information object for the given primitive.

### 2.24 `type.build`

This directive creates a custom type.

#### 2.24.1 Usage

    @type.build <size>

#### 2.24.2 Arguments

- `size` - the size of the new type, in bytes, as a non-negative integer.

#### 2.24.3 Result

The result is a new type builder; see section 3.3.

### 2.25 `type.extend`

This directive creates a custom type that extends an existing type.

#### 2.25.1 Usage

    @type.extend <super_type> <addional_size>

#### 2.25.2 Arguments

- `super_type` - the type object for the type that this type is extending.
- `additional_size` - the size, in bytes, that this type adds to the super type, as a non-negative integer.

#### 2.25.3 Result

The result is a new type builder; see section 3.3.

### 2.26 `type.get`

This directive gets the type of a value.

#### 2.26.1 Usage

    @type.get <value>

#### 2.26.2 Arguments

- `value` - the value of which to get the type.

#### 2.26.3 Result

The result is the type object of the value.

### 2.27 `type.super`

This directive gets the super type of a type.

#### 2.27.1 Usage

    @type.super <type>

#### 2.27.2 Arguments

- `type` - the type object of the type to get the super type of.

#### 2.27.3 Result

The result is the type object of the super type of the type, or null if the type does not extend another type.

### 2.28 `type.size`

This directive gets the size of instances of a type.

#### 2.28.1 Usage

    @type.size <type>

#### 2.28.2 Arguments

- `type` - the type object of the type to get the size of.

#### 2.28.3 Result

The result is the size of the type in bytes, as an integer.

### 2.29 `true`

This directive always evaluates to true.

#### 2.29.1 Usage

    @true

#### 2.29.2 Result

The result is always true.

### 2.30 `false`

This directive always evaluates to false.

#### 2.30.1 Usage

    @false

#### 2.30.2 Result

The result is always false.

## 3 Language Features

This section describes how to use various features of the Boa language.

### 3.1 Types

This section describes the function of the Boa type system.

#### 3.1.1 Primitive Types

Primitive types are types that are built into the Boa language itself; they are the only types not defined using the directives `type.build` or `type.extend`:

Name|Tokenized Name|Size|Description
----|--------------|----|-----------
Array | `array` | Platform pointer size \* 2 | An arbitrary, but fixed, number of elements, which can be of any type.
Boolean | `bool` | 1 | A value that is always either true or false.
Integer | `int` | 8 | A 64-bit signed integer.
Floating-point number | `float` | 8 | A double-precision (FP64) IEEE 754 floating point number.
Lambda | `lambda` | Variable | A callable reference to a piece of code and relevant context; a closure.
Null | `null` | 0 | The null value.
Pointer | `pointer` | Platform pointer size | A raw memory address.
Token | `token` | Platform pointer size | A series of numbers, letters, underscores, and period, which does not start with a number and neither begins nor ends with a period. Tokens cannot be constructed at runtime.
Type builder | `builder` | Variable | A special object that aids in the creation and implementation of custom types.
Type object | `type` | Platform pointer size | A special object that identifies a specific type.

#### 3.1.2 Type Specifications

A type specification is a lambda that can be used to check if a given value conforms to certain requirements.

#### 3.1.3 Type Builders

### 3.2 Arrays

Arrays, as described in section 1.1, can be created by placing values in square brackets:

    @var @primitive array foo [1 2 3]

## 4 Modules, Libraries, and Programs

### 4.1 Submodules and Filesystem Structure

### 4.2 Search paths

## 5 Standard Library
