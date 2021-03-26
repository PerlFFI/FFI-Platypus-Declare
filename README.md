# FFI::Platypus::Declare ![linux](https://github.com/PerlFFI/FFI-Platypus-Declare/workflows/linux/badge.svg)

(discouraged) Declarative interface to FFI::Platypus

# SYNOPSIS

```perl
use FFI::Platypus::Declare 'string', 'int';

lib undef; # use libc
attach puts => [string] => int;

puts("hello world");
```

# DESCRIPTION

This module is officially **discouraged**.  The idea was to provide a
simpler declarative interface without the need of (directly) creating
an [FFI::Platypus](https://metacpan.org/pod/FFI::Platypus) instance.  In practice it is almost as complicated
and makes it difficult to upgrade to the proper OO interface if the
need arises.  I have stopped using it mainly for this reason.  It will
remain as part of the Platypus core distribution to keep old code working,
but you are encouraged to write new code using the OO interface.
Alternatively, you can try the Perl 6 inspired [NativeCall](https://metacpan.org/pod/NativeCall), which
provides most of the goals this module was intended for (that is
a simple interface at the cost of some power), without much of the
complexity.  The remainder of this document describes the interface.

This module provides a declarative interface to [FFI::Platypus](https://metacpan.org/pod/FFI::Platypus). It
provides a more concise interface at the cost of a little less power,
and a little more namespace pollution.

Any strings passed into the `use` line will be declared as types and
exported as constants into your namespace, so that you can use them
without quotation marks.

Aliases can be declared using a list reference:

```perl
use FFI::Platypus [ 'int[48]' => 'my_integer_array' ];
```

Custom types can also be declared as a list reference (the type name
must include a ::):

```perl
use FFI::Platypus [ '::StringPointer' => 'my_string_pointer' ];
# short for FFI::Platypus::Type::StringPointer
```

# FUNCTIONS

All functions are exported into your namespace.  If you do not want that,
then use the OO interface (see [FFI::Platypus](https://metacpan.org/pod/FFI::Platypus)).

## lib

```
lib $libpath;
```

Specify one or more dynamic libraries to search for symbols. If you are
unsure of the location / version of the library then you can use
[FFI::CheckLib#find\_lib](https://metacpan.org/pod/FFI::CheckLib#find_lib).

## type

```
type $type;
type $type = $alias;
```

Declare the given type.

Examples:

```perl
type 'uint8'; # only really checks that uint8 is a valid type
type 'uint8' => 'my_unsigned_int_8';
```

## custom\_type

```perl
custom_type $alias => \%args;
```

Declare the given custom type.  See [FFI::Platypus::Type#Custom-Types](https://metacpan.org/pod/FFI::Platypus::Type#Custom-Types)
for details.

## load\_custom\_type

```perl
load_custom_type $name => $alias, @type_args;
```

Load the custom type defined in the module _$name_, and make an alias
with the name _$alias_. If the custom type requires any arguments, they
may be passed in as _@type\_args_. See [FFI::Platypus::Type#Custom-Types](https://metacpan.org/pod/FFI::Platypus::Type#Custom-Types)
for details.

If _$name_ contains `::` then it will be assumed to be a fully
qualified package name. If not, then `FFI::Platypus::Type::` will be
prepended to it.

## type\_meta

```perl
my $meta = type_meta $type;
```

Get the type meta data for the given type.

Example:

```perl
my $meta = type_meta 'int';
```

## attach

```perl
attach $name => \@argument_types => $return_type;
attach [$c_name => $perl_name] => \@argument_types => $return_type;
attach [$address => $perl_name] => \@argument_types => $return_type;
```

Find and attach a C function as a Perl function as a real live xsub.

If just one _$name_ is given, then the function will be attached in
Perl with the same name as it has in C.  The second form allows you to
give the Perl function a different name.  You can also provide a memory
address (the third form) of a function to attach.

Examples:

```perl
attach 'my_function', ['uint8'] => 'string';
attach ['my_c_function_name' => 'my_perl_function_name'], ['uint8'] => 'string';
my $string1 = my_function($int);
my $string2 = my_perl_function_name($int);
```

## closure

```perl
my $closure = closure $codeblock;
```

Create a closure that can be passed into a C function.  For details on closures, see [FFI::Platypus::Type#Closures](https://metacpan.org/pod/FFI::Platypus::Type#Closures).

Example:

```perl
my $closure1 = closure { return $_[0] * 2 };
my $closure2 = closure sub { return $_[0] * 4 };
```

## sticky

```perl
my $closure = sticky closure $codeblock;
```

Keyword to indicate the closure should not be deallocated for the life
of the current process.

If you pass a closure into a C function without saving a reference to it
like this:

```
foo(closure { ... });         # BAD
```

Perl will not see any references to it and try to free it immediately.
(this has to do with the way Perl and C handle responsibilities for
memory allocation differently).  One fix for this is to make sure the
closure remains in scope using either `my` or `our`.  If you know the
closure will need to remain in existence for the life of the process (or
if you do not care about leaking memory), then you can add the sticky
keyword to tell [FFI::Platypus](https://metacpan.org/pod/FFI::Platypus) to keep the thing in memory.

```
foo(sticky closure { ... });  # OKAY
```

## cast

```perl
my $converted_value = cast $original_type, $converted_type, $original_value;
```

The `cast` function converts an existing _$original\_value_ of type
_$original\_type_ into one of type _$converted\_type_.  Not all types
are supported, so care must be taken.  For example, to get the address
of a string, you can do this:

```perl
my $address = cast 'string' => 'opaque', $string_value;
```

## attach\_cast

```perl
attach_cast "cast_name", $original_type, $converted_type;
my $converted_value = cast_name($original_value);
```

This function creates a subroutine which can be used to convert
variables just like the [cast](https://metacpan.org/pod/FFI::Platypus::Declare#cast) function
above.  The above synopsis is roughly equivalent to this:

```perl
sub cast_name { cast($original_type, $converted_type, $_[0]) }
my $converted_value = cast_name($original_value);
```

Except that the [attach\_cast](https://metacpan.org/pod/FFI::Platypus::Declare#attach_cast)
variant will be much faster if called multiple times since the cast does
not need to be dynamically allocated on each instance.

## sizeof

```perl
my $size = sizeof $type;
```

Returns the total size of the given type.  For example to get the size
of an integer:

```perl
my $intsize = sizeof 'int'; # usually 4 or 8 depending on platform
```

You can also get the size of arrays

```perl
my $intarraysize = sizeof 'int[64]';
```

Keep in mind that "pointer" types will always be the pointer / word size
for the platform that you are using.  This includes strings, opaque and
pointers to other types.

This function is not very fast, so you might want to save this value as
a constant, particularly if you need the size in a loop with many
iterations.

## lang

```
lang $language;
```

Specifies the foreign language that you will be interfacing with. The
default is C.  The foreign language specified with this attribute
changes the default native types (for example, if you specify
[Rust](https://metacpan.org/pod/FFI::Platypus::Lang::Rust), you will get `i32` as an alias for
`sint32` instead of `int` as you do with [C](https://metacpan.org/pod/FFI::Platypus::Lang::C)).

In the future this may attribute may offer hints when doing demangling
of languages that require it like [C++](https://metacpan.org/pod/FFI::Platypus::Lang::CPP).

## abi

```
abi $abi;
```

Set the ABI or calling convention for use in subsequent calls
to ["attach"](#attach).  May be either a string name or integer value
from [FFI::Platypus#abis](https://metacpan.org/pod/FFI::Platypus#abis).

# SEE ALSO

- [FFI::Platypus](https://metacpan.org/pod/FFI::Platypus)

    Object oriented interface to Platypus.

- [FFI::Platypus::Type](https://metacpan.org/pod/FFI::Platypus::Type)

    Type definitions for Platypus.

- [FFI::Platypus::API](https://metacpan.org/pod/FFI::Platypus::API)

    Custom types API for Platypus.

- [FFI::Platypus::Memory](https://metacpan.org/pod/FFI::Platypus::Memory)

    memory functions for FFI.

- [FFI::CheckLib](https://metacpan.org/pod/FFI::CheckLib)

    Find dynamic libraries in a portable way.

- [FFI::TinyCC](https://metacpan.org/pod/FFI::TinyCC)

    JIT compiler for FFI.

# AUTHOR

Author: Graham Ollis <plicease@cpan.org>

Contributors:

Carlos D. Álvaro (cdalvaro)

# COPYRIGHT AND LICENSE

This software is copyright (c) 2020 by Graham Ollis.

This is free software; you can redistribute it and/or modify it under
the same terms as the Perl 5 programming language system itself.
